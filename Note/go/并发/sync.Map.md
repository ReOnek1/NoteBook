```
type Map struct {  
  mu  Mutex // 排他锁，用于对dirty map操作时候加锁处理  
​  
  read atomic.Value // read map  
​  
  // dirty map。新增key时候，只写入dirty map中，需要使用mu  
  dirty map[interface{}]*entry  
​  
  // 用来记录从read map中读取key时miss的次数  
  misses int  
}
```
1. 适合多读少写，用空间换时间
    
2. 底层两个map，优先从read map中读取，更新，删除，无需加锁
    
3. 新增key-value是加锁然后加入到dirty-map中
    
4. 读取/更新/删除：当read-map中不存在但是dirty-map中存在的时候，会将dirty-map升级到read-map，这个过程也会加锁
    
5. 延迟删除：删除时只是加上标记，当dirty-map到read-map时才会清理数据
    

##### 多写少读（concurrent-map）

1. 使用分片锁，跟据key进行hash处理后，找到其对应读写锁，然后进行锁定处理。
    
2. 通过分片锁机制，可以降低锁的粒度来实现读少写多情况下高并发
    
3. 不同线程可以同时获取不同分片的锁，从而提高并发性能。
    

具体实现：

1. 分片锁：make([]*sync.RWMutex, ShardCount)
    
2. 通过将key进行hash得到分片锁的id： int(h.Sum32()) % ShardCount
    

注意事项：

1. 分片数量的选择（高效的hash函数CityHash:快速散列表，一致性哈希）
    
2. 数据均匀分布（预分区，数据预处理：先排序分组再处理）