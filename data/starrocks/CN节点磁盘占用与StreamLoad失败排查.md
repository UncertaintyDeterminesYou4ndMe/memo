# StarRocks CN 节点磁盘占用与 StreamLoad 失败排查

## 背景

云监控告警：EMR Serverless StarRocks 一个 CN 节点 `disk1` 利用率 85.35%，持续近 2 天，导致 StreamLoad 写入失败。

入口问题是「BE 磁盘存了哪些数据」，但本质上要先分清当前是**存算一体的 BE**还是**存算分离的 CN** —— 两者磁盘画像差很多。podName 前缀 `cn-` 就是判断依据。

## 核心判断：先分清 BE 还是 CN

| 节点类型 | 磁盘角色 | 主要占用 |
|---|---|---|
| 经典 BE（存算一体） | 数据真源 | `/data/<shard>/<tablet>/` 下的 rowset/segment 全量 |
| CN（存算分离 / shared-data） | **本地缓存盘**，真源在对象存储 | starlet_cache、datacache 占大头 |

判错对象会导致排查方向完全跑偏。CN 上把 starlet_cache 当数据真源去清理，会反复触发回源；BE 上去找 starlet_cache 目录则根本不存在。

## CN 节点磁盘上的目录全貌

按 CN 上典型占比从大到小：

| 目录 | 内容 | 配置项 |
|---|---|---|
| `starlet_cache/` | 湖表本地数据缓存（segment、索引副本），CN 上最大头 | `starlet_cache_dir` |
| `datacache/` | block cache，加速查询的数据块缓存 | `datacache_disk_path` |
| `spill/` | 查询执行内存不足时的落盘临时文件 | `spill_local_storage_dir` |
| `/data/<shard>/<tablet>/` | 临时 rowset/segment（写入未上传到对象存储前） | `DATA_PREFIX` (`be/src/storage/olap_define.h:63`) |
| `/trash/` | 已删除 tablet 的回收站 | `TRASH_PREFIX`，`trash_file_expire_time_sec` 默认 86400s |
| `/snapshot/` | 快照、备份/克隆中转 | `SNAPSHOT_PREFIX` |
| `/persistent/` | 主键索引持久化文件（PK 表） | `PERSISTENT_INDEX_PREFIX` |
| `/tmp/`、`/clone/`、`/replication/` | 导入/克隆/复制临时区 | 同上头文件 |
| `error_log/`、`rejected_record/` | StreamLoad 错误行日志、拒收记录 | `ERROR_LOG_PREFIX` |

所有 prefix 字符串统一定义在 `be/src/storage/olap_define.h:60-73`。

## StreamLoad 写入失败的常见根因

按发生概率排（CN 节点情境）：

1. **starlet_cache 占满** —— CN 上几乎都是它。先确认是否设置了上限（`starlet_cache_evict_high_water` / `starlet_star_cache_disk_size_percent`），默认值很激进，常常会把整盘吃掉。
2. **trash 没清** —— 删表/分区或大量 compaction 后，`/trash` 累积大文件，等 `trash_file_expire_time_sec`（默认 24h）到期才删。**这个默认值常常是磁盘缓慢爬高的隐性元凶**。
3. **spill 残留** —— 异常 query 失败后 spill 文件没回收。
4. **datacache 配置过大**或与 starlet_cache 路径重叠。
5. **写入侧瓶颈** —— 持续写入但本地未及时上传到对象存储，rowset 堆在 `/data` 下。

## 排查命令（CN pod 内）

```bash
du -h --max-depth=1 /opt/starrocks/be/storage/disk1/ | sort -h
du -h --max-depth=2 /opt/starrocks/be/storage/disk1/starlet_cache/ 2>/dev/null
ls -la /opt/starrocks/be/storage/disk1/trash/ | head
ls -la /opt/starrocks/be/storage/disk1/spill/ 2>/dev/null
df -h /opt/starrocks/be/storage/disk1
```

## 紧急缓解

- 调小 starlet_cache 上限触发 LRU 淘汰，或安全清理已过期的 `/trash/*`（**绝不动** `/data`、`/starlet_cache` 文件本身）
- `ADMIN SET FRONTEND CONFIG ("trash_file_expire_time_sec" = "3600")` 让 trash 提前清
- 阿里云 EMR Serverless StarRocks 控制台一般有"扩容存储"或"调高 cache 上限"入口；长期方案是扩盘或调低 cache 比例

## 关键认知

- **CN 上磁盘是缓存，不是数据**。打满 ≠ 数据丢失风险，但会立刻阻塞写入，因为 `/data`、`/persistent`、`spill` 都共享同一块盘。
- **报错信息能直接指向目录**：`No space left` → 整盘满；`failed to write rowset` → `/data`；`tablet not found` 配合磁盘高 → starlet_cache 异常；`spill failed` → spill 目录。先看错误码再看 du，比闷头清磁盘高效。
- **trash 24h 默认值是个坑**，删大表后磁盘曲线要等一天才回落，监控告警很容易在这个窗口期触发。

## 参考代码位置

- 目录前缀定义：`be/src/storage/olap_define.h:60-73`
- 相关配置项：`be/src/common/config.h`
  - `storage_root_path`（行 260）
  - `starlet_cache_dir`（行 992）
  - `spill_local_storage_dir`（行 1100）
  - `datacache_disk_path`（行 1146）
