# Shared-Data StarRocks CN 节点磁盘管理与 StreamLoad 失败排查

## 背景

shared-data 架构下 CN 节点的本地盘是**缓存盘**，不是数据真源。但它会同时承载缓存、写入临时区、spill、主键索引等多种用途。理解每个目录的角色 + 三个核心配置的相互作用，是排查 StreamLoad / spill / 写入失败的基础。

入口问题往往是「BE 磁盘存了哪些数据」，但本质上要先分清当前是**存算一体的 BE**还是**存算分离的 CN** —— 两者磁盘画像差很多，排查方向也不同。

## CN vs BE：先分清节点类型

| 节点类型 | 磁盘角色 | 主要占用 |
|---|---|---|
| 经典 BE（存算一体） | 数据真源 | `/data/<shard>/<tablet>/` 下的 rowset/segment 全量 |
| CN（存算分离 / shared-data） | **本地缓存盘**，真源在对象存储 | starlet_cache、datacache 占大头 |

判错对象会让排查方向完全跑偏：CN 上把 starlet_cache 当数据真源去清理，会反复触发回源；BE 上去找 starlet_cache 目录则根本不存在。

## CN 节点磁盘上的目录全貌

按 CN 上典型占比从大到小：

| 目录 | 内容 | 配置项 |
|---|---|---|
| `starlet_cache/` | 湖表本地数据缓存（segment、索引副本），CN 上最大头 | `starlet_cache_dir` |
| `datacache/` | block cache，加速查询的数据块缓存 | `datacache_disk_path` |
| `spill/` | 查询执行内存不足时的落盘临时文件 | `spill_local_storage_dir` |
| `/data/<shard>/<tablet>/` | 临时 rowset/segment（写入未上传到对象存储前） | `DATA_PREFIX` (`be/src/storage/olap_define.h:63`) |
| `/trash/` | 已删除 tablet 的回收站 | `TRASH_PREFIX`，受 `trash_file_expire_time_sec` 控制 |
| `/persistent/` | 主键索引持久化文件（PK 表） | `PERSISTENT_INDEX_PREFIX` |
| `/snapshot/` | 快照、备份/克隆中转 | `SNAPSHOT_PREFIX` |
| `/tmp/`、`/clone/`、`/replication/` | 导入/克隆/复制临时区 | 同上头文件 |
| `error_log/`、`rejected_record/` | StreamLoad 错误行日志、拒收记录 | `ERROR_LOG_PREFIX` |

所有 prefix 字符串统一定义在 `be/src/storage/olap_define.h:60-73`。

## 三个核心配置的相互作用（最关键的认知）

| 配置 | 含义 | 开源默认 |
|---|---|---|
| `starlet_star_cache_disk_size_percent` | starlet cache 允许占盘的百分比上限 | 80（即可吃到 80%） |
| `starlet_cache_evict_high_water` | 触发淘汰的**剩余空间**阈值，默认 0.2 表示「空闲 < 20% / 已用 > 80%」时启动淘汰 | 0.2 |
| `trash_file_expire_time_sec` | 回收站文件保留时长 | 86400 (24h) |

**关键洞察**：默认配置下，**cache 上限 = 80%**、**淘汰触发线 = 80%**，意味着 cache 设计上就会把磁盘填到 ~80% 才开始淘汰。再叠加 `/data`、`/persistent`、`/spill`、log 等占 5%~10%，**整盘稳态在 85%~90% 是配置允许的"预期行为"，不是缓存失控**。

直接推论：如果磁盘告警阈值设在 80%~85%，**会被永远踩**。排查"为什么这么高"是没意义的——就是被这么设计的。**告警线必须高于稳态线**（≥88% 或 90%），或者改成基于 `/data + /spill` 这类"硬空间"目录而不是整盘。

## StreamLoad 写入失败的真实原因（重新定位）

整盘到 85% 本身**不会**让 StreamLoad 失败——缓存高水位被踩到会自动淘汰。真正让写入失败的是 **`/data` 临时区被挤光**：

- CN 写入新数据要先落 `/data/<shard>/<tablet>/`，再上传对象存储
- 如果 cache 占了 80%，剩下不到 20% 要被 `/data`、`/persistent`、`/spill` 共享
- 大批量写入、spill 高峰、PK 索引增长 → `/data` 短暂溢出 → 写入报错

错误信息 → 目录速查：

| 报错关键字 | 真实原因 |
|---|---|
| `No space left` | 整盘满（罕见，cache 应该会先淘汰） |
| `failed to write rowset` | `/data` 写入临时区不够 |
| `tablet not found` 配合磁盘高 | starlet_cache 异常 / 缓存元数据丢失 |
| `spill failed` | spill 目录被挤光 |

**先看错误码再看 du，比闷头清磁盘高效**。

## 调整思路

按目标：

| 目标 | 调整 |
|---|---|
| 给写入腾出硬空间 | 降低 `starlet_star_cache_disk_size_percent`（80 → 60~70），给 `/data`、`/spill` 留 30%+ 硬余量 |
| 让淘汰更早开始 | 提高 `starlet_cache_evict_high_water`（0.2 → 0.3），「空闲 < 30%」就启动淘汰，把节奏提前 |
| 加快 trash 释放 | 降低 `trash_file_expire_time_sec`（24h → 1800s 30min） |
| 告警阈值匹配现实 | 提到 88%~90%；或者改基于 `/data + /spill` 监控 |

按节点角色（多计算组场景）：
- **写入压力大**的计算组：cache cap 调到 55%~60%，留更多空间给 `/data`
- **查询为主**的计算组：cache cap 可保留 70%，命中率优先
- 不必一刀切

**注意**：调小 `starlet_star_cache_disk_size_percent` 会立即触发 cache 缩容 + evict，命中率会短期跌，应在低峰期操作；查询可能慢几个小时直到 cache 重新填充热数据。

## 排查命令

```bash
du -h --max-depth=1 <storage_root>/diskN/ | sort -h
du -h --max-depth=2 <storage_root>/diskN/starlet_cache/ 2>/dev/null
ls -la <storage_root>/diskN/trash/ | head
ls -la <storage_root>/diskN/spill/ 2>/dev/null
df -h <storage_root>/diskN
```

## 关键认知

1. **CN 磁盘是缓存盘，不是数据盘** —— 全盘满不会丢数据，但会立刻阻塞写入，因为 `/data`、`/persistent`、`/spill` 都共享同一块盘
2. **默认配置下磁盘稳态就是 80%+** —— 告警线必须高于这个，不然永远在响。"高磁盘"在 CN 上是设计预期，不是异常
3. **StreamLoad 失败 ≠ 整盘满** —— 大概率是 `/data` 被 cache 挤掉，看错误码比看整盘水位更准
4. **Trash 默认 24h 容易被忽视** —— 删大表/分区后磁盘曲线会延迟一天才回落，监控告警容易在这个窗口期触发

## 参考代码位置

- 目录前缀定义：`be/src/storage/olap_define.h:60-73`
- 相关配置：`be/src/common/config.h`
  - `storage_root_path`（行 260）
  - `starlet_cache_dir`（行 992）
  - `spill_local_storage_dir`（行 1100）
  - `datacache_disk_path`（行 1146）
