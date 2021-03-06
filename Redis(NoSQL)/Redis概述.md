### Redis概述：

Redis 是速度非常快的非关系型（NoSQL）内存键值数据库，可以存储键和五种不同类型的值之间的映射。



***

##### 数据结构与对象：

1. 简单动态字符串
2. 链表
3. 字典
4. 跳跃表
5. 整数集合
6. 压缩列表
7. 对象



Redis数据库里面每个键值对（k-v pair）都是由对象组成，其中：

- 数据库键总是一个字符串对象
- 数据库值可以是字符串对象、列表对象、集合对象、有序集合对象、哈希对象（散列表）中的一种。



***

##### 单机数据库的实现：

1. 数据库
2. RDB持久化
3. AOF持久化
4. 事件
5. 客户端
6. 服务器



Redis有两种持久化的实现：将内存中的数据持久化到硬盘，持久化文件可用来还原数据库。

- RDB持久化和AOF持久化。



***

##### 多机数据库的实现：

1. 复制
2. Sentinel（哨兵）
3. 集群



通过使用 slaveof host port 命令来**让一个服务器成为另一个服务器的从服务器**。

- 一个从服务器只能有一个主服务器，并且不支持主主复制。

Sentinel（哨兵）可以**监听集群中的服务器**，并在主服务器进入下线状态时，自动从从服务器中**选举出新的**主服务器。

分片是将数据划分为多个部分的方法，可以将数据存储到多台机器里面，这种方法在解决某些问题时**可以获得线性级别的性能提升**。

***

##### 独立功能的实现：

1. 发布与订阅
2. 事务
3. Lua脚本
4. 排序
5. 二进制数组
6. 慢查询日志
7. 监视器

