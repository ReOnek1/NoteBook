#### Mutex

1. 组成
    
    type Mutex struct {  
      state int32        // 表示互斥锁状态 一个32bit == waitercounter | starving | woken |        locked  
      sema  uint32       // 控制锁状态的信号量  
    }
    
2. 两种模式
    1. 正常模式：等待者会先进先出的获取锁，刚被唤醒的锁和新创建的goroutine竞争大概率是获取不到
        
    2. 饥饿模式：
        1. 一旦有goroutine超过1ms没有获取到锁，就会切换成饥饿模式，防止部分goroutine被饿死
            
        2. 互斥锁会直接交给最前面的goroutine,新的goroutine此时不能直接获取锁并且也不会自旋，只能在队尾等待
            
        3. 如果队尾的的goroutine获取了锁获取等待时间少于1ms，切回正常模式
            
3. 加锁和解锁
    
    加锁
    
    1. 判断当前goroutine是否进入自旋
        
        1. 自旋：多线程同步机制，进入自旋会一直保持cpu的占用，持续检查某个条件是否为真
			    - 自旋锁会持续占有cpu时间，直到获取锁
			    - 适合短时间内的上下文切换
			    - 无法获取锁，当前goroutine会进入忙等(busy-waiting)状态
            
        2. 多核cpu可以避免goroutine的切换
            
    2. 通过自旋等待锁的释放
        
    3. 计算互斥锁的最新状态
        
    4. 更新状态并获取锁
        
    
    解锁
    
    1. 直接调用sync/atomic.AddInt32快速解锁
        
    2. 如果返回值非0，则调用unlockSlow开始慢速解锁
        
        1. 校验锁状态的合法性
            
        2. 饥饿状态下直接给下个等待的goroutine锁
            
        3. 正常情况下唤醒其它goroutine，如果没有等待就直接返回