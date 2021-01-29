<!-- TOC -->

- [1节点流和处理流](#1节点流和处理流)
- [2I/O模型](#2io模型)
- [3 select、poll、epoll](#3-selectpollepoll)
- [4 零拷贝技术](#4-零拷贝技术)
- [5 Reactor模式](#5-reactor模式)

<!-- /TOC -->
# 1节点流和处理流  
[节点流和处理流](https://blog.csdn.net/zhangzhaoyuan30/article/details/90730314)
# 2I/O模型
[常见I/O模型对比](https://blog.csdn.net/zhangzhaoyuan30/article/details/92067996)
- 阻塞和非阻塞？ 同步和异步？  
    - 首先一个IO操作(read/write系统调用)其实分成了两个步骤：
        1. 发起IO请求
        2. 实际的IO读写(内核态与用户态的数据拷贝)
    - 阻塞IO和非阻塞IO的区别在于第一步：**发起IO请求的进程是否会被阻塞**，如果阻塞直到IO操作完成才返回那么就是传统的阻塞IO，如果不阻塞，那么就是非阻塞IO。
    - 同步IO和异步IO的区别就在于第二步：**实际的IO读写(内核态与用户态的数据拷贝)是否需要进程参与**，如果需要进程参与则是同步IO，如果不需要进程参与就是异步IO。
# 3 select、poll、epoll
[select、poll、epoll](https://www.cnblogs.com/aspirant/p/9166944.html)  
[Linux IO模式及 select、poll、epoll详解](https://www.cnblogs.com/aspirant/p/9166944.html)  
- select：数组存储fd，轮询
- poll：链表存储，轮询。**没有最大文件描述符数量的限制**
- select和poll缺点：
    - 每次调用select，都需要把**fd集合从用户态拷贝到内核态**，开销大
    - 同时每次调用select都需要在内核**遍历**传递进来的所有fd
- epoll：
    - select和poll都只提供了一个函数——select或者poll函数。而epoll提供了三个函数，epoll_create，epoll_ctl和epoll_wait。epoll_create是创建一个epoll句柄；epoll_ctl是注册要监听的事件类型；epoll_wait则是等待事件的产生。
    - 对于第一个缺点，epoll的解决方案在epoll_ctl函数中。每次注册新的事件到epoll句柄中时（在epoll_ctl中指定EPOLL_CTL_ADD），**会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝**。epoll保证了每个fd在整个过程中只会拷贝一次。
    - 对于第二个缺点，epoll的解决方案不像select或poll一样每次都把current进程轮流加入fd对应的设备等待队列中，而只在epoll_ctl时把current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，**而这个回调函数会把就绪的fd加入一个就绪链表**）。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd（利用schedule_timeout()实现睡一会，判断一会的效果，和select实现中的第7步是类似的）。
- 总结：
    1. select，poll实现需要自己不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要调用epoll_wait不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是**设备就绪时，调用回调函数，把就绪fd放入就绪链表**中，并唤醒在epoll_wait中进入睡眠的进程。虽然都要睡眠和交替，**但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的时候只要判断一下就绪链表是否为空就行了**，这节省了大量的CPU时间。这就是回调机制带来的性能提升。
    2. select，poll每次调用都要把fd集合从用户态往内核态拷贝一次，并且要把current往设备等待队列中挂一次，而epoll只要一次拷贝，而且把current往等待队列上挂也只挂一次（在epoll_wait的开始，注意这里的等待队列并不是设备等待队列，只是一个epoll内部定义的等待队列）。这也能节省不少的开销。 
# 4 零拷贝技术
[深入剖析Linux IO原理和几种零拷贝机制的实现](https://juejin.cn/post/6844903949359644680)
# 5 Reactor模式
- 概念：是一种事件处理模式，用于处理发送到一个service handler的并发服务请求。service handler会将传入的请求进行解复用(demultiplex)，并分发到关联的请求处理器。
- 优点：业务处理代码和请求处理分离，从而使业务代码模块化、可复用
- 代码  
[Reactor I/O模型](https://www.xncoding.com/2018/04/05/java/reactor.html)
