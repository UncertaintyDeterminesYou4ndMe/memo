### 原理


#### 湖仓 catalog
1. 在StarRocks数据库系统中，如何在堆外（off-heap）内存中存储表数据，并且如何通过Starrocks后端（BE）的C++代码来解析这些数据。
在计算机科学中，堆外内存指的是直接通过操作系统的内存分配函数（如malloc或mmap）分配的内存，而不是通过编程语言的内存分配机制（如Java的堆内存）。使用堆外内存可以更好地控制内存使用，避免垃圾收集器的干扰，从而提高性能。

在StarRocks的堆外表内存布局中，数据按列存储，且每一列的数据在内存中是连续的。不同的数据列存储在堆外内存的不同位置。此外，为了处理数据中的空值，引入了空值指示列（null indicator columns）。还有一个元数据列（meta column），用于保存不同数据列的内存地址、空值指示列的内存地址和行数。

##### 具体的内存布局如下：

元数据列布局：元数据列的起始地址包含了行数，每个固定长度列的空值指示列和数据列的起始地址，以及每个可变长度列的空值指示列、偏移列和数据列的起始地址。

空值指示列布局：空值指示列的起始地址包含了一系列1字节的布尔值，每个布尔值对应一行数据，表示该行的字段是否为空。

数据列布局：
对于固定长度列（如BOOLEAN、INT、LONG），使用一级索引寻址方法。通过元数据列获取数据列的起始地址，然后使用该地址直接读取固定长度的数据。
对于可变长度列（如STRING、DECIMAL），使用二级索引寻址方法。首先通过元数据列获取数据列的起始地址，然后使用偏移列在特定行索引处获取字段的起始内存地址，通过当前行和下一行的偏移计算字段长度，最后使用字段的起始地址和长度读取可变长度的数据。
这种内存布局优化了数据访问的效率，尤其是在处理大型数据集时，可以减少内存的分配和复制，提高查询和数据处理的性能。此外，由于数据是按列存储的，因此可以更有效地进行列级别的压缩和编码，从而降低存储成本并提高I/O效率。

##### 示意图
示例数据表
假设我们有一个简单的数据表，包含三列：ID（固定长度整数），Name（可变长度字符串），和Age（固定长度整数），表中有4行数据：
|ID	|Name	|Age
|-	|-	|-
|1	|Alice	|30
|2	|Bob	|25
|3	|Charlie	|35
|4	|David	|NULL

堆外内存布局
元数据列布局
元数据列保存了各个数据列和空值指示列的内存地址以及行数。
```
+----------------+----------------------+--------------------+------------------------+
| Rows (4 bytes) | ID column addr (8 bytes) | Name column addr (8 bytes) | Age column addr (8 bytes) |
+----------------+----------------------+--------------------+------------------------+
| 4              | 0x1000               | 0x2000             | 0x3000                 |
+----------------+----------------------+--------------------+------------------------+
```
空值指示列布局
空值指示列包含布尔值，指示相应行的字段是否为空。

Age列空值指示器：
```
+---+---+---+---+
| 0 | 0 | 0 | 1 |  (1 indicates NULL)
+---+---+---+---+
```
数据列布局
ID列（固定长度列）：
```
+----+----+----+----+
| 1  | 2  | 3  | 4  |
+----+----+----+----+
```
Name列（可变长度列）：

偏移列指示每个字符串的起始地址。
```
偏移列：
+----+----+----+----+----+
|  0 |  5 |  8 | 15 | 20 | (终止偏移量)
+----+----+----+----+----+

数据列：
+-------+-------+--------+-------+------+
| Alice | Bob   | Charlie| David |
+-------+-------+--------+-------+
```
Age列（固定长度列）：
```
+----+----+----+----+
| 30 | 25 | 35 |    | (最后一个位置因为空值而为空)
+----+----+----+----+
```
具体操作示意
读取ID列的第2行（Bob的ID）
从元数据列获取ID列的起始地址 0x1000。
直接读取偏移 0x1000 + 1 * sizeof(int) 位置的值。
读取Name列的第2行（Bob的名字）
从元数据列获取Name列的起始地址 0x2000 和偏移列的起始地址 0x2000 + offset。
读取偏移列中第2和第3个偏移量 5 和 8。
读取 0x2000 + 5 到 0x2000 + 8 位置的字符串。
读取Age列的第4行（David的年龄）
从元数据列获取Age列的起始地址 0x3000。
检查空值指示列中第4个值是否为1。
由于第4个值为1，因此该行数据为空。
总结
通过这种内存布局，StarRocks能够高效地处理数据，尤其是对于大型数据集。按列存储使得访问特定列的数据更加高效，并且便于压缩和编码，从而提高整体性能和I/O效率。

2. 直接读取的 hms 的元数据信息，然后通过查询直接访问物理文件。不利用原有查询引擎。拉进 Doris 内存后利用 MPP、向量化、pipeline、CBO等引擎能力快速处理数据。Scan 是 be 做的。
```
联邦查询引擎是指计算在 doris，数据源是外部。be 是用来计算的。fe 没有计算能力
scan 都是 be 在做 并且有大量算子也是在 doris 侧做 比如 select sum(a)from jdbctable where b = 1 其中 sum(a) 也是在 doris 做的。没有 be 就没法运行了。所以联邦查询引擎不是 sql 转发，不是 sql proxy
```



性能：如果是 JDBC Catalog 的话是要依赖原库的性能的，Hive 的话主要还是 iops 以及网络 io

```
catalog Hive 源码

```

### 物化视图
刷新核心逻辑：mv会维护一个visiblemap，记录刷新过哪些分区；每次调度（周期/手动/自动）时候，检查哪些分区变更了，就会触发mv的刷新。
当前是分区级别全量刷新，目前看调度频繁对刷新压力很大的 。所以后续可以做成级联刷新(构造Task DAG)或者增量来解决比较好。

### 运维
```
curl 127.0.0.1:8040/metrics | grep "^starrocks_be_.*_mem_bytes"
```
#### 配置项解读
存算分离本地盘pk索引（persistent目录下）的淘汰策略：
当磁盘占用超过水位线：be配置 starlet_cache_evict_low_water时，会开始执行索引evict，evict的条件是index在一段时间（be.conf lake_local_pk_index_unused_threshold_seconds）内没有被使用过，也就是这个tablet在这段时间内没有导入过，那这个索引就可以被删除掉。
##### debug
be配置 sys_log_verbose_modules 设置成 * 
fe配置 sys_log_level 设置成 debug
##### jemalloc
FE 的配置项 JAVA_OPTS_FOR_JDK_9
改成  
```
"-Dlog4j2.formatMsgNoLookups=true -Xmx20g -XX:MaxDirectMemorySize=1g -XX:+UseMembar -XX:+UseStringDeduplication -XX:+UseG1GC -Xlog:gc*:file=${STARROCKS_HOME}/log/fe.gc.log.$DATE:time -XX:+PrintConcurrentLocks"
#####  **解决 fe fullGC 问题**
增加 JVM 参数
```
-XX:-UseAdaptiveSizePolicy -XX:+OptimizeFill -XX:ArrayCopyLoadStoreMaxElem=64
```
关闭自适应 启动填充优化,设置数组 copy 优化数值.
```

JAVA_OPTS_FOR_JDK_11=" -server -XX:+UnlockExperimentalVMOptions -XX:InitiatingHeapOccupancyPercent=90  -Dlog4j2.formatMsgNoLookups=true -Xmx22G -Xms22G -XX:MaxGCPauseMillis=200 -XX:+UseStringDeduplication -XX:StringDeduplicationAgeThreshold=7  -XX:G1HeapRegionSize=16m -XX:G1HeapWastePercent=20 -XX:G1MixedGCLiveThresholdPercent=70  -XX:MetaspaceSize=256m -XX:MaxTenuringThreshold=15  -XX:MaxMetaspaceSize=1g -XX:+UseG1GC -Xlog:gc*:${LOG_DIR}/fe.gc.log.$DATE:time"
