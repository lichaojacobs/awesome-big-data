---
title: Flink实战总结
date: 2018-12-20 20:19:06
tags:
---

## 基础
### 相关概念
- flink程序能实现在分布式的结合上进行各种转换操作，集合通常来自订阅的来源（文件，kafka,local,in-memory），结果被返回到sinks里（大多数写入分布式文件系统，或者标准输出）
- DataSet and DataStream
	- DataSet和DataStream在flink中都代表一种数据结构，是不可变且包含重复记录的集合。区别在于DataSet是有限的集合，而DataStream是无界的
- flink 配置interlij ideal 在本地运行调试
	- 只需要将flink依赖的包引入项目中即可启动项目
![HBase 架构图](http://imgs.wanhb.cn/flink1.png)
- **讲解Flink怎么序列化objects，怎么分配内存**[Apache Flink: Juggling with Bits and Bytes](https://flink.apache.org/news/2015/05/11/Juggling-with-Bits-and-Bytes.html)

###  DataStream
-  [Apache Flink 1.7 Documentation: Flink DataStream API Programming Guide](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/datastream_api.html)
- datasource（数据源）: 
	- File-based: readTextFile, readFile…
	- Socket-based: socketTextStream
	- Collection-based: fromCollection, fromElements
	- custom: addSource, FlinkKafkaConsumer08 or other connectors

### DataSet
- [Apache Flink 1.7 Documentation: Flink DataSet API Programming Guide](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/batch/)

### savepoint
- [Apache Flink 1.7 Documentation: Savepoints](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/state/savepoints.html)
- Savepoints are created, owned, and deleted by the user.
- 目前savepoint和checkpoint实现和format方式都相同（除了checkpoint选择了rocksdb作为state backend，这样format会有些微不同）
- Operations：
	- Triggering Savepoints： FsStateBackend or RocksDBStateBackend:
	- Trigger a Savepoint
	- Cancel Job with Savepoint
		- `bin/flink cancel -s [:targetDirectory] :jobId`
	- Resuming from Savepoints 
		-  `$ bin/flink run -s :savepointPath [:runArgs]`
	- Disposing Savepoints
		- `$ bin/flink savepoint -d :savepointPath`

### checkpoint
- [Apache Flink 1.7 Documentation: Checkpoints](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/state/checkpoints.html)
- 生命周期是由Flink管理，checkpoint的管理，创建以及释放统一通过Flink，而不需要用户干预
- **Checkpoints are usually dropped（随应用退出被清除）** after the job was terminated by the user (except if explicitly configured as retained Checkpoints)
- checkpoint 优化 [Apache Flink 1.7 Documentation: Tuning Checkpoints and Large State](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/state/large_state_tuning.html)
	- state 双写：一份在distributed storage(HDFS)；一份在local
	- task-local recovery：默认是关闭的状态,可以通过`state.backend.local-recovery` 打开

### Barriers
- [Apache Flink 1.8-SNAPSHOT Documentation: Data Streaming Fault Tolerance](https://ci.apache.org/projects/flink/flink-docs-master/internals/stream_checkpointing.html)

###  Window，waterMark，Trigger
- [Window，waterMark，Trigger介绍- 简书](https://www.jianshu.com/p/a883262241ef)
- window
	- 滚动窗口：分配器将每个元素分配到一个指定窗口大小的窗口中，并且不会重叠；TumblingEventTimeWindows.of(Time.seconds(5))
	- 滑动窗口：滑动窗口分配器将元素分配到固定长度的窗口中，与滚动窗口类似，窗口的大小由窗口大小参数来配置，另一个窗口滑动参数控制滑动窗口开始的频率；因此可能出现窗口重叠，如果滑动参数小于滚动参数的话；SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5))
	- 会话窗口：通过session活动来对元素进行分组，跟滚动窗口和滑动窗口相比，不会有重叠和固定的开始时间和结束时间的情况。当他在一个固定的时间周期内不再收到元素，即非活动间隔产生，那么窗口就会关闭；
		- 一个session窗口通过一个session间隔来配置，这个session间隔定义了非活跃周期的长度。当这个非活跃周期产生，那么当前的session将关闭并且后续的元素将被分配到新的session窗口中去。如：EventTimeSessionWindows.withGap(Time.minutes(10)
- 触发器(Triggers)
	- 触发器定义了一个窗口何时被窗口函数处理
	- EventTimeTrigger
	- ProcessingTimeTrigger
	- CountTrigger
	- PurgingTrigger
- 驱逐器(Evictors)

## 任务提交与停止姿势
- 任务提交
	- **启动命令详解** :[Apache Flink 1.7 Documentation: YARN Setup](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/deployment/yarn_setup.html)
	- 参数
```
Usage:
   Required
     -n,--container <arg>   Number of YARN container to allocate (=Number of Task Managers)
   Optional
     -D <arg>                        Dynamic properties
     -d,--detached                   Start detached
     -jm,--jobManagerMemory <arg>    Memory for JobManager Container with optional unit (default: MB)
     -nm,--name                      Set a custom name for the application on YARN
     -q,--query                      Display available YARN resources (memory, cores)
     -qu,--queue <arg>               Specify YARN queue.
     -s,--slots <arg>                Number of slots per TaskManager
     -tm,--taskManagerMemory <arg>   Memory per TaskManager Container with optional unit (default: MB)
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths for HA mode
```

	- 提交到yarn-cluster上需要以 ::y:: 或者::yarn::作为前缀；如: `ynm=nm`
```
flink run -c com.jacobs.jobs.realtime.wordcount.WindowWordCount target/real-time-jobs-1.0.0-SNAPSHOT.jar

flink run -m yarn-cluster -ynm TestSinkUserLogStream -yn 4 -yjm 1024m -ytm 4096m -ys 4 -yqu feed.prod -c com.weibo.api.feed.dm.stream.TestFlinkStream /data1/dm-flink/feed-dm-flink-1.0.4-SNAPSHOT.jar
```

- **停止任务**
	- **关闭或重启flink程序不能直接kill掉**，这样会导致flink来不及制作checkpoint，而应该调用flink提供的cancel语意

```
//重启正确姿势, with savepoint
1. 调用cancel，cancel之前先触发savepoint
bin/flink cancel -s [:targetDirectory] :jobId -yid: yarnAppId
例子: flink cancel -s hdfs://vcp-yz-nameservice1/user/hcp/hcpsys/feed/flink-checkpoints/test-user-logs 97b4e67859af4bfb1b597355f1c846f3 -yid application_1542801635735_2121
2. 从savepoint中恢复flink程序
bin/flink run -s :savepointPath [:runArgs]
例子: flink run -s hdfs://vcp-yz-nameservice1/user/hcp/hcpsys/feed/flink-checkpoints/test-user-logs/savepoint-97b4e6-22dd5890dd0c -m yarn-cluster -ynm TestSinkUserLogStream -yn 4 -yjm 1024m -ytm 4096m -ys 4 -yqu feed.prod -c com.weibo.api.feed.dm.stream.TestFlinkStream /data1/dm-flink/feed-dm-flink-1.0.4-SNAPSHOT.jar
```

## 运行模式
### Standalone
- standalone 启动cluster
```
/usr/local/flink-1.6.0/bin;./start-cluster.sh
```

### On Yarn Cluster
- [Apache Flink 1.7 Documentation: YARN](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/deployment/yarn_setup.html)
- **参考文章**[Flink1.6 - flink on yarn分布式部署架构 - 深山含笑](https://zhouhai02.com/post/flink-internals/flink1.6-flip6-flink-on-yarn-arch/)
- 架构图
![HBase 架构图](http://imgs.wanhb.cn/flink2.png)
	- JobManager 和 ApplicationMaster  运行在同一个JVM里

- **on yarn 两种模式**
	- session模式：允许运行多个作业在同一个Flink集群中，代价是作业之间没有资源隔离（同一个TM中可能跑多个不同作业的task）
	- per-job模式（生产环境）：per-job模式是指在yarn上为每一个Flink作业都分配一个单独的Flink集群，这样就解决了不同作业之间的资源隔离问题
- **摘录参考文章**相比旧的Flink-on-YARN架构（Flink 1.5之前），新的yarn架构带来了以下的优势：
	 - client可以直接在yarn上面启动一个作业，不在像以前需要先启动一个固定大小的Flink集群然后把作业提交到这个Flink集群上
	 - 按需申请容器（指被同一个作业的不同算子所使用的容器可以有不同的CPU/Memory配置），没有被使用的容器将会被释放
![HBase 架构图](http://imgs.wanhb.cn/flink3.png)
- slot资源申请/分配流程分析
- 请求新TaskExecutor的slot分配
![HBase 架构图](http://imgs.wanhb.cn/flink4.png)
- ResourceManager挂掉 ：不会挂掉task,不断尝试重新注册ResourceManager**详细见参考文章**
- TaskExecutor挂掉
- JobMaster挂掉

## 资源分配相关？
- 在operator中对并行度的设置将决定任务分配到几个task slot里面去

## Flink程序运行流程分解
### 基本步骤
- 1. Obtain an execution environment

```
getExecutionEnvironment()
createLocalEnvironment()
createRemoteEnvironment(host: String, port: Int, jarFiles: String*)
```
- 2. Load/create the initial data

```
val text: DataStream[String] = env.readTextFile("file:///path/to/file")
```
- 3. Specify transformations on this data

```
//create a new DataStream by converting every String in the original collection to an integer
val mapped = input.map { x => x.toInt }
```
- 4. Specify where to put the results of your computations
	
```
writeAsText(path: String)
print()
```
- 5. Trigger the program execution

## Flink watermark机制
- **【重要】详细讲解watermark**:  [Flink流计算编程—watermark（水位线）](https://blog.csdn.net/lmalds/article/details/52704170) 
	- window 触发的两个条件

	```
	1、watermark时间 >= window_end_time
	2、在[window_start_time,window_end_time)中有数据存在
	```
- [摘录：深入理解Flink核心技术 ](https://zhuanlan.zhihu.com/p/20585530)
	- 纵坐标为event_time，横坐标为processingTime，理想情况自然是两者一致，但实际情况肯定不可能
![HBase 架构图](http://imgs.wanhb.cn/flink5.png)
- [摘录：使用EventTime与WaterMark进行流数据处理](http://shiyanjun.cn/archives/1785.html)

```
 // 这块结合上图理解watermark的值
@Override
    public final Watermark getCurrentWatermark() {
        long potentialWM = currentMaxTimestamp - maxOutOfOrderness; // 当前最大事件时间戳，减去允许最大延迟到达时间
        if (potentialWM >= lastEmittedWatermark) { // 检查上一次emit的WaterMark时间戳，如果比lastEmittedWatermark大则更新其值
            lastEmittedWatermark = potentialWM;
        }
        return new Watermark(lastEmittedWatermark);
    }
```

- Windowing, WaterMark,Trigger 三者依赖关系
	- Windowing：就是负责该如何生成Window，比如Fixed Window、Slide Window，当你配置好生成Window的策略时，Window就会根据时间动态生成，最终得到一个一个的Window，包含一个时间范围：[起始时间, 结束时间)，它们是一个一个受限于该时间范围的事件记录的容器，每个Window会收集一堆记录，满足指定条件会触发Window内事件记录集合的计算处理
	- WaterMark：它其实不太好理解，可以将它定义为一个函数E=f(P)，当前处理系统的处理时间P，根据一定的策略f会映射到一个事件时间E，可见E在坐标系中的表现形式是一条曲线，根据f的不同曲线形状也不同。假设，处理时间12:00:00，我希望映射到事件时间11:59:30，这时对于延迟30秒以内（事件时范围11:59:30~12:00:00）的事件记录到达处理系统，都指派到时间范围包含处理时间12:00:00这个Window中。事件时间超过12:00:00的就会由Trigger去做补偿了。
	- Trigger：为了满足实际不同的业务需求，对上述事件记录指派给Window未能达到实际效果，而做出的一种补偿，比如事件记录在WaterMark时间戳之后到达事件处理系统，因为已经在对应的Window时间范围之后，我有很多选择：选择丢弃，选择是满足延迟3秒后还是指派给该Window，选择只接受对应的Window时间范围之后的5个事件记录，等等，这都是满足业务需要而制定的触发Window重新计算的策略，所以非常灵活。


## Sink Connectors
- Kafka 
- Elasticsearch
- RabbitMQ
- Rolling File Sink (HDFS)
	-  [Apache Flink 1.7 Documentation: HDFS Connector](https://ci.apache.org/projects/flink/flink-docs-stable/dev/connectors/filesystem_sink.html)
-  Streaming File Sink 
	-  partfile 有三种状态：in-progress, pending,finished；part file先被写成in-progress，一旦被关闭写入，会变成pending，当检查点成功之后，pending状态的文件将变成finished; 
	-   [Apache Flink 1.7 Documentation: Streaming File Sink](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/connectors/streamfile_sink.html)
	- Using Row-encoded Output Formats 
		- 可以指定RollingPolicy 来滚动生成分区中的文件
	- Using Bulk-encoded Output Formats
		- **支持parquet，orc等文件格式**，批量编码文件
		- 通过BulkWriter.Factory定义不同的文件格式   [ParquetAvroWriters (flink 1.7-SNAPSHOT API)](https://ci.apache.org/projects/flink/flink-docs-release-1.7/api/java/org/apache/flink/formats/parquet/avro/ParquetAvroWriters.html)
		- **源码：** [flink/StreamingFileSink.java at master · apache/flink · GitHub](https://github.com/apache/flink/blob/master/flink-streaming-java/src/main/java/org/apache/flink/streaming/api/functions/sink/filesystem/StreamingFileSink.java)
		- 使用这种方式只能配合 `OnCheckpointRollingPolicy`  使用来滚动生成分区文件，通过设置 `env.enableCheckpointing(interval)`来设置文件滚动间隔
		- **Streaming to parquet in hdfs 出现问题，内存溢出导致job无限崩溃重启，大量part file**
		- 如果失败，将从上一个检查点开始重新store，期间回滚in-progress中的文件，以确保不会重复保存上一个检查点之后的数据
		- **part文件过多问题** [Streaming to parquet files not happy with flink 1.6.1 - Stack Overflow](https://stackoverflow.com/questions/52638193/streaming-to-parquet-files-not-happy-with-flink-1-6-1)
		- **rolling parquet file 重点邮件** [Apache Flink User Mailing List archive. - Streaming to Parquet Files in HDFS](http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/Streaming-to-Parquet-Files-in-HDFS-td23492.html)
		- 注意压缩的时候内存溢出的情况，flink陷入无限的重启循环中
![HBase 架构图](http://imgs.wanhb.cn/flink6.png)

##  StreamingFileSink与Kafka 结合
### 如何做到exactly once？
- [ An Overview of End-to-End Exactly-Once Processing in Apache Flink (with Apache Kafka, too!)](https://flink.apache.org/features/2018/03/01/end-to-end-exactly-once-apache-flink.html)
	- 二阶段提交
- partfile 有三种状态：in-progress, pending,finished；part file先被写成in-progress，一旦被关闭写入，会变成pending，当检查点成功之后，pending状态的文件将变成finished;
- 如果失败，将从上一个检查点开始重新store，期间回滚in-progress中的文件，以确保不会重复保存上一个检查点之后的数据
### flink如何控制kafka offset提交与checkpoint&&savepoint相结合
- [FlinkKafkaConsumer使用详解](http://m.sohu.com/a/168546400_617676)
- 关闭checkpoint(**Checkpointingdisabled**): 
	- 此时， Flink Kafka Consumer依赖于它使用的具体的Kafka client的自动定期提交offset的行为，相应的设置是 Kafka properties中的 enable.auto.commit (或者 auto.commit.enable 对于Kafka 0.8) 以及 auto.commit.interval.ms
- 开启checkpoint(**Checkpointingenabled**):
	- 在这种情况下，Flink Kafka Consumer会将offset存到checkpoint中
	- **制作完checkpoint 一并提交offsets** 当checkpoint 处于completed的状态时（**整个job的所有的operator都收到了这个checkpoint的barrier**）。将offset记录起来并提交，从而保证exactly-once
* ::exactly once的两个风险点：可结合savepoint来做::
	* 1. 异常退出的情况，没法来得及做checkpoint，而checkpoint间隔太长会导致丢失大量数据；可以通过airflow周期性手动触发savepoint恢复；封装hflink脚本
		* 解决思路是结合savepoint来做，通过**airflow定时的触发savepoint**操作，防止因checkpoint未及时做数据丢失
		* 规定一分钟savepoint一次，这样即使分钟级别的数据丢失也是可以容忍
	* 2. 第一点利用savepoint来做也有风险：在做savepoint的时候，如果异常退出，parfile未及时关闭导致数据丢失
		* **暂时可以认为问题较小？**

###  **如何控制背压**
- 如何做到挂很久之后重新启动时限制拉取的消息量？（类似spark.streaming.kafka.maxRatePerPartition）
	- 背压通过task slot 的stackTrace判断
	- 可以在kafka source那层控制一次性消费量，类似于spark

## Flink 高性能部署
- [Apache Flink 1.8-SNAPSHOT Documentation: JobManager High Availability (HA)](https://ci.apache.org/projects/flink/flink-docs-master/ops/jobmanager_high_availability.html)

## metric监控rest api
- [Apache Flink 1.8-SNAPSHOT Documentation: Monitoring REST API](https://ci.apache.org/projects/flink/flink-docs-master/monitoring/rest_api.html)

## Restart Strategies 
- **doc** [Apache Flink 1.7 Documentation: Restart Strategies](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/restart_strategies.html)
- Fixed Delay Restart Strategy
- Failure Rate Restart Strategy
- No Restart Strategy
- Fallback Restart Strategy