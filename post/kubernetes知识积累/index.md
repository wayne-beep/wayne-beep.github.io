

## 1. Kubernetes各个组件的功能

### 1.1. kube-apiserver

master组件kube-apiserver负责将kubernetes资源组、资源版本、资源已restful形式对外暴露并且提供服务。kubernetes集群中所有的组件都是通过它来操作资源对象。kube-apiserver也是唯一与ETCD集群进行交互的核心组件，kube-apiserver是无状态组件。例如kubectl创建一个deployment资源通过请求api-server的HTTP接口将资源对象存储在ETCD中。kubernetes将所有的数据存储在ETCD集群中前缀为/registry目录下。

kube-apiserver的特性：

1. 将kubernetes系统中的所有资源对象都封装成RESTFUl风格的API接口进行管理
2. 可以进行集群状态和数据管理，是唯一与ETCD交互的组件
3. 拥有丰富的机器安全访问机制，以及认证、授权、准入控制
4. 体用里集群个组件的通信和交互功能

### 1.2. kube-controller-manager

master组件kube-controller-manager，被称为管理控制器，负责管理kubernetes集群中的节点Node、Pod副本、Service、Endpoint、namespace、ServiceAccount、ResourceQuta等。

Controller Manager负责确保kubernetes系统的实际状态维护，默认提供了一些控制器比如：Deployment Controller、StatefulSet Controller、Namespace Controller、PersistentVolume Controller、Service Controller、Endpoint Controller等等。每个控制器都通过api-server组件提供的接口试试监控整个集群每个资源对象的状态，当系统更新或者发生各种故障而导致系统状态出现变化时，会尝试将系统修复到期望状态。

总结kube-controller-manager的功能就是：**及时发现并执行自动化修复流程，确保集群始终处于预期的工作状态**。

kube-controller-manager是可实现高可用的，通过基于ETCD分布式锁实现leader选举。多实例运行时，通过api-server提供的资源锁进行竞争，抢先获取锁的实例称为Leader节点，并且运行kube-controller-manager组件的主逻辑，而未获取锁的实例被称为Candidate候选节点，处于阻塞状态，主节点因某些原因退出后，候选节点通过选举机制参与竞选，称为Leader的节点继续工作。

### 1.3. kube-scheduler

Kube-scheduler组件被称为调度器，**负责在kubernetes集群中为一个pod资源对象找到合适的节点并在该节点上运行。调度器每次只调度一个pod资源对象，为每一个pod资源对象寻找合适节点的过程是一个调度期**。

kube-scheduler组件监控整个集群的pod资源对象和node资源对象，当监控到新的pod资源对象时，会通过调度算法为其选择最优节点。调度算法有两种，分别为节点预选和节点优选。除调度策略外，kubernetes还支持优先级调度、抢占机制和亲和性调度功能。

#### 1.3.1. 预选

根据配置的PredicatesPolicies（默认为DefaultProvider中定义的default predicates policies集合）过滤掉那些不满足这些Policies 的 Node，预选的输出作为优选的输入；

**预选阶段算法**

**NoDiskConflict**：评估是否存在volume冲突。如果该 volume 已经 mount 过了，k8s可能会不允许重复mount(取决于volume类型)；

**NoVolumeZoneConflict**：评估该节点上是否存在 Pod 请求的 volume；

**PodFitsResources**：检查节点剩余资源(CPU、内存)是否能满足 Pod 的需求。剩余资源=总容量-所有 Pod 请求的资源；

**MatchNodeSelector**：判断是否满足 Pod 设置的 NodeSelector；

**CheckNodeMemoryPressure**：检查 Pod 是否可以调度到存在内存压力的节点；

**CheckNodeDiskPressure**：检查 Pod 是否可以调度到存在硬盘压力的节点；

#### 1.3.2. 优选

根据配置的PrioritiesPolicies给预选后的 Node 进行打分排名，得分最高的 Node 即作为最适合的 Node ，该 Pod 就绑定（Bind）到这个 Node 。

**注：**如果经过优选将 Node 打分排名后，有多个 Node 并列得分最高，那么kube-scheduler将使用RR机制从中选择一个 Node 作为目标 Node 。

**优选阶段算法**

依次计算该 Pod 运行在每一个 Node 上的得分。主要算法有：

**LeastRequestedPriority**：最低请求优先级，即 Node 使用率越低，得分越高；

**BalancedResourceAllocation**：资源平衡分配，即CPU/内存配比合适的 Node 得分更高；

**SelectorSpreadPriority**：尽量将同一 RC/Replica 的多个 Pod 分配到不同的 Node 上；

**CalculateAntiAffinityPriority**：尽量将同一 Service 下的多个相同 Label 的 Pod 分配到不同的 Node；

**ImageLocalityPriority**：Image本地优先，Node 上如果已经存在 Pod 需要的镜像，并且镜像越大，得分越高，从而减少 Pod 拉取镜像的开销(时间)；

**NodeAffinityPriority**：根据亲和性标签进行选择；

#### 1.3.3. 选举

kube-scheduler的选举机制与kube-controller-manager相同

### 1.4. kubelet

kubelet组件用于管理节点，接收、处理、上报api-server组件下发的任务。kubelet会像api-server注册节点自身信息。它主要负责所在节点上的pod资源对象的管理，例如pod资源对象的创建、修改、删除、驱逐及pod生命周期管理。

kubelet会定期监控所在节点的资源使用状态并上报给api-server，这些资源数据帮助kube-scheduler调度器为pod资源对象预选节点。kubelet也会对所在节点的镜像和容器做清理工作，保证节点上的镜像不会沾满磁盘空间、删除的容器释放相关资源。

kubelet组件提供了三个开发接口：

- Container Runtime Interface：简称CRI（容器运行时接口），提供容器运行时通用插件接口服务。CRI定义了如果您器镜像服务的接口。
