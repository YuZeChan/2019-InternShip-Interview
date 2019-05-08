### Hadoop存储模型：

***

分布式系统的核心问题：

- 数据分块
- 元数据管理
- 高可靠性
- 高可用性
- 高可扩展性
- 容错控制
- 高吞吐量
- 高传输



***

HDFS是有一个Master和大量块服务器Slave构成的。

- Master中存放文件系统的所有元数据，包括**命名空间、访问控制、文件分块信息、文件块位置信息**等等。

- 文件默认切分为64M的块进行存储，每份数据会默认保存3个以上的副本（高可靠性，高可用性）。
- 使用版本号来控制备份处于一直状态（一致性）



***

#### 基本概念：

##### NameNode：

首先Master/Slave是架构，是一种设计思想，不是真实存在的。

在HDFS中，**NameNode就是其中的Master**。主要负责管理两部分：namespace，block。

NameNode是始终**被动接受服务**的Server。

- ClientProtocol接口，用于客户端访问NameNode
- DataNodeProtocol接口，用于DataNode向NameNode通信
- NameNodeProtocol接口，用于多个NameNode之间的通信



数据块存在DataNode中，而NameNode则管理这些数据块的元数据信息（**两个映射**）：

- 文件名->数据块（直接在磁盘中存储）
- 数据块->DataNode列表（不保存在NameNode，而是DataNode上报建立）



**NameNode执行文件系统的名称空间操作，例如打开、关闭、重命名文件和目录，同时决定文件数据块到具体DataNode的节点映射。**



##### Secondary NameNode：

- 定时对NameNode数据snapshot（快照）备份，因为Master只有一个，尽量减少宕机后数据丢失的风险。
- 从NameNode获取fsimage和edits后，把两者重新合并发回，减轻NameNode负担



##### DataNode：

DataNode负责存储数据（**数据块ID和数据块内容**）：

- 一个数据块Block会备份到多个DataNode中
- 一个DataNode最多只存一个分Block的备份

DataNode定时和NameNode通信（上报更新映射表），也接收NameNode指令。



NameNode对DataNode只命令，不请求。

DataNode接受来自客户端的访问，处理块读写清求。

DataNode之间也会相互通信，执行块复制，保持和一致性。



##### 客户端：

访问HDFS的程序或Shell命令。



##### 块：

默认64MB，可修改。

世纪存储大小有时不占用一个块的大小（不对齐）



客户端读取HDFS上文件的过程：

1. 客户端把文件名和程序指定的字节偏移，根据块的大小，**计算出文件的Block索引**
2. 客户端将文件名和Block索引发到Master节点，Master节点将相应**Block标识和副本位置信息返回客户端**
3. 客户端发送请求（**一般为最近的**）一个副本，请求信息包含Block标识和字节范围
4. **后续操作，客户端可不再与Master通信**（除非元数据过期或文件被重新打开）

后来为了优化范围查询，减少通信，第二步中Master会返回对应Block后面的Block信息。



***

#### HDFS特性和目标：

##### HDFS的特性：

- 高容错，高可扩展，可配置性强
- 跨平台
- shell命令接口
- Web界面
- 文件权限和授权
- 机架感知功能
- 安全模式
- Rebalancer
- 升级和回滚

##### HDFS的目标：

- **硬件错误的检测和快速自动恢复**
- **数据的高速率、大批量处理**（大数据问题关心的是分析与读取，而非写入的速率）
- 大规模数据集
- **一次写多次读，尽量简化一致性**
- **计算靠近数据（移动计算比移动数据的代价低）**
- 可移植性



***

#### HDFS架构：



##### Master/Slave架构：

**节点的概念是逻辑概念，不一定是物理独立的**，比如客户端可以和NameNode或DataNode在同一台服务器上（只要资源允许）。

HDFS采用基于Master/Slave主从架构的分布式文件系统，一个HDFS集群包含一个单独的Master节点和多个Slave节点服务器。（同样是逻辑上的，如一个Master逻辑结点，可以包括两台物理主机；两台Master服务器可以组成双NameNode集群，而且可以被多个客户端访问）

![HDFS架构](/resources/HDFS架构.png)

- NameNode负责保存和管理所有HDFS元数据
  - 名称空间
  - 访问控制信息
  - 文件和Block映射信息（文件分块时为每个Block分配唯一ID）
  - 当前Block的位置信息
- NameNode还管理系统范围内的**活动**（如Block租用，孤立Block的回收，Block在DataNode服务器间的迁移）

- DataNode上进行文件数据的读写（不通过NameNode）

**HDFS客户端代码以<u>库的形式被链接</u>到客户程序中。客户端代码需要实现HDFS文件系统的API接口，以和NameNode和DataNode通信，对数据读写。**

<u>客户端和NameNode通信只获得元数据，所有数据操作都是由客户端直接和DataNode服务器交互的。</u>





***

##### NameNode和Secondary NameNode通信模型：

NameNode将对文件系统的改动追加保存在本地文件系统上的一个日志文件edits中

在NameNode启动时，先读取一个映像文件（fsimage），获取HDFS状态，而后执行edits中的编辑操作；将新的状态写入到fsimage中，使用一个空edits文件正常操作。

**NameNode在启动时才合并fsimage和edits。edits日志文件会久而久之变得越来越大（一直在运行）。**引入Secondary NameNode就是为了解决这类问题的。

![Secondary_NameNode](/resources/Secondary_NameNode.png)

- 两者通过**HTTP协议通信**，Secondary NameNode**定期合并**fsimage和edits日志，控制edits日志的大小。（**合并控制大小，备份以防宕机**）
- Secondary NameNode**内存消耗**和NameNode同数量级，所以两者通常运行在**不同**机器上。

- 几个参数
  - edits控制在64MB，否则强制检查合并
  - 默认检查间隔1h



***

##### 文件存取机制：（都是通过流对象读写）

###### 读文件过程：

1. 调用FileSystem的**open()**打开文件
2. DistributedFileSystem使用**RPC调用**NameNode，得到文件的数据块元数据信息，并返回**FSDataInputStream**给客户端（获取数据块的位置信息）
3. HDFS客户端调用stream的**read(**)函数开始读数据
4. 调用FSDataInputStream**直接从DataNode获取文件数据块**
5. 读完文件时，调用FSDataInputStream的**close()**函数

![HDFS读文件](/resources/HDFS读文件.png)

###### 写文件过程：

1. 初始化FileSystem，客户端调用create()来创建文件
2. FileSystem用**RPC调用**元数据节点，在文件系统的**命名空间中创建一个新的文件**，元数据节点首先确定文件原来不存在，并且客户端有创建文件的权限，然后创建新文件。
3. FileSystem返回**DFSOutputStream**，客户端用于写数据，客户端开始写入数据。
4. DFSOutputStream将数据**分成块**，**写入data queue**。data queue由Data Streamer读取，并通知元数据节点分配数据节点，用来存储数据块(每块默认复制3块)。分配的数据节点放在一个**pipeline**里。**Data Streamer将数据块写入pipeline中的第一个数据节点。第一个数据节点将数据块发送给第二个数据节点。第二个数据节点将数据发送给第三个数据节点。**
5. DFSOutputStream为发出去的数据块保存了**ack queue**，等待pipeline中的数据节点告知数据已经写入成功。
6. 当客户端结束写入数据，则调用stream的**close()函数**。此操作将所有的数据块写入pipeline中的数据节点，并等待ack queue返回成功。**最后通知元数据节点写入完毕**。
7. 如果数据节点在写入的过程中失败，关闭pipeline，将ack queue中的数据块放入data queue的开始的位置。

![HDFS写文件](D:\学习\基础知识汇总\Hadoop\resources\HDFS写文件.png)



***

#### HDFS核心设计：



##### Block大小的设计：

较大Block的优点：

1. 减少和NameNode的通信量（一次通信，多次读写操作）
2. 保持TCP长连接减少网络负载（客户端对一个较大的块操作多次（对较小的块可能就操作一次））
3. 减少NameNode保存的元数据量

较大Block的缺点：

1. 如果一个块保存整个小文件，可能会访问集中，导致局部过载。（更多备份来分散访问）



##### 数据复制：

**每个文件的Block大小和Replication因子都是可配置的。**

文件所有块的**复制，全权由NameNode进行管理**，通过心跳包。



##### 数据副本存放策略：

HDFS通过**“机架感知（rack-aware）”策略**来改进数据的可靠性、可用性和网络带宽利用率。

- 不同机架上的两台机器间通信需要通过交换机
- 同一机架内两台机器的带宽要更大

NameNode通过机架感知，获取每个DataNode所属的机架ID。



副本存放策略（默认三个副本）：

1. 第一个副本放在本地机架节点上
2. 第二个副本放在同一机架的另一个节点上
3. 第三个副本放在不同机架的节点上
4. 其他副本均匀分布

**Hadoop 0.17之后**

1. **副本一：同Client的节点上**
2. **副本二：不同机架中的节点上**
3. **副本三：同第二个副本的机架中的另一个节点上**
4. 其他副本：随机挑选

![副本存放](/resources/副本存放.png)



##### 数据组织：

符合write-once-read-many且读的**速度要满足流式读**。

客户端写文件时，**现在本地创建一个临时文件（缓存）**，数据量积累到一个Block大小，再与NameNode通信，写入文件系统目录，分配DataNode；**在文件关闭时才将文件保存在持久存储**（这阶段前NameNode宕机会导致文件丢失）。

**流水线复制**。



##### 空间回收：

用户删除某文件，不直接删除，而是会被重命名，而后被移动到Trash目录下，当超过超时时间（默认6小时，可设置），NameNode删除namespace数据，释放文件数据块。

当减少副本系数时，NameNode会在心跳检测时将消息传到DataNode，移除相应Block释放空间。



##### 通信协议：

TCP/IP协议

**NameNode从不会主动发起RPC，而是响应来自客户端和DataNode的RPC请求。**



##### 安全模式：

此模式下，不接受任何命名空间的修改和数据复制或删除。



##### 机架感知：

集群管理员配置dfs.network.script参数来确定结点所处的机架。



##### 健壮性：

数据错误：DataNode宕机，NameNode通过心跳包获取信息，会有副本低于指定值，NameNode会进行必要副本复制。

集群均衡：某个DataNode太满了，会搬迁至空闲的DataNode；某个文件访问增加，会复制出新的副本。

数据完整性：文件内容校验（创建文件时产生，读取时校验）

元数据磁盘错误：fsimage和edits副本

快照：数据损坏时回到某个之前的时间点



##### 负载均衡：

![HDFS数据重分布](/resources/HDFS数据重分布.png)



##### 升级和回滚：

```C++
bin/hadoop namenode -upgrade
bin/hadoop namenode -rollback
bin/hadoop namenode -finalize				//运行正常，提交
bin/hadoop namenode -importCheckpoint		//从Checkpoint恢复
```





***

#### HDFS缺点：

1. 访问时延：处理速度快，牺牲了效率。（HBase的时延更低）
2. 不善于大量小文件处理：NameNode内存保存文件目录、文件元数据等，需较大空间。（billion级别就无能为力了）
   1. 归档（HBase基于此）
   2. 多个Hadoop横向扩展
   3. 多Master设计
3. 只支持单用户写



***

#### HDFS的使用：

##### C接口libhdfs：

###### libhdfs介绍：

​	libhdfs是HDFS的一个基于JNI的C语言API接口，通过libhdfs可以使用c/c++语言来操作HDFS。

​	JNI（Java Native Interface）提供API实现Java和其他语言的通信。

**原理：**libhdfs中的函数通过JNI调用Java虚拟机，在JVM中构造对应的HDFS的Java类，然后**反射**调用该类的功能函数。（这块的解释我觉得不太对，反射应该是作用于对象的，所以构造的不应该是类应该是对象，或者说这个反射是错的。）

因为简介调用总会发生**JVM和程序之间的内存复制**，大规模下效率堪忧。



###### 编译与部署：

虽然Hadoop发行版包含已编译好的libhdfs库，但由于编译平台的兼容性问题，最好在使用前重新编译。

编译的版本和当前JVM版本（32位或64位）匹配。

```shell
ant -Dcompile.c++=true -Dlibhdfs=true compile-c++-libhdfs
```

另外要手动添加环境变量，这就不用多说了。



###### libhdfs接口介绍：

一、（首先要建立与HDFS的通信）建立与关闭HDFS连接：

```C++
//返回的是一个filesystem句柄，失败返回NULL
hdfsFS hdfsConnectAsUser(const char* host, 		//NameNode节点的IP地址或主机名，“default”表示本机系统
                        tPort port,				//HDFS对外服务的监听端口号
                        const char* user)		//要指定的Hadoop用户名

int hdfsDisconnect(hdfsFS fs)					//传入的就是被打开的句柄
```



二、建立连接之后，打开与关闭HDFS文件进行读写访问：

```C++
//返回的是打开文件的句柄，失败返回NULL
hdfsFile hdfsOpenFile(hdfsFS fs, 				//建立连接后的文件句柄
                     const char* path, 			//要打开文件的HDFS完全路径
                     int flags, 				//文件标志位，O_RDONLY、O_WRONLY、O_APPEND
                     int bufferSize, 			//读写缓冲区大小，0表示HDFS默认配置
                     short replication, 		//文件块复制因数~
                     tSize blocksize)			//文件块大小~
    
int hdfsCloseFile(hdfsFS fs, hdfsFile file);
```



三、之后可以进行读写操作，刷新Flush保存，查询操作，复制和删除操作，这里就不记了。

四、完整示例：

```C++
#include "hdfs.h"
int main(int argc, char** argv){
    //建立连接，本机系统
    hdfsFS fs = hdfsConnect("default", 0);
    const char* writePath = "test_write_file.txt";
    //打开文件
    hdfsFile writeFile = hdfsOpenFile(fs, writePath, O_WRONLY|O_CREAT, 0, 0, 0);
    if(!writeFile){
        fprintf(stderr, "Failed to open %s for writing!\n", writePath);
        exit(-1);
    }
    char* buffer = "Hello, libhdfs api!";
    tSize num_written_bytes = hdfsWrite(fs, writeFile, (void*)buffer, strlen(buffer)+1);
    //调用flush，并判断是否成功
    if(hdfsFlush(fs, writeFile)){
        fprintf(stderr, "Failed to 'Flush' %s !\n", writePath);
        exit(-1);
    }
    //关闭文件
    hdfsCloseFile(fs, writeFile);
}
```

