### Hadoop Pipes原理与实现：

Hadooop的设计目标是实现一个通用的分布式计算平台，跨平台和兼容性是很重要的，Hadoop的实现是使用Java语言，但用户开发时可以选择其他语言进行开发。

Hadoop提供了Streaming编程框架和Pipes接口：

- Streaming编程框架**通过标准输入/输出** 作为媒介和Hadoop框架交换数据。
- Pipes接口针对C/C++，**通过Socket**和Hadoop交换数据并通信



***

#### Pipes原理浅析：

Hadoop Pipes接口则针对C/C++语言**通过Socket（媒介）**让用户的C/C++程序进程空间和Hadoop的Java框架，进行交互。

在Pipes框架中通过Java将用户的C++程序封装成为MapReduce的任务作业，然后提交到集群运行并进行监控。![HadoopPipes原理示意图](/resources/HadoopPipes原理示意图.png)

可以看出来，真正处理程序是C/C++的，但数据split是从Java环境中进行的，产生<k,v>以socket的方式传到C/C++环境中处理的。

同理，处理结束后，再传回ReduceTask中，相当于**每次都经过通信，间接执行**。



#### Pipes实现架构：

在用户C++进程程序和Hadoop框架通过socket通信的工程中，**Hadoop框架可以看成服务端**，用户程序可以看成客户端。

实现上，服务端MapTask/ReduceTask其实是对用户使用C++Pipes接口编写的Mapper/Reducer类的一个Java端封装。

**运行时，用户的程序在C++进程空间以<u>【独立进程】</u>的方式启动，而数据交互式通过Socket来传输的。**

![HadoopPipes实现架构](/resources/HadoopPipes实现架构.png)

可以看到，分层架构。虚线上是HadoopJava空间，下面是用户C++进程空间。注意其中的DownwardProtocol和UpwardProtocol的相对关系。

HadoopPipes框架中有两个进程空间：Pipes本身的Java空间，用户的C++进程空间。



```java
//PipesMapRunner.java中的run()方法
//所以在运行时，也需要启动两个进程，并且相互调用。

```

