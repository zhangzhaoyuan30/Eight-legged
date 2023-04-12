<!-- TOC -->

- [1 Frame](#1-frame)
- [2 流控机制](#2-流控机制)
- [3 API](#3-api)
- [4 Server API](#4-server-api)
- [5 Client API](#5-client-api)
- [5 和Netty协作](#5-和netty协作)
- [6 Client线程模型](#6-client线程模型)
- [7 server 线程模型](#7-server-线程模型)
- [8 gRPC连接状态机](#8-grpc连接状态机)
- [9 流式](#9-流式)
- [10 客户端类型](#10-客户端类型)
- [11 源码](#11-源码)
    - [11.1 启动](#111-启动)
    - [11.2 服务端处理请求](#112-服务端处理请求)
    - [11.2 服务端处理请求](#112-服务端处理请求)
        - [11.2.1 accept](#1121-accept)
        - [11.2.2 read](#1122-read)
- [12 客户端猜测实现（todo）](#12-客户端猜测实现todo)
- [13 连接池](#13-连接池)

<!-- /TOC -->
---
[GRPC Java源码解析](https://blog.csdn.net/fs3296/article/details/103608383)

[gRPC Java 服务端实现简析](https://www.modb.pro/db/168765)

---

# 1 Frame
- SETTINGS帧：用于设置连接级别的配置，协议协商与流控窗口变更相关依赖于它
- PING帧：用于心跳保持，gRPC KeepAlive就是利用这个帧实现
- GOAWAY帧：发起关闭连接请求时候使用，通常是连接达到IDLE状态之后
- RST_STREAM帧：关闭当前流的帧，gRPC会经常用到，特别是当错误发生的时候
- WINDOW_UPDATE帧：流控帧，gRPC的流控实现依赖于它
- DATA帧：实际传输数据帧，如果数据发送完毕会带END_STEAM标识
- HEADER帧：消息头帧，通常是一个请求开始帧，基于此帧创建请求初始化相关内容
# 2 流控机制
- 流控机制同时作用于单个Stream和整个连接，对于Stream而言只作用于Data帧（保证不影响重要帧）
- 初始窗口大小都是65535字节，发送端需要严格遵守接收端的窗口限制，连接序言中可以通过SETTINGS帧设置SETTINGS_INITIAL_WINDOW_SIZE来指定Stream的初始窗口大小，但是无法配置连接级别的初始窗口大小（gRPC支持设置初始窗口的大小）
- gRPC通过WINDOW_UPDATE帧来实现流量控制，但相关说明如下
    - gRPC支持自定义流控但默认没开启，仍处于VisableForTesting阶段；使用Netty内置的默认流控功能：基本思路就是当已经处理过的数据超过窗口一半是就发送WINDOW_UPADTE来更新窗口
    - gRPC默认流控初始窗口大小是1M
    - gRPC如果开启KeepAlive功能那么Ping也会占用窗口大小
    - gRPC默认给每个Stream分配的字节数是16K
    - gRPC内部支持的自动跟手动流控，只是针对gRPC本身而言，这里流控的含义是Client何时发起从buffer中读取需要数据的请求，类似于一种应用层级的流控，默认是自动的，当有特殊需求时可以开启手动控制，但是比较复杂而且容易出错，不推荐使用
# 3 API
gRPC整体设计思路依附于HTTP/2协议，而HTTP/2是一个双向流协议，因此gRPC在API设计上也采用了Stream的方式。通常意义上（Unary）一次gRPC请求可以理解为一个Stream创建到销毁的过程，框架内部封装了Stream的整个生命周期，暴露给用户的是何时发送数据以及何时完成发送，因此gRPC通过暴露给业务Listener的方式来完成这项工作。看到这里你会问："为什么Unary模式也要暴露StreamAPI给用户？"，答案很简单，就是为了API设计上的一致性，毕竟是框架。因为是双向流协议，因此可以分成如下四种调用模式：
- Unary模式：即请求响应模式
- Client Streaming模式：Client发送多次，Server回复一次
- Server Streaming模式：Client发送一次，Server发送多次
- 双向 Streaming模式：Client/Server都发送多次
# 4 Server API
Server端我们能接触到API包括：
1. StreamObserver
 ServerCall：封装从远程客户端收到的单个调用
3. ServerCall.Listener

一次unary调用

1. Listener.onMessage被执行，代表服务端已经收到客户端一个完整的Message
 Listener.onHalfClose被执行，服务端即将执行我们实现的实际方法，譬如上述的margin方法
3. StreamObserver.onNext被调用，代表服务端完成消息处理并将结果返回
4. ServerCall.sendHeaders被执行，结果返回时会优先触发Header发送，符合预期
5. SeverCall.sendMessage被执行，通过该方法将返回结果投递出去
6. StreamObserver.onCompleted/onError被调用，代表业务标识本次请求服务端处理完成或者失败
7. ServerCall.close被调用，具体成功失败取决于6
8. Listener.onComplete/onCancel，具体成功失败取决于6
# 5 Client API
1. StreamObserver
 ClientCall：一次远程方法的调用的实例
3. ClientCall.Listener

一次unary调用

1. 通过用户XXXGrpc.newFutureStub获取的XXXFutureStub调用方法发起请求
 ClientCall.start被调用，客户端创建当前请求相关的Stream及其上下文
3. ClientCall.sendMessage被调用，数据被发送到Buffer中等待被投递
4. ClientCall.halfClose被调用，代表Client完成本次请求的数据发送不会再进行更多数据发送
5. Listener.onHeaders被调用，代表服务端开始回数据，首先获取到Headers
6. Listener.onMessage被调用，收到完整的返回数据
7. Listener.onClose被调用，代表本次请求结束，最终用户拿到的ListenableFuture会被set，然后用户获取到本次的结果

# 5 和Netty协作
1. 依赖Netty实现的HTTP/2协议的封装，通过Listener机制监听HTTP/2的数据报文事件，完成网络相关处理
 Reactor IO模型的依赖，Nio/Epoll
3. 依赖Netty的ByteBuf完成流数据在内部中的缓存与流转

# 6 Client线程模型
1. Caller线程：业务当前线程
2. Worker线程（grpc-default-executor）：请求处理与响应回调线程
3. IO线程（grpc-nio-worker-ELG）：使用Netty的ELG

以futureClient为例，一次client请求的线程切换如下图：
![](./pic/krpc/7-client-netty)
整体一次执行流程描述如下：

1. 创建链接，通常是Delayed的模式，第一次创建Stream时候之前才会创建连接
 创建Stream，代表发起一次请求，在Caller Thread完成
3. 发送Header，将Header Frame投递到WriteQueue，Call Thread完成，ELG异步消费WriteQueue
4. 发送Message，将Message Frame投递到WriteQueue，Call Thread完成，ELG异步消费WriteQueue
5. 接收Header，这里完成一次从ELG -> Worker线程池的切换，**需要注意的是如果创建Client时我们指定了directExecutor模式，那么将统一由ELG线程完成**
6. 接收Message，这里完成一次从ELG -> Worker线程池的切换，需要注意的是如果创建Client时我们指定了directExecutor模式，那么将统一由ELG线程完成
7. 接收关闭Stream请求，线程切换同5，6，最终会回调Future的set方法
8. Caller Thread通过Future.get获取到结果

# 7 server 线程模型
1. Worker线程（grpc-default-executor）：Server启动时候指定
2. IO线程（grpc-nio-worker-ELG）：使用Netty的ELG

![](./pic/krpc/8-server-netty)

1. 接收Client发送的创建Stream的请求，ELG -> Worker线程切换，最终创建一个ServerCall及其Listener
 接收Client发送的Msg，ELG -> Worker线程切换，反序列化保存
3. Client完成发送，ELG -> Worker线程切换，回调Server业务逻辑
4. 处理完成发送Header，扔到WriteQueue，ELG异步消费发送Header Frame到Client
5. 处理完成发送Message，扔到WriteQueue，ELG异步消费发送Message Frame到Client
6. 完成全部消息发送，标记endofframe，对于unary而言跟5合成一步，扔到WriteQueue，ELG异步消费发送Message Frame到Client
7. Client发送RstStream请求，并不一定会发送，取决于配置，回调Listener

# 8 gRPC连接状态机
1. CONNECTING：代表Channel正在初始化建立连接，流程会涉及DNS解析、TCP建连、TLS握手等
 READY：成功建立连接，包括HTTP2协商，代表Channel可以正常收发数据
3. TRANSIENT_FAILURE：建连失败或者CS之间网络问题导致，Channel最终会重新发起建立连接请求，gRPC提供了一套backoff Retry机制来保证不会出现重连风暴
4. IDLE：Channel中长期没有请求或收到HTTP2的GO_AWAY信号会进入此状态，此时CS之间连接已经断开，一旦新请求发起会转移到CONNECTING。主要目的是保障SERVER连接数不会太大造成压力
5. SHUTDOWN：Channel已经关闭，状态不可逆，新请求会立即失败掉，排队的请求会继续处理完
![](./pic/krpc/9-状态机)

# 9 流式
客户端流式和双向流式，需要使用异步客户端。服务端的参数和返回值都是流，但是客户端流式服务端的onNext和onCompleted一起调

SQL QPI是不是流式rpc
# 10 客户端类型
1. 异步：Stub
2. 阻塞：BlockingStub
3. FutureStub

# 11 源码
## 11.1 启动
- 快手DefaultServiceBootstrap
- ServiceBootstrap
```java
public void start(ServerListener serverListener) throws IOException {
    listener = checkNotNull(serverListener, "serverListener");

    final ServerBootstrap b = new ServerBootstrap();
    b.option(ALLOCATOR, Utils.getByteBufAllocator(forceHeapBuffer));
    b.childOption(ALLOCATOR, Utils.getByteBufAllocator(forceHeapBuffer));
    b.group(bossExecutor, workerGroup);
    b.channelFactory(channelFactory);
    // For non-socket based channel, the option will be ignored.
    b.childOption(SO_KEEPALIVE, true);

    if (channelOptions != null) {
      for (Map.Entry<ChannelOption<?>, ?> entry : channelOptions.entrySet()) {
        @SuppressWarnings("unchecked")
        ChannelOption<Object> key = (ChannelOption<Object>) entry.getKey();
        b.option(key, entry.getValue());
      }
    }

    if (childChannelOptions != null) {
      for (Map.Entry<ChannelOption<?>, ?> entry : childChannelOptions.entrySet()) {
        @SuppressWarnings("unchecked")
        ChannelOption<Object> key = (ChannelOption<Object>) entry.getKey();
        b.childOption(key, entry.getValue());
      }
    }

    b.childHandler(new ChannelInitializer<Channel>() {
      @Override
      public void initChannel(Channel ch) {

        ChannelPromise channelDone = ch.newPromise();

        long maxConnectionAgeInNanos = NettyServer.this.maxConnectionAgeInNanos;
        if (maxConnectionAgeInNanos != MAX_CONNECTION_AGE_NANOS_DISABLED) {
          // apply a random jitter of +/-10% to max connection age
          maxConnectionAgeInNanos =
              (long) ((.9D + Math.random() * .2D) * maxConnectionAgeInNanos);
        }

        NettyServerTransport transport =
            new NettyServerTransport(
                ch,
                channelDone,
                protocolNegotiator,
                streamTracerFactories,
                transportTracerFactory.create(),
                maxStreamsPerConnection,
                autoFlowControl,
                flowControlWindow,
                maxMessageSize,
                maxHeaderListSize,
                keepAliveTimeInNanos,
                keepAliveTimeoutInNanos,
                maxConnectionIdleInNanos,
                maxConnectionAgeInNanos,
                maxConnectionAgeGraceInNanos,
                permitKeepAliveWithoutCalls,
                permitKeepAliveTimeInNanos,
                eagAttributes);
        ServerTransportListener transportListener;
        // This is to order callbacks on the listener, not to guard access to channel.
        synchronized (NettyServer.this) {
          if (terminated) {
            // Server already terminated.
            ch.close();
            return;
          }
          // `channel` shutdown can race with `ch` initialization, so this is only safe to increment
          // inside the lock.
          sharedResourceReferenceCounter.retain();
          transportListener = listener.transportCreated(transport);
        }

        /**
         * Releases the event loop if the channel is "done", possibly due to the channel closing.
         */
        final class LoopReleaser implements ChannelFutureListener {
          private boolean done;

          @Override
          public void operationComplete(ChannelFuture future) throws Exception {
            if (!done) {
              done = true;
              sharedResourceReferenceCounter.release();
            }
          }
        }

        transport.start(transportListener);
        ChannelFutureListener loopReleaser = new LoopReleaser();
        channelDone.addListener(loopReleaser);
        ch.closeFuture().addListener(loopReleaser);
      }
    });
    Future<Map<ChannelFuture, SocketAddress>> bindCallFuture =
        bossExecutor.submit(
            new Callable<Map<ChannelFuture, SocketAddress>>() {
          @Override
          public Map<ChannelFuture, SocketAddress> call() {
            Map<ChannelFuture, SocketAddress> bindFutures = new HashMap<>();
            for (SocketAddress address: addresses) {
                ChannelFuture future = b.bind(address);
                channelGroup.add(future.channel());
                bindFutures.put(future, address);
            }
            return bindFutures;
          }
        }
    );
    Map<ChannelFuture, SocketAddress> channelFutures =
        bindCallFuture.awaitUninterruptibly().getNow();

    if (!bindCallFuture.isSuccess()) {
      channelGroup.close().awaitUninterruptibly();
      throw new IOException(String.format("Failed to bind to addresses %s",
          addresses), bindCallFuture.cause());
    }
    final List<InternalInstrumented<SocketStats>> socketStats = new ArrayList<>();
    for (Map.Entry<ChannelFuture, SocketAddress> entry: channelFutures.entrySet()) {
      // We'd love to observe interruption, but if interrupted we will need to close the channel,
      // which itself would need an await() to guarantee the port is not used when the method
      // returns. See #6850
      final ChannelFuture future = entry.getKey();
      if (!future.awaitUninterruptibly().isSuccess()) {
        channelGroup.close().awaitUninterruptibly();
        throw new IOException(String.format("Failed to bind to address %s",
            entry.getValue()), future.cause());
      }
      final InternalInstrumented<SocketStats> listenSocketStats =
          new ListenSocket(future.channel());
      channelz.addListenSocket(listenSocketStats);
      socketStats.add(listenSocketStats);
      future.channel().closeFuture().addListener(new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future) throws Exception {
          channelz.removeListenSocket(listenSocketStats);
        }
      });
    }
    listenSocketStatsList = Collections.unmodifiableList(socketStats);
  }
```
## 11.2 服务端处理请求
4-8-4-1-0-3-7
### 11.2.1 accept
1. 首先，构建NettyServerTransport，将其作为参数调用ServerImpl \$ ServerListenerImpl的transportCreated方法。在transportCreated中会构建ServerImpl \$ ServerTransportListenerImpl，返回抽象的ServerTransportListener。然后，将抽象的ServerTransportListener作为参数去调用新构建的NettyServerTransport的start方法；
2. 在NettyServerTransport的start方法中，它会构建一个NettyServerHandler，将之作为参数调用ProtocolNegotiator的newHandler方法，然后再将返回negotiationHandler作为委托参数构建一个WriteBufferingAndExceptionHandler，并将WriteBufferingAndExceptionHandler设置为这个连接channel的Netty处理器；

```java
public void start(ServerTransportListener listener) {
    Preconditions.checkState(this.listener == null, "Handler already registered");
    this.listener = listener;

    // Create the Netty handler for the pipeline.
    grpcHandler = createHandler(listener, channelUnused);

    // Notify when the channel closes.
    final class TerminationNotifier implements ChannelFutureListener {
      boolean done;

      @Override
      public void operationComplete(ChannelFuture future) throws Exception {
        if (!done) {
          done = true;
          notifyTerminated(grpcHandler.connectionError());
        }
      }
    }

    ChannelHandler negotiationHandler = protocolNegotiator.newHandler(grpcHandler);
    ChannelHandler bufferingHandler = new WriteBufferingAndExceptionHandler(negotiationHandler);

    ChannelFutureListener terminationNotifier = new TerminationNotifier();
    channelUnused.addListener(terminationNotifier);
    channel.closeFuture().addListener(terminationNotifier);

    channel.pipeline().addLast(bufferingHandler);
  }
```

### 11.2.2 read

io.netty.handler.codec.http2.Http2InboundFrameLogger#readFrame

io.netty.handler.codec.http2.DefaultHttp2Connection#streamMap

io.grpc.netty.shaded.io.netty.handler.codec.http2.DefaultHttp2Connection#stream 断点打在这个地方可以看到stream的机制

1. 核心是NettyServerHandler，作为NioSocketChannel 的 channelPipeline的一个handler
2. 在创建NettyServerHandler时会同时创建一个DefaultHttp2Connection，并创建Http2ConnectionDecoder和Http2ConnectionEncoder，这两个类是NettyServerHandler的父类GrpcHttp2ConnectionHandler的成员。
3. DefaultHttp2Connection中维护了一个streamMap，key是streamId，value是htttp2Stream状态机
4. 读取帧是在io.netty.handler.codec.http2.DefaultHttp2FrameReader#processPayloadState 方法中
5. 一次grpc请求过程：4set 1head 0data
    1. 4-8（一次连接）
    2. 4-1-0  
    3. 3-7
6. 在收到http2的header帧时调用NettyServerHandler的OnHeadersRead方法，调用ServerTransportListener(实现为ServerTransportListenerImpl)接口的streamCreated方法构建NettyServerStream。
7. 在 NettyServerHandler.onDataRead时会进行方法调用 io.grpc.stub.ServerCalls.UnaryServerCallHandler.UnaryServerCallListener#onHalfClose

# 12 客户端猜测实现（todo）
注册写事件，创建stream，绑定在key上，写入队列，注册写事件。写完注册读事件，读事件回调结果写到future。future提前返回。

# 13 连接池
因为grpc利用http2做多路复用，理论上不需要像thrift那样创建连接池并保证同一时刻socket只能有一个请求（client线程不安全）