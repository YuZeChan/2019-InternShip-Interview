### Hadoop



***

##### Hadoop是什么：

​	Hadoop是一个**分布式基础架构系统**，是Google云计算基础架构系统的**开源实现**，主要包含两个核心部分：HDFS（Hadoop Distributed File System）和 MapReduce。



​	Hadoop实现了（主要用Java）实现了**HDFS文件系统**和**MapReduce计算框架**，使Hadoop成为了一个分布式计算平台。用户只要实现了Map和Reduce，并注册Job即可自动分布式运行。



​	**Hadoop是一种典型的Master-Slave架构：**![Hadoop基本架构](/resources/Hadoop基本架构.jpg)

一个Master逻辑结点（不一定是某一台机器）：

- NameNode：是HDFS的master，负责分布式文件系统**元数据**的管理工作。
- JobTracker：是MapReduce的master，主要负责启动、跟踪、调度各个TaskTraker的任务执行。

多个Slave逻辑结点：

- DataNode：实际文件的存储位置。
- TaskTraker：根据要求结合本地数据执行Map任务和Reduce任务。



**Hadoop生态链：**

- HDFS
- MapReduce
- Zookeeper分布式协调系统，是高可用和可靠的分布式系统（分布式锁等服务）。
- Hbase基于Hadoop的分布式数据库（非关系型）
- Hive为提供简单的数据操作而设计的分布式数据仓库（SQL查询）

![Hadoop生态系统](/resources/Hadoop生态系统.png)



***

##### 大数据、Hadoop和云计算：



###### 大数据的定义：

 	**数据量巨大**，需要运用**新处理模式**才能具有更强的决策力、洞察力和流程优化能力的海量、高增长率和多样化话的**信息资产**。



###### 大数据的特征（4个V）：

- 数据量巨大（Volume）
- 数据类型繁多（Variable）
- 价值密度低（Value）
- 处理速度快（Velocity）



###### 大数据挖掘：两个阶段（数据过滤清洗，商业价值挖掘）



***

###### Hadoop数据存储与切分：

是由HDFS负责的，具有以下特征：

- 多整个集群具有**单一的命名空间**
- **数据一致性。**适合一次写入多次读取，客户端在文件没有被成功创建前无法看到文件的存在。
- 文件会被分割成多个文件块，每个文件块被分配存储到数据节点上，且根据配置会复制文件块来保证**数据的安全性**。

HDFS的组成部分：

- 名称节点（NameNode）：负责管理文件系统的**命名空间、集群配置信息、存储块的复制。存储元数据**，包括文件对应的文件块信息和其所在DataNode信息。
- 数据节点（DataNode）：**文件存储的基本单元**。将Block存储在本地文件系统中，保存Block的元数据，并周期性发送所有存在的Block的报告给NameNode。
- 客户端：需要获取分布式文件系统的**应用程序**。

![HDFS读取](/resources/HDFS读取.jpg)

**文件写入HDFS的基本流程：**

1. 客户端向NameNode发起文件写入请求
2. NameNode根据文件大小和文件块配置情况，向客户端返回其管理的DataNode信息
3. 客户端将文件分为多个Block，根据DataNode的地址信息，按顺序写入每一个DataNode中。

**可见，分块是在客户端处进行的，客户端也需加载配置信息。**



**文件读取HDFS的基本流程：**

1. 客户端向NameNode发起文件读取请求
2. NameNode返回文件存储的DataNode信息
3. 客户端读取文件信息



**在HDFS中复制文件块的基本流程：**

1. NameNode发现部分文件的Block**不符合最小复制数**或**部分DataNode失效**
2. 通知DataNode相互复制Block
3. DataNode相互复制





***

###### MapReduce：

​	“分而治之，然后规约”

适合半结构化或非机构化数据。

用户实现Map和Reduce函数（都是运行在Key-Value对的基础上的）



MapReduce处理流程：

1. Map前先对输入数据有一个split过程
2. 一个split对应一个Map进程
3. Map之后进行Shuffle和Sort（Shuffle映射，Sort依键排序）
4. 将形成的中间Key-Value对交给Reduce函数

