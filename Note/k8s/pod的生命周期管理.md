## pod创建的过程
### kubectl create

#### 客户端参数检查验证
1. 镜像名称
2. 镜像拉取策略校验
#### 对象生成
1. 获取pod默认生成器
2. 生成运行时对象
> [!NOTE] api groups和version 
> k8s的api带版本号并且被划分为不同的api groups
> k8s一般支持多版本的api groups,为了找到最合适的api,会调用apiserver进行获取
> 但是为了提高性能，一般会在本地~/.kube/cache/discovery目录缓存这些schema文件。

#### 按照正确的格式输出创建的对象
#### 客户端的认证支持
  -- kubeconfig

### apiserver
#### 认证
	apiserver在第一次启动的时候，会根据用户提供的命令行参数组装一个合适的认证器列表
	每一个请求都会逐一经过这个请求列表，直到有一个认证通过
- 客户端证书认证（x509）
- HTTP Basic（静态密码文件）
- Bearer Token（静态Token文件）
- webhook认证（使用外部 webhook 服务进行认证）

> [!NOTE] 
> 认证成功后，Authorization头信息将会从请求中移除，用户信息会添加到请求的上下文信息中。这样后续步骤就可以访问到认证阶段确定的请求用户的信息了。

#### 鉴权
	kube-apiserver需要基于用户提供的命令行参数，来组装一个合适的鉴权器列表来处理每一个请求。当所有的鉴权器都拒绝该请求时，请求会终止
	通过指定--authorization-mode参数设置授权机制，至少需要指定一个
- AlwaysAllow
- AlwaysDeny
- ABAC
- Webhook
- RBAC
- Node

#### Admission control
	持久化（存到etcd之前）的最后一道保障
- 变更准入控制器（Mutating Admission Controller）用于变更信息，能够修改用户提交的资源对象信息
- 验证准入控制器（Validating Admission Controller）用于身份验证，能够验证用户提交的资源对象信息

### etcd
	kube-apiserver将反序列化HTTP请求（解码），构造运行时对象（runtime object），并将它持久化到etcd。

> [!NOTE] apiserver如何找到每一个资源对应的操作Handler
>  
> - 1.API注册。当 kube-apiserver 启动时，它会加载所有已注册的资源类型和它们的处理函数。这些资源和它们的处理逻辑通常在 Kubernetes 源代码中定义，并在编译时注册到 API Server 中。
> 2.REST处理器
> - 每个资源类型都有一个对应的 REST 处理器，该处理器用于处理 HTTP 请求。REST 处理器实现了资源的创建、读取、更新、删除等（CRUD）操作的具体逻辑。
> 

### schedule
	调度器会过滤出所有在PodSpec中NodeName字段为空的pod，然后尝试为这些pod找到一个适合其运行的节点。
	当调度器将一个pod调度到一个节点后，那个节点上的kubelet就会接手开始具体的创建工作。

> [!NOTE] kubelet是如何指导pod被绑定到节点上
> - Pod 对象在 Kubernetes API Server 上通过调度器绑定到一个节点之后，Pod 对象中会更新 `.spec.nodeName` 字段。
> - 绑定节点的 Kubelet 通过 Kubernetes API Server Watch 机制监控 Pod 资源的变化。当发现有新添加或更新到它管理的节点上的 Pod 时，会触发相应的处理逻辑。
### kubelet

![[kubelet.png]]
#### 容器管理线程模型
	kubelet中的线程模型属于master/wroker模型，通过单master来监听各种事件源，并为每个Pod创建一个goroutine来进行Pod业务逻辑的处理，master和wroker之间通过一个状态管道来进行通信

1. 事件管道与容器管理线程

	- 当 kubelet 接收到一个新创建的 Pod 的信息时，它做的第一件事就是为这个 Pod 创建一个专用的**事件管道(Event Pipe)**。
	- 同时，kubelet 会启动一个[[#容器管理主线程]] 这个线程专门负责从该 Pod 的事件管道中读取和处理事件。

2. 获取 Pod 最新状态信息

- 容器管理主线程会根据**最后同步时间 (Last Synchronized Time)** 来获取 kubelet 中关于该 Pod 的最新事件。
    - 对于新建 Pod 而言，由于还没有进行过同步，因此会利用 PLEG (Pod Lifecycle Event Generator) 的机制：
        - PLEG 会更新 Pod 的时间戳，并广播一个默认的空状态作为当前最新状态。
- 这样一来，容器管理主线程就能获得两方面的信息：
    - 本地 podCache 中关于该 Pod 的最新状态。
    - 事件源中传递的关于该 Pod 的信息。

3. 整合状态信息并更新

- 获取到上述两方面信息后，容器管理主线程会将其与 kubelet 中两个重要组件所维护的 Pod 容器状态信息进行整合：
    - **Status Manager:** 负责将 Pod 状态的更新及时同步到 API Server。
    - **Probe Manager:** 负责管理 Pod 中容器的 Liveness Probe 和 Readiness Probe (探针) 逻辑。
- 通过整合来自事件源、本地缓存以及 Status Manager 和 Probe Manager 的信息， kubelet 得到了对当前 Pod 状态的最最新感知。

#### kubelet创建容器流程
##### 准入检查（硬性条件）
1. 驱逐管理
	根据当前的资源压力，检测对应的Pod是否容忍当前的资源压力（是否满足资源压力限制，是否可以污点容忍）
2. 预选检查
	1. 根据当前活跃的容器和当前节点的信息来检查是否满足当前Pod的基础运行环境（亲和性检查）
	2. 如果当前的Pod的优先级特别高或者是静态Pod，则会尝试为其进行资源抢占，会按照QoS等级逐级来进行抢占从而满足其运行环境（QoS）
3. **kubelet创建事件管道与容器管理主线程**，同步最新状态
4. kubelet进行准入控制检查（软件环境）
	1. **容器运行时**和版本等检查
5. 更新容器状态
6. Cgroup配置
	1. 启动一个PodContainerManager为对应的Pod根据其QoS等级来进行Cgroup配置的更新
![[kubelet-cgroup.png]]

> [!NOTE] Cgroup配置
> > kubelet 会在 node 上创建了 4 个 cgroup 层级，从 node 的**root cgroup**（一般都是**/sys/fs/cgroup**）往下：
> 1. **Node 级别**：针对 SystemReserved、KubeReserved 和 k8s pods 分别创建的三个Node-level cgroup；
> 2. **QoS 级别**：在`kubepods`cgroup 里面，又针对三种 pod QoS （guaranteed、brustable、besteffort）分别创建一个 sub-cgroup：
> 3. **Pod 级别**：每个 pod 创建一个 cgroup，用来限制这个 pod 使用的总资源量；
> 4. **Container 级别**：在 pod cgroup 内部，限制单个 container 的资源使用量。

7. Pod基础运行环境准备
![[kubelet1.png]]
## pod删除的过程

1. **发出删除请求**：
    
    - 当用户使用 `kubectl delete pod` 命令删除一个 Pod 时，这个请求首先被发送到 Kubernetes API 服务器。
    - API 服务器会将这个删除请求记录下来，并将 Pod 的状态标记为 "Terminating"。
	    - 将 Pod 的 `deletionTimestamp` 字段设为当前时间，表明这个 Pod 正在被删除。
	    - 更新 Pod 对象状态，并触发相应的事件。

2. kubelet：
	*由于 Pod 对象的变化（`deletionTimestamp` 字段被设置），调度器和与该 Pod 相关的 Kubelet 会通过 on-watch 事件获取到这个变动。*
		    
3. **执行优雅终止**：
    
    - kubelet 会向 Pod 中的容器发送一个 `SIGTERM` 信号，触发优雅终止过程。此时，应用程序有机会进行清理和保存数据。
    - kubelet 会等待一段时间（按照 `terminationGracePeriodSeconds` 设置）以给容器足够时间关闭。
4. **强制终止**：
    
    - 如果容器在优雅终止时间内没有退出，kubelet 将发送 `SIGKILL` 信号，以强制终止容器。
5. **反馈状态**：
    
    - 在容器终止后，kubelet 会将 Pod 的最终状态反馈给 API 服务器。在 API 服务器确认 Pod 已彻底删除后，会将其从集群状态中移除。  ^54a0c9

### 一个长期处于terminating的pod如何查找原因

#### 排查问题的方法
- kubectl describe pod pod_name
- kubectl logs pod-name -n namespace
- kubectl debug pod/pod-name -n namespace --image=busybox sh
- journalctl -u kubelet

#### 可能存在的原因
1. 挂载的持久卷资源未卸载
		- kubelet删除的时候会check
2. 优雅时间过长
		- 容器一个优雅的终止时间 terminationGracePeriodSeconds
		- 在此时间内，Pod 会一直处于 "Terminating" 状态
		- 容器可能需要较长时间来处理终止信号
			[[#容器可能需要长时间处理SIGTERM信号的原因]]
3. kubelet与apiserver通信问题
		[[#^54a0c9]]


#### 容器可能需要长时间处理SIGTERM信号的原因
1. **应用程序逻辑**：
    
    - 应用程序在收到 SIGTERM 信号后，需要进行较为复杂的数据清理或保存状态的操作。
    - 一些数据库或状态保存服务需要在关闭前完成事务清理或者数据同步。
2. **外部依赖**：
    
    - 容器可能依赖某些外部服务进行清理操作，这些服务的响应时间影响了容器的终止时间。
3. **资源密集型任务**：
    
    - 容器正在处理资源密集型任务（如大规模的文件处理或数据分析），这些任务需要时间完成

### 容器管理主线程
1. **PLEG (Pod Lifecycle Event Generator):**
    
    - 负责监控容器运行时的事件, 并将这些事件转换为 Pod 级别的事件。
    - 调用容器运行时接口获取本节点的容器/沙箱信息，与本地维护的pod缓存进行对比，生成对应的PodLifecycleEvent，然后通过eventChannel发送给Kubelet syncLoop，然后通过定时任务最终达到用户期望的状态。
2. **Pod Workers:**
    
    - Kubelet 会为每个 Pod 分配一个专门的 Pod Worker 线程。
    - Pod Worker 负责该 Pod 的整个生命周期管理，包括：
    - 从事件管道读取事件。
    - 根据事件类型执行相应的操作，例如创建、启动、停止、删除容器等。
    - 调用 CRI (Container Runtime Interface) 与底层容器运行时进行交互。
    - 更新 Pod 状态到 kubelet 的 Pod Cache。
3. **Status Manager:**
    
    - 负责将 Pod 状态的变更同步到 Kubernetes API Server。
    - 它会定期从 kubelet 的 Pod Cache 中读取最新的 Pod 状态信息，并将这些信息更新到 API Server。
4. **Probe Manager:**
    
    - 负责执行容器的 Liveness Probe 和 Readiness Probe。
    - 它会定期检查容器的健康状态，并在必要时采取相应的措施，例如重启容器或将 Pod 从 Service 的 Endpoint 列表中移除。