---
title: Yarn Federation源码串读
date: 2019-11-05 01:47:46
tags:
	- 架构
	- Hadoop
	- Yarn
---

[知乎链接](https://zhuanlan.zhihu.com/p/79378807)

## Federation架构总览

- Federation: 主要有四个模块，Router ，StateStore，AMRMProxy, Global Policy Generator；从架构上来看，有点类似于后端的微服务架构中**服务注册发现**模块

![img](https://pic4.zhimg.com/v2-7ee20bc86d8be49d25b5ca3897d3278f_b.png)

##  Router模块

- 类似于微服务的网关模块；通过state store获取具体的集群配置策略，将client端submit请求转发到对应的subCluster中
- 代码结构
- hadoop-yarn-server-router：router组件核心实现，分为对接admin用户的协议和client用户协议，以及web server三个子模块实现  

![img](https://pic3.zhimg.com/v2-b6ee24339266083c97b6642c4f3a081e_b.png)



- hadoop-yarn-server-common-federation-router：包含了Router的各种Policy，具体控制router给子集群分配app的策略

![img](https://pic2.zhimg.com/v2-e7402699fce4808743375957f68a8b11_b.png)



### **Router- clientrm**

- 负责接收客户端命令请求，并根据对应router具体配置的policy将客户端请求转发到HomeSubcluster上
- 在每一个router服务上随着启动，用来监听客户端作业提交，实现了Client与RM沟通的RPC协议接口(ApplicationClientProtocol)；作为client的proxy，执行一系列的chain interceptor），通常FederationClientInterceptor需作为最后一个拦截器
- 当然RouterClientRMService某种程度上针对的是Server测，取代原来RM侧**RMClientService**；在客户端具体的调用还是在**YarnClientImpl**；之间通过RPC通信
- 初始化： 获取配置文件中配置的拦截器，默认是DefaultClientRequestInterceptor  

![img](https://pic3.zhimg.com/v2-74b97de7e4a90be9f4f7f5e703dcbd56_b.png)

- DefaultClientRequestInterceptor只是做了简单的请求透明转发；没涉及到多子集群的处理
- FederationClientInterceptor：面向client，隐藏了多个sub cluster RM；但是目前只实现了四个接口：**getNewApplication, submitApplication, forceKillApplication and getApplicationReport**
- **FederationClientInterceptor**
- clientRMProxies: 子集群id与对应的通信client的key value集合
- federationFacade: 对应的state store具体实现
- policyFacade: 路由策略的工厂  

![img](https://pic1.zhimg.com/v2-d6bbfd466b88388bdb9777d17d159210_b.png)

- 一个任务的提交需经过**FederationClientInterceptor.getNewApplication**和**submitApplication**接口，前者获得新的**applicationId**, 后者通过获得的**applicationId**将任务提交到具体的sub Cluster RM；这一个阶段没有经过与state store的写操作

![img](https://pic2.zhimg.com/v2-6824ab1d9c5489ea0264ef5b79f9f075_b.png)

- getNewApplication实现只是**随机**的选择一个active sub cluster来获取一个新的**applicationId**；而subClustersActive是通过具体实现的**state store**来获取，此处有过滤active的字段
- submitApplication，方法注释有讨论各种failover的处理情况；
- RM没挂的情况：如果state store 更新成功了，则多次提交任务都是幂等的
- RM挂了：则router time out之后重试，选择其他的sub cluster
- Client挂了：跟原来的/ClientRMService/一样
- 通过policyFacade加载策略，根据context与blacklist为当前提交选择sub cluster；具体逻辑在**FederationRouterPolicy.getHomeSubcluster**  

![img](https://pic4.zhimg.com/v2-c6dbd82e45d5c69bd30b07e4f7e077a3_b.png)

- 同步提交任务至目标sub cluster

![img](https://pic3.zhimg.com/v2-da2fa220baae8ba7238a1b77bebe009a_b.png)

**疑问&&待确定的点**

- client —> router —> rm： 这条链路如果router挂了如何failover；**在submitApplication方法上方有较为详细的边界情况处理解释**
- **是否支持多个router？以及在配置中如何指定多个router？防止一个router挂掉的情况**
- **需要确定是否有机制来维系真正存活的cluster，是否会动态摘除down掉的RM**

## Policy State Store模块

### FederationStateStoreFacade

- 作为statestore的封装，抽象出一些重试和缓存的逻辑

### FederationStateStore

- 一般采用**ZookeeperFederationStateStore**的方式
- **ZookeeperFederationStateStore**  实现中，对应的数据存储结构如下  

![img](https://pic4.zhimg.com/v2-b4e79639629bdec688ec4efb6f9b275f_b.png)



- 通过心跳维系了RM是否是active；通过**filterInactiveSubClusters**来决定是否需要过滤存活的RM

![img](https://pic1.zhimg.com/v2-1de1b20d5b5a5c01c26e6724695cf440_b.png)

- **实例化过程**
- 加载配置***yarn.federation.state-store.class***：默认实现是***MemoryFederationStateStore***

![img](https://pic4.zhimg.com/v2-e5888bd36c0c482c10c2bb22782ceb27_b.png)

### SubClusterResolver

- 用来判断某个指定的node是属于哪个子集群的工具类;主要有getSubClusterForNode，getSubClustersForRack方法
- 实例化过程
- 加载配置yarn.federation.subcluster-resolver.class: 默认实现是DefaultSubClusterResolverImpl

![img](https://pic2.zhimg.com/v2-cf9ebad47e5c250ef5a545ee4e1c1b05_b.png)

- 在**load**方法中，获取了machineList，定义list的地方是在一个文件中通过**yarn.federation.machine-list**获取文件位置；且文件中的内容格式如下

![img](https://pic3.zhimg.com/v2-7d1685ce8602e9884412e42a9faaf72e_b.png)

- 解析文件之后，将machine依次添加到**nodeToSubCluster**，**rackToSubClusters**集合中

## AMRMProxy模块

- 看完client—>rm侧的提交任务模块之后（**router**），接下来可以分析AM与RM侧的交互模块(**AMRMProxy**)  

![img](https://pic4.zhimg.com/v2-d90061b8beeb05e40586a3dada4222a7_b.png)

- AMRMProxyService ：如上图所示，起于所有的NM之上的服务，作为AM与RM之间通信的代理；会将AM请求转发到正确的HomeSubCluster
- FederationInterceptor: 作为AMRMProxyService中的拦截器，主要做AM与RM之间请求转发

### AMRMProxyService — FederationInterceptor

- 类比Router，FederationInterceptor作为AMRMProxy的请求拦截处理
- 在AM的视角，**FederationInterceptor**的作用就RM上的**ApplicationMasterService**；AM通过**AMRMClientAsyncImpl**或**AMRMClientImpl** 走RPC协议与**AMRMProxyService** 交互

![img](https://pic1.zhimg.com/v2-0ec5e08fa702ab8a00bb11528da4b138_b.png)

**registerApplicationMaster详解**

- 按照正常的AM流程分析，由**AMLauncher**启动container之后须首先会调用**registerApplicationMaster**方法初始化权限信息以及将自己注册到对应的RM上去；对应到**FederationInterceptor**是如下方法

![img](https://pic1.zhimg.com/v2-a322c31e9908ed4f8b08c05130c67688_b.png)

- 制造一种假象：RM永不会挂掉；有可能会因为超时或者RM挂掉等原因而导致发出多个重复注册的请求，此时都会返回最近一次成功的注册结果；所以这也就是为什么registermaster这个方法必须为线程安全的原因

![img](https://pic4.zhimg.com/v2-e127ca2f6e0120575bc76c5b38126a43_b.png)

- 目前只是往HomeSubCluster上注册AM，而不会往其他子集群上注册。是为了不影响扩展性；即不会随着集群的增多AM呈线性扩展；应该是后续按需注册sub-cluster rm

![img](https://pic1.zhimg.com/v2-571134517c85d3c6fd5a237412618a74_b.png)

- **this.homeRMRelayer**是具体的跟RM通信的代理，其创建方式在**FederationInterceptor.init**方法中

![img](https://pic1.zhimg.com/v2-4afe5125de9a0e80a3440f6807e12e84_b.png)

- 最后在返回response之前，会根据作业所属的queue信息从statestore中获取对应的策略，并初始化**policyInterpreter**

![img](https://pic1.zhimg.com/v2-bab165a60bd63a3c43549950825d28a0_b.png)

### Allocate详解

- 周期性的通过心跳与HomeCluster和SubCluster RMs交互；期间可能伴随有SubCluster 上AM的启动和注册
- **splitAllocateRequest**：将原来的request重新构造成面向所有已经注册的sub-cluster rm request

![img](https://pic2.zhimg.com/v2-bf455e34f88d53a7dcc5ffea1d51d8a9_b.png)

- 具体到实现：通过requestMap来放置clusterId与allocateRequest的对应关系；通过uamPool获取已经注册UAM的sub clusterId并构建request

![img](https://pic3.zhimg.com/v2-3bb90c887c818638bb335ff6d46b9d46_b.png)

- 后面的步骤是根据所有已经注册的home cluster和sub cluster id构建release, ask, blacklist等请求
- 对于资源的请求拆分：这里会去调federation policy interpreter将原来request中的**askList(Resource Request List)**根据策略拆分到各个子集群；所以这里会涉及到Federation Policy调用，具体的分析接下来会单独拎出一小节解释

![img](https://pic1.zhimg.com/v2-675411d4dfdc72c8d89ee301a8709b7c_b.png)

- 拿到**asks**后，会将<subClusterId, resourceRequestList>的对应关系，加入到**requestMap**中
- **注意：**这里借助**findOrCreateAllocateRequestForSubCluster**方法实现如果requestMap中不存在asks中对应的subClusterId，会新new一个request塞入map；后续这个request会在对应的subCluster上启动**UAM**
- **因为对于新的job，刚开始确实是只在homeCluster上启动了AM**
- **sendRequestsToResourceManagers**
- splitAllocateRequest之后就是将构造好的请求发送到对应的cluster上；顺带在所有的subcluster启动UAM并注册上(如果之前没有启动的话)；返回值是所有新注册上的UAM

![img](https://pic3.zhimg.com/v2-34f142d2135a9f8b12ae234966bde9b6_b.png)

- **registerWithNewSubClusters** 用来在其他子集群中创建新的UAM实例
- 在uamPool中不存在的被认为是新集群（*有点与**splitAllocateRequest**） 取AllUAMIds逻辑矛盾*）
- 对newSubClusters集合迭代，依次在subClaster上启动UAM，并注册UAM

![img](https://pic4.zhimg.com/v2-1051b6772a9c893a910f3497cb9cc327_b.png)

- 最后针对不同的cluster，调用不同的clientRPC请求资源

![img](https://pic2.zhimg.com/v2-2e0d3265a416a8bd9131559ce958b2f1_b.png)

- **mergeAllocateResponses**
- 用于合并所有资源请求返回的allocateResponse。实现里面是对**asyncResponseSink**容器的迭代，而asyncResponseSink的写入是在HeartBeatCallback逻辑里的
- 对于allocateResponse的合并操作在**mergeAllocateResponse**中
- **mergeRegistrationResponses**
- 是在注册完其他的sub cluster之后将UAM加入到最终合并的AllocateResponse中；主要是对allocatedContainers以及NMTokens集合做增加

### finishApplicationMaster详解

- 结束任务的时候有点类似allocate，需要向所有的sub cluster发送finish请求；目前是丢到一个compSvc线程池中批量执行*finshApplicationMaster
- 在线程池中执行sub cluster finish的同时，也会调用home cluster rm进行finish操作

## Federation Policy模块

- federation policy模块通过FederationPolicyManager的接口实现来统一加载

![img](https://pic1.zhimg.com/v2-6a2db128fc5762aba3591cf4912bfc40_b.png)

- **FederationPolicyInitializationContext**：初始化FederationAMRMProxyPolicy和FederationRouterPolicy的上下文类
- **federationStateStoreFacade**: policy state strore的具体实现实例
- **federationPolicyConfiguration**: 具体的策略配置
- **federationSubclusterResolver**：用来判断某个指定的node是属于哪个子集群的工具类
- **homeSubcluster**：当前application实际AM运行的集群ID

## Policy 具体的实现列举

### amrmproxy模块的policy实现

- **LocalityMulticastAMRMProxyPolicy**
- \1. 如果是有偏好的host的话，会根据*SubClusterResolver* resolve cluster的结果转发到对应的cluster，但如果没有resolve的话，会默认将请求转向home cluster
- \2. 如果有机架的限制，策略同上
- \3. 如果没有host/rack偏好的话，会根据*weights*转发到对应的集群；weights的计算根据*WeightedPolicyInfo*以及*headroom*中的信息
- \4. 所有请求量为0的请求都会转发到所有我们曾经调度过的子集群中（以防用户在尝试取消上一次的请求）
- 注：该实现始终排除当前未活跃的RM
- **具体实现细节待深究**

**router模块的policy实现**

- 总体来说router端的策略偏简单，自己定制也容易
- 默认实现是**UniformRandomRouterPolicy**，随机转发client请求到某个alive的cluster

## 一些问题

- 在NM侧，不能开启**FederationRMFailoverProxyProvider**，这个统一在获取RMAddress逻辑上有不足，导致NM启动时拿到的RMAddress是localhost无法通过ResourceTracker连上RM，最终注册失败

![img](https://pic1.zhimg.com/v2-edc97c0d9ba93b9094e561e6cd7b5464_b.png)