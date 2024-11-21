#### GMP
	将Goroutine分配、负载、调度到处理器上
#### G-M-P分别代表：

- G - Goroutine，Go协程，是参与调度与执行的最小单位
- M - Machine，指的是系统级线程
- P - Processor，指的是逻辑处理器，P关联了的本地可运行G的队列(也称为LRQ)，最多可存放256个G

#### 调度流程

![[GMP调度流程.png]]

1. 每个P有个局部队列，`局部队列`保存待执行的goroutine(流程2)，
2. 当M绑定的P的的局部队列已经满了之后就会把goroutine放到`全局队列`(流程2-1)
3. `每个P和一个M绑定`，M是真正的执行P中goroutine的实体(流程3)，M从绑定的P中的局部队列获取G来执行
4. 当M绑定的P的局部队列为空时，M会从全局队列获取到本地队列来执行G(流程3.1)，
5. 当从全局队列中没有获取到可执行的G时候，M会从其他P的局部队列中偷取G来执行(流程3.2)，这种从其他P偷的方式称为`work stealing`
6. 当G因系统调用(syscall)阻塞时会阻塞M，此时P会和M解绑即`hand off`，并寻找新的idle的M，若没有idle的M就会新建一个M(流程5.1)。
7. 当G因channel或者network I/O阻塞时，不会阻塞M，M会寻找其他runnable的G；当阻塞的G恢复后会重新进入runnable进入P队列等待执行(流程5.3)

#### 调度过程中阻塞

- I/O，select
- block on syscall
- channel
- 等待锁
- runtime.Gosched()


> [!NOTE] 关于阻塞
> - 用户态阻塞不会导致M阻塞，仅阻塞G
> 	- 对应的G会被放置到某个wait队列(如channel的waitq)，该G的状态由_Gruning变为_Gwaitting，而M会跳过该G尝试获取并执行下一个G，
> 	- 如果此时没有runnable的G供M运行，那么M将解绑P，并进入sleep状态；
> 	- 当阻塞的G被另一端的G2唤醒时（比如channel的可读/写通知），G被标记为runnable，尝试加入G2所在P的runnext，然后再是P的Local队列和Global队列。
> - 系统调用阻塞会导致M阻塞，此时M可被抢占
> 	- 执行该G的M会与P解绑，而P则尝试与其它idle的M绑定，继续执行其它G。
> 	- 如果没有其它idle的M，但P的Local队列中仍然有G需要执行，则创建一个新的M；
> 	- 当系统调用完成后，G会重新尝试获取一个idle的P进入它的Local队列恢复执行，如果没有idle的P，G会被标记为runnable加入到Global队列。

#### M什么时候会和P解绑
1. 系统调用阻塞，M可被抢占
2. P中没有可runnable的G供M运行，此时M将解绑P并进入sleep状态