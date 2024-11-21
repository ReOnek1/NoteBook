### 相关背景

1. client-go通过Config结构维护访问kube-apiserver，其中最核心的就是Transport，默认是http/2
2. 未显式指定 Transport，通过 Config 创建了 clientset，使用 clientset 创建了 informerFactory，并同时针对不同的资源启动了对应的 informer
	1. 此时clientset与apiserver是一条连接，通过informerFactory去对不同资源建立stream--http的io多路复用

### 相关的问题

```
一个恶意的 HTTP/2 客户端，迅速创建请求并立即重置它们，可能导致服务器资源的过度消耗。尽管请求的总数受限于 `http2.Server.MaxConcurrentStreams` 设置，但在处理中的请求被重置允许攻击者在现有请求仍在执行时创建新请求。

```

### 如何解决

1. 在服务端添加相关逻辑，判断执行中的请求是否超过了 `MaxConcurrentStreams` 的值，没有超过的话就直接执行，超过的话则会入队列，等到运行中的某个请求完成之后再从队列中取一个请求开始处理。如果队列里面积压的未处理请求数量超过 `MaxConcurrentStreams` 的 4 倍的话，服务端会报错 too_many_early_resets 并返回 `ErrCodeEnhanceYourCalm`，然后强制关闭连接。
2. 就是超过了限制就将新的连接放入队列中，队列中的数量积压如果超过了MaxConcurrentStreams的4倍就强制关闭连接
3. 对于apiserver来说，`MaxConcurrentStreams` **从 250 缩小到 100**，是为了降低服务端可能受到的影响，在受到攻击时关闭连接