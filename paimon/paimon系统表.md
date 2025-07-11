# Paimon系统表  



Paimon系统表用于存储Paimon表的元数据和特定的数据消费行为。本文介绍Paimon元数据系统表和特定消费行为的系统表作用、相关字段及含义等。  


## 元数据系统表  


### Snapshots系统表  
#### 作用  
查询每个快照文件的具体信息，例如快照文件的编号、创建时间等。  

#### 查询示例  
```sql
SELECT * FROM mycatalog.mydb.`mytbl$snapshots`;
```

#### 常用列说明  

| **列名**       | **数据类型** | **含义**                                                                 |
|----------------|--------------|--------------------------------------------------------------------------|
| snapshot_id    | Long         | 快照文件的编号。                                                         |
| schema_id      | Long         | 快照文件对应的表结构文件编号，表结构可在Schemas系统表中查看。           |
| commit_time    | Timestamp    | 快照文件的创建时间。                                                     |
| total_record_count | Long       | 快照文件指向的数据文件中数据的总条数（数据需归并后才是实际数据量）。   |
| delta_record_count | Long       | 与上一个快照相比，数据文件中增加的数据条数。                             |
| changelog_record_count | Long     | 本次快照产生的变更数据条数。                                             |


### Schemas系统表  
#### 作用  
查询表的当前及历史结构信息，每次修改表结构时会新增一条记录。  

#### 查询示例  
```sql
SELECT * FROM mycatalog.mydb.`mytbl$schemas`;
```

#### 常用列说明  

| **列名**       | **数据类型** | **含义**                                                                 |
|----------------|--------------|--------------------------------------------------------------------------|
| schema_id      | Long         | 表结构的编号。                                                           |
| fields         | String       | 每一列的名称及类型等。                                                   |
| partition_keys | String       | 分区列的名称。                                                           |
| primary_keys   | String       | 主键的名称。                                                             |
| options        | String       | 表参数的值。                                                             |
| comment        | String       | 表的备注信息。                                                           |
| update_time    | Timestamp    | 表结构修改的时间。                                                       |


### Options系统表  
#### 作用  
查询当前配置的表参数。  

#### 查询示例  
```sql
SELECT * FROM mycatalog.mydb.`mytbl$options`;
```

#### 常用列说明  

| **列名** | **数据类型** | **含义**               |
|----------|--------------|------------------------|
| key      | String       | 配置项的名称。         |
| value    | String       | 配置项的值。           |

**说明**：未显示的表参数使用默认值。  


### Partitions系统表  
#### 作用  
查询Paimon表的分区、每个分区的数据总数及文件总量。  

#### 查询示例  
```sql
SELECT * FROM mycatalog.mydb.`mytbl$partitions`;
```

#### 常用列说明  

| **列名**         | **数据类型** | **含义**                                                                 |
|------------------|--------------|--------------------------------------------------------------------------|
| partition        | String       | 分区值，格式为`[分区值1, 分区值2, ...]`。                                 |
| record_count     | Long         | 分区内的数据条数（数据需归并后才是实际数据量）。                         |
| file_size_in_bytes | Long       | 分区内的文件总大小（未被当前快照指向的历史文件不统计）。                |


### Files系统表  
#### 作用  
查询某个快照文件指向的所有数据文件，包括格式、数据条数和文件大小等。  

#### 查询示例  
```sql
-- 查询最新快照对应的Files表
SELECT * FROM mycatalog.mydb.`mytbl$files`;

-- 查询编号为5的快照对应的Files表
SELECT * FROM mycatalog.mydb.`mytbl$files` /*+ OPTIONS('scan.snapshot-id'='5') */;
```

#### 常用列说明  

| **列名**         | **数据类型** | **含义**                                                                 |
|------------------|--------------|--------------------------------------------------------------------------|
| partition        | String       | 文件所在的分区，格式为`[分区值1, 分区值2, ...]`。                         |
| bucket           | Integer      | 文件所在的分桶（仅对固定分桶的主键表有意义）。                           |
| file_path        | String       | 文件路径。                                                               |
| file_format      | String       | 文件格式。                                                               |
| schema_id        | Long         | 快照对应的表结构编号（可在Schemas表中查看）。                           |
| level            | Integer      | LSM层级（仅对主键表有意义，level=0为未合并的小文件）。                   |
| record_count     | Long         | 文件内的数据条数。                                                       |
| file_size_in_bytes | Long       | 文件大小（字节）。                                                       |

**说明**：未被当前快照指向的历史文件不显示。  


### Tags系统表  
#### 作用  
查询每个Tag文件的具体信息，例如Tag名称、创建时使用的快照编号等。  

#### 查询示例  
```sql
SELECT * FROM mycatalog.mydb.`mytbl$tags`;
```

#### 常用列说明  

| **列名**       | **数据类型** | **含义**                                                                 |
|----------------|--------------|--------------------------------------------------------------------------|
| tag_name       | String       | Tag名称。                                                                 |
| snapshot_id    | Long         | 创建Tag时基于的快照编号。                                                 |
| schema_id      | Long         | Tag对应的表结构编号（可在Schemas表中查询）。                             |
| commit_time    | Timestamp    | 创建Tag时基于的快照的创建时间。                                           |
| record_count   | Long         | 文件内的数据条数。                                                       |


## 特定消费行为的系统表  


### Read-optimized系统表  
#### 作用  
提升Paimon主键表的批作业读取或即席（OLAP）查询效率，仅读取无需合并的文件（如全量合并后的大文件），省去内存归并流程，但数据时效性较低。  

#### 配置参数  
| **参数**                   | **说明**                           | **数据类型** | **默认值** | **备注**                                                                 |
|----------------------------|------------------------------------|--------------|------------|--------------------------------------------------------------------------|
| compaction.optimization-interval | 超过该时间强制合并小文件。 | Duration     | 无         | 推荐设置为`30 min`以上，避免频繁合并影响写入效率。                       |

#### 查询示例  
```sql
SELECT * FROM mycatalog.mydb.`mytbl$ro`;
```


### Audit Log系统表  
#### 作用  
查询数据的操作类型（插入、删除等），在每条数据前新增`rowkind`列，记录操作类型（+I、-U、+U、-D）。  

#### 查询示例  
```sql
SELECT * FROM mycatalog.mydb.`mytbl$audit_log`;
```
