<!-- TOC -->

- [1 EventLoop](#1-eventloop)
- [2 Netty 启动过程梳理](#2-netty-启动过程梳理)
- [2 Netty 接受请求过程梳理](#2-netty-接受请求过程梳理)
- [2 Netty 接受请求过程梳理](#2-netty-接受请求过程梳理)
- [2 Pipeline Handler HandlerContext 创建过程梳理](#2-pipeline-handler-handlercontext-创建过程梳理)
- [3 hanler自己的线程池](#3-hanler自己的线程池)
- [4 JDK 空轮训bug](#4-jdk-空轮训bug)
- [5 总结](#5-总结)
- [6 pipeline的 事件传播](#6-pipeline的-事件传播)
    - [6.1 inBound](#61-inbound)
    - [6.2 outBound](#62-outbound)
    - [6.3 异常事件](#63-异常事件)
- [7 unsafe](#7-unsafe)

<!-- /TOC -->

---
[Netty](https://dongzl.github.io/netty-handbook)

---
[Netty源码阅读笔记](https://solthx.github.io/categories/Netty/)


# 1 EventLoop
单个线程
1. select
2. 当有事件到来时，进行processSelectionKeys操作
3. 执行taskQueue中的任务，runAllTasks操作

bossGroup的EventLoop
# 2 Netty 启动过程梳理
1. 创建 2 个 EventLoopGroup 线程池数组。数组默认大小 CPU * 2，方便 chooser 选择线程池时提高性能
2. BootStrap 将 boss 设置为 group 属性，将 worker 设置为 childer 属性
3. 通过 bind 方法启动，内部重要方法为 initAndRegister 和 dobind 方法
4. initAndRegister 方法会反射创建 NioServerSocketChannel 及其相关的 NIO 的对象，pipeline，unsafe，同时也为 pipeline 初始了 head 节点和 tail 节点。
5. 在 register0 方法成功以后调用在 dobind 方法中调用 doBind0 方法，该方法会调用 NioServerSocketChannel 的 doBind 方法对 JDK 的 channel 和端口进行绑定，完成 Netty 服务器的所有启动，并开始监听连接事件。

```java
// AbstractNioChannel
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                // Force the Selector to select now as the "canceled" SelectionKey may still be
                // cached and not removed because no Select.select(..) operation was called yet.
                eventLoop().selectNow();
                selected = true;
            } else {
                // We forced a select operation on the selector before but the SelectionKey is still cached
                // for whatever reason. JDK bug ?
                throw e;
            }
        }
    }
}
```
# 2 Netty 接受请求过程梳理
[Netty源码阅读笔记（三）Netty的两种Channel](https://solthx.github.io/2020/10/15/Netty%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%AC%94%E8%AE%B0%EF%BC%88%E4%B8%89%EF%BC%89Netty%E7%9A%84%E4%B8%A4%E7%A7%8DChannel/)

总体流程：接受连接 --> 创建一个新的 NioSocketChannel --> 注册到一个 workerEventLoop 上 --> 注册 selecotRead 事件。

1. 服务器轮询 Accept 事件，获取事件后调用 unsafe 的 read 方法，这个 unsafe 是 ServerSocket 的内部类，该方法内部由 2 部分组成
2. doReadMessages 用于创建 NioSocketChannel 对象，该对象包装 JDK 的 NioChannel 客户端。该方法会像创建 ServerSocketChanel 类似创建相关的 pipeline，unsafe，config
3. 随后执行执行 pipeline.fireChannelRead 方法，**并将自己注册到一个 chooser 选择器选择的 workerGroup 中的一个 EventLoop**。并且注册一个 0，表示注册成功，但并没有注册读（1）事件

ServerBootstrapAcceptor的channelRead()

# 2 Pipeline Handler HandlerContext 创建过程梳理
1. 每当创建 ChannelSocket 的时候都会创建一个绑定的 pipeline，一对一的关系，创建 pipeline 的时候也会创建 tail 节点和 head 节点，形成最初的链表。
2. 在调用 pipeline 的 addLast 方法的时候，会根据给定的 handler 创建一个 Context，然后，将这个 Context 插入到链表的尾端（tail 前面）。
3. Context 包装 handler，多个 Context 在 pipeline 中形成了双向链表
4. 入站方向叫 inbound，由 head 节点开始，出站方法叫 outbound，由 tail 节点开始

# 3 hanler自己的线程池
[ handler 中加入线程池和 Context 中添加线程池的源码剖析](https://dongzl.github.io/netty-handbook/#/_content/chapter10?id=_108-handler-%e4%b8%ad%e5%8a%a0%e5%85%a5%e7%ba%bf%e7%a8%8b%e6%b1%a0%e5%92%8c-context-%e4%b8%ad%e6%b7%bb%e5%8a%a0%e7%ba%bf%e7%a8%8b%e6%b1%a0%e7%9a%84%e6%ba%90%e7%a0%81%e5%89%96%e6%9e%90)

# 4 JDK 空轮训bug
[JDK空轮询Bug及出现原因](https://solthx.github.io/2020/10/15/JDK%E7%A9%BA%E8%BD%AE%E8%AF%A2Bug%E5%8F%8A%E5%87%BA%E7%8E%B0%E5%8E%9F%E5%9B%A0/)

[当 Thrift 遇到 JDK Epoll Bug](https://blog.csdn.net/xiao_hao_ge/article/details/77123764)

[JDK空轮询Bug出现原因及解决方法](https://solthx.github.io/2020/10/15/JDK%E7%A9%BA%E8%BD%AE%E8%AF%A2Bug%E5%8F%8A%E5%87%BA%E7%8E%B0%E5%8E%9F%E5%9B%A0/)
# 5 总结
thrift 里 selectionKey attach的是 frameBuffer，frameBuffer持有SocketChannel

netty 里 selectionKey attach的是 nioSocketChannel

# 6 pipeline的 事件传播
```java
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    // 在初始的时候创建tail，head这两个节点
    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```
责任链模式
## 6.1 inBound
- head -> tail
## 6.2 outBound
- tail -> head
- Pipline的fireXxx是从Tail节点开始，向Head方向传播， 当传播到Head节点的时候，会基于Unsafe对象来处理最终的出站操作
## 6.3 异常事件
- 触发异常的当前节点 -> Tail
- 不管是InBound节点还是OutBound节点，都会进入（不像InBound传播只会进入InBound，OutBound只会进入OutBound）

# 7 unsafe
Unsafe定义了实际IO处理的接口，它的含义不是说它的方法是不安全的，而是说它的接口是给框架本身调用的，不要暴露给业务层调用。

Unsafe的最底层实现类采用了模板方法模式
- NioMessageUnsafe绑定到了NioServerSocketChannel
- NioByteUnsafe绑定到NioSocketChannel

最终的IO读写方法实现在NioServerSocketChannel和NioByteUnsafe中，调用了Java的ServerSocketChannel和SocketChannel来实现。