---
tags:
  - linux
  - pagecache
  - 内存
---


![[page cache.png]]
```
cat /proc/meminfo
```
![[pagecache-meminfo.png]]

```
Buffers + Cached + SwapCached = Active(file) + Inactive(file) + Shmem + SwapCached
```
	两边都是page cache的组成，左边的Buffers更偏向于内核，右边为更具体的

### 相关概念
#### `Buffers`与`Cached`的区别

- **Buffers**：主要用于块设备的缓存。`Buffers` 缓存的是用于原始块访问的数据，而不是文件系统本身的数据。它代表的是==文件系统元数据==（例如：超级块、目录项等）和==直接块设备操作数据。==主要用于优化块设备的原始IO操作，如磁盘的读写。缓存的是低级别的块设备数据，不涉及具体文件的内容。
    
- **Cached**：主要用于文件系统的缓存。`Cached` 缓存的是文件系统中的文件数据和目录内容。它缓存了文件的内容，从而加速文件的读取和写入操作。

#### Active(file)+Inactive(file)

- **Active(file)**: 活跃文件页缓存，表示最近被访问过的文件页缓存。这些缓存页是“热的”，意味着它们频繁被使用，因此内存管理系统更倾向于把这些数据保存在内存中，以便快速访问。
    
- **Inactive(file)**: 不活跃文件页缓存，表示较长时间未被访问的文件页缓存。这些缓存页是“冷的”，意味着它们使用频率较低。当系统内存压力较大时，这些缓存页更有可能被释放或换出到交换空间。

> [!NOTE]
> - Active(file) + Inactive(file)代表所有用于缓存文件内容的内存页面总和，不包含共享内存shmem以及Buffers（存块设备的buffer io）
>   
> - 平时用的==mmap()内存映射方式和buffered I/O==(指的是常规的buffered io，非块设备的buffer io)来消耗的内存就属于这部分

#### SwapCached
![[swapswapcache.png]]
	SwapCached是在打开了Swap分区后，把Inactive(anon)+Active(anon)这两项里的匿名页给交换到磁盘（swap out），然后再读入到内存（swap in）后分配的内存。**由于读入到内存后原来的Swap File还在，所以SwapCached也可以认为是File-backed page，即属于Page Cache。**

### Page Cache 的主要产生来源

- Buffered I/O（标准I/O）
	- 需要先写到用户缓冲区,然后再复制到page cache
  
- Memory-Mapped I/O（存储映射I/O）
	- 直接映射到page cache,用户直接读写page cache

![[page-cache-diff.png]]

#### 涉及到的内核机制

1. 用户往用户缓冲区写数据（userspace buffer）
2. 用户缓冲区copy到内核缓冲区，
	1. 发生[[#缺页中断]]  >>> **分配Page**
	2. 将数据copy到内核缓冲区(page cache) >>> **Dirty Page**
	3. 将脏页同步到磁盘(==脏页回写==) >>> **Clean Page**
#### Clean Page 和 Dirty Page

1. 对于读操作产生的page cache，内容和磁盘内容一致，就是clean page
   
2. 修改了page cache内容，和磁盘中的不一致，就是dirty page

```
cat /proc/vmstat | egrep "dirty|writeback" 
nr_dirty 40   // 积压了多少脏页
nr_writeback 2 // 多少脏页正在回写到磁盘
```
![[../../pic/Pasted image 20241126102944.png]]

### Page Cache的回收机制
![[../../pic/Pasted image 20241126163121.png]]

##### 主要的回收方式：

1. **后台回收**
   内存紧张时，由kswapd守护进程回收不活跃的页
2. **直接回收**
   当内存非常紧张且kswapd无法足够快速释放内存时，进程会直接进行内存页扫描和回收 ------ <u>整个系统内所有进程</u>
##### 主要关注的指标
- pgscank/s : kswapd(**后台回收线程**)每秒扫描的page个数。
- pgscand/s: Application（**系统内的进程**）在内存申请过程中每秒直接扫描的page个数。
- pgsteal/s: 扫描的page中**每秒被回收的个数**。
- %vmeff: pgsteal/(pgscank+pgscand), 回收效率，越接近100说明系统越安全，越接近0说明系统内存压力越大。
![[../../pic/Pasted image 20241126170508.png]]

### 相关问题
#### 为什么第一次读写某个文件，Page Cache 是 Inactive 的？
当第一次读取一个文件时，其数据会被加载到内存中，并存储在 Page Cache 的 Inactive 列表中。
这个设计的目的是为了防止新加载的数据立即占据大量的活跃内存，可能导致系统之前已经缓存的数据被过早地淘汰掉。

#### Active和Inactive之间的转换

-  Inactive To Active
	  应用程序多次读写
-  Active To Inactive
	  系统内存压力增大，执行页回收机制，将长时间未访问的Active页移动到Inactive列表中

#### 系统中有哪些控制项可以影响 Inactive 与 Active Page Cache 的大小或者二者的比例

#### vm.swappiness

	这个参数控制系统将内存页交换到 swap 空间的倾向，其取值范围是 0 到 100。

- **较低的值（如 10）**：
    
    - 系统会==更倾向于尽量使用物理内存==，减少将数据页换出到 swap 空间。
    - 结果是，更多的内存页（包括 Active 页和 Inactive 页）会保留在物理内存中。
    - 这会增加 Active 页的数量，因为频繁访问的页更不易被换出到 swap。
    - ==适用于对性能要求高且内存充足的系统==。
- **较高的值（如 60 或更高）**：
    
    - 系统会更倾向于使用 swap 来换出内存页，以便于腾出更多的物理内存。
    - 结果是，一些 Inactive 页和可能的 Active 页会被换出到 swap。
    - ==适用于内存较少或希望将更多内存用于缓存的系统==。

##### `vm.vfs_cache_pressure`

	此参数控制<u>内核回收目录和 inode 缓存</u>的倾向，其取值范围是 0 到 200，默认值是 100。

- **较低的值（如 50）**：
    
    - 系统会保留更多的目录和 inode 缓存，而不急于回收。
    - 这会导致 Active 页保持较多，因为目录和 inode 缓存页会更长时间呆在内存中。
    - ==适用于文件系统操作频繁的应用==，因为缓存目录和 inode 可以提高性能。
- **较高的值（如 150）**：
    
    - 系统会更积极地回收目录和 inode 缓存页。
    - 这会增加 Inactive 页的数量，因为回收的目录和 inode 缓存页会被放入 Inactive 列表，甚至可能被释放。
    - ==适用于需要释放更多内存用于其他用途的系统==，代价是文件系统操作的性能可能略有下降。

##### `vm.dirty_background_ratio` 和 `vm.dirty_ratio`

	这些参数控制脏页何时被写回磁盘
	
- **`vm.dirty_background_ratio`** 是后台写回脏页到磁盘的比例，取值范围是 0 到 100（以总内存百分比表示）。
    
    - 当脏页占用的内存达到这个比例时，后台进程会开始将脏页写回磁盘。
    - <u>较低的值会触发更频繁的脏页写回</u>，因此减少了脏页在 Page Cache 中的停留时间，Active 页的比例可能增加。
- **`vm.dirty_ratio`** 是前台写回脏页到磁盘的比例，取值范围是 0 到 100。
    
    - 当脏页占用的内存达到这个比例时，任何生成新脏页的进程（如写操作）将被阻塞，直到一些脏页被写回磁盘。
    - <u>较低的值会促使进程更频繁地将脏页写回磁盘</u>，从而减少脏页累计。反之，更高的值允许更多的脏页累积。





### 缺页中断
	1. 根据新的 address 查找对应的 vma  
		` vma = find_vma(mm, address);`
	2. 如果找到的 vma 的开始地址比 address 小 ，那么就不调用expand_stack了，直接调用<font color=#2485E3>handle_mm_fault</font>来完成真正的内存申请 
	3. handle_mm_fault --->
			`Linux 是用四级页表来管理虚拟地址空间到物理内存之间的映射管理的。所以在实际申请物理页面之前，需要先 check 一遍需要的每一级页表项是否存在，不存在的话需要申请。`
	1. handle_pte_fault --->
		1. 文件映射缺页处理
		2. swap缺页处理
		3. 写时复制缺页处理
		4. 匿名映射页处理
	2. do_anonymous_page -->
		1. `调用 alloc_zeroed_user_highpage_movable 分配一个可移动的匿名物理页出来。在底层会调用到伙伴系统的 alloc_pages 进行实际物理页面的分配。
		2. `内核是用伙伴系统来管理所有的物理内存页的。其它模块需要物理页的时候都会调用伙伴系统对外提供的函数来申请物理内存
‌‌‌‌　　![[缺页中断.png]]

‌‌‌‌　　

> [!NOTE]
> ### 进程创建的过程中，如何申请pid
> ‌‌‌‌　　在进程创建(fork)的过程中，有一系列的copy过程[源码](https://elixir.bootlin.com/linux/v6.10/source/kernel/fork.c#L2375)
> ‌‌‌‌　　其中对于pid的申请主要有两个注意点：
> 	- 只要是申请pid失败都是返回“内存无法分配”的错误（ENOMEM）
> 		- pid不够了
> 		- 申请失败
> 	- 会通过for循环同时申请多个pid（一个是在容器ns里面的pid，另外一个是在根目录下的）[源码](https://elixir.bootlin.com/linux/v6.10/source/kernel/pid.c#L166)

