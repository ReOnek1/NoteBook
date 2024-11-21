##### 简介

1. `channel` 是一种用于 goroutine 之间通信的机制。
2. 它提供了一种类型安全的方式来在不同的 goroutine 之间传递消息，并且可以在某种程度上替代共享内存和锁的使用。
3. `channel` 是一种同步数据结构，可以保证发送和接收操作的同步完成。

#### 循环队列的队满和队空的二义性问题
    
1. 用一个空白单元来解决
	
2. goland的channel通过添加一个qcount来记录
        
#### channel规则
    
1. Close(ch)
	
	1. 关闭空chan和已关闭的chan都会panic
		
2. 从chan中取值
	
	1. 空的chan会永远阻塞
		
	2. 已经关闭的chan不会阻塞
		
3. 往chan中送值
	
	1. 空的channel会永远阻塞
		
	2. 关闭的channel会panic
            
#### 发送流程（val --> chan）
    
1. 快速路径 - 非阻塞发送:
	
	- 尝试获取 channel 的 `lock`，如果获取不到，说明其他 goroutine 正在操作 channel，则进入慢速路径。
		
	- 如果 channel 为空 (`qcount == 0`) 且接收队列 `recvq` 不为空，说明有等待接收数据的 goroutine，则直接将数据发送给该 goroutine，这是一种优化策略，避免数据入队和出队的开销，也称为 "handoff"。
		
	- 如果 channel 未满 (`qcount < dataqsiz`)，则将数据拷贝到循环队列 `buf` 中，增加 `qcount`，唤醒一个阻塞在发送队列 `sendq` 的 goroutine（如果有）。
		
	- 释放 `lock`。
		
2. 慢速路径 - 阻塞发送:
	
	- 加锁：获取 channel 的 `lock`。
		
	- 判断 channel 是否已关闭：如果已关闭，则 panic，抛出 `send on closed channel` 错误。
		
	- 判断 channel 是否已满：如果已满，则将当前 goroutine 封装成 `sudog` 结构体，并将其加入到发送队列 `sendq` 中，然后将当前 goroutine 阻塞，等待被唤醒。
		
	- 判断 channel 是否为空且接收队列不为空：如果满足条件，则直接将数据发送给 `recvq` 中的第一个 goroutine，无需入队。
		
	- 将数据拷贝到循环队列 `buf` 中，增加 `qcount`。
		
	- 唤醒一个阻塞在接收队列 `recvq` 中的 goroutine（如果有）。
		
	- 释放 `lock`。
		

**一些注意点**

- `lock`: channel 使用互斥锁 `lock` 保证并发安全，任何发送和接收操作都需要先获取锁。
	
- `recvq` 和 `sendq`: channel 使用两个队列 `recvq` 和 `sendq` 分别存储阻塞的接收者和发送者，队列元素类型为 `sudog`，`sudog` 中保存了 goroutine 的相关信息。
	
- `panic`: 向已关闭的 channel 发送数据会引发 panic，因为这通常是一个编程错误，需要开发者排查代码逻辑。
	
	 
#### 接收流程 (val <-- chan)
    
1. 快速路径 - 非阻塞接收:
	
	- 尝试获取 channel 的 `lock`，如果获取不到，说明其他 goroutine 正在操作 channel，则进入慢速路径。
		
	- 如果 channel 不为空 (`qcount > 0`)，则从循环队列 `buf` 中读取数据，减少 `qcount`，唤醒一个阻塞在发送队列 `sendq` 的 goroutine（如果有）。
		
	- 如果 channel 为空 (qcount == 0）sendq不为空，说明有等待发送数据的 goroutine，则直接从sendq中取出第一个 goroutine，并根据 channel 是否为缓冲通道进行处理：
		
		- **缓冲通道:** 从发送者 goroutine 复制数据到 channel 的缓冲区 `buf` 中，增加 `qcount`，然后从 `buf` 中读取数据返回给接收者。
			
		- **非缓冲通道:** 直接将发送者 goroutine 的数据传递给接收者 goroutine，无需经过 channel 缓冲区，完成 "handoff" 过程。
			
	- 释放 `lock`。
		
2. 慢速路径 - 阻塞接收:
	
	- 加锁：获取 channel 的 `lock`。
		
	- 判断 channel 是否已关闭：如果已关闭，且缓冲区为空 (`qcount == 0`)，则直接返回默认值（零值），如果缓冲区不为空，则从缓冲区读取数据，直到缓冲区为空。
		
	- 判断 channel 是否为空：如果为空，则将当前 goroutine 封装成 `sudog` 结构体，并将其加入到接收队列 `recvq` 中，然后将当前 goroutine 阻塞，等待被唤醒。
		
	- 判断发送队列 `sendq` 是否为空：如果不为空，则直接从 `sendq` 中取出第一个 goroutine，并根据 channel 是否为缓冲通道进行处理（处理逻辑同快速路径）。
		
	- 从循环队列 `buf` 中读取数据，减少 `qcount`。
		
	- 唤醒一个阻塞在发送队列 `sendq` 中的 goroutine（如果有）。
		
	- 释放 `lock`。
		
	
	**一些注意点**
	
	- 关闭 channel 后，仍然可以从 channel 接收数据，直到 channel 中的数据全部被读取完毕，之后再接收将会返回零值。
		
	- 非缓冲通道的发送和接收操作会同步阻塞，直到另一方准备好进行数据传输，因此非缓冲通道也称为同步通道。
		
	- 缓冲通道的发送操作只有在缓冲区满时才会阻塞，接收操作只有在缓冲区为空时才会阻塞。


#### 简化过程
        
1. 先获取lock,没获取到则阻塞

2. 判断是否关闭，关闭的时候再发送数据则会Panic

3. 判断是否为空，如果为空（qcount=0）并且接受者队列/发送者队列不为空,则直接将数据复制到对应的goroutine，不需要经过buf

4. 如果不为空

5. 对于阻塞场景，则将当前的goroutine封装成sudog结构体，并加入接受队列/发送队列中，然后阻塞goroutine,等待被唤醒
	
2. 对于非阻塞场景，则将数据拷贝到循环队列 `buf` 中，增加/减少 `qcount`，唤醒一个阻塞在发送队列 `sendq` /recvq的 goroutine（如果有）