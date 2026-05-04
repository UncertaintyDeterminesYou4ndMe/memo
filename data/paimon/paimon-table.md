# 前文
> [表模型](https://help.aliyun.com/zh/flink/use-cases/basic-concepts?spm=a2c4g.11186623.0.0.167a719ekfAuH4#64abb97f8fnfe)

# Paimon 主键表

## 一、定义
创建 Paimon 表时指定了主键（primary key），则该表即为 Paimon 主键表。

## 二、语法结构
例如，创建一张分区键为`dt`，主键为`dt`、`shop_id`和`user_id`，分桶数固定为 4 的 Paimon 主键表。
```sql
CREATE TABLE T (
  dt STRING,
  shop_id BIGINT,
  user_id BIGINT,
  num_orders INT,
  total_amount INT,
  PRIMARY KEY (dt, shop_id, user_id) NOT ENFORCED
) PARTITIONED BY (dt) WITH (
  'bucket' = '4'
);
```

## 三、分桶方式

### （一）类别
1. **动态分桶（默认）**：创建 Paimon 主键表时，不在 WITH 参数中指定`bucket`或指定`'bucket' = '-1'`，将会创建动态分桶的 Paimon 表。
   - 动态分桶表不支持多个作业同时写入。
   - 对于动态分桶的 Paimon 主键表，可以支持跨分区更新主键。
2. **固定分桶**：创建 Paimon 主键表时，在 WITH 参数中指定`'bucket' = '<num>'`，即可指定非分区表的分桶数为`<num>`，或者分区表单个分区的分桶数为`<num>`。`<num>`是一个大于 0 的整数。
   - 如果在创建固定分桶的 Paimon 表之后，需要修改分桶数，详情请参见调整固定分桶表的分桶数量。
   - 对于固定分桶的 Paimon 主键表，分区表的主键需要完全包含分区键（partition key），以避免主键的跨分区更新。

### （二）动态分桶表更新

1. **跨分区更新的动态分桶表**：
   - 对于主键不完全包含分区键的动态分桶表，Paimon 无法根据主键确定该数据属于哪个分区的哪个分桶，因此需要使用 RocksDB 维护主键与分区以及分桶编号的映射关系。相比固定分桶而言，数据量较大的表可能会产生明显的性能损失。另外，因为作业启动时需要将映射关系全量加载至 RocksDB 中，作业的启动速度也会变慢。
   - 数据合并机制会对跨分区更新的结果产生影响：
     - `deduplicate`：数据将会从老分区删除，并插入新分区。
     - `aggregation`与`partial-update`：数据将会直接在老分区中更新，无视新数据的分区键。
     - `first-row`：如果相同主键的数据已经存在，则新数据将被直接丢弃。
2. **非跨分区更新的动态分桶表**：
   - 对于主键完全包含分区键的动态分桶表，Paimon 可以确定该主键属于哪个分区，但无法确定属于哪个分桶，因此需要使用额外的堆内存创建索引，以维护主键与分桶编号的映射关系。
   - 具体来说，每 1 亿条主键将额外消耗 1 GB 的堆内存。只有当前正在写入的分区才会消耗堆内存，历史分区中的主键不会消耗堆内存。
   - 除堆内存的消耗外，相比固定分桶而言，主键完全包含分区键的动态分桶表不会有明显的性能损失。

### （三）数据分发

1. **动态分桶**：
   - 动态分桶的 Paimon 表会先将数据写入已有的分桶中，当分桶的数据量超过限制时，再自动创建新的分桶。以下 WITH 参数将会影响动态分桶的行为。
     - `dynamic-bucket.target-row-num`：每个分桶最多存储几条数据。默认值为 2000000。
     - `dynamic-bucket.initial-buckets`：初始的分桶数。如果不设置，初始将会创建等同于 writer 算子并发数的分桶。
2. **固定分桶**：
   - 默认情况下，Paimon 将根据每条数据主键的哈希值，确定该数据属于哪个分桶。
   - 如果您需要修改数据的分桶方式，可以在创建 Paimon 表时，在 WITH 参数中指定`bucket-key`参数，不同列的名称用英文逗号分隔，主键必须完整包含`bucket-key`。例如，如果设置了`'bucket-key' = 'c1,c2'`，则 Paimon 将根据每条数据`c1`和`c2`两列的值，确定该数据属于哪个分桶。

## 四、调整固定分桶表的分桶数量
由于分桶数限制了实际工作的作业并发数，且单个分桶内数据总量太大可能导致读写性能的降低，因此分桶数不宜太小。但是，分桶数过大也会造成小文件数量过多。建议每个分桶的数据大小在 2 GB 左右，最大不超过 5 GB。调整固定分桶表的分桶数量具体的操作步骤如下：
1. 停止所有写入该 Paimon 表或消费该 Paimon 表的作业。
2. 新建查询脚本，执行以下 SQL 语句，调整 Paimon 表的`bucket`参数。
```sql
ALTER TABLE `<catalog-name>`.`<database-name>`.`<table-name>` SET ('bucket' = '<bucket-num>');
```
3. 整理非分区表中的所有数据，或分区表中仍需写入的分区中的所有数据。
   - 非分区表：新建空白批作业草稿，在 SQL 编辑器中编写以下 SQL 语句，之后部署并启动批作业。
```sql
INSERT OVERWRITE `<catalog-name>`.`<database-name>`.`<table-name>`
SELECT * FROM `<catalog-name>`.`<database-name>`.`<table-name>`;
```
   - 分区表：新建空白批作业草稿，在 SQL 编辑器中编写以下 SQL 语句，之后部署并启动批作业。
```sql
INSERT OVERWRITE `<catalog-name>`.`<database-name>`.`<table-name>`
PARTITION (<partition-spec>)
SELECT * FROM `<catalog-name>`.`<database-name>`.`<table-name>`
WHERE <partition-condition>;
```
其中，`<partition-spec>`和`<partition-condition>`指定了需要整理的分区。例如，整理`dt = 20240312, hh = 08`分区中的数据的 SQL 语句如下：
```sql
INSERT OVERWRITE `<catalog-name>`.`<database-name>`.`<table-name>`
PARTITION (dt = '20240312', hh = '08')
SELECT * FROM `<catalog-name>`.`<database-name>`.`<table-name>`
WHERE dt = '20240312' AND hh = '08';
```
4. 批作业执行完成后，即可恢复 Paimon 表的写入作业与消费作业。

## 五、变更数据产生机制
Paimon 表需要将数据的增删与更新改写为完整的变更数据（类似于数据库的 Binlog），才能让下游进行流式消费。通过在 WITH 参数中设置`changelog-producer`，Paimon 将会以不同的方式产生变更数据，其取值如下：

### （一）参数取值及说明
1. `none`（默认）：Paimon 主键表将不会产出完整的变更数据。Paimon 表仅能通过批作业进行消费，不能通过流作业进行消费。
2. `input`：Paimon 主键表会直接将输入的消息作为变更数据传递给下游消费者。仅在输入数据流本身是完整的变更数据时（例如数据库的 Binlog）使用。由于`input`机制不涉及额外的计算，因此其效率最高。
3. `lookup`：Paimon 主键表会通过批量点查的方式，在 Flink 作业每次创建检查点（checkpoint）时触发小文件合并（compaction），并利用小文件合并的结果产生完整的变更数据。无论输入数据流是否为完整的变更数据，都可以使用该机制。与`full-compaction`机制相比，`lookup`机制的时效性更好，但总体来看耗费的资源更多。推荐在对数据新鲜度有较高要求（分钟级）的情况下使用。
4. `full-compaction`：Paimon 主键表会在每一次执行小文件全量合并（full compaction）时，产生完整的变更数据。无论输入数据流是否为完整的变更数据，都可以使用该机制。与`lookup`机制相比，`full-compaction`机制的时效性较差，但其利用了小文件合并流程，不产生额外计算，因此总体来看耗费的资源更少。推荐在对数据新鲜度要求不高（小时级）的情况下使用。

### （二）说明
为了保证`full-compaction`机制的时效性，您可以在 WITH 参数中设置`'full-compaction.delta-commits' = '<num>'`，要求 Paimon 在每`<num>`个 Flink 作业的检查点执行小文件全量合并。然而，由于小文件全量合并会消耗较多计算资源，因此频率不宜过高，建议每 30 分钟至 1 小时强制执行一次。

默认情况下，即使更新后的数据与更新之前相同，Paimon 仍然会产生变更数据。如果您希望消除此类无效的变更数据，可以在 WITH 参数中设置`'changelog-producer.row-deduplicate' = 'true'`。该参数仅对`lookup`与`full-compaction`机制有效。由于设置该参数后，需要引入额外的计算对比更新前后的值，推荐仅在无效变更数据较多的情况下使用该参数。

## 六、数据合并机制

### （一）参数说明
如果将多条具有相同主键的数据写入 Paimon 主键表，Paimon 将会根据 WITH 参数中设置的`merge-engine`参数对数据进行合并。参数取值如下：

1. `deduplicate`（默认值）：
   - 设置`'merge-engine' = 'deduplicate'`后，对于多条具有相同主键的数据，Paimon 主键表仅会保留最新一条数据，并丢弃其它具有相同主键的数据。如果最新数据是一条`delete`消息，所有具有该主键的数据都会被丢弃。创建 Paimon 表的 DDL 语句如下：
```sql
CREATE TABLE T (
  k INT,
  v1 DOUBLE,
  v2 STRING,
  PRIMARY KEY (k) NOT ENFORCED
) WITH (
  'merge-engine' = 'deduplicate' -- deduplicate 是默认值，可以不设置
);
```
   - 如果写入 Paimon 表的数据依次为`+I(1, 2.0, 'apple')`，`+I(1, 4.0, 'banana')`，`+I(1, 8.0, 'cherry')`，则`SELECT * FROM T WHERE k = 1`将查询到`(1, 8.0, 'cherry')`这条数据。
   - 如果写入 Paimon 表的数据依次为`+I(1, 2.0, 'apple')`，`+I(1, 4.0, 'banana')`，`-D(1, 4.0, 'banana')`，则`SELECT * FROM T WHERE k = 1`将查不到任何数据。
2. `first-row`：
   - 设置`'merge-engine' = 'first-row'`后，Paimon 只会保留相同主键数据中的第一条。与`deduplicate`合并机制相比，`first-row`只会产生`insert`类型的变更数据，且变更数据的产出效率更高。
   - 说明：如果下游需要流式消费`first-row`的结果，需要将`changelog-producer`参数设为`lookup`。`first-row`无法处理`delete`与`update_before`消息。您可以设置`'first-row.ignore-delete' = 'true'`以忽略这两类消息。`first-row`合并机制不支持指定`sequence field`。
   - 例如，创建 Paimon 表的 DDL 语句如下：
```sql
CREATE TABLE T (
  k INT,
  v1 DOUBLE,
  v2 STRING,
  PRIMARY KEY (k) NOT ENFORCED
) WITH (
  'merge-engine' = 'first-row'
);
```
   - 如果写入 Paimon 表的数据依次为`+I(1, 2.0, 'apple')`，`+I(1, 4.0, 'banana')，+I(1, 8.0, 'cherry')`，则`SELECT * FROM T WHERE k = 1`将查询到`(1, 2.0, 'apple')`这条数据。
3. `aggregation`：
   - 对于多条具有相同主键的数据，Paimon 主键表将会根据您指定的聚合函数进行聚合。对于不属于主键的每一列，都需要通过`fields.<field-name>.aggregate-function`指定一个聚合函数，否则该列将默认使用`last_non_null_value`聚合函数。
   - 说明：如果下游需要流式消费`aggregation`的结果，需要将`changelog-producer`参数设为`lookup`或`full-compaction`。
   - 例如，`price`列将会使用`max`函数进行聚合，而`sales`列将会使用`sum`函数进行聚合。
```sql
CREATE TABLE T (
  product_id BIGINT,
  price DOUBLE,
  sales BIGINT,
  PRIMARY KEY (product_id) NOT ENFORCED
) WITH (
  'merge-engine' = 'aggregation',
  'fields.price.aggregate-function' = 'max',
  'fields.sales.aggregate-function' = 'sum'
);
```
   - 如果写入 Paimon 表的数据依次为`+I(1, 23.0, 15)`和`+I(1, 30.2, 20)`，`SELECT * FROM T WHERE product_id = 1`将查询到`(1, 30.2, 35)`这条数据。
   - 支持的聚合函数与对应的数据类型如下：
     - `sum`（求和）：支持`DECIMAL`、`TINYINT`、`SMALLINT`、`INTEGER`、`BIGINT`、`FLOAT`和`DOUBLE`。
     - `product`（求乘积）：支持`DECIMAL`、`TINYINT`、`SMALLINT`、`INTEGER`、`BIGINT`、`FLOAT`和`DOUBLE`。
     - `count`（统计非 null 值总数）：支持`INTEGER`和`BIGINT`。
     - `max`（最大值）和`min`（最小值）：`CHAR`、`VARCHAR`、`DECIMAL`、`TINYINT`、`SMALLINT`、`INTEGER`、`BIGINT`、`FLOAT`、`DOUBLE`、`DATE`、`TIME`、`TIMESTAMP`和`TIMESTAMP_LTZ`。
     - `first_value`（返回第一次输入的值）和`last_value`（返回最新输入的值）：支持所有数据类型，包括 null。
     - `first_not_null_value`（返回第一次输入的非 null 值）和`last_non_null_value`（返回最新输入的非 null 值）：支持所有数据类型。
     - `listagg`（将输入的字符串依次用英文逗号连接）：支持`STRING`。
     - `bool_and`和`bool_or`：支持`BOOLEAN`。
   - 说明：上述聚合函数中，只有`sum`、`product`和`count`支持回撤消息（`update_before`与`delete`消息）。您可以设置`'fields.<field-name>.ignore-retract' = 'true'`使对应列忽略回撤消息。
4. `partial-update`：
   - 设置`'merge-engine' = 'partial-update'`后，您可以通过多条消息对数据进行逐步更新，并最终得到完整的数据。即具有相同主键的新数据将会覆盖原来的数据，但值为 null 的列不会进行覆盖。
   - 说明：如果下游需要流式消费`partial-update`的结果，需要将`changelog-producer`参数设为`lookup`或`full-compaction`。`partial-update`无法处理`delete`与`update_before`消息。您可以设置`'partial-update.ignore-delete' = 'true'`以忽略这两类消息。
   - 例如，考虑以下创建 Paimon 表的 DDL 语句：
```sql
CREATE TABLE T (
  k INT,
  v1 DOUBLE,
  v2 BIGINT,
  v3 STRING,
  PRIMARY KEY (k) NOT ENFORCED
) WITH (
  'merge-engine' = 'partial-update'
);
```
   - 如果写入 Paimon 表的数据依次为`+I(1, 23.0, 10, NULL)`，`+I(1, NULL, NULL, 'This is a book')`，`+I(1, 25.2, NULL, NULL)`，则`SELECT * FROM T WHERE k = 1`将查询到`(1, 25.2, 10, 'This is a book')`这条数据。
   - 在`partial-update`合并机制中，您还可以设置指定 WITH 参数指定合并顺序或对数据进行打宽与聚合，详情如下：
     - 通过 Sequence Group 为不同列分别指定合并顺序：除了`sequence field`之外，您也可以通过`sequence group`为不同列分别指定合并顺序。您可以为来自不同源表的列分别指定合并顺序，帮您在打宽场景下处理乱序数据。例如，`a`、`b`两列根据`g_1`列的值从小到大进行合并，而`c`、`d`两列将根据`g_2`列的值从小到大进行合并。
```sql
CREATE TABLE T (
  k INT,
  a STRING,
  b STRING,
  g_1 INT,
  c STRING,
  d STRING,
  g_2 INT,
  PRIMARY KEY (k) NOT ENFORCED
) WITH (
  'merge-engine' = 'partial-update',
  'fields.g_1.sequence-group' = 'a,b',
  'fields.g_2.sequence-group' = 'c,d'
);
```
     - 同时进行数据的打宽与聚合：您还可以在 WITH 参数中设置`fields.<field-name>.aggregate-function`，为`<field-name>`这一列指定聚合函数，对该列的值进行聚合。`<field-name>`这一列需要属于某个 sequence group。aggregation 合并机制支持的聚合函数均可使用。例如，`a`、`b`两列将根据`g_1`列的值从小到大进行合并，其中`a`列将会保留最新的非 null 值，而`b`列将会保留输入的最大值。`c`、`d`两列将根据`g_2`列的值从小到大进行合并，其中`c`列将会保留最新的非 null 值，而`d`列将会求出输入数据的和。
```sql
CREATE TABLE T (
  k INT,
  a STRING,
  b INT,
  g_1 INT,
  c STRING,
  d INT,
  g_2 INT,
  PRIMARY KEY (k) NOT ENFORCED
) WITH (
  'merge-engine' = 'partial-update',
  'fields.g_1.sequence-group' = 'a,b',
  'fields.b.aggregate-function' = 'max',
  'fields.g_2.sequence-group' = 'c,d',
  'fields.d.aggregate-function' = 'sum'
);
```
更多信息，详情请参见 Merge Engine。

## 七、乱序数据处理
默认情况下，Paimon 会按照数据的输入顺序确定合并的顺序，最后写入 Paimon 的数据会被认为是最新数据。如果您的输入数据流存在乱序数据，可以通过在 WITH 参数中指定`'sequence.field' = '<column-name>'`，具有相同主键的数据将按`<column-name>`这一列的值从小到大进行合并。可以作为 sequence field 的数据类型有：`TINYINT`、`SMALLINT`、`INTEGER`、`BIGINT`、`TIMESTAMP`和`TIMESTAMP_LTZ`。

说明：如果您使用 MySQL 的 op_t 元数据作为 sequence field，会导致一对 update_before 与 update_after 消息具有相同的 sequence field 值，需要在 WITH 参数中设置`'sequence.auto-padding' = 'row-kind-flag'`，以保证 Paimon 会先处理 update_before 消息，再处理 update_after 消息。

# Paimon Append Only 表（非主键表）
如果在创建 Paimon 表时没有指定主键（Primary Key），则该表就是 Paimon Append Only 表。您只能以流式方式将完整记录插入到表中，适合不需要流式更新的场景（例如日志数据同步）。

语法结构：例如，以下 SQL 语句将创建一张分区键为`dt`的 Append Scalable 表。
```sql
CREATE TABLE T (
  dt STRING
  order_id BIGINT,
  item_id BIGINT,
  amount INT,
  address STRING
) PARTITIONED BY (dt) WITH (
  'bucket' = '-1'
);
```

## 一、类型及定义
1. **Append Scalable 表**：创建 Paimon Append Only 表时，在 WITH 参数中指定`'bucket' = '-1'`，将会创建 Append Scalable 表。
2. **Append Queue 表**：创建 Paimon Append Only 表时，在 WITH 参数中指定`'bucket' = '<num>'`，将会创建 Append Queue 表。其中，`<num>`是一个大于 0 的整数，指定了非分区表的分桶数，或分区表单个分区的分桶数。

## 二、说明

### （一）Append Scalable 表
作为 Hive 表的高效替代，在对数据的流式消费顺序没有需求的情况下，应尽量选择 Append Scalable 表。它具有写入无 shuffle、数据可排序、并发控制便捷，可直接将直接吸收或者转换现存的 hive 表、且可以使用异步合并等优势，进一步加快写入过程。

### （二）Append Queue 表
作为消息队列具有分钟级延迟的替代。Paimon 表的分桶数此时相当于 Kafka 的 Partition 数或云消息队列 MQTT 版的 Shard 数。

## 三、数据的分发

### （一）类型及说明
1. **Append Scalable 表**：由于无视桶的概念，多个并发可以同时写同一个分区，无考虑顺序以及数据是否分到对应桶的问题。在写入时，直接由上游将数据推往 writer 节点。其中，不需要对数据进行 hash partitioning。由此，在其他条件相同的情况下，通常此类型的表写入速度更快。当上游并发和 writer 并发相同时，需要注意是否会产生数据倾斜问题。
2. **Append Queue 表**：默认情况下，Paimon 将根据每条数据所有列的取值，确定该数据属于哪个分桶（bucket）。如果您需要修改数据的分桶方式，可以在创建 Paimon 表时，在 WITH 参数中指定`bucket-key`参数，不同列的名称用英文逗号分隔。例如，设置`'bucket-key' = 'c1,c2'`，则 Paimon 将根据每条数据`c1`和`c2`两列的值，确定该数据属于哪个分桶。

### （二）说明
建议尽可能设置`bucket-key`，以减少分桶过程中参与计算的列的数量，提高 Paimon 表的写入效率。

## 四、数据消费顺序

### （一）类型及说明
1. **Append Scalable 表**：无法像 Append Queue 表一样，在流式消费 Paimon 表时保证数据的消费顺序与数据写入 Paimon 表的顺序一致。适合对数据的流式消费顺序没有需求场景。
2. **Append Queue 表**：Append Queue 表可以保证流式消费 Paimon 表时，每个分桶中数据的消费顺序与数据写入 Paimon 表的顺序一致。具体来说：
   - 对于两条来自不同分区（partition）的数据：
     - 如果表参数中设置了`'scan.plan-sort-partition' = 'true'`，则分区值更小的数据会首先产出。
     - 如果表参数中未设置`'scan.plan-sort-partition' = 'true'`，则分区创建时间更早的数据会首先产出。
   - 对于两条来自相同分区的相同分桶的数据，先写入 Paimon 表的数据会首先产出。
   - 对于两条来自相同分区但不同分桶的数据，由于不同分桶可能被不同的 Flink 作业并发处理，因此不保证两条数据的消费顺序。

## 五、相关文档
Paimon Catalog 和 Paimon 表的创建操作，详情请参见管理 Paimon Catalog。如果您需要了解 Paimon 主键表的常用优化，详情请请见 Paimon 性能优化。

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
