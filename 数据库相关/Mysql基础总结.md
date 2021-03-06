##### 1、红黑树、B-树、B+树的基本区别：

**B-树与红黑树而言：**

- B-树通过节点内存多个key来减少层数，将比较过程的一部分迁到内存中进行（一般排好序了，用**二分查找**），从而减少一部分IO操作，提升效率。
- 所有的叶节点都位于同一层，使得B-树高度稳定。



**B+树与B-树而言：**

- B+树的非叶结点不放key，只放索引，在叶结点中放指向数据的指针（或者存放卫星数据。ps：卫星数据：索引元素所指向的数据记录（如数据库中的哪一行））。

- - 这样B+树每次查询的比较次数一致（都是树的高度），**性能稳定**。
  - 另外可以在相同页面大小的情况下，读入更多节点元素，使得树更加“矮胖"，**减少IO操作**。

- B+树的叶结点都用类似链表的结构相连，所以在**范围查找**时更加快速。



***

##### 2、聚簇索引和非聚簇索引。

- 聚簇索引：叶结点直接包含卫星数据。
- 非聚簇索引：叶结点带有指向卫星数据的指针。



***

##### 3、MySQL的基本结构：

- 连接池组件
- 管理服务和工具组件
- SQL接口组件
- 查询分析器组件（explain）
- 优化器组件（优化查询路径的）
- 缓冲组件
- 插件式存储引擎（基于表，而非数据库InnoDB和MyISAM）
- 物理文件



***

##### 4、MySQL索引：

索引是在**存储引擎层**实现的，而不是在服务器层实现的，所以不同存储引擎具有不同的索引类型和实现。

- B+Tree索引：引擎常用数据结构
- 哈希索引：和B+Tree索引共用，用于某个索引值频繁使用的情况，在B+树索引之上在创建一个哈希索引（InnoDB自适应哈希）
- 全文索引：MyISAM支持，InnoDB5.6.4以上版本支持
- 空间索引（R-Tree）：空间数据索引会从所有维度来索引数据，可以有效地使用任意维度来进行组合查询。**MyISAM支持**。



***

##### 5、事务和事务隔离级别：

- 事务的定义：首先事务是数据库管理系统（DBMS）执行的一个逻辑单位，它由有限个数据库操作序列组成。

- 事务的性质：ACID，原子性、一致性（A给B转账，A减B加必须全部执行。）、隔离性、持久性。

- 并发下事务会产生的问题：

- - 脏读；**事务A读到了事务B还没有提交的数据。**
  - 不可重复读；**在一个事务里面读取了两次某个数据，读出来的数据不一致。**
  - 幻读；**在一个事务里面的操作中发现了未被操作的数据。**

- 事务的隔离及级别：

  **事务隔离级别，就是为了解决上面几种问题而诞生的。**为什么要有事务隔离级别，因为事务隔离级别越高，在并发下会产生的问题就越少，但同时付出的性能消耗也将越大，因此很多时候必须在**并发性和性能**之间做一个权衡。

- - 读未提交
  - 读已提交；防止脏读
  - 可重复读；防止脏读，不可重复读
  - 序列化；防止脏读，不可重复读，幻读



***

##### 6、InnoDB和MyISAM的区别：

这两种数据库引擎是最常讨论的，而非就只有这两种引擎。

- InnoDB是**聚簇索引**的代表。

  - 其主键索引树的叶结点挂的是行数据本身。
  - 辅助索引树的叶结点挂的是主键。
  - 所以其查询方式是，有主键索引先查主键索引树，没有主键索引，则先查辅助索引树获得主键索引，再查主键索引树获取内容。
  - 这样分离性存储可以使得，**在行移动时，辅助索引树不受影响**。

- InnoDB是以页（page 16K）为单位存储的：

  - 分为主索引树非叶结点，主索引树叶结点，辅助索引数非叶结点，辅助索引树叶结点。
  - 每个page的头通过双向链表相连。每个page内部数据也是首尾相连的。

- InnoDB的默认隔离级别是可重复读（利用MVCC多版本并发控制实现：**同一份数据临时保留多版本的方式**），可以用Next-Lock来使之实现序列化。

- InnoDB的锁机制：悲观锁，支持事务，支持行锁和表锁。

  - InnoDB的表锁是：意向锁（互斥，共享）
    - 当修改行时：整表加意向互斥锁，在所修改的行加互斥锁（因为InnoDB不记录行的信息，修改时需要逐行遍历检查），这样的话就直接检验意向互斥锁即可。

  - InnoDB的三种行锁：
    - 记录锁，Record Lock 单行记录
    - 间隙锁，Gap Lock 锁定范围，不含当前行
    - Next-Key Lock，前两个加起来。
  - **InnoDB对主键是锁行的，对非主键是锁表的。**

- InnoDB支持外键。

- 清空整个表时，InnoDB时一行一行的删除，效率非常慢。这样也基本可以理解，毕竟InnoDB是行锁，写的时候加锁，所以只能一行一行删除。







- MyISAM则是非聚簇索引，所有的叶节点，都存的是行数据的指针：
  - 设计简单，数据以紧密格式存储。对于只读数据，或者表比较小、可以容忍修复操作，则依然可以使用它。
  - 提供了大量的特性，包括压缩表、空间数据索引等。
  - 不支持事务。
  - 不支持行级锁，**只能对整张表加锁**，读取时会对需要读到的所有表加共享锁，写入时则对表加排它锁。但在表有读取操作的同时，也可以往表中插入新的记录，这被称为并发插入（CONCURRENT INSERT）。
  - 可以手工或者自动执行检查和修复操作，但是和事务恢复以及崩溃恢复不同，可能导致一些数据丢失，而且修复操作是非常慢的。
  - 支持延时写入，先写入缓冲，表关闭时再写入。
  - MyISAM在清空整个表的时候，是直接重建表的。



​	InnoDB 中**不保存表的行数**，如 select count(*) from table 时，InnoDB需要扫描一遍整个表来计算有多少行，但是 MyISAM 只要简单的读出保存好的行数即可。注意的是，当 count(*)语句包含 where 条件时 MyISAM 也需要扫描整个表；





***

##### 7、单列索引和复合索引：

复合（联合）索引：两个或两个以上列的索引，称为复合索引。

- 当查找所有列或前几列，复合索引很有用。
- 查询的顺序，a,ab,abc使用复合索引，bc使用单列索引。
- 多个单列索引在多条件查询时，只会生效第一个索引！所以多条件联合查询时，最好建立联合索引。



**查询优化器**：多个索引可以走到，根据查询成本来选择。



***

##### 8、查询性能优化：

- 使用Explain进行分析

- 优化数据访问

- - 减少访问量	
  - 减少服务端扫描行数

- 重构查询方式

- - 切分大查询
  - 分解大连接（join）



***

##### 9、索引优化：

- 独立的列
- 多列索引
- 索引列的顺序（选择性强的放前面）
- 前缀索引（用于BLOB，TEXT，VARCHAR类型的列）
- 覆盖索引



***

##### 10、数据库的调优：

- SQL调优（慢查询系统来定位，Explain工具来调优）

- **架构层面的调优**

- - 多从库负载均衡
  - 读写分离
  - 水平和垂直分库分表

- 连接池调优（Connection Pool 连接池允许闲置的连接被其他需要的线程使用）



***

##### 11、读写分离和主从复制：

​	**读写分离：**其实很简单，就是基于主从复制架构，简单来说，就搞一个主库，挂多个从库，然后我们就单单只是写主库，然后主库会自动把数据给同步到从库上去。（**主库读写，从库只读**）

​	**主从复制：**主库将变更写入 **binlog 日志**，然后从库连接到主库之后，从库有一个 **IO 线程**，将主库的 binlog 日志拷贝到自己本地，写入一个 **relay 中继日志**中。接着从库中有一个 **SQL 线程**会从中继日志读取 binlog，然后执行 binlog 日志中的内容，也就是在自己本地再次执行一遍 SQL，这样就可以保证自己跟主库的数据是一样的。

- - 半同步复制（ack确认）
  - 并发复制（多线程）



***

##### 12、MySQL整个查询执行过程，总的来说分为5个步骤：

- 客户端向MySQL服务器发送一条查询请求
- 服务器首先检查查询缓存，如果命中缓存，则立刻返回存储在缓存中的结果。否则进入下一阶段
- 服务器进行SQL解析、预处理、再由优化器生成对应的执行计划
- MySQL根据执行计划，调用存储引擎的API来执行查询
- 将结果返回给客户端，同时缓存查询结果



***

##### 13、一些面试中遇到的奇奇怪的小知识点：

联合索引的生效原则（**最左匹配原则**）：按照联合索引的顺序，**从前往后依次使用生效**，如果中间某个索引没有使用，那么断点前面的索引部分起作用，**断点后面的索引没有起作用**。



应该注意：只要列中包含有NULL值都将不会被包含在索引中，复合索引中只要有一列含有NULL值，那么这一列对于此复合索引就是无效的。所以我们在数据库设计时不要让字段的默认值为NULL。（但无效并非代表不能设置为索引，只是其在查询中不起作用）



复合主键：指表的主键含有一个以上的字段组成。

```SQL
create table test(    
    name varchar(19),    
    id number,  
    value varchar(10),  
    primary key (name,id)
) 
```

上面的name和id字段组合起来就是test表的复合主键 ,它的出现是因为name字段**可能会出现重名**，所以要加上id字段，这样就可以保证记录的唯一性 。

**<u>某几个主键字段值出现重复是没有问题的，只要不是有多条记录的所有主键值完全一样，就不算重复。</u>**



**两者的区别：**

- 联合主键：这几个字段加起来是**唯一**的，且**不能为空**。
- 组合索引：这几个字段加起来不一定是唯一的，可以为空。





***

##### 14、查询关键词的顺序：

查询中用到的关键词主要包含六个，并且他们的顺序依次为

select--from--where--group by--having--order by



having为何放在group by后面？

因为having是对分组结果进行筛选的，即**使用having的前提是分组**。

![img](/resources/查询关键词顺序.png)



***

##### 15、如何对数据库进行垂直拆分和水平拆分：

​	水平拆分：把一个表的数据给弄到多个库的多个表中，但是每个库中的表**结构都一样**，只不过每个库表存放的**数据是不同的**，所有库表的数据加起来就是全部数据。

​	水平拆分的意义：将数据均匀放置到更多的库里，然后用多个库来抗住更高的并发量，另外使用多个库可以扩充容量。

![img](/resources/水平拆分.png)

​	垂直拆分：把一个有很多字段的表拆分成多个表，或者是多个库。每个库表的结构都不一样，每个库表都包含全部字段。一般来说会把访问频率很高的字段放到一个表里去，将访问量较少的字段放到另一个表中。

​	垂直拆分的意义：因为数据库的**缓存**是固定大小的，将频率高的字段数减少（单行数据量减少），则可以读入更多行的数据，这样性能越好。

![img](/resources/垂直拆分.png)

一般实际操作的就是这两种**分库分表的方式**：

- 一种是按照 range 来分，就是每个库一段**连续**的数据，这个一般是按比如**时间范围**来的，但是这种一般较少用，因为很容易产生热点问题，大量的流量都打在最新的数据上了。
- 或者是按照某个字段hash一下**均匀分散**，这个较为常用。

range 来分，好处在于说，**扩容的时候很简单**，因为你只要预备好，给每个月都准备一个库就可以了，到了一个新的月份的时候，自然而然，就会写新的库了；缺点，但是大部分的请求，都是访问最新的数据。实际生产用 range，要看场景。

hash 分发，好处在于说，可以**平均分配每个库的数据量和请求压力**；坏处在于说扩容起来比较麻烦，会有一个数据迁移的过程，之前的数据需要重新计算 hash 值重新分配到不同的库或表。



***

16、哈希索引和B+Tree索引相比的劣势：

- **如果是等值查询，那么哈希索引明显有绝对优势**，因为只需要经过一次算法即可找到相应的键值；当然了，这个前提是，键值都是唯一的。如果键值不是唯一的，就需要先找到该键所在位置，然后再根据链表往后扫描，直到找到相应的数据；
- 从示意图中也能看到，**如果是范围查询检索，这时候哈希索引就毫无用武之地了**，因为原先是有序的键值，经过哈希算法后，有可能变成不连续的了，就没办法再利用索引完成范围查询检索；
- 同理，**哈希索引也没办法利用索引完成排序，**以及like ‘xxx%’ 这样的部分模糊查询（这种部分模糊查询，其实本质上也是范围查询）；
- **哈希索引也不支持多列联合索引的最左匹配规则**；
- B+树索引的关键字检索效率比较平均，不像B树那样波动幅度大，**在有大量重复键值情况下，哈希索引的效率也是极低的，因为存在所谓的哈希碰撞问题**。