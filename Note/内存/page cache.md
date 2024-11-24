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

- ==Buffered I/O（标准I/O）==
	- 需要先写到用户缓冲区,然后再复制到page cache
  
- ==Memory-Mapped I/O（存储映射I/O）==
	- 直接映射到page cache,用户直接读写page cache

![[page-cache-diff.png]]

#### 涉及到的内核机制

1. 用户往用户缓冲区写数据（userspace buffer）
2. 用户缓冲区copy到内核缓冲区，
	1. 发生[[#缺页中断]]  ==〉分配Page
	2. 将数据copy到内核缓冲区



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
### 进程创建的过程中，如何申请pid
‌‌‌‌　　在进程创建(fork)的过程中，有一系列的copy过程[源码](https://elixir.bootlin.com/linux/v6.10/source/kernel/fork.c#L2375)
‌‌‌‌　　其中对于pid的申请主要有两个注意点：
	- 只要是申请pid失败都是返回“内存无法分配”的错误（ENOMEM）
		- pid不够了
		- 申请失败
	- 会通过for循环同时申请多个pid（一个是在容器ns里面的pid，另外一个是在根目录下的）[源码](https://elixir.bootlin.com/linux/v6.10/source/kernel/pid.c#L166)

