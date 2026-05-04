## 原理

### jdbc catalog join paimon catalog
```
PLAN FRAGMENT 0
 OUTPUT EXPRS:5: user_id
  PARTITION: UNPARTITIONED

  RESULT SINK

  12:Project
  |  <slot 5> : 5: user_id
  |  
  11:HASH JOIN
  |  join op: NULL AWARE LEFT ANTI JOIN (BROADCAST)
  |  colocate: false, reason: 
  |  equal join conjunct: 41: cast = 42: cast
  |  
  |----10:EXCHANGE
  |    
  3:Project
  |  <slot 5> : 5: user_id
  |  <slot 41> : CAST(5: user_id AS DOUBLE)
  |  
  2:AGGREGATE (update finalize)
  |  group by: 5: user_id
  |  
  1:Project
  |  <slot 5> : user_id
  |  
  0:SCAN JDBC
     TABLE: `channel_position_trans_record`
     QUERY: SELECT `task_id`, `user_id`, `applet_id`, `init_scene_name`, `finish_date` FROM `channel_position_trans_record` WHERE (`applet_id` = 800000000000) AND (`init_scene_name` = 'shequn_yunma') AND (`task_id` = '789') AND (`finish_date` = '2024-10-29')

PLAN FRAGMENT 1
 OUTPUT EXPRS:
  PARTITION: HASH_PARTITIONED: 39: get_json_object

  STREAM DATA SINK
    EXCHANGE ID: 10
    UNPARTITIONED

  9:Project
  |  <slot 42> : CAST(39: get_json_object AS DOUBLE)
  |  
  8:AGGREGATE (merge finalize)
  |  group by: 39: get_json_object
  |  
  7:EXCHANGE

PLAN FRAGMENT 2
 OUTPUT EXPRS:
  PARTITION: RANDOM

  STREAM DATA SINK
    EXCHANGE ID: 07
    HASH_PARTITIONED: 39: get_json_object

  6:AGGREGATE (update serialize)
  |  STREAMING
  |  group by: 39: get_json_object
  |  
  5:Project
  |  <slot 39> : get_json_object(25: extparams, '$.uid')
  |  
  4:PaimonScanNode
     TABLE: paimon_track_event_dt
     NON-PARTITION PREDICATES: 37: dt = '2024-10-29', 34: pagealias = '聚合页', 23: eventid = 'taskFinish', get_json_object(25: extparams, '$.task_name') = 'shequn_yunma_789', get_json_object(25: extparams, '$.task_desc') = '逛15秒直播'
     MIN/MAX PREDICATES: 37: dt <= '2024-10-29', 37: dt >= '2024-10-29', 34: pagealias <= '聚合页', 34: pagealias >= '聚合页', 23: eventid <= 'taskFinish', 23: eventid >= 'taskFinish'
     cardinality=-1
     avgRowSize=0.0
```

### 湖仓 catalog
1. 在StarRocks数据库系统中，如何在堆外（off-heap）内存中存储表数据，并且如何通过Starrocks后端（BE）的C++代码来解析这些数据。
在计算机科学中，堆外内存指的是直接通过操作系统的内存分配函数（如malloc或mmap）分配的内存，而不是通过编程语言的内存分配机制（如Java的堆内存）。使用堆外内存可以更好地控制内存使用，避免垃圾收集器的干扰，从而提高性能。

在StarRocks的堆外表内存布局中，数据按列存储，且每一列的数据在内存中是连续的。不同的数据列存储在堆外内存的不同位置。此外，为了处理数据中的空值，引入了空值指示列（null indicator columns）。还有一个元数据列（meta column），用于保存不同数据列的内存地址、空值指示列的内存地址和行数。

#### 具体的内存布局如下：

元数据列布局：元数据列的起始地址包含了行数，每个固定长度列的空值指示列和数据列的起始地址，以及每个可变长度列的空值指示列、偏移列和数据列的起始地址。

空值指示列布局：空值指示列的起始地址包含了一系列1字节的布尔值，每个布尔值对应一行数据，表示该行的字段是否为空。

数据列布局：
对于固定长度列（如BOOLEAN、INT、LONG），使用一级索引寻址方法。通过元数据列获取数据列的起始地址，然后使用该地址直接读取固定长度的数据。
对于可变长度列（如STRING、DECIMAL），使用二级索引寻址方法。首先通过元数据列获取数据列的起始地址，然后使用偏移列在特定行索引处获取字段的起始内存地址，通过当前行和下一行的偏移计算字段长度，最后使用字段的起始地址和长度读取可变长度的数据。
这种内存布局优化了数据访问的效率，尤其是在处理大型数据集时，可以减少内存的分配和复制，提高查询和数据处理的性能。此外，由于数据是按列存储的，因此可以更有效地进行列级别的压缩和编码，从而降低存储成本并提高I/O效率。

#### 示意图
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
3. 查询并发
并发和每个查询的扇出有关。如果基于主键查询，并且表的分桶是以主键来划分到多台机器，那么假如每个be支持1000个并发，那么10个be就可以支持1w的并发。如果不是以主键为分桶，那么每个查询可能都要查询多个be，那么你无论增加多少be，都最多只能达到1个be的并发，比如1000.并且因为随着be的个数增多，网络扇入扇出增多，其实可能连1个be的1000都达不到了，网络有损耗。如果非主键，还会降低速度.如果希望并发度很高，尽量把查询的键作为主键，然后以主键多分桶，分布到多个be上。如果是查询非主键，尽量把主键作为一些过滤，如果有分区，也要尽量带上分区条件，尽量减少要查询的be数。如果是非主键，可以试一下inverted index，比bloomfilter要好。


性能：如果是 JDBC Catalog 的话是要依赖原库的性能的，Hive 的话主要还是 iops 以及网络 io

```
catalog Hive 源码

```

## 导入导出 connector
### 概览
![image](https://github.com/user-attachments/assets/4073aca5-4725-436b-90e8-9c7308207376)

### 导入参数控制
#### FE
| 参数	| 描述 |
|---|---|
max_load_timeout_second<br>min_load_timeout_second | 导入超时时间的最大、最小取值范围，均以秒为单位。默认的最大超时时间为3天，最小超时时间为1秒。您自定义的导入超时时间不可超过该范围。该参数通用于所有类型的导入任务。
desired_max_waiting_jobs | 等待队列可以容纳的最多导入任务数目，默认值为100。<br>例如，FE中处于PENDING状态（即等待执行）的导入任务数目达到该值，则新的导入请求会被拒绝。此配置仅对异步执行的导入有效，如果处于等待状态的异步导入任务数达到限额，则后续创建导入的请求会被拒绝。
max_running_txn_num_per_db |每个数据库中正在运行的导入任务的最大个数（不区分导入类型、统一计数），默认值为100。<br>当数据库中正在运行的导入任务超过最大值时，后续的导入任务不会被执行。如果是同步作业，则作业会被拒绝；如果是异步作业，则作业会在队列中等待。
label_keep_max_second	| 导入任务记录的保留时间。已经完成的（FINISHED或CANCELLED）导入任务记录会在StarRocks系统中保留一段时间，时间长短则由此参数决定。参数默认值为3天。该参数通用于所有类型的导入任务。


#### BE

| 参数| 描述|
|---|---|
push_write_mbytes_per_sec |	BE上单个Tablet的写入速度限制。默认值是10，即10MB/s。<br>根据Schema以及系统的不同，通常BE对单个Tablet的最大写入速度大约在10~30MB/s之间。您可以适当调整该参数来控制导入速度。
write_buffer_size|	导入数据在BE上会先写入到一个内存块，当该内存块达到阈值后才会写回磁盘。默认值为100 MB。<br>过小的阈值可能导致BE上存在大量的小文件。您可以适当提高该阈值减少文件数量。但过大的阈值可能导致RPC超时，详细请参见参数tablet_writer_rpc_timeout_sec。
tablet_writer_rpc_timeout_sec	|导入过程中，发送一个Batch（1024行）的RPC超时时间。默认为600秒。<br>因为该RPC可能涉及多个分片内存块的写盘操作，所以可能会因为写盘导致RPC超时，可以适当调整超时时间来减少超时错误（例如send batch fail）。同时，如果调大参数write_buffer_size，则tablet_writer_rpc_timeout_sec参数也需要适当调大。
streaming_load_rpc_max_alive_time_sec|	在导入过程中，StarRocks会为每个Tablet开启一个Writer，用于接收数据并写入。该参数指定了Writer的等待超时时间。默认为600秒。<br>如果在参数指定时间内Writer没有收到任何数据，则Writer会被自动销毁。当系统处理速度较慢时，Writer可能长时间接收不到下一批数据，导致导入报错TabletWriter add batch with unknown id。此时可适当调大该参数。
load_process_max_memory_limit_percent	|分别为最大内存和最大内存百分比，限制了单个BE上可用于导入任务的内存上限。系统会在两个参数中取较小者，作为最终的BE导入任务内存使用上限。<br>  1. load_process_max_memory_limit_percent：表示对BE总内存限制的百分比。默认为80。总内存限制mem_limit默认为80%，表示对物理内存的百分比。即假设物理内存为M，则默认导入内存限制为M * 80% * 80%。<br> 2. load_process_max_memory_limit_bytes：默认为100 GB。

### stream load
代码集成示例
Java开发Stream Load，详情请参见 [stream_load](https://github.com/StarRocks/demo/tree/master/MiscDemo/stream_load)。

Spark集成Stream Load，详情请参见 [flink/spark](https://github.com/StarRocks/demo/blob/master/docs/05_flinkConnector_Bean2StarRocks.md)。

#### java demo
```java
import org.apache.commons.codec.binary.Base64;
import org.apache.http.HttpHeaders;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpPut;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.DefaultRedirectStrategy;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
/**
 * This class is a java demo for starrocks stream load
 *
 * The pom.xml dependency:
 *
 *         <dependency>
 *             <groupId>org.apache.httpcomponents</groupId>
 *             <artifactId>httpclient</artifactId>
 *             <version>4.5.3</version>
 *         </dependency>
 *
 * How to use:
 *
 * 1 create a table in starrocks with any mysql client
 *
 * CREATE TABLE `stream_test` (
 *   `id` bigint(20) COMMENT "",
 *   `id2` bigint(20) COMMENT "",
 *   `username` varchar(32) COMMENT ""
 * ) ENGINE=OLAP
 * DUPLICATE KEY(`id`)
 * DISTRIBUTED BY HASH(`id`) BUCKETS 20;
 *
 *
 * 2 change the StarRocks cluster, db, user config in this class
 *
 * 3 run this class, you should see the following output:
 *
 * {
 *     "TxnId": 27,
 *     "Label": "39c25a5c-7000-496e-a98e-348a264c81de",
 *     "Status": "Success",
 *     "Message": "OK",
 *     "NumberTotalRows": 10,
 *     "NumberLoadedRows": 10,
 *     "NumberFilteredRows": 0,
 *     "NumberUnselectedRows": 0,
 *     "LoadBytes": 50,
 *     "LoadTimeMs": 151
 * }
 *
 * Attention:
 *
 * 1 wrong dependency version(such as 4.4) of httpclient may cause shaded.org.apache.http.ProtocolException
 *   Caused by: shaded.org.apache.http.ProtocolException: Content-Length header already present
 *     at shaded.org.apache.http.protocol.RequestContent.process(RequestContent.java:96)
 *     at shaded.org.apache.http.protocol.ImmutableHttpProcessor.process(ImmutableHttpProcessor.java:132)
 *     at shaded.org.apache.http.impl.execchain.ProtocolExec.execute(ProtocolExec.java:182)
 *     at shaded.org.apache.http.impl.execchain.RetryExec.execute(RetryExec.java:88)
 *     at shaded.org.apache.http.impl.execchain.RedirectExec.execute(RedirectExec.java:110)
 *     at shaded.org.apache.http.impl.client.InternalHttpClient.doExecute(InternalHttpClient.java:184)
 *
 *2 run this class more than once, the status code for http response is still ok, and you will see
 *  the following output:
 *
 * {
 *     "TxnId": -1,
 *     "Label": "39c25a5c-7000-496e-a98e-348a264c81de",
 *     "Status": "Label Already Exists",
 *     "ExistingJobStatus": "FINISHED",
 *     "Message": "Label [39c25a5c-7000-496e-a98e-348a264c81de"] has already been used.",
 *     "NumberTotalRows": 0,
 *     "NumberLoadedRows": 0,
 *     "NumberFilteredRows": 0,
 *     "NumberUnselectedRows": 0,
 *     "LoadBytes": 0,
 *     "LoadTimeMs": 0
 * }
 * 3 when the response statusCode is 200, that doesn't mean your stream load is ok, there may be still
 *   some stream problem unless you see the output with 'ok' message
 */
public class StarRocksStreamLoad {
    private final static String STARROCKS_HOST = "xxx.com";
    private final static String STARROCKS_DB = "test";
    private final static String STARROCKS_TABLE = "stream_test";
    private final static String STARROCKS_USER = "root";
    private final static String STARROCKS_PASSWORD = "xxx";
    private final static int STARROCKS_HTTP_PORT = 8030;

    private void sendData(String content) throws Exception {
        final String loadUrl = String.format("http://%s:%s/api/%s/%s/_stream_load",
                STARROCKS_HOST,
                STARROCKS_HTTP_PORT,
                STARROCKS_DB,
                STARROCKS_TABLE);

        final HttpClientBuilder httpClientBuilder = HttpClients
                .custom()
                .setRedirectStrategy(new DefaultRedirectStrategy() {
                    @Override
                    protected boolean isRedirectable(String method) {
                        return true;
                    }
                });

        try (CloseableHttpClient client = httpClientBuilder.build()) {
            HttpPut put = new HttpPut(loadUrl);
            StringEntity entity = new StringEntity(content, "UTF-8");
            put.setHeader(HttpHeaders.EXPECT, "100-continue");
            put.setHeader(HttpHeaders.AUTHORIZATION, basicAuthHeader(STARROCKS_USER, STARROCKS_PASSWORD));
            // the label header is optional, not necessary
            // use label header can ensure at most once semantics
            put.setHeader("label", "39c25a5c-7000-496e-a98e-348a264c81de");
            put.setEntity(entity);

            try (CloseableHttpResponse response = client.execute(put)) {
                String loadResult = "";
                if (response.getEntity() != null) {
                    loadResult = EntityUtils.toString(response.getEntity());
                }
                final int statusCode = response.getStatusLine().getStatusCode();
                // statusCode 200 just indicates that starrocks be service is ok, not stream load
                // you should see the output content to find whether stream load is success
                if (statusCode != 200) {
                    throw new IOException(
                            String.format("Stream load failed, statusCode=%s load result=%s", statusCode, loadResult));
                }

                System.out.println(loadResult);
            }
        }
    }

    private String basicAuthHeader(String username, String password) {
        final String tobeEncode = username + ":" + password;
        byte[] encoded = Base64.encodeBase64(tobeEncode.getBytes(StandardCharsets.UTF_8));
        return "Basic " + new String(encoded);
    }

    public static void main(String[] args) throws Exception {
        int id1 = 1;
        int id2 = 10;
        String id3 = "Simon";
        int rowNumber = 10;
        String oneRow = id1 + "\t" + id2 + "\t" + id3 + "\n";

        StringBuilder stringBuilder = new StringBuilder();
        for (int i = 0; i < rowNumber; i++) {
            stringBuilder.append(oneRow);
        }
        
        stringBuilder.deleteCharAt(stringBuilder.length() - 1);

        String loadData = stringBuilder.toString();
        StarRocksStreamLoad starrocksStreamLoad = new StarRocksStreamLoad();
        starrocksStreamLoad.sendData(loadData);
    }
}
```

```
package app.label;

import app.label.function.StarRocksProcess;
import com.alibaba.fastjson.JSONObject;
import com.starrocks.connector.flink.StarRocksSink;
import com.starrocks.connector.flink.table.sink.StarRocksSinkOptions;
import lombok.extern.slf4j.Slf4j;
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.sink.SinkFunction;
import org.apache.flink.util.Collector;

@Slf4j
public class LabelSinkLocal {
    public static void main(String[] args) throws Exception {
        System.out.println("===> 程序开始启动...");
        
        // 创建执行环境
        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        System.out.println("===> 已创建执行环境");
        
        // 从 Socket 读取数据
        DataStream<String> dataStream = env.socketTextStream("localhost", 9999, "\n")
                .flatMap(new FlatMapFunction<String, JSONObject>() {
                    @Override
                    public void flatMap(String value, Collector<JSONObject> out) throws Exception {
                        try {
                            System.out.println("===> 收到Socket数据: " + value);
                            JSONObject jsonObject = JSONObject.parseObject(value);
                            System.out.println("===> 解析为JSON: " + jsonObject);
                            out.collect(jsonObject);
                        } catch (Exception e) {
                            System.out.println("===> 解析JSON失败: " + e.getMessage());
                            e.printStackTrace();
                        }
                    }
                })
                .uid("flatMap").name("flatMap")
                .keyBy((KeySelector<JSONObject, Long>) json -> {
                    Long memberId = json.getLong("member_id");
                    System.out.println("===> 按member_id分组: " + memberId);
                    return memberId;
                })
                .process(new StarRocksProcess())
                .uid("process").name("process");

        // 强制增加一个打印算子，确保数据被输出到控制台
        dataStream.print("数据输出");

        StarRocksSinkOptions options = StarRocksSinkOptions.builder()
                .withProperty("jdbc-url", "jdbc:mysql://127.0.0.1:9030")
                .withProperty("load-url", "127.0.0.1:8030")
                .withProperty("database-name", "test")
                .withProperty("table-name", "ads_user_profile")
                .withProperty("username", "root")
                .withProperty("password", "root")
                .withProperty("sink.semantic", "at-least-once")
                .withProperty("sink.version", "V2")
                .withProperty("sink.label-prefix", "test-")
                .withProperty("sink.properties.format", "json")
                .withProperty("sink.buffer-flush.interval-ms", "1000")
                .withProperty("sink.buffer-flush.max-bytes", "67108864")
                .withProperty("sink.properties.partial_update", "true")
                .withProperty("sink.properties.partial_update_mode", "row")
                .withProperty("sink.properties.strip_outer_array", "true")
                .withProperty("sink.properties.columns", "member_id, user_right_type_s, card_country_s")
                .build();

        System.out.println("===> StarRocks sink配置完成");
        log.info("StarRocks sink配置: {}", options);

        // 打印一下输出的数据，便于调试
        DataStream<String> loggedStream = dataStream.map(data -> {
            System.out.println("===> 即将写入StarRocks的数据: " + data);
            log.info("即将写入StarRocks的数据: {}", data);
            return data;
        }).name("debug-logging").uid("debug-logging");

//        String[] records = new String[]{
//                "{\"id\":1, \"name\":\"starrocks-json\", \"score\":100}",
//                "{\"id\":2, \"name\":\"flink-json\", \"score\":100}",
//        };
//        DataStream<String> source = env.fromElements(records);

/**
 * Configure the Flink connector with the required properties.
 * You also need to add properties "sink.properties.format" and "sink.properties.strip_outer_array"
 * to tell the Flink connector the input records are JSON-format and to strip the outermost array structure.
 */

//        StarRocksSinkOptions options = StarRocksSinkOptions.builder()
//                .withProperty("jdbc-url", "jdbc:mysql://127.0.0.1:9030")
//                .withProperty("load-url", "127.0.0.1:8030")
//                .withProperty("database-name", "test")
//                .withProperty("table-name", "score_board")
//                .withProperty("username", "root")
//                .withProperty("password", "root")
//                .withProperty("sink.properties.format", "json")
//                .withProperty("sink.properties.strip_outer_array", "true")
//                .withProperty("sink.connect.timeout-ms", "10000")
//                .withProperty("sink.properties.timeout", "600")
//                .withProperty("sink.max-retries", "3")
//                .withProperty("sink.properties.strict_mode", "false")
//                .build();
// Create the sink with the options.
//        SinkFunction<String> starRockSink = StarRocksSink.sink(options);
//        source.addSink(starRockSink);

        loggedStream.addSink(StarRocksSink.sink(options))
                .uid("sink").name("sink");

        System.out.println("===> Flink作业已配置完成，开始执行...");
        log.info("Flink作业已配置完成，开始执行...");
        env.execute("localTest");
    }
}

```

```resource/log4j.properties
# Root logger option
log4j.rootLogger=INFO, stdout, file

# Direct log messages to stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.Target=System.out
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n

# Direct log messages to a log file
log4j.appender.file=org.apache.log4j.RollingFileAppender
log4j.appender.file.File=./logs/application.log
log4j.appender.file.MaxFileSize=10MB
log4j.appender.file.MaxBackupIndex=10
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n

# Set specific logger levels
log4j.logger.app.label=DEBUG
log4j.logger.com.starrocks=INFO 
```

### 物化视图
刷新核心逻辑：mv会维护一个visiblemap，记录刷新过哪些分区；每次调度（周期/手动/自动）时候，检查哪些分区变更了，就会触发mv的刷新。
当前是分区级别全量刷新，目前看调度频繁对刷新压力很大的 。所以后续可以做成级联刷新(构造Task DAG)或者增量来解决比较好。

