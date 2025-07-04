## 一、HIVE

### over中的partition by与group by

```sql
name	orderdate	cost
jack	2017-01-01	10
jack	2017-02-03	23
jack	2017-01-05	46
jack	2017-04-06	42
jack	2017-01-08	55
mart	2017-04-08	62
mart	2017-04-09	68
mart	2017-04-11	75
mart	2017-04-13	94
=======================gruop by=======================
select
    name,
    count(*)
from
    overdemo
where
    date_format(orderdate,'yyyy-MM')='2017-04'
group by
    name;
--------结果---------
name	_c1
jack	1
mart	4
=======================partition by========================
select
    name,
    count(*) over(partition by name)
from
    overdemo
where
    date_format(orderdate,'yyyy-MM')='2017-04';
--------结果---------
name	count_window_0
jack	1
mart	4
mart	4
mart	4
mart	4
```

* 差别：partition by 将所有符合过滤条件的列分组选中，故group by 可看成partition by 的去重版
* 执行顺序：group by -> over()，故group by分组去重，再partition by结合聚合函数(count/sum)分区计算

```sql
=====同时有group by name和over(partition by name)=====
select
    name,
    count(*) over(partition by name)
from
    overdemo
where
    date_format(orderdate,'yyyy-MM')='2017-04'
group by 
    name;
---------结果--------
name	count_window_0
jack	1
mart	1
=====同时有group by name和over()=====over()无参数时结合聚合函数对全量数据进行运算
select
    name,
    count(*) over()
from
    overdemo
where
    date_format(orderdate,'yyyy-MM')='2017-04'
group by
    name;
----------结果-------
name	count_window_0
mart	2
jack	2
```

* 当over()无参时，结合聚合函数 count(*)，对得到的数据进行计数，得到的结果=2，并且会在mart和Jack后显示2（因为没有进行partition by name，mart和Jack是在同一个分区的）*
* *②当over(partition by name)之后，group by得到的数据会再次进行分区，分为mart和jack，两个分区中进行count(*)计算，都只有1个，所以在两个分区中分别显示为1
* 此时，再解释上面只使用over(partition by name)时，为什么mart会显示4次，而且count(*）也是4。因为符合过滤条件的mart有4条，都被partition by分到了一个分区，所以count(*)的结果就是4，而且over(partition by name)并不会去重，所以显示了4次。
* 综上，over(partition by)可以视为group by的升级版：①会保留所有进入分区的数据，不去重（没有数据损失）；②在group by之后执行，可以进行更为复杂个性化的操作

## 二、spark

### Spark为什么比MR快

> 1、Spark基于内存计算

```
Spark vs MapReduce ≠ 内存 vs 磁盘

其实Spark和MapReduce的计算都发生在内存中，区别在于：

MapReduce通常需要将计算的中间结果写入磁盘，然后还要读取磁盘，从而导致了频繁的磁盘IO。

Spark则不需要将计算的中间结果写入磁盘，这得益于Spark的RDD（弹性分布式数据集，很强大）和DAG（有向无环图），其中DAG记录了job的stage以及在job执行过程中父RDD和子RDD之间的依赖关系。中间结果能够以RDD的形式存放在内存中，且能够从DAG中恢复，大大减少了磁盘IO。
```

> 2、多进程模型 vs 多线程模型的区别

```
MapReduce采用了多进程模型，而Spark采用了多线程模型。多进程模型的好处是便于细粒度控制每个任务占用的资源，但每次任务的启动都会消耗一定的启动时间。就是说MapReduce的Map Task和Reduce Task是进程级别的，而Spark Task则是基于线程模型的，就是说mapreduce 中的 map 和 reduce 都是 jvm 进程，每次启动都需要重新申请资源，消耗了不必要的时间（假设容器启动时间大概1s，如果有1200个block，那么单独启动map进程事件就需要20分钟）
Spark则是通过复用线程池中的线程来减少启动、关闭task所需要的开销。（多线程模型也有缺点，由于同节点上所有任务运行在一个进程中，因此，会出现严重的资源争用，难以细粒度控制每个任务占用资源）
```



### Yarn Container



### mapreduce on yarn ：

* Application Master 向Resource Manager申请 Container ， 每个MapTask/ReduceTask 申请一个Container。

> **每一个Task就是一个进程**。也是一个Container 资源
>
> 进程的名字叫做：**YarnChild**
>
> MapTask/ReduceTask 的资源申请是Container资源范围内。
>
> ## 那么：对于 Container 里面的task是单线程在运行的?

### spark on hive

```
```

![image-20220119151723987](../Library/Application Support/typora-user-images/image-20220119151723987.png)

### spark on yarn :
* Application Master 向Resource Manager申请 Container ， 每个Executor申请一个Container。

> **每一个Executor是一个JVM进程**。也是一个Container 资源
>
> 在spark on yarn 中 Executor 的进程 名字叫 **CoarseGrainedExecutorBackend** 类似于Hadoop MapReduce中的YarnChild， <u>一个CoarseGrainedExecutorBackend进程</u> 有且仅有一个 **<u>executor对象</u>**，它负责将Task包装成taskRunner，并从线程池中抽取出一个空闲**线程**运行**Task**。
>
> 一个Yarn的container就是一个Spark的CoarseGrainedExecutorBackend，也就是常说的Spark的Executor进程，每个executor可以并行执行多个task。CoarseGrainedExecutorBackend是Executor的RPC endpoint服务，具体执行task是CoarseGrainedExecutorBackend持有的Executor对象的launchTask方法启动.
>
> Executor 的资源申请是Container资源范围内。



* 在 Executor 中并行执行多个 Task

> 把executor的jvm进程看做task执行池，每个executor最大有 spark.executor.cores / spark.task.cpus 个执行槽( solt )， 一个执行槽可以同时执行一个task。
>
> – spark.task.cpus 是指每个任务需要的 CPU 核数， 大部分应用使用默认值 1
> – spark.executor.cores 是指每个executor的核数。



* Application --> 多个job --> 多个stage --> 多个task

> 1.Spark的**driver端**只是用来**请求资源获取资源**的，**executor端**是用来**执行代码程序**，=>**application在driver端**，**job 、stage 、task在executor端。**
>
> 2.在executor端划分job、划分stage、划分task。
>
> **job            :     ** 程序遇见    **action算子**    划分一次，
>
> **stage        :     ** 每个 job遇见     shuffle（或者宽依赖）  划分一次，
>
> **task数量  :**       每个stage中    **最后一个rdd的分区(分片)数**    就是。



### YarnCluster-Spark 执行流程：

![image-20210507092846530](C:\Users\hason\AppData\Roaming\Typora\typora-user-images\image-20210507092846530.png)



### SparkStreaming

##### 数据源API：

> ReceiverAPI：需专门Executor接受数据，发送给其他处理数据的Executor
>
> * 在速度上 :  接收数据  >>  处理数据 的Executor，导致计算节点内存溢出
>
> DirectAPI：由计算Executor 主动消费kafka数据，速度自身控制

### spark-submit常用参数:

> --master  指定任务提交到哪个资源调度器
> 		local/local[N]/local[*] 本地执行
> 		spark://master主机名1:7077,master主机名2:7077,. 提交到spark内置的调度器执行
> 		yarn  提交到yarn执行
> 	--deploy-mode 指定部署模式[client/cluster]
> 		client模式: 不管是提交到standalone还是yarn,Driver所在的位置都在Spark-Submit进程中
> 		cluster模式：
> 			如果是任务提交到standalone中，Driver在其中一个Worker中。
> 			如果任务是提交到yarn中，Driver在ApplicationMaster所在进程中
> 	--class 指定要执行的主类的全类名[带有main方法的object]
> 	--driver-memory 指定Driver内存大小
> 	--executor-memory 指定每个executor内存大小
> 	--total-executor-cores 指定所有的executor cpu的总核数 [仅用于standalone模式]
> 	--executor-cores 指定每个executor cpu核数[仅用于 standalone yarn模式]
> 	--num-executors 指定executor的个数[仅用于yarn模式]
> 	--queue 指定当前任务提交到哪个队列中[仅用于yarn模式]
>
> =================
>
> bin/spark-submit \
> --master yarn \ --指定主服务器地址
> --deploy-mode cluster \  --运行模式
> --num-executors 80 \  每台机器3-12个
> --driver-memory 6g \
> --executor-memory 6g \  --总内存给数据量的8倍以上
> --executor-cores 3 \  --每个exe给1-2个
> --queue root.default \  --队列
> --class com.myproject.spark.WordCount \  --主类名
> --conf spark.yarn.executor.memoryOverhead=2048 \
> --conf spark.core.connect.ack.wait.timeout=300 \
> /usr/local/spark/spark.jar

### 常规性能调优

#### 调优1：RDD持久化

> 一般有RDD复用2次及以上，最好cache，因为不用的话 需要cpu资源计算，cache的话用的内存

#### 调优2：并行度调节

>  spark默认并行度200
>
> spark推荐，**task并行度**数量应该设置为**spark作业总CPUcore数量**的**2-3倍**
>
> set("spark.default.parallelism,"500")

#### 调优3：广播大变量

> 广播变量1-2G

#### 调优4：Kryo序列化

> 从spark2.0开始，简单类型、简单类型数组、字符串类型的Shuffle RDDs
>
> 以上 已经默认使用Kryo序列化

``` scala
//使用Kryo序列化库，如果要使用Java序列化库，需要把该行屏蔽掉
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer");  
//在Kryo序列化库中注册自定义的类集合，如果要使用Java序列化库，需要把该行屏蔽掉
conf.set("spark.kryo.registrator", "myproject.com.MyKryoRegistrator"); 
```

#### 调优5：调节本地化等待时长

> 数据本地化，当节点资源不足是会迁移到其他节点计算，迁移时间太长不合适；故需要设置迁移级别和迁移超时时间

``` scala
set("spark.locality.wait","6");//秒级别
```

### 算子调优

#### mapPartition/foreachPartition

> 集群资源充足的情况下，可以适当使用，提高效率，当然也容易OOM
>
> foreachPartition 适合 与外部存储建立连接



#### filter与coalesce结合使用

> 过滤后再进行分区缩小，以较少reduceTask数量



repartition 解决sparkSql低并行度问题

> 常规调优对并行度调节策略：设置spark.default.parallelism参数指定并行度，在sparkSql中是没有效果的，只有在 **没有**sparkSql  的stage中生效。
>
> sparkSql的并行度无法手动调节，在sparkSql查询出来的RDD后，使用repratition增大分区数的方式增加Task的数量，并行度也就增加了

#### reduceByKey预聚合

> map端**数据量**变小，减小**磁盘IO和占用**
>
> **下个stage**  **拉取**  的数据量变小，减小**网络IO**
>
> reduce端**缓存**占用减小，**聚合**的数据量减小

### Shuffle调优

#### 调节map端缓冲区大小-32K

> shuffle的map 处理的数据量较大，但map端缓冲区大小**默认32k**，可能会出现map端缓冲区数据频繁溢出磁盘文件，性能降低
>
> 通过调节map缓冲区大小，避免频繁的磁盘IO
>
> spark.shuffle.file.buffer

#### reduce端拉取数据缓冲区调节-默认48m

> spark shuffle过程中，shuffle reduce task 的 shuffle缓冲区(**默认48M**) 决定了reduce task每次缓冲(拉取)的数据量，适当增大可以减少拉取次数/网络传输次数
>
> spark.reducer.maxSizeInFlight

#### reduce端拉取数据重试次数-默认3次

> 网络异常等 导致拉取失败自动重试(**默认3次**)
>
> 对于耗时shuffle作业，建议增加重试次数(60)，避免由于JVM的full gc或者网络异常导致拉取失败
>
> spark.shuffle.io.maxRetries

#### reduce端拉取数据等待间隔-默认5s

> spark.shuffle.io.retryWait

#### SortShuffle->byPass-默认<200

> 当使用SortShuffle时，分区数大于200 ，又不想排序，可以设置byPass阈值
>
> spark.shuffle.sort.bypassMergeThreshold

### JVM调优

fullGC/minorGC 导致JVM线程停止工作

#### 减低cache操作内存占比

#####  1. Spark**静态**内存管理机制，堆内存被划分为了两块，Storage和Execution

> * Storage主要用于缓存RDD数据和broadcast数据。**Storage占系统内存的60%**
>
> * Execution主要用于缓存在shuffle过程中产生的中间数据。**Execution占系统内存的20%，并且两者完全独立。**

> 在一般情况下，Storage的内存都提供给了cache操作，但是如果在某些情况下cache操作内存不是很紧张，而task的算子中创建的对象很多，Execution内存又相对较小，这回导致频繁的minor gc，甚至于频繁的full gc，进而导致Spark频繁的停止工作，性能影响会很大。
>
> 在Spark UI中可以查看每个stage的运行情况，包括每个task的运行时间、gc时间等等，如果发现gc太频繁，时间太长，就可以考虑调节Storage的内存占比，让task执行算子函数式，有更多的内存可以使用。
>
> Storage内存区域可以通过spark.storage.memoryFraction参数进行指定，默认为0.6，即60%，可以逐级向下递减，如代码清单所示：
>
> val conf = new SparkConf().set("spark.storage.memoryFraction", "0.4")

##### 2.Spark统一内存管理机制，堆内存被划分为了两块，Storage和Execution

> Storage主要用于缓存数据，
>
> Execution主要用于缓存在shuffle过程中产生的中间数据，
>
> 两者所组成的内存部分称为统一内存，Storage和Execution各占统一内存的50%，由于动态占用机制的实现，shuffle过程需要的内存过大时，会自动占用Storage的内存区域，因此**无需**手动进行调节。

#### 调节Executor堆外内存

> Executor的堆外内存主要用于程序的共享库、Perm Space、 线程Stack和一些Memory mapping等, 或者类C方式allocate object。

> 有时，如果你的Spark作业处理的数据量非常大，达到几亿的数据量，此时运行Spark作业会时不时地报错，例如shuffle output file cannot find，executor lost，task lost，out of memory等，这可能是Executor的堆外内存不太够用，导致Executor在运行的过程中内存溢出。
>
> stage的task在运行的时候，可能要从一些Executor中去拉取shuffle map output文件，但是Executor可能已经由于内存溢出挂掉了，其关联的BlockManager也没有了，这就可能会报出shuffle output file cannot find，executor lost，task lost，out of memory等错误，此时，就可以考虑调节一下Executor的堆外内存，也就可以避免报错，与此同时，堆外内存调节的比较大的时候，对于性能来讲，也会带来一定的提升。
>
> 默认情况下，Executor堆外内存上限大概为300多MB，在实际的生产环境下，对海量数据进行处理的时候，这里都会出现问题，导致Spark作业反复崩溃，无法运行，此时就会去调节这个参数，到至少1G，甚至于2G、4G。
>
> Executor堆外内存的配置需要在spark-submit脚本里配置，如代码清单所示：
>
> --conf spark.yarn.executor.memoryOverhead=2048
>
> 以上参数配置完成后，会避免掉某些JVM OOM的异常问题，同时，可以提升整体Spark作业的性能。

#### 调节连接等待时长

> 在Spark作业运行过程中，Executor优先从自己本地关联的BlockManager中获取某份数据，如果本地BlockManager没有的话，会通过TransferService远程连接其他节点上Executor的BlockManager来获取数据。
>
> 如果task在运行过程中创建大量对象或者创建的对象较大，会占用大量的内存，这回导致频繁的垃圾回收，但是垃圾回收会导致工作线程全部停止，也就是说，垃圾回收一旦执行，Spark的Executor进程就会停止工作，无法提供相应，此时，由于没有响应，无法建立网络连接，会导致网络连接超时。
>
> 在生产环境下，有时会遇到file not found、file lost这类错误，在这种情况下，很有可能是Executor的BlockManager在拉取数据的时候，无法建立连接，然后超过默认的连接等待时长60s后，宣告数据拉取失败，如果反复尝试都拉取不到数据，可能会导致Spark作业的崩溃。这种情况也可能会导致DAGScheduler反复提交几次stage，TaskScheduler返回提交几次task，大大延长了我们的Spark作业的运行时间。
>
> 此时，可以考虑调节连接的超时时长，连接等待时长需要在spark-submit脚本中进行设置，设置方式如代码清单所示：
>
> --conf spark.core.connection.ack.wait.timeout=300
>
> 调节连接等待时长后，通常可以避免部分的XX文件拉取失败、XX文件lost等报错

### Spark数据倾斜

主要指shuffle过程中出现的数据倾斜问题，是由于不同的key对应的数据量不同导致的不同task所处理的数据量不同的问题。

> 例如，reduce点一共要处理100万条数据，第一个和第二个task分别被分配到了1万条数据，计算5分钟内完成，第三个task分配到了98万数据，此时第三个task可能需要10个小时完成，这使得整个Spark作业需要10个小时才能运行完成，这就是数据倾斜所带来的后果。

注意，要区分开数据倾斜与数据量过量这两种情况，数据倾斜是指少数task被分配了绝大多数的数据，因此少数task运行缓慢；数据过量是指所有task被分配的数据量都很大，相差不多，所有task都运行缓慢。

> **数据倾斜的表现：**
>
> Ø Spark作业的大部分task都执行迅速，只有有限的几个task执行的非常慢，此时可能出现了数据倾斜，作业可以运行，但是运行得非常慢；
>
> Ø Spark作业的大部分task都执行迅速，但是有的task在运行过程中会突然报出OOM，反复执行几次都在某一个task报出OOM错误，此时可能出现了数据倾斜，作业无法正常运行。
>
> **定位数据倾斜问题：**
>
> Ø 查阅代码中的**shuffle算子**，例如reduceByKey、countByKey、groupByKey、join等算子，根据代码逻辑判断此处是否会出现数据倾斜；
>
> Ø 查看Spark作业的**log文件**，log文件对于错误的记录会精确到代码的某一行，可以根据**异常定位**到的**代码位置来**明确错误发生在**第几个stage**，**对应的shuffle算子**是哪一个；

### 倾斜指标：

1. 会员/vip等级(数值越小规模越大)：人数，消费金额、客单价、浏览数。。。
2. 商品分类/类名(生活日用（倾斜）/奢侈品(少))：销售金额、浏览、收藏、点赞、评论；

3. 活动时间维度：双十一是一周的活动，统计每天：销售额、浏览、收藏。。。
4. 地区（广东多，新疆少），（男女-T恤/羽绒服）：统计服饰销量、浏览、点赞。。。

​       

### spark 生产经验

###### 1、java.lang.OutOfMemoryError: GC overhead limit exceeded

原因：数据量太大，内存不够
解决方案：(1)增大spark.executor.memory的值，减小spark.executor.cores
         (2)减少输入数据量，将原来的数据量分几次任务完成，每次读取其中一部分

###### 2、ERROR An error occurred while trying to connect to the Java server (127.0.0.1:57439) Connection refused

原因：(1)节点上运行的container多，每个任务shuffle write到磁盘的量大，导致磁盘满，节点重启
     (2)节点其他服务多，抢占内存资源，NodeManager处于假死状态
解决方案：(1)确保节点没有过多其他服务进程
         (2)扩大磁盘容量
         (3)降低内存可分配量，比如为总内存的90%，可分配内存少了，并发任务数就少了，出现问题概率降低
         (4)增大NodeManager的堆内存

###### 3、org.apache.spark.shuffle.FetchFailedException: Failed to connect to /9.4.36.40:7337

背景：shuffle过程包括shuffle read和shuffle write两个过程。对于spark on yarn，shuffle write是container写数据到本地磁盘(路径由core-site.xml中hadoop.tmp.dir指定)过程；
     shuffle read是container请求external shuffle服务获取数据过程，external shuffle是NodeManager进程中的一个服务，默认端口是7337，或者通过spark.shuffle.service.port指定。
定位过程：拉取任务运行日志，查看container日志；查看对应ip上NodeManager进程运行日志，路径由yarn-env.sh中YARN_LOG_DIR指定
原因：container请求NodeManager上external shufflle服务，不能正常connect，说明NodeManager可能挂掉了，原因可能是(1)节点上运行的container多，每个任务shuffle write到磁盘的量大，导致磁盘满，节点重启 (2)节点其他服务多，抢占内存资源，NodeManager处于假死状态
解决方案：(1)确保节点没有过多其他服务进程
         (2)扩大磁盘容量
         (3)降低内存可分配量，比如为总内存的90%，可分配内存少了，并发任务数就少了，出现问题概率降低
         (4)增大NodeManager的堆内存

###### 4、org.apache.spark.shuffle.FetchFailedException: Connection from /xxx:7337 closed

背景：shuffle过程包括shuffle read和shuffle write两个过程。对于spark on yarn，shuffle write是container写数据到本地磁盘(路径由core-site.xml中hadoop.tmp.dir指定)过程；
     shuffle read是container请求external shuffle服务获取数据过程，external shuffle是NodeManager进程中的一个服务，默认端口是7337，或者通过spark.shuffle.service.port指定。
定位过程：拉取任务运行日志，查看container日志；查看对应ip上NodeManager进程运行日志，路径由yarn-env.sh中YARN_LOG_DIR指定
原因：container已经连接上NodeManager上external shufflle服务，原因可能是
     (1)external shuffle服务正常，但在规定时间内将数据返回给container，可能是中间数据量大且文件数多，external shuffle服务搜索数据过程久，最终导致containter误认为connection dead，因此抛出xxx:7337 closed了异常
     (2)NameNode进程不正常
解决方案：针对原因(1)，调大spark.network.timeout值，如1800s，此参数可以在spark-defaults.conf设置，对所有任务都生效；也可以单个任务设置
        针对原因(2)，参考org.apache.spark.shuffle.FetchFailedException: Failed to connect to /9.4.36.40:7337的解决方案

###### 5、org.apache.spark.shuffle.FetchFailedException: Failed to send RPC XXX to /xxx:7337:java.nio.channels.ColsedChannelException

背景：shuffle过程包括shuffle read和shuffle write两个过程。对于spark on yarn，shuffle write是container写数据到本地磁盘(路径由core-site.xml中hadoop.tmp.dir指定)过程；
     shuffle read是container请求external shuffle服务获取数据过程，external shuffle是NodeManager进程中的一个服务，默认端口是7337，或者通过spark.shuffle.service.port指定。
定位过程：拉取任务运行日志，查看container日志；查看对应ip上NodeManager进程运行日志，路径由yarn-env.sh中YARN_LOG_DIR指定
原因：external shuffle服务将数据发送给container时，发现container已经关闭连接，出现该异常应该和org.apache.spark.shuffle.FetchFailedException: Connection from /xxx:7337 closed同时出现
解决方案：参考org.apache.spark.shuffle.FetchFailedException: Connection from /xxx:7337 closed的解决方案

###### 6、spark任务中stage有retry

原因：下一个stage获取上一个stage没有获取到全部输出结果，只获取到部分结果，对于没有获取的输出结果retry stage以产出缺失的结果
     (1)部分输出结果确实已经丢失
     (2)部分输出结果没有丢失，只是下一个stage获取结果超时，误认为输出结果丢失
解决方案：针对原因(1)，查看进程是否正常，查看机器资源是否正常，比如磁盘是否满或者其他
         针对原因(2)，调大超时时间，如调大spark.network.timeout值

###### 7、Final app status: FAILED, exitCode: 11, (reason: Max number of executor failures (200) reached)

原因：executor失败重试次数达到阈值
解决方案：1.调整运行参数，减少executor失败次数
        2.调整spark.yarn.max.executor.failures的值，可在spark-defaults.conf中调整
确定方式：在日志中搜索"Final app status:"，确定原因，在日志统计"Container marked as failed:"出现次数





## 三、Kafka

### **kafka写数据：**

* 顺序写，往磁盘上写数据时，就是追加数据，没有随机写的操作。**经验**: 如果一个服务器磁盘达到一定的个数，磁盘也达到一定转数，往磁盘里面顺序写（追加写）数据的速度和写内存的速度差不多`生产者生产消息，经过kafka服务先写到os cache 内存中，然后经过sync顺序写到磁盘上`



### **消费者读取数据流程：**

* 普通

> 1. 消费者发送请求给kafka服务
> 2. kafka服务去os cache缓存读取数据（缓存没有就去磁盘读取数据）
> 3. 从磁盘读取了数据到os cache缓存中
> 4. os cache复制数据到kafka应用程序中
> 5. kafka将数据（复制）发送到socket cache中
> 6. socket cache通过网卡传输给消费者

* 零拷贝（页缓存：os cache,有os管理）

>1.消费者发送请求给kafka服务 
>
>2.kafka服务去os cache缓存读取数据（缓存没有就去磁盘读取数据） 
>
>3.从磁盘读取了数据到os cache缓存中 
>
>4.os cache直接将数据发送给网卡 
>
>5.通过网卡将数据传输给消费者

![image-20210507093741624](C:\Users\hason\AppData\Roaming\Typora\typora-user-images\image-20210507093741624.png)

> **kafka 数据直接持久化到pageCache中**
>
> 好处：
>
> * I/O Scheduler 会将**连续的小块**写**组装成大块的物理写** 从而提升性能
>
> * I/O Scheduler  会尝试将一些写操作重新**按顺序排序**，从而减少磁头的移动时间
> * 充分利用所有的空间内存(非JVM内存)，若使用引用层Cache(JVM堆内存)会增加GC负担
> * 读操作可直接在PageCache内进行，若**消费和生产速度相当**，甚至不需要通过物理磁盘(直接通过Page Cache )交换数据
> * 若进程重启，JVM内的Cache会失效，但PageCache仍可用【宕机导致PageCache数据丢失，但**Repatition机制解决**。若为保数据不丢而强制刷写磁盘，反而会降低性能】
>
> 

### **Kafka日志分段保存**

Kafka中一个主题，一般会设置分区；比如创建了一个`topic_a`，然后创建的时候指定了这个主题有三个分区。其实在三台服务器上，会创建三个目录。服务器1（kafka1）创建目录topic_a-0:。目录下面是我们文件（存储数据），kafka数据就是message，数据存储在log文件里。.log结尾的就是日志文件，在kafka中把数据文件就叫做日志文件 。**一个分区下面默认有n多个日志文件（分段存储），一个日志文件默认1G**。

![image-20210507093928761](C:\Users\hason\AppData\Roaming\Typora\typora-user-images\image-20210507093928761.png)



### **Kafka二分查找定位数据**

消息体对应offset ->  在对应  index文件(稀疏索引4k一个) -> 定位log文件对应的消息体

![image-20210507094404722](C:\Users\hason\AppData\Roaming\Typora\typora-user-images\image-20210507094404722.png)



### **副本冗余保证高可用**

> 0.8以前没有副本机制
>
> leader：1.读写操作；2.维护ISR列表
>
> ​	ISR:{a,b,c}若一follower分区 超过10秒 没向leader partition拉取数据，这个分区就从ISR列表里移除。
>
> follower：向leader同步数据

### 框架总结

> **高可用：**多副本机制
>
> **高并发：**网络架构设计 三层架构：多selector -> 多线程 -> 队列的设计（NIO） 
>
> **高性能：**
>
> A 写数据：
>
> 1. 把数据先写入到OS Cache
> 2. 写到磁盘上面是顺序写，性能很高
>
> B 读数据：
>
> 1. 根据稀疏索引，快速定位到要消费的数据
> 2. 零拷贝机制 减少数据的拷贝 减少了应用程序与操作系统上下文切换

### 专有名词

**QPS（Query Per Second）：**每秒请求数，就是说服务器在一秒的时间内处理了多少个请求。

 **吞吐量(Throughput) （TPS）：**吞吐量是指系统在单位时间内处理请求的数量。

**响应时间(RT) ：**响应时间是指系统对请求作出响应的时间。

**并发用户数 ：**并发用户数是指系统可以同时承载的正常使用系统功能的用户的数量。

### 生产环境分析

#### 

#### 需求分析：

> pqs/day：每天10亿，高峰期3h处理6亿请求：qps6亿 / 3h = 5.5万/s pqs
>
> 数据量：10亿 * 50kb/条 * 2副本 * 3天 = 276T
>
> 注： (50kb/条 )我司因在生产端封装了数据，然后多条数据合并，故一个请求才会有这么大

#### 物理机数：

> 评估机器：  按照高峰期 * 4倍 = 20万pqs
>
> 每台机器 承受 4万请求 -> 5台机器

#### 磁盘选择：

> SSD硬盘：性能好，指它 **随机读写性能 ** 较好，但顺序写跟SAS盘差不多，适合**MySql集群**
>
> SAS盘：某方面性能不是很好，但比较便宜。
>
> kafka的理解：就是用的顺序写。所以我们就用普通的【`机械硬盘`】就可以了。
>
> 每台机器：276T / 5台 = 60T/台
>
> 现有配置：11块硬盘/机器 * 7T/块硬盘 = 77 T/机器
>
> **总结：**10亿请求，5台机器，每个机器 11(SAS) * 7T 

#### 内存评估：

> kafka读写流程 都**基于os cache**  => 尽可能多的内存资源要给 os cache
>
> Kafka中核心代码是scala，客户端java。都是**基于jvm** => 部分的内存给jvm
>
> ​	Kafka的设计，没有把很多数据结构都放在jvm里面   =>jvm不需太大内存
>
> **根据经验:>=10G**
>
> * 假设10个请求，拥有100Topic => **总partition数** = 100Topic * 5 partition * 2副本 = 1000partition
>
>   **kafka使用内存/机器：**1000partition  * 1G /.log 文件 * 25%常驻内存  / 5机器 = 50G
>
>   **机器总内存**： kafka(50G) + JVM(10G)+ OS(10G-5G) = 64G  (128G更好)

#### CPU压力评估

> 评估需要多少个cpu (线程依托cpu运行)服务有多少线程ing
>
> 线程多，cpu core少，会导致机器负载高性能差
>
> kafka 启动线程数：
>
> * Acceptor线程 1 ，processor线程 3， 6~9个线程 处理请求线程 8个， 32个线程 定时清理的线程，拉取数据的线程，定时检查ISR列表的机制 等等。
>
> * **所以大概一个Kafka的服务启动起来以后，会有一百多个线程。**
>
> 机器cpu 线程数：
>
> * cpu core = 4个，几十个线程，就肯定把cpu 打满了。
> * cpu core = 8个，轻松支持几十个线程。如是100多，或差不多200个，那么8 个 cpu core是搞不定的。
> * 所以我们这儿建议：CPU core = 16个。如果可以的话，能有32个cpu core 那就最好。
> * **结论**：kafka集群，最低也要给16个cpu core，如果能给到32 cpu core那就更好。2cpu * 8 =16 cpu core 4cpu * 8 = 32 cpu core
>
> **总结：10亿请求，5台物理机，11（SAS） * 7T ，需64G内存（128G更好），需16cpu core（32更好）**

#### 网络需求评估：

> **千兆的网卡（1G/s）**，万兆的网卡（10G/s）
>
> 网络请求流量：5.5w pqs / 5机器 * 50kb/条 * 2副本 = 976m/s
>
> 注： 网卡的带宽是达不到极限的，70%左右

#### 集群规划：

> 主从式的架构：controller -> 通过zk集群来管理整个集群的元数据。
>
> 注：kafka集群 理论上来讲，不应该把kafka的服务与zk的服务安装在一起

### 1.运维：

##### **场景一：topic数据量太大，要增加topic数**

> kafka-topics.sh --alter --zookeeper hdp1:2181,hdp2:2181,hdp3:2181 --partitions 2 --topic test6

##### 场景二：**核心topic增加副本因子**

> 1. 核心业务数据需要增加副本因子 vim test.json脚本，将下面一行json脚本保存

```json
{
	“version”:1,
	“partitions”:[
					{“topic”:“test6”,“partition”:0,“replicas”:[0,1,2]},
					{“topic”:“test6”,“partition”:1,“replicas”:[0,1,2]},
					{“topic”:“test6”,“partition”:2,“replicas”:[0,1,2]}
				]
}
```

> 2. 执行上面json脚本：

```
kafka-reassign-partitions.sh \
--zookeeper hadoop1:2181,hadoop2:2181,hadoop3:2181 \
--reassignment-json-file test.json \
--execute
```

##### 场景三：**负载不均衡的topic，手动迁移**vi topics-to-move.json

```
{“topics”: [{“topic”: “test01”}, {“topic”: “test02”}], “version”: 1} // 把你所有的topic都写在这里
```

> 2. 执行：

```
kafka-reassgin-partitions.sh \
--zookeeper hadoop1:2181,hadoop2:2181,hadoop3:2181 \
--topics-to-move-json-file topics-to-move.json \
--broker-list “5,6” \
--generate
```

> 把你所有的包括新加入的broker机器都写在这里，就会说是把所有的partition均匀的分散在各个broker上，包括新进来的broker此时会生成一个迁移方案，可以保存到一个文件里去：expand-cluster-reassignment.json

```
kafka-reassign-partitions.sh \
--zookeeper hadoop01:2181,hadoop02:2181,hadoop03:2181 \
--reassignment-json-file expand-cluster-reassignment.json \
--execute

kafka-reassign-partitions.sh \
--zookeeper hadoop01:2181,hadoop02:2181,hadoop03:2181 \
--reassignment-json-file expand-cluster-reassignment.json \
--verify
```

> 这种数据迁移操作一定要在晚上低峰的时候来做，因为他会在机器之间迁移数据，非常的占用带宽资源
>
> –generate: 根据给予的Topic列表和Broker列表生成迁移计划。generate并不会真正进行消息迁移，而是将消息迁移计划计算出来，供execute命令使用。
>
> –execute: 根据给予的消息迁移计划进行迁移。
>
> –verify: 检查消息是否已经迁移完成。

##### 场景四：**如果某个broker leader partition过多**

> 正常情况下，我们的leader partition在服务器之间是负载均衡，现在各个业务方可以自行申请创建topic，分区数量都是自动分配和后续动态调整的， kafka本身会自动把leader partition均匀分散在各个机器上，这样可以保证每台机器的读写吞吐量都是均匀的
>
> * 如果某些broker宕机，会导致leader partition过于集中在其他少部分几台broker上， 这会导致少数几台broker的读写请求压力过高，其他宕机的broker重启之后都是follower partition，读写请求很低造成集群负载不均衡 
>
>    **=>**   有一个参数，auto.leader.rebalance.enable，默认是true， 每隔300秒（leader.imbalance.check.interval.seconds）检查leader负载是否平衡
>
>   
>
> * 如果一台broker上的不均衡的leader超过了10%，leader.imbalance.per.broker.percentage(每个broker允许的不平衡的leader的比率) 
>
>    **=>**    就会对这个broker进行选举 配置参数：auto.leader.rebalance.enable 默认是true 。
>
>   ​		leader.imbalance.check.interval.seconds( leader不平衡检查间隔时间)：默认值300

##### 场景五：kafka 常用脚本

#### 1.0.TopicCommand

#### 1.1.Topic创建

> ```
> bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 3 --partitions 3 --topic test
> ```

------

相关可选参数

| 参数                                             | 描述                                                         | 例子                                                         |
| :----------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `--bootstrap-server` 指定kafka服务               | 指定连接到的kafka服务; 如果有这个参数,则 `--zookeeper`可以不需要 | --bootstrap-server localhost:9092                            |
| `--zookeeper`                                    | 弃用, 通过zk的连接方式连接到kafka集群;                       | --zookeeper localhost:2181 或者localhost:2181/kafka          |
| `--replication-factor`                           | 副本数量,注意不能大于broker数量;如果不提供,则会用集群中默认配置 | --replication-factor 3                                       |
| `--partitions`                                   | 分区数量,当创建或者修改topic的时候,用这个来指定分区数;如果创建的时候没有提供参数,则用集群中默认值; 注意如果是修改的时候,分区比之前小会有问题 | --partitions 3                                               |
| `--replica-assignment`                           | 副本分区分配方式;创建topic的时候可以自己指定副本分配情况;    | `--replica-assignment` BrokerId-0:BrokerId-1:BrokerId-2,BrokerId-1:BrokerId-2:BrokerId-0,BrokerId-2:BrokerId-1:BrokerId-0  ; 这个意思是有三个分区和三个副本,对应分配的Broker; 逗号隔开标识分区;冒号隔开表示副本 |
| `--config`<String: name=value>                   | 用来设置topic级别的配置以覆盖默认配置;**只在--create 和--bootstrap-server 同时使用时候生效**; 可以配置的参数列表请看文末附件 | 例如覆盖两个配置 `--config retention.bytes=123455 --config retention.ms=600001` |
| `--command-config` <String: command    文件路径> | 用来配置客户端Admin Client启动配置,**只在--bootstrap-server 同时使用时候生效**; | 例如:设置请求的超时时间 `--command-config config/producer.proterties`; 然后在文件中配置 request.timeout.ms=300000 |

#### 1.2.删除Topic

> ```
> bin/kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic test
> ```

------

支持正则表达式匹配Topic来进行删除,只需要将topic 用双引号包裹起来

**例如: 删除以`create_topic_byhand_zk`为开头的topic;**

> bin/kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic "create_topic_byhand_zk.*"
>
> `.`表示任意匹配除换行符 \n 之外的任何单字符。要匹配 . ，请使用 . 。 `·*·`：匹配前面的子表达式零次或多次。要匹配 * 字符，请使用 *。

`.*` : 任意字符

**删除任意Topic (慎用)**

>   bin/kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic ".*?" 更多的用法请[参考正则表达式](https://www.runoob.com/regexp/regexp-syntax.html)

#### 1.3.Topic分区扩容

**zk方式(不推荐)**

```
>bin/kafka-topics.sh --zookeeper localhost:2181 --alter --topic topic1 --partitions 2
```

**kafka版本 >= 2.2 支持下面方式（推荐）**

**单个Topic扩容**

> ```
> bin/kafka-topics.sh --bootstrap-server broker_host:port --alter --topic test_create_topic1 --partitions 4
> ```

**批量扩容** (将所有正则表达式匹配到的Topic分区扩容到4个)

> ```
> sh bin/kafka-topics.sh --topic ".*?" --bootstrap-server 172.23.248.85:9092 --alter --partitions 4
> ```

`".*?"` 正则表达式的意思是匹配所有; 您可按需匹配

**PS:** 当某个Topic的分区少于指定的分区数时候,他会抛出异常;但是不会影响其他Topic正常进行;

------

相关可选参数

| 参数                   | 描述                                                      | 例子                                                         |
| :--------------------- | :-------------------------------------------------------- | :----------------------------------------------------------- |
| `--replica-assignment` | 副本分区分配方式;创建topic的时候可以自己指定副本分配情况; | `--replica-assignment` BrokerId-0:BrokerId-1:BrokerId-2,BrokerId-1:BrokerId-2:BrokerId-0,BrokerId-2:BrokerId-1:BrokerId-0  ; 这个意思是有三个分区和三个副本,对应分配的Broker; 逗号隔开标识分区;冒号隔开表示副本 |

**PS: 虽然这里配置的是全部的分区副本分配配置,但是正在生效的是新增的分区;**

比如: 以前3分区1副本是这样的

| Broker-1 | Broker-2 | Broker-3 | Broker-4 |
| :------- | :------- | :------- | :------- |
| 0        | 1        | 2        |          |

现在新增一个分区,`--replica-assignment`  2,1,3,4 ; 看这个意思好像是把0，1号分区互相换个Broker

| Broker-1 | Broker-2 | Broker-3 | Broker-4 |      |
| :------- | :------- | :------- | :------- | ---- |
| 1        | 0        | 2        | 3        |      |

但是实际上不会这样做,Controller在处理的时候会把前面3个截掉; 只取新增的分区分配方式,原来的还是不会变

| Broker-1 | Broker-2 | Broker-3 | Broker-4 |      |
| :------- | :------- | :------- | :------- | ---- |
| 0        | 1        | 2        | 3        |      |

#### 1.4.查询Topic描述

**1.查询单个Topic**

> ```
> sh bin/kafka-topics.sh --topic test --bootstrap-server xxxx:9092 --describe --exclude-internal
> ```

**2.批量查询Topic**(正则表达式匹配,下面是查询所有Topic)

> ```
> sh bin/kafka-topics.sh --topic ".*?" --bootstrap-server xxxx:9092 --describe --exclude-internal
> ```

支持正则表达式匹配Topic,只需要将topic 用双引号包裹起来

------

相关可选参数

| 参数                               | 描述                                                         | 例子                              |      |
| :--------------------------------- | :----------------------------------------------------------- | :-------------------------------- | ---- |
| `--bootstrap-server` 指定kafka服务 | 指定连接到的kafka服务; 如果有这个参数,则 `--zookeeper`可以不需要 | --bootstrap-server localhost:9092 |      |
| `--at-min-isr-partitions`          | 查询的时候省略一些计数和配置信息                             | `--at-min-isr-partitions`         |      |
| `--exclude-internal`               | 排除kafka内部topic,比如`__consumer_offsets-*`                | `--exclude-internal`              |      |
| `--topics-with-overrides`          | 仅显示已覆盖配置的主题,也就是单独针对Topic设置的配置覆盖默认配置；不展示分区信息 | `--topics-with-overrides`         |      |

#### 1.5.查询Topic列表

**1.查询所有Topic列表**

> ```
> sh bin/kafka-topics.sh  --bootstrap-server xxxxxx:9092 --list --exclude-internal
> ```

**2.查询匹配Topic列表**(正则表达式)

> 查询`test_create_`开头的所有Topic列表
>
> ```
> sh bin/kafka-topics.sh  --bootstrap-server xxxxxx:9092 --list --exclude-internal  --topic "test_create_.*"
> ```

------

相关可选参数

| 参数                 | 描述                                          | 例子                 |
| :------------------- | :-------------------------------------------- | :------------------- |
| `--exclude-internal` | 排除kafka内部topic,比如`__consumer_offsets-*` | `--exclude-internal` |
| `--topic`            | 可以正则表达式进行匹配,展示topic名称          | `--topic`            |

### 2.ConfigCommand

> Config相关操作; 动态配置可以覆盖默认的静态配置; 

#### 2.1 查询配置

#### Topic配置查询

展示关于Topic的动静态配置

**1.查询单个Topic配置**(只列举动态配置)

> ```bash
> sh bin/kafka-configs.sh --describe  --bootstrap-server xxxxx:9092 --topic test_create_topic
> ```
>
> ```bash
> sh bin/kafka-configs.sh --describe  --bootstrap-server 172.23.248.85:9092 --entity-type topics --entity-name test_create_topic
> ```
>
> ###### **2.查询所有Topic配置**(包括内部Topic)(只列举动态配置)
>
> ```bash
> sh bin/kafka-configs.sh --describe  --bootstrap-server 172.23.248.85:9092 --entity-type topics
> ```

**3.查询Topic的详细配置(动态+静态)**

> 只需要加上一个参数`--all`

##### 4.其他配置/clients/users/brokers/broker-loggers 的查询

> 同理 ；只需要将`--entity-type` 改成对应的类型就行了 (topics/clients/users/brokers/broker-loggers)  

##### 5.查询kafka版本信息

> ```
> sh bin/kafka-configs.sh --describe  --bootstrap-server xxxx:9092 --version
> ```

**<font color=red>所有可配置的动态配置 请看最后面的 \***附件***  部分</font>**

#### 2.2 增删改 配置 `--alter`

**--alter**  

**删除配置**: `--delete-config` k1=v1,k2=v2

**添加/修改配**置: `--add-config`  k1,k2

**选择类型**: `--entity-type` (topics/clients/users/brokers/broker-

```js
                                     loggers)
```

 **类型名称**: `--entity-name`                                         

#### Topic添加/修改动态配置

 `--add-config`

> ```
> sh bin/kafka-configs.sh   --bootstrap-server xxxxx:9092 --alter  --entity-type topics --entity-name test_create_topic1 --add-config file.delete.delay.ms=222222,retention.ms=999999
> ```

#### Topic删除动态配置

```
--delete-config
```

> ```
> sh bin/kafka-configs.sh   --bootstrap-server xxxxx:9092 --alter  --entity-type topics --entity-name test_create_topic1 --delete-config file.delete.delay.ms,retention.ms
> ```

#### 其他配置同理,只需要类型改下`--entity-type`

> 类型有: (topics/clients/users/brokers/broker- loggers)

<font color=red>哪些配置可以修改 请看最后面的附件：**ConfigCommand 的一些可选配置** </font>

### 3.副本扩缩、分区迁移、跨路径迁移 kafka-reassign-partitions

请戳 【kafka运维】副本扩缩容、数据迁移、副本重分配、副本跨路径迁移  (如果点不出来,表示文章暂未发表,请耐心等待)

### 4.Topic的发送kafka-console-producer.sh

**4.1 生产无key消息**

```js
## 生产者
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test --producer.config config/producer.properties
```

 **4.2 生产有key消息**

加上属性`--property parse.key=true`

```js
## 生产者
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test --producer.config config/producer.properties  --property parse.key=true
```

<font color=red>默认消息key与消息value间使用“Tab键”进行分隔，所以消息key以及value中切勿使用转义字符(\t)</font>

------

可选参数

| 参数                         | 值类型          | 说明                                                         | 有效值                                                       |
| :--------------------------- | :-------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| --bootstrap-server           | String          | 要连接的服务器必需(除非指定--broker-list)                    | 如：host1:prot1,host2:prot2                                  |
| --topic                      | String          | (必需)接收消息的主题名称                                     |                                                              |
| --batch-size                 | Integer         | 单个批处理中发送的消息数                                     | 200(默认值)                                                  |
| --compression-codec          | String          | 压缩编解码器                                                 | none、gzip(默认值)snappy、lz4、zstd                          |
| --max-block-ms               | Long            | 在发送请求期间，生产者将阻止的最长时间                       | 60000(默认值)                                                |
| --max-memory-bytes           | Long            | 生产者用来缓冲等待发送到服务器的总内存                       | 33554432(默认值)                                             |
| --max-partition-memory-bytes | Long            | 为分区分配的缓冲区大小                                       | 16384                                                        |
| --message-send-max-retries   | Integer         | 最大的重试发送次数                                           | 3                                                            |
| --metadata-expiry-ms         | Long            | 强制更新元数据的时间阈值(ms)                                 | 300000                                                       |
| --producer-property          | String          | 将自定义属性传递给生成器的机制                               | 如：key=value                                                |
| --producer.config            | String          | 生产者配置属性文件--producer-property优先于此配置	配置文件完整路径 |                                                              |
| --property                   | String          | 自定义消息读取器                                             | parse.key=true/false key.separator=<key.separator>ignore.error=true/false |
| --request-required-acks      | String          | 生产者请求的确认方式                                         | 0、1(默认值)、all                                            |
| --request-timeout-ms         | Integer         | 生产者请求的确认超时时间                                     | 1500(默认值)                                                 |
| --retry-backoff-ms           | Integer         | 生产者重试前，刷新元数据的等待时间阈值                       | 100(默认值)                                                  |
| --socket-buffer-size         | Integer         | TCP接收缓冲大小                                              | 102400(默认值)                                               |
| --timeout                    | Integer         | 消息排队异步等待处理的时间阈值                               | 1000(默认值)                                                 |
| --sync                       | 同步发送消息    |                                                              |                                                              |
| --version                    | 显示 Kafka 版本 | 不配合其他参数时，显示为本地Kafka版本                        |                                                              |
| --help                       | 打印帮助信息    |                                                              |                                                              |

### 5. Topic的消费kafka-console-consumer.sh

 **1. 新客户端从头消费`--from-beginning` (注意这里是新客户端,如果之前已经消费过了是不会从头消费的)**

 下面没有指定客户端名称,所以每次执行都是新客户端都会从头消费

> sh bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test  --from-beginning 

**2. 正则表达式匹配topic进行消费`--whitelist`**

**`消费所有的topic`**

> sh bin/kafka-console-consumer.sh --bootstrap-server localhost:9092  --whitelist '.*' 

**`消费所有的topic，并且还从头消费`**

> sh bin/kafka-console-consumer.sh --bootstrap-server localhost:9092  --whitelist '.*'   --from-beginning 

**3.显示key进行消费`--property print.key=true`**

> sh bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test  --property print.key=true

**4. 指定分区消费`--partition` 指定起始偏移量消费`--offset`**

> sh bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --partition 0 --offset	100

**5. 给客户端命名`--group`**

注意给客户端命名之后,如果之前有过消费，那么`--from-beginning`就不会再从头消费了

> sh bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test  --group test-group

**6. 添加客户端属性`--consumer-property`**

这个参数也可以给客户端添加属性,但是注意 不能多个地方配置同一个属性,他们是互斥的;比如在下面的基础上还加上属性`--group test-group` 那肯定不行

> sh bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test  `--consumer-property group.id=test-consumer-group`

**7. 添加客户端属性`--consumer.config`** 

跟`--consumer-property` 一样的性质,都是添加客户端的属性,不过这里是指定一个文件,把属性写在文件里面, `--consumer-property` 的优先级大于 `--consumer.config`

> sh bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test   --consumer.config config/consumer.properties

------

| 参数                      | 描述                                                         | 例子                                                         |      |
| :------------------------ | :----------------------------------------------------------- | :----------------------------------------------------------- | ---- |
| `--group`                 | 指定消费者所属组的ID                                         |                                                              |      |
| `--topic`                 | 被消费的topic                                                |                                                              |      |
| `--partition`             | 指定分区 ；除非指定`–offset`，否则从分区结束(latest)开始消费 | `--partition	0`                                           |      |
| `--offset`                | 执行消费的起始offset位置 ;默认值: latest;   /latest /earliest /偏移量 | `--offset` 10                                                |      |
| `--whitelist`             | 正则表达式匹配topic；`--topic`就不用指定了; 匹配到的所有topic都会消费; 当然用了这个参数,`--partition` `--offset`等就不能使用了 |                                                              |      |
| `--consumer-property`     | 将用户定义的属性以key=value的形式传递给使用者                | `--consumer-property`group.id=test-consumer-group            |      |
| `--consumer.config`       | 消费者配置属性文件请注意，`consumer-property`优先于此配置    | `--consumer.config` config/consumer.properties               |      |
| `--property`              | 初始化消息格式化程序的属性                                   | print.timestamp=true,false 、print.key=true,false 、print.value=true,false 、key.separator=<key.separator> 、line.separator=<line.separator>、key.deserializer=<key.deserializer>、value.deserializer=<value.deserializer> |      |
| `--from-beginning`        | 从存在的最早消息开始，而不是从最新消息开始,注意如果配置了客户端名称并且之前消费过，那就不会从头消费了 |                                                              |      |
| `--max-messages`          | 消费的最大数据量，若不指定，则持续消费下去                   | `--max-messages` 100                                         |      |
| `--skip-message-on-error` | 如果处理消息时出错，请跳过它而不是暂停                       |                                                              |      |
| `--isolation-level`       | 设置为read_committed以过滤掉未提交的事务性消息,设置为read_uncommitted以读取所有消息,默认值:read_uncommitted |                                                              |      |
| `--formatter`             | kafka.tools.DefaultMessageFormatter、kafka.tools.LoggingMessageFormatter、kafka.tools.NoOpMessageFormatter、kafka.tools.ChecksumMessageFormatter |                                                              |      |

### 6.kafka-leader-election Leader重新选举

**6.1 指定Topic指定分区用重新`PREFERRED：优先副本策略` 进行Leader重选举**

```js
> sh bin/kafka-leader-election.sh --bootstrap-server xxxx:9090 --topic test_create_topic4 --election-type PREFERRED --partition 0
```

**6.2 所有Topic所有分区用重新`PREFERRED：优先副本策略` 进行Leader重选举**

```js
sh bin/kafka-leader-election.sh --bootstrap-server xxxx:9090 --election-type preferred  --all-topic-partitions
```

**6.3 设置配置文件批量指定topic和分区进行Leader重选举**

先配置leader-election.json文件

```js
{
  "partitions": [
    {
      "topic": "test_create_topic4",
      "partition": 1
    },
    {
      "topic": "test_create_topic4",
      "partition": 2
    }
  ]
}
 sh bin/kafka-leader-election.sh --bootstrap-server xxx:9090 --election-type preferred  --path-to-json-file config/leader-election.json
 
```

------

相关可选参数

| 参数                               | 描述                                                         | 例子                              |
| :--------------------------------- | :----------------------------------------------------------- | :-------------------------------- |
| `--bootstrap-server` 指定kafka服务 | 指定连接到的kafka服务                                        | --bootstrap-server localhost:9092 |
| `--topic`                          | 指定Topic，此参数跟`--all-topic-partitions`和`path-to-json-file` 三者互斥 |                                   |
| `--partition`                      | 指定分区,跟`--topic`搭配使用                                 |                                   |
| `--election-type`                  | 两个选举策略(`PREFERRED:`优先副本选举,如果第一个副本不在线的话会失败;`UNCLEAN`: 策略) |                                   |
| `--all-topic-partitions`           | 所有topic所有分区执行Leader重选举; 此参数跟`--topic`和`path-to-json-file` 三者互斥 |                                   |
| `--path-to-json-file`              | 配置文件批量选举，此参数跟`--topic`和`all-topic-partitions` 三者互斥 |                                   |

### 7. 持续批量推送消息kafka-verifiable-producer.sh

**单次发送100条消息`--max-messages 100`**

一共要推送多少条，默认为-1，-1表示一直推送到进程关闭位置

> sh bin/kafka-verifiable-producer.sh --topic test_create_topic4 --bootstrap-server  localhost:9092 `--max-messages 100`

**每秒发送最大吞吐量不超过消息 `--throughput 100`**

推送消息时的吞吐量，单位messages/sec。默认为-1，表示没有限制

> sh bin/kafka-verifiable-producer.sh --topic test_create_topic4 --bootstrap-server   localhost:9092  `--throughput 100`

**发送的消息体带前缀`--value-prefix`**

> sh bin/kafka-verifiable-producer.sh --topic test_create_topic4 --bootstrap-server   localhost:9092  `--value-prefix 666`

注意`--value-prefix 666`必须是整数,发送的消息体的格式是加上一个 点号`.`  例如： `666.`

其他参数：

 `--producer.config CONFIG_FILE` 指定producer的配置文件

`--acks ACKS`            每次推送消息的ack值，默认是-1

### 8. 持续批量拉取消息kafka-verifiable-consumer

**持续消费**

> sh bin/kafka-verifiable-consumer.sh --group-id test_consumer  --bootstrap-server  localhost:9092   --topic test_create_topic4 

**单次最大消费10条消息`--max-messages 10`**

> sh bin/kafka-verifiable-consumer.sh --group-id test_consumer  --bootstrap-server  localhost:9092  --topic test_create_topic4 `--max-messages 10`

------

相关可选参数

| 参数                               | 描述                                                         | 例子                              |
| :--------------------------------- | :----------------------------------------------------------- | :-------------------------------- |
| `--bootstrap-server` 指定kafka服务 | 指定连接到的kafka服务;                                       | --bootstrap-server localhost:9092 |
| `--topic`                          | 指定消费的topic                                              |                                   |
| `--group-id`                       | 消费者id；不指定的话每次都是新的组id                         |                                   |
| `group-instance-id`                | 消费组实例ID,唯一值                                          |                                   |
| `--max-messages`                   | 单次最大消费的消息数量                                       |                                   |
| `--enable-autocommit`              | 是否开启offset自动提交；默认为false                          |                                   |
| `--reset-policy`                   | 当以前没有消费记录时，选择要拉取offset的策略，可以是`earliest`, `latest`,`none`。默认是earliest |                                   |
| `--assignment-strategy`            | consumer分配分区策略，默认是`org.apache.kafka.clients.consumer.RangeAssignor` |                                   |
| `--consumer.config`                | 指定consumer的配置文件                                       |                                   |

### 9.生产者压力测试kafka-producer-perf-test.sh

**1. 发送1024条消息`--num-records 100`并且每条消息大小为1KB`--record-size 1024` 最大吞吐量每秒10000条`--throughput 100`**

> sh bin/kafka-producer-perf-test.sh --topic test_create_topic4 --num-records 100 --throughput 100000  --producer-props bootstrap.servers=localhost:9092 --record-size 1024

你可以通过[LogIKM](https://github.com/didi/Logi-KafkaManager)查看分区是否增加了对应的数据大小

![img](https://ask.qcloudimg.com/http-save/8815606/4e7154bcd66f812d7de4d72dd488af59.png?imageView2/2/w/1620)在这里插入图片描述

从[LogIKM](https://github.com/didi/Logi-KafkaManager) 可以看到发送了1024条消息; 并且总数据量=1M; 1024条*1024byte = 1M;

**2. 用指定消息文件`--payload-file`发送100条消息最大吞吐量每秒100条`--throughput 100`**

1. 先配置好消息文件

   ```
   batchmessage.txt
   ```

   ![img](https://ask.qcloudimg.com/http-save/8815606/8e4ef2926f49e3c6fea3da9a6ee83a6d.png?imageView2/2/w/1620)在这里插入图片描述

2. 然后执行命令 发送的消息会从`batchmessage.txt`里面随机选择; 注意这里我们没有用参数`--payload-delimeter`指定分隔符，默认分隔符是\n换行;

```js
> bin/kafka-producer-perf-test.sh --topic test_create_topic4 --num-records 100 --throughput 100  --producer-props bootstrap.servers=localhost:9090 --payload-file config/batchmessage.txt
>
```

1. 验证消息，可以通过 [LogIKM](https://github.com/didi/Logi-KafkaManager)   查看发送的消息

```js
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210624175212664.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2MzQwNjY=,size_16,color_FFFFFF,t_70)
```

------

相关可选参数

| 参数                        | 描述                                                         | 例子                                                         |
| :-------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `--topic`                   | 指定消费的topic                                              |                                                              |
| `--num-records`             | 发送多少条消息                                               |                                                              |
| `--throughput`              | 每秒消息最大吞吐量                                           |                                                              |
| `--producer-props`          | 生产者配置, k1=v1,k2=v2                                      | `--producer-props` bootstrap.servers= localhost:9092,client.id=test_client |
| `--producer.config`         | 生产者配置文件                                               | `--producer.config` config/producer.propeties                |
| `--print-metrics`           | 在test结束的时候打印监控信息,默认false                       | `--print-metrics` true                                       |
| `--transactional-id`        | 指定事务 ID，测试并发事务的性能时需要，只有在 --transaction-duration-ms > 0 时生效，默认值为 performance-producer-default-transactional-id |                                                              |
| `--transaction-duration-ms` | 指定事务持续的最长时间，超过这段时间后就会调用 commitTransaction 来提交事务，只有指定了 > 0 的值才会开启事务，默认值为 0 |                                                              |
| `--record-size`             | 一条消息的大小byte; 和 --payload-file 两个中必须指定一个，但不能同时指定 |                                                              |
| `--payload-file`            | 指定消息的来源文件，只支持 UTF-8 编码的文本文件，文件的消息分隔符通过 `--payload-delimeter`指定,默认是用换行\nl来分割的，和 --record-size 两个中必须指定一个，但不能同时指定 ; 如果提供的消息 |                                                              |
| `--payload-delimeter`       | 如果通过 `--payload-file` 指定了从文件中获取消息内容，那么这个参数的意义是指定文件的消息分隔符，默认值为 \n，即文件的每一行视为一条消息；如果未指定`--payload-file`则此参数不生效；发送消息的时候是随机送文件里面选择消息发送的; |                                                              |

### 10.消费者压力测试kafka-consumer-perf-test.sh

**消费100条消息`--messages 100`**

> sh bin/kafka-consumer-perf-test.sh -topic test_create_topic4 --bootstrap-server  localhost:9090 --messages 100

------

相关可选参数

| 参数                    | 描述                          | 例子                          |      |
| :---------------------- | :---------------------------- | :---------------------------- | ---- |
| `--bootstrap-server`    |                               |                               |      |
| `--consumer.config`     | 消费者配置文件                |                               |      |
| `--date-format`         | 结果打印出来的时间格式化      | 默认：yyyy-MM-dd HH:mm:ss:SSS |      |
| `--fetch-size`          | 单次请求获取数据的大小        | 默认1048576                   |      |
| `--topic`               | 指定消费的topic               |                               |      |
| `--from-latest`         |                               |                               |      |
| `--group`               | 消费组ID                      |                               |      |
| `--hide-header`         | 如果设置了,则不打印header信息 |                               |      |
| `--messages`            | 需要消费的数量                |                               |      |
| `--num-fetch-threads`   | feth 数据的线程数             | 默认：1                       |      |
| `--print-metrics`       | 结束的时候打印监控数据        |                               |      |
| `--show-detailed-stats` |                               |                               |      |
| `--threads`             | 消费线程数;                   | 默认 10                       |      |

### 11.删除指定分区的消息kafka-delete-records.sh

**删除指定topic的某个分区的消息删除至offset为1024**

先配置json文件`offset-json-file.json`

```js
{"partitions":
[{"topic": "test1", "partition": 0,
  "offset": 1024}],
  "version":1
}
```

在执行命令

> sh bin/kafka-delete-records.sh --bootstrap-server  172.23.250.249:9090 --offset-json-file config/offset-json-file.json

验证 通过 [LogIKM](https://github.com/didi/Logi-KafkaManager)   查看发送的消息

![img](https://ask.qcloudimg.com/http-save/8815606/28b346656fe2cbb634a47682879855ac.png?imageView2/2/w/1620)在这里插入图片描述

**从这里可以看出来,配置`"offset": 1024` 的意思是从最开始的地方删除消息到 1024的offset; 是从最前面开始删除的**

### 12. 查看Broker磁盘信息

**查询指定topic磁盘信息`--topic-list    topic1,topic2`**

> sh bin/kafka-log-dirs.sh  --bootstrap-server xxxx:9090 --describe  --topic-list test2

**查询指定Broker磁盘信息`--broker-list 0   broker1,broker2`**

> sh bin/kafka-log-dirs.sh  --bootstrap-server xxxxx:9090 --describe  --topic-list test2 --broker-list 0

例如我一个3分区3副本的Topic的查出来的信息

```
logDir` Broker中配置的`log.dir
{
	"version": 1,
	"brokers": [{
		"broker": 0,
		"logDirs": [{
			"logDir": "/Users/xxxx/work/IdeaPj/ss/kafka/kafka-logs-0",
			"error": null,
			"partitions": [{
				"partition": "test2-1",
				"size": 0,
				"offsetLag": 0,
				"isFuture": false
			}, {
				"partition": "test2-0",
				"size": 0,
				"offsetLag": 0,
				"isFuture": false
			}, {
				"partition": "test2-2",
				"size": 0,
				"offsetLag": 0,
				"isFuture": false
			}]
		}]
	}, {
		"broker": 1,
		"logDirs": [{
			"logDir": "/Users/xxxx/work/IdeaPj/ss/kafka/kafka-logs-1",
			"error": null,
			"partitions": [{
				"partition": "test2-1",
				"size": 0,
				"offsetLag": 0,
				"isFuture": false
			}, {
				"partition": "test2-0",
				"size": 0,
				"offsetLag": 0,
				"isFuture": false
			}, {
				"partition": "test2-2",
				"size": 0,
				"offsetLag": 0,
				"isFuture": false
			}]
		}]
	}, {
		"broker": 2,
		"logDirs": [{
			"logDir": "/Users/xxxx/work/IdeaPj/ss/kafka/kafka-logs-2",
			"error": null,
			"partitions": [{
				"partition": "test2-1",
				"size": 0,
				"offsetLag": 0,
				"isFuture": false
			}, {
				"partition": "test2-0",
				"size": 0,
				"offsetLag": 0,
				"isFuture": false
			}, {
				"partition": "test2-2",
				"size": 0,
				"offsetLag": 0,
				"isFuture": false
			}]
		}]
	}, {
		"broker": 3,
		"logDirs": [{
			"logDir": "/Users/xxxx/work/IdeaPj/ss/kafka/kafka-logs-3",
			"error": null,
			"partitions": []
		}]
	}]
}
```

如果你觉得通过命令查询磁盘信息比较麻烦，你也可以通过 [LogIKM](https://github.com/didi/Logi-KafkaManager)    查看

![img](https://ask.qcloudimg.com/http-save/8815606/fdab5f7c1e51447b4eb463a0aa9f8d34.png?imageView2/2/w/1620)在这里插入图片描述

### 13. 消费者组管理

#### 1. 查看消费者列表`--list`

> ```bash
> sh bin/kafka-consumer-groups.sh --bootstrap-server xxxx:9090 --list
> ```
>

先调用`MetadataRequest`拿到所有在线Broker列表

再给每个Broker发送`ListGroupsRequest`请求获取 消费者组数据

#### 2. 查看消费者组详情`--describe`

```
DescribeGroupsRequest
```

**查看消费组详情`--group` 或 `--all-groups`**

> **查看指定消费组详情`--group`**
>
> ```
> sh bin/kafka-consumer-groups.sh --bootstrap-server xxxxx:9090  --describe --group test2_consumer_group
> ```
>
> 
>
> undefined
>
> **查看所有消费组详情`--all-groups`**
>
> ```
> sh bin/kafka-consumer-groups.sh --bootstrap-server xxxxx:9090  --describe --all-groups
> ```
>
> 查看该消费组 消费的所有Topic、及所在分区、最新消费offset、Log最新数据offset、Lag还未消费数量、消费者ID等等信息

**查询消费者成员信息`--members`**

> **所有消费组成员信息**
>
> ```bash
> sh bin/kafka-consumer-groups.sh  --describe  --all-groups --members --bootstrap-server xxx:9090
> ```
>
> **指定消费组成员信息**
>
> ```bash
> sh bin/kafka-consumer-groups.sh --describe   --members  --group test2_consumer_group --bootstrap-server xxxx:9090
> ```

**查询消费者状态信息`--state`**

> **所有消费组状态信息**
>
> ```bash
> sh bin/kafka-consumer-groups.sh  --describe  --all-groups --state --bootstrap-server xxxx:9090
> ```
>
> **指定消费组状态信息**
>
> ```bash
> sh bin/kafka-consumer-groups.sh  --describe   --state --group test2_consumer_group --bootstrap-server xxxxx:9090
> ```
>

#### 3. 删除消费者组`--delete`

```
DeleteGroupsRequest
```

**删除消费组--delete**

> **删除指定消费组`--group`**
>
> ```
> sh bin/kafka-consumer-groups.sh  --delete  --group test2_consumer_group   --bootstrap-server xxxx:9090
> ```
>
> **删除所有消费组`--all-groups`**
>
> ```
> sh bin/kafka-consumer-groups.sh  --delete  --all-groups   --bootstrap-server xxxx:9090
> ```

**PS: 想要删除消费组前提是这个消费组的所有客户端都停止消费/不在线才能够成功删除;否则会报下面异常**

```js
Error: Deletion of some consumer groups failed:
* Group 'test2_consumer_group' could not be deleted due to: java.util.concurrent.ExecutionException: org.apache.kafka.common.errors.GroupNotEmptyException: The group is not empty.
```

#### 4. 重置消费组的偏移量 `--reset-offsets`

<font color=red>能够执行成功的一个前提是 消费组这会是不可用状态;</font>

下面的示例使用的参数是: `--dry-run` ;这个参数表示预执行,会打印出来将要处理的结果; 

等你想真正执行的时候请换成参数`--excute` ;

下面示例 重置模式都是 `--to-earliest` 重置到最早的;

请根据需要参考下面 **相关重置Offset的模式** 换成其他模式; 

**重置指定消费组的偏移量 `--group`**

> **重置指定消费组的所有Topic的偏移量`--all-topic`**
>
> ```
> sh bin/kafka-consumer-groups.sh  --reset-offsets  --to-earliest --group test2_consumer_group   --bootstrap-server xxxx:9090 --dry-run --all-topic
> ```
>
> **重置指定消费组的指定Topic的偏移量`--topic`**
>
> ```
> sh bin/kafka-consumer-groups.sh  --reset-offsets  --to-earliest --group test2_consumer_group   --bootstrap-server xxxx:9090 --dry-run --topic test2
> ```

**重置所有消费组的偏移量 `--all-group`**

> **重置所有消费组的所有Topic的偏移量`--all-topic`**
>
> ```
> sh bin/kafka-consumer-groups.sh  --reset-offsets  --to-earliest --all-group   --bootstrap-server xxxx:9090 --dry-run --all-topic
> ```
>
> **重置所有消费组中指定Topic的偏移量`--topic`**
>
> ```
> sh bin/kafka-consumer-groups.sh  --reset-offsets  --to-earliest --all-group   --bootstrap-server xxxx:9090 --dry-run --topic test2
> ```

`--reset-offsets` 后面需要接**重置的模式**

**相关重置Offset的模式**

| 参数                  | 描述                                                         | 例子                                                         |
| :-------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **`--to-earliest` :** | 重置offset到最开始的那条offset(找到还未被删除最早的那个offset) |                                                              |
| **`--to-current`:**   | 直接重置offset到当前的offset，也就是LOE                      |                                                              |
| **`--to-latest`：**   | 重置到最后一个offset                                         |                                                              |
| **`--to-datetime`:**  | 重置到指定时间的offset;格式为:`YYYY-MM-DDTHH:mm:SS.sss`;     | `--to-datetime "2021-6-26T00:00:00.000"`                     |
| `--to-offset`         | 重置到指定的offset,但是通常情况下,匹配到多个分区,这里是将匹配到的所有分区都重置到这一个值; 如果 1.目标最大offset<`--to-offset`, 这个时候重置为目标最大offset；2.目标最小offset>`--to-offset` ，则重置为最小; 3.否则的话才会重置为`--to-offset`的目标值; **一般不用这个** | `--to-offset 3465` ![img](https://img-blog.csdnimg.cn/20210626153528929.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2MzQwNjY=,size_16,color_FFFFFF,t_70)在这里插入图片描述 |
| `--shift-by`          | 按照偏移量增加或者减少多少个offset；正的为往前增加;负的往后退；当然这里也是匹配所有的; | `--shift-by 100` 、`--shift-by -100`                         |
| `--from-file`         | 根据CVS文档来重置; 这里下面单独讲解                          |                                                              |

**`--from-file`着重讲解一下** 

> 上面其他的一些模式重置的都是匹配到的所有分区; 不能够每个分区重置到不同的offset；不过**`--from-file`**可以让我们更灵活一点;

1. 先配置cvs文档 格式为: Topic:分区号: 重置目标偏移量```cvs test2,0,100 test2,1,200 test2,2,300 ```
2. 执行命令`sh bin/kafka-consumer-groups.sh  --reset-offsets   --group test2_consumer_group   --bootstrap-server xxxx:9090 --dry-run  --from-file config/reset-offset.csv`

#### 5. 删除偏移量`delete-offsets`

<font color=red>能够执行成功的一个前提是 消费组这会是不可用状态;</font>

偏移量被删除了之后,Consumer Group下次启动的时候,会从头消费;

> ```
> sh bin/kafka-consumer-groups.sh  --delete-offsets  --group test2_consumer_group2  --bootstrap-server XXXX:9090 --topic test2
> ```

------

相关可选参数

| 参数                 | 描述                                                         | 例子                              |
| :------------------- | :----------------------------------------------------------- | :-------------------------------- |
| `--bootstrap-server` | 指定连接到的kafka服务;                                       | --bootstrap-server localhost:9092 |
| `--list`             | 列出所有消费组名称                                           | `--list`                          |
| `--describe`         | 查询消费者描述信息                                           | `--describe`                      |
| `--group`            | 指定消费组                                                   |                                   |
| `--all-groups`       | 指定所有消费组                                               |                                   |
| `--members`          | 查询消费组的成员信息                                         |                                   |
| `--state`            | 查询消费者的状态信息                                         |                                   |
| `--offsets`          | 在查询消费组描述信息的时候,这个参数会列出消息的偏移量信息; 默认就会有这个参数的; |                                   |
| `dry-run`            | 重置偏移量的时候,使用这个参数可以让你预先看到重置情况，这个时候还没有真正的执行,真正执行换成`--excute`;默认为`dry-run` |                                   |
| `--excute`           | 真正的执行重置偏移量的操作;                                  |                                   |
| `--to-earliest`      | 将offset重置到最早                                           |                                   |
| `to-latest`          | 将offset重置到最近                                           |                                   |

### 14.可选配置 config

**ConfigCommand 的一些可选配置**

------

Topic相关可选配置

| key                                     | value                                                        | 示例    |
| :-------------------------------------- | :----------------------------------------------------------- | :------ |
| cleanup.policy                          | 清理策略                                                     |         |
| compression.type                        | 压缩类型(通常建议在produce端控制)                            |         |
| delete.retention.ms                     | 压缩日志的保留时间                                           |         |
| file.delete.delay.ms                    |                                                              |         |
| flush.messages                          | 持久化message限制                                            |         |
| flush.ms                                | 持久化频率                                                   |         |
| follower.replication.throttled.replicas | flowwer副本限流 格式：分区号:副本follower号,分区号:副本follower号 | 0:1,1:1 |
| index.interval.bytes                    |                                                              |         |
| leader.replication.throttled.replicas   | leader副本限流 格式：分区号:副本Leader号                     | 0:0     |
| max.compaction.lag.ms                   |                                                              |         |
| max.message.bytes                       | 最大的batch的message大小                                     |         |
| message.downconversion.enable           | message是否向下兼容                                          |         |
| message.format.version                  | message格式版本                                              |         |
| message.timestamp.difference.max.ms     |                                                              |         |
| message.timestamp.type                  |                                                              |         |
| min.cleanable.dirty.ratio               |                                                              |         |
| min.compaction.lag.ms                   |                                                              |         |
| min.insync.replicas                     | 最小的ISR                                                    |         |
| preallocate                             |                                                              |         |
| retention.bytes                         | 日志保留大小(通常按照时间限制)                               |         |
| retention.ms                            | 日志保留时间                                                 |         |
| segment.bytes                           | segment的大小限制                                            |         |
| segment.index.bytes                     |                                                              |         |
| segment.jitter.ms                       |                                                              |         |
| segment.ms                              | segment的切割时间                                            |         |
| unclean.leader.election.enable          | 是否允许非同步副本选主                                       |         |

Broker相关可选配置

| key                                            | value | 示例 |
| :--------------------------------------------- | :---- | :--- |
| advertised.listeners                           |       |      |
| background.threads                             |       |      |
| compression.type                               |       |      |
| follower.replication.throttled.rate            |       |      |
| leader.replication.throttled.rate              |       |      |
| listener.security.protocol.map                 |       |      |
| listeners                                      |       |      |
| log.cleaner.backoff.ms                         |       |      |
| log.cleaner.dedupe.buffer.size                 |       |      |
| log.cleaner.delete.retention.ms                |       |      |
| log.cleaner.io.buffer.load.factor              |       |      |
| log.cleaner.io.buffer.size                     |       |      |
| log.cleaner.io.max.bytes.per.second            |       |      |
| log.cleaner.max.compaction.lag.ms              |       |      |
| log.cleaner.min.cleanable.ratio                |       |      |
| log.cleaner.min.compaction.lag.ms              |       |      |
| log.cleaner.threads                            |       |      |
| log.cleanup.policy                             |       |      |
| log.flush.interval.messages                    |       |      |
| log.flush.interval.ms                          |       |      |
| log.index.interval.bytes                       |       |      |
| log.index.size.max.bytes                       |       |      |
| log.message.downconversion.enable              |       |      |
| log.message.timestamp.difference.max.ms        |       |      |
| log.message.timestamp.type                     |       |      |
| log.preallocate                                |       |      |
| log.retention.bytes                            |       |      |
| log.retention.ms                               |       |      |
| log.roll.jitter.ms                             |       |      |
| log.roll.ms                                    |       |      |
| log.segment.bytes                              |       |      |
| log.segment.delete.delay.ms                    |       |      |
| max.connections                                |       |      |
| max.connections.per.ip                         |       |      |
| max.connections.per.ip.overrides               |       |      |
| message.max.bytes                              |       |      |
| metric.reporters                               |       |      |
| min.insync.replicas                            |       |      |
| num.io.threads                                 |       |      |
| num.network.threads                            |       |      |
| num.recovery.threads.per.data.dir              |       |      |
| num.replica.fetchers                           |       |      |
| principal.builder.class                        |       |      |
| replica.alter.log.dirs.io.max.bytes.per.second |       |      |
| sasl.enabled.mechanisms                        |       |      |
| sasl.jaas.config                               |       |      |
| sasl.kerberos.kinit.cmd                        |       |      |
| sasl.kerberos.min.time.before.relogin          |       |      |
| sasl.kerberos.principal.to.local.rules         |       |      |
| sasl.kerberos.service.name                     |       |      |
| sasl.kerberos.ticket.renew.jitter              |       |      |
| sasl.kerberos.ticket.renew.window.factor       |       |      |
| sasl.login.refresh.buffer.seconds              |       |      |
| sasl.login.refresh.min.period.seconds          |       |      |
| sasl.login.refresh.window.factor               |       |      |
| sasl.login.refresh.window.jitter               |       |      |
| sasl.mechanism.inter.broker.protocol           |       |      |
| ssl.cipher.suites                              |       |      |
| ssl.client.auth                                |       |      |
| ssl.enabled.protocols                          |       |      |
| ssl.endpoint.identification.algorithm          |       |      |
| ssl.key.password                               |       |      |
| ssl.keymanager.algorithm                       |       |      |
| ssl.keystore.location                          |       |      |
| ssl.keystore.password                          |       |      |
| ssl.keystore.type                              |       |      |
| ssl.protocol                                   |       |      |
| ssl.provider                                   |       |      |
| ssl.secure.random.implementation               |       |      |
| ssl.trustmanager.algorithm                     |       |      |
| ssl.truststore.location                        |       |      |
| ssl.truststore.password                        |       |      |
| ssl.truststore.type                            |       |      |
| unclean.leader.election.enable                 |       |      |

Users相关可选配置

| key                | value                  | 示例 |
| :----------------- | :--------------------- | :--- |
| SCRAM-SHA-256      |                        |      |
| SCRAM-SHA-512      |                        |      |
| consumer_byte_rate | 针对消费者user进行限流 |      |
| producer_byte_rate | 针对生产者进行限流     |      |
| request_percentage | 请求百分比             |      |

clients相关可选配置

| key                | value | 示例 |
| :----------------- | :---- | :--- |
| consumer_byte_rate |       |      |
| producer_byte_rate |       |      |
| request_percentage |       |      |

**以上大部分运维操作,都可以使用**  [**LogI-Kafka-Manager**](https://github.com/didi/Logi-KafkaManager) **在平台上可视化操作;**

kafka-consumer-groups.sh --bootstrap-server 30.12.96.14:9092 --group *$topic_group* --describe 





+++

### 15.常用命令

* kafka-console-consumer.sh --bootstrap-server 30.12.96.14:9092 --topic *$topic* --property print.timestamp=true

kafka-console-consumer.sh 脚本是一个简易的消费者控制台。该 shell 脚本的功能通过调用 kafka.tools 包下的 `ConsoleConsumer` 类，并将提供的命令行参数全部传给该类实现。

#### 1.手动插入数据

```
kafka-console-producer.sh --broker-list 192.168.104.91:9092,192.168.104.92:9092,192.168.104.93:9092 --topic capture_test
```

#### 2. 消息消费

```
bin``/kafka-console-consumer``.sh --bootstrap-server 192.168.104.91:9092,192.168.104.92:9092,192.168.104.93:9092 --topic topicName``bin``/kafka-console-consumer``.sh --zookeeper 192.168.104.91:2181,192.168.104.92:2181,192.168.104.93:2181 --topic topicName``表示从 latest 位移位置开始消费该主题的所有分区消息，即仅消费正在写入的消息。
```

#### 3.从开始位置消费

```
bin``/kafka-console-consumer``.sh --bootstrap-server 192.168.104.91:9092,192.168.104.92:9092,192.168.104.93:9092 --from-beginning --topic topicName``bin``/kafka-console-consumer``.sh --zookeeper 192.168.104.91:2181,192.168.104.92:2181,192.168.104.93:2181 --from-beginning --topic topicName
```

#### 4.显示key消费

```bash
bin /kafka-console-consume.sh --bootstrap-server 192.168.104.91:9092,192.168.104.92:9092,192.168.104.93:9092 --property print.key=true --topic topicName
```

消费出的消息结果将打印出消息体的 key 和 value。

####  5.**常用脚本**

```bash
## 查找大数据量中的一条数据 需指定 partition 和 开始 offset
kafka-run-class kafka.tools.ConsoleConsumer --bootstrap-server 30.105.15.1:9092 --topic sdf_invalid_record_topic --partirion 0 --offset 267576117 --max-message 10000 |grep ''
```

```shell
## 获取一个时间区间内 kafka 消费的数据条数 1634140800000=2021-10-14 00:00:00 ~ 1634227199000=2021-10-14 23:59:59
ret1=$(/opt/kafka/bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list 30.105.15.1:9092 --topic test_topic --time 1634140800000 |awk 'BEGIN{FS=":"}{sum += $3};END {print sum}')
echo "ret1: " $ret1
ret2=$(/opt/kafka/bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list 30.105.15.1:9092 --topic test_topic --time 1634227199000 |awk 'BEGIN{FS=":"}{sum += $3};END {print sum}')
echo "ret2: " $ret2
echo "total: " $(($ret2-$ret1))
```

```bash
## 查询所有的消费组
bin/kafka-consumer-groups.sh --bootstrap-server 10.129.131.154:9092 --list
## kafka 0.9+ 的新版consumer API
bin/kafka-run-class.sh kafka.admin.ConsumerGroupCommand --bootstrap-server 10.129.131.154:9092 --list
```

```bash
## 显示某个消费组的消费详情（0.10.1.0版本+）
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group group1
```

```bash
/sensorsmounts/hybriddata/binddirs/main/program/sp/sdp/1.0.0.0-123/kafka/bin/kafka-consumer-groups.sh  --bootstrap-server 10.129.131.154:9092 --group gf_consumer_group_test011 --describe |grep -Ev 'grep|LAG' |awk '{sum+=$5};END{print "lag total:"sum}'
```



| 参数                      | 值类型  | 说明                                                         | 有效值                                                       |
| ------------------------- | ------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| --topic                   | string  | 被消费的topic                                                |                                                              |
| --whitelist               | string  | 正则表达式，指定要包含以供使用的主题的白名单                 |                                                              |
| --partition               | integer | 指定分区 除非指定’–offset’，否则从分区结束(latest)开始消费   |                                                              |
| --offset                  | string  | 执行消费的起始offset位置 默认值:latest                       | latest earliest <offset>                                     |
| --consumer-property       | string  | 将用户定义的属性以key=value的形式传递给使用者                |                                                              |
| --consumer.config         | string  | 消费者配置属性文件 请注意，[consumer-property]优先于此配置   |                                                              |
| --formatter               | string  | 用于格式化kafka消息以供显示的类的名称 默认值:kafka.tools.DefaultMessageFormatter | kafka.tools.DefaultMessageFormatter kafka.tools.LoggingMessageFormatter kafka.tools.NoOpMessageFormatter kafka.tools.ChecksumMessageFormatter |
| --property                | string  | 初始化消息格式化程序的属性                                   | print.timestamp=true\|false print.key=true\|false print.value=true\|false key.separator=<key.separator> line.separator=<line.separator> key.deserializer=<key.deserializer> value.deserializer=<value.deserializer> |
| --from-beginning          |         | 从存在的最早消息开始，而不是从最新消息开始                   |                                                              |
| --max-messages            | integer | 消费的最大数据量，若不指定，则持续消费下去                   |                                                              |
| --timeout-ms              | integer | 在指定时间间隔内没有消息可用时退出                           |                                                              |
| --skip-message-on-error   |         | 如果处理消息时出错，请跳过它而不是暂停                       |                                                              |
| --bootstrap-server        | string  | 必需(除非使用旧版本的消费者)，要连接的服务器                 |                                                              |
| --key-deserializer        | string  |                                                              |                                                              |
| --value-deserializer      | string  |                                                              |                                                              |
| --enable-systest-events   |         | 除记录消费的消息外，还记录消费者的生命周期 (用于系统测试)    |                                                              |
| --isolation-level         | string  | 设置为read_committed以过滤掉未提交的事务性消息 设置为read_uncommitted以读取所有消息 默认值:read_uncommitted |                                                              |
| --group                   | string  | 指定消费者所属组的ID                                         |                                                              |
| --blacklist               | string  | 要从消费中排除的主题黑名单                                   |                                                              |
| --csv-reporter-enabled    |         | 如果设置，将启用csv metrics报告器                            |                                                              |
| --delete-consumer-offsets |         | 如果指定，则启动时删除zookeeper中的消费者信息                |                                                              |
| --metrics-dir             | string  | 输出csv度量值 需与[csv-reporter-enable]配合使用              |                                                              |
| --zookeeper               | string  | 必需(仅当使用旧的使用者时)连接zookeeper的字符串。 可以给出多个URL以允许故障转移 |                                                              |

　　

___





#### 6.kafka生产者

###### 如何提升吞吐量

> 参数一：`buffer.memory`：设置发送消息的缓冲区，默认值是33554432，就是32MB 
>
> 参数二：`compression.type`：默认是none，不压缩，但是也可以使用lz4压缩，效率还是不错的，压缩之后可以减小数据量，提升吞吐量，但是会加大producer端的cpu开销 
>
> 参数三：`batch.size`：设置batch的大小，
>
> * 如果batch太小，会导致频繁网络请求，吞吐量下降；
> * 如果batch太大，会导致一条消息需要等待很久才能被发送出去，而且会让内存缓冲区有很大压力，过多数据缓冲在内存里，默认值是：16384，就是16kb，也就是一个batch满了16kb就发送出去，
> * 一般在实际**生产环境**，这个batch的值**可以增大一些**来提升吞吐量，如果一个批次设置大了，会有延迟。一般根据一条消息大小来设置。如果我们消息比较少。配合使用的参数linger.ms，这个值默认是0，意思就是消息必须立即被发送，但是这是不对的，一般设置一个100毫秒之类的，这样的话就是说，这个消息被发送出去后进入一个batch，如果100毫秒内，这个batch满了16kb，自然就会发送出去。

###### 处理异常

> 1. LeaderNotAvailableException：这个就是如果某台机器挂了，此时leader副本不可用，会导致你写入失败，要等待其他follower副本切换为leader副本之后，才能继续写入，此时可以重试发送即可；如果说你平时重启kafka的broker进程，肯定会导致leader切换，一定会导致你写入报错，是LeaderNotAvailableException。
> 2. NotControllerException：这个也是同理，如果说Controller所在Broker挂了，那么此时会有问题，需要等待Controller重新选举，此时也是一样就是重试即可。
> 3. NetworkException：网络异常 timeout 
>    * a. 配置retries参数，他会自动重试的 
>    * b. 但是如果重试几次之后还是不行，就会提供Exception给我们来处理了,我们获取到异常以后，再对这个消息进行单独处理。我们会有**备用链路**。发送不成功的消息发送到**Redis**或者写到文件系统中，甚至是**丢弃**。

###### 重试机制

> 1. **消息会重复**有的时候一些**leader切换**之类的问题，需要进行重试，设置retries即可，但是消息重试会导致,重复发送的问题，
>    * 比如说网络抖动一下导致他以为没成功，就重试了，其实人家都成功了.
> 2. **消息乱序**消息重试是可能导致消息的乱序的，因为可能排在你后面的消息都发送出去了。所以可以使用"`max.in.flight.requests.per.connection`"参数设置为1， 这样可以`保证producer同一时间只能发送一条消息`。两次重试的间隔默认是100毫秒，用"retry.backoff.ms"来进行设置 基本上在开发过程中，靠重试机制基本就可以搞定95%的异常问题。

#### 7.Kafka消费者

###### 偏移量管理

> 1. 每个consumer内存里数据结构保存对每个topic的每个分区的消费offset，定期会提交offset，老版本是写入zk，但是那样高并发请求zk是不合理的架构设计，zk是做分布式系统的协调的，轻量级的元数据存储，不能负责高并发读写，作为数据存储。
> 2. 新版本提交offset发给kafka内部topic：__consumer_offsets，提交KV结构 
>    * key：group.id+topic+分区号，
>    * value：当前offset的值，
>    * 每隔一段时间，kafka内部会对这个topic进行compact(合并)，也就是每个group.id+topic+分区号就**保留最新数据**。
> 3. __consumer_offsets可能会接收高并发的请求，所以**默认分区50个**(leader partitiron -> 50 kafka)，这样如果你的kafka部署了一个大的集群，比如有50台机器，就可以用50台机器来抗offset提交的请求压力. 

###### 偏移量监控工具介绍

> 1. web页面管理的一个管理软件(kafka Manager) 修改bin/kafka-run-class.sh脚本，第一行增加JMX_PORT=9988 **重启kafka进程**
> 2. 另一个软件：主要监控的consumer的偏移量。就是一个jar包 java -cp KafkaOffsetMonitor-assembly-0.3.0-SNAPSHOT.jar com.quantifind.kafka.offsetapp.OffsetGetterWeb –offsetStorage kafka \（根据版本：偏移量存在kafka就填kafka，存在zookeeper就填zookeeper） –zk hadoop1:2181 –port 9004 –refresh 15.seconds –retain 2.days。

###### 消费异常感知

> * **heartbeat.interval.ms：consumer心跳时间间隔**，必须得与coordinator保持心跳才能知道consumer是否故障了， 如果故障之后，就会通过心跳下发rebalance的指令给其他的consumer通知他们进行rebalance的操作 
>
> * session.timeout.ms：默认是10秒，kafka多长时间感知不到一个consumer就认为他故障了，
>
> * max.poll.interval.ms：如果在两次poll操作之间，超过了这个时间，那么就会认为这个consume处理能力太弱了，会被踢出消费组，分区分配给别人去消费
>
> 一般来说结合业务处理的性能来设置就可以了。

###### 核心参数解释

> fetch.max.bytes：获取一条消息最大的字节数，一般建议设置大一些，**默认是1M** 其实我们在之前多个地方都见到过这个类似的参数，意思就是说一条信息最大能多大？
>
> 1. Producer 发送的数据，一条消息最大多大， -> **10M**
> 2. Broker 存储数据，一条消息最大能接受多大 -> **10M**
> 3. Consumer max.poll.records: 一次poll返回消息的最大条数，默认是**500条** 
>    * connection.max.idle.ms：consumer跟broker的socket连接如果空闲超过了一定的时间，此时就会自动回收连接，但是下次消费就要重新建立socket连接，这个建议设置为-1，不要去回收 
>    * enable.auto.commit: 开启自动提交偏移量 
>    * auto.commit.interval.ms: 每隔多久提交一次偏移量，默认值5000毫秒 
>    * _consumer_offset auto.offset.reset：
>      * **earliest** 
>        * 当各分区下有已提交的offset时，从提交的offset开始消费；
>        * 无提交的offset时，从头开始消费 topica -> partition0:1000 partitino1:2000 
>      * **latest** 
>        * 当各分区下有已提交的offset时，从提交的offset开始消费；
>        * 无提交的offset时，消费新产生的该分区下的数据 none topic各分区都存在已提交的offset时，从offset后开始消费；只要有一个分区不存在已提交的offset，则抛出异常

###### group coordinator原理

* 面试题：消费者是如何实现rebalance的？答：根据coordinator实现

> 1. 什么是coordinator
>
>    * **每个consumer group**都会选择一个broker作为自己的coordinator，他是负责监控这个**消费组里各个消费者**的**心跳**，以及**判断是否宕机**，然后**开启rebalance**的
>
> 2. 如何选择coordinator机器 
>
>    * 首先对groupId进行hash（数字），
>    * 接着对__consumer_offsets的分区数量取模，默认是50，_consumer_offsets的分区数可以通过offsets.topic.num.partitions来设置 
>    * __找到分区以后，这个分区所在的broker机器就是coordinator机器。__
>    * eg：比如说：groupId，“myconsumer_group” -> hash值（数字）-> 对50取模 -> 8 __consumer_offsets 这个主题的8号分区在哪台broker上面，那一台就是coordinator 就知道这个consumer group下的所有的消费者提交offset的时候是往哪个分区去提交offset，
>
> 3. 运行流程
>
>    1）每个consumer都发送JoinGroup请求到Coordinator， 然后Coordinator从一个consumer group中选择一个consumer作为leader,**（选择leader）**
>
>    2）把consumer group情况发送给leader，接着leader会负责制定消费方案， **(leader定制消费方案)**
>
>    3）通过SyncGroup发给Coordinator ,Coordinator就把消费方案下发给各个consumer，他们会从指定的分区的 leader broker,进行socket连接以及消费消息**(Coordinator把消费方案下发各个consumer)**

![image-20210507142400943](C:\Users\hason\AppData\Roaming\Typora\typora-user-images\image-20210507142400943.png)

######  rebalance策略

> consumer group靠coordinator实现了Rebalance，rebalance的策略：range、round-robin、**sticky**
>
> * **sticky策略：**尽可能保证在rebalance的时候，让原本属于这个consumer 的分区还是属于他们，然后把多余的分区再均匀分配过去，这样尽可能维持原来的分区分配的策略

#### 8.Broker管理

##### LEO、HW:

在kafka里面，无论leader partition还是follower partition统一都称作副本（replica）。

> LEO：每次partition接收到一条消息，都会更新自己的LEO，也就是log end offset，LEO其实就是最新的offset + 1
>
> HW：高水位 LEO有一个很重要的功能就是更新HW，如果follower和leader的LEO同步了，此时HW就可以更新 HW之前的数据对消费者是可见，消息属于commit状态。HW之后的消息消费者消费不到。

##### controller如何管理整个集群

> 1. 竞争controller的 /controller/id 
>
> 2. controller服务监听的目录：
>    * /broker/ids/ 用来感知 broker上下线 
>    * /broker/topics/ 创建主题，我们当时创建主题命令，提供的参数，ZK地址。
>    * /admin/reassign_partitions 分区重分

##### 延时任务（扩展知识）

> 1. 第一类延时的任务：
>    * 比如说producer的acks=-1，必须等待leader和follower都写完才能返回响应。有一个超时时间，**默认是30秒（request.timeout.ms）**。所以需要在写入一条数据到leader磁盘之后，就必须有一个延时任务，
>    * 30秒之后，到期时间是30秒延时任务 放到**DelayedOperationPurgatory（延时管理器）**中。
>    * 假如在30秒之前
>      * 如果所有follower都写入副本到本地磁盘了，那么这个任务就会被自动触发苏醒，就可以返回响应结果给客户端了， 否则的话，这个延时任务自己指定了最多是30秒到期，
>      * 如果到了超时时间，都没等到，就直接超时返回异常。
>
> 2. 第二类延时的任务：
>    * follower往leader拉取消息的时候，如果**发现是空的**，此时会**创建一个延时拉取任务** 延时时间到了之后（比如到了100ms），就**给follower返回一个空的数据**，然后**follower再次发送请求读取消息**，
>    * 但是如果延时的过程中(还没到100ms)，leader写入了消息(不为空)，这个任务就会**自动苏醒**，**自动执行拉取任务**。
>
> 海量的延时任务，需要去调度。

##### 时间轮机制

时间轮说白其实就是一个数组。

> Kafka内部有很多延时任务，没有基于JDK Timer来实现，那个插入和删除任务的时间复杂度是O(nlogn)， 而是基于了自己写的时间轮来实现的，时间复杂度是O(1)，依靠时间轮机制，延时任务插入和删除，O(1)
>
> **tickMs:**时间轮间隔 1ms 
>
> **wheelSize：**时间轮大小 20 
>
> **interval：**timckMS * whellSize，**一个时间轮的总的时间跨度**。20ms 
>
> **currentTime：**当时时间的指针。
>
> ​	1. 因为时间轮是一个数组，所以要获取里面数据的时候，靠的是index，时间复杂度是O(1) 
>
> ​	2. 数组某个位置上对应的任务，用的是**双向链表**存储的，往双向链表里面插入，删除任务，时间复杂度也是O（1） 
>
> 	* 举例：插入一个8ms以后要执行的任务 19ms 
>
> * **多层级的时间轮 ：**
>   * 比如：要插入一个110毫秒以后运行的任务。
>     * tickMs:时间轮间隔 20ms 
>     * wheelSize：时间轮大小 20 
>     * interval：timckMS * whellSize，一个时间轮的总的时间跨度。20ms 
>     * currentTime：当时时间的指针。
>     * **第一层时间轮：1ms * 20 ；第二层时间轮：20ms * 20 ；第三层时间轮：400ms * 20**



#### 9.case

```bash
#创建消费者

kafka-console-consumer.sh --zookeeper collector1:2181,collector2:2181,collector3:2181 --from-beginning --topic pad_report_data

#查看topic详情

kafka-topics.sh --describe --topic pad_report_data --zookeeper collector1:2181,collector2:2181,collector3:2181

#启动kafka

kafka-server-start.sh -daemon /wls/soft/kafka_2.10-0.10.1.1/config/server.properties

#创建消费者，新api

kafka-console-consumer.sh --bootstrap-server collector1:9092,collector2:9092,collector3:9092 --from-beginning --topic pad_report_data

#创建生产者及topic

kafka-console-producer.sh --broker-list collector1:9092,collector2:9092,collector3:9092 --topic demo_test

#删除topic

kafka-topics.sh --delete --zookeeper collector1:2181,collector2:2181,collector3:2181  --topic pad_report_data



zkCli.sh -server collector3:2181 

#查看分区offset

kafka-run-class.sh kafka.tools.GetOffsetShell --time -1 --broker-list collector1:9092,collector2:9092,collector3:9092 --topic pad_report_data  

#查看分区消费堆积量

kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --group group_id_1pad_report_data  --topic pad_report_data  --zookeeper collector1:2181,collector2:2181,collector3:2181



nohup flume-ng agent -c conf -f /wls/soft/apache-flume-1.7.0-bin/conf/collect-conf.properties -n producer02 -Dflume.monitoring.type=http -Dflume.monitoring.port=34545 >/wls/soft/apache-flume-1.7.0-bin/logs/cat.out 2>&1 &



auto.leader.rebalance.enable=true



kafka-preferred-replica-election.sh --zookeeper collector1:2181,collector2:2181,collector3:2181


## 查看日志
kafka-topics.sh --describe --topic pad_report_data --zookeeper collector1:2181,collector2:2181,collector3:2181 --unavailable-partitions

kafka-topics.sh --describe --topic pad_report_data --zookeeper collector1:2181,collector2:2181,collector3:2181 --under-replicated-partitions 

kafka-topics.sh --describe --topic pad_report_data --zookeeper collector1:2181,collector2:2181,collector3:2181 --topics-with-overrides

[my@hadoop102 kafka]$./bin/kafka-topics.sh --list(--describe, --create, --alter or --delete) --bootstrap-server hadoop102:9092 （话题操作）

[my@hadoop102 kafka]$./bin/kafka-console-producer.sh --topic first1 --broker-list hadoop102:9092,hadoop103:9092 （生产）

[my@hadoop102 kafka]$./bin/kafka-console-consumer.sh --topic first1 --bootstrap-server hadoop102:9092,hadoop103:9092 (--from-beginning从头消费) (g1)（消费）

##**
[my@hadoop102 kafka]$./bin/kafka-dump-log.sh --files logs/first1-0/00000000000000000000.log(index) print-data-log （查看log，index）

[my@hadoop102 kafka]$./bin/kafka-producer-perf-test.sh --topic first1 --num-records 1000000 --throughput -1 --record-size 1024 --producer-props bootstrap.server=hadoop102:9092 （自动产生数据）

[my@hadoop102 kafka]$./bin/kafka-consumer-groups.sh --all-groups --all-topics --describe --bootstrap-server hadoop102:9092 （查看消费者组信息）

# 配置

broker.id=2

advertised.host.name=collector2

host.name=collector2

auto.create.topics.enable=true

delete.topic.enable=true

default.replication.factor=2

num.network.threads=8

num.io.threads=16

num.partitions=9

num.recovery.threads.per.data.dir=1

socket.send.buffer.bytes=1048576

socket.receive.buffer.bytes=1048576

socket.request.max.bytes=104857600

delete.topic.enable=true

log.dirs=/wls/soft/kafka_2.10-0.10.1.1/logs

log.retention.hours=168

log.segment.bytes=1073741824

log.cleaner.enable=false

log.retention.check.interval.ms=300000

listeners=PLAINTEXT://:9092

zookeeper.connection.timeout.ms=20000

zookeeper.connect=collector1:2181,collector2:2181,collector3:2181

replica.lag.max.messages=100000

replica.lag.time.max.ms=60000

replica.fetch.wait.max.ms=3000

replica.socket.receive.buffer.bytes=524288

replica.fetch.max.bytes=5242880
```



### 16.kafka API

```xml
				<dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka_2.11</artifactId>
            <version>0.10.1.1</version>
        </dependency> 
```



```java
//获取 kafka lag

package com.asiainfo.kafka.monitor2;
 
import java.io.BufferedInputStream;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
 
import org.apache.commons.lang3.StringUtils;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.common.serialization.StringDeserializer;
 
import com.asiainfo.kafka.monitor.Initer;
/**
 * 
 * @author Administrator
 * 
 */
public class Init2 {
	
	public static String JDBC_DRIVER;
	public static String JDBC_URL;
	public static String JDBC_USER;
	public static String JDBC_PASSWORD;
	public static String FLAG;
	public static String THREAD_SLEEP_MS;
	public static String EXIT_TIMES;
	
	public static String ZOOKEEPER_CONNECT;
	public static String ZKPATH_KAFKA_GROUP_OFFSET;
	public static String ZKPATH_RESOURCE_DATA_SAVE;
	
	
	public static String SPARK_TAG_RESTART_COMMAND;
	public static String SPARK_TAG_STOP_COMMAND;
	
	public static String KAFKA_TOPIC_LAG_LIMT;
	public static Map<String, Long> LAG_LIMIT_MAP=new HashMap<>();
	
	public static String SPARK_PROVINCE_RESOURCE;
	public static Map<String, String> RESOURCE_MAP=new HashMap<>();
	
	public static KafkaConsumer<String, String> CONSUMER;
	public static KafkaProducer<String, String> PRODUCER;
	public static String[] TOPICS;
	public static String GROUP ;
	public static Properties props ;
	public static String produce_args=null;
	public static ExecutorService threadPool=null;
	
		
	/**
	 * 初始化参数
	 * @param arr
	 */
	//public static void init(String[] args) {
	static {
		
		props = new Properties();
		produce_args="1213";
		try {
			if (StringUtils.isNotBlank(produce_args)) {
				
				//String filePath = System.getProperty("user.dir") + "/init.properties";
//				InputStream inputStream = new BufferedInputStream(new FileInputStream(filePath));
//				props.load(inputStream);
				props.load(Initer.class.getClassLoader().getResourceAsStream("init.properties"));
 
				
//				System.setProperty("java.security.auth.login.config", props.getProperty("kafka.jaas.conf")); 
//				System.setProperty("java.security.krb5.conf", props.getProperty("kafka.krb5.conf")); 
				
//				props.put("security.protocol", "SASL_PLAINTEXT"); 
//				props.put("sasl.mechanism", "GSSAPI"); 
//				props.put("sasl.kerberos.service.name", "kafka");
				
				//System.out.println(props.getProperty("kafka.jaas.conf")+"-------------"+props.getProperty("kafka.krb5.conf"));
			}else{
				System.out.println("---------------------加载本地文件---------------------");
				props.load(Init2.class.getResourceAsStream("/initkafka.properties"));
				
			}
				
				
			
		} catch (Exception e) {
			e.printStackTrace();
 
		}
		
		
		threadPool = Executors.newFixedThreadPool(Integer.parseInt(props.getProperty("thread.pool.num")));
		
		SPARK_TAG_RESTART_COMMAND=props.getProperty("spark.tag.restart.command");
		SPARK_TAG_STOP_COMMAND=props.getProperty("spark.tag.stop.command");
		
		SPARK_PROVINCE_RESOURCE=props.getProperty("spark.province.resource");
		KAFKA_TOPIC_LAG_LIMT=props.getProperty("kafka.topic.lag.limt");
		setLagLimit(KAFKA_TOPIC_LAG_LIMT);
		setResource(SPARK_PROVINCE_RESOURCE);
		
		ZOOKEEPER_CONNECT=props.getProperty("zookeeper.connect");
		//ZKPATH_KAFKA_GROUP_OFFSET=props.getProperty("zkpath.kafka.group.offset");
		ZKPATH_RESOURCE_DATA_SAVE=props.getProperty("zkpath.resource.data.save");
		
		
		THREAD_SLEEP_MS = props.getProperty("thread.sleep.ms");
		EXIT_TIMES = props.getProperty("exit.times");
		
		
		JDBC_DRIVER = props.getProperty("jdbc-driver");
		JDBC_URL=props.getProperty("url");
		JDBC_USER=props.getProperty("user");
		JDBC_PASSWORD=props.getProperty("password");
		
		TOPICS = props.getProperty("kafka.topic").split(",");
		
		GROUP=props.getProperty("kafka.group", "group_29");
		props.put("bootstrap.servers", props.getProperty("kafka.host"));
		props.put("group.id", GROUP);
		props.put("enable.auto.commit", "false");
		props.put("auto.commit.interval.ms", "1000");
		props.put("max.poll.records", 100);
		props.put("session.timeout.ms", "30000");
		props.put("auto.offset.reset", "earliest");
		props.put("key.deserializer", StringDeserializer.class.getName());
		props.put("value.deserializer", StringDeserializer.class.getName());
		
    	System.out.println("kafka地址---" +props.getProperty("kafka.host")+"---主题：-----"+Arrays.toString(TOPICS)+"--消费组-----"+GROUP);
 
        
		CONSUMER = new KafkaConsumer<String, String>(props);
		
//		CONSUMER.subscribe(Arrays.asList(TOPICS));
	}
	/**
	 * 初始化RESOURCE_MAP
	 * @param resources 参数如： beijing|8g_1_2500_54_18|8g_1_3500_54_18,xian|8g_2_2500_54_20|8g_2_2500_54_20
	 */
	public static void setResource(String resources){
		String[] split = resources.split(",");
		for (String resc : split) {
			String[] split2 = resc.split("\\|",2);
			RESOURCE_MAP.put(split2[0], split[1]);
		}
	}
	public static void setLagLimit(String lagLimit){
		String[] split = lagLimit.split(",");
		for (String limt : split) {
			LAG_LIMIT_MAP.put(limt.split("\\|")[0], Long.parseLong(limt.split("\\|")[1]));
		}
		
	}
	
}
 
 
 
 
package com.asiainfo.kafka.monitor2;
 
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Map.Entry;
 
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.consumer.OffsetAndMetadata;
import org.apache.kafka.common.PartitionInfo;
import org.apache.kafka.common.TopicPartition;
 
import com.asiainfo.kafka.monitor2.TopicMessageInfo;
import com.asiainfo.kafka.service.TaskService;
 
public class KafkaUtils {
 
	public KafkaUtils() {
 
	}
 
	/**
	 * 获取所有的topic endOffset
	 * 
	 * @param topics 
	 * @return
	 */
	public List<Map<String, Long>> getAllEndOffset(String[] topics) {
 
		List<Map<String, Long>> endOffsetAllTopic = new ArrayList<>();
		int topicCount = 0;
		for (int i = 0; i < topics.length; i++) {
 
			KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(Init2.props);
 
			consumer.subscribe(Arrays.asList(topics[i]));
 
			Map<String, Long> endOffset = getEndOffset(consumer, topics[i]);
			
			endOffsetAllTopic.add(endOffset);
			topicCount++;
		}
		System.out.println("所有topic的endOffset--------------"+endOffsetAllTopic);
		System.out.println("--kafka---topic的个数---------" + topicCount);
		return endOffsetAllTopic;
 
	}
	
	/**
	 * 获取kafka单个topic消息写入总量
	 * 
	 * @param consumer
	 * @param topic
	 * @return
	 */
	public Map<String, Long> getEndOffset(KafkaConsumer<String, String> consumer, String topic) {
		Map<String, Long> topicEndOffset = new HashMap<>();
		// 获取分区信息
		List<PartitionInfo> partitionsFor = consumer.partitionsFor(topic);
 
		Collection<TopicPartition> partitions = new ArrayList<>();
		for (PartitionInfo partitionInfo : partitionsFor) {
			TopicPartition tp = new TopicPartition(partitionInfo.topic(), partitionInfo.partition());
			partitions.add(tp);
		}
 
		Map<TopicPartition, Long> endOffset = consumer.endOffsets(partitions);
		Long logSizeEndOffset = 0L;
		for (Map.Entry<TopicPartition, Long> topicPartition : endOffset.entrySet()) {
			logSizeEndOffset += topicPartition.getValue();
		}
		topicEndOffset.put(topic, logSizeEndOffset);
		System.out.println("------------kakfka写入消息总量----------" + topicEndOffset);
		return topicEndOffset;
 
	}
	
	/**
	 * 获取所有的topic commitedOffset
	 * 
	 * @param topics
	 * @return
	 */
	public Map<String, Long> getAllCommittedOffset(String[] topics) {
		
		Map<String, Long> committedOffsetMap = new HashMap<>();
		int topicCount = 0;
		for (int i = 0; i < topics.length; i++) {
			
			KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(Init2.props);
			
			consumer.subscribe(Arrays.asList(topics[i]));
			
			Long commitedOffset = getOffsetCommitted(topics[i], consumer);
			
			committedOffsetMap.put(topics[i],commitedOffset);
			topicCount++;
		}
		System.out.println("所有topic的committedOffset--------------"+committedOffsetMap);
		return committedOffsetMap;
		
	}
 
	
	/**
	 *  获取被提交的偏移量
	 * @param topicPartitionMap
	 * @param consumer
	 * @return
	 */
	
	
	public Long getOffsetCommitted (String topic, KafkaConsumer<String, String> consumer){
		
		Long commitedOffset=0L;
		List<PartitionInfo> partitionsFor = consumer.partitionsFor(topic);
		for (PartitionInfo partitionInfo : partitionsFor) {
			TopicPartition tp = new TopicPartition(partitionInfo.topic(), partitionInfo.partition());
			OffsetAndMetadata committed = consumer.committed(tp);
			commitedOffset +=committed.offset();
		}
		System.out.println("提交的偏移量------"+topic+"-----------"+commitedOffset);
		return commitedOffset;
		
	}
	
	
	
	
 
	/**
	 * 获取topic积压量
	 * 
	 * @param topicsCommitOffsetSize
	 *            所有topic的提交的偏移量总和
	 * @param topicEndOffset
	 *            topic的消息写入总量
	 * @return
	 */
	public Map<String, TopicMessageInfo> getTopicLagSize(String[] topics) {
		Map<String, TopicMessageInfo> TMImap = new HashMap<>();
		// Map<String, Long> endOffset = getEndOffset(consumer, topic);//单个topic
		List<Map<String, Long>> allEndOffset = getAllEndOffset(topics);//获取所有topic的消息写入
		System.out.println("所有的topic的endOffset"+allEndOffset);
		Map<String, Long> allCommittedOffset = getAllCommittedOffset(topics);
		Map<String, Long> topicsLagSize = new HashMap<>();//打印使用无具体用处
		for (Map<String, Long> e : allEndOffset) {
 
			for (Entry<String, Long> entry : e.entrySet()) {// 只有一个键值对
				Long lag = 0L;
				if (allCommittedOffset.containsKey(entry.getKey())) {
					Long tcommited = allCommittedOffset.get(entry.getKey());
					lag = entry.getValue() - tcommited;
					topicsLagSize.put(entry.getKey(), lag);
					TopicMessageInfo tmi = new TopicMessageInfo();
					tmi.setLogEndOffset(entry.getValue());
					tmi.setTopic(entry.getKey());
					tmi.setLag(lag);
					tmi.setCurrentTime(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()));
					tmi.setCurrentOffset(tcommited);
					//判断积压标准  C_DPI_beijing_4G   C_DPI_beijing
					Long lagLimit=0L;
					if(Init2.LAG_LIMIT_MAP.containsKey(entry.getKey())){
						lagLimit=Init2.LAG_LIMIT_MAP.get(entry.getKey());
					}
					if (lag <= lagLimit) {
						tmi.setLagLevel("0");//没有加压
						TaskService taskService=new TaskService(tmi);
						Init2.threadPool.execute(taskService);
						
					}else{
						tmi.setLagLevel("1");//有积压
						//启用线程 分配资源
						TaskService taskService=new TaskService(tmi);
						Init2.threadPool.execute(taskService);
					}
					TMImap.put(entry.getKey(), tmi);
					
 
				}
			}
		}
 
		System.out.println("------------kafka积压量-----------" + topicsLagSize);
 
		return TMImap;
 
	}
 
}
 
 
 
package com.asiainfo.kafka.monitor2;
 
import java.util.List;
import java.util.Map;
 
public class SparkTagMonitorMain {
	//private static final Logger logger = LoggerFactory.getLogger(Main.class);
 
	public static void main(String[] args) {
		while(true){
			run();
			try {
				Thread.sleep(2*60*1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		
	}
 
	public static void run() {
		//ZKUtils zkUtils = new ZKUtils();
		try {
			//System.out.println("zk地址-------------"+Init2.ZOOKEEPER_CONNECT);
			//zkUtils.connectZookeeper(Init2.ZOOKEEPER_CONNECT);
			//zkUtils.isExistNode(Init2.ZKPATH_KAFKA_GROUP_OFFSET);
			// /mykafka/offset/group_29/C_DPI_beijing_4G
			//List<String> topics = zkUtils.getChildren(Init2.ZKPATH_KAFKA_GROUP_OFFSET);
			//System.out.println("-------------主题-------" + topics);
			//Map<String, Long> topicsCommitedSize = zkUtils.getTopicsCommitedSize(zkUtils, topics, Init2.ZKPATH_KAFKA_GROUP_OFFSET);// 获取各topic消费总量
			KafkaUtils ku = new KafkaUtils();
 
			Map<String, TopicMessageInfo> topicConsumInfo = ku.getTopicLagSize(Init2.TOPICS);
			// logger.info("logger日志："+topicConsumInfo.toString());
			System.out.println("消费主题的信息------------------"+topicConsumInfo);
				
			MysqlCURD.batchSave(topicConsumInfo);
 
		} catch (Exception e) {
			e.printStackTrace();
		}
 
	}
 
}
 
```



```xml
<groupId>org.apache.kafka</groupId>
<artifactId>kafka_2.11</artifactId>
<version>0.10.1.1</version>
```

```java
package com.fengjr.elk.web.write;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;
import java.util.Map.Entry;

import kafka.api.PartitionOffsetRequestInfo;
import kafka.common.ErrorMapping;
import kafka.common.OffsetMetadataAndError;
import kafka.common.TopicAndPartition;
import kafka.javaapi.*;
import kafka.javaapi.consumer.SimpleConsumer;
import kafka.network.BlockingChannel;

public class KafkaOffsetTools {

    public static void main(String[] args) {

        String topic = "app-log-all-beta";
        String broker = "10.255.73.160";
        int port = 9092;
        String group = "fengjr-elk-group-es";
        String clientId = "Client_app-log-all-beta_1";
        int correlationId = 0;
        BlockingChannel channel = new BlockingChannel(broker, port,
                BlockingChannel.UseDefaultBufferSize(),
                BlockingChannel.UseDefaultBufferSize(),
                5000 );
        channel.connect();

        List<String> seeds = new ArrayList<String>();
        seeds.add(broker);
        KafkaOffsetTools kot = new KafkaOffsetTools();

        TreeMap<Integer,PartitionMetadata> metadatas = kot.findLeader(seeds, port, topic);

        long sum = 0l;
        long sumOffset = 0l;
        long lag = 0l;
        List<TopicAndPartition> partitions = new ArrayList<TopicAndPartition>();
        for (Entry<Integer,PartitionMetadata> entry : metadatas.entrySet()) {
            int partition = entry.getKey();
            TopicAndPartition testPartition = new TopicAndPartition(topic, partition);
            partitions.add(testPartition);
        }
        OffsetFetchRequest fetchRequest = new OffsetFetchRequest(
                group,
                partitions,
                (short) 0,
                correlationId,
                clientId);
        for (Entry<Integer,PartitionMetadata> entry : metadatas.entrySet()) {
            int partition = entry.getKey();
            try {
                channel.send(fetchRequest.underlying());
                OffsetFetchResponse fetchResponse = OffsetFetchResponse.readFrom(channel.receive().payload());
                TopicAndPartition testPartition0 = new TopicAndPartition(topic, partition);
                OffsetMetadataAndError result = fetchResponse.offsets().get(testPartition0);
                short offsetFetchErrorCode = result.error();
                if (offsetFetchErrorCode == ErrorMapping.NotCoordinatorForConsumerCode()) {
                } else {
                    long retrievedOffset = result.offset();
                    sumOffset += retrievedOffset;
                }
                String leadBroker = entry.getValue().leader().host();
                String clientName = "Client_" + topic + "_" + partition;
                SimpleConsumer consumer = new SimpleConsumer(leadBroker, port, 100000,
                        64 * 1024, clientName);
                long readOffset = getLastOffset(consumer, topic, partition,
                        kafka.api.OffsetRequest.LatestTime(), clientName);
                sum += readOffset;
                System.out.println(partition+":"+readOffset);
                if(consumer!=null)consumer.close();
            } catch (Exception e) {
                channel.disconnect();
            }
        }

        System.out.println("logSize："+sum);
        System.out.println("offset："+sumOffset);

        lag = sum - sumOffset;
        System.out.println("lag:"+ lag);


    }

    public KafkaOffsetTools() {
    }


    public static long getLastOffset(SimpleConsumer consumer, String topic,
                                     int partition, long whichTime, String clientName) {
        TopicAndPartition topicAndPartition = new TopicAndPartition(topic,
                partition);
        Map<TopicAndPartition, PartitionOffsetRequestInfo> requestInfo = new HashMap<TopicAndPartition, PartitionOffsetRequestInfo>();
        requestInfo.put(topicAndPartition, new PartitionOffsetRequestInfo(
                whichTime, 1));
        kafka.javaapi.OffsetRequest request = new kafka.javaapi.OffsetRequest(
                requestInfo, kafka.api.OffsetRequest.CurrentVersion(),
                clientName);
        OffsetResponse response = consumer.getOffsetsBefore(request);
        if (response.hasError()) {
            System.out
                    .println("Error fetching data Offset Data the Broker. Reason: "
                            + response.errorCode(topic, partition));
            return 0;
        }
        long[] offsets = response.offsets(topic, partition);
        return offsets[0];
    }

    private TreeMap<Integer,PartitionMetadata> findLeader(List<String> a_seedBrokers,
                                                          int a_port, String a_topic) {
        TreeMap<Integer, PartitionMetadata> map = new TreeMap<Integer, PartitionMetadata>();
        loop: for (String seed : a_seedBrokers) {
            SimpleConsumer consumer = null;
            try {
                consumer = new SimpleConsumer(seed, a_port, 100000, 64 * 1024,
                        "leaderLookup"+new Date().getTime());
                List<String> topics = Collections.singletonList(a_topic);
                TopicMetadataRequest req = new TopicMetadataRequest(topics);
                kafka.javaapi.TopicMetadataResponse resp = consumer.send(req);

                List<TopicMetadata> metaData = resp.topicsMetadata();
                for (TopicMetadata item : metaData) {
                    for (PartitionMetadata part : item.partitionsMetadata()) {
                        map.put(part.partitionId(), part);
                    }
                }
            } catch (Exception e) {
                System.out.println("Error communicating with Broker [" + seed
                        + "] to find Leader for [" + a_topic + ", ] Reason: " + e);
            } finally {
                if (consumer != null)
                    consumer.close();
            }
        }
        return map;
    }

}
```





## 四、HIVE

#### 解析/编译/优化/执行器

> 1. 解析器 SQL Parser：将SQL字符串转换为 **抽象语法树AST**（依赖第三方**antlr**库），对AST进行语法分析(例如：表、字段是否存在，sql语义是否有误)
> 2. 编译器Physical Plan：将AST编译成逻辑执行计划
> 3. 优化器Query Optimizer：对逻辑执行计划进行优化
> 4. 执行器Executor：把逻辑执行计划转换成可以执行的物理计划（MR程序）



## 五、sqoop

#### 参数

```
sqoop
--import
--connect 数据库URL
--username
--passwd
#目标路径及删除
--target-dir
--delete-target-dir
#空值处理
#mysql->hdfs 导入
--null-string '\\N'
--null-non-string '\\N'
#hdfs->mysql 导出
--input-null-string '\\N'
--input-null-non-string '\\N'
#指定分割符
--fields-terminated-dir '\t'
#sql
--query  "select .. from table ...and$CONDITIONS"

-----以下可选参数------
#切分字段、map数
--split-by id
--num-mappers 100

-----同步策略--------
全量：sql后加 where 1=1
新增：sql后加 where 创建时间=昨天 or 操作时间=昨天
	 --increment append （用在直接导入hive表的时候）
特殊：只到一次
新增及变化：用户表（拉链）
```



场景一：导入mysql表数据中，若出现问题(BUG、中断、出错)导致导入数据不正确

* 临时表，完成后再导入正式表，出问题就清空临时表再重新导入正式表

> --staging-table 
>
> --clear-staging-table

场景二：sqoop导数据慢

* 可将表进行指定字段切分

> split-by 指定字段【指定字段最好不是String字符串类型】

* 增加map个数(默认4map)

> --mappers 100 

场景三：1G 需要多久跑完

> 几分钟

场景四：ads层的Parquet格式表 导mysql ，报错

> 方案一： 在hive中将该表 加载到 textfile格式的临时表 再导入mysql
>
> 方案二： ads不要建parquet格式表





## 六、Azkaban

#### 自定义报警**onealter睿象云**

 微信、钉钉、微信、电话等等

> 电话需要调用电信运营商接口 **付费**
>
> 使用第三方运维平台 **onealter睿象云** 集成多种组件(ZABBIX、阿里云、restAPI) **付费**
>
> * 购买相应服务后，获取通知API。然后进行MailAlter二次开发

#### 异常处理

> 优先重跑：1 自动重跑，2 手动重跑    【注】重跑还失败：看日志 
>
> `yarn logs -applicationId "appId" | less`  =》 shift+g 
>
> * 【注】 “| less ” 为一页一页翻着看；G：跳到最后一页





## 七、Hbase

> 

#### Hbase中Region/Store/StoreFile/Hfile之间的关系

> 1. Table: Hbase的数据是以表的形式存在	
>
> 2. ColumnFamily列簇: 相当于mysql Table的字段,是Hbase的表结构		
>
>    * Column: 列, hbase的列 = 列簇:列限定符		
>    * Cell: 根据 rowkey+列簇:列限定符+时间戳 确定唯一一个cell		
>    * Row: 一行数据,主键相同的为一行数据
>
> 3. 列限定符: 放在列簇下,是数据的形式存在
>
> 4. version: 在创建表的时候再列簇上指定的版本号,version代表列簇下的列限定符中最多可以保存多少个历史数据	
>
> 5. namespace: 命名空间,相当于Mysql的库
>
> 6. table 在行的方向分割为多个region。region是Hbase中**分布式存储**  和 **负载均衡** 的**最小单元**
>
>    *即不同的 region可以分别在不同 的 regionServer上，但是同一个 region是不会拆分到多个 server上*
>
> 7. Store：每一个region有一个或多个store组成，至少是一个store，hbase会把一起访问的数据放在一个store里，即为每个columnFamily建一个store(即有几个columnFamily 就有几个store)。一个store由**一个memStore**和**0或多个storeFile**组成
>
>    * HBase以store的大小来判断是否需要切分region。
>
> 8. MemStore：内存，KV数据，刷写阈值默认64M -> 一个快照。专属线程负责刷写操作
>
> 9. StoreFile：memStore每次刷写后形成新StoreFile文件(以HFile格式保存)
>
> 10. HFile：KV数据，是hadoop二进制格式文件，一个StoreFile对应着一个HFile。而HFile是存储在HDFS之上的。
>
>     * HFile文件是不定长的，长度固定的只有其中的两块：Trailer和FileInfo。Trailer中又指针指向其他数据块的起始点，FileInfo记录了文件的一些meta信息。

#### Hbase架构

> * Master:
>   * 1、负责表的创建、删除、修改
>   * 2、监控regionserver的状态,一旦regionserver宕机，会将宕机的regionserver中的region进行故障转移
>   * 3、负责region的分配
> * Regionserver:
>   * 1、负责客户端的数据的增删改查
>   * 2、负责region split、storeFile compact等操作
>     * region： region保存在regionserver中
>     * Store: store的个数 = 列簇的个数
>     * memstore: 是一块内存区域,后续数据会先写入memstore中,等达到一定的条件之后,memstore会flush，flush的时候会生成storeFile
>     * storeFile: memstore flush生成,每 flush一次会生成一个storeFile，最终以HFile这种文件格式保存在HDFS

> * HLog: 预写日志,因为memstore是内存区域m,所以数据如果直接写到memstore中，可能因为regionserver造成数据丢失，所以后续写入的时候会将数据写入日志中,等到memstore数据丢失之后，可以从日志中回复数据【一个regionserver只有一个Hlog】
> * Zookeeper: master监控regionserver状态需要依赖zk

#### Hbase原理

##### 	1、Hbase元数据: hbase:meta

​		hbase元数据表中数据的rowkey = 表名,region起始rowkey,时间戳.region名称
​		hbase元数据中保存的有region的起始rowky,结束rowkey,所处于哪个regionserver
​		hbase元数据表只有一个region

##### 	2、写入流程

​		1、client会向zookeeper发起请求,获取元数据处于哪个regionserver
​		2、zookeeper会将元数据所在regionserver返回给client
​		3、client会向元数据所在的regionserver发起请求,获取元数据
​		4、元数据所在的regionserver返回元数据信息给client,client会缓存元数据信息
​		5、client会根据元数据得知数据应该写到哪个region，region处于哪个regionserver，会region所在的regionserver发起数据写入请求
​		6、数据首先写入HLog,然后再写入region的store中的memstore中
​		7、返回信息告知client写入完成

##### 	3、读取流程:

​		1、client会向zookeeper发起请求,获取元数据处于哪个regionserver
​		2、zookeeper会将元数据所在regionserver返回给client
​		3、client会向元数据所在的regionserver发起请求,获取元数据
​		4、元数据所在的regionserver返回元数据信息给client,client会缓存元数据信息
​		5、client会根据元数据得知数据应该写到哪个region，region处于哪个regionserver，会region所在的regionserver发起数据读取请求
​		6、首先会从block cache中获取数据,如果block cache中没有或者只有一部分数据,会再从memstore和storeFile中获取数据
​			从storeFile中读取数据的时候会根据布隆过滤器判断数据可能存在与哪些storeFile中，然后再根据HFile文件格式中的数据索引获取数据。
​			布隆过滤器特性: 如果判断存在不一定存在，如果判断不存在则一定不存在
​		7、获取到最终数据之后,如果需要缓存结果的话,会将当前查询结果缓存到block cache中，返回数据给客户端

blockCache 块缓存 内存存储 的 数据淘汰机制 基于 LRU算法

> 算法认为最近使用的数据是热门数据，下一次很大概率将会再次被使用。而最近很少被使用的数据，很大概率下一次不再用到。当缓存容量的满时候，优先淘汰最近很少使用的数据。

##### 	4、memstore flush的触发条件

​		**memstore flush的最小单位是region,不会只单独flush某个memstore,所以如果region中有多个store的话，可能在flush的时候会生成大量的小文件,所以工作中创建表的时候列簇的个数一般设置为1个,最多不超过两个**
​		1、region中某个memstore的大小达到128M,会触发flush
​		2、region中所有的memstore的总大小达到128*4,会触发flush
​		3、regionserver中所有的region中的所有的memstore的总大小达到javaHeap * 0.4 *0.95,会根据region中所有的memstore占用的内存大小排序,优先flush占用空间大的region
​		4、当写入特别频繁的时候,第三个触发条件可能会适当的延迟,延迟到regionserver中所有的region中的所有的memstore的总大小达到javaHeap * 0.4,会阻塞client的写入，会根据region中所有的memstore占用的内存大小排序,优先flush占用空间大的region，flush到占用总空间<=javaHeap * 0.4 *0.95的时候会停止flush，允许client写入
​		5、regionserver对应WAL日志文件数量达到32的时候，会flush
​		6、region距离上一次flush的时间超过一个小时，会自动flush
​		7、手动flush: flush '表名'

> ​		**memStore级别**： 某个memStore 达到 xxxx.flush.size = 128M 
> ​		**Region级别：**   region内部的所有memStore总和达到了 128M * 4 ， 阻塞写 
> ​		**RegionServer级别：** 公式 =》   堆内存 * 0.4 * 0.95  ，超过这个阈值，开始刷写， 由大到小依次刷写
> ​				刷写到什么时候： 小于 堆内存 * 0.4 * 0.95
> ​		**HLog数量：** 现在不用手动配置，上限 32个 
> ​		**定期刷写：** 默认1小时，最后一次编辑时间
> ​		**手动刷写：** 执行命令

##### 	5、storeFile compact

​		原因: memstore每flush一次会生成一个storeFile,storeFile文件越来越多会影响查询性能,所以需要将storeFile文件合并
​		minor compact:
​			触发条件: 符合文件数>=3 的时候会自动合并
​				符合文件数: 判断当前文件是否需要合并, 当前文件大小 <= sum(小于当前文件大小的所有文件) * 比例,如果满足该条件，则代表当前文件需要合并
​			合并的过程: 单纯的将小文件合并成大文件,在合并的过程中不会删除无效数据【无效版本数据、标记删除数据】
​			合并结果: 会有多个大文件
​		major compact:
​			触发条件: 7天一次
​			合并过程: 在合并的过程中会删除无效数据【无效版本数据、标记删除数据】
​			合并结果: 合并之后，store中只有一个文件

> 		为什么要合并： 每次刷写都生成一个新的 HFile ，时间久了很多小文件
> 		小合并：相邻的几个HFile，合并成一个新的大的HFile，原先的小HFile删除掉
> 				不会删除 被标记为删除、过期的数据
> 		大合并：所有的HFile，合并成一个大的HFile
> 				删除 被标记为删除、过期的数据
> 				注意：大合并会影响正常使用

> LSM Tree（Log-structured merge-tree）起源于1996年的一篇论文：The log-structured merge-tree (LSM-tree)。当时的背景是：为一张数据增长很快的历史数据表设计一种存储结构，使得它能够解决：在内存不足，磁盘**随机IO**太慢下的严重**写入**性能问题。
>
> ck tree是一个有序的树状结构，数据的写入流转从C0 tree 内存开始，不断被合并到磁盘上的更大容量的Ck tree上。由于内存的读写速率都比外存要快非常多，因此数据写入的效率很高。并且数据从内存刷入磁盘时是预排序的，也就是说，LSM树将原本的随机写操作转化成了顺序写操作，写性能大幅提升。不过它牺牲了一部分读性能，因为读取时需要将内存中的数据和磁盘中的数据合并。

##### 	6、region split

​		原因: 表在创建的时候默认只有一个region,后续所有的读写请求，都会落在该region上,随着时间流逝,region中保存的数据越来越多,导致后续region所在的regionserver可能出现读写负载的不均衡，所以需要将region切分,分配到不同的regionserver，从而分担负载压力
​		切分规则:
​			0.9版本之前: 当region中某个store中storeFile总大小达到10G的时候,该region会切分为两个
​			0.9版本-2.0版本:
​				当region中某个store中storeFile总大小达到 [N == 0 || N>100 ? 10G: min(10G,2*128M*N^3) ]的时候,该region会切分为两个
​				N代表region所属表在当前regionserver上的region个数
​			2.0版本之后:
​				当region中某个store中storeFile总大小达到 [N == 1 ? 2 *128M : 10G ]的时候,该region会切分为两个
​				N代表region所属表在当前regionserver上的region个数

#### Hbase优化

##### 	1、预分区: 

​		原因:Hbase在创建表的时候默认只有一个region,客户端如果读取/写入的并发量比较大，会造成regionserver负载压力，所以需要偶在创建表的时候多指定几个region,从而分担负载压力
​		如何预分区?
​			1、create '表名','列簇名',..,SPLITS=>['rowkey1','rowkey2',..]
​			2、create '表名','列簇名',..,{NUMREGIONS=>region个数,SPLITLGO=>'HexStringSplit'}
​			3、create '表名','列簇名',SPLIT_FILE=>'分区文件'
​			4、通过Api建表预分区
​				byte[][] splitKeys = {"rowkey".getBytes(),...}
​				1、admin.createTable(table,splitKeys)
​				byte[] startRow;
​				byte[] stopRow;
​				numRegions指表的region个数
​				2、admin.createTable(table,startRow,stopRow,numRegions)

> 指定分区键：
> 		建表语句，指定 splits => { 001|,002|,003|}
> 			为什么给 | ，因为 rowkey是 按位比较，按照 ASC码，   _比较小的值， |比较大的值
>
> 		注意：即使指定了分区键，后续还是会进行自动切分 直接按照10G切
> 			-∞ ~ 001  ====》 -∞ ~ 中间rowkey  ，  中间rowkey ~ 001
> 			001 ~ 002 
> 			002 ~ +∞

##### 	2、rowkey设计

​		原则:
​			1、唯一性原则: 两条数据的rowkey不能相同
​			2、长度原则: 一般保持在16字节以下
​				hbase rowkey不能太长,太长会导致rowkey占用过多的存储空间,client会缓存元数据，如果rowkey过长,而clint缓存空间不太足,就会导致缓存的元数据相对比较少
​			3、hash原则: 保证数据均衡分配在不同的region中
​		如果rowkey设计不合理可能会导致数据全部聚在一个region中，从而出现数据热点问题
​		如何解决数据热点问题?
​			1、在原来的rowkey基础上添加随机数
​			2、将rowkey反转
​			3、将原来rowkey hashcode值作为新rowkey

##### 	3、内存优化:

​		 一般会给hbase节点分配16-48G的内存

##### 	4、基础优化

​		**1、允许HDFS追加内容**
​			Hbase写入数据到HDFS的时候是以追加的形式写入，HDFS默认就是允许的
​		**2、调整datanode最大文件打开数**
​			HBase读取数据与写入数据都需要打开HDFS文件，如果同一时间有大量的读写请求,就可能导致HDFS文件打开数不够造成请求需要排队的情况
​		**3、调整HDFS数据写入的超时时间**
​			Hbase写入数据的时候，如果网络不是太好，可能造成超时导致数据写入失败
​		**4、调整HDFS数据写入效率**
​			如果数据量相对比较大，此时可以将数据压缩之后写入HDFS,能够提高效率
​		**5、调整RPC监听数**
​			Hbase内部有一个线程池专门用来处理client的请求,如果同一时间client有大量的请求,此时肯能造成线程池中的线程不够用,从而出现请求排队的情况
​		**6、调整HStore的文件大小**
​			HStore文件的大小如果过小，可能导致频繁的split,如果过大，可能导致region里面存放大量数据，负载不均衡
​		**7、调整Client缓存**
​			client在获取元数据之后，会缓存在客户端，后续在获取元数据的时候可以直接从缓存中获取,如果缓存的大小比较小,会导致client只能缓存一部分元数据,后续在获取元数据的时候可能就不能从缓存中直接获取到,需要从regionserver中获取
​		**8、flush、compact、split参数值调整**
​			hbase.regionserver.global.memstore.size.lower.limit一般需要调整,调整为0.7/0.8
​		**9、scan扫描的数据条数**
​			scan是全表扫描,如果扫描的数据条数比较大会占用过多的内存空间

##### 5、数据倾斜与数据热点

​		数据倾斜：存储位置、大小 
​		热点：数据本身分布不均

#### Phoenix

	1、shell使用
			1、查询所有表: !tables
			2、创建表
				1、hbase有表[在phoenix中创建表与hbase的表建立映射关系]
					1、通过create table建表与hbase的表建立映射关系
						create table创建的表在删除的时候会同步删除hbase的表
						create table创建的表可以增删改查数据
						CREATE TABLE hbase表名(
							字段名 类型 primary key, --映射hbase的rowkey
							列簇名.列限定符 类型, --映射hbase表中column
							...
						)COLUM_ENCODED_BYTES=0			
				2、通过create view建视图与hbase的表建立映射关系
					create view创建的视图在删除的时候不会删除hbase的表
					create view创建的视图只能查询数据
					CREATE VIEW hbase表名(
						字段名 类型 primary key, --映射hbase的rowkey
						列簇名.列限定符 类型, --映射hbase表中column
						...
					)COLUM_ENCODED_BYTES=0
				注意: 如果列簇名与列限定符以及表名是小写需要用""括起来
			2、hbase没有表[在phoenix中创建表的同时会在hbase中创建一个同名表]
				单主键:
				CREATE TABLE 表名(
					字段名 字段类型 primary key,
					...
				)COLUM_ENCODED_BYTES=0
				复合主键:
				CREATE TABLE 表名(
					字段名 字段类型,
					...
					CONSTRAINT 主键名 primary key(字段名,..)
				)COLUM_ENCODED_BYTES=0
				
				注意事项:
					1、COLUM_ENCODED_BYTES=0 是指在创建hbase表的时候列限定符不进行编码
					2、在创建表的时候phoenix会默认将小写转成大写[不管是字段还是表名],如果想要保持表名/字段小写需要将表名/字段名用""括起来
		3、插入/修改数据: upsert into 表名(字段名,..) values(值,..) [如果表名/字段名是小写需要用""括起来]
		4、查询: select .. from .. where .. [如果表名/字段名是小写需要用""括起来]
		5、如果想要映射命名空间的表
			1、在hbase/conf/hbase-site.xml中配置两个参数
			<property>
					<name>phoenix.schema.isNamespaceMappingEnabled</name>
					<value>true</value>
			</property>
			<property>
					<name>phoenix.schema.mapSystemTablesToNamespace</name>
					<value>true</value>
			</property>
			2、在client端配置两个参数
			<property>
					<name>phoenix.schema.isNamespaceMappingEnabled</name>
					<value>true</value>
			</property>
			<property>
					<name>phoenix.schema.mapSystemTablesToNamespace</name>
					<value>true</value>
			</property>
			3、重启hbase
			4、创建一个schema,schema的名称必须与命名空间的名称一致
			5、在phoenix中创建表与命名空间的表建立映射关系
					CREATE TABLE schema名称.hbase表名(
						字段名 类型 primary key, --映射hbase的rowkey
						列簇名.列限定符 类型, --映射hbase表中column
						...
					)COLUM_ENCODED_BYTES=0

> 3、**hbase二级索引**
> 	原因: hbase使用**get查询**的是可以**通过rowkey+元数据**知道数据处于哪个region，该region处于哪个regionserver，所以能够很快定位到数据。但是如果是根据value值来查询数据的时候使用的scan查询，**scan是扫描全表**，所以查询速度比较慢
> 	1、全局二级索引
> 		1、语法: **create index 索引名 on schema名称.表名([列簇名.]字段名,..) [include([列簇名.]字段名,..)]**
> 		2、原理: 
> 			给某个/某几个字段建索引之后，会在hbase中新创建一个索引表,后续根据索引字段查询数据的时候，此时是从索引表中查询数据
> 			新创建的索引表: 
> 				rowkey = 索引字段值1_索引字段值2_.._原来的rowkey
> 		3、注意事项:
> 			在查询数据的时候，查询条件中必须包含第一个索引字段,select与from中的查询字段必须在索引表中能够全部找到，不然就是全部扫描
> 	2、本地二级索引
> 		1、语法:create local index 索引名 on schema名称.表名([列簇名.]字段名,..)
> 		2、原理: 给某个/某几个字段建索引之后,会在原表中插入对应的数据[数据的rowkey=__索引字段值1_索引字段值2_.._原来的rowkey],后续在根据索引字段查询数据的时候会首先扫描索引对应的region获取原来的rowkey,再通过原来的rowkey查询原始数据
> 	3、**协处理器**（二级索引基于协处理器）
> 		协处理器相当于是一个触发器，会监控表，一旦client做出对应的操作，会触发对应的动作
> 		步骤:
> 			1、需要继承两个接口
> 			2、重写一个对应的方法
> 			3、重写一个对应的动作方法
> 			4、打包,上传到hdfs
> 			5、禁用表
> 			6、加载协处理器
> 			7、启动表
>
> ```
> Phoenix （或ES）：基于协处理器（写之后）
> 		全局索引： 索引表和 数据表 不在一个地方
> 			create index 索引表名  ON 数据表名 (索引列) include（其他列）
> 			索引表的id =  索引列_原始rowkey ，如果有include的列，也会在里面
> 		本地索引： 索引数据 和 数据表 在同一个region
> 			CREATE LOCAL INDEX 索引表名 ON 数据表名 (索引列) include（其他列）;
> 			在原表中插入索引数据：
> 				分区键_索引列_原始rowkey，		多加一列IDX ： 1表示对应数据在， 0表示对应数据过期了
> 				
> 		如果直接往HBase写数据，Phoenix获取不到
> 		补救：往Phoenix再写一份
> 		
> 		建议：写的时候通过Phoenix写
> ```
>
> 

#### 与hive集成

> 1、内部表
> 		在hive创建表的时候会同步在hbase中创建表，如果hbase的表已经存在则报错
> 		删除hive内部表的时候会同步删除Hbase的表
> 			CREATE TABLE student_hive_2(id string,name string,age string) 
> 			STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
> 			WITH SERDEPROPERTIES ("hbase.columns.mapping" = ":key,f1:name,f1:age")
> 			TBLPROPERTIES ("hbase.table.name" = "student_hbase");
> 		:key  指hbase的主键
> 		hbase.columns.mapping： 指定hive的字段与hbase的column的映射
> 		hbase.table.name： 指定创建的hbase的表名
> 	2、外部表[与hbase的表建立映射关系]
> 		在hive创建表的时候要求hbase的表必须已经存在,如果hbase的表不存在则报错
> 		删除hive外部表的时候不会删除hbase的表
> 			CREATE EXTERNAL TABLE student_hive_2(id string,name string,age string) 
> 			STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
> 			WITH SERDEPROPERTIES ("hbase.columns.mapping" = ":key,f1:name,f1:age")
> 			TBLPROPERTIES ("hbase.table.name" = "student_hbase");
> 		:key  指hbase的主键
> 		hbase.columns.mapping： 指定hive的字段与hbase的column的映射
> 		hbase.table.name： 指定hbase的表名



## 八、Flink

### 编程模型

* source -> transform -> sink

### flink 内存模型

![image](https://github.com/UncertaintyDeterminesYou4ndMe/memo/assets/72533078/712e1d3f-242d-452b-a301-e0d1835bfebe)

#### JVM 进程总内存（Total Process Memory）

该区域表示在容器环境下，TaskManager 所在 JVM 的最大可用的内存配额，包含了本文后续介绍的所有内存区域，超用时可能被强制结束进程。我们可以通过 taskmanager.memory.process.size 参数控制它的大小。
例如我们设置 JVM 进程总内存为 4G，TaskManager 运行在 Kubernetes 平台，则 Pod 配置的 spec -> resources -> limits -> memory 项会被设置为 4Gi

而对于 YARN，如果 yarn.nodemanager.pmem-check-enabled 设为 true, 则也会在运行时定期检查容器内的进程是否超用内存。
如果进程总内存用量超出配额，容器平台通常会直接发送最严格的 SIGKILL 信号（相当于 kill -9）来中止 TaskManager，此时不会有任何延期退出的机会，可能会造成作业崩溃重启、外部系统资源无法释放等严重后果。
因此，在 有硬性资源配额检查 的容器环境下，请务必妥善设置该参数，对作业充分压测后，尽可能预留一部分安全余量，避免 TaskManager 频繁被 KILL 而导致的作业频繁重启。

#### Flink 总内存（Total Flink Memory）

该内存区域指的是 Flink 可以控制的内存区域，即上述提到的 JVM 进程总内存 减去 Flink 无法控制的 Metaspace（元空间）和 Overhead（运行时开销）区域。Flink 随后又把这部分内存区域划分为堆内、堆外（Direct）、堆外（Managed）等不同子区域，后面我们会逐一讲解他们的配置指南。
对于没有硬性资源限制的环境，我们建议使用 taskmanager.memory.flink.size 参数来配置 Flink 总内存的大小，然后 Flink 自己也会自动根据参数，计算得到各个子区域的配额。如果作业运行正常，则无需单独调整。
例如 4G 的 进程总内存 配置下，JVM 运行时开销（Overhead）占 进程总内存 的 10% 但最多 1G（下图是 409.6M），元空间（Metaspace）占 256M；堆外直接（Direct）内存网络缓存占 Flink 总内存 的 10% 但最多 1G（下图是 343M），框架堆和框架堆外各占 128M，堆外管控（Managed）内存占 Flink 总内存 的 40%（下图是 1372M 即 1.34G），其他空间留给任务堆，即用户程序代码可以使用的内存空间（1459M 即 1.42G），我们接下来会讲到它。

#### JVM 堆内存（JVM Heap Memory）

堆内存大家想必都不陌生，它是由 JVM 提供给用户程序运行的内存区域，JVM 会按需运行 GC（垃圾回收器），协助清理失效对象。
当任务启动时，ProcessMemoryUtils#generateJvmParametersStr 方法会通过 -Xmx-Xms 参数设置堆内存的最大容量。
Flink 将堆内存从逻辑上划分为 ”框架堆“、”任务堆“ 两个子区域，分别通过 taskmanager.memory.framework.heap.size 和 taskmanager.memory.task.heap.size 来指定其大小：框架堆默认是 128m，任务堆如果未显式设置其大小，则会通过扣减其他区域配额来计算得到。例如对于 4G 的进程总内存，扣除了其他区域后，任务堆可用的只有不到 1.5G。
但需要注意的是，Flink 自身并不能精确控制框架自身及任务会用多少堆内存，因此上述配置项只提供理论上的计算依据。如果实际用量超出配额，且 JVM 难以回收对象释放空间，则会抛出 OutOfMemoryError，此时 Flink TaskManager 会退出，导致作业崩溃重启。因此对于堆内存的监控是必须要配置的，当堆内存用量超过一定比率，或者 Full GC 时长和次数明显增长时，需要尽快介入并考虑扩容。
高级内容：对于使用 HashMapStateBackend（旧版本称之为 FileSystem StateBackend）的流作业用户，如果在进程总内存固定的前提下，希望尽可能提升任务堆的空间，则可以减少 托管内存（Managed Memory）的比例。我们接下来也会讲到它。

#### JVM 堆外内存（JVM Off-Heap Memory）

广义上的 堆外内存 指的是 JVM 堆之外的内存空间，而我们这里特指 JVM 进程总内存除了元空间（Metaspace）和运行时开销（Overhead）以外的内存区域。因为上述两个区域是 JVM 自行管理，Flink 无法介入，我们后面单独划分和讲解。

托管内存（Managed Memory）

文章开头的总览图中，把托管内存区域设为 0，此时任务堆空间约 3G；而使用 Flink 默认配置时，任务堆只有 1.5G。这是因为默认情况下，托管内存占了 40% 的 Flink 总内存，导致堆内存可用的量变的相当少。因此我们非常有必要了解什么是托管内存。
从官方文档和 Flink 源码上来看，托管内存主要有三大使用场景：
批处理算法，例如排序、HashJoin 等。他们会从 Flink 的 MemoryManager 请求内存片段（MemorySegment），而 MemoryManager 则会调用 UNSAFE.allocateMemory 分配堆外内存。
RocksDB StateBackend，Flink 只会预留一部分空间并扣除预算，但是不介入实际内存分配。因此该类型的内存资源被称为 OpaqueMemoryResource. 实际的内存分配还是由 JNI 调用的 RocksDB 自己通过 malloc 函数申请。
PyFlink。与 JNI 类似，在与 Python 进程交互的过程中，也会用到一部分托管内存。
显然，对于普通的流式 SQL 作业，如果启用了 RocksDB 状态后端时，才会大量使用托管内存。因此如果您的业务场景并未用到 RocksDB，那么可以调小托管内存的相对比例（taskmanager.memory.managed.fraction）或绝对大小（taskmanager.memory.managed.size），以增大任务堆的空间。
对于 RocksDB 作业，之所以分配了 40% Flink 总内存，是因为 RocksDB 的内存用量实在是一个很头疼的问题。早在 2017 年，就有 FLINK-7289: Memory allocation of RocksDB can be problematic in container environments [6] 这个问题单，随后社区对此做了大量的工作（通过 LRUCache 参数、增强 WriteBufferManager 的 Slot 内空间复用等），来尽可能地限制 RocksDB 的总内存用量。在我之前的 Flink on RocksDB 参数调优指南 [7] 文章中，也有提到 RocksDB 内存调优的各项参数，其中 MemTable、Block Cache 都是托管内存空间的用量大户。

#### JVM 元空间（JVM Metaspace）

JVM Metaspace 主要保存了加载的类和方法的元数据，Flink 配置的参数是 taskmanager.memory.jvm-metaspace.size，默认大小为 256M，JVM 参数是 -XX:MaxMetaspaceSize.
如果用户编写的 Flink 程序中，有大量的动态类加载的需求，例如我们之前遇到过一个用户作业，动态编译并加载了 44 万个类，此时就容易出现元空间用量远超预期，发生 OOM 报错。此时就需要适当调大元空间的大小，或者优化用户程序，及时卸载无用的 Classloader。

#### JVM 运行时开销（JVM Overhead）

除了上述描述的内存区域外，JVM 自己还有一小块 ”自留地“，用来存放线程栈、编译的代码缓存、JNI 调用的库所分配的内存等等，Flink 配置参数是 taskmanager.memory.jvm-overhead.fraction，默认是 JVM 总内存的 10%。
对于旧版本（1.9 及之前）的 Flink，RocksDB 通过 malloc 分配的内存也属于 Overhead 部分，而新版 Flink 把这部分归类到托管内存（Managed），但由于 FLINK-15532 Enable strict capacity limit for memory usage for RocksDB [9] 问题仍未解决，RocksDB 仍然会少量超用一部分内存。
因此在生产环境下，如果 RocksDB 频繁造成内存超用，除了调大 Managed 托管内存外，也可以考虑调大 Overhead 区空间，以留出更多的安全余量。

### 算子
##### keyBy:

> 两次hash：第一次hashcode，第二次murmurhash  
>
> 然后 跟最大并行度 计算 =》 得到keyGroupbyId(键组对，一个键多个value)
>
> 计算公式：id * 下游并行度 / 最大并行度
>
> 【最大并行度 默认128M，env.setMaxParallelism(128)可修改】

##### intervalJoin：

> 底层 ： connect+keyBy
>
> 两条流都存在一个**Map类型** 的状态 数据存放里边，key是ts，value是数据的集合
>
> 每条流的数据来，都遍历对方的Map状态
>
> 到达一定条件，会清理状态	                                                                                                                                                                                                                        

#### slot与并行度的关系

> env.readtextfile.flatmap.keyby.sum.print
>
> * **同**一个算子的子任务，**不能**存在于同一个slot中，
>
> * **不同**算子的子任务，**可以**存在于同一个slot中
>
> slot可以共享，slotSharingGroup

#### JM/TM内存分配

> 设置依据：顶住洪峰就可以
>
> 百万日活：百万qps 洪峰 600/700万每秒

#### solt核数

> 最好与机器核数1：1，若计算逻辑简单 可以 1核：2slot

---

### flink 消费 kafka 实现精确一致性

```
1、第一条数据来了之后，开启一个kafka的事务，正常写入kafka分区日志但是标记为未提交，这就是未提交。
2、jobmanager触发checkpoint操作，barrier从source开始向下传递，遇到barrier的算子将状态存入状态后端，并通知jobmanager
3、sink连接器收到barrier，保存当前状态，存入checkpoint，通知jobmanager，并开启下一阶段的事务，用于提交下一个检查点的数据。
4、jobmanager收到所有任务的通知，发出确认信息，表示checkpoint完成
5、sink任务收到jobmanager的确认信息，正式提交这段时间的数据
外部kafka关闭事务，提交的数据可以正常消费了。
```


### flink checkpoint & savepoint

1.总览savepoints是外部存储的自包含的checkpoints，可以用来stop and resume，或者程序升级。savepoints利用checkpointing机制来创建流式作业的状态的完整快照（非增量快照），将checkpoint的数据和元数据都写入到一个外部文件系统。如何触发、恢复或者释放savepoint了？下面一一道来。

2.分配Operator ID极度推荐你给每个方法分配一个uid，这样才可以升级应用。ID起到的作用是明确每个operator的状态的使用范围。[![复制代码](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)`DataStream<String> stream = env.  // Stateful source (e.g. Kafka) with ID  .addSource(new StatefulSource())  .uid("source-id") // ID for the source operator  .shuffle()  // Stateful mapper with ID  .map(new StatefulMapper())  .uid("mapper-id") // ID for the mapper  // Stateless printing sink  .print(); // Auto-generated ID`[![复制代码](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)如果不手动指定ID，则系统会自动创建。如果这些ID不变，则应用可以被自动恢复出来。但是自动创建的ID依赖于应用的结构，任何应用的变动都可能导致ID的变化。所以，请手动分配ID吧。Savepoint State什么是savepoint了？savepoint对每个有状态的operator保存一个map的KV结构，`Operator ID -> State` 。`Operator ID | State ------------+------------------------ source-id   | State of StatefulSource mapper-id   | State of StatefulMapper`上面代码片段的print方法没有savepoint结构，因为他是无状态的。通常，会尝试将map的每个entry都恢复回去。

3.Operations可以利用cli来触发savepoint，或者cancel一个作业的同时做savepoint，或者从某个savepoint恢复，或者释放savepoint。如果Flink版本大于1.2.0，则可以通过webui来恢复savepoints。Triggering Savepoints当触发savepoint的时候，新的savepoint目录就会被创建，数据和元信息都会保存在这里。保存的位置可以是默认的目录，也可以是trigger命令指定的目录。但要注意，这个目录需要是JM和TM都可以访问的目录。举例，对于`FsStateBackend` 或者 `RocksDBStateBackend而言：`[![复制代码](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)`# Savepoint target directory /savepoints/ # Savepoint directory /savepoints/savepoint-:shortjobid-:savepointid/ # Savepoint file contains the checkpoint meta data /savepoints/savepoint-:shortjobid-:savepointid/_metadata # Savepoint state /savepoints/savepoint-:shortjobid-:savepointid/...`[![复制代码](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)通常，不建议移动savepoints到别的地方，因为_metadata里面有绝对路径。但是在使用MemoryStateBackend的时候，元信息和数据会一起被存入_metadata文件，所以可以移动。

Trigger a Savepoint`$ bin/flink savepoint :jobId [:targetDirectory]`如果命令会以jobid触发一次savepoint，返回的是本次savepoint的路径，这个路径可以用来恢复或者释放savepoint。

Trigger a Savepoint with YARN

```bash
bin/flink savepoint :jobId [:targetDirectory] -yid :yarnAppId
```



以jobId和yarnAppId来触发savepoint。

Cancel Job with Savepoint

`$ bin/flink cancel -s [:targetDirectory] :jobId`取消作业的同时触发一次savepoint。

Resuming from Savepoints

`$ bin/flink run -s :savepointPath [:runArgs]`

```bash
bin/flink run -m yarn-cluster -ym $jobName -s $savepoint_path jar
```



这样提交作业就会让作业在指定的savepoint恢复出来，路径可以是savepoint的目录，也可以是_metadata的文件地址。

Allowing Non-Restored State通常，恢复意味着savepoint的每一个状态都要恢复到应用中去，但如果你恰好去掉了某个operator，你可以通过设置来忽略这个状态，

--allowNonRestoredState。

`$ bin/flink run -s :savepointPath -n [:runArgs]`

Disposing Savepoints`$ bin/flink savepoint -d :savepointPath`

如上，就释放了存储在savepointPath位置的savepoint。其实也可以手动删除某个savepoint，这通过常规的文件系统操作就可以做到，并且不影响别的savepoints和checkpoints。Configuration可以通过配置项state.savepoints.dir来定义一个默认的savepoint存储目录。当触发savepoints的时候，这个目录就会被用来存储savepoint，但是你也可以通过在trigger命令中指定目录来覆盖默认设置。

`# Default savepoint target directory state.savepoints.dir: hdfs:///flink/savepoints`如果既没有默认目录，也没有指定目录，则触发savepoint就会失败。

4.FAQ是否需要给所有的operator指定ID？原则上，只需要给有状态的operator设置id就可以。但建议给所有的operator都设置。如果新的应用添加了一个有状态的operator会怎样？应用恢复的时候，新添加的operator会被当做没有状态来处理。如果删除了一个了？默认，恢复作业是所有状态都要恢复，删除了一个就会导致恢复失败，除非你指定可以忽略，见上面。如果改变了operator的顺序了？如果是你手动指定的id，则恢复不受影响。如果是自动生成的，改变了顺序往往也意味着id的改变，所以恢复会失败。如果添加、删除或者改变了没有状态的operator的顺序了？同4，手动设置了id则不受影响，否则会失败。如果改变了应用的并行度了？对于版本在1.2.0之后的，没影响？版本在之前怎么办？只能将应用和savepoint都升级到1.2.0之后。

### Flink 参数配置和常见参数调优

### Flink参数配置

- jobmanger.rpc.address jm的地址。
- jobmanager.rpc.port jm的端口号。
- jobmanager.heap.mb jm的堆内存大小。不建议配的太大，1-2G足够。
- taskmanager.heap.mb tm的堆内存大小。大小视任务量而定。需要存储任务的中间值，网络缓存，用户数据等。
- taskmanager.numberOfTaskSlots slot数量。在yarn模式使用的时候会受到`yarn.scheduler.maximum-allocation-vcores`值的影响。此处指定的slot数量如果超过yarn的maximum-allocation-vcores，flink启动会报错。在yarn模式，flink启动的task manager个数可以参照如下计算公式：

> num_of_tm = ceil(parallelism / slot)  即并行度除以slot个数，结果向上取整。

- parallelsm.default 任务默认并行度，如果任务未指定并行度，将采用此设置。
- web.port Flink web ui的端口号。
- jobmanager.archive.fs.dir 将已完成的任务归档存储的目录。
- history.web.port 基于web的history server的端口号。
- historyserver.archive.fs.dir history server的归档目录。该配置必须包含`jobmanager.archive.fs.dir`配置的目录，以便history server能够读取到已完成的任务信息。
- historyserver.archive.fs.refresh-interval 刷新存档作业目录时间间隔
- state.backend 存储和检查点的后台存储。可选值为rocksdb filesystem hdfs。
- state.backend.fs.checkpointdir 检查点数据文件和元数据的默认目录。
- state.checkpoints.dir 保存检查点目录。
- state.savepoints.dir save point的目录。
- state.checkpoints.num-retained 保留最近检查点的数量。
- state.backend.incremental 增量存储。
- akka.ask.timeout Job Manager和Task Manager通信连接的超时时间。如果网络拥挤经常出现超时错误，可以增大该配置值。
- akka.watch.heartbeat.interval 心跳发送间隔，用来检测task manager的状态。
- akka.watch.heartbeat.pause 如果超过该时间仍未收到task manager的心跳，该task manager 会被认为已挂掉。
- taskmanager.network.memory.max 网络缓冲区最大内存大小。
- taskmanager.network.memory.min 网络缓冲区最小内存大小。
- taskmanager.network.memory.fraction 网络缓冲区使用的内存占据总JVM内存的比例。如果配置了`taskmanager.network.memory.max`和`taskmanager.network.memory.min`，本配置项会被覆盖。
- fs.hdfs.hadoopconf hadoop配置文件路径（已被废弃，建议使用HADOOP_CONF_DIR环境变量）
- yarn.application-attempts job失败尝试次数，主要是指job manager的重启尝试次数。该值不应该超过`yarn-site.xml`中的`yarn.resourcemanager.am.max-attemps`的值。

### Flink HA(Job Manager)的配置

- high-availability: zookeeper 使用zookeeper负责HA实现
- high-availability.zookeeper.path.root: /flink flink信息在zookeeper存储节点的名称
- high-availability.zookeeper.quorum: zk1,zk2,zk3 zookeeper集群节点的地址和端口
- high-availability.storageDir: hdfs://nameservice/flink/ha/ job manager元数据在文件系统储存的位置，zookeeper仅保存了指向该目录的指针。

### Flink metrics 监控相关配置

- metrics.reporters: prom
- metrics.reporter.prom.class: org.apache.flink.metrics.prometheus.PrometheusReporter
- metrics.reporter.prom.port: 9250-9260

### Kafka相关调优配置

- linger.ms/batch.size 这两个配置项配合使用，可以在吞吐量和延迟中得到最佳的平衡点。batch.size是kafka producer发送数据的批量大小，当数据量达到batch size的时候，会将这批数据发送出去，避免了数据一条一条的发送，频繁建立和断开网络连接。但是如果数据量比较小，导致迟迟不能达到batch.size，为了保证延迟不会过大，kafka不能无限等待数据量达到batch.size的时候才发送。为了解决这个问题，引入了`linger.ms`配置项。当数据在缓存中的时间超过`linger.ms`时，无论缓存中数据是否达到批量大小，都会被强制发送出去。

ack 数据源是否需要kafka得到确认。all表示需要收到所有ISR节点的确认信息，1表示只需要收到kafka leader的确认信息，0表示不需要任何确认信息。该配置项需要对数据精准性和延迟吞吐量做出权衡。

### Kafka topic分区数和Flink并行度的关系

- Flink kafka source的并行度需要和kafka topic的分区数一致。最大化利用kafka多分区topic的并行读取能力。

### Yarn相关调优配置

- yarn.scheduler.maximum-allocation-vcores
- yarn.scheduler.minimum-allocation-vcores

Flink单个task manager的slot数量必须介于这两个值之间

- yarn.scheduler.maximum-allocation-mb
- yarn.scheduler.minimum-allocation-mb

Flink的job manager 和task manager内存不得超过container最大分配内存大小。

yarn.nodemanager.resource.cpu-vcores yarn的虚拟CPU内核数，建议设置为物理CPU核心数的2-3倍，如果设置过少，会导致CPU资源无法被充分利用，跑任务的时候CPU占用率不高。

---

### **CASE**

概述：通过 flink 监控 API 获取程序状态脚本

```python
import __future__
import argparse
import requests
import json
import os
import time

# ！！！ 确认
yarnWebServer = ["hybrid03"]
yarnWebPort = 8088

parser = argparse.ArgumentParser(description='manual to this script')
parser.add_argument('--app_name', '-n', help='flink application name', required=True)

# 获取 yarn application 数据
def get_application_info():
    url = "http://{}:{}/ws/v1/cluster/apps".format(yarnWebServer[0], yarnWebPort)
    response = requests.get(url=url)
    data = response.text
    return data


# 获取 flink job 信息
def get_flink_job_info(application_id):
    url = "http://{}:{}/proxy/{}/jobs/overview".format(yarnWebServer[0], yarnWebPort, application_id)
    response = requests.get(url=url)
    data = response.text
    return data


# 开始检查 Flink Job 任务
def check_running(input_args):
    application_data = get_application_info()
    apps = json.loads(application_data)['apps']['app']

    # killed_app_max_time = max([i['finishedTime'] for i in apps
    #                            if i['name'] == input_args.app_name
    #                            and i['state'] == 'FINISHED'
    #                            and i['finalStatus'] == 'KILLED'])
    # finished_app_max_time = max([i['finishedTime'] for i in apps
    #                              if i['name'] == input_args.app_name])

    live_apps = [i for i in apps if i['name'] == input_args.app_name and i['trackingUI'] != 'History']
    if len(live_apps) == 0:
        print("\nCurrent time: " + time.asctime(time.localtime(time.time())))
        print("%s app killed !!!" % (input_args.app_name))
        return 0

    for app in live_apps:
        if app['state'] in ['FINISHED', 'FAILED', 'KILLED']:
            os.system('yarn application -kill ' + app['id'])
            live_apps.remove(app)
            print("Job " + app['id'] + " is killed.")
        else:
            app_id = app['id']
            state = app['state']
            final_status = app['finalStatus']
            pkg_name = app['name']

    if len(live_apps) == 0:
        return 0
    elif len(live_apps) > 1:
        print("\n!!!More than one " + input_args.app_name + " application are alive")
        return 1

    print("\nCurrent time: " + time.asctime(time.localtime(time.time())))
    print("===Current application info:===")
    print("\tapp_id:\t\t%s,\n\tstate:\t\t%s,\n\tfinal_status:\t%s,\n\tappName:\t%s" % (
        app_id, state, final_status, pkg_name))

    # 开始检查 Flink Job 任务
    job_data = get_flink_job_info(app_id)
    jobs = json.loads(job_data)['jobs']
    if len(jobs) > 1:
        print("\n!!!More than one Flink jobs exist !!!")
        return 1
    elif len(jobs) == 0:
        return 0

    if jobs[0]['state'] == 'FAILED':
        os.system('yarn application -kill ' + app_id)
        print("Job " + app_id + " is killed.")
        return 0
    else:
        print("===Current job info:===")
        print("\tjob_id:\t\t%s,\n\tjob_name:\t%s,\n\tstate:\t\t%s" % (
            jobs[0]['jid'], jobs[0]['name'], jobs[0]['state']))

    return 1


if __name__ == '__main__':
    args = parser.parse_args()
    exit(check_running(args))
```

### 生产场景
Flink的延迟指标有一个问题，比如你实际任务并没有延迟，但是显示的延迟指标是你最后一条变更数据的时间和当前时间的差值，作为的延迟指标。如果没有新的数据更新，延迟就会一直变大。
当网络出现问题，或者丢包很严重的时候，binlog可能订阅不到，此时延迟应该增大。但是现在flink的延迟是不增大的。
基于MySQL的心跳机制，做延迟指标更新。当Binlog没有任何变更的时候，通过mysql hearbeat来更新延迟（mysql heatbeat在没有任何变更的时候，默认是15s一次发一次心跳包），因此只要延迟在15秒以内波动都是正常的



## 九、JSON

### 1、JSONNODE：

1、JsonNode 表示 json 节点，整个节点模型的根接口为 TreeNode，json 节点主要用于手动构建 json 对象。

2、JsonNode 有各种数据类型的实现类，其中最常用的就是 ObjectNode 与 ArrayNode，前者表示 json 对象，后者表示 json 对象数组。

3、json 节点对象可以通过 JsonNodeFactory 创建，如 JsonNodeFactory.instance.objectNode();

| JsonNode json节点通用方法                                    |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| JsonNode get(String fieldName)                               | 用于访问对象节点的指定字段的值，如果此节点不是对象、或没有指定字段名的值，或没有这样名称的字段，则返回 null。 |
| JsonNode get(int index)JsonNode get(int index)               | 访问数组节点的指定索引位置上的值，对于其他节点，总是返回 null.如果索引小于0，或等于、大于节点大小，则返回 null，对于任何索引都不会引发异常。 |
| boolean isArray()                                            | 判断此节点是否为 {@link ArrayNode} 数组节点                  |
| boolean isObject()                                           | 如果此节点是对象节点，则返回 true，否则返回 false            |
| int size()                                                   | 获取 json 节点的大小，比如 json 对象中有多少键值对，或者 json 数组中有多少元素。 |
| ObjectNode deepCopy()                                        | json 节点对象深度复制，相当于克隆                            |
| Iterator<String> fieldNames()                                | 获取 json 对象中的所有 key                                   |
| Iterator<JsonNode> elements()                                | 如果该节点是JSON数组或对象节点，则访问此节点的所有值节点,对于对象节点，不包括字段名（键），只包括值，对于其他类型的节点，返回空迭代器。 |
| boolean has(int index)                                       | 检查此节点是否为数组节点，并是否含有指定的索引。             |
| boolean has(String fieldName)                                | 检查此节点是否为 JSON 对象节点并包含指定属性的值。           |
| *将 json 属性的值转为 java 数据类型*                         |                                                              |
| int asInt() int asInt(int defaultValue)                      | asInt()：尝试将此节点的值转换为 int 类型，布尔值 false 转换为 0,true 转换为 1。如果不能转换为 int（比如值是对象或数组等结构化类型），则返回默认值 0 ，不会引发异常。asInt(int defaultValue)：设置默认值 |
| boolean asBoolean() boolean asBoolean(boolean defaultValue)  | 尝试将此节点的值转换为 Java 布尔值，0以外的整数映射为true，0映射为false，字符串“true”和“false”映射到相应的值。如果无法转换为布尔值（包括对象和数组等结构化类型），则返回默认值 false，不会引发异常。可以自己设置默认值。 |
| asText(String defaultValue)String asText()                   | 如果节点是值节点（isValueNode 返回 true），则返回容器值的有效字符串表示形式，否则返回空字符串。 |
| long asLong()long asLong(long defaultValue)                  | 与 asInt 同理                                                |
| double asDouble() double asDouble(double defaultValue)       | 尝试将此节点的值转换为 double，布尔值转换为0.0（false）和1.0（true），字符串使用默认的Java 语言浮点数解析规则进行解析。如果表示不能转换为 double（包括对象和数组等结构化类型），则返回默认值 0.0,不会引发异常。 |
| BigInteger bigIntegerValue()                                 | 返回此节点的整数值（BigDecimal），当且仅当此节点为数字时（isNumber}返回true）。对于其他类型，返回 BigInteger.ZERO。 |
| boolean booleanValue()                                       | 用于访问 JSON 布尔值（值文本“true”和“false”）的方法,对于其他类型，始终返回false |
| BigDecimal decimalValue()                                    | 返回此节点的浮点值 BigDecimal, 当且仅当此节点为数字时（isNumber 返回true）,对于其他类型，返回 BigDecimal.ZERO |
| double doubleValue()                                         | 返回此节点的64位浮点（双精度）值，当且仅当此节点为数字时（isNumber返回true），对于其他类型，返回0.0。 |
| float floatValue()                                           | 返回此节点的32位浮点值，当且仅当此节点为数字时（isNumber返回true），对于其他类型，返回0.0。 |
| int intValue() long longValue() Number numberValue() short shortValue() String textValue() | 返回此节点的 int、long、Number、short、String 值。           |
| ObjectNode 对象节点常用方法                                  |                                                              |
| ObjectNode put(String fieldName, String v) ObjectNode put(String fieldName, int v) | 1、将字段的值设置为指定的值，如果字段已经存在，则更新值，value 可以为 null.2、其它 8 种基本数据类型以及 String、BigDecimal、BigInteger 都可以 put，但是没有 Date 类型3、Date 日期类型只能通过 Long 长整型设置 |
| ArrayNode putArray(String fieldName)                         | 构造新的 ArrayNode 子节点，并将其作为此 ObjectNode 的字段添加。 |
| ObjectNode putNull(String fieldName):                        | 为指定字段添加 null 值，put                                  |
| ObjectNode putObject(String fieldName)                       | 构造新的 ObjectNode 字节的，并将其作为此 ObjectNode 的字段添加。 |
| *替换与删除元素*                                             |                                                              |
| JsonNode replace(String fieldName, JsonNode value)           | 将特定属性的值替换为传递的值，字段存在时更新，不存在时新增   |
| JsonNode set(String fieldName, JsonNode value)               | 设置指定属性的值为 json 节点对象，字段存在时更新，不存在时新增，类似 replace 方法 |
| JsonNode setAll(Map<String,? extends JsonNode> properties)   | 同时设置多个 json 节点                                       |
| JsonNode setAll(ObjectNode other)                            | 添加给定对象（other）的所有属性，重写这些属性的任何现有值.   |
| ArrayNode withArray(String propertyName)                     | 将 json 节点转为 json 数组对象                               |
| ObjectNode with(String propertyName)                         | 将 json 节点转为 ObjectNode 对象                             |
| JsonNode remove(String fieldName)                            | 删除指定的 key，返回被删除的节点                             |
| ObjectNode remove(Collection<String> fieldNames)             | 同时删除多个字段                                             |
| ObjectNode removeAll()                                       | 删除所有字段属性                                             |
| JsonNode without(String fieldName):                          | 删除指定的 key，底层也是 remove                              |
| ObjectNode without(Collection<String> fieldNames)            | 同时删除多个字段，底层也是 removeAll                         |
| ArrayNode 数组节点常用方法                                   |                                                              |
| ArrayNode add(String v)                                      | 将指定的字符串值添加到此 json 数组的末尾，其它数据类型也是同理。除了可以添加 String 类型，还有 Java 的 8 种基本数据类型，以及 BigDecimal、BigInteger、JsonNode 类型。 |
| ArrayNode addAll(ArrayNode other)                            | 用于添加给定数组的所有子节点                                 |
| ArrayNode addNull()                                          | 该方法将在此数组节点的末尾添加 null 值。                     |
| ArrayNode addArray()                                         | 构造新的 ArrayNode 子节点，并将其添加到此数组节点的末尾      |
| ObjectNode addObject()                                       | 构造一个新的 ObjectNode 字节的，并将其添加到此数组节点的末尾 |

### 2、FASTJSON：

## 十一、数仓

#### 权限控制range/sentry

区别：

range：以资源为中心(为每个库/表 选择可以操作的用户)

sentry：以用户为中心(为每个用户 选择可操作的 库/表)

#### 数仓分层

ODS：**Operation Data Store** 原始数据层

DWD：**data warehouse datail** 明细数据层

DWS：**data warehouse service** 服务数据层

DWT：**data warehouse Topic** 数据主题层

ADS：**Application data store** 数据应用层

#### mysql/clickhouse/es/hbase

mysql主要用于oltp，
clickhouse主要用于olap，
es主要用于全文检索，
hbase要进行聚合需要使用phoenix





## 十二、flume

#### 1.source

taildir：拉取数据 ，同时会将拉取的最后位置offset 持久化一份在磁盘上，下次拉取会先读offset 再去拉取offset后的数据

【注】kafka宕机重启会导致少量数据重复

#### 2.用于批量处理flume向HDFS写文件时产生未释放租约的异常文件

```python
# encoding: utf-8
'''
@function:用于批量处理flume向HDFS写文件时产生未释放租约的异常文件
 
'''
import os
 
def get_unrealease_files(prefix = "/data/logs", tmp_suffix = ".tmp"):
    '''
    @function 获取过期未释放租约的文件列表
    @return: list []
    '''
    file_list = []
 
    cmd = "hdfs fsck / -openforwrite | grep /data"
    info = os.popen(cmd).readlines()
 
    for row in info:
        f = row.split(":")[0]
        if f.startswith(prefix) and not f.endswith(tmp_suffix):
            file_list.append(f)
 
    return file_list
 
 
def recover_lease(hdfs_path, retries = 3):
    '''
    @param hdfs_path : hdfs文件路径（全路径）
    @param retries :   重试次数
    '''
    cmd = "hdfs debug recoverLease -path %s -retries %s" %(hdfs_path, retries)
    try:
        ret = os.system(cmd)
        if ret == 0:
            print 'Recover lease of [%s] Successful!' %hdfs_path
        else:
            print 'Recover lease of [%s] unsuccessful,please check by manual!' %hdfs_path
    except Exception,e:
        print e
 
def batch_recover_lease(file_list):
    '''
    @param file_list: 待恢复文件列表
    '''
    if not file_list:
        print 'no file need to be recover lease, ended.'
        return
 
    for f in file_list:
        recover_lease(f)
 
if __name__ == '__main__':
    #获取指定条件未释放租约的文件列表,默认/data/logs目录下不是从.tmp结尾的文件
    file_list = get_unrealease_files()
    #批量释放租约，恢复文件为可读
    batch_recover_lease(file_list)
```

## 十三、MaxWell

#### maxwell提高并行度设置

​	Maxwell默认还是输出到指定Kafka主题的**一个kafka分区**，因为多个分区并行可能会打乱binlog的顺序

​	如果要提高并行度，首先设置kafka的分区数>1,然后设置**producer_partition_by**属性

​	可选值producer_partition_by=database|table|primary_key|random| column

**综合上面对比，Maxwell想做监控分析，选择row格式比较合适**

#### Maxwell同步历史数据的原理：bootstrap

bin/maxwell-bootstrap
--user maxwell  --password 000000 --host hadoop102  --database gmall2021 --table user_info --client_id
maxwell_1

###### ps:（maxwell-bootstrap不具备将数据直接导入kafka或者hbase的能力，通过--client_id指定将数据交给哪个maxwell进程处理，在maxwell的conf.properties中配置）



##  十四、数据倾斜的优化

###### 1、大表join小表时 map join

将小表加载到内存，复制多份，让每个maptask内存中存在一份，（比如存放在hashtable中），然后只扫描大表，对于大表中的每一条记录key/value，在hashtable中查找是否有对应的记录，如果有则连接后输出即可。

###### 2、大表join大表 SMB join（sparksql/hivesql需要配置参数）

Sort Merge Bucket Join 存在的目的主要是为了解决大表与大表间的 Join 问题，分桶其实就是把大表化成了“小表”，然后 Map-Side Join 解决之，这是典型的分而治之的思想。

连接两个在（包含连接列的）相同列上划分了桶的表，可以使用 Map 端连接 （Map-side join）高效的实现。比如JOIN操作。对于JOIN操作两个表有一个相同的列，如果对这两个表都进行了桶操作。那么将保存相同列值的桶进行JOIN操作就可以，可以大大较少JOIN的数据量.

```
set hive.input.format=org.apache.hadoop.hive.ql.io.BucketizedHiveInputFormat;
set hive.optimize.bucketmapjoin = true;
set hive.optimize.bucketmapjoin.sortedmerge = true;
```

（1）两表进行分桶，桶的个数必须相等

（2）两边进行join时，join列==排序列==分桶列

###### 3、小表不小不大，怎么用 map join 解决倾斜问题

使用 map join 解决小表(记录数少)关联大表的数据倾斜问题，这个方法使用的频率非常高，但如果小表很大，大到map join会出现bug或异常，这时就需要特别的处理。 以下例子:

```sql
select * from log a
  left outer join users b
  on a.user_id = b.user_id;
```

users 表有 600w+ 的记录，把 users 分发到所有的 map 上也是个不小的开销，而且 map join 不支持这么大的小表。如果用普通的 join，又会碰到数据倾斜的问题。
假如，log里user_id有上百万个，这就又回到原来map join问题。所幸，每日的会员uv不会太多，有交易的会员不会太多，有点击的会员不会太多，有佣金的会员不会太多等等。所以下面这个方法能解决很多场景下的数据倾斜问题。
**解决方法：**

```sql
select /*+mapjoin(x)*/* from log a
  left outer join (
    select  /*+mapjoin(c)*/d.*
      from ( select distinct user_id from log ) c
      join users d
      on c.user_id = d.user_id
    ) x
  on a.user_id = b.user_id;
```





## 十五、表的导入策略

#### 增量导入：

###### order_detail:不允许修改

​	id,

#### 全量同步：

###### 一共有13张表，如商品sku、spu、品类表、活动规则表等等

#### 新增及变化：[不允许删除]

#### 特殊表：



---



## 十六、生产环境变更：

###### SIT：



###### UAT：





## 十七、IDE 重要快捷键 

### idea

ctr（mac comm） + shift + f ： find in path

ctr（mac comm） + shift + r ： replace in path

ctr（mac comm） + F12 ： 查看类的方法

ctr + h ： 查看类的继承关系

### vscode

#### 文件和编辑器管理：

- 打开新文件：`Command + N` [cloud.tencent.com](https://cloud.tencent.com/developer/article/1483501)
- 打开文件：`Command + O` [cloud.tencent.com](https://cloud.tencent.com/developer/article/1483501)
- 保存文件：`Command + S` [cloud.tencent.com](https://cloud.tencent.com/developer/article/1483501)
- 关闭编辑器：`Command + W` [cloud.tencent.com](https://cloud.tencent.com/developer/article/2093798)
- 切换编辑器：`Control + Tab` [blog.csdn.net](https://blog.csdn.net/wqs1028/article/details/117696152)

#### 代码导航：

- 跳转到某行：`Control + G` [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/339258835)
- 跳转到声明位置：`F12` [blog.csdn.net](https://blog.csdn.net/JH_Cao/article/details/123724437)
- 显示所有符号：`Command + T` [cloud.tencent.com](https://cloud.tencent.com/developer/article/2093798)

#### 代码编辑：

- 代码格式化：`Shift + Option + F` [blog.csdn.net](https://blog.csdn.net/wqs1028/article/details/117696152)
- 注释当前行：`Command + /` [randyfield.cn](http://randyfield.cn/post/2021-06-20-mac-vscode/)
- 复制当前行：`Shift + Option + Up/Down Arrow` [cloud.tencent.com](https://cloud.tencent.com/developer/article/1483501)
- 删除行：`Shift + Command + K` [cloud.tencent.com](https://cloud.tencent.com/developer/article/1483501)

#### 查找和替换：

- 查找：`Command + F` [blog.csdn.net](https://blog.csdn.net/JH_Cao/article/details/123724437)
- 替换：`Command + Option + F` [blog.csdn.net](https://blog.csdn.net/JH_Cao/article/details/123724437)

#### 调试：

- 设置或取消断点：`F9` [cloud.tencent.com](https://cloud.tencent.com/developer/article/2093798)
- 开始或继续：`F5` [cloud.tencent.com](https://cloud.tencent.com/developer/article/2093798)

以上只是一部分VSCode的快捷键，你可以使用`Command + K, Command + S`打开快捷键编辑页面查看和设置更多的快捷键 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/60894789)。

## 十八、presto

拼接常量表：

```sql
select user_id (
values ('111'),
('222'),
('333')
  
-- ...
('999')
) from t (user_id)
```

## 十九、Git

```
- 日常最佳实践
git status
git pull origin qa_clustering
git stash push -m "保存本地更改"
git stash pop
```

```
Commit message格式
<type>: <subject>

注意冒号后面有空格。

type
用于说明 commit 的类别，只允许使用下面7个标识。

feat：新功能（feature）
fix：修补bug
docs：文档（documentation）
style： 格式（不影响代码运行的变动）
refactor：重构（即不是新增功能，也不是修改bug的代码变动）
test：增加测试
chore：构建过程或辅助工具的变动
如果type为feat和fix，则该 commit 将肯定出现在 Change log 之中。

subject
subject是 commit 目的的简短描述，不超过50个字符，且结尾不加句号（.）。

3、使用工具校验commit是否符合规范
3.1 全局安装
npm install -g @commitlint/cli @commitlint/config-conventional
1
3.2 生成配置配件
这个文件在根目录下生成就可以了。

echo "module.exports = {extends: ['@commitlint/config-conventional']}" > commitlint.config.js
1
3.2 在commitlint.config.js制定提交message规范
"module.exports = {extends: ['@commitlint/config-conventional']}"

module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      "feat", "fix", "docs", "style", "refactor", "test", "chore", "revert"
    ]],
    'subject-full-stop': [0, 'never'],
    'subject-case': [0, 'never']
  }
};
1
2
3
4
5
6
7
8
9
10
11
12
上面我们就完成了commitlint的安装与提交规范的制定。检验commit message的最佳方式是结合git hook，所以需要配合Husky

3.4 husky介绍
husky继承了Git下所有的钩子，在触发钩子的时候，husky可以阻止不合法的commit,push等等。注意使用husky之前，必须先将代码放到git 仓库中，否则本地没有.git文件，就没有地方去继承钩子了。

npm install husky --save-dev
1
安装成功后需要在项目下的package.json中配置

"scripts": {
    "commitmsg": "commitlint -e $GIT_PARAMS",

 },
 "config": {
    "commitizen": {
      "path": "cz-customizable"
    }
  },
1
2
3
4
5
6
7
8
9
最后我们可以正常的git操作

git add .
git commit -m ""
1
2
git commit的时候会触发commlint。下面演示下不符合规范提交示例：

F:\accesscontrol\access_control>git commit -m "featdf: aas"
husky > npm run -s commitmsg (node v8.2.1)

⧗   input:
featdf: aas

✖   type must be one of [feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert] [type-enum]
✖   found 1 problems, 0 warnings

husky > commit-msg hook failed (add --no-verify to bypass)

F:\accesscontrol\access_control>
1
2
3
4
5
6
7
8
9
10
11
12
上面message不符合提交规范，所以会提示错误。

我们修改下type

F:\accesscontrol\access_control>git commit -m "feat: 新功能"
husky > npm run -s commitmsg (node v8.2.1)

⧗   input: feat: 新功能
✔   found 0 problems, 0 warnings

[develop 7a20657] feat: 新功能
 1 file changed, 1 insertion(+)

F:\accesscontrol\access_control>
1
2
3
4
5
6
7
8
9
10
commit成功。

3.5 husky的钩子
可以在package.json下面添加如下的钩子。

"husky": {
    "hooks": {
      "pre-commit": "npm run lint"
    }
  },
1
2
3
4
5
6
4、最后总结过程中遇到一些问题
git commit后可能报错相关‘regenerator-runtime’模块找不到；解决方式：npm install regenerator-runtime –save。
git commit -m “messge”,用双引号
```

## 二十、nginx

```bash
sbin/nginx -s reload
```

## 二十一、yarn

```
1，yarn top 

    类似linux里的top命令，查看正在运行的程序资源使用情况

2， yarn queue -status root.users.xxxx 

查看指定queue使用情况

3,yarn application -list -appStates 【ALL,NEW,NEW_SAVING,SUBMITTED,ACCEPTED,RUNNING,FINISHED,FAILED,KILLED】

yarn application -list -appTypes [SUBMITTED, ACCEPTED, RUNNING]

查看app状态

yarn application -movetoqueue application_1528080031923_0067 -queue root.users.xxx

移动app到对应的队列

yarn application -kill application_1528080031923_0067

kill掉app

yarn application -status application_1528080031923_0067

查看app状态

4，yarn applicationattempt -list application_1528080031923_0064

查看app尝试信息

5，yarn classpath --glob

打印类路径

6，yarn container -list appattempt_1528080031923_0068_000001

打印正在执行任务的容器信息

yarn container -status container_1528080031923_0068_01_000002

打印当前容器信息

7，yarn jar [mainClass] args...

提交任务到yarn

8，yarn logs -applicationId application_1528080031923_0064 >> logs

查看app运行日志

9，yarn node -all -list

查看所有节点信息

10，yarn daemonlog -getlevel n0:8088 rg.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl

查看守护进程日志级别

11，yarn resourcemanager [-format-state-store]

RMStateStore的格式化. 如果过去的应用程序不再需要，则清理RMStateStore

12， Usage: yarn rmadmin

-refreshQueues 重载队列的ACL，状态和调度器特定的属性，ResourceManager将重载mapred-queues配置文件

-refreshNodes 动态刷新dfs.hosts和dfs.hosts.exclude配置，无需重启NameNode。

dfs.hosts：列出了允许连入NameNode的datanode清单（IP或者机器名）

dfs.hosts.exclude：列出了禁止连入NameNode的datanode清单（IP或者机器名）

重新读取hosts和exclude文件，更新允许连到Namenode的或那些需要退出或入编的Datanode的集合。

-refreshUserToGroupsMappings 刷新用户到组的映射。

-refreshSuperUserGroupsConfiguration 刷新用户组的配置

-refreshAdminAcls 刷新ResourceManager的ACL管理

-refreshServiceAclResourceManager 重载服务级别的授权文件。

-getGroups [username] 获取指定用户所属的组。

-transitionToActive [–forceactive] [–forcemanual] 尝试将目标服务转为 Active 状态。如果使用了–forceactive选项，不需要核对非Active节点。如果采用了自动故障转移，这个命令不能使用。虽然你可以重写–forcemanual选项，你需要谨慎。

-transitionToStandby [–forcemanual] 将服务转为 Standby 状态. 如果采用了自动故障转移，这个命令不能使用。虽然你可以重写–forcemanual选项，你需要谨慎。

-failover [–forceactive] 启动从serviceId1 到 serviceId2的故障转移。如果使用了-forceactive选项，即使服务没有准备，也会尝试故障转移到目标服务。如果采用了自动故障转移，这个命令不能使用。

-getServiceState 返回服务的状态。（注：ResourceManager不是HA的时候，时不能运行该命令的）

-checkHealth 请求服务器执行健康检查，如果检查失败，RMAdmin将用一个非零标示退出。（注：ResourceManager不是HA的时候，时不能运行该命令的）

-help [cmd]显示指定命令的帮助，如果没有指定，则显示命令的帮助。

```

## 二十二、web

```
从
 http://tomcat.apache.org/
 下载tomcat，把它解压到电脑本地。

 进入tomcat根目录，进入bin文件夹，双击startup.bat（linux/mac则是将startup.sh拖入命令行/终端）启动tomcat。

 打开浏览器，在浏览器输入localhost:8080。如能打开tomcat网页则启动成功。

 把要传输的文件放入tomcat目录的webapps\ROOT下，在浏览器输入http://localhost:8080/[文件名]，可以下载成功。

 
 

 应用场景：

 当无法通过正常途径将文件导入设备时，可以使用此方法配合wget命令将刷机包导入设备，再通过flash.sh命令刷机。

 比如：本次测试时某版本下载连接失效。而且设备无法通过SSH进入，不能通过pcsp将刷机包导入到设备内。

 则可以通过启动Tomcat，将刷机包firmware_1.0.6.bin放入webapps\ROOT下。

 通过Telnet连接设备，输入

 cd /tmp/

 wget http://192.168.31.165:8080/firmware_1.0.6.bin

 flash.sh firmware_1.0.6.bin

 
 

 http://192.168.31.165:8080/miwifi_p01_firmware_1.0.6.bin

 其中，192.168.31.164是我的电脑在内网的IP；8080是端口号，tomcat默认8080；firmware_1.0.6.bin是文件名。

 http://[本机IP]:[端口号]/[文件名]
```



## 二十三、kudu

```
介绍

        kudu集群部署完毕后，如何管理kudu集群？官方提供了kudu command tool工具来管理kudu集群，通过kudu command，可以对master、tserver、tablet、tables、wal、fs、replica进行管理。

cluster
    ksck：检查kudu集群的健康
kudu cluster ksck <master_addresses> [-checksum_cache_blocks] [-checksum_scan] [-checksum_scan_concurrency=<concurrency>] [-nochecksum_snapshot] [-color=<color>] [-tables=<tables>] [-tablets=<tablets>] 
master_addresses：逗号分隔的master地址
checksum_cache_blocks：是否检查扫描read blocks
checksum_scan：对集群中的数据进行校验和扫描
checksum_scan_concurrency：每个ts设置多少扫描校验执行器，默认4
color：输出是否有颜色
tables：一个逗号分隔的检查tables，空表示检查所有tables
tablets：逗号分隔的tablets id，空表示检查所有tablets
fs
    format：格式化一个新的kudu文件系统
kudu fs format [-fs_wal_dir=<dir>] [-fs_data_dirs=<dirs>] [-uuid=<uuid>]
fs_wal_dir：WAL的目录，没有指定则无法启动
fs_data_dirs：逗号分隔的data blocks，没有指定则无法启动
uuid：filesystem使用的uuid，如果没有指定，则会自动产生一个
    dump：dump出kudu文件系统
        cfile：dump出cfile文件内容
kudu fs dump cfile <block_id> [-fs_wal_dir=<dir>] [-fs_data_dirs=<dirs>] [-noprint_meta] [-noprint_rows] 
block_id：block的定义
fs_wal_dir：WAL的目录，没有指定则无法启动
fs_data_dirs：逗号分隔的data blocks，没有指定则无法启动
print_meta：输出中打印元数据信息
print_rows：打印cfile的每一行
        tree：dump出kudu文件系统的tree
kudu fs dump tree [-fs_wal_dir=<dir>] [-fs_data_dirs=<dirs>] 
fs_wal_dir：WAL的目录，没有指定则无法启动
fs_data_dirs：逗号分隔的data blocks，没有指定则无法启动
        uuid：dump出kudu文件系统的uuid
kudu fs dump uuid [-fs_wal_dir=<dir>] [-fs_data_dirs=<dirs>] 
fs_wal_dir：WAL的目录，没有指定则无法启动
fs_data_dirs：逗号分隔的data blocks，没有指定则无法启动
local_replica
    copy_from_remote：从远程服务器拷贝tablet到本地
kudu local_replica copy_from_remote <tablet_id> <source> [-fs_wal_dir=<dir>] [-fs_data_dirs=<dirs>]
tablet_id：tablet的定义
source：远程服务器地址，host：port
fs_wal_dir：WAL的目录，没有指定则无法启动
fs_data_dirs：逗号分隔的data blocks，没有指定则无法启动
    delete：删除本地文件系统的tablet副本
kudu local_replica delete <tablet_id> [-fs_wal_dir=<dir>] [-fs_data_dirs=<dirs>] [-clean_unsafe]
tablet_id：tablet的定义
fs_wal_dir：WAL的目录，没有指定则无法启动
fs_data_dirs：逗号分隔的data blocks，没有指定则无法启动
clean_unsafe：删除本地副本，但是保留tombstone record
    list：显示本地的tablet副本
kudu local_replica list [-fs_wal_dir=<dir>] [-fs_data_dirs=<dirs>] [-list_detail] 
fs_wal_dir：WAL的目录，没有指定则无法启动
fs_data_dirs：逗号分隔的data blocks，没有指定则无法启动
list_detail：打印副本的分区信息
    cmeta：操作本地文件系统tablet的一致性元数据文件
        print_replica_uuids：打印所有tablet副本的uuid
kudu local_replica cmeta print_replica_uuids <tablet_id> [-fs_wal_dir=<dir>] [-fs_data_dirs=<dirs>] 
tablet_id：tablet的定义
fs_wal_dir：WAL的目录，没有指定则无法启动
fs_data_dirs：逗号分隔的data blocks，没有指定则无法启动
        rewrite_raft_config：重写tablet副本的raft配置
kudu local_replica cmeta rewrite_raft_config <tablet_id> <peers>…​ [-fs_wal_dir=<dir>] [-fs_data_dirs=<dirs>] 
tablet_id：tablet的定义
peers：一个peers列表，格式为'uuid:hostname:port'
fs_wal_dir：WAL的目录，没有指定则无法启动
fs_data_dirs：逗号分隔的data blocks，没有指定则无法启动
    dump：dump出kudu的文件系统
        block_ids：dump出所有本地副本的blocks的ids
kudu local_replica dump block_ids <tablet_id> [-fs_wal_dir=<dir>] [-fs_data_dirs=<dirs>]
tablet_id：tablet的定义
fs_wal_dir：WAL的目录，没有指定则无法启动
fs_data_dirs：逗号分隔的data blocks，没有指定则无法启动
        meta：dump出本地副本的元数据
kudu local_replica dump meta <tablet_id> [-fs_wal_dir=<dir>] [-fs_data_dirs=<dirs>] 
tablet_id：tablet的定义
fs_wal_dir：WAL的目录，没有指定则无法启动
fs_data_dirs：逗号分隔的data blocks，没有指定则无法启动
        rowset：dump出本地副本的rowset内容
kudu local_replica dump rowset <tablet_id> [-dump_data] [-fs_wal_dir=<dir>] [-fs_data_dirs=<dirs>] [-metadata_only] [-nrows=<nrows>] [-rowset_index=<index>] 
tablet_id：tablet的定义
dump_data：dump每列的rowset
fs_wal_dir：WAL的目录，没有指定则无法启动
fs_data_dirs：逗号分隔的data blocks，没有指定则无法启动
metadata_only：只dump block的元数据信息
nrows：dump出多少行
rowset_index：本地副本的index
        wats：dump出本地副本的所有WAL
kudu local_replica dump wals <tablet_id> [-fs_wal_dir=<dir>] [-fs_data_dirs=<dirs>] [-print_entries=<entries>] [-noprint_meta] [-truncate_data=<data>] 
tablet_id：tablet的定义
fs_data_dirs：逗号分隔的data blocks，没有指定则无法启动
print_entries：
print_meta：打印元数据信息
truncate_data：在打印之前将数据字段截断到给定的字节数
master
    set_flag：改变kudu master的gflag值
kudu master set_flag <master_address> <flag> <value> [-force] 
master_addresses：逗号分隔的master地址
flag：gflag的名称
value：gflag的值
force：如果为true，则允许set_flag命令设置未显式标记为运行时可设置的标志。 这样的标志更改可能会在服务器上被忽略，或者可能导致服务器崩溃。
    status：获取kudu master的状态
kudu master status <master_address> 
master_addresses：逗号分隔的master地址
    timestamp：获取kudu master的当前timestamp
kudu master timestamp <master_address>
master_addresses：逗号分隔的master地址
pbc
    dump：dump出PBC文件
kudu pbc dump <path> [-oneline]
path：PBC文件保存路径
oneline：打印出每个protobuf的每一行
remote_replica
    check：检查tserver上的所有的tablet副本
kudu remote_replica check <tserver_address>
tserver_address：kudu tablet servcer地址
    copy：拷贝一个tablet副本到其他tablet server上
kudu remote_replica copy <tablet_id> <src_address> <dst_address> [-force_copy] 
tablet_id：tablet的定义
src_address：源地址
dst_address：目的地址
force_copy：如果远程目标有该副本，也强制拷贝
    delete：删除kudu tablet上面的副本
kudu remote_replica delete <tserver_address> <tablet_id> <reason> 
tserver_address：kudu tablet servcer地址
tablet_id：tablet的定义
reason：删除副本的原因
    dump：dump出kudu tablet server上tablet的副本
kudu remote_replica dump <tserver_address> <tablet_id> 
tserver_address：kudu tablet servcer地址
tablet_id：tablet的定义
    list：列出kudu tablet server所有tablet的副本
kudu remote_replica list <tserver_address> 
tserver_address：kudu tablet servcer地址
table
    delete：删除一个table
kudu table delete <master_addresses> <table_name>
master_addresses：逗号分隔的master地址
table_name：要删除的表名称
    list：列出所有tables
kudu table list <master_addresses> [-list_tablets] 
master_addresses：逗号分隔的master地址
tablet_id：tablet的定义
tablet
    leader_step_down：强制使tablets leader down
kudu tablet leader_step_down <master_addresses> <tablet_id>
master_addresses：逗号分隔的master地址
tablet_id：tablet的定义
    change_config：改变tablets的raft配置
            add_replica：给tablet副本增加一个新的raft配置
kudu tablet change_config add_replica <master_addresses> <tablet_id> <replica_uuid> <replica_type>
master_addresses：逗号分隔的master地址
tablet_id：tablet的定义
replica_uuid：新的replica的uuid
replica_type：replica类型，VOTER或者NON-VOTER
            change_replica_type：改变已经存在的tablet副本的类型
kudu tablet change_config change_replica_type <master_addresses> <tablet_id> <replica_uuid> <replica_type> 
master_addresses：逗号分隔的master地址
tablet_id：tablet的定义
replica_uuid：新的replica的uuid
replica_type：replica类型，VOTER或者NON-VOTER
            remove_replica：删除一个存在的tablet副本
kudu tablet change_config remove_replica <master_addresses> <tablet_id> <replica_uuid> 
master_addresses：逗号分隔的master地址
tablet_id：tablet的定义
replica_uuid：新的replica的uuid
tserver
    set_flag：改变kudu tablet server的gflag值
kudu tserver set_flag <tserver_address> <flag> <value> [-force]
tserver_addresses：tserver地址
flag：gflag的名称
value：gflag的值
force：如果为true，则允许set_flag命令设置未显式标记为运行时可设置的标志。 这样的标志更改可能会在服务器上被忽略，或者可能导致服务器崩溃。
    status：查看kudu tablet server的状态
kudu tserver status <tserver_address> 
tserver_addresses：tserver地址
    timestamp：获取kudu tablet server的当前timestamp
kudu tserver timestamp <tserver_address> 
tserver_addresses：tserver地址
wal
    dump：dump出WAL文件
kudu wal dump <path> [-print_entries=<entries>] [-noprint_meta] [-truncate_data=<data>] 
  	path：WAL文件路径
 print_entries：如何打印条目：false | 0 | no =不打印true | 1 |是| decode =打印它们解码pb =打印raw protobuf id =仅打印其ID
 print_meta：打印元数据信息
 truncate_data：在打印之前将数据字段截断到给定的字节数
test
    loadgen：运行负载生成测试
kudu test loadgen <master_addresses> [-buffer_flush_watermark_pct=<pct>] [-buffer_size_bytes=<bytes>] [-buffers_num=<num>] [-flush_per_n_rows=<rows>] [-keep_auto_table] [-num_rows_per_thread=<thread>] [-num_threads=<threads>] [-run_scan] [-seq_start=<start>] [-show_first_n_errors=<errors>] [-string_fixed=<fixed>] [-string_len=<len>] [-table_name=<name>] [-table_num_buckets=<buckets>] [-table_num_replicas=<replicas>] [-use_random]
```

## 二十四、java JVM

### 1、动态修改 jar

打包以及修改jar包

cd genesys_data_etl
mvn clean package -Poffline -Dmaven.test.skip=true
日志如下：
[INFO] --- maven-jar-plugin:2.6:jar (default-jar) @ genesys_data_etl ---
[INFO] Building jar: /Users/xx/IdeaProjects/genesys_data_etl/target/genesys_data_etl-0.0.1-SNAPSHOT.jar
生成jar包
此时可以通过命令
java -jar genesys_data_etl-0.0.1-SNAPSHOT.jar 
运行jar包。


但是要修改jar包中的配置文件怎么办呢？

方式一 通过vim命令直接修改保存jar。超方便。


1.通过vim命令直接编辑jar
vim xxx.jar 该命令首先会列出全部文件，可以通过输入/abc来搜索，定位到对应的abc文件后回车进入配置文件内进行编辑，:wq保存。



方式二 通过jar命令替换jar包中的文件(也可新增)


1.列出jar包中的文件清单
jar tf genesys_data_etl-0.0.1-SNAPSHOT.jar

2.提取出内部jar包的指定文件

```bash
jar -xf genesys_data_etl-0.0.1-SNAPSHOT.jar BOOT-INF/classes/realtime/t_ivr_data_bj.json
##比如：
jar -xf sso-1.0-SNAPSHOT-exec.jar BOOT-INF/classes/application.yml
```



3.然后可以修改文件
vim BOOT-INF/classes/realtime/t_ivr_data_bj.json

4.更新配置文件到内部jar包.(存在覆盖，不存在就新增)

```bash
jar -uf genesys_data_etl-0.0.1-SNAPSHOT.jar BOOT-INF/classes/realtime/t_ivr_data_bj.json
```

4.1更新内部jar包到jar文件
jar -uf genesys_data_etl-0.0.1-SNAPSHOT.jar 内部jar包.jar     


5.可以查看验证是否已经更改
vim genesys_data_etl-0.0.1-SNAPSHOT.jar




方式三 解压jar包，修改后重新打包jar


1.解压
unzip genesys_data_etl-0.0.1-SNAPSHOT.jar 
2.移除jar包,最好备份
rm genesys_data_etl-0.0.1-SNAPSHOT.jar
3.重新打包
 jar -cfM0 new-genesys_data_etl-0.0.1-SNAPSHOT.jar *
 或者
 jar -cvfm0 genesys_data_etl-0.0.1-SNAPSHOT.jar ./META-INF/MANIFEST.MF ./
4.运行
java -jar new-genesys_data_etl-0.0.1-SNAPSHOT.jar

jar命令参数:
-c 创建新的存档
-f 指定存档文件名
-M 不配置配置清单，这样还可以使用maven生成的配置清单也就是MANIFEST.MF
-0 不进行压缩,如果压缩会有问题
-m 指定清单文件
-t 列出归档目录
-x 从档案中提取指定的 (或所有) 文件 
-u 更新现有的归档文件 
-v 在标准输出中生成详细输出 

```
Linux下如何在不解压jar包查看或修改配置文件
https://jingyan.baidu.com/article/91f5db1b1b66a41c7e05e36c.html
更新jar包里的配置文件
https://www.cnblogs.com/dayou123123/p/6845432.html
修改jar包中的配置文件
https://blog.csdn.net/young_kim1/article/details/50482398
```

### 2、常用 jvm 命令

```bash
jstack  pid > a1.txt
```

```bash
## jmap -dump:format=b,live,file=路径   进程pid
jmap -dump:format=b,live,file=22374.hprof 22374
```

### 3、java 启动 jar 包

```bash
java -XX:+PrintCompilation -Xdebug -Xrunjdwp:server=y,transport=dt_socket,address=10000,suspend=n -Xms2G -Xmx2G -Xmn1G -XX:MetaspaceSize=128M -XX:MaxMetaspaceSize=512M -XX:ReservedCodeCacheSize=128M -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:MaxTenuringThreshold=6 -XX:-OmitStackTraceInFastThrow -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/home/admin/logs/gc/ -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:/home/admin/logs/gc/gc.log -Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dspring.config.location=/etc/shulidata/conf.d/admin.properties -Dcom.sun.management.jmxremote.port=8729 -jar /home/admin/app-run/xxl-admin.jar
java -XX:+PrintCompilation -Xdebug -Xrunjdwp:server=y,transport=dt_socket,address=10001,suspend=n -Xms2G -Xmx2G -Xmn1G -XX:MetaspaceSize=128M -XX:MaxMetaspaceSize=512M -XX:ReservedCodeCacheSize=128M -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:MaxTenuringThreshold=6 -XX:-OmitStackTraceInFastThrow -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/home/admin/logs/gc/ -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:/home/admin/logs/gc/gc-exec.log -Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dspring.config.location=/etc/shulidata/conf.d/executor.properties -Dcom.sun.management.jmxremote.port=8730 -jar /home/admin/app-run/executor.jar
```



## 二十五、maven

```bash
##安装外部 jar //You need use below command to add external jar into .m2 folder
mvn install:install-file -Dfile=[JAR] -DgroupId=[some.group] -DartifactId=[Some Id] -Dversion=1.0.0 -Dpackaging=jar
```

```bash
mvn clean package -DskipTests
```

### Shade 配置文件

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.1.1</version>
            <configuration>
                <!-- put your configurations here -->
                <!--只包含该项目代码中用到的jar,在父项目中引入了，但在当前模块中没有用到就会被删掉-->
                <minimizeJar>true</minimizeJar>
                <!--重新定位类位置，就好像类是自己写的一样，修改别人jar包的package-->
                <relocations>
                    <relocation>
                        <pattern>com.alibaba.fastjson</pattern>
                        <shadedPattern>com.gavinzh.learn.fastjson</shadedPattern>
                        <excludes>
                            <!--这些类和包不会被改变-->
                            <exclude>com.alibaba.fastjson.not.Exists</exclude>
                            <exclude>com.alibaba.fastjson.not.exists.*</exclude>
                        </excludes>
                    </relocation>
                </relocations>
            </configuration>
            <executions>
                <execution>
                    <configuration>
                        <!--创建一个你自己的标识符，位置在原有名称之后-->
                        <shadedArtifactAttached>true</shadedArtifactAttached>
                        <shadedClassifierName>gavinzh</shadedClassifierName>
                        <!--在打包过程中对文件做一些处理工作-->
                        <transformers>
                            <!--在META-INF/MANIFEST.MF文件中添加key: value 可以设置Main方法-->
                            <transformer
                                    implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                <manifestEntries>
                                    <mainClass>com.gavinzh.learn.shade.Main</mainClass>
                                    <Build-Number>123</Build-Number>
                                    <Built-By>your name</Built-By>
                                    <X-Compile-Source-JDK>1.7</X-Compile-Source-JDK>
                                    <X-Compile-Target-JDK>1.7</X-Compile-Target-JDK>
                                </manifestEntries>
                            </transformer>
                            <!--阻止META-INF/LICENSE和META-INF/LICENSE.txt-->
                            <transformer implementation="org.apache.maven.plugins.shade.resource.ApacheLicenseResourceTransformer"/>
                            <!--合并所有notice文件-->
                            <transformer implementation="org.apache.maven.plugins.shade.resource.ApacheNoticeResourceTransformer">
                                <addHeader>true</addHeader>
                            </transformer>
                            <!--如果多个jar包在META-INF文件夹下含有相同的文件，那么需要将他们合并到一个文件里-->
                            <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                <resource>META-INF/spring.handlers</resource>
                            </transformer>
                            <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                <resource>META-INF/spring.schemas</resource>
                            </transformer>
                            <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                <resource>META-INF/spring.factories</resource>
                            </transformer>
                            <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                <resource>META-INF/spring.tld</resource>
                            </transformer>
                            <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                <resource>META-INF/spring-form.tld</resource>
                            </transformer>
                            <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                <resource>META-INF/spring.tooling</resource>
                            </transformer>
                            <!--如果多个jar包在META-INF文件夹下含有相同的xml文件，则需要聚合他们-->
                            <transformer implementation="org.apache.maven.plugins.shade.resource.ComponentsXmlResourceTransformer"/>
                            <!--排除掉指定资源文件-->
                            <transformer implementation="org.apache.maven.plugins.shade.resource.DontIncludeResourceTransformer">
                                <resource>.no_need</resource>
                            </transformer>
                            <!--将项目下的文件file额外加到resource中-->
                            <transformer implementation="org.apache.maven.plugins.shade.resource.IncludeResourceTransformer">
                                <resource>META-INF/pom_test</resource>
                                <file>pom.xml</file>
                            </transformer>
                            <!--整合spi服务中META-INF/services/文件夹的相关配置-->
                            <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                        </transformers>
                    </configuration>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

##### eg: pom.xml

```xml
<build>
    <plugins>
        <!-- scala编译插件 -->
        <plugin>
            <groupId>net.alchim31.maven</groupId>
            <artifactId>scala-maven-plugin</artifactId>
            <version>3.2.2</version>
            <configuration>
                <recompileMode>incremental</recompileMode>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>testCompile</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        <!-- java编译插件 -->
        <plugin>
            <artifactId>maven-compiler-plugin</artifactId>
            <!-- 插件的版本 -->
            <version>3.6.1</version>
            <!-- 编译级别 强制为jdk1.8-->
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
                <!-- 编码格式 -->
                <encoding>UTF-8</encoding>
            </configuration>
        </plugin>
        <!-- maven 打包插件 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>2.4.3</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <transformers>
                            <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                <!-- 指定main的位置 -->
                                <mainClass>*</mainClass>
                            </transformer>
                        </transformers>
                        <filters>
                            <filter>
                                <artifact>*:*</artifact>
                                <!-- 过滤不需要的jar包 -->
                                <excludes>
                                    <exclude>META-INF/*.SF</exclude>
                                    <exclude>META-INF/*.DSA</exclude>
                                    <exclude>META-INF/*.RSA</exclude>
                                </excludes>
                            </filter>
                        </filters>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```



## 二十六、HDFS 命令

```bash
##常用命令
##统计文件夹数据量大小
hadoop dfs -du -h /user/hive/warehouse/ | sed 's/ //' | sort -hr

##

```



```
##概览
1.在本地Linux文件系统的“/home/hadoop/”目录下创建一个文件txt，里面可以随意输入一些单词.

2.在本地查看文件位置（ls）

3.在本地显示文件内容

cd /usr/local/hadoop
    touch test1.txt
    cat test1.txt
    
4.使用命令把本地文件系统中的“txt”上传到HDFS中的当前用户目录的input目录下。

./sbin/start-dfs.sh
    ./bin/hdfs dfs -mkdir -p /user/hadoop
    ./bin/hdfs dfs -mkdir input
    ./bin/hdfs dfs -put ./test1.txt input
    
5.查看hdfs中的文件(-ls)

./bin/hdfs dfs -ls /input

6.显示hdfs中该的文件内容

./bin/hdfs dfs -cat input/test1.txt

7.删除本地的txt文件并查看目录

./bin/hdfs dfs -rm -ls input/test1.txt

8.从hdfs中将txt下载地本地原来的位置。

./bin/hdfs dfs -get input/test.txt ~/test1.txt

9.从hdfs中删除txt并查看目录
./bin/hdfs dfs -rm -ls input/test1.txt
向HDFS中上传任意文本文件，如果指定的文件在HDFS中已经存在，由用户指定是追加到原有文件末尾还是覆盖原有的文件；

if $(hdfs dfs -test -e text.txt);
then $(hdfs dfs -appendToFile local.txt text.txt);
else $(hdfs dfs -copyFromLocal -f local.txt text.txt);
fi
从HDFS中下载指定文件，如果本地文件与要下载的文件名称相同，则自动对下载的文件重命名；

if $(hdfs dfs -test -e file:///home/hadoop/text.txt);
then $(hdfs dfs -copyToLocal text.txt ./text2.txt); 
else $(hdfs dfs -copyToLocal text.txt ./text.txt); 
fi
将HDFS中指定文件的内容输出到终端中；

hdfs dfs -cat text.txt
显示HDFS中指定的文件的读写权限、大小、创建时间、路径等信息；

hdfs dfs -ls -h text.txt
给定HDFS中某一个目录，输出该目录下的所有文件的读写权限、大小、创建时间、路径等信息，如果该文件是目录，则递归输出该目录下所有文件相关信息；

hdfs dfs -ls -R -h /user/hadoop
提供一个HDFS内的文件的路径，对该文件进行创建和删除操作。如果文件所在目录不存在，则自动创建目录；

if $(hdfs dfs -test -d dir1/dir2);
then $(hdfs dfs -touchz dir1/dir2/filename); 
else $(hdfs dfs -mkdir -p dir1/dir2 && hdfs dfs -touchz dir1/dir2/filename); 
fi
删除文件：hdfs dfs -rm dir1/dir2/filename
提供一个HDFS的目录的路径，对该目录进行创建和删除操作。创建目录时，如果目录文件所在目录不存在则自动创建相应目录；删除目录时，由用户指定当该目录不为空时是否还删除该目录；

创建目录：hdfs dfs -mkdir -p dir1/dir2
删除目录（如果目录非空则会提示not empty，不执行删除）：hdfs dfs -rmdir dir1/dir2
强制删除目录：hdfs dfs -rm -R dir1/dir2
向HDFS中指定的文件追加内容，由用户指定内容追加到原有文件的开头或结尾；

追加到文件末尾：hdfs dfs -appendToFile local.txt text.txt
追加到文件开头：
hdfs dfs -get text.txt
cat text.txt >> local.txt
hdfs dfs -copyFromLocal -f text.txt text.txt
删除HDFS中指定的文件；

hdfs dfs -rm text.txt
删除HDFS中指定的目录，由用户指定目录中如果存在文件时是否删除目录；

删除目录（如果目录非空则会提示not empty，不执行删除）：hdfs dfs -rmdir dir1/dir2
强制删除目录：hdfs dfs -rm -R dir1/dir2
在HDFS中，将文件从源路径移动到目的路径。

hdfs dfs -mv text.txt text2.txt
复制代码
 
从HDFS中下载指定文件，如果本地文件与要下载的文件名称相同，则自动对下载的文件重命名；

if $(hdfs dfs -test -e file:///home/hadoop/text.txt);
then $(hdfs dfs -copyToLocal text.txt ./text2.txt); 
else $(hdfs dfs -copyToLocal text.txt ./text.txt); 
fi
 

将HDFS中指定文件的内容输出到终端中；

hdfs dfs -cat text.txt
显示HDFS中指定的文件的读写权限、大小、创建时间、路径等信息；

hdfs dfs -ls -h text.txt
给定HDFS中某一个目录，输出该目录下的所有文件的读写权限、大小、创建时间、路径等信息，如果该文件是目录，则递归输出该目录下所有文件相关信息；

hdfs dfs -ls -R -h /user/hadoop
提供一个HDFS内的文件的路径，对该文件进行创建和删除操作。如果文件所在目录不存在，则自动创建目录；

if $(hdfs dfs -test -d dir1/dir2);
then $(hdfs dfs -touchz dir1/dir2/filename); 
else $(hdfs dfs -mkdir -p dir1/dir2 && hdfs dfs -touchz dir1/dir2/filename); 
fi
删除文件：hdfs dfs -rm dir1/dir2/filename
提供一个HDFS的目录的路径，对该目录进行创建和删除操作。创建目录时，如果目录文件所在目录不存在则自动创建相应目录；删除目录时，由用户指定当该目录不为空时是否还删除该目录；

创建目录：hdfs dfs -mkdir -p dir1/dir2
删除目录（如果目录非空则会提示not empty，不执行删除）：hdfs dfs -rmdir dir1/dir2
强制删除目录：hdfs dfs -rm -R dir1/dir2
向HDFS中指定的文件追加内容，由用户指定内容追加到原有文件的开头或结尾；
追加到文件末尾：hdfs dfs -appendToFile local.txt text.txt
追加到文件开头：
（由于没有直接的命令可以操作，方法之一是先移动到本地进行操作，再进行上传覆盖）：
hdfs dfs -get text.txt
cat text.txt >> local.txt
hdfs dfs -copyFromLocal -f text.txt text.txt
删除HDFS中指定的文件；

hdfs dfs -rm text.txt

删除HDFS中指定的目录，由用户指定目录中如果存在文件时是否删除目录；

删除目录（如果目录非空则会提示not empty，不执行删除）：hdfs dfs -rmdir dir1/dir2

强制删除目录：hdfs dfs -rm -R dir1/dir2

在HDFS中，将文件从源路径移动到目的路径。

hdfs dfs -mv text.txt text2.txt
```

## 二十八、正则表达式

```
// 满足 yyyy-MM-dd 格式
"^\d{4}\-\d{2}\-\d{2}$"
```

## 三十、kubernetes

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    k8s.aliyun.com/pod-ips: 192.168.22.241
    prometheus.io/path: /metrics
    prometheus.io/port: '10254'
    prometheus.io/scrape: 'true'
  creationTimestamp: '2024-04-01T02:54:10Z'
  generateName: sparkoperator-779b5c8bc8-
  labels:
    app.kubernetes.io/instance: sparkoperator
    app.kubernetes.io/name: sparkoperator
    pod-template-hash: 779b5c8bc8
    product: emr
  managedFields:
    - apiVersion: v1
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:annotations':
            .: {}
            'f:prometheus.io/path': {}
            'f:prometheus.io/port': {}
            'f:prometheus.io/scrape': {}
          'f:generateName': {}
          'f:labels':
            .: {}
            'f:app.kubernetes.io/instance': {}
            'f:app.kubernetes.io/name': {}
            'f:pod-template-hash': {}
            'f:product': {}
          'f:ownerReferences':
            .: {}
            'k:{"uid":"fbcd4e0d-ddca-44d7-b721-24e5c8da7f6c"}': {}
        'f:spec':
          'f:containers':
            'k:{"name":"sparkoperator"}':
              .: {}
              'f:args': {}
              'f:env':
                .: {}
                'k:{"name":"APPLICATION_TTL"}':
                  .: {}
                  'f:name': {}
                  'f:value': {}
                'k:{"name":"AUTH_SIGNIN"}':
                  .: {}
                  'f:name': {}
                  'f:value': {}
                'k:{"name":"AUTH_URL"}':
                  .: {}
                  'f:name': {}
                  'f:value': {}
                'k:{"name":"SPARK_CONF_DIR"}':
                  .: {}
                  'f:name': {}
                  'f:value': {}
              'f:image': {}
              'f:imagePullPolicy': {}
              'f:name': {}
              'f:ports':
                .: {}
                'k:{"containerPort":10254,"protocol":"TCP"}':
                  .: {}
                  'f:containerPort': {}
                  'f:name': {}
                  'f:protocol': {}
              'f:resources': {}
              'f:securityContext': {}
              'f:terminationMessagePath': {}
              'f:terminationMessagePolicy': {}
              'f:volumeMounts':
                .: {}
                'k:{"mountPath":"/etc/spark/conf"}':
                  .: {}
                  'f:mountPath': {}
                  'f:name': {}
                'k:{"mountPath":"/etc/webhook-certs"}':
                  .: {}
                  'f:mountPath': {}
                  'f:name': {}
          'f:dnsPolicy': {}
          'f:enableServiceLinks': {}
          'f:imagePullSecrets':
            .: {}
            'k:{"name":"emr-image-regsecret"}': {}
          'f:nodeSelector': {}
          'f:restartPolicy': {}
          'f:schedulerName': {}
          'f:securityContext': {}
          'f:serviceAccount': {}
          'f:serviceAccountName': {}
          'f:terminationGracePeriodSeconds': {}
          'f:tolerations': {}
          'f:volumes':
            .: {}
            'k:{"name":"spark-defaults-configmap-volume"}':
              .: {}
              'f:configMap':
                .: {}
                'f:defaultMode': {}
                'f:name': {}
              'f:name': {}
            'k:{"name":"webhook-certs"}':
              .: {}
              'f:name': {}
              'f:secret':
                .: {}
                'f:defaultMode': {}
                'f:secretName': {}
      manager: kube-controller-manager
      operation: Update
      time: '2024-04-01T02:54:10Z'
    - apiVersion: v1
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:annotations':
            'f:k8s.aliyun.com/pod-ips': {}
      manager: terwayd
      operation: Update
      time: '2024-04-01T02:54:10Z'
    - apiVersion: v1
      fieldsType: FieldsV1
      fieldsV1:
        'f:status':
          'f:conditions':
            'k:{"type":"ContainersReady"}':
              .: {}
              'f:lastProbeTime': {}
              'f:lastTransitionTime': {}
              'f:status': {}
              'f:type': {}
            'k:{"type":"Initialized"}':
              .: {}
              'f:lastProbeTime': {}
              'f:lastTransitionTime': {}
              'f:status': {}
              'f:type': {}
            'k:{"type":"Ready"}':
              .: {}
              'f:lastProbeTime': {}
              'f:lastTransitionTime': {}
              'f:status': {}
              'f:type': {}
          'f:containerStatuses': {}
          'f:hostIP': {}
          'f:phase': {}
          'f:podIP': {}
          'f:podIPs':
            .: {}
            'k:{"ip":"192.168.22.241"}':
              .: {}
              'f:ip': {}
          'f:startTime': {}
      manager: kubelet
      operation: Update
      subresource: status
      time: '2024-04-01T02:54:11Z'
  name: sparkoperator-779b5c8bc8-gmscv
  namespace: c-8b5bbb7fe2428456
  ownerReferences:
    - apiVersion: apps/v1
      blockOwnerDeletion: true
      controller: true
      kind: ReplicaSet
      name: sparkoperator-779b5c8bc8
      uid: fbcd4e0d-ddca-44d7-b721-24e5c8da7f6c
  resourceVersion: '42303757'
  uid: cfcab305-9be9-4913-a27f-db7d4a4ae780
spec:
  containers:
    - args:
        - '-v=2'
        - '-logtostderr'
        - '-namespace=c-8b5bbb7fe2428456'
        - '-enable-ui-service=true'
        - >-
          -ingress-url-format={{$appName}}.c-8b5bbb7fe2428456.ca13fca2ac9544da9ac1fff4a9f2b2617.cn-hangzhou.alicontainer.com
        - '-controller-threads=10'
        - '-resync-interval=30'
        - '-enable-batch-scheduler=false'
        - '-label-selector-filter='
        - '-enable-metrics=true'
        - '-metrics-labels=app_type'
        - '-metrics-port=10254'
        - '-metrics-endpoint=/metrics'
        - '-metrics-prefix='
        - '-enable-webhook=true'
        - '-webhook-svc-namespace=c-8b5bbb7fe2428456'
        - '-webhook-port=8080'
        - '-webhook-svc-name=sparkoperator-webhook'
        - '-webhook-config-name=sparkoperator-webhook-config-c-8b5bbb7fe2428456'
        - '-webhook-namespace-selector='
        - '-enable-resource-quota-enforcement=false'
        - '-aliyun-owner=1554710753828130'
        - '-aliyun-region=cn-hangzhou'
        - '-use-spark-app-name-as-ingress-name=true'
      env:
        - name: APPLICATION_TTL
          value: '259200'
        - name: AUTH_URL
          value: >-
            http://oauth2proxy.c-8b5bbb7fe2428456.svc.cluster.local:4180/oauth2/auth
        - name: AUTH_SIGNIN
          value: >-
            http://c-8b5bbb7fe2428456.ca13fca2ac9544da9ac1fff4a9f2b2617.cn-hangzhou.alicontainer.com/oauth2/start
        - name: SPARK_CONF_DIR
          value: /etc/spark/conf
      image: >-
        registry-vpc.cn-hangzhou.aliyuncs.com/emr/spark-operator:v1beta2-1.3.3-3.1.1-deca11f
      imagePullPolicy: Always
      name: sparkoperator
      ports:
        - containerPort: 10254
          name: metrics
          protocol: TCP
      resources: {}
      securityContext: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
        - mountPath: /etc/webhook-certs
          name: webhook-certs
        - mountPath: /etc/spark/conf
          name: spark-defaults-configmap-volume
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-z2z2z
          readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  imagePullSecrets:
    - name: emr-image-regsecret
  nodeName: cn-hangzhou.192.168.30.51
  nodeSelector:
    emr-spark: emr-spark
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: spark-operator
  serviceAccountName: spark-operator
  terminationGracePeriodSeconds: 30
  tolerations:
    - effect: NoSchedule
      key: emr-node-taint
      operator: Equal
      value: emr
    - effect: NoExecute
      key: node.kubernetes.io/not-ready
      operator: Exists
      tolerationSeconds: 300
    - effect: NoExecute
      key: node.kubernetes.io/unreachable
      operator: Exists
      tolerationSeconds: 300
  volumes:
    - name: webhook-certs
      secret:
        defaultMode: 420
        secretName: sparkoperator-webhook-certs
    - configMap:
        defaultMode: 420
        name: emr-spark-defaults-conf
      name: spark-defaults-configmap-volume
    - name: kube-api-access-z2z2z
      projected:
        defaultMode: 420
        sources:
          - serviceAccountToken:
              expirationSeconds: 3607
              path: token
          - configMap:
              items:
                - key: ca.crt
                  path: ca.crt
              name: kube-root-ca.crt
          - downwardAPI:
              items:
                - fieldRef:
                    apiVersion: v1
                    fieldPath: metadata.namespace
                  path: namespace
status:
  conditions:
    - lastTransitionTime: '2024-04-01T02:54:10Z'
      status: 'True'
      type: Initialized
    - lastTransitionTime: '2024-04-01T02:54:11Z'
      status: 'True'
      type: Ready
    - lastTransitionTime: '2024-04-01T02:54:11Z'
      status: 'True'
      type: ContainersReady
    - lastTransitionTime: '2024-04-01T02:54:10Z'
      status: 'True'
      type: PodScheduled
  containerStatuses:
    - containerID: >-
        containerd://fac207a17a07ef05a2f08b10f69100dcf1c6078b8fccf405cc2e6d1b52b20ad1
      image: >-
        registry-vpc.cn-hangzhou.aliyuncs.com/emr/spark-operator:v1beta2-1.3.3-3.1.1-deca11f
      imageID: >-
        registry-vpc.cn-hangzhou.aliyuncs.com/emr/spark-operator@sha256:02d26ebad1dfc7eda822d325c892fdf934a76dfe698be4dca51040bbaebf93dd
      lastState: {}
      name: sparkoperator
      ready: true
      restartCount: 0
      started: true
      state:
        running:
          startedAt: '2024-04-01T02:54:11Z'
  hostIP: 192.168.30.51
  phase: Running
  podIP: 192.168.22.241
  podIPs:
    - ip: 192.168.22.241
  qosClass: BestEffort
  startTime: '2024-04-01T02:54:10Z'

```

## 三十一、mysql

```sql
##加索引
ALTER TABLE `ad_platform_two_jump_statistics_dt` 
DROP KEY `idx_dt`,ADD KEY `idx_dt_plan_id`(`dt`,`plan_id`) USING BTREE;
```
```
UPDATE crowd_es_task_record set `status` = 'INIT' WHERE `status` = 'RUNNING';
```
```
# 常用函数
1. HOUR(DATE_SUB(NOW(), INTERVAL 1 HOUR))

NOW(): 返回当前日期和时间。
DATE_SUB(NOW(), INTERVAL 1 HOUR): 将当前时间减去 1 小时。
HOUR(...): 提取上述结果的小时部分。
作用: 获取当前时间前一小时的小时数。例如，如果当前时间是 16:30，该函数将返回 15。

2. hour(date_sub(current_timestamp,interval 1 hour))

current_timestamp: 等同于 NOW(), 返回当前日期和时间。
date_sub(current_timestamp, interval 1 hour): 将当前时间减去 1 小时。
hour(...): 提取上述结果的小时部分。
作用: 与第一个函数相同，也是获取当前时间前一小时的小时数。

3. date_format(date_add('day',-1,current_date),'%Y-%m-%d')

current_date: 返回当前日期。
date_add('day', -1, current_date): 将当前日期减去 1 天。
date_format(..., '%Y-%m-%d'): 将日期格式化为 "YYYY-MM-DD" 的字符串格式。
作用: 获取昨天日期，并格式化为 "YYYY-MM-DD" 的字符串。

4. date_format(date_sub(current_date,interval 1 day),'%Y-%m-%d')

current_date: 返回当前日期。
date_sub(current_date, interval 1 day): 将当前日期减去 1 天。
date_format(..., '%Y-%m-%d'): 将日期格式化为 "YYYY-MM-DD" 的字符串格式。
作用: 与第三个函数相同，也是获取昨天日期，并格式化为 "YYYY-MM-DD" 的字符串。

```

### binlog 命令
```
# 一、配置信息
# 是否启用binlog日志
show variables like 'log_bin';

# 查看详细的日志配置信息
show global variables like '%log%';

# mysql数据存储目录
show variables like '%dir%';

# 查看binlog的目录
show global variables like "%log_bin%";

# 查看当前服务器使用的binlog文件及大小
show binary logs;

# 查看主服务器使用的binlog文件及大小


# 二、管理binlog
# 查看所有binlog的日志列表
show master logs;

# 查看最后一个binlog日志的编号名称及最后一个时间结束的位置pos
show master status;

# 刷新binlog，会生成一个新编号的binlog日志文件
flush log;

# 清空所有binlog日志（慎用）
reset master;

# 设置binlog文件保存事件，过期删除，单位天
set global expire_log_days=3; 

# 删除指定日期前的日志索引中binlog日志文件
purge master logs before '2019-03-09 14:00:00';

# 删除指定日志文件
purge master logs to 'master.000003';

# 删除slave的中继日志
reset slave;

# 三、事件查询命令
# 查看 binlog 内容
show binlog events;

# 查看具体一个binlog文件的内容 （in 后面为binlog的文件名）
show binlog events in 'master.000001';

# IN 'log_name' ：指定要查询的binlog文件名(不指定就是第一个binlog文件)
# FROM pos ：指定从哪个pos起始点开始查起(不指定就是从整个文件首个pos点开始算)
# LIMIT [offset,] ：偏移量(不指定就是0)
# row_count ：查询总条数(不指定就是所有行)
show binlog events [IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count];
```
