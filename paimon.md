# 前文
> [表模型](https://help.aliyun.com/zh/flink/use-cases/basic-concepts?spm=a2c4g.11186623.0.0.167a719ekfAuH4#64abb97f8fnfe)
# 主键表优化
写入作业优化
Paimon写入作业的瓶颈通常由小文件合并引起。默认情况下，Flink每次做检查点时，如果分桶中小文件数量过多或使用了lookup变更数据产生机制，则需要等待当前的Paimon小文件合并过程结束。如果等待时间过长，或部分并发的检查点出现了长尾，会造成反压，影响作业效率。

您可以从以下角度进行优化：

调整Paimon Sink并发

通过SQL Hints设置sink.parallelism参数，以调整Paimon Sink的并发数。需要注意调整并发数可能会引起资源使用方面的变化。

调整Flink检查点间隔

检查点间隔将影响Paimon的时效性。增大检查点间隔时，需要注意业务能否接受更长的延迟。

增大Flink检查点间隔，即增加execution.checkpointing.interval作业参数的值，配置方法请参见如何配置作业运行参数？。

设置作业参数execution.checkpointing.max-concurrent-checkpoints: 3，其将允许至多3个检查点同时进行，主要用于减小部分并发检查点长尾的影响。

考虑执行批作业。

将小文件合并改为完全异步

将小文件合并完全异步化之后，Flink做检查点时无需等待小文件合并完成。

通过ALTER TBALE语句或SQL Hints设置以下表参数：

 
'num-sorted-run.stop-trigger' = '2147483647',
'sort-spill-threshold' = '10',
'changelog-producer.lookup-wait' = 'false'
参数的含义如下：

|参数 | 数据类型 | 默认值 | 说明|
| -- | -- | -- | -- |
| num-sorted-run.stop-trigger | Integer | 5 | 当一个分桶内的小文件数量超过该参数时，将强制停止该分桶的写入，直到小文件合并完成。这一参数主要是防止小文件合并流程太慢，导致的小文件数量无限增长，影响Paimon表的批式消费和即席（OLAP）查询的效率。小文件数量对下游流式消费的影响不大。将该参数设为一个很大的值后，小文件合并将不再强制停止分桶数据的写入，而是完全异步进行。如果您需要监控分桶中小文件的数量，可以查询 [Files系统表](https://help.aliyun.com/zh/flink/use-cases/paimon-system-table?spm=a2c4g.11186623.0.0.4fd59c454Jruv5#89c4fd9625xl4)。|
| sort-spill-threshold | Integer | 无 | 默认情况下，小文件合并会在内存中进行归并排序，每个小文件的reader都需要占用一定的堆内存。如果小文件数量过多，可能会造成堆内存不足。设置该参数后，若小文件数量超过该参数，小文件合并流程将由归并排序改为外部排序，减小堆内存的消耗。 |
| changelog-producer.lookup-wait | Boolean | true | Flink在执行检查点操作时是否等待lookup变更数据产生机制触发的小文件合并。参数取值如下：<br>true：等待。您可以通过Checkpoint创建的速度觉察到作业的延迟情况，从而决定是增加资源还是采用异步化处理方式。<br>false：不等待，允许已完成小文件合并的并发继续处理后续数据。提高了CPU的利用率，并对变更数据的产出无太大影响。但开启异步后，Checkpoint创建速度较快，因此无法准确判断当前数据处理的延迟情况。 |

更改文件格式

如果您不需要对Paimon表进行即席（OLAP）查询，只需进行批式或流式消费，可以选择配置以下表参数，将数据文件格式改为avro，并关闭采集统计数据，以进一步提高写入作业的效率。

 
'file.format' = 'avro',
'metadata.stats-mode' = 'none'
说明
该配置只能在新建表时指定，无法更改已经创建表的数据文件格式。

消费作业优化
调整Paimon Source并发

通过SQL Hints设置scan.parallelism参数调整Paimon Source的并发数。

从Read Optimized系统表消费数据

在批作业消费、流作业消费的全量阶段，以及即席（OLAP）查询中，Paimon源表的瓶颈主要由小文件引起。Paimon源表需要将小文件中的数据在内存中归并才能产出结果，因此小文件数量过多会增加归并过程的代价。然而写入作业过于频繁的小文件合并也会降低写入作业的效率，因此您需要在写入效率与消费效率之间做出平衡。

您可以在对应的写入作业中调整将小文件合并改为完全异步中提及的参数。如果您不需要消费最新的数据，也可以选择消费Read-optimized系统表以提高消费效率。

Append Scalable表常用优化
写入作业优化
通常情况下，Paimon Append Scalable表写入作业的瓶颈由并发限制以及文件系统或对象存储的带宽限制所决定。当遇见写入作业瓶颈时，应首先查看对应文件系统写入带宽是否到达了瓶颈，若带宽资源充裕，可尝试从以下几个方面对Append Scalable表进行优化。

调整Paimon Sink并发

通过[SQL Hints](https://help.aliyun.com/zh/flink/user-guide/manage-apache-paimon-catalogs?spm=a2c4g.11186623.0.0.4fd59c45MSrzsH#c4d92b2f0fl6k)设置sink.parallelism参数，以调整Paimon Sink的并发数。需要注意调整并发数可能会引起资源使用方面的变化。

检查数据是否倾斜

Paimon Append Scalable表在写入节点与上游节点之间没有数据重分布（Shuffle）。因此，若数据有较大倾斜，可能会导致部分节点资源利用率不高，整体写入效率低下。此时，可以将sink.parallelism参数设为与上游节点并发不同的值。完成设置后，在实时计算开发控制台看到上游节点与写入节点之间出现数据重分布（不在同一个Subtask中）即可。

消费作业优化
调整Paimon Source并发

通过SQL Hints设置scan.parallelism参数调整Paimon Source的并发数。

使用zorder、hilbert或者order对数据排序

数据的顺序对批作业处理或即席（OLAP）查询的效率有较大影响。您可以对Append Scalable表中的数据进行排序，以提高查询效率。数据排序需要进行相关配置，详情请参见Paimon数据管理配置。作业的部署模式需要使用批模式，并在Entry Point Main Arguments中配置相关参数。

例如，当需要对分区中的数据按date和type字段排序时，Entry Point Main Arguments配置项的填写示例如下。

```bash
compact
--warehouse 'oss://your-bucket/data-warehouse'
--database 'your_database'
--table 'your_table'
--order_strategy 'zorder'
--order_by 'date,type'
--partition 'dt=20240311,hh=08;dt=20240312,hh=09'
--catalog_conf 'fs.oss.endpoint=oss-cn-hangzhou-internal.aliyuncs.com'
--catalog_conf 'fs.oss.endpoint=oss-cn-beijing-internal.aliyuncs.com'
--table_conf 'write-buffer-size=256 MB'
--table_conf 'your_table.logRetentionDuration=7 days'
```
参数说明如下。

| 配置项 | 说明 |
| -- | -- |
| warehouse | 需要排序的Paimon表所在的catalog的warehouse目录。|
|database|需要排序的Paimon表所在的数据库名称。table:需要排序的Paimon表名。|
| order_strategy |排序的方式，有以下三种排序方式：zorder：推荐在过滤字段小于5个且包含范围查询时使用。hilbert：推荐在过滤字段至少5个且包含范围查询时使用。order：推荐在过滤条件均为等值条件时使用。
|order_by|排序字段，请用英文逗号（,）分隔。
|partition|需要排序的分区，不同分区之间请用英文分号（;）分隔。非分区表可以删除该参数。
|catalog_conf|需要排序的Paimon表所在的Catalog的WITH参数，每一项WITH参数需要独占一行catalog_conf。
|table_conf|临时修改需要排序的Paimon表参数，与SQL Hint功能相同，每一项参数需要独占一行table_conf。
