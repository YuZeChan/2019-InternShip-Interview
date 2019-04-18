### Nginx和Apache比较：

***

本质区别：

Apache：使用同步多进程模式，一个进程对应一个连接。

Nginx：使用基于事件驱动模式，一个进程处理多个连接。



Apache的三种工作模式：

- **prefork（默认）：多进程，每个请求用一个进程响应，用select机制通知。**
- worker：多线程，每个请求用一个线程响应，一个进程生成多个线程，select机制。
- event：异步IO模型？每个进程或线程响应多个用户的请求，基于事件驱动（callback和epoll）

![Nginx和Apache的对比](/resources/Nginx和Apache的对比.jpg)

Nginx相比Apache来说：**使用更少的资源，支持更多的并发连接**。

- **启动容易，可长时间不间断运行，支持热部署，更新不需重启，处理静态资源快。**



Http服务器本质上也是一种**应用程序**，它通常**运行**在服务器（主机）上，**绑定**服务器的IP地址，**监听**某个tcp端口，来接收和处理Http请求。这样客户端（IE，Chrome，Firefox……）就能通过Http协议来获取服务器上的网页（HTML），文档（PDF），音频（MP4），视频（Mov）等资源。



所以Nginx和Apache其实都是http server，本身只支持静态资源。但是可以通过其他模块来动态支持（PHP，Python……）

Apache社区中的子项目Tomcat服务器（运行在JVM上，是servlet的容器），是一种application server，能够动态生成资源，并返回到客户端。

Nginx 和 Tomcat可以搭档使用实现动静态资源的分离。





