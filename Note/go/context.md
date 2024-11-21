#### 并发控制包

- 用于请求超时或者取消，相关goroutine马上退出释放资源
    
- 在多个goroutine或者多个处理函数之间传递共享信息
    

#### context接口类型

- emptyCtx
    
    - Backgroud()
        
    - TODO()
        
- cancelCtx
    
    - 取消操作，同时取消实现了canceler接口的子代
        
        - WithCancel()进行创建
            
- timerCtx
    
    - 基于cancelCtx实现，支持过期取消
        
    - WithDeadline()进行创建
        
- valueCtx
    
    - 可以传递共享信息的context
        
    - WithValue()进行创建