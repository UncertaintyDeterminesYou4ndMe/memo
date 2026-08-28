# Spark UI 排查 Executor 容器 OOM

## 背景

云上托管的 Serverless Spark（Spark 3.5.2，跑在 K8s 上），一条每天定时跑的 SQL：从一张 Paimon 埋点事件表读一天分区，用 30 多个 `GET_JSON_OBJECT` 把 `properties` 里的字段摊平，`INSERT OVERWRITE` 写进一张下游明细宽表。

跑了很久都正常，某天开始失败。前一天同一条 SQL 是成功的，SQL 本身没改过。

只有 Spark History UI 可以看，日志和 REST API 都拿不到。这篇记的是在这个限制下怎么从 UI 里把结论抠出来，以及每一页该盯哪几个数。

## 第一个分叉点

从 `/jobs/` 页拿到的顶层错误串：

```
Total Uptime: 9.8 min   Failed Jobs: 1
Job 0  Stages 0/1 (1 failed)   Tasks 154/190 (20 failed) (22 killed)

Job aborted due to stage failure: Task 135 in stage 0.0 failed 4 times,
most recent failure: Lost task 135.3 (TID 186) (<executor_ip> executor 4):
ExecutorLostFailure
  exit code: 137 (SIGKILL, possible container OOM)
  termination reason: OOMKilled
  container started at:  ...T19:12:56Z
  container finished at: ...T19:22:26Z
```

`OOMKilled` / `exit 137` 和 `java.lang.OutOfMemoryError` 是两种完全不同的故障，看到哪个决定了后面所有步骤往哪走。

| 错误 | 谁杀的 | 内存在哪超 | 往哪查 |
|---|---|---|---|
| `exit 137` / `OOMKilled` | K8s cgroup 从外面 SIGKILL | 容器 RSS 超 memory limit | `memoryOverhead`、堆外、native lib、javaagent |
| `java.lang.OutOfMemoryError` | JVM 自己抛 | 堆内 | `executor.memory`、数据倾斜、spill、缓存 |

拿到 `exit 137` 之后，`executor.memory` 就基本可以先放一边了 —— JVM 自己都没察觉出事，说明压力不在堆内。

## 排查路径

### Step 1 · `/jobs/`

看两个数：**Failed Jobs 计数**，以及失败 job 行末尾那串聚合错误。

这串错误是整次排查里唯一的错误文本来源（原因见后面「走不通的路」），要完整读，尤其是 `exit code` 和 `termination reason`。

`154/190 (20 failed) (22 killed)` 这个比例也有信息量：失败发生在跑到八成的时候，不是一起步就崩。

### Step 2 · `/stages/stage/?id=0` 排除倾斜

看 **Summary Metrics for Completed Tasks** 这张表的 min → max 分布。

| Metric | Min | Median | Max |
|---|---|---|---|
| Duration | 0.4 s | 1.3 min | 2.5 min |
| GC Time | 0 ms | 0.8 s | **2 s** |
| Peak Execution Memory | 64.1 MiB | 400 MiB | **464 MiB** |
| Input Size / Records | 807 B / 15 | 142 KiB / 2,376,000 | 457.8 KiB / **2,837,000** |

从这三行读出三个判断。

- max/median 的记录数只有 1.19 倍，没有倾斜
- **GC Time 最大 2 秒**，堆内完全没在挣扎，反过来印证不是 heap OOM
- Peak Execution Memory 封顶 464 MiB，乘以单 executor 的 2 个并发槽约 0.9 GiB，6 G 堆装得下

再往下看 **Aggregated Metrics by Executor** 的 Failed Tasks 列。这次的失败均匀散在 10 个 executor 上，每个恰好 2 个 —— 正好等于「容器被杀时手上挂着的 task 数」。均匀分布说明是系统性内存不足，某一个 executor 独自暴毙才是倾斜或坏节点。

### Step 3 · `/executors/` 看死亡时间线

三列有用：

| 列 | 怎么用 |
|---|---|
| **Storage Memory** `140.3 KiB / 3.3 GiB` | 反推堆大小。Spark 的 storage pool ≈ `(heap - 300MB) × spark.memory.fraction`，`(6144-300)×0.6 ≈ 3.5 G`，确认 executor heap 就是 6 G |
| **Cores** | 单 executor 的 task 并发数，用来算 Step 2 里的峰值内存要乘几 |
| **Add Time / Remove Time** | 死亡时间线 |

时间线读出来是：12 个 executor 在 03:13 齐起，一路正常到 03:18，然后 **03:18:32 到 03:22:27 这四分钟里成片死了 10 个**。

跑满五分钟才开始死，而且是成片死，指向堆外缓冲随读入数据稳步累积、同时越线。启动即崩通常是 limit 设得太小或镜像问题，某个 task 瞬间打爆则是倾斜。

### Step 4 · `/SQL/execution/?id=0` 拿真实数据量

Jobs 和 Stages 页对 Paimon 表报的 `Input Size` 是假的 —— 这次显示 27.6 MiB，实际读了 28.5 GiB。Paimon 的 `DataSourceRDD` 不上报真实字节数，只有 SQL 页的 BatchScan 节点里有：

```
BatchScan <埋点事件表>
  size of splits read      : 28.5 GiB      ← 真实读入量
  number of splits read    : 175
  avg size of splits read  : 153.9 MiB
  number of output rows    : 256,398,655
Filter → 43,796,834
Sort
  peak memory  total 39.9 GiB (med 196 MiB, max 464 MiB)
  spill size   0.0 B                       ← 没 spill，堆内够用
Execute InsertIntoHadoopFsRelationCommand
```

`spill size = 0` 是又一个「堆内没问题」的证据。堆内真不够时会先 spill 到盘，spill 量非零才轮到怀疑 heap。

`avg size of splits read = 153.9 MiB` 这个数直接决定单 task 要解压多少列式数据，是估堆外用量的起点。

### Step 5 · `/environment/` 和成功的那次对比

这一步才定案。同时看两处：上面的 **Spark Properties** 表，和页面底部的 **Resource Profile**。

| | 失败那次 | 前一天成功那次 |
|---|---|---|
| executor.memory / cores / instances | 6144m / 2 / 12 | 6144m / 2 / 12 |
| dynamicAllocation | false | false |
| container image | 同一个 | 同一个 |
| **executor.memoryOverhead** | **整行不存在 → 默认 max(384m, 0.1×6144) = 614m** | **4GB** |
| **driver.memoryOverhead** | **整行不存在** | **4GB** |
| memory.offHeap.size | 无 | 4GB（`offHeap.enabled` 没开，空转） |
| **容器 memory limit** | **≈ 6.6 GiB** | **= 10 GiB** |

Resource Profile 是最硬的证据，它直接把生效的资源画像打出来：

```
失败：  Executor Reqs:  cores: 2   memory: 6144   offHeap: 0
成功：  Executor Reqs:  cores: 2   memory: 6144   offHeap: 0   memoryOverhead: 4096
```

失败那次连 `memoryOverhead` 这一行都没打印出来。

顺带一个坑：`spark.memory.offHeap.size=4GB` 在成功那次也是空转的，因为没配 `spark.memory.offHeap.enabled=true`。Resource Profile 里 `offHeap: 0` 就是它没生效的证据。别看 Properties 表里有这条就以为它在起作用。

### Step 6 · 交叉验证数据量，排掉「数据涨了」

| | 失败 | 成功 |
|---|---|---|
| size of splits read | 28.5 GiB | 28.3 GiB |
| splits / avg size | 175 / 153.9 MiB | 176 / 164.9 MiB |
| 扫描行数 | 256,398,655 | **270,319,172** |
| 输出行数 | 43,796,834（未跑完） | **47,420,402** |
| Sort peak / task | med 196 MiB, **max 464 MiB** | med 400 MiB, **max 464 MiB** |
| Duration | 9.3 min 后 abort | 10 min 成功 |

成功那次数据更多，单 task 峰值一模一样。变量锁死在 `memoryOverhead` 一条上。

## 为什么 614 MB 堆外不够

这个 executor 的堆外常驻比想象中重：

- JDK 17，加两个厂商注入的 javaagent（Prometheus JMX exporter + 诊断 agent）
- JindoFS native 库读对象存储上的 28 GiB
- Paimon 列式读，平均一个 split 154 MiB，ORC/Parquet 的原生解压缓冲 + netty direct buffer
- `processTreeMetrics`、`nioDiskMetrics`、`jindoMetrics` 全开着

这些全部要塞进 614 MB。RSS 一超 cgroup limit 就是 SIGKILL，JVM 没有任何机会抛异常，所以看到的只有 `exit 137`。

## 两条走不通的路

**executor stderr 日志页是空的。** 打开 `/logs/.../executor/4/stderr` 只有标题栏。pod 被 SIGKILL 之后日志没落盘，OOMKilled 场景下这条路基本永远是空的。指望从 executor 日志里找 OOM 现场是白费力气。

**REST API 被网关拦了。** `/api/v1/applications` 返回 `403 url need add white list`。云厂商的这套 token 网关只放行 UI 页面路径。连带的后果是 `/stages/stage/` 的 task 明细表分页也依赖被拦的 API，只能拿到首屏 20 行，**看不到 failed task 的逐条错误** —— 所以 Step 1 里 job 级别那串聚合错误是唯一的错误文本。

网关只放行 UI 的情况下，用浏览器自动化直接读 DOM 里的表最省事：

```js
document.querySelector("#summary-metrics-table").innerText
document.querySelector("#active-executors-table").innerText
// Summary Metrics 默认只显示 4 行，先把 additional metrics 的 checkbox 全勾上
document.querySelectorAll("input[type=checkbox]").forEach(c => { if (!c.checked) c.click() })
```

## 判断树

```
Spark on K8s 任务失败
└─ /jobs/ 读聚合错误串
   ├─ exit 137 / OOMKilled ──────────► 容器级 OOM，查堆外
   │   ├─ /stages/  GC Time 低吗？          低 → 堆内没问题
   │   ├─ /stages/  spill size 为 0 吗？    是 → 堆内没问题
   │   ├─ /stages/  Failed Tasks 均匀吗？   均匀 → 系统性，不是倾斜
   │   ├─ /executors/ Add/Remove Time       跑一段才死 → 堆外累积
   │   └─ /environment/ Resource Profile 里有没有 memoryOverhead   ★决定性
   │
   ├─ java.lang.OutOfMemoryError ────► 堆内 OOM
   │   └─ /stages/ Peak Execution Memory、spill size、
   │              Input Records 的 max/median 比值
   │
   └─ FetchFailed / Shuffle ─────────► 查 shuffle service / remote shuffle
```

## 三个反直觉的读数方式

**容器 OOM 时，GC Time 越低越说明是 overhead 的问题。** 堆内舒服恰恰说明压力全在堆外。按堆内 OOM 的套路去加 `executor.memory` 会让情况更糟 —— 容器 limit 里堆占得更多，留给堆外的更少。

**Paimon 表在 Jobs/Stages 页的 `Input Size` 不可信。** 这次差了三个数量级（27.6 MiB vs 28.5 GiB）。真实读入量只认 SQL 页 BatchScan 节点的 `size of splits read`。

**Resource Profile 比 Properties 表更可信。** Properties 表是提交上来的配置，Resource Profile 是最终生效的资源画像。`offHeap.size` 配了但 `offHeap.enabled` 没开这种情况，只有在 Resource Profile 里才看得出来。

## 结论与动作

失败的原因是 `spark.executor.memoryOverhead` 在那次提交里整组丢了（`executor` / `driver` / `offHeap.size` 三条一起消失），容器堆外空间从 4 GB 掉回默认 614 MB。

改回来：

```
spark.executor.memoryOverhead=4GB
spark.driver.memoryOverhead=4GB
```

三条配置是一起消失的，指向上游的配置模板或资源组没下发，值得往回追一层。手删单行不会这么整齐。

以后每次跑批前，去 Environment 页搜一下 `memoryOverhead`，在不在一眼可判。
