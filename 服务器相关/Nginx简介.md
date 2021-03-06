### Nginx介绍：

***

Nginx和Apache都是一种web服务器，基于REST架构风格，以统一资源描述符(Uniform Resources Identifier)URI或者统一资源定位符(Uniform Resources Locator)URL作为沟通依据，通过HTTP协议提供各种网络服务。



Apache的发展时间很长，它稳定，开源，跨平台。但因为当时的环境影响，它是重量级的，不支持很高的并发量，使用多进程的方式，切换消耗大，响应速度慢等缺点。



**Nginx用C语言开发，是一个基于事件驱动，高度模块化，开源，跨平台的web服务器。同样也是反向代理服务器，负载均衡器。轻量级，支持超高的并发量。**

只支持fastCGI



***

##### 正向代理：

类似跳板机，**代理的是客户端**，（如VPN访问google，需要配置正向代理服务器，IP端口等；也可以做缓存，加快资源的访问。）

##### 反向代理：

代理的是服务器端，且过程**对客户端透明**。如外网访问百度，先转发到内网，在发到百度的服务器。

![正向代理和反向代理](/resources/正向代理和反向代理.jpg)



***

##### Nginx作为反向代理服务器：

其实一定程度上是Nginx支持一定的**负载均衡算法**，将请求**分发**到不同服务器上：

- **weight轮询(默认)**：接收到的请求按照顺序逐一分配到不同的后端服务器，即使在使用过程中，某一台后端服务器宕机，Nginx会自动将该服务器剔除出队列，请求受理情况不会受到任何影响。 这种方式下，可以给不同的后端服务器设置一个权重值(weight)，用于调整不同的服务器上请求的分配率；权重数据越大，被分配到请求的几率越大；该权重值，主要是针对实际工作环境中不同的后端服务器硬件配置进行调整的。
- **ip_hash**：每个请求按照发起客户端的ip的hash结果进行匹配，这样的算法**下一个固定ip地址的客户端总会访问到同一个后端服务器**，这也在一定程度上解决了集群部署环境下session共享的问题。
- **url_hash**：按照访问的url的hash结果分配请求，**每个请求的url会指向后端固定的某个服务器**，可以在Nginx作为静态服务器的情况下提高缓存效率。同样要注意Nginx默认不支持这种调度算法，要使用的话需要安装Nginx的hash软件包。
- **fair**：智能调整调度算法，动态的根据后端服务器的请求处理到响应的时间进行均衡分配，响应时间短处理效率高的服务器分配到请求的概率高，响应时间长处理效率低的服务器分配到的请求少；结合了前两者的优点的一种调度算法。但是需要注意的是Nginx**默认不支持fair算法**，如果要使用这种调度算法，请安装upstream_fair模块。



***

##### Nginx架构：

![Nginx架构](/resources/Nginx架构.png)



**进程模型：**一个master进程和多个worker进程，默认多进程模式，也可多线程。

- worker进程间平等竞争，一个worker进程处理一个请求，互不干扰。

- 当有worker进程挂了，master进程会fork出新的worker进程。



**master进程：**读取并验证配置文件，nginx.conf；管理worker进程。

**worker进程：**每个worker进程维护一个线程（避免切换的开销），来处理连接和请求。worker进程的数量通常由配置文件确定，一般小于CPU的核数（避免进程上下文切换）。



**Nginx支持热部署：**nginx.conf修改后，不需停止，不需中断请求，直接可以使配置生效。

- 是由worker进程完成的，当nginx.conf改变后，worker进程会根据新的配置fork出新的worker进程。
- 原先的worker进程会继续执行完毕当前的连接请求，完成后被kill掉。



***

##### 处理连接的步骤：

1. master进程建listen的socket（listenfd），而后fork出worker进程
2. worker进程在读事件前要先争抢accept_mutex
3. 抢到后注册listenfd读事件，调用accept接受连接
4. 处理连接



***

##### Nginx的事件驱动模型：

Nginx采用的是同步非阻塞的epoll方式处理IO。

epoll可以监控多个事件是否准备完毕，如果OK返回准备好的socket，worker进程从epoll的就绪队列中循环处理准备好的socket。

提供CPU亲缘性绑定：某一个进程绑定在某个核上，**不会因进程切换而导致Cache的失效。**



***

##### Nginx基础概念：

connection：对tcp连接的封装（socket，读事件，写事件）；另外也用作对http请求处理。

ngx_connection_t：这个结构体分装了连接。

ngx_http_request_t：来**保存解析请求**和**输出响应数据**。

ngx_http_handler：这个函数真正**处理**数据。



keepalive：支持长连接；

- 长连接：一个连接上有多个http请求，不用每次都建立连接。
- 但需要确定每个请求的header和body的长度（content_length）。
- 会设置等待时间，如果长时间不发数据。不会一直等待。



***

##### 负载均衡和虚拟主机：

负载均衡：一个应用部署到多台服务器上，一个域名解析到不同地址。

虚拟主机：一台服务器上部署多个应用（访问量比较小时），几个域名解析到同一地址。

- 如何区分？与Http请求的请求头中的host字段相匹配。