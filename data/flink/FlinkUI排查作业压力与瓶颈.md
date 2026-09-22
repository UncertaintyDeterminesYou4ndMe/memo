# Flink UI 排查作业压力与瓶颈

## 背景

云上托管 Flink（Flink 1.20 / VVR 11.7，Serverless，走 `flink-<region>.data.alibabacloud.com/flink-ui/...` 这个原生 Web Dashboard 入口），一条 Kafka → StarRocks 的 CDC Pipeline 作业。

需求不是排故障，是建立一套「打开 UI 该盯哪几个数」的判读路径。结论那一栏最后是「完全没有压力」，所以这篇记的是**方法**和**健康态的基线读数长什么样** —— 基线本身有价值，下次不正常时才知道哪个数偏了。

## 关键手法：不点 UI，直接打 REST

Flink Dashboard 是个 SPA，UI 上的每个数字背后都是一次 REST 调用。与其一页页点、截图、肉眼读，不如：

1. 用浏览器工具的 network 抓一下页面自己发的请求，拿到 REST base path
2. 之后所有数据用 `fetch` 在页面上下文里取（cookie 自动带上，绕过鉴权问题）

这次的 base 是 `.../jobs/<job-uuid>/`，这个路径映射到 Flink REST 根，后面直接拼标准端点：

```
GET  {base}/jobs/overview                                  # 作业列表，拿真实 jid
GET  {base}/jobs/{jid}                                     # 拓扑 + 每个 vertex 的累计指标
GET  {base}/jobs/{jid}/vertices/{vid}/backpressure         # 反压采样
GET  {base}/jobs/{jid}/vertices/{vid}/metrics?get=a,b,c    # 任意指标批量取值
GET  {base}/jobs/{jid}/checkpoints
GET  {base}/jobs/{jid}/exceptions?maxExceptions=3
GET  {base}/taskmanagers  /  {base}/taskmanagers/{tmid}/metrics?get=...
GET  {base}/jobmanager/config
```

**一个坑**：URL 路径里的 job UUID 是带横杠的（`xxxxxxxx-xxxx-xxxx-...`），但 REST 的 `jobid` 参数要**无横杠**形式。直接拿 URL 里那个去请求会 400 `Cannot resolve path parameter (jobid)`。先打 `/jobs/overview` 拿 `jid` 最稳。

`vertices/{vid}/metrics` 不带 `get=` 时返回的是**指标名清单**（这个作业有 204 个），挑出想要的再用 `get=` 批量取值。这是 UI 上 Metrics tab 手动勾选的程序化版本。

## 判读路径

### 1. BackPressure —— 定位瓶颈算子

每个 subtask 给三个比例：`busyRatio` / `backpressureLevel(ratio)` / `idleRatio`，三者互补凑满 1。

判读规则（这是整篇最该记住的一条）：

> 从 source 往下游走，**第一个「busy 高、backpressured 低」的算子就是瓶颈**。它上游的所有算子会呈现 backpressured 高。

因为反压是从堵点往上游传导的，堵点自己不被反压，它只是忙。往下游继续找只会找到一堆 idle。

三种组合的含义：

| busy | backpressured | idle | 含义 |
|---|---|---|---|
| 高 | 低 | 低 | **瓶颈在这里**，算不过来 |
| 低 | 高 | 低 | 被下游堵住，不是它的问题 |
| 低 | 低 | 高 | 在等数据，链路整体空闲 |

本次基线：三个算子 busy 2.3% / 0.1% / 0.6%，backpressured 全 0，idle 97%+。

### 2. Source 的 `pendingRecords` —— 反压为 0 不等于健康

反压只描述 **Flink 内部**的传导。如果 source 自己就是瓶颈（Kafka 分区数不够、并行度不够、反序列化慢），内部一路畅通，反压全绿，但外部积压在涨。

所以第 1 步之后必须补一眼 source 的：

- `pendingRecords` —— 未消费的积压条数
- `currentEmitEventTimeLag` —— 事件时间落后多少
- `numRecordsInPerSecond` / `numRecordsOutPerSecond`

本次基线：`pendingRecords=0`，`currentEmitEventTimeLag=14ms`，输入 6 条/s。

### 3. Checkpoint —— 反压的第二个指纹

反压会先在 checkpoint 上留下痕迹：对齐时间（alignment）飙升、端到端时长变长、进而超时失败。state size 单调增长则是另一类问题（state 没设 TTL / key 无界）。

本次基线：10/10 成功，端到端 432ms，alignment 0，state 20KB。

注意配置：`execution.checkpointing.interval=60s` 但 `min-pause=60s`，**实际间隔是 120s**，不是 60s。只看 interval 会误判 checkpoint 频率。

另外 `restored=2` 而 exception history 为空 —— 说明是人为重启或从 savepoint 恢复，不是故障重启。这两个数要对照着看。

### 4. Data Skew / SubTasks —— 查倾斜

并行度 > 1 时看各 subtask 的 Records Received 是否差一个量级。倾斜的特征是「一两个 subtask busy 100%，其余 idle」，这时**加并行度无效**，要改 key 或加随机前缀。并行度 1 的作业这一步跳过。

### 5. TaskManager metrics —— 排除资源问题

`Status.JVM.CPU.Load` / `Heap.Used` vs `heapMax` / GC count+time / `Status.Shuffle.Netty.UsedMemory` / direct memory。

典型误判：busy 不高但吞吐上不去 —— 常见原因是 Full GC 频繁或 direct memory 不足，而不是算子逻辑慢。

本次基线：CPU Load 5.9%，heap 262MB/688MB，GC 累计 848ms（10 分钟内）。

### 6. FlameGraph —— 落到方法级（默认关闭）

点开 FlameGraph tab 得到的是：

```
The flame graph feature is currently disabled (enable it by setting rest.flamegraph.enabled: true)
```

Flink 默认关闭，要在作业运行参数里显式打开、重启生效。值得预先开着：

- **On-CPU**：某算子 busy 高时，直接定位到哪个方法吃 CPU
- **Off-CPU**：busy 不高但就是慢的场景 —— 等锁、等 IO、等下游 sink 响应。这类问题前五步全是绿的，只有 Off-CPU 能看见

## 一个边界：原生 UI 只有瞬时值

Flink Dashboard 给的是「此刻」的快照，没有历史曲线。判断趋势（延迟是不是在爬、checkpoint 是不是在变慢、昨天到今天变了多少）必须去云厂商控制台的监控页。

分工：**Flink UI 定位是哪个算子，监控平台看它随时间怎么变**。把两者混着用会两头落空 —— 在 UI 上刷新盯着看趋势，或在监控大盘上找不到算子粒度。

## 顺带的观察

数据在 Transform 层收敛得很厉害：source 读入 128MB / 5064 条 → 346 条 → 68 条 → sink。

如果这是过滤造成的，把过滤前移能省掉大量无效反序列化。但 CDC Pipeline 也可能只是按表路由（一个 topic 里混着多张表的变更），那就不是浪费。**看数字不足以下结论，要看 YAML 定义** —— 记下来提醒自己别把「读入远大于写出」直接当成优化点。
