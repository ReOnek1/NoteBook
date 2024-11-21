### 主要干了哪些事情
1. 提供Kubernetes API,包括认证授权，数据校验以及集群状态变更
2. 代理集群中的一些附加组件，如kubernetes UI,metrics-server,npd（node problem detect?）
3. 创建k8s服务，apiserver service以及kubernetes service
4. 资源在不同版本之间的转换

### 一次请求的具体流程
#### 访问接口
curl -k https://masterIP:6443

#### 1.Decode
	版本适配与转换
- kubernetes 中的多数 resource 都会有一个 `internal version`
- 在解码时，首先从 HTTP path 中获取期待的 version，然后使用 scheme 以正确的 version 创建一个与之匹配的空对象，并使用 JSON 或 protobuf 解码器进行转换，
- 在转换的第一步中，如果用户省略了某些字段，Decoder 会把其设置为默认值。

#### 2.认证
	apiserver在第一次启动的时候，会根据用户提供的命令行参数组装一个合适的认证器列表
   
每一个请求都会逐一经过这个请求列表，直到有一个认证通过
- 客户端证书认证（x509）
- HTTP Basic（静态密码文件）
- Bearer Token（静态Token文件）
- webhook认证（使用外部 webhook 服务进行认证）

> [!NOTE] 
> 认证成功后，Authorization头信息将会从请求中移除，用户信息会添加到请求的上下文信息中。这样后续步骤就可以访问到认证阶段确定的请求用户的信息了。

#### 3.鉴权
	kube-apiserver需要基于用户提供的命令行参数，来组装一个合适的鉴权器列表来处理每一个请求。当所有的鉴权器都拒绝该请求时，请求会终止

通过指定--authorization-mode参数设置授权机制，至少需要指定一个
- AlwaysAllow
- AlwaysDeny
- ABAC
- Webhook
- RBAC
- Node

#### 4.Admission control
	持久化（存到etcd之前）的最后一道保障

- 变更准入控制器（Mutating Admission Controller）用于变更信息，能够修改用户提交的资源对象信息
- 验证准入控制器（Validating Admission Controller）用于身份验证，能够验证用户提交的资源对象信息

#### 5.etcd
	kube-apiserver将反序列化HTTP请求（解码），构造运行时对象（runtime object），并将它持久化到etcd。

> [!NOTE] apiserver如何找到每一个资源对应的操作Handler
>  
> - 1.API注册。当 kube-apiserver 启动时，它会加载所有已注册的资源类型和它们的处理函数。这些资源和它们的处理逻辑通常在 Kubernetes 源代码中定义，并在编译时注册到 API Server 中。
> 2.REST处理器
> - 每个资源类型都有一个对应的 REST 处理器，该处理器用于处理 HTTP 请求。REST 处理器实现了资源的创建、读取、更新、删除等（CRUD）操作的具体逻辑。


### kube-apiserver与etcd的连接
#### 相关背景
1. 与etcd连接时，每种资源类型都会存在一个etcd Client实例
2. etcd clientv3 本身已经是通过 gRPC 方式访问 etcd 
#### 问题
	在 TLS 模式下访问 Etcd 时**，在**针对同一个资源进行并行的 LIST 请求且资源量较大时，这个问题依然存在，且会影响 k8s 的所有版本
#### 问题的根源
	 golang 1.19 中入 PriorityWriteScheduler，PriorityWriteScheduler 可以支持基于流的优先级的写的能力，但是**当没有优先级的差异时**，就会退化为 LIFO 的模式，新创建的流会优先得到处理，而已经创建一段时间并且发送了很多数据的流会排在靠后的位置等待处理
1. **高读响应负载**：
    - 当一个etcd客户端产生高读响应负载时（例如并发运行多个大型Range请求），单个连接的带宽和资源可能被这些大数据量的响应占用。
    - 这种情况下，watch响应流相对较小，容易被高优先级的读响应“饿死”。
2. **TLS的影响**：
    - 启用TLS增加了额外的加密与解密的计算开销，可能会进一步加剧资源竞争，导致watch响应被延迟。
3. **单一连接的限制**：
    - 当所有操作使用同一个连接时，高负载的读请求可能占用了连接的带宽和处理能力，导致watch响应被延迟。
#### 如何解决
##### goland（版本迭代）
1. 采取了 round robin write scheduler
##### etcd （版本迭代）
2. 同时通过暴露新的参数 `--listen-client-http-urls` 来解决 TLS 模式下通过 http server 处理 grpc handler 时受到 golang 影响的问题
##### 新版本的etcd和k8s已经修复了这个问题

### client-go与kube-apiserver的连接

#### 相关背景
1. client-go通过Config结构维护访问kube-apiserver，其中最核心的就是Transport，默认是http/2
2. 未显式指定 Transport，通过 Config 创建了 clientset，使用 clientset 创建了 informerFactory，并同时针对不同的资源启动了对应的 informer(此时clientset与apiserver是一条连接，通过informerFactory去对不同资源建立stream--http的io多路复用)
#### 相关的问题
	一个恶意的 HTTP/2 客户端，迅速创建请求并立即重置它们，可能导致服务器资源的过度消耗。尽管请求的总数受限于 `http2.Server.MaxConcurrentStreams` 设置，但在处理中的请求被重置允许攻击者在现有请求仍在执行时创建新请求。

#### 如何解决
1. 在服务端添加相关逻辑，判断执行中的请求是否超过了 `MaxConcurrentStreams` 的值，没有超过的话就直接执行，超过的话则会入队列，等到运行中的某个请求完成之后再从队列中取一个请求开始处理。如果队列里面积压的未处理请求数量超过 `MaxConcurrentStreams` 的 4 倍的话，服务端会报错 too_many_early_resets 并返回  `ErrCodeEnhanceYourCalm`，然后强制关闭连接。
2. 就是超过了限制就将新的连接放入队列中，队列中的数量积压如果超过了MaxConcurrentStreams的4倍就强制关闭连接
3. 对于apiserver来说，`MaxConcurrentStreams` **从 250 缩小到 100**，是为了降低服务端可能受到的影响，在受到攻击时关闭连接

### apiserver 的内存消耗
#### 主要内存消耗来源
1. 缓存集群中（除去event,event为k8s资源类型）所有数据，并为每种资源缓存了历史的==watchCacheEvent==
2. 客户端的请求，尤其是list请求（需要在内存中进行数据的深拷贝以及序列化的操作），list请求压力主要来自于informor，goland的GC没有办法完全回收

#### list请求占用内存多的原因
1. 没有指定resourceversion，直接从etcd获取数据可能需要大量内存，超过完整响应的大小数倍。

#### 常见OOM的场景
1. daemonset用到informer，在进行变更或者故障重启的时候，在集群规模大的时候很容易造成kube-apiserver的OOM,并且在OOM之后异常连接会转移到其它节点，引起雪崩
2. 某种类型资源的数据量很大，kube-apiserver 配置的 timeout 参数太小，不足以支持完成一次 list 请求的情况下，Informer 会一直不断地尝试进行 list 操作，这种情况多发生在控制面组件，因为他们往往需要获取全量数据
#### **too old resource version
	客户端调用watch api时携带非0的resourceversion,服务端在cacher的watch是需要根据传入的resourceversion去cyclic buffer中二分查找大于resourceversion的所有watchCacheEvent作为初始event返回给客户端
	