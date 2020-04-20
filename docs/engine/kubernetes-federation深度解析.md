## Federation v1

### 基本概念

项目地址 (**deprecated**)：

[kubernetes-retired/federationgithub.com![图标](https://pic3.zhimg.com/v2-406bd93b1ff9779bd3cfd693d0a6b552_ipico.jpg)](https://link.zhihu.com/?target=https%3A//github.com/kubernetes-retired/federation)

架构图

![img](https://pic3.zhimg.com/80/v2-6115cfbfc11c62f64423f018edbe495a_720w.jpg)

主要由 API Server、Controller Manager 和外部存储 etcd 构成。v1.11之后被废弃

- federation-apiserver：类似kube-apiserver，兼容k8s API，只是对联邦处理的特定资源做了过滤（部分资源联邦不支持，故使用apiserver来过滤）
- federation-controller-manager：提供多个集群间资源调度及状态通同步，工作原理类似kube-controller-manager
- kubefed：Federation CLI工具，用来将子集群加入到联邦中
- etcd：存储federation层面的资源对象，供federation control plane同步状态
- 定义demo；基于annotation

![img](https://pic4.zhimg.com/80/v2-96198f56d616222ff724e4cc2c22f7df_720w.jpg)

### 缺点

- 将成员集群作为一种资源进行管理，但是并未添加新的资源定义，以至于每当创建一种新资源都要新增Adapter
- 无法有效的在多个集群管理权限，如不支持 RBAC
- 没有独立的API，版本迭代不好演进，联邦层级的设定与策略依赖 API 资源的 Annotations 内容，这使得弹性不佳

## Federation v2

Github项目：

[https://github.com/kubernetes-sigs/kubefedgithub.com](https://link.zhihu.com/?target=https%3A//github.com/kubernetes-sigs/kubefed%5d(https%3A/github.com/kubernetes-sigs/kubefed))


官方文档

[https://github.com/kubernetes-sigs/kubefed/blob/master/docs/userguide.md#verify-your-deployment-is-workinggithub.com](https://link.zhihu.com/?target=https%3A//github.com/kubernetes-sigs/kubefed/blob/master/docs/userguide.md%23verify-your-deployment-is-working)

架构图

![img](https://pic1.zhimg.com/80/v2-cc5ce2ba68be184aa114635cfd76fec4_720w.jpg)

### 基本组成

- admission-webhook：提供了准入控制；做各种校验(api version, 参数校验)
- kubefed controller-manager: 处理自定义资源以及协调不同集群间的状态

### 基本概念

相比v1来说，v2在架构上有较大的调整，去掉aggregated api server实现，采用CRD+operator的方式定义一系列自定义联邦资源，并通过自定义controller实现跨集群资源协调能力

- Federate：联邦（Federate）是指联结一组 Kubernetes 集群，并为其提供公共的跨集群部署和访问接口
- KubeFed：Kubernetes Cluster Federation，为用户提供跨集群的资源分发、服务发现和高可用
- Host Cluster：部署 Kubefed API 并允许 Kubefed Control Plane 通过 kubefedctl join 使得成员集群加入到主集群（Host Cluster）
- Member Cluster：通过 KubeFed API 注册为成员并受 KubeFed 管理的集群，主集群（Host Cluster）也可以作为成员集群（Member Cluster）
- ServiceDNSRecord： 记录 Kubernetes Service 信息，并通过DNS 使其可以跨集群访问
- IngressDNSRecord：记录 Kubernetes Ingress 信息，并通过DNS 使其可以跨集群访问
- DNSEndpoint：一个记录（ServiceDNSRecord/IngressDNSRecord的） Endpoint 信息的自定义资源

### Admission Webhook

为custom resource 提供的准入控制，作用在CR的创建过程的校验阶段

- mutating 和validating两种类型的hook差别在于mutating可以在校验过程中通过打补丁的形式修改resource，而validating只能允许或拒绝resource；kubeFed实现的是validating模式

![img](https://pic2.zhimg.com/80/v2-bfb30c55a2ffd3fe8dd670e73b34fa51_720w.jpg)

目前有三种CR的admissionHook；用于校验自定义资源的格式正确性

![img](https://pic2.zhimg.com/80/v2-05d97bee9b14188534f64ed3b01de7c9_720w.jpg)

### CRD详解

- 主要CRD以及交互图

![img](https://pic4.zhimg.com/80/v2-6e0cb36a54d7e2b3d96fe2cee1a233f3_720w.jpg)

- CRD 两大基本块

- - Type Configuration：自定义的被联邦托管的对象；主要组成是：Templates, Placement, Overrides…
  - Cluster Configuration：用来保存被联邦托管的集群的 API 认证信息

- Type Configuration：用来描述将被联邦托管的资源类型， 定义了哪些Kubernetes API资源要被用于联邦管理；可通过`kubefedctl enable <> `来使新的api 资源可以被联邦管理；该操作会在HostCluster中创建一个新的CRD，然后通过sync controller 同步到其他member cluster

- - Templates：用于描述被联邦的资源
  - Placement：用来描述将被部署的集群
  - Overrides：允许对部分集群的部分资源进行覆写；schedule的过程就是通过复写member cluster中配置的replica数量来动态调整跨集群负载的
  - demo

![img](https://pic2.zhimg.com/80/v2-fe7268d9605757cdcbd7e92ac9e80781_720w.jpg)

- Cluster Configuration：用来定义哪些Kubernetes集群要被联邦。可透过kubefedctl join/unjoin来加入/删除集群，当成功加入时，会建立一个KubeFedCluster组件来储存集群相关信息，如API Endpoint、CA Bundle等。这些信息会被用在KubeFed Controller存取不同Kubernetes集群上，以确保能够建立Kubernetes API资源

- - Cluster类型

  - - Host: 用于提供KubeFed API与控制平面的集群
    - Member: 通过KubeFed API注册的集群，并提供相关身份凭证来让KubeFed Controller能够存取集群。Host集群也可以作为Member被加入

![img](https://pic3.zhimg.com/80/v2-36e0d718473940d54ae9b47896e9ced6_720w.jpg)

### 源码详解

### 总览

四种API群组

![img](https://pic1.zhimg.com/80/v2-3390976612b1782f334d4793bbb6265c_720w.jpg)

代码级组件交互流程（不包含网络模块）

![img](https://pic2.zhimg.com/80/v2-103179db76c75bea0efd7dfa2ac9ebbd_720w.jpg)

- StatusController和SyncController 都使用了**FederatedInformer，**用来感知所有member cluster中某中联邦资源的变更。如果变更则从HostCluster中获取最新的同步到memberCluster中

**FederatedInformer**实现原理图

![img](https://pic2.zhimg.com/80/v2-4677a9a5f3977eac2964dd8170623d59_720w.jpg)

### 多集群调度方式

- KubeFed提供了一种自动化机制来将工作负载实例分散到不同的集群中，这能够基于总副本数与集群的定义策略来将Deployment或ReplicaSet资源进行编排。编排策略是通过建立ReplicaSchedulingPreference(RSP)文件，再由KubeFed RSP Controller监听与RSP内容来将工作负载实例建立到指定的集群上
- 目前仅支持Deployment和ReplicaSet两种调度类型，社区有支持StatefulSet的设计文档，还未实现 [https://github.com/kubernetes/community/pull/437/files](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/community/pull/437/files)

![img](https://pic1.zhimg.com/80/v2-0e454ba069efb9a6b29d00ceacfb4dd0_720w.jpg)

- 以下为一个RSP 范例：假设有三个集群被联邦，名称分别为：ap-northeast、us-east 与 us-west

![img](https://pic2.zhimg.com/80/v2-88d362c882e843a378013e54ba399bb9_720w.jpg)

- 分配机制；源码可参考：kubefed/pkg/controller/util/planner/planner.go

- - 若spec.clusters未定义的话，replicas会被均匀的分配到member clusters中，即配置为：`{“*”:{Weight: 1}}`
  - 第一轮分配时，先按照每个集群minReplica 分配，如果总的分配未达到totalReplica，则再按照权重进行第二次分配（分配过程中需注意目标集群是否overflow）
  - 如果RSP可以打开reblance 开关，用于平衡集群负载：将长期pending的pod move到资源较为空闲的集群
  - 集群是否空闲是通过check capacity 来计算：capacity根据当前集群上所有pod的状态来决定

### Enable Resource过程详解

- 针对某个apiResource enable federation时，需根据apiResource生成**FederatedTypeConfig**；这是用来生成custom resource的配置，将生成新的CR资源，kind带Federated 前缀

![img](https://pic2.zhimg.com/80/v2-7625983764c25ac8795e58bc822b97a9_720w.jpg)

- 通过`newSchemaAccessor` 获得待federate的apiResource json schema

![img](https://pic4.zhimg.com/80/v2-b021269bba0949c09f98fd152fa9a60f_720w.jpg)

- 第三步是生成由federateTypeConfig和validation组成的自定义资源CustomResourceDefinition（真正重要的部分）

![img](https://pic4.zhimg.com/80/v2-4c96ae24f96a10e44954da0434a078c7_720w.jpg)

- - validation是通过`federatedTypeValidationSchema`生成

![img](https://pic2.zhimg.com/80/v2-d0978d06f8c48653562d7f1c329f5cc5_720w.png)

- - 这其中有我们熟悉的placements，overrides，template的校验定义，用于校验federated resource 的一些基本属性，template不做校验

![img](https://pic4.zhimg.com/80/v2-03d59eed980883a940b64b5b76743ddf_720w.jpg)

- - crd 和federateTypeConfig 共同组成typeResource，更新到etcd中

![img](https://pic3.zhimg.com/80/v2-5a723440c55e178831cea1aae210cd4e_720w.jpg)

### 网络模式

- 跨集群访问设计文档：[ https://github.com/kubernetes/community/blob/master/contributors/design-proposals/multicluster/federated-services.md](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/community/blob/master/contributors/design-proposals/multicluster/federated-services.md)
- Kubefed 还有一个亮点功能是跨集群间的网络访问。Kubefed 通过引入外部 DNS（外部DNS可使用自建的CoreDNS），将 Ingress Controller 和 metallb 等外部 LB 结合起来，使跨集群的流量可配置
- 以 Ingress 举例，当集群中创建了一个Ingress（路由规则）Ingress DNS Controller 就能观察到，并在所有联邦集群中同步更新IngressDNSRecord；IngressDNSRecord 变更导致被DNSEndpointController感知到也重新sync一遍DNSEndpoints；此时全局DNSController紧接着更新自己的dns记录

![img](https://pic2.zhimg.com/80/v2-a107b39ac385ba2b0ec5435db5621215_720w.jpg)