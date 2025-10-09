# db 缓存

MySQL 有两个缓存机制，一个是 query cache，另一个是 buffer pool。

query cache 缓存的是 SQL 语句及其结果集，缓存在内存中。如果 SQL 语句重复度低，那么 query cache 的缓存命中率就很低。

buffer pool 缓存的是数据库中表的结果，加速写和查询。如果表中的数据在 buffer pool 中，那么直接使用，如果没有则再从硬盘上加载。它是 MySQL 主要的缓存方式。



## query cache 相关的参数

~~~sql
mysql> show variables like '%query_cache%';
+------------------------------+---------+
| Variable_name                | Value   |
+------------------------------+---------+
| have_query_cache             | YES     |
| query_cache_limit            | 1048576 |
| query_cache_min_res_unit     | 4096    |
| query_cache_size             | 1048576 |
| query_cache_type             | OFF     |
| query_cache_wlock_invalidate | OFF     |
+------------------------------+---------+
~~~

参数解析

- `have_query_cache`  表示是否支持 query cache
- `query_cache_limit` 表示缓存块的大小，超了就不会被缓存。
- `query_cache_size` 表示缓存的总内存大小，一般设置为 200M，不建议超过 256M
- `query_cache_type` 表示是否开启 query cache



**查看 query cache 的使用情况**

~~~sql
mysql> show status like '%qcache%';
+-------------------------+---------+
| Variable_name           | Value   |
+-------------------------+---------+
| Qcache_free_blocks      | 1       |
| Qcache_free_memory      | 1031832 |
| Qcache_hits             | 0       |
| Qcache_inserts          | 0       |
| Qcache_lowmem_prunes    | 0       |
| Qcache_not_cached       | 63      |
| Qcache_queries_in_cache | 0       |
| Qcache_total_blocks     | 1       |
+-------------------------+---------+
~~~

参数解析

- `Qcache_free_blocks` 表示内存碎片的大小。值不应该过大，大了应该清理，清理的方法：`flush query cache;`
- `Qcache_free_memory` 表示 qcache 剩余的缓存空间大小。
- `Qcache_hits` 缓存命中次数
- `Qcache_inserts` 未命中次数。
- 命中率：`Qcache_hits` / ( `Qcache_hits`  + `Qcache_inserts` )
- `Qcache_lowmem_prunes` 表示因为内存不足而被清除 query 的次数
- `Qcache_not_cached` 不能被 cache 的查询次数。
- 当前 qucher cache 中被缓存的 query 数量。



## buffer pool 相关参数

buffer pool 是 innodb 存储引擎的一个缓存池。Buffer pool 的设置越大越好，可以设置物理服务器内存的 70%。

buffer pool 参数

~~~sql
mysql> show variables like '%innodb_buffer_pool%';
+-------------------------------------+----------------+
| Variable_name                       | Value          |
+-------------------------------------+----------------+
| innodb_buffer_pool_chunk_size       | 134217728      |
| innodb_buffer_pool_dump_at_shutdown | ON             |
| innodb_buffer_pool_dump_now         | OFF            |
| innodb_buffer_pool_dump_pct         | 25             |
| innodb_buffer_pool_filename         | ib_buffer_pool |
| innodb_buffer_pool_instances        | 1              |
| innodb_buffer_pool_load_abort       | OFF            |
| innodb_buffer_pool_load_at_startup  | ON             |
| innodb_buffer_pool_load_now         | OFF            |
| innodb_buffer_pool_size             | 134217728      |
+-------------------------------------+----------------+
~~~

参数解释

- `innodb_buffer_pool_size` 表示 buffer pool 内存大小
- `innodb_buffer_pool_dump_now` 值为开启则表示立刻将缓存池中的热数据保存到硬盘。
- `innodb_buffer_pool_load_now` 值为开启则表示立刻加载数据到缓存池。
- `innodb_buffer_pool_load_at_startup` 值为开启则表示启动 MySQL 时将本地热数据加载到缓存池。
- `innodb_buffer_pool_dump_at_shutdown` 值为开启则表示关闭 MySQL 时自动保存缓存池中数据到硬盘。



查看 innodb_buffer_pool 状态

~~~
show status like '%innodb_buffer_pool%';
~~~

其中两个参数可以计算缓存命中率：

（ `Innodb_buffer_pool_read_requests` - `Innodb_buffer_pool_reads` ）/ `Innodb_buffer_pool_read_requests` 



**注意**：MySQL 的 buffer pool 缓存池在数据量小的情况下足以应对数据查询的需求，但是数据量大了之后，受限于自身内存的大小，难以满足查询需要。因此需要引入更强的缓存方案。



## Redis 为什么快

Redis 是一种基于单线程模型的数据库（NOSQL，非关系型数据库），数据存在内存，使用 epoll 模型，高效的内存管理机制和数据结构，使得 Redis 的查询速度极快。

一般使用 Redis 做缓存和数据库使用。

Redis 快的原因：

- 基于单线程+ io 多路复用。单线程避免竞争消耗，io 多路复用优化了单线程下的 io 阻塞。推荐使用 epoll 模型（linux操作系统支持epoll），epoll 没有最大并发限制，基于事件回调机制，无需轮询；共享内存无需数据拷贝，不需要把用户态数据拷贝到内核态，效率高。redis6.x 之后推出了多线程模型，但主流程依然是单线程，只是网络 IO 处理环节使用多线程模型来加快分发事件。
- 数据存在内存。
- 高效的数据结构。牺牲空间换查询时间。
- 高效的内存管理。存数据时严格按照最大内存限制。过期数据采用惰性删除，同时也会定期删除键及时释放内存。高效的数据淘汰策略。

redis 的数据淘汰策略：

~~~bash
- noeviction：当内存不足以容纳新写入数据时，新写入操作会报错。
- allkeys-lru：当内存不足以容纳新写入数据时，，移除最近最少使用的 Key。
- allkeys-random：当内存不足以容纳新写入数据时，随机移除某个 Key。
- volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的 Key。
- volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个 Key。
- volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的 Key 优先移除。
~~~

