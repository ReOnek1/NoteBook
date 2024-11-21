![[kubelet-cgroup.png]]

kubelet 会在 node 上创建了 4 个 cgroup 层级，从 node 的**root cgroup**（一般都是**/sys/fs/cgroup**）往下：

1. **Node 级别**：针对 SystemReserved、KubeReserved 和 k8s pods 分别创建的三个Node-level cgroup；
2. **QoS 级别**：在`kubepods`cgroup 里面，又针对三种 pod QoS （guaranteed、brustable、besteffort）分别创建一个 sub-cgroup：
3. **Pod 级别**：每个 pod 创建一个 cgroup，用来限制这个 pod 使用的总资源量；
4. **Container 级别**：在 pod cgroup 内部，限制单个 container 的资源使用量。

（1）**Container 级别 cgroup**：在创建 pod 时，可以直接在 container 级别设置 requests/limits，kubelet 在这里做的事情很简单：创建 container 时，将 spec 中指定 requests/limits传给 docker/containerd 等 container runtime。换句话说，底层能力都是 container runtime 提供的，k8s 只是通过接口把 requests/limits 传给了底层。

（2）**Pod 级别 cgroup**：这种级别的 cgroup 是针对单个 pod 设置资源限额的，因为：

1. 某些资源是这个 pod 的所有 container 共享的；
2. 每个 pod 也有自己的一些开销，例如 sandbox container；
3. Pod 级别还有一些内存等额外开销；

所以pod 的 requests/limits 并不是是对它的 containers 简单累加得到，为了防止一个 pod 的多个容器使用资源超标，k8s 引入了 引入了 pod-level cgroup，每个 pod 都有自己的 cgroup。

（3）**QoS 级别 cgroup**：

实际的业务场景需要我们能根据优先级高低区分几种 pod。例如，

- 高优先级 pod：无论何时，都应该首先保证这种 pod 的资源使用量；
- 低优先级 pod：资源充足时允许运行，资源紧张时优先把这种 pod 赶走，释放出的资源分给中高优先级 pod；
- 中优先级 pod：介于高低优先级之间，看实际的业务场景和需求。

如果设置了 kubelet--cgroups-per-qos=true参数（默认为 true）， 就会将所有 pod 分成三种 QoS，优先级从高到低：Guaranteed > Burstable > BestEffort。三种 QoS 是根据`requests/limits`的大小关系来定义的：

1. Guaranteed:requests == limits, requests != 0， 即`正常需求 == 最大需求`，换言之 spec 要求的资源量必须得到保证，少一点都不行；
2. Burstable:requests < limits, requests != 0， 即`正常需求 < 最大需求`，资源使用量可以有一定弹性空间；
3. BestEffort:request == limits == 0， 创建 pod 时不指定 requests/limits就等同于设置为 0，kubelet 对这种 pod 将尽力而为；有好处也有坏处：

（4）**Node 级别 cgroup**：

所有的 k8s pod 都会落入`kubepods`cgroup；因此所有 k8s pods 占用的资源都已经能够通过 cgroup 来控制，剩下的就是那些 k8s 组件自身和操作系统基础服务所占用的资源了，即`KubeReserved`和`SystemReserved`。k8s 无法管理这两种服务的资源分配，但能管理它们的限额：有足够权限给它们创建并设置 cgroup 就行了。但是否会这样做需要看 kubelet 配置，

- `--kube-reserved-cgroup=""`
- `--system-reserved-cgroup=""`

默认为空，表示不创建，也就是系统组件和 pod 之间并没有严格隔离