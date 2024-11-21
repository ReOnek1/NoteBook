### kubelet 熔断机制

##### 背景
	跨大版本升级可能带来 Kubernetes API 版本的变化，如果控制平面（API Server 和 Controller Manager等）已经升级到新的版本，而 Kubelet 尚未完成升级，Kubelet 可能会因为不兼容的 API 版本无法正确同步资源列表，可能会表现为删除旧的pod，或者无法调度/管理新的pod

##### 解决方案
1. 逐步升级
2. 升级过程的协调
		确保 API Server 和 Kubelet 的版本兼容，先升级控制平面组件，然后逐步升级节点上的 Kubelet，kube-proxy
3. 使用熔断机制
	1. 基于监控告警触发
	2. 通过ansible放置/etc/shein_kubelet_stop的占位文件
	3. 在每个节点上运行定时脚本去检查是否存在占位文件
	4. 如果存在则通过 `kubectl get pods --all-namespaces -o json | jq -r 
	5. 停止所有调度

	- 另外一个方案是直接修改kubelet的代码
	- 在NewMainKubelet这边添加一个go func去check 占位文件
	- 如果存在则将标识设立为true，并在killpod这个函数的时候去进行判断

5. 监控以及验证

### 大规模apiserver优化

##### 背景

1. 内存消耗来源
	1. apiserver在watchCache中缓存了集群所有云数据，并且为每种资源缓存了历史watchCacheEvent
	2. 一些controller在没有指定resourceVersion的情况发起list请求，会导致apiServer在内存中进行大量的数据深拷贝，这些深拷贝的数据无法被直接GC
		1. `resourceVersion` 是 Kubernetes API 对象的一个字段，用于标识对象的版本
		2. 当 Controller 发起 List 请求时，如果指定了 `resourceVersion`，API Server 可以根据该版本号判断是否有数据更新
		3. **如果没有数据更新，API Server 可以直接返回缓存中的数据，避免进行深拷贝操作**
2. 实际场景
	1. 在 daemonset 中使用 informer ，或故障重启节点时，请求量突增，且在apiServer一个副本挂了后，异常连接传递到其他副本，导致 OOM 雪崩。
	2. 当请求的资源数据量过大时， kube-apiserver 配置的默认timeout（1分钟）不足以支撑一次资源数据量返回，infromer不断尝试 list ，导致OOM。
##### 解决方案

1. 优化list/watch请求
    1. 尽可能的使用FieldSelector/LabelSelector，减少apiserver的压力
    2. 设置合理的resourceVersion参数，避免获取重复的已经处理过的数据
    3. 限制请求频率，设置合适的timeoutSeconds控制持续请求的时间
    4. 使用watch bookmarks: 保证在重新连接后不会错过任何事件
        1. 服务端和客户端协同维护同一个同步点（bookmark）-- AllowWatchBookmarks: true
        2. 客户端发送watch请求
        3. 服务端在返回的watch事件流会定期发送特殊的bookmark事件流
            
2. 使用informer机制
    1. 利用客户端的缓存，避免重复请求apiserver
    2. 事件处理机制，避免频繁轮询apiserver
        
3. 调整apiserver参数
    1. 增加apiserver资源限制：--max-requests-inflight控制并发量的请求数量
    2. 启用api优先级和公平性：确保关键组件的请求优先得到处理
4. 在apiserver前套用限流
	1. nginx
	2. istio
5. 流式api,从listWatch到WatchList
