# Paimon表类型详解

## Paimon主键表

### 语法结构
创建Paimon表时指定了主键（primary key），则该表即为Paimon主键表。
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
Paimon主键表中每行数据的主键值各不相同，若多条相同主键数据写入，将根据数据合并机制进行合并。

### 分桶方式
分桶（Bucket）是Paimon表读写操作的最小单元，支持以下类别：
| 类别 | 定义 | 说明 |
|------|------|------|
| 动态分桶（默认） | 创建时不在WITH参数中指定`bucket`或指定`'bucket' = '-1'` | - 不支持多个作业同时写入<br>- 支持跨分区更新主键 |
| 固定分桶 | 创建时在WITH参数中指定`'bucket' = '<num>'`（`num`为大于0的整数） | 分区表的主键需完全包含分区键，避免主键的跨分区更新 |

### 动态分桶表更新
#### 跨分区更新的动态分桶表
对于主键不完全包含分区键的动态分桶表，Paimon需使用RocksDB维护主键与分区及分桶编号的映射关系，可能导致：
- 大数据量表产生明显性能损失
- 作业启动时需全量加载映射关系，启动速度变慢
- 数据合并机制影响更新结果：
  - deduplicate：数据从老分区删除并插入新分区
  - aggregation与partial-update：直接在老分区更新，无视新数据分区键
  - first-row：相同主键数据已存在时，新数据被丢弃

#### 非跨分区更新的动态分桶表
对于主键完全包含分区键的动态分桶表，Paimon需使用额外堆内存创建索引维护主键与分桶编号的映射关系：
- 每1亿条主键额外消耗1 GB堆内存
- 仅当前写入分区消耗堆内存，历史分区不消耗
- 除堆内存消耗外，无明显性能损失

### 数据分发
| 类别 | 数据分发机制 |
|------|------|
| 动态分桶 | - 数据先写入已有分桶，超过容量限制时自动创建新分桶<br>- 影响参数：<br>  `dynamic-bucket.target-row-num`（默认2000000条/桶）<br> `dynamic-bucket.initial-buckets`（初始分桶数，默认等于writer算子并发数） |
| 固定分桶 | - 默认按主键哈希值确定分桶<br>- 可通过`bucket-key`参数自定义分桶规则（主键需完整包含`bucket-key`） |

### 调整固定分桶表的分桶数量
#### 操作步骤
1. 停止所有写入或消费该表的作业
2. 执行SQL调整bucket参数：
```sql
ALTER TABLE `<catalog-name>`.`<database-name>`.`<table-name>` SET ('bucket' = '<bucket-num>');
```
3. 整理数据：
   - 非分区表：
```sql
INSERT OVERWRITE `<catalog-name>`.`<database-name>`.`<table-name>`
SELECT * FROM `<catalog-name>`.`<database-name>`.`<table-name>`;
```
   - 分区表（示例：整理`dt = 20240312, hh = 08`分区）：
```sql
INSERT OVERWRITE `<catalog-name>`.`<database-name>`.`<table-name>`
PARTITION (dt = '20240312', hh = '08')
SELECT * FROM `<catalog-name>`.`<database-name>`.`<table-name>`
WHERE dt = '20240312' AND hh = '08';
```
4. 批作业执行完成后恢复读写作业

### 变更数据产生机制
通过WITH参数中设置`changelog-producer`控制变更数据生成方式：
| 参数取值 | 说明 | 使用场景 |
|------|------|------|
| none（默认） | 不产出完整变更数据 | 仅支持批作业消费 |
| input | 直接将输入消息作为变更数据传递给下游 | 输入流本身是完整变更数据（如数据库Binlog），效率最高 |
| lookup | 通过批量点查触发小文件合并生成变更数据 | 对数据新鲜度要求高（分钟级），资源消耗较多 |
| full-compaction | 在每一次小文件全量合并时生成变更数据 | 对数据新鲜度要求不高（小时级），利用合并流程不产生额外计算 |

**说明**：
- 默认情况下，即使更新后数据与之前相同仍会产生变更数据
- 可通过`'changelog-producer.row-deduplicate' = 'true'`消除无效变更数据（仅对lookup与full-compaction有效）

### 数据合并机制
#### 参数说明
通过WITH参数中`merge-engine`参数控制合并行为：

##### deduplicate（默认值）
设置`'merge-engine' = 'deduplicate'`后，仅保留相同主键的最新数据：
```sql
CREATE TABLE T (
  k INT,
  v1 DOUBLE,
  v2 STRING,
  PRIMARY KEY (k) NOT ENFORCED
) WITH (
  'merge-engine' = 'deduplicate'
);
```
- 写入顺序：`+I(1, 2.0, 'apple')`，`+I(1, 4.0, 'banana')`，`+I(1, 8.0, 'cherry')` → 查询结果：`(1, 8.0, 'cherry')`
- 写入顺序：`+I(1, 2.0, 'apple')`，`+I(1, 4.0, 'banana')`，`-D(1, 4.0, 'banana')` → 查询无结果

##### first-row
设置`'merge-engine' = 'first-row'`后，保留相同主键的第一条数据：
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
- 写入顺序：`+I(1, 2.0, 'apple')`，`+I(1, 4.0, 'banana')`，`+I(1, 8.0, 'cherry')` → 查询结果：`(1, 2.0, 'apple')`
- **说明**：
  - 下游流式消费需设置`changelog-producer`为lookup
  - 无法处理delete与update_before消息，可通过`'first-row.ignore-delete' = 'true'`忽略
  - 不支持指定sequence field

##### aggregation
按指定聚合函数合并相同主键数据，需为非主键列指定聚合函数：
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
- 写入顺序：`+I(1, 23.0, 15)`，`+I(1, 30.2, 20)` → 查询结果：`(1, 30.2, 35)`
- **支持的聚合函数**：
  - sum（求和）：DECIMAL、TINYINT、SMALLINT、INTEGER、BIGINT、FLOAT、DOUBLE
  - product（求乘积）：同上
  - count（统计非null值总数）：INTEGER、BIGINT
  - max/min（最大/最小值）：CHAR、VARCHAR、DECIMAL、数值类型、DATE、TIME、TIMESTAMP、TIMESTAMP_LTZ
  - first_value/last_value（首次/最新值）：所有数据类型（含null）
  - first_not_null_value/last_non_null_value（首次/最新非null值）：所有数据类型
  - listagg（字符串连接）：STRING
  - bool_and/bool_or：BOOLEAN
- **说明**：仅sum、product、count支持回撤消息，可通过`'fields.<field-name>.ignore-retract' = 'true'`忽略

##### partial-update
设置`'merge-engine' = 'partial-update'`后，新数据覆盖旧数据，null值不覆盖：
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
- 写入顺序：`+I(1, 23.0, 10, NULL)`，`+I(1, NULL, NULL, 'This is a book')`，`+I(1, 25.2, NULL, NULL)` → 查询结果：`(1, 25.2, 10, 'This is a book')`
- **高级功能**：
  - **Sequence Group**：为不同列指定合并顺序
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
  - **聚合与打宽**：为列指定聚合函数
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

##### 乱序数据处理
通过`'sequence.field' = '<column-name>'`指定顺序字段（支持TINYINT、SMALLINT、INTEGER、BIGINT、TIMESTAMP、TIMESTAMP_LTZ）：
- 使用MySQL的`op_t`元数据作为sequence field时，需设置`'sequence.auto-padding' = 'row-kind-flag'`保证update_before先于update_after处理

## Paimon Append Only表（非主键表）
创建时未指定主键的Paimon表，仅支持流式插入完整记录，适合无需流式更新的场景（如日志同步）。

### 语法结构
```sql
CREATE TABLE T (
  dt STRING,
  order_id BIGINT,
  item_id BIGINT,
  amount INT,
  address STRING
) PARTITIONED BY (dt) WITH (
  'bucket' = '-1'
);
```

### 表类型
| 类型 | 定义 | 说明 |
|------|------|------|
| Append Scalable表 | 创建时指定`'bucket' = '-1'` | Hive表的高效替代，写入无shuffle、数据可排序、并发控制便捷，支持异步合并加速写入 |
| Append Queue表 | 创建时指定`'bucket' = '<num>'`（`num`为大于0的整数） | 消息队列的替代，分桶数相当于Kafka的Partition数或MQTT的Shard数 |

### 数据的分发
| 类型 | 说明 |
|------|------|
| Append Scalable表 | 无视分桶，多并发可同时写同一分区，写入时直接由上游推往writer节点，无需hash partitioning，写入速度快（需注意数据倾斜） |
| Append Queue表 | - 默认按全列取值确定分桶<br>- 可通过`bucket-key`参数指定分桶列（建议设置以提高写入效率） |

### 数据消费顺序
| 类型 | 说明 |
|------|------|
| Append Scalable表 | 不保证流式消费顺序与写入顺序一致，适合无顺序需求场景 |
| Append Queue表 | - 同分区同桶数据按写入顺序消费<br>- 同分区不同桶数据因并发处理不保证顺序<br>- 跨分区数据：<br>  `'scan.plan-sort-partition' = 'true'`时按分区值排序消费<br>  未设置时按分区创建时间排序消费 |
