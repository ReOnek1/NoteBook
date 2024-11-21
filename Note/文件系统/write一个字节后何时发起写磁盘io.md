
- 第一种情况，如果write系统调用时，如果发现PageCache中脏页占比太多，超过了dirty_ratio或dirty_bytes，write就必须等待了。
    
- 第二种情况，write写到PageCache就已经返回了。worker内核线程异步运行的时候，再次判断脏页占比，如果超过了dirty_background_ratio或dirty_background_bytes，也发起写回请求。
    
- 第三种情况，这时同样write调用已经返回了。worker内核线程异步运行的时候，虽然系统内脏页一直没有超过dirty_background_ratio或dirty_background_bytes，但是脏页在内存中呆的时间超过dirty_expire_centisecs了，也会发起回写。