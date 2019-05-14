### Hadoop Pipes原理与实现：

Hadooop的设计目标是实现一个通用的分布式计算平台，跨平台和兼容性是很重要的，Hadoop的实现是使用Java语言，但用户开发时可以选择其他语言进行开发。

Hadoop提供了Streaming编程框架和Pipes接口：

- Streaming编程框架**通过标准输入/输出** 作为媒介和Hadoop框架交换数据。
- Pipes接口针对C/C++，**通过Socket**和Hadoop交换数据并通信



***

#### Pipes原理浅析：

