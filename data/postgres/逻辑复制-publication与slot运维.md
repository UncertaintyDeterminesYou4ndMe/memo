# PostgreSQL 逻辑复制 —— Publication 与 Slot 的关系和防积压运维

## 背景

在用 Flink CDC（postgres-cdc connector）做库到库同步时，绕不开 PostgreSQL 逻辑复制的两个核心对象：**publication** 和 **replication slot**。
一开始容易把它们混为一谈，调试过程中才逐步意识到它们是**正交的两层概念**——理解这个分层是整个排错过程的关键转折点。

本文记录这套心智模型，以及生产环境下防止 slot 积压撑爆源库磁盘的一套防御策略。

---

## 核心判断：Publication 和 Slot 是正交的两层

| 维度 | Publication | Replication Slot |
|---|---|---|
| 作用 | 定义**哪些表 / 哪些操作**要被发布 | 定义订阅者的**消费位点**，保证 WAL 不被提前回收 |
| 类比 | 订阅号（发什么内容） | 订阅者的"书签"（读到哪了） |
| 数量关系 | 1 个 publication 可被 N 个 slot 共用 | 1 个 slot 同时只能被 1 个订阅连接独占 |
| 作用域 | **DB 级**（属于某个 database） | **集群级**（逻辑 slot 绑定到创建时所在的 DB） |
| 系统目录 | `pg_publication` / `pg_publication_tables` | `pg_replication_slots` |
| 副作用 | 无（纯元数据） | **会持续保留 WAL**，不消费就会撑爆磁盘 |

**一句话 mental model**：
> publication 是"过滤器"，slot 是"指针 + 锚点"。
> 没有 publication → pgoutput 不知道该解码哪张表。
> 没有 slot → 订阅者重启后丢 WAL。

---

## 订阅连接的四步流程

当 Flink（或任何逻辑复制订阅者）连上 PG 时，内部发生：

1. 指定要用的 **slot 名**（决定从哪个 LSN 开始读 WAL）
2. 指定要用的 **publication 名**（告诉 pgoutput 只解码这些表的变更）
3. PG 从 slot 记录的 `restart_lsn` 开始重放 WAL，只把 publication 覆盖表的变更流式推给订阅者
4. 订阅者确认消费后，PG 更新 slot 的 `confirmed_flush_lsn`，这之前的 WAL 就可被回收

第 2 步是经常忽视的坑：**Flink 默认会自动创建 `dbz_publication FOR ALL TABLES`**，这个动作需要类似 `rds_superuser` 级别权限，普通 DB 账号会失败。
正确做法是手动 `CREATE PUBLICATION` 覆盖需要的表，然后在 Flink source WITH 里加：
```
'debezium.publication.name'            = '<your_publication>',
'debezium.publication.autocreate.mode' = 'disabled'
```

---

## 在 PG 中如何查看

```sql
-- 查 publication（包含哪些表）
SELECT * FROM pg_publication;
SELECT pubname, schemaname, tablename FROM pg_publication_tables;

-- 查 slot
SELECT slot_name, plugin, database, active,
       restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;

-- 查 slot 占用了多少未回收的 WAL（最关键的监控指标）
SELECT slot_name,
       pg_size_pretty(
         pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)
       ) AS behind_size,
       active
FROM pg_replication_slots;
```

常见异常状态：
- `active = false` 且 `behind_size` 持续涨 → **最危险**，订阅者断开但 WAL 还在堆积
- `wal_status = 'lost'` → slot 已被 `max_slot_wal_keep_size` 作废
- `active = true` 但 `confirmed_flush_lsn` 不动 → 订阅者连着但没 checkpoint 成功

---

## 典型组织方式：一库一 publication，表级 slot

通用的布局范式（以 N 个源库、每库 M 张表为例）：

| Publication（每 DB 一个，列出本库要同步的表） | 对应 Slot |
|---|---|
| 库 A 的 publication | 库 A 的 M 个 slot 共用 |
| 库 B 的 publication | 库 B 的 M 个 slot 共用 |
| 库 C 的 publication | 库 C 的 M 个 slot 共用 |

**设计判断**：
- 为什么每张表一个 slot？Flink CDC 的 source 粒度是每张表，每个 source 独占一个 slot 连接。
- 为什么一个 DB 共用一个 publication？publication 只是过滤器，共用不会相互干扰，且方便统一管理覆盖范围。
- 同一 slot 不能并发连接——如果预览和正式作业同时用同一 slot 会报错 "slot is active"。

---

## 最关键的运维问题：Flink 停机 → Slot 积压 → 源库磁盘打爆

这是生产环境最需要防御的风险链路：

```
Flink 作业挂了 → 没人消费 → confirmed_flush_lsn 不推进
              → WAL 无法回收 → 源库磁盘持续膨胀
              → 磁盘满 → 主库宕机 → 整条业务线瘫痪
```

防御必须做**三层**：监控告警 → 自动兜底 → 紧急处置预案。

### 第一层：监控告警（必做）

```sql
SELECT slot_name,
       active,
       pg_size_pretty(
         pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)
       ) AS behind_size
FROM pg_replication_slots
WHERE slot_name LIKE '<your_prefix>%';
```

告警阈值建议：
- behind_bytes > 10GB → 黄色警告
- behind_bytes > 50GB → 红色告警，立刻介入
- **active = false 且 behind_bytes > 1GB** → 最危险的信号，slot 没人连还在堆积

### 第二层：PG 侧兜底（强烈推荐）

PostgreSQL 13+ 原生参数：

```sql
ALTER SYSTEM SET max_slot_wal_keep_size = '50GB';
SELECT pg_reload_conf();
```

超过 50GB，PG 自动把 slot 标记为 `invalidated`——牺牲该 slot，保住主库磁盘。
**这个参数是"保命"的，不是"解决问题"的**。触发后订阅作业必须重建，但比主库宕机好得多。

RDS / Aurora 的参数名可能是 `rds.logical_replication_slot_wal_keep_size` 类似，需查具体平台。

### 第三层：降低 Flink 停机概率

1. **Restart strategy**：
   ```
   restart-strategy: fixed-delay
   restart-strategy.fixed-delay.attempts: 10
   restart-strategy.fixed-delay.delay: 30 s
   ```
2. **Checkpoint 间隔 ≤ 3 分钟**：Flink 只在 checkpoint 成功时推进 `confirmed_flush_lsn`，间隔太长会让 slot 看起来像没消费。
3. **必加 Debezium heartbeat**：
   ```
   'debezium.heartbeat.interval.ms' = '30000'
   ```
   关键作用：**即使 publication 覆盖的表没有任何 WAL 事件**，心跳也会主动推进 slot 的 flush LSN。对于低流量表（如变化很少的 mapping / 字典表）尤其重要，没有 heartbeat 会造成"假性积压"。
4. **作业告警**：部署失败 / 重启超阈值推送到 IM。

### 紧急处置预案

| 场景 | 处置 |
|---|---|
| Flink 挂了但发现得早 | 重启作业，观察 `confirmed_flush_lsn` 推进 |
| 积压过大追不上 | 评估业务容忍度，要么扩 Flink 资源硬追，要么弃 slot 重建 + 用离线工具补数据缺口 |
| slot 被 max_slot_wal_keep_size 作废 | 无法恢复，删除重建 + 补历史 |
| 磁盘要爆了 | **立即 `pg_drop_replication_slot('xxx')`** 释放 WAL，事后补数据 |

---

## 关键知识点速记

1. **Publication ≠ Slot**，它们是两层正交概念，分别解决"过滤"和"位点"问题。
2. Publication 是 DB 级元数据，slot 是集群级实体但绑定创建时的 DB。
3. Flink CDC 默认会自动建 `dbz_publication FOR ALL TABLES`，生产环境必须手动建 publication + 设 `autocreate.mode = disabled`。
4. Slot 不消费 = 源库磁盘定时炸弹。**max_slot_wal_keep_size + heartbeat + 监控告警**是标准三件套。
5. Debezium heartbeat 不仅防连接超时，更重要的是**即使无数据变更也推进 LSN**，避免低流量 slot 的假性积压。
6. 同一 slot 只能被一个订阅连接独占。预览 / 正式作业并存时，必须用不同 slot 名。
7. 紧急时 `pg_drop_replication_slot` 胜过主库宕机——先保集群，再补数据。

---

## 后续可做

- 把 slot 监控 SQL 接入内部监控平台，设置分级告警
- 用 IaC 管理 publication 和 slot 的创建，避免手工漏建
- 在流式作业模板里默认内置 heartbeat、publication 配置、合理 restart strategy
