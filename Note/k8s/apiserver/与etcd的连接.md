### 相关背景

1. 与etcd连接时，每种资源类型都会存在一个etcd Client实例
2. etcd clientv3 本身已经是通过 gRPC 方式访问 etcd

### 问题

```
在 TLS 模式下访问 Etcd 时，在针对同一个资源进行并行的 LIST 请求且资源量较大时，这个问题依然存在，且会影响 k8s 的所有版本
```

### 问题的根源

```
 golang 1.19 中入 PriorityWriteScheduler，PriorityWriteScheduler 可以支持基于流的优先级的写的能力，但是当没有优先级的差异时，就会退化为 LIFO 的模式，新创建的流会优先得到处理，而已经创建一段时间并且发送了很多数据的流会排在靠后的位置等待处理
```

1. **高读响应负载**：
    - 当一个etcd客户端产生高读响应负载时（例如并发运行多个大型Range请求），单个连接的带宽和资源可能被这些大数据量的响应占用。
    - 这种情况下，watch响应流相对较小，容易被高优先级的读响应“饿死”。
2. **TLS的影响**：
    - 启用TLS增加了额外的加密与解密的计算开销，可能会进一步加剧资源竞争，导致watch响应被延迟。
3. **单一连接的限制**：
    - 当所有操作使用同一个连接时，高负载的读请求可能占用了连接的带宽和处理能力，导致watch响应被延迟。

### 如何解决

### goland（版本迭代）

1. 采取了 round robin write scheduler

### etcd （版本迭代）

1. 同时通过暴露新的参数 `-listen-client-http-urls` 来解决 TLS 模式下通过 http server 处理 grpc handler 时受到 golang 影响的问题

### 新版本的etcd和k8s已经修复了这个问题