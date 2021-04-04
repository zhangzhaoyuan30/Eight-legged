<!-- TOC -->

- [1 特性](#1-特性)
- [2 局限性](#2-局限性)
- [3 DataNode 和 NameNode ?](#3-datanode-和-namenode-)
- [3.1 NameNode](#31-namenode)
    - [3.1.1 小米采用 Federation（联邦）集群](#311-小米采用-federation联邦集群)
- [3.2 DataNode](#32-datanode)
- [3.3 心跳](#33-心跳)
- [4 读写过程？](#4-读写过程)
    - [4.1 读](#41-读)
    - [4.2 写](#42-写)

<!-- /TOC -->
# 1 特性
HDFS的核心设计思想：
- 分散均匀存储 dfs.blocksize = 128M
- 备份冗余存储 dfs.replication = 3
# 2 局限性
1. 低延时数据访问
2. 大量的小文件：抛出问题：HDFS文件系统为什么不适用于存储小文件？

    任何一个文件不管有多小，**都是一个独立的数据块**，而这些**数据块的信息则是保存在元数据**中的，元数据的信息主要包括以下3部分：  
    1. 抽象目录树  
    2. 文件和数据块的映射关系，一个数据块的元数据大小大约是150byte  
    3. 数据块的多个副本存储地  
    
    而元数据的存储在磁盘（1和2）和内存中（1、2和3），而服务器中的内存是有上限的。存储小文件**占用太多NameNode** 

3. 多用户写入，修改文件。HDFS的文件只能有一个写入者，而且写操作只能在文件结尾以追加的方式进行。它不支持多个写入者，也不支持在文件写入后，对文件的任意位置的修改。
# 3 DataNode 和 NameNode ?
# 3.1 NameNode
1. 负责客户端请求（读写数据请求）的响应 
2. 维护元数据（见上）
## 3.1.1 小米采用 Federation（联邦）集群
1. 普通HDFS集群只有一个NameNode（HA的情况下有两个，一主一备），Federation集群包含横向扩展的多个NameNode，共享所有DataNodes。
2. 普通HDFS集群通过DistributedFileSystem访问，Federation集群通过接口ViewFileSystem(社区)/FederatedDFSFileSystem(小米)访问，使得多NameNode看起来就像是一个。
3. 普通HDFS集群使用hdfs协议，社区版Federation集群使用viewFs协议，小米版仍使用hdfs协议。
# 3.2 DataNode
1. 存储管理用户的文件块数据
2. 定期向 namenode 汇报自身所持有的 block 信息（通过心跳信息上报）
# 3.3 心跳
- 汇报内容：
    1. 自身DataNode的状态信息
    2. 自身DataNode所持有的所有的数据块的信息
- 最终NameNode判断一个DataNode死亡的时间计算公式：timeout = **10 * dfs.heartbeat.interval  + 2 * heartbeat.recheck.interval**
    >heartbeat.recheck.interval：为了保险起见，在10次没有心跳后，NameNode会向DataNode确认2遍，每5分钟确认一次。
# 4 读写过程？
## 4.1 读
![](../picture/大数据/hdfs/1-读.png)
1. 客户端调用FileSystem 实例的open 方法，获得这个文件对应的输入流InputStream。
2. 通过RPC 远程调用NameNode ，**获得 NameNode 中此文件对应的数据块保存位置，包括这个文件的副本的保存位置**(主要是各DataNode的地址)
3. 获得输入流之后，客户端调用read 方法读取数据。**选择最近的DataNode 建立连接**并读取数据。
4. 如果客户端和其中一个DataNode 位于同一机器(比如MapReduce 过程中的mapper 和reducer)，那么就会直接从本地读取数据
5. 到达数据块末端，关闭与这个DataNode 的连接，然后重新查找下一个数据块
6. 不断执行第2 - 5 步直到数据全部读完
7. 客户端调用close
## 4.2 写
![](../picture/大数据/hdfs/2-写.png)
1. 客户端调用 DistribuedFileSystem 的 **Create()** 方法来创建文件。
2. DistributedFileSystem 用 RPC 连接 NameNode，请求在文件系统的命名空间中创建一个新的文件；**NameNode 首先确定文件原来不存在，并且客户端有创建文件的权限，然后创建新文件**；DistributedFileSystem 返回 FSOutputStream 给客户端用于写数据。
3. 客户端调用 FSOutputStream 的 **Write() 函数**，向对应的文件写入数据。
4. 当客户端开始写入文件的时候，客户端会将文件切分成多个 packets，并在内部以数据队列“data queue（数据队列）”的形式管理这些 packets，并**向 namenode 申请 blocks，获取用来存储 replicas 的合适的 datanode 列表**，列表的大小根据 namenode 中 replication 的设定而定；队列中的分包被打包成数据包，将第一个块写入第一个Datanode，第一个 Datanode写完传给第二个节点，第二个写完传给第三节点
5. 为了保证所有 DataNode 的数据都是准确的，接收到数据的 DataNode 要向发送者发送确认包（ACK Packet）。**确认包沿着数据流管道反向而上**，当第三个节点写完返回一个ack packet给 第二个节点，第二个返回一个ack packet给第一个节点，第一个节点返回ack packet给 FSDataOutputStream对象，意思标识第一个块写完，副本数为3；然后剩余的块依次这样写
6. 不断执行第 (3)~(5)步，直到数据全部写完。
7. 调用 FSOutputStream 的 Close() 方法