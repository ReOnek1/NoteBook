### 设计目的：

1. ip全局唯一且可路由：为集群中的所有节点重新规划ip地址的使用规则，使得集群中的不同节点主机创建的容器都具有全集群唯一性且可路由的ip地址
    
2. ip可用性：etcd分布式协同功能（每个节点上的flaaneld进程会list可用的ip网段）
    

#### UDP模式：
 
容器A发出ICMP请求-->容器A内的route-->cni网桥-->节点上的route-->flannel0接口(tun设备)-->flanneld进程（用户态）--> eth0发出

#### VXLAN模式：

容器A发出ICMP请求-->容器A内的route-->cni网桥-->节点上的route-->flannel.1接口（vtep设备）--> 封包（

- 通过etcd得知目的地址属于哪个节点
    
- 再在当前节点的转发表FIB中找到对应节点的VTEP的MAC
    
- 最后根据（VNI, local, IP, Port）进行vxlan封包
    

）--> 底层网络发出