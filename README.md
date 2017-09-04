# reading
redis阅读理解，带详细注释  



说明
===================================  
本份代码从https://github.com/huangz1990/redis-3.0-annotated clone下来，然后自己添加自己的理解，再次基础上增加函数调用流程注释。
参考书籍《redis设计与实现》 


阅读计划和进度：
===================================  
第一阶段：
   阅读redis的数据结构部分
   内存分配 amalloc.c和zmalloc.h
   动态字符串sds.c和sds.h
   双端队列 adlist.h和adlist.c
   字典 dict.h和dict.c
   跳跃表 关于zskiplist结构和zskiplistNode结构
   日志类型 hyperloglog.c的hllhdr
第二阶段
	熟悉redis的内存编码结构
	整数集合数据结构 intset.h和intset.c
	压缩列表数据结构 ziplist.h和ziplist.c
第三阶段
	熟悉 Redis 数据类型的实现
	对象系统 object.c
	字符串键 t_string.c
	列表键 t_list.c
	散列键 t_hash.c
	集合键 t_set.c
	有序集合键 t_zset.c
	hyperloglog键
第四阶段
 	熟悉redis数据库的实现
 	数据库实现 redis.h文件中redisDb结构以及db.c文件
 	通知功能notify.c
 	RDB持久化rdb.c
 	AOF持久化 aof.c
 	以及一些独立功能模块的实现
 	发布和订阅 redis.h文件的pubsubPattern 以及pubsub.c文件
 	事务redis.h文件的multiState结构以及multiCmd结构 multi.c文件
 第五阶段
 	熟悉客户端和服务端的代码实现
 	时间处理模型
 	网络链接库anet.c和network.c
 	服务端 redis.c
 	客户端 redis-cli.c
 	独立功能模块代码实现
 	lua脚本  scripting.c
 	慢查询 slowlog.c
 	监视 monitor.c
 第六阶段 
 	熟悉Redis多机部分代码实现
 	复制功能 replication.c
 	Redis Sentinel sentinel.c 哨点检测
 	集群 cluster.c

进度：
	2017.9.5 完成代码走读，了解nginx基本的实现机制。
	后期计划重点学习redis 集群和备份实现机制。

问题及改造点： 
===================================  
	1、在应答客户端请求数据的时候，全是epoll采用epool write事件触发，这样不太好，每次发送数据前通过epoll_ctl来触发epoll write事件，即使发送一个"+OK"字符串，也是这个流程。
	解决办法：
		开始不把socket加入epoll，需要向socket写数据的时候，直接调用write或者send发送数据。如果返回EAGAIN，把socket加入epoll，在epoll的驱动下写数据，全部数据发送完毕后，再移出epoll。  
	优点是：
		1、数据不多的时候可以避免epoll的事件处理，提高效率。
		2、返回数据小时能得到及时处理，大value数据添加poll写回调进行写数据。
	缺点：
		1、有可能导致添加到epoll写事件无法及时处理。如果write或send发送数据添加到送缓冲区，导致缓冲区不可写，epoll时间不能及时得到调用。

	2、master多slave，新的slave通过投票机制表为master后，其他slave需要和这个新master进行全量同步，效率低下，且新master不一定最新同步的slave。
	解决办法：
	 	slave也存储积压缓冲区，每次slave通过投票称为master后，采用部分同步机制实现。
	优点：
	 	1、减少集群故障恢复时新的master和slave之间数据同步数据。
	 	2、新master数据不一定是最新的，有可能slave复制偏移大于新的master，这样就需要master和slave之间先进行同步，尽量做到故障恢复少丢失数据。
	缺点：
	 	1、slave也需要存储积压缓冲区，浪费内存。
	 	2、故将恢复需要新master和slave之间进行一轮同步。
	
	3、主从进行全量同步时，是否需要进行必要限速，否则导致网卡打满，影响正常服务的相应。
	优点：
		1、正常服务不会因为主从全量同步而受影响，导致服务阻塞。
	缺点：
		1、主从同步速度变慢。 

	4、主服务器同步rdb文件给从服务器的时候是采用直接读取文件，然后通过网络发送出去，首先需要把文件内容从内核读取到应用层，在通过网络应用程序从应用层到内核网络协议栈
		这样有点浪费CPU资源和内存。  
  	解决办法：
  		通过sendfile或者aio方式发送，避免多次内核与应用层交互，提高性能。或者在全量同步的时候，不做RDB重写，而是直接把内存中KV安装RDB格式组包直接发送到slave。  


问题：
	1、qps比较高的情况下，由于单线程的原因，时延会慢慢增大。
	2、aof会占用磁盘空间很大，当磁盘空间满了后，flushAppendOnlyFile会失败，这时候无法恢复，会丢失部分数据,只能加硬盘修复。
	3、hash结构存储的HGETALL、HDEL，如果hash上存储的kv对太多，容易造成redis阻塞，进一步引起集群节点反复掉线，集群抖动进一步引起整体同步.
		是否应该起一个线程单独做这个事情，或者像scan机制那样异步逐条清理 。
	4、slave重启会触发全量同步，可以优化为增量同步，LAVE重启需要全量同步，是否可以在重启前先记录下ID和偏移量。
	5、rdb aof重写容易触发oom 
	6、热点数据和大value数据容易造成负载不均。
	
=================================== 
运维方面：  
	1. 多实例部署的时候，最好保证master slave在不同的物理机上，保证一个物理机掉电等故障，能正常提供服务  
	2. 同一物理机多实例部署的时候，最好每个实例的redis-server放在不同路径下面，当对redis做二次开发的时候，初期验证阶段可以只替换部分实例，这样对业务影响面较小  
