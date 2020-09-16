---
layout: post
title: 你真的了解Netty中@Sharable？
category: java
excerpt: 深度分析@Sharable注解作用
tags: [java]
--- 


# 一、前言
Netty 是一个可以快速开发网络应用程序的基于事件驱动的异步 网络通讯 框架，它大大简化了 TCP 或者 UDP 服务器的网络编程。Netty 的应用还是比较广泛的，比如阿里巴巴开源的 Dubbo 和 Sofa-Bolt等 框架底层网络通讯都是基于 Netty 来实现的。Netty的设计是精妙的，其中每个设计点都值得我们去深思，本节我们来看看Netty中@Sharable的设计哲学

# 二、Netty基础快速回顾
如果你熟悉netty请直接看第三节

io.netty.channel.Channel 是 Netty 框架自己定义的一个通道接口，Netty 实现的客户端 NIO 套接字通道是 NioSocketChannel，提供的服务器端 NIO 套接字通道是 NioServerSocketChannel。

*   NioSocketChannel：客户端套接字通道，内部管理了一个 Java NIO 中的 java.nio.channels.SocketChannel 实例，用来创建 SocketChannel 实例和设置该实例的属性，并调用 Connect 方法向服务端发起 TCP 链接等。

*   NioServerSocketChannel：服务器端监听套接字通道，内部管理了一个 Java NIO 中的 java.nio.channels.ServerSocketChannel 实例，用来创建 ServerSocketChannel 实例和设置该实例属性，并调用该实例的 bind 方法在指定端口监听客户端的链接。

*   Channel 与 socket 的关系：在Netty中 Channel 有两种，对应客户端套接字通道 NioSocketChannel，内部管理 java.nio.channels.SocketChannel 套接字，对应服务器端监听套接字通道 NioServerSocketChannel，其内部管理自己的 java.nio.channels.ServerSocketChannel 套接字。也就是 Channel 是对 socket 的装饰或者门面，其封装了对 socket 的原子操作。

*   EventLoopGroup：Netty 之所以能提供高性能网络通讯，其中一个原因是因为它使用 Reactor 线程模型。在netty中每个 EventLoopGroup 本身是一个线程池，其中包含了自定义个数的 NioEventLoop，每个 NioEventLoop 是一个线程，并且每个 NioEventLoop 里面持有自己的 selector 选择器。

在 Netty 中客户端持有一个 EventLoopGroup 用来处理网络 IO 操作，在服务器端持有两个 EventLoopGroup，其中 boss 组是专门用来接收客户端发来的 TCP 链接请求的，worker 组是专门用来具体处理完成三次握手的链接套接字的网络 IO 请求的。

*   Channel 与 EventLoop 的关系：Netty 中 NioEventLoop 是 EventLoop 的一个实现，每个 NioEventLoop 中会管理自己的一个 selector 选择器和监控选择器就绪事件的线程；每个 Channel 只会关联一个 NioEventLoop；

当 Channel 是客户端通道 NioSocketChannel 时候，会注册 NioSocketChannel 管理的 SocketChannel 实例到自己关联的 NioEventLoop 的 selector 选择器上，然后 NioEventLoop 对应的线程会通过 select 命令监控感兴趣的网络读写事件；

当 Channel 是服务端通道 NioServerSocketChannel 时候，NioServerSocketChannel 本身会被注册到 boss EventLoopGroup 里面的某一个 NioEventLoop 管理的 selector 选择器上，而完成三次握手的链接套接字是被注册到了 worker EventLoopGroup 里面的某一个 NioEventLoop 管理的 selector 选择器上；

需要注意是多个 Channel 可以注册到同一个 NioEventLoop 管理的 selector 选择器上，这时候 NioEventLoop 对应的单个线程就可以处理多个 Channel 的就绪事件；但是每个 Channel 只能注册到一个固定的 NioEventLoop 管理的 selector 选择器上。

*   ChannelPipeline：Netty 中的 ChannelPipeline 类似于 Tomcat 容器中的 Filter 链，属于设计模式中的责任链模式，其中链上的每个节点就是一个 ChannelHandler。在 netty 中每个 Channel 有属于自己的 ChannelPipeline，对从 Channel 中读取或者要写入 Channel 中的数据进行依次处理, 如下图是 netty 源码里面的一个图：


![image.png](/assets/images/2020/nettyshare-1.png)


需要注意一点是虽然每个 Channel（更底层说是每个 socket）有自己的 ChannelPipeline，但是每个 ChannelPipeline 里面可以复用通一个 ChannelHandler(也就是标注了@Sharable注解的handler可以被复用)。

# 三、ChannelHandler
上节我们提到每个 Channel（更底层说是每个 socket）有自己的 ChannelPipeline，每个 ChannelPipeline里面管理者一系列的ChannelHandler。

正常情况下每个 Channel自己的 ChannelPipeline管理的同一个ChannelHandler Class对象的实例都是直接new的一个新实例，也就是原型模式，而不是单例模式。
![image.png](/assets/images/2020/nettyshare-2.png)


如上图当我们启动Netty服务端时候，会设置childHander，这个childHander会当服务器接受到完成TCP三次握手链接的时候给当前完成握手的Channel通道创建一个ChannelPipeline，并且创建一个EchoServerHandler的实例加入到Channel通道的ChannelPipeline。也就是说服务器接受的所有Channel对应的ChannelPipeline里面管理者自己的EchoServerHandler实例，而不是同一个。

但是有时候我们却想让不同Channel对应的ChannelPipeline里面管理同一个EchoServerHandler实例，比如为了全局的一些统计信息，既然上面说当服务器接受到完成TCP三次握手链接的时候给当前完成握手的Channel通道创建一个ChannelPipeline，并且创建一个EchoServerHandler的实例加入到Channel通道的ChannelPipeline，那么我们创建一个单例的EchoServerHandler传递给childHandler是不是就可以了？我们修改上面代码如下：

![image.png](/assets/images/2020/nettyshare-3.png)

这样当服务器接受到完成TCP三次握手链接的时候给当前完成握手的Channel通道创建一个ChannelPipeline，并且添加同一个EchoServerHandler的实例到对应管道。

启动上面代码，然后客户端发起多个链接时候，会有下面结果：
![image.png](/assets/images/2020/nettyshare-4.png)

这是因为我们的EchoServerHandler没有被添加@Sharable注解，现在添加如下：
![image.png](/assets/images/2020/nettyshare-5.png)

再次运行就OK了。

具体检查的代码是DefaultChannelPipeline类的checkMultiplicity的checkMultiplicity方法：
![image.png](/assets/images/2020/nettyshare-6.png)

![image.png](/assets/images/2020/nettyshare-7.png)


可知当添加到不同管线的是不同的实例时候，不同连接在检查时候h.added总是返回的false，所以不会抛出异常。当添加到不同管线的是同一个实例时候，由于是单例，所以第一个连接会把单例的对象的added设置为了true，所以其他连接检查时候发现没有添加@Sharable注解并且当前added为true则会抛出异常。
# 四、总结
正常情况下同一个ChannelHandler,的不同的实例会被添加到不同的Channel管理的管线里面的，但是如果你需要全局统计一些信息，比如所有连接报错次数（exceptionCaught）等，这时候你可能需要使用单例的ChannelHandler，需要注意的是这时候ChannelHandler上需要添加@Sharable注解。
