# Paimon 分桶键选择与裁剪失效排查

## 背景

一张埋点明细表（无主键 append 表，按天分区，`bucket=16`，`bucket-key='event,distinct_id'`），单分区约 3.75 亿行 / 30 GB / 512 种事件类型。BI 侧一条最普通的查询：

```sql
SELECT ... FROM <埋点明细表>
WHERE pt = '<某天>' AND event = '<某个事件>'
```

耗时 27 秒，扫描整个分区。从「1000 条数据怎么分布到 bucket」这个问题起头，一路查到分桶键配置根本没在查询里起过作用。

这篇记的是排查过程中几个反直觉的判断，以及两次自我纠正。

## 认知纠正一：分桶的分配单位是取值，不是行

最初的问题问的是「N 条数据怎么分布在 bucket 中」，这个问法本身就偏了。

Paimon 的分桶函数（`DefaultBucketFunction`）是：

```java
int hash = row.hashCode();          // row = 只含 bucket-key 列的 BinaryRow
return Math.abs(hash % numBuckets);
```

`row.hashCode()` 是对 bucket-key 列的二进制表示做 MurmurHash（`MurmurHashUtils`，seed=42）。所以：

- **哈希只吃 bucket-key 的值**，同一个值的所有行必然进同一个桶，永远不会被拆开
- 分桶是**分区内**独立的，但同一个值在所有分区里落的桶号相同（哈希不含分区字段）——好处是桶的格局跨天稳定，坏处是热点永远热

因此「数据怎么分布」等价于两个问题：bucket-key 有多少种不同取值，以及每种取值各有多少行。行总数本身不决定分布。

## 根因：复合 bucket-key 让裁剪永不触发

`BucketSelectConverter.java:62-66`：

```java
Set<String> predicateFields = collectFieldNames(predicate);
if (!predicateFields.containsAll(bucketKeyType.getFieldNames())) {
    return Optional.empty();   // 少一个字段就整体放弃裁剪
}
```

**关键点：不是部分裁剪，是完全放弃。** bucket-key 是 `event,distinct_id`，而 BI 查询只给 `event`，于是裁剪率从 1/16 直接掉到 0。

| 查询 | `bucket-key='event'` | `bucket-key='event,distinct_id'` |
|---|---|---|
| `WHERE event='x'` | 读 1/16 | **全读，无裁剪** |
| `WHERE distinct_id='y'` | 全读 | **全读，无裁剪** |
| `WHERE event='x' AND distinct_id='y'` | 读 1/16 | 读 1/16 |

往 bucket-key 里加一个字段，等于把原有字段的裁剪能力整个废掉。这是复合 bucket-key 最容易被忽略的代价。

**还有一个 1000 的天花板**（`BucketSelector.java:60,166`）：裁剪要枚举各字段取值的笛卡尔积，`MAX_VALUES = 1000`，乘积超过就放弃。单字段时 `event IN (...)` 能容 1000 个值；复合键下 `event='x' AND distinct_id IN (2000 个值)` = 2000 组合，直接不裁剪了。复合键把这个预算吃掉了。

另外**字段顺序会改变落桶结果**：实测一万组样本，`(event, distinct_id)` 与 `(distinct_id, event)` 落桶一致率 12.51%，即纯随机。两种写法是两套不同分桶，改顺序等于重新分桶（虽然裁剪语义相同）。

## 反直觉规律：低基数列做 bucket-key，加桶会让倾斜更严重

用真实事件分布（512 种事件、3.75 亿行）算 `bucket-key='event'` 下不同桶数的倾斜：

| 桶数 | 最热桶行数 | 最热/平均 |
|---|---|---|
| 8 | 1.353 亿 | 2.88x |
| 16 | 1.305 亿 | 5.56x |
| 32 | 8018 万 | 6.83x |
| 64 | 8010 万 | 13.66x |
| 128 | 7990 万 | 27.24x |

**看第二列：桶数翻 4 倍，最热桶几乎纹丝不动。**

原因是单个取值是不可分割的原子单位，最热桶的下限就是 top1 取值的行数。加桶只能把小取值摊得更薄，热桶被钉在原地，结果是**相对倾斜急剧恶化**。

推论：低基数列（这里 512 种事件，头部事件占 18%）做 bucket-key 时，桶数存在一个「有效上限」，超过就只有坏处。这张表的有效台阶在 32 左右（把 top2 两个撞在一起的大事件拆开），再往上完全无效。

## 认知纠正二：数据没聚簇时，file index 是空转

中途一度建议加 `file-index.bitmap.columns='event'`，用真实数据一算发现无效，这个纠正很重要。

Paimon 的 file index 在 scan 阶段做的是**整文件保留/丢弃**（`AppendOnlyFileStoreScan.filterByStats` → `testFileIndex`，返回 boolean），只能跳过**完全不含**目标值的文件。算一下：

- 2754 个文件 / 2.88 亿行 = 约 **10.5 万行/文件**
- 目标事件占 1.09% → 每个文件期望含 1140 行该事件
- 某文件一行都不含的概率 ≈ e^(-1140) ≈ **0**

数据按到达时间交错写入，**每个文件里都混着全部 512 种事件**，没有任何文件能被跳过。连排第 100 名的冷门事件（1.9 万行）也只能跳掉 0.5% 的文件。

Profile 里 `PageSkipCounter` 只有 467 也印证了同一件事——Parquet 页级 min/max 同样裁不动。

**前提没解决（数据没有按目标列聚集），任何基于 min/max 或索引的裁剪都是空转。** 要先 clustering，索引才有意义。

## 怎么验证裁剪到底生效没有

`EXPLAIN` 看不出来。它只显示引擎下推了什么谓词，不显示 Paimon 侧最终返回了多少 split。计划里那个 `MIN/MAX PREDICATES: event <= 'x', event >= 'x'` 是 StarRocks 自己的 zone-map 改写，跟 Paimon 的分桶裁剪不是一回事，别当证据。

要看 query profile 的三个数：

| 指标 | 含义 |
|---|---|
| `ScanRanges` | 进入扫描的文件数。等于分区全部文件数 = 没裁剪 |
| `RawRowsRead` | 从存储真正读出的行数 |
| `RowsRead` | 过滤后命中的行数 |

判别式很干净：先算出目标值所在桶占分区的比例，再看 `RawRowsRead` 是否落在那个量级。本例 `RawRowsRead=288M` / `RowsRead=3.13M`，有效率 1.09%，`ScanRanges=2754`（=全部文件），三个数一起把「完全没裁剪」钉死了。

顺带：分桶裁剪是**引擎无关**的，发生在 Paimon core：

```
ReadBuilder.withFilter(pred)
  → SnapshotReaderImpl.withFilter()      (SnapshotReaderImpl.java:264)
  → scan.withCompleteFilter(pred)
  → BucketSelectConverter.convert(pred)  (AppendOnlyFileStoreScan.java:94)
```

只要引擎把等值谓词交给 `ReadBuilder.withFilter`，裁剪就自动发生，不需要查询引擎侧做适配。注意只有 `withCompleteFilter` 触发，单独调 `withFilter` 不会。

## 方法论：别估，用官方 jar 复刻分桶函数

这个手法可复用，成本很低：

```bash
# 只需两个 jar，不用编译整个 paimon
curl -O https://repo1.maven.org/maven2/org/apache/paimon/paimon-common/1.4.2/paimon-common-1.4.2.jar
curl -O https://repo1.maven.org/maven2/org/apache/paimon/paimon-api/1.4.2/paimon-api-1.4.2.jar
```

```java
static int bucket(int numBuckets, String... fields) {
    BinaryRow row = new BinaryRow(fields.length);
    BinaryRowWriter w = new BinaryRowWriter(row);
    w.reset();
    for (int i = 0; i < fields.length; i++) {
        w.writeString(i, BinaryString.fromString(fields[i]));
    }
    w.complete();
    return Math.abs(row.hashCode() % numBuckets);   // 与 DefaultBucketFunction 一致
}
```

配上一条 `SELECT <bucket_key_col>, count(*) ... GROUP BY 1` 的真实分布，就能算出精确落桶、每桶行数、倾斜倍数，以及「查某个值需要扫多少」。比拍脑袋估强太多——本例算出来的倾斜（16 桶 5.56x）比按 Zipf 模拟的预期（1.4–3x）差得多，因为真实数据里 top2 和 top3 两个大事件恰好撞在同一个桶，这种运气问题只能靠真实数据发现。

## 踩到的坑：体积估算错了 4–10 倍

我按埋点行含 JSON、压缩后 200–500 B/行估，实际是 **~54 B/行**（30 GB ÷ 3.75 亿行）。埋点数据列多但基数低，Parquet 字典编码 + zstd 的压缩率能到这个程度。

差 4–10 倍直接让第一版结论过激（说「桶大小超官方建议 20–150 倍」，实际只超 3 倍）。**教训：宽表的压缩率不要按行内容想象，让对方给实测体积。**

顺带记下官方口径：每桶建议 **200 MB – 1 GB**，约「1 bucket per 1GB per partition」。

## write-only=true 的连带影响

表上有 `'write-only' = 'true'`。`CoreOptions.java:757-764` 的说明是：跳过 compaction 和 snapshot expiration，需配合独立 compaction 作业。

这一条同时解释了两件事：

1. **小文件的归因**：2754 个文件 / 30.4 GB = 平均 11 MB（默认 target-file-size 是 128 MB），16 桶下每桶积了约 172 个碎文件。不是「compaction 没配」，而是那个独立作业没跟上或没生效
2. **clustering 方案的落地前提**：`clustering.incremental` 要靠 compaction 才能生效。写入侧已经把 compaction 交出去了，所以改完配置能不能真的重排数据，取决于托管 compaction 作业是否支持聚簇

第 2 点是差点漏掉的隐藏依赖——方案本身没错，但执行路径被 `write-only` 切断了。

另外 `num-sorted-run.stop-trigger` 和 `sort-spill-threshold` 在这张无主键表上是空转（只在 `MergeTreeCompactManagerFactory` / `MergeSorter` 的主键 LSM 路径消费），大概率是从主键表建表语句抄来的。但一旦开启 clustering，`sort-spill-threshold` 就会开始生效（`ClusteringCompactManagerFactory:121`）。

## 结论与方案取舍

两个方案，最后推荐先做成本低的那个：

| | 方案 A：`bucket-key` 改成单字段 | 方案 B：保留 bucket-key + 按目标列聚簇 |
|---|---|---|
| 裁剪粒度 | 桶级，平均少读 4.5–16.3 倍 | 文件级，理论上限更高 |
| 倾斜 | 引入 5.56x | 无 |
| 重建表 | 必须，且要重写全部历史分区 | 不用，一次 ALTER |
| 回滚 | 难 | 容易 |
| 收益确定性 | 已测算 | 未实测 |

```sql
-- 方案 B
ALTER TABLE <表> SET (
  'bucket-append-ordered'  = 'false',   -- 前置条件：分桶表开聚簇要关掉桶内有序
  'clustering.incremental' = 'true',
  'clustering.columns'     = '<目标列>'
);
```

取舍逻辑：**方案 B 的成本低到可以直接当验证手段**。先花一条 ALTER 拿到实测收益，再决定要不要付出方案 A 的重建代价，比反过来风险小得多。`bucket-key` 是不可变更的 schema 属性，改它必须重建表并重写历史数据，这个不可逆性本身就该让它排在后面。

留在文档里没关闭的两个未知：下游有没有依赖桶内有序的流式消费（决定 B 能不能做），托管 compaction 作业是否支持 clustering（决定 B 做了有没有用）。

## 关键知识点

- 分桶的分配单位是 bucket-key 的**取值**，不是行；同一取值永不跨桶，跨分区桶号也相同
- 复合 bucket-key 要求谓词**覆盖全部字段**，否则裁剪归零；`MAX_VALUES=1000` 的笛卡尔积上限也被它吃掉
- bucket-key 字段顺序改变落桶结果，两种顺序是两套分桶
- 低基数列做 bucket-key 时，最热桶被 top1 取值钉死，**加桶只恶化相对倾斜**
- file index / min-max 裁剪的前提是数据已按目标列聚集；交错写入时每个文件都含全部取值，一个都跳不掉
- 验证裁剪只能看 profile 的 `ScanRanges` / `RawRowsRead` / `RowsRead`，`EXPLAIN` 看不出来
- 分桶裁剪在 Paimon core 完成，引擎无关，但只走 `withCompleteFilter` 路径
- `write-only=true` 跳过 compaction 和 snapshot 过期，任何依赖 compaction 的方案（clustering）都要先确认独立作业是否在跑
- 每桶建议 200 MB – 1 GB；宽表压缩率别靠想象，要实测体积
