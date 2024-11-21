## apiserver

#### 基础流程
1. [[pod的生命周期管理#apiserver|基础概念]]  

#### default ns下面内置的kubernetes svc的作用
	1. 是由 kube-apiserver 中的 bootstrap controller 进行控制的
	2. 代表 Kubernetes API Server

	bootstrap controller的主要作用：
		- 创建 kubernetes service；
		- 创建 default、kube-system 和 kube-public 命名空间
		- 提供基于 Service ClusterIP 的修复及检查功能
		- 提供基于 Service NodePort 的修复及检查功能

#### 多个apiserver实例是如何统一管理的
1. 一个集群中 apiserver 的所有实例会在 etcd 中的对应目录下创建 key，并定期更新这个 key 来上报自己的心跳信息。
2. ReconcileEndpoints 会从 etcd 中获取 apiserver 的实例信息并更新 endpoint。

## NodeLifecycleController
	定期监控node的状态并根据node的condition添加对应的taint标签或者直接驱逐node上的pod
#### taint
##### 使用效果
- `PreferNoSchedule`：调度器尽量避免把 pod 调度到具有该污点的节点上，如果不能避免(如其他节点资源不足)，pod 也能调度到这个污点节点上，已存在于此节点上的 pod 不会被驱逐；
- `NoSchedule`：不容忍该污点的 pod 不会被调度到该节点上，通过 kubelet 管理的 pod(static pod)不受限制，之前没有设置污点的 pod 如果已运行在此节点(有污点的节点)上，可以继续运行；
- `NoExecute`：不容忍该污点的 pod 不会被调度到该节点上，同时会将已调度到该节点上但不容忍 node 污点的 pod 驱逐掉；


> [!NOTE] 避免因网络等问题引起的pod驱逐行为
>  NodeLifecycleController 会为 node 进行分区并会为每个区设置不同的驱逐速率，即实际上会以 rate-limited 的方式添加 taint，在某些情况下可以避免 pod 被大量驱逐。

## garbage collector controller
#### 删除策略
- `Orphan` 策略：非级联删除，删除对象时，不会自动删除它的依赖或者是子对象，这些依赖被称作是原对象的孤儿对象，例如当执行以下命令时会使用 `Orphan` 策略进行删除，此时 ds 的依赖对象 `controllerrevision` 不会被删除

- `Background` 策略：在该模式下，kubernetes 会立即删除该对象，然后垃圾收集器会在后台删除这些该对象的依赖对象；

- `Foreground` 策略：在该模式下，对象首先进入“删除中”状态，即会设置对象的 `deletionTimestamp` 字段并且对象的 `metadata.finalizers` 字段包含了值 “foregroundDeletion”，此时该对象依然存在，然后垃圾收集器会删除该对象的所有依赖对象，垃圾收集器在删除了所有“Blocking” 状态的依赖对象（指其子对象中 `ownerReference.blockOwnerDeletion=true`的对象）之后，然后才会删除对象本身；

#### finalizer机制
	finalizer 是在删除对象时设置的一个 hook，其目的是为了让对象在删除前确认其子对象已经被完全删除.

- k8s中默认有两种finalizer:
	- finalizerOrphanFinalizer 
	- ForegroundFinalizerfinalizer
- 当一个对象的依赖对象被删除后其对应的 finalizers 字段也会被移除，只有 finalizers 字段为空时，apiserver 才会删除该对象。

## schedule
	schedule内部的pod informer只监听status.phase 不为 succeeded 以及 failed 状态的 pod，即非 terminating 的 pod。

#### scheduleOne的主要逻辑
 - 从 scheduler 调度队列中取出一个 pod，如果该 pod 处于删除状态则跳过
- 执行调度逻辑 `sched.schedule()` 返回通过预算及优选算法过滤后选出的最佳 node
- 如果过滤算法没有选出合适的 node，则返回 core.FitError
- 若没有合适的 node 会判断是否启用了抢占策略，若启用了则执行抢占机制
- 判断是否需要 VolumeScheduling 特性
- 执行 reserve plugin
- pod 对应的 spec.NodeName 写上 scheduler 最终选择的 node，更新 scheduler cache
- 请求 apiserver 异步处理最终的绑定操作，写入到 etcd
- 执行 permit plugin
- 执行 prebind plugin
- 执行 postbind plugin

#### 正常的调度过程（包含优选和预选）

 1. 检查 pod pvc 信息
 2. 执行 prefilter plugins
 3. 获取 scheduler cache 的快照，每次调度 pod 时都会获取一次快照
 4. 执行 `g.findNodesThatFit()` 预选算法
 5. 执行 postfilter plugin
 6. 若 node 为 0 直接返回失败的 error，若 node 数为1 直接返回该 node
 7. 执行 `g.priorityMetaProducer()` 获取 metaPrioritiesInterface，计算 pod 的metadata，检查该 node 上是否有相同 meta 的 pod
 8. 执行 `PrioritizeNodes()` 算法
 9. 执行 `g.selectHost()` 通过得分选择一个最佳的 node

#### predicates

##### 预选调度算法
1. GeneralPredicates -- pod和宿主机的各种适配检查
	- PodFitsHost：检查宿主机的名字是否跟 Pod 的 spec.nodeName 一致
	- PodFitsHostPorts：检查 Pod 申请的宿主机端口（spec.nodePort）是不是跟已经被使用的端口有冲突
	- PodMatchNodeSelector：检查 Pod 的 nodeSelector 或者 nodeAffinity 指定的节点是否与节点匹配等
	- PodFitsResources：检查主机的资源是否满足 Pod 的需求，根据实际已经分配（Request）的资源量做调度
  2.与 Volume 相关的过滤规则
  3.第三种类型是宿主机相关的过滤规则，主要是PodToleratesNodeTaintsPred
  4.Pod 相关的过滤规则，主要是 MatchInterPodAffinityPred。
  5.新增的过滤规则，与宿主机的运行状况有关

#### priorities
1. `PrioritizeNodes()` 通过并行运行各个优先级函数来对节点进行打分
2. 每个优先级函数会给节点打分，打分范围为 0-10 分，0 表示优先级最低的节点，10表示优先级最高的节点
3. 每个优先级函数有各自的权重
4. 优先级函数返回的节点分数乘以权重以获得加权分数
5. 最后计算所有节点的总加权分数

常见的priorities 算法
	- SelectorSpreadPriority：按 service，rs，statefulset 归属计算 Node 上分布最少的
	- InterPodAffinityPriority：pod 亲和性选择策略，默认权重为1
	- LeastRequestedPriority：选择空闲资源（CPU 和 Memory）最多的节点，默认权重为1
	- BalancedResourceAllocation：CPU、Memory 以及 Volume 资源分配最均衡的节点，默认权重为1
	- NodeAffinityPriority：节点亲和性选择策略，默认权重为1
	- TaintTolerationPriority
	- ImageLocalityPriority：待调度 Pod 需要使用的镜像是否存在于该节点

### 优先级和抢占机制

#### 为什么要有优先级抢占机制
	在线业务要抢占离线业务的资源
----------------------------------------------------------
在离线混布的场景下：当一个 pod 调度失败后，就会被暂时 “搁置” 处于 `pending` 状态，直到 pod 被更新或者集群状态发生变化，调度器才会对这个 pod 进行重新调度。但在实际的业务场景中会存在在线与离线业务之分，若在线业务的 pod 因资源不足而调度失败时，此时就需要离线业务下掉一部分为在线业务提供资源，即在线业务要抢占离线业务的资源

#### 优先级
	PriorityClass，优先级是一个32 bit的整数，最大值不超过1000000000（10亿，1 billion），并且值越大代表优先级越高。**而超出10亿的值，其实是被Kubernetes保留下来分配给系统 Pod使用的。显然，这样做的目的，就是保证系统 Pod 不会被用户抢占掉
#### 抢占机制

 