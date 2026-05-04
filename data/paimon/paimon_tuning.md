# Paimon 性能优化指南

## 主键表优化

### 写入作业优化
Paimon 写入作业的瓶颈通常由小文件合并引起。默认情况下，Flink 每次做检查点时，若分桶中小文件数量过多或使用了 lookup 变更数据产生机制，需等待小文件合并完成，可能导致反压。

#### 优化措施：
1. **调整 Paimon Sink 并发**  
   通过 SQL Hints 设置 `sink.parallelism` 参数调整并发数，注意并发数调整可能引起资源使用变化。

2. **调整 Flink 检查点配置**  
   - 增大检查点间隔：增加 `execution.checkpointing.interval` 作业参数值（需注意业务延迟容忍度）。  
   - 允许并发检查点：设置 `execution.checkpointing.max-concurrent-checkpoints: 3`，减少检查点长尾影响。

3. **将小文件合并改为完全异步**  
   通过 ALTER TABLE 语句或 SQL Hints 设置以下表参数：
   ```sql
   'num-sorted-run.stop-trigger' = '2147483647',
   'sort-spill-threshold' = '10',
   'changelog-producer.lookup-wait' = 'false'
   ```
   **参数说明：**
   | 参数                     | 类型   | 默认值 | 说明                                                                 |
   |--------------------------|--------|--------|----------------------------------------------------------------------|
   | num-sorted-run.stop-trigger | Integer | 5      | 分桶小文件数超过该值时强制停止写入并合并，设为大值可实现异步合并。  |
   | sort-spill-threshold     | Integer | 无     | 小文件数超过该值时合并流程由内存归并改为外部排序，减少堆内存消耗。  |
   | changelog-producer.lookup-wait | Boolean | true   | 检查点操作时是否等待 lookup 触发的合并，设为 false 可提高 CPU 利用率。 |

4. **更改文件格式**  
   若无需 OLAP 查询，仅需批式或流式消费，可配置以下参数：
   ```sql
   'file.format' = 'avro',
   'metadata.stats-mode' = 'none'
   ```
   **说明：该配置仅在新建表时有效，无法修改已创建表的格式。**

5. **限制本地临时文件大小**  
   通过 SQL Hints 设置 `write-buffer-spill.max-disk-size` 参数，避免磁盘空间不足。

### 消费作业优化
1. **调整 Paimon Source 并发**  
   通过 SQL Hints 设置 `scan.parallelism` 参数调整并发数。

2. **从 Read Optimized 系统表消费**  
   小文件过多会增加内存归并代价，可在写入作业中调整异步合并参数，或消费 [Read-optimized 系统表] 以提高效率。

3. **限制 Lookup 缓存大小**  
   通过 SQL Hints 设置 `lookup.cache-max-disk-size` 和 `lookup.cache-file-retention` 参数，避免磁盘空间不足。

## Append Scalable 表常用优化

### 写入作业优化
Append Scalable 表写入瓶颈通常由并发限制或存储带宽引起，优先检查带宽是否不足，若资源充裕可尝试以下优化：

1. **调整 Paimon Sink 并发**  
   通过 SQL Hints 设置 `sink.parallelism` 参数调整并发数，注意与上游节点并发数差异化以避免数据倾斜。

2. **检查数据倾斜**  
   若数据倾斜导致节点资源利用率低，可设置 `sink.parallelism` 与上游节点不同，使数据重分布。

3. **限制本地临时文件大小**  
   通过 SQL Hints 设置 `write-buffer-spill.max-disk-size` 参数，避免磁盘空间不足。

### 消费作业优化
1. **调整 Paimon Source 并发**  
   通过 SQL Hints 设置 `scan.parallelism` 参数调整并发数。

2. **使用 zorder/hilbert/order 对数据排序**  
   数据排序可提升批作业和 OLAP 查询效率，需在批模式下配置相关参数。示例如下：
   ```sql
   compact
   --warehouse 'oss://your-bucket/data-warehouse'
   --database 'your_database'
   --table 'your_table'
   --order_strategy 'zorder'
   --order_by 'date,type'
   --partition 'dt=20240311,hh=08;dt=20240312,hh=09'
   --catalog_conf 'fs.oss.endpoint=oss-cn-hangzhou-internal.aliyuncs.com'
   --table_conf 'write-buffer-size=256 MB'
   ```
   **参数说明：**
   | 配置项         | 说明                                                                 |
   |----------------|----------------------------------------------------------------------|
   | order_strategy | 排序方式（zorder 适用于 <5 个过滤字段，hilbert 适用于 ≥5 个，order 适用于等值条件）。 |
   | order_by       | 排序字段，用英文逗号分隔。                                            |
   | partition      | 需要排序的分区，非分区表可删除该参数。                                |
