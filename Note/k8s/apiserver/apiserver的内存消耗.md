### 主要内存消耗来源

1. 缓存集群中（除去event,event为k8s资源类型）所有数据，并为每种资源缓存了历史的==watchCacheEvent==
2. 客户端的请求，尤其是list请求（需要在内存中进行数据的深拷贝以及序列化的操作），list请求压力主要来自于informor，goland的GC没有办法完全回收

### list请求占用内存多的原因

1. 没有指定resourceversion，直接从etcd获取数据可能需要大量内存，超过完整响应的大小数倍。

### 常见OOM的场景

1. daemonset用到informer，在进行变更或者故障重启的时候，在集群规模大的时候很容易造成kube-apiserver的OOM,并且在OOM之后异常连接会转移到其它节点，引起雪崩
2. 某种类型资源的数据量很大，kube-apiserver 配置的 timeout 参数太小，不足以支持完成一次 list 请求的情况下，Informer 会一直不断地尝试进行 list 操作，这种情况多发生在控制面组件，因为他们往往需要获取全量数据

### too old resource version

1. 因为每个 kubelet 只关心自己节点的 Pod，如果自身节点 Pod 一直没有变化，而其他节点上的 Pod 变化频繁，则可能 kubelet 本地 Informer 记录的 last RV 就会比 cyclic buffer 中的最小的 RV 还要小，这时如果发生重连（网络闪断，或者 Informer 自身 timeout 重连）则可以在 kube-apiserver 的日志中看到 "too old resoure version" 的字样。
2. kube-apiserver 重启的场景，如果集群中部分类型资源变更频繁，部分变更不频繁，则对于去 watch 变更不频繁的资源类型的 Informer 来说起本地的 last RV 是要比最新的 RV 小甚至小很多的，在 kube-apiserver 发生重启时，他以本地这个很小的 RV 去 watch，还是有可能会触发这个问题

- **客户端 Informer 遇到这个报错的话会退出 ListAndWatch，重新开始执行 LIstAndWatch，进而造成 kube-apiserver 内存增加甚至 OOM。**
- **问题本质原因：RV 是全局的。场景的景本质区别在于场景 1 是在一种资源中做了筛选导致的，场景 2 是多种资源类型之间的 RV 差异较大导致的。**
- 如何解决：
    - cyclic buffer 长度有限 >>>>>自适应窗口大小
    - 客户端的informer持有的last RV过于陈旧 >>>>> bookmark标记