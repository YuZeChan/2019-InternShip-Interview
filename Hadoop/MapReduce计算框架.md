### MapReduce计算框架：

​	Hadoop MapReduce是一个使用简捷的软件框架，是Google云计算模型MapReduce的Java开源实现，基于它写出的程序可以运行在由上千台普通机器组成的大型集群系统中，并以一种可靠的、容错的方式并行处理上T级别的数据集。

​	过程：输入数据集分成若干块；Map任务完全并行处理（数据块切分，机器数众多）；将输出排序后交由Reduce任务处理；将输出结果储存在文件系统中。

​	**<u>MapReduce框架和分布式文件系统是运行在一组相同的节点上，即计算节点和存储节点通常在一起。（考虑调度的速度）</u>**

​	MapReduce框架也是主从结构，单一JobTracker（负责调度一个作业的所有任务，监控，任务重启等等）和多个TaskTracker（任务的执行）节点共同组成。

​	

***

#### MapReduece模型：



##### MapReduce编程模型：

Map函数：处理输入的键值对，并产生一组中间的键值对

MapReduce框架收集所有键值相同的中间键值对（迭代器来传递），并发送给Reduce函数处理

Reduce函数：处理中间键值对，相关的中间键值对合并，形成较小的集合

![MapReduce编程模型](/resources/MapReduce编程模型.png)



##### MapReduce实现原理：

- **步骤1：**把输入文件分成M块（每一块大小Hadoop默认是64M，可修改）。
- **步骤2：**master选择空闲的执行者worker节点，把总共M个Map任务和R个Reduce任务分配给他们，如上图中（2）所示。
- **步骤3：**一个分配了Map任务的worker读取并处理输入数据块。从数据块中解析出key/value键值对，把他们传递给用户自定义的Map函数，由Map函数生成并输出中间key/value键值对，暂时缓存在内存中，如上图中（3）所示。
- **步骤4：**缓存中的key/value键值对通过分区函数分成R个区域，之后周期性地写入本地磁盘上。并把本地磁盘上的存储位置回传给master，由master负责把这些存储位置传送给Reduce worker，如上图中（4）所示。
- **步骤5：**当Reduce worker接收到master发来的存储位置后，使用RPC协议从Map worker所在主机的磁盘上读取数据。在获取所有中间数据后，通过对key排序使得相同具有key的数据聚集在一起。如上图中（5）所示。
- **步骤6：**Reduce worker程序对排序后的中间数据进行遍历，对每一个唯一的中间key，Reduce worker程序都会将这个key对应的中间value值的集合传递给用户自定义的Reduce函数，完成计算后输出文件（每个Reduce任务产生一个输出文件）。如上图中（6）所示。

![MapReduce实现原理图](/resources/MapReduce实现原理图.jpg)