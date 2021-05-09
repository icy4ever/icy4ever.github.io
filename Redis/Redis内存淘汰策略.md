## Redis内存淘汰策略

MAXMEMORY POLICY：当达到最大内存限制时，Redis如何选择要删除的内容
您可以选择以下五种行为：

|    淘汰策略     |                 描述                  |
| :-------------: | :-----------------------------------: |
|  volatile-lru   | 使用LRU算法删除设置了过期时间的键值对 |
|   allkeys-lru   |     根据LRU算法在所有键值对中淘汰     |
| volatile-random |    随机删除设置了过期时间的键值对     |
| allkeys-random  |        在所有键值对中随机删除         |
|  volatile-ttl   |         删除最快要到期的密钥          |
|   noeviction    | 不过期，只在容量满了的情况下返回错误  |


＃注意：使用上述任何策略，Redis会在没有合适退出键的情况下写入时返回错误
＃默认为：
＃maxmemory-policy noeviction