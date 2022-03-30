# MySQL随笔 - Buffer Pool

​	MySQL和大部分程序一样，会在内存和磁盘中存储数据。内存中存储数据主要是提示数据的访问速度，让热数据能更快的返回。而对于Buffer Pool来说，它还会被分为young和old两块区域，那么下面我们就来看一下它具体的实现。

## 数据分区

​	MySQL的buffer pool采用的是**链表**数据结构，并使用了**LRU算法**来淘汰旧数据。数据按照冷热会被分为两块区域，如下图所示：

![Content is described in the surrounding text.](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/images/innodb-buffer-pool-list.png)



​	当用户通过一条查询语句来查询数据时（全表查询除外），新的数据会被放在上图中Old Sublist的Head位置，而溢出的Old Sublist的尾部将被淘汰。当它再次被访问时，Old Sublist的内存页会被上升到New Sublist里。通过这样的机制，buffer pool保证里在内存里的数据都是最热的数据。也不难的出结论：当MySQL的buffer pool增加到一定程度时，它其实本质上就会变为一个缓存。

## Buffer Pool监控

​	buffer pool可以通过如下方式来查看其状态：

```sql
show engine innodb status;

----------------------
BUFFER POOL AND MEMORY
----------------------
Total memory allocated 140509184; in additional pool allocated 0 # 分配给buffer pool的总共大小（bytes）
Total memory allocated by read views 1224
Internal hash tables (constant factor + variable factor)
    Adaptive hash index 2299488 	(2213368 + 86120)
    Page hash           139112 (buffer pool 0 only)
    Dictionary cache    795176 	(554768 + 240408)
    File system         880072 	(812272 + 67800)
    Lock system         333952 	(332872 + 1080)
    Recovery system     0 	(0 + 0)
Dictionary memory allocated 240408 # innodb字典大小（bytes）
Buffer pool size        8191 # buffer pool分配到的页数量
Buffer pool size, bytes 134201344 # buffer pool的大小为128M
Free buffers            1024 # buffer pool剩余的空闲的页数量
Database pages          7162 # LRU list使用的页数量
Old database pages      2623 # old sublist页数量
Modified db pages       0 # buffer pool中正在变动的页数量
Percent of dirty pages(LRU & free pages): 0.000
Max dirty pages percent: 75.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 7720576, not young 456154990 # 这就是读old导致变young的页数量
0.00 youngs/s, 0.00 non-youngs/s
Pages read 10267303, created 1514710, written 2334136
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 7162, unzip_LRU len: 0 # 和上文的database pages一致
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
```

​	从上面看出buffer pool的页大小为7162，但是算一下按照3/8是Old Sublist的话，这里的页数量大致符合。其次是Buffer pool size 约等于 database pages和old database pages的和。

​	buffer pool在表information_schema.INNODB_BUFFER_POOL_STATS中页也有对应的信息：

| POOL\_ID | POOL\_SIZE | FREE\_BUFFERS | DATABASE\_PAGES | OLD\_DATABASE\_PAGES | MODIFIED\_DATABASE\_PAGES | PENDING\_DECOMPRESS | PENDING\_READS | PENDING\_FLUSH\_LRU | PENDING\_FLUSH\_LIST | PAGES\_MADE\_YOUNG | PAGES\_NOT\_MADE\_YOUNG | PAGES\_MADE\_YOUNG\_RATE | PAGES\_MADE\_NOT\_YOUNG\_RATE | NUMBER\_PAGES\_READ | NUMBER\_PAGES\_CREATED | NUMBER\_PAGES\_WRITTEN | PAGES\_READ\_RATE | PAGES\_CREATE\_RATE | PAGES\_WRITTEN\_RATE | NUMBER\_PAGES\_GET | HIT\_RATE | YOUNG\_MAKE\_PER\_THOUSAND\_GETS | NOT\_YOUNG\_MAKE\_PER\_THOUSAND\_GETS | NUMBER\_PAGES\_READ\_AHEAD | NUMBER\_READ\_AHEAD\_EVICTED | READ\_AHEAD\_RATE | READ\_AHEAD\_EVICTED\_RATE | LRU\_IO\_TOTAL | LRU\_IO\_CURRENT | UNCOMPRESS\_TOTAL | UNCOMPRESS\_CURRENT |
| :------- | :--------- | :------------ | :-------------- | :------------------- | :------------------------ | :------------------ | :------------- | :------------------ | :------------------- | :----------------- | :---------------------- | :----------------------- | :---------------------------- | :------------------ | :--------------------- | :--------------------- | :---------------- | :------------------ | :------------------- | :----------------- | :-------- | :------------------------------- | :------------------------------------ | :------------------------- | :--------------------------- | :---------------- | :------------------------- | :------------- | :--------------- | :---------------- | :------------------ |
| 0        | 8191       | 1024          | 7162            | 2623                 | 0                         | 0                   | 0              | 0                   | 0                    | 7720576            | 456154990               | 0                        | 0                             | 10267303            | 1514710                | 2334136                | 0                 | 0                   | 0                    | 1969270034         | 0         | 0                                | 0                                     | 1897385                    | 54410                        | 0                 | 0                          | 0              | 0                | 0                 | 0                   |

​	相关信息的主要内容其实在上面已经叙述过，所以这里不再讨论。
