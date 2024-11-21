## 问题

1. informer和sharedInformer的区别，share的内容是什么
    
    - informer只能注册单个 **ResourceEventHandler**
    - sharedInformer可以注册多个ResourceEventHandler
    - sharedInformer可以在一个事件来了之后去触发多个动作或者有多个观察者的
    - **shared 的是 Reflector**；
2. 什么场景需要开ReSync
    
    - **需要与外部（非 k8s）的组件或者基础设施交互时使用 SharedInformer**
    - SharedInformer 同步是将 Store 里面存储的对象重新放回 Queue 中，重新触发一次全量通知，**Resync 并不会触发重新去 kube-apiserver 获取数据**；
    - Informer 的 ReSync 不会执行任何操作，直接返回
    
3. HasSynced 返回 true，代表什么意思
    
    - 对于 Informer，HasSynced 返回 true 代表其所注册的 event handler 已经执行完了，
    - 对于SharedInformer，则代表 **Queue（Reflector Store）里面的数据已经全部 Pop 出来，经过处理将对应的 Indexer 里面的数据分发给所有的 processorListener 了，但是否注册的所有的 event handler 已经都处理完了并不一定；**
4. 使用 NamespacedName 作为 key，有没有问题？
    
	    一般索引都用 ID 表示
    ![[informer1.png]]
## client-go组件

- **Reflector**
    
    - 通过`ListAndWatch`的方法，watch可以是k8s内建的资源或者是自定义的资源
    - 通过watch API接收到有关新资源实例存在的通知时，它使用相应的列表API获取新创建的对象，并将其放入watchHandler函数内的Delta Fifo队列中
- **Informer**
    
    - informer从Delta Fifo队列中弹出对象。执行此操作的功能是processLoop
    - 根据事件类型对本地缓存进行更新
    - 调用注册的事件处理函数来处理这些变化

## 设计思路

### 关键点

为了更快的返回list/get请求的结果，减少对kubernetes API的直接调用，Informer被设计为:

- 依赖kubernetes List/Watch API
    - informer 只会调用 Kubernetes List 和 Watch 两种类型的 API
    - 只在初始化时，调用list获取某种resource的全部Object缓存在内存
    - 随后调用Watch api去watch resource变化
- 可监听事件并触发回调函数
- 二级缓存工具包

### 更快的返回List/Get请求

使用 Informer 实例的 Lister() 方法， List/Get Kubernetes 中的 Object 时，Informer 不会去请求 Kubernetes API，而是**直接查找缓存在本地内存中的数据**(这份数据由 Informer 自己维护)。通过这种方式，Informer 既可以更快地返回结果，又能减少对 Kubernetes API 的直接调用。

### 流程解析

1. informer在初始化时，reflector首先会list api获取所有资源
    
2. reflector在拿到全部资源后然后全部放到Store中
    
    1. 通过Lister()的List/Get方法获取资源会直接从Store中拿数据

![[Pasted image 20241031103911.png]]
    
3. 初始化完成之后，reflector会调用k8s的watch api监听资源的所有事件
    
4. watch监听到的事件会被发送到DeltaFIFO
    
    1. DeltaFIFO将事件存储到自己的数据结构(queue)
    2. 接着会操作（更删改）Store的数据
    3. 最后Pop事件到Controller中
5. Controller接受事件再触发Processor的回调函数
    ![[Pasted image 20241031103924.png]]
