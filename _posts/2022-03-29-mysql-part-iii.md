# MySQL随笔 - Buffer Pool

​	MySQL和大部分程序一样，会在内存和磁盘中存储数据。内存中存储数据主要是提示数据的访问速度，让热数据能更快的返回。在内存中，主要的数据会放在buffer pool中，其中又分为四大部分：buffer pool list、change buffer、adaptive hash index、log buffer。

## Buffer Pool

### Buffer Pool List

​	MySQL的buffer pool采用的是**链表**数据结构，并使用了**LRU算法**来淘汰旧数据。数据按照冷热会被分为两块区域，如下图所示：

![Content is described in the surrounding text.](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/images/innodb-buffer-pool-list.png)



​	当用户通过一条查询语句来查询数据时（全表查询除外），新的数据会被放在上图中Old Sublist的Head位置，而溢出的Old Sublist的尾部将被淘汰。当它再次被访问时，Old Sublist的内存页会被上升到New Sublist里。通过这样的机制，buffer pool保证里在内存里的数据都是最热的数据。也不难的出结论：当MySQL的buffer pool增加到一定程度时，它其实本质上就会变为一个缓存。

### Change Buffer

​	change buffer也是MySQL内存中的一个数据结构，它的存在是为了减少二级索引对查询性能的影响。

![Content is described in the surrounding text.](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/images/innodb-change-buffer.png)

​	当MySQL存在一些update、delete等数据变动对二级索引产生影响时，不会立即去改动磁盘中的数据，防止磁盘的随机i/o对数据造成影响，这些变动的页会被记录到change buffer中。当数据被读取到buffer pool中时，change buffer的数据才会被同步刷新到磁盘。

### Adaptive Hash Index

​	适应性hash索引是一种将经常访问的数据添加到内存中的数据结构，它可以让用户更快的访问到内存中的数据。

### Log Buffer

​	Log Buffer是一个会周期性同步数据到Log file的缓存，当Log Buffer增加时，我们可以获得更快的事务支持。因为它会将redo log放入缓存中，加速了事务的提交，减少了磁盘的I/O。

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

​	buffer pool在表**information_schema.INNODB_BUFFER_POOL_STATS**中页也有对应的信息：

| POOL\_ID | POOL\_SIZE | FREE\_BUFFERS | DATABASE\_PAGES | OLD\_DATABASE\_PAGES | MODIFIED\_DATABASE\_PAGES | PENDING\_DECOMPRESS | PENDING\_READS | PENDING\_FLUSH\_LRU | PENDING\_FLUSH\_LIST | PAGES\_MADE\_YOUNG | PAGES\_NOT\_MADE\_YOUNG | PAGES\_MADE\_YOUNG\_RATE | PAGES\_MADE\_NOT\_YOUNG\_RATE | NUMBER\_PAGES\_READ | NUMBER\_PAGES\_CREATED | NUMBER\_PAGES\_WRITTEN | PAGES\_READ\_RATE | PAGES\_CREATE\_RATE | PAGES\_WRITTEN\_RATE | NUMBER\_PAGES\_GET | HIT\_RATE | YOUNG\_MAKE\_PER\_THOUSAND\_GETS | NOT\_YOUNG\_MAKE\_PER\_THOUSAND\_GETS | NUMBER\_PAGES\_READ\_AHEAD | NUMBER\_READ\_AHEAD\_EVICTED | READ\_AHEAD\_RATE | READ\_AHEAD\_EVICTED\_RATE | LRU\_IO\_TOTAL | LRU\_IO\_CURRENT | UNCOMPRESS\_TOTAL | UNCOMPRESS\_CURRENT |
| :------- | :--------- | :------------ | :-------------- | :------------------- | :------------------------ | :------------------ | :------------- | :------------------ | :------------------- | :----------------- | :---------------------- | :----------------------- | :---------------------------- | :------------------ | :--------------------- | :--------------------- | :---------------- | :------------------ | :------------------- | :----------------- | :-------- | :------------------------------- | :------------------------------------ | :------------------------- | :--------------------------- | :---------------- | :------------------------- | :------------- | :--------------- | :---------------- | :------------------ |
| 0        | 8191       | 1024          | 7162            | 2623                 | 0                         | 0                   | 0              | 0                   | 0                    | 7720576            | 456154990               | 0                        | 0                             | 10267303            | 1514710                | 2334136                | 0                 | 0                   | 0                    | 1969270034         | 0         | 0                                | 0                                     | 1897385                    | 54410                        | 0                 | 0                          | 0              | 0                | 0                 | 0                   |

​	在非生产环境下，表**information_schema.INNODB_BUFFER_PAGE**将提供给我们关于buffer pool中每一个内存页的信息：

| POOL\_ID | BLOCK\_ID | SPACE | PAGE\_NUMBER | PAGE\_TYPE | FLUSH\_TYPE | FIX\_COUNT | IS\_HASHED | NEWEST\_MODIFICATION | OLDEST\_MODIFICATION | ACCESS\_TIME | TABLE\_NAME | INDEX\_NAME | NUMBER\_RECORDS | DATA\_SIZE | COMPRESSED\_SIZE | PAGE\_STATE | IO\_FIX  | IS\_OLD | FREE\_PAGE\_CLOCK |
| :------- | :-------- | :---- | :----------- | :--------- | :---------- | :--------- | :--------- | :------------------- | :------------------- | :----------- | :---------- | :---------- | :-------------- | :--------- | :--------------- | :---------- | :------- | :------ | :---------------- |
| 0        | 0         | 1847  | 5779         | INDEX      | 0           | 0          | NO         | 0                    | 0                    | 3620866828   | \`a\`.\`b\` | PRIMARY     | 61              | 14844      | 0                | FILE\_PAGE  | IO\_NONE | NO      | 11767432          |
| 0        | 1         | 1847  | 10176        | INDEX      | 0           | 0          | NO         | 0                    | 0                    | 3620887249   | \`a\`.\`b\` | uuid        | 227             | 8853       | 0                | FILE\_PAGE  | IO\_NONE | YES     | 0                 |
| 0        | 2         | 1848  | 9398         | INDEX      | 0           | 0          | NO         | 0                    | 0                    | 2507606063   | \`a\`.\`c\` | PRIMARY     | 99              | 15129      | 0                | FILE\_PAGE  | IO\_NONE | NO      | 11762960          |
| 0        | 3         | 0     | 669          | UNDO\_LOG  | 1           | 0          | NO         | 23585329985          | 0                    | 3298651665   | NULL        | NULL        | 0               | 0          | 0                | FILE\_PAGE  | IO\_NONE | NO      | 11764958          |
| 0        | 4         | 1847  | 15252        | INDEX      | 0           | 0          | NO         | 0                    | 0                    | 3620867147   | \`a\`.\`b\` | PRIMARY     | 36              | 15138      | 0                | FILE\_PAGE  | IO\_NONE | YES     | 0                 |
