深入做hadoop相关的工作也有一段时间了，期间零零散散看了不少源码，但很多都是看完就忘了，很形成结构化的记忆。于是决定通过流程图的方式来刻画一个MR任务在Hadoop子系统中的状态机流转过程。

## MR任务提交过程

![img](https://pic1.zhimg.com/80/v2-73ff12c7a51dc7962821e06d67c9be1c_hd.png)

一个MR任务在hadoop 客户端通过rpc 方式提交到yarn上；大致过程如上图

- JobSubmitter
  - 封装了向yarn(ClientRMService)提交的过程
  - 与hdfs交互计算任务输入数据的分片大小，以及将jar包加入DistributedCache中
- ClientServiceDelegate
  - 设计的目的是统一封装monitorJob过程获取任务执行状态，counters等信息的rpc client代理；其背后通过Java反射的方式，在任务的不同阶段会分别请求RM, AM, 或MR History Server（以下简称MHS）服务
  - 因各种情况会有部分线上任务流量降级穿透到MHS服务，而MHS服务自身实现有较大瓶颈，我们其进行了leveldb方案的改造，整体查询性能提升**20倍**

## 分片计算过程

下面看一下getSplits过程，我们默认用的是CombineFileInputFormat实现，先上图

![img](https://pic4.zhimg.com/80/v2-8add590ce801446aef9d08c031a8c3ff_hd.png)

我们知道，MR计算框架强调的是数据本地性。在图中有三个结构nodeToBlocks，rackToBlocks，blockToNodes；其中nodeToBlocks代表的是local级别，优先选择此集合中的分片，当剩下的blocks不足minSizeNode阈值时会通过blockToNodes数据结构，进行次优的分片划分的过程，以此类推。

## ApplicationMaster启动过程

当一个任务提交到RM后，需要等待RM分配资源启动AM之后才开始后续自己的资源&任务处理过程，先上图（**以下板块省去了内部自实现的调度算法**）

![img](https://pic1.zhimg.com/80/v2-616195ee1bab87905cb4e3c31182142c_hd.png)

此时涉及到RMApp, RMAppAttempt，RMNode，以及RMContainer等状态机的轮转

![img](https://pic4.zhimg.com/80/v2-3f2f539ad5ebb736d7a85d4b606a9683_hd.png)

## MR任务状态机流转

当Yarn RM通过ContainerManagerProtocol协议将AM Container启动之后，AM便开始了Map/Reduce（一般map执行完之后）任务调度过程

![img](https://pic3.zhimg.com/80/v2-c9babe0d2fb818f1037c5c00a369da74_hd.png)

运行期间涉及到的状态机有Job, Task, 以及TaskAttempt，当然还有AM register/unregister过程Yarn RM系统对应的状态机转换；大致描述一下流程：

- MR AM启动，通过Job状态机初始化
  - 初始化CommitterEventHandler，用于最后job完成时通过commit过程将temp目录的数据转移到final 目录中
  - 初始化Map/Reduce Task以及对应的TaskAttempt（Task具体的某次尝试），通过RMContainerAllocator先对map task进行调度，后进行reduce task调度
- 通过ApplicationMasterService向RM注册自己，代表某个RMAppAttempt对应的AM Container已启动，可以定期向RM发送allocate心跳了
- 在RMContainerAllocator中通过allocate心跳向RM请求资源，得到response之后将分得的container再按优先级assign给对应的task
- 在TaskAttempt得到container之后通过ContainerLanucher向NodeManager请求启动Container
- 任务运行完成做对应的commit，clean操作之后，通过ApplicatioinMasterService告知RM任务完成，此时RMApp/Attempt做任务完成的状态转换