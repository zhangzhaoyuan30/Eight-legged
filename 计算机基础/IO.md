<!-- TOC -->

- [1 节点流和处理流](#1-节点流和处理流)
- [2 I/O模型](#2-io模型)
    - [2.1 阻塞和非阻塞？ 同步和异步？](#21-阻塞和非阻塞-同步和异步)
- [3 select、poll、epoll](#3-selectpollepoll)
- [4 零拷贝技术](#4-零拷贝技术)
    - [4.1 MMAP](#41-mmap)
    - [4.2 sendfile（Kafka）](#42-sendfilekafka)
    - [4.3 写时复制（Redis）](#43-写时复制redis)
    - [4.4 直接IO](#44-直接io)
- [5 Reactor模式](#5-reactor模式)
- [6 AIO](#6-aio)
- [7 磁盘I/O](#7-磁盘io)
- [8 FileChannel](#8-filechannel)
- [8.1 FileChannel 优势](#81-filechannel-优势)
- [8.2 FileChannel API](#82-filechannel-api)

<!-- /TOC -->
# 1 节点流和处理流  
[节点流和处理流](https://blog.csdn.net/zhangzhaoyuan30/article/details/90730314)
# 2 I/O模型
[常见I/O模型对比](https://blog.csdn.net/zhangzhaoyuan30/article/details/92067996)
## 2.1 阻塞和非阻塞？ 同步和异步？  
- 首先一个IO操作(read/write系统调用)其实分成了两个步骤：
    1. 发起IO请求
    2. 实际的IO读写(内核态与用户态的数据拷贝)
- 阻塞IO和非阻塞IO的区别在于第一步：**发起IO请求的进程是否会被阻塞**，如果阻塞直到IO操作完成才返回那么就是传统的阻塞IO，如果不阻塞，那么就是非阻塞IO。
- 同步IO和异步IO的区别就在于第二步：**实际的IO读写(内核态与用户态的数据拷贝)是否需要进程参与**，如果需要进程参与则是同步IO，如果不需要进程参与就是异步IO。
# 3 select、poll、epoll
[select、poll、epoll](https://www.cnblogs.com/aspirant/p/9166944.html) 
[IO多路复用，select、poll、epoll区别](https://juejin.cn/post/6931543528971436046)
- select：数组存储fd，轮询，打开FD数量有限1024
    - select 实现多路复用的方式是，将已连接的 Socket 都放到一个文件描述符集合，然后调用 select 函数将文件描述符集合**拷贝到内核**里，让内核来检查是否有网络事件产生，检查的方式很粗暴，就是通过**遍历**文件描述符集合的方式，当检查到有事件产生后，将此 Socket 标记为可读或可写， 接着再把整个文件描述符集合**拷贝回用户态**里，然后用户态还需要再通过**遍历**的方法找到可读或可写的 Socket，然后再对其处理。
- poll：链表存储，轮询。**没有最大文件描述符数量的限制**
- select和poll缺点：
    - 每次调用select，都需要把**fd集合从用户态拷贝到内核态**，开销大
    - 同时每次调用select都需要在内核**遍历**传递进来的所有fd
- epoll（红黑树加就绪链表）：
    - select和poll都只提供了一个函数——select或者poll函数。而epoll提供了三个函数，epoll_create，epoll_ctl和epoll_wait。
        - epoll_create是创建一个epoll对象，返回 epoll 对象的文件描述符
        - epoll_ctl对 epoll 对象进行操作，包括添加、修改和删除事件
        - epoll_wait则是等待事件的产生，返回已经发生的事件
    - 对于第一个缺点，epoll的解决方案在epoll_ctl函数中。每次注册新的事件到epoll句柄中时（在epoll_ctl中指定EPOLL_CTL_ADD），**会把所有的fd拷贝进内核的红黑树，而不是在epoll_wait的时候重复拷贝**。epoll保证了每个fd在整个过程中只会拷贝一次。
    - 对于第二个缺点， epoll 使用事件驱动的机制，内核里维护了一个链表来记录就绪事件，当某个 socket 有事件发生时，通过回调函数内核会将其加入到这个就绪事件列表中，当用户调用 epoll_wait() 函数时，只会返回有事件发生的文件描述符的个数，不需要像 select/poll 那样轮询扫描整个 socket 集合

- epoll 的水平触发（LT）和边缘触发（ET）的区别
    - LT模式（默认，select、poll只有LT）：当被监控的 Socket 上有可读事件发生时，服务器端不断地从 epoll_wait 中苏醒，直到内核缓冲区数据被 read 函数读完才结束
    - ET模式：当被监控的 Socket 描述符上有可读事件发生时，服务器端只会从 epoll_wait 中苏醒一次，即使进程没有调用 read 函数从内核读取数据，也依然只苏醒一次，因此我们程序要保证一次性将内核缓冲区的数据读取完； 
# 4 零拷贝技术
[深入剖析Linux IO原理和几种零拷贝机制的实现](https://zhuanlan.zhihu.com/p/83398714)
## 4.1 MMAP
![](../picture/Java/IO/2-mmap.jpg)
- 将一个文件或者其它设备映射到进程的地址空间，进程就可以采用指针的方式读写操作这一段内存，而系统会自动回写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必再调用read,write等系统调用函数（在用户空间和内核空间切换）。相反，内核空间对这段区域的修改也直接反映用户空间，从而可以实现不同进程间的文件共享。
- 创建虚拟区间并完成地址映射时，并没有将任何文件数据的拷贝至主存。进程真正发起对这片映射空间的访问，引发缺页异常，实现文件内容到page cache的拷贝
- 主要是节省了内核和用户缓存之间的拷贝
## 4.2 sendfile（Kafka）
总结：不经过用户空间，适用于不需要修改数据的情况
- 传统数据从文件到网络
![](../picture/微服务/kafka/2-零拷贝.png)
    1. 操作系统从磁盘读取数据到内核空间的 pagecache(DMA)
    2. 应用程序读取内核空间的pagecache到用户空间的缓冲区(CPU)
    3. 应用程序将数据从用户空间的缓冲区写回到socket buffer(内核空间)(CPU)
    4. 操作系统将数据从socket buffer复制到通过网络发送的 NIC 缓冲区(DMA)
    >DMA 的全称叫直接内存存取（Direct Memory Access），是一种允许**外围设备（硬件子系统）直接访问系统主内存**的机制。也就是说，基于 DMA 访问方式，系统主内存与硬盘或网卡之间的数据传输可以绕开 CPU 的全程调度。目前大多数的硬件设备，包括磁盘控制器、网卡、显卡以及声卡等都支持 DMA 技术。（跟零拷贝本身没关系，负责磁盘到内核page cache的拷贝）
- sendifle：
![](../picture/微服务/kafka/3-零拷贝-2.png)
- 从 pagecache 将数据由DMA直接复制到 NIC 缓冲区
- 它将内核空间（kernel space）的读缓冲区（read buffer）中对应的**数据描述信息**（内存地址、地址偏移量）记录到相应的网络缓冲区（ socket buffer）中，由 **DMA** 根据内存地址、地址偏移量将数据批量地从读缓冲区（read buffer）拷贝到网卡设备中（在Linux2.4之前还需要将数据通过CPU拷贝到socket）
## 4.3 写时复制（Redis）
1. fork()之后，kernel把父进程中所有的内存页的权限都设为read-only，然后**子进程的物理地址空间指向父进程**。当父子进程都只读内存时，相安无事
2. 当其中某个进程写内存时，CPU硬件检测到内存页是read-only的，于是触发缺页异常（page-fault），陷入kernel的一个中断例程。中断例程中，kernel就会把**触发的异常的页复制一份**，于是父子进程各自持有独立的一份，父进程的可写，子进程还是只读的

- 优点
    - 减少复制数据带来的瞬间延时（Redis）
    - 减少不必要的资源分配：有的子进程并不需要共享父进程数据
- 缺点：
    - 如果在fork()之后，父子进程都还需要继续进行写操作，那么会产生大量的分页错误(页异常中断page-fault)
## 4.4 直接IO
![](../picture/Java/IO/1-直接IO.jpg)
- 用户态直接访问硬件设备，数据直接跨过内核进行传输
- 场景：
    - 适用于随机IO比较多的场景
    - 适用于**不需要内核缓冲区**处理的应用程序，这些应用程序通常在进程地址空间**有自己的数据缓存机制**，称为自缓存应用程序，如数据库管理系统就是一个代表。
    - 由于 CPU 和磁盘 I/O 之间的执行时间差距，会造成大量资源的浪费，解决方案是**配合异步 I/O 使用**
# 5 Reactor模式
- 概念：是一种事件处理模式，用于处理**并发服务请求**。有一个Service Handler，有多个Request Handlers，service handler会将传入的请求进行解复用(demultiplex)，并分发到关联的Request Handler。
- 优点：业务处理代码和请求处理分离，从而使业务代码模块化、可复用
- 代码  
[Reactor I/O模型](https://www.xncoding.com/2018/04/05/java/reactor.html)
- 总结：
reactor多线程和主从reactor多线程的主要区别是：从reacor是一个select线程，有自己的selector
# 6 AIO
AsynchronousServerSocketChannel
- windows：IOCP
- linux

# 7 磁盘I/O
[磁盘I/O那些事](https://tech.meituan.com/2017/05/19/about-desk-io.html)
系统将文件存储到磁盘上时，按柱面、磁头、扇区的方式进行，即最先是第1磁道的第一磁头下（也就是第1盘面的第一磁道）的所有扇区，然后，是同一柱面的下一磁头，……，一个柱面存储满后就推进到下一个柱面，直到把文件内容全部写入磁盘

# 8 FileChannel
[文件操作之 FileChannel 与 mmap ](https://haobin.work/2022/09/17/IO/%E6%96%87%E4%BB%B6%E6%93%8D%E4%BD%9C%E4%B9%8B%20FileChannel%20%E4%B8%8E%20mmap/#FileChannel-API)

# 8.1 FileChannel 优势
1. 可以在文件的特定位置进行读写操作(操作文件指针)；
2. 可以直接将文件的一部分加载到内存中(mmap)；
3. 可以以更快的速度从一个通道传输文件数据到另一个通道(转入/转出其他通道)；
4. 可以锁定文件的一部分，以限制其他线程访问(文件锁)；
5. 为了避免数据丢失，我们可以强制将对文件的写入更新立即写入存储(force 刷盘)；

# 8.2 FileChannel API
|  方法   | 描述  |
|  :----:  | :----:  |
| open  | 创建 FileChannel  |
|read/write	|读写|
|force	                 |  强制刷盘|
|map	                 | mmap 内存|
|transferTo/transferFrom|	转入/转出通道|
|lock/tryLock	         |   获取文件锁|





	

