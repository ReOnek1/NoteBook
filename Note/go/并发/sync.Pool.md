#### Sync.Pool

- 减少对象内存分配和GC压力
    
- 两个对象存储容器：
    
    - local pool
        
    - victim pool (cache)
	    - 作为缓冲区，在对象从某个处理器 (P) 被清理时，暂时存储这些对象以备其他处理器复用
    
##### 清除对象

2个GC循环清除缓存池中的对象（对象回收时会先放入local pool）

1. 当GC开始时，先将victim cache中的对象清除，
    
2. 将local pool中的对象移动到victim cache中
    
3. func (p *Pool) Put(x interface{})
    

##### 获取对象

1. 优先从`local pool`中查找，若未找到则再从`victim cache`中查找，
    
2. 若也未获取到，则调用New方法创建一个对象返回。
    
3. func (p *Pool) Get() interface{}
    

##### 使用方法

1. 先通过obj=sync.Pool.New一个临时对象
    
2. 再用obj.Get()从缓冲池中取出对象
    
3. 用obj.Put()将对象放回缓冲池
    

##### 底层数据结构

type Pool struct {  
  noCopy noCopy // nocopy机制，用于go vet命令检查是否复制后使用  
​  
  local     unsafe.Pointer // 指向[P]poolLocal数组，P等于runtime.GOMAXPROCS(0)  
  localSize uintptr        // local数组大小，即[P]poolLocal大小  
​  
  victim     unsafe.Pointer // 指向上一个gc循环前的local  
  victimSize uintptr        // victim数组大小  
​  
  New func() interface{} // 创建临时对象的方法，当从local数组和victim数组中都没有找到临时对象缓存，那么会调用此方法现场创建一个  
}

- local指向runtime.GOMAXPROCS(0)
    
- 每一个P都通过其ID关联一个poolLocal，与P关联的M和G可以无锁的操作这个poolLocal
    

##### Get()操作

- 调用pin方法，获取当前G关联的P对应的poolLocal和该P的ID
    
- 接着查看poolLocal的private对象是否存放了对象，如果存放则直接返回（快速路径）
    
- 如果没有则从poolLocal的双端队列取出对象，此操作是lock-free的
    
- 若G关联的per-P级poolLocal的双端队列中没有取出来对象，那么就尝试从其他P关联的poolLocal中偷一个。若从其他P关联的poolLocal没有偷到一个，那么就尝试从victim cache中取。
    
- 若也没没有取到缓存对象，那么只能调用pool.New方法新创建一个对象。
    

##### Put()操作

1. 调用pin方法，返回当前P的localPool
    
2. 若当前P的localPool的private属性没有存放对象，那就存放其上面，这是最快路径，取的时候优先从private上面取
    
3. 若当前P的localPool的private属性已经存放了归还的对象，那么就将对象入队存储。
    

##### 双端队列

pool --> local --> [P]poolLocal --> private

pool --> local --> [P]poolLocal --> poolChain --> poolDequeue(双端队列)

type poolDequeue struct {  
  headTail uint64  
  vals []eface  
}  
​  
type eface struct {  
  typ, val unsafe.Pointer  
}  
​  
type dequeueNil *struct{}

- 无锁的
    
- 固定大小的单一生产者，多消费者
    
- 具有队列和栈性质
    
- 两端都可以删除，但是插入只能在head端