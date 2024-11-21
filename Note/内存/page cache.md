---
tags:
  - linux
  - pagecache
  - 内存
---
- [[#相关概念|相关概念]]
	- [[#相关概念#`Buffers`与`Cached`的区别|`Buffers`与`Cached`的区别]]
	- [[#相关概念#Active(file)+Inactive(file)|Active(file)+Inactive(file)]]
	- [[#相关概念#SwapCached|SwapCached]]
- [[#Page Cache 的主要产生来源|Page Cache 的主要产生来源]]

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

