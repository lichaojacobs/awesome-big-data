---
title: Spark实战总结
date: 2018-10-15 00:12:35
tags:
    - 学习
    - 大数据
    - Spark
---

## Spark 基础

### Spark 的构成

- ClusterManager: 在standalone模式中即为，Master主节点，控制整个集群，监控worker。在yarn模式中为资源管理器
- worker :从节点，负责控制计算节点，启动Executro和Driver。在yarn模式中NodeManager,负责计算节点的控制。
- Driver：运行Application的main()函数并且创建SparkContext。
- Executor: 执行器，是为某Application运行在worker node上的一个进程，启动线程池运行任务上，每个Application拥有一组独立的executors
- SparkContext: 整个应用程序的上下文，控制整个应用的生命周期
- RDD：Spark的基本计算单元，一组RDD形成执行的有向无环图RDD Graph(DAG)
- DAG Scheduler: 根据Job构建基于stage的DAG，并且提交stage给TaskScheduler
- TaskScheduler: 可以将提交给它的stage 拆分为更多的task并分发给Executor执行
- SparkEnv: 线程级别的上下文，存储运行时的重要组件的引用
- DStream: 是一个RDD的序列，由若干RDD组成。在一个batchInterval中，会产生一个RDD，产生的数据统一塞入到这个RDD中，采用内存+磁盘的模式，尽可能放到内存中，当数据量太大时会spill到磁盘中。

### Spark 概念释义

- Transformation返回值还是一个RDD。它使用了链式调用的设计模式，对一个RDD进行计算后，变换成另外一个RDD，然后这个RDD又可以进行另外一次转换。这个过程是分布式的。 Action返回值不是一个RDD。它要么是一个Scala的普通集合，要么是一个值，要么是空，最终或返回到Driver程序，或把RDD写入到文件系统中。
- Action是返回值返回给driver或者存储到文件，是RDD到result的变换，Transformation是RDD到RDD的变换。只有action执行时，rdd才会被计算生成，这是rdd懒惰执行的根本所在。
- Driver是我们提交Spark程序的节点，并且所有的reduce类型的操作都会汇总到Driver节点进行整合。节点之间会将map/reduce等操作函数传递一个独立副本到每一个节点，这些变量也会复制到每台机器上，而节点之间的运算是相互独立的，变量的更新并不会传递回Driver程序。
- Spark中分布式执行的条件
	- 只要生成了task，就都是在executor中执行的，在driver中执行不会单独生成task
	- 生成task的操作有: spark.read 读取文件，之后对文件做各种map, filter, reduce操作，都是针对partition而言的


## spark 工作机制

- 一个Job被拆分成若干个Stage，每个Stage执行一些计算，产生一些中间结果。它们的目的是最终生成这个Job的计算结果。而每个Stage是一个task set，包含若干个task。Task是Spark中最小的工作单元，在一个executor上完成一个特定的事情。
- 除非用户指定持久化操作，否则转换过程中产生的中间数据在计算完毕后会被丢弃，即数据是非持久化的。
- 窄依赖:父RDD中的一个分区最多只会被子RDD中的一个分区使用，父RDD中，一个分区内的数据是不能被分割的，必须整个交付给子RDD中的一个分区。
- 宽依赖（Shuffle依赖）：父RDD中的分区可能会被多个子RDD分区使用。因为父RDD中一个分区内的数据会被分割，发送给子RDD的所有分区。因此Shuffle依赖也意味着父RDD与子RDD之间存在着Shuffle过程。

### Spark作业

- Application: 用户自定义的Spark程序，用户提交之后，Spark为App分配资源程序转换并执行。
- Driver Program: 运行Application的main函数并且创建SparkContext
- RDD DAG： 当RDD遇到Action算子，将之前的所有算子形成一个有向无环图（DAG）。再在Spark中转化为Job,提交到集群进行执行，一个App中可以包含多个Job
- Job： RDD Graph触发的作业，由spark Action算子触发，在SparkContext中通过runJob方法向spark提交Job
- stage： 每个Job会根据RDD的宽依赖关系被切分很多stage ,每个stage包含一组相同的task，这一组task也叫taskset
- Task: 一个分区对应一个Task,Task 执行RDD中对应stage中所包含的算子，Taksk 被封装好后放入Executor的线程池中执行。


## spark调度原理

### 作业调度

系统的设计很重要的一环便是资源调度。设计者将资源进行不同粒度的抽象建模，然后将资源统一放入调度器，通过一定的算法进行调度。

- spark的多种运行模式：Local模式，standalone模式、YARN模式，Mesos模式。

### Standalone VS Yarn

- 角色对比

	```
	standalone:  		yarn: 
	client      		client 
	Master 	  			ApplicationMaster 
	Worker 	  			ExecutorRunnable 
	Scheduler   		YarnClusterScheduler 
	SchedulerBackend 	YarnClusterSchedulerBackend
	
	```
- 在yarn中application Master 与Application Driver 运行于同一个JVM进程中
- standalone架构图

	![standalone](http://ol7zjjc80.bkt.clouddn.com/standalone.png)
	
- on yarn架构图

	![on yarn](http://ol7zjjc80.bkt.clouddn.com/on%20yarn.png)
	

### application调度

用户提交到spark中的作业集合，通过一定的算法对每个按一定次序分配集群中资源的过程。

- FIFO模式，用户先提交的作业1优先分配需要的资源，之后提交的作业再分配资源，依次类推。
- Mesos: 粗粒度模式和细粒度模式
- YARN模式：独占模式，可以控制应用分配资源
  - yarn-cluster: 适用于生产环境。client将用户程序提交到到spark集群中就与spark集群断开联系了，此时client将不会发挥其他任何作用，仅仅负责提交。在此模式下。AM和driver是同一个东西，但官网上给的是driver运行在AM里，可以理解为AM包括了driver的功能就像Driver运行在AM里一样，此时的AM既能够向AM申请资源并进行分配，又能完成driver划分RDD提交task等工作
  
  - yarn-client: y适用于交互、调试，希望立即看到app的输出。Driver运行在客户端上，先有driver再用AM，此时driver负责RDD生成、task生成和分发，向AM申请资源等 ,AM负责向RM申请资源，其他的都由driver来完成

### Job调度

Job调度就是在application内部的一组Job集合，在application分配到的资源量，通过一定的算法，对每个按一定次序分配Application中资源的过程。
- FIFO模式：先进先出模式
- FAIR模式：spark在多个job之间以轮询的方式给任务进行资源分配，所有的任务拥有大致相当的优先级来共享集群的资源。这就意味着当一个长任务正在执行时，短任务仍可以分配到资源，提交并执行，并且获得不错的响应时间。

### tasks延迟调度

- 数据本地性：尽量的避免数据在网络上的传输，传输任务为主，将任务传输到数据所在的节点

- 延时调度机制：拥有数据的节点当前正被其他的task占用，如果预测当前节点结束当前任务的时间要比移动数据的时间还要少，那么调度会等待，直到当前节点可用。否则移动数据到资源充足节点，分配任务执行。


## spark transformation和action的算子

### transformation
- [ ] map(func) 返回一个新的分布式数据集，由每个原元素经过func函数处理后的新元素组成 
- [ ] filter(func) 返回一个新的数据集，由经过func函数处理后返回值为true的原元素组成 
- [ ] flatMap(func) 类似于map，但是每一个输入元素，会被映射为0个或多个输出元素，(因此，func函数的返回值是一个seq，而不是单一元素) 
- [ ] mapPartitions(func) 类似于map，对RDD的每个分区起作用，在类型为T的RDD上运行时，func的函数类型必须是Iterator[T]=>Iterator[U]
- [ ] mapPartitionsWithIndex(func) 和mapPartitions类似，但func带有一个整数参数表上分区的索引值，在类型为T的RDD上运行时，func的函数参数类型必须是(int,Iterator[T])=>Iterator[U] 
- [ ] sample(withReplacement,fraction,seed) 根据给定的随机种子seed，随机抽样出数量为fraction的数据 
- [ ] pipe(command,[envVars]) 通过管道的方式对RDD的每个分区使用shell命令进行操作，返回对应的结果 
- [ ] union(otherDataSet) 返回一个新的数据集，由原数据集合参数联合而成 
- [ ] intersection(otherDataset) 求两个RDD的交集 
- [ ] distinct([numtasks]) 返回一个包含源数据集中所有不重复元素的i新数据集 
- [ ] groupByKey([numtasks]) 在一个由(K,v)对组成的数据集上调用，返回一个(K,Seq[V])对组成的数据集。默认情况下，输出结果的并行度依赖于父RDD的分区数目，如果想要对key进行聚合的话，使用reduceByKey或者combineByKey会有更好的性能 
- [ ] reduceByKey(func,[numTasks]) 在一个(K,V)对的数据集上使用，返回一个(K,V)对的数据集，key相同的值，都被使用指定的reduce函数聚合到一起，reduce任务的个数是可以通过第二个可选参数来配置的 
- [ ] sortByKey([ascending],[numTasks]) 在类型为(K,V)的数据集上调用，返回以K为键进行排序的(K,V)对数据集，升序或者降序有boolean型的ascending参数决定 
- [ ] join(otherDataset,[numTasks]) 在类型为(K,V)和(K,W)类型的数据集上调用，返回一个(K,(V,W))对，每个key中的所有元素都在一起的数据集 
- [ ] cogroup(otherDataset,[numTasks]) 在类型为(K,V)和(K,W)类型的数据集上调用，返回一个数据集，组成元素为(K,Iterable[V],Iterable[W]) tuples 
- [ ] cartesian(otherDataset) 笛卡尔积，但在数据集T和U上调用时，返回一个(T,U)对的数据集，所有元素交互进行笛卡尔积 
- [ ] coalesce(numPartitions) 对RDD中的分区减少指定的数目，通常在过滤完一个大的数据集之后进行此操作 
- [ ] repartition(numpartitions) 将RDD中所有records平均划分到numparitions个partition中

### action算子操作
- [ ] reduce(func) 通过函数func聚集数据集中的所有元素，这个函数必须是关联性的，确保可以被正确的并发执行 
- [ ] collect() 在driver的程序中，以数组的形式，返回数据集的所有元素，这通常会在使用filter或者其它操作后，返回一个足够小的数据子集再使用 
- [ ] count() 返回数据集的元素个数 
- [ ] first() 返回数据集的第一个元素(类似于take(1)) 
- [ ] take(n)  返回一个数组，由数据集的前n个元素组成。注意此操作目前并非并行执行的，而是driver程序所在机器 
- [ ] takeSample(withReplacement,num,seed) 返回一个数组，在数据集中随机采样num个元素组成，可以选择是否用随机数替换不足的部分，seed用于指定的随机数生成器种子 
- [ ] saveAsTextFile(path) 将数据集的元素，以textfile的形式保存到本地文件系统hdfs或者任何其他hadoop支持的文件系统，spark将会调用每个元素的toString方法，并将它转换为文件中的一行文本 
- [ ] takeOrderd(n,[ordering]) 排序后的limit(n) 
- [ ] saveAsSequenceFile(path) 将数据集的元素，以sequencefile的格式保存到指定的目录下，本地系统，hdfs或者任何其他hadoop支持的文件系统，RDD的元素必须由key-value对组成。并都实现了hadoop的writable接口或隐式可以转换为writable 
- [ ] saveAsObjectFile(path) 使用java的序列化方法保存到本地文件，可以被sparkContext.objectFile()加载 
- [ ] countByKey()  对(K,V)类型的RDD有效，返回一个(K,Int)对的map，表示每一个可以对应的元素个数 
- [ ] foreache(func) 在数据集的每一个元素上，运行函数func,t通常用于更新一个累加器变量，或者和外部存储系统做交互

## Spark常用存储格式parquet 详解

- [新型列式存储格式 Parquet 详解 - 后端 - 掘金](https://juejin.im/entry/589932fab123db16a3ace2d1)
- 三个组成部分
	- 存储格式(storage format)
	- 对象模型转换器(object model converters)
	- 对象模型(object models) ：简单理解为数据在内存中的表示
	![parquet](http://ol7zjjc80.bkt.clouddn.com/parquet.png)
	
- 列式存储
	- 把某一列数据连续存储，每一行数据离散存储技术
	- 带来的优化
		- 查询的时候不需要扫描全部的数据，而只需要读取每次查询涉及的列，这样可以将I/O消耗降低N倍，另外可以保存每一列的统计信息(min、max、sum等)，实现部分的谓词下推
		- 由于每一列的成员都是同构的，可以针对不同的数据类型使用更高效的数据压缩算法，进一步减小I/O
		- 由于每一列的成员的同构性，可以使用更加适合CPU pipeline的编码方式，减小CPU的缓存失效
	
- 数据模型

	```
	message AddressBook {
 		required string owner;
 		repeated string ownerPhoneNumbers;
 		repeated group contacts {
   			required string name;
   			optional string phoneNumber;
 		}
	}

	
	```
	- 根被叫做message，有多个field
	- 每个field包含三个属性:repetition, type, name
		- repetition可以是required（出现1次）, optional（出现0次或1次），repeated（出现0次或者多次）。type可以是一个group或者一个primitive类型
	- parquet数据类型不需要复杂的Map, List, Set等，而是使用repeated fields 和 groups来表示。例如List和Set可以被表示成一个repeated field,Map可以表示成一个包含有Key-value对的repeated group， 而且key是required的

- 两个概念
	- repetition level ：指明该值在路径中哪个repeated field重复
		- 针对的是repeted field的 。
		- 它能用一个数字告诉我们在路径中的什么重复字段，此值重复了，以此来确定此值的位置
		- 我们用深度0表示一个纪录的开头（虚拟的根节点），深度的计算忽略非重复字段（标签不是repeated的字段都不算在深度里）
	- definition level：指明该列的路径上多少个可选field被定义了
		- 如果一个field是定义的，那么它的所有的父节点都是被定义的
		- 从根节点开始遍历，当某一个field的路径上的节点开始是空的时候我们记录下当前的深度作为这个field的Definition Level
		- 如果一个field的definition Level等于这个field的最大definition Level就说明这个field是有数据的
		- **注意**：是指该路径上有定义的repeated field 和 optional field的个数，不包括required field，因为required field是必须有定义的

- 谓词下推：通过将一些过滤条件尽可能的在最底层执行可以减少每一层交互的数据量，从而提升性能
	- 例如”select count(1) from A Join B on A.id = B.id where A.a > 10 and B.b < 100″SQL查询中
		- 在处理Join操作之前需要首先对A和B执行TableScan操作，然后再进行Join，再执行过滤，最后计算聚合函数返回
		- 但是如果把过滤条件A.a > 10和B.b < 100分别移到A表的TableScan和B表的TableScan的时候执行，可以大大降低Join操作的输入数据
	- 无论是行式存储还是列式存储，都可以在将过滤条件在读取一条记录之后执行以判断该记录是否需要返回给调用者，在Parquet做了更进一步的优化
		- 优化的方法时对每一个Row Group的每一个Column Chunk在存储的时候都计算对应的统计信息，包括该Column Chunk的最大值、最小值和空值个数。
		- 通过这些统计值和该列的过滤条件可以判断该Row Group是否需要扫描。
		- 另外Parquet未来还会增加诸如Bloom Filter和Index等优化数据，更加有效的完成谓词下推

- 映射下推：它意味着在获取表中原始数据时只需要扫描查询中需要的列
	- 在Parquet中原生就支持映射下推，执行查询的时候可以通过Configuration传递需要读取的列的信息
	- 这些列必须是Schema的子集，映射每次会扫描一个Row Group的数据，然后一次性得将该Row Group里所有需要的列的Cloumn Chunk都读取到内存中，每次读取一个Row Group的数据能够大大降低随机读的次数，除此之外，Parquet在读取的时候会考虑列是否连续，如果某些需要的列是存储位置是连续的，那么一次读操作就可以把多个列的数据读取到内存

## Spark Stremaing 相关

### 概念
- BlockRDD：生成由spark.stremaing.blockInterval决定，一个BatchDuration 有几个block就会产生几个 partition

- 消息消费速率限定
	- 开启背压模式：spark.streaming.backpressure.enabled=true
		- 此模式如果消息堆积严重，会一次性拉取kafka中所有堆积的消息进行处理。很可能会导致程序崩溃
	- 设置每个partition消费速率, spark.streaming.kafka.maxRatePerPartition
		- 对应每个batch拉取到的消息为: 
		
			```
			maxRatePerPartition*partitionNum*batch_interval  
		
			```

## Spark 优化相关

### 优化建议

- stage 的数量跟一个job中是否要进行shuffle有关，像reduceByKey，groupbyKey等等
- 尽量用broadcast和filter规避join操作
- 因为每次job partition数量过多，导致hive表中过多小文件产生，所以需要重新指定分区，有以下俩种方法：repartition(numPartitions:Int):RDD[T]和coalesce(numPartitions:Int，shuffle:Boolean=false):RDD[T]
他们两个都是RDD的分区进行重新划分，repartition只是coalesce接口中shuffle为true的简易实现，（假设RDD有N个分区，需要重新划分成M个分区）
	- N<M。一般情况下N个分区有数据分布不均匀的状况，利用HashPartitioner函数将数据重新分区为M个，这时需要将shuffle设置为true。
	- 如果N>M并且N和M相差不多，(假如N是1000，M是100)那么就可以将N个分区中的若干个分区合并成一个新的分区，最终合并为M个分区，这时可以将shuff设置为false，在shuffl为false的情况下，如果M>N时，coalesce为无效的，不进行shuffle过程，父RDD和子RDD之间是窄依赖关系。
	- 如果N>M并且两者相差悬殊，这时如果将shuffle设置为false，父子ＲＤＤ是窄依赖关系，他们同处在一个stage中，就可能造成Spark程序的并行度不够，从而影响性能，如果在M为1的时候，为了使coalesce之前的操作有更好的并行度，可以讲shuffle设置为true。

	- **总之**：如果shuffle为false时，如果传入的参数大于现有的分区数目，RDD的分区数不变，也就是说不经过shuffle，是无法将RDD的分区数变多的。

### 参数调优

- [美团Spark参数调优参考文章](http://tech.meituan.com/spark-tuning-basic.html)
- 最重要的是数据序列化和内存调优。对于大多数程序选择Kyro序列化器并持久化序列后的数据能解决常见的性能问题。
- Executor
  - 每个节点可以起一个或多个Executor。每个Executor上的一个核只能同时执行一个task,如果一个Executor被分到了多个task只能排队依次执行
  - Executor内存主要分三块：
  		- 1、让task执行我们自己编写的代码，默认占总内存的20%。
  		- 2、让task通过shuffle过程拉取了上一个stage的task输出后，进行聚合等操作时，默认占用总内存20%；
  		- 3、让RDD持久化使用，默认60%
  - task的执行速度是跟每个Executor进程的CPU core数量有直接关系的。
  		- 一个CPU core同一时间只能执行一个线程。
  		- 每个Executor进程上分配到的多个task，都是以每个task一条线程的方式，多线程并发运行的。如果CPU core数量比较充足，而且分配到的task数量比较合理，那么通常来说，可以比较快速和高效地执行完这些task线程。

- 广播大变量
  - 当需要用到外部变量时，默认每个task都会存一份，这样会增加GC次数，使用广播变量能确保一个Executor中只有一份

- 使用Kryo优化序列化性能(如果希望RDD序列化存储在内存中，面临GC问题的时候，优先使用序列化缓存技术)
  - spark没有默认使用Kryo作为序列化类库，是因为Kryo要求注册所有需要序列化的自定义类型，这对开发比较麻烦