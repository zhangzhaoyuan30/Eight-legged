<!-- TOC -->

- [1 CAP和BASE](#1-cap和base)
- [2 2PC、3PC](#2-2pc3pc)
    - [2.1 2PC](#21-2pc)
    - [2.2 3PC](#22-3pc)
    - [2.3 参考](#23-参考)
- [3 paxos](#3-paxos)
- [4 ZK的基本概念、设计目标？](#4-zk的基本概念设计目标)
- [5 ZK的分布式一致性特性？](#5-zk的分布式一致性特性)
    - [5.1 单一视图如何保证？](#51-单一视图如何保证)
- [6 ZAB(ZK Atomic Broadcast ZK原子消息广播协议)协议和Leader选举](#6-zabzk-atomic-broadcast-zk原子消息广播协议协议和leader选举)
    - [6.1 Leader选举](#61-leader选举)
    - [6.2 ZAB协议的任务？](#62-zab协议的任务)
    - [6.3 ZAB两种基本模式](#63-zab两种基本模式)
    - [6.4 理论算法](#64-理论算法)
    - [6.5 所以ZAB和paxos区别？](#65-所以zab和paxos区别)
- [7 原生客户端和ZKClient区别](#7-原生客户端和zkclient区别)
- [8ZK的应用？](#8zk的应用)
    - [8.1 数据发布订阅](#81-数据发布订阅)
    - [8.2 命名服务](#82-命名服务)
    - [8.3 分布式协调通知](#83-分布式协调通知)
        - [8.3.1 Master选举和分布式锁](#831-master选举和分布式锁)
    - [8.4 分布式队列和barrier](#84-分布式队列和barrier)
- [9 ZK的stat对象和版本号？](#9-zk的stat对象和版本号)
- [10 Watcher机制？](#10-watcher机制)
    - [10.1 工作机制](#101-工作机制)
    - [10.2 特性](#102-特性)
- [11 ACL？](#11-acl)
- [12 客户端原理？](#12-客户端原理)
- [13 ZK如何进行会话管理？](#13-zk如何进行会话管理)
- [14 Leader、Follower、Observer的请求处理过程？](#14-leaderfollowerobserver的请求处理过程)
    - [14.1 Leader](#141-leader)
    - [14.2 Follower](#142-follower)
    - [14.3 Observer](#143-observer)
- [15 ZK的数据与存储，及数据初始化和同步（启动过程）？](#15-zk的数据与存储及数据初始化和同步启动过程)
    - [15.1 数据与存储](#151-数据与存储)
    - [16.2 初始化和同步](#162-初始化和同步)
    - [16.3 启动过程](#163-启动过程)
- [17 ZK运维（了解）](#17-zk运维了解)
    - [17.1 配置](#171-配置)
    - [17.2 四字命令](#172-四字命令)
    - [17.3 集群高可用](#173-集群高可用)

<!-- /TOC -->
# 1 CAP和BASE
![](../picture/微服务/zk/6-CPA和BASE.png)
- 强一致性：任意时刻，所有节点中的数据是一样，一个集群如果需要对外部提供强一致性，则只要集群内部某一台服务器的数据发生了改变，那么就需要等待集群内其他服务器的**数据同步完成后**，才能正常的对外提供服务
- 一致性：每次读取都会收到最新的写入或错误响应
- 可用性：每个请求都会收到一个（非错误）响应，但**不能保证它包含最新的写操作**
- 分区容忍：尽管节点之间的网络丢弃（或延迟）了任意数量的消息，但系统仍继续运行
# 2 2PC、3PC
目的：为了保证分布式事务ACID特性的一致性算法
## 2.1 2PC
- 阶段
    - 阶段一：提交事务请求
    - 阶段二：执行事务提交
- 存在问题：
    - 阻塞
    - 保守：没有容错机制，只能依赖超时，任意一个节点是失败都会导致整个事务的失败
    - 单点故障
    - 协调者和第一个接收commit的参与者都挂了，重启后不确定处于什么状态
## 2.2 3PC
- 改善：
    - 引入canCommit阶段（减少锁定资源时间）
    - 引入阶段三**参与者**无法接收来自协调者的doCommit或abort请求会等待**超时提交**：减少2PC阻塞问题，并能在单点故障后继续达成一致（改善阻塞问题、单点问题、保守问题）
    - （能解决协调者和参与者都挂掉的情况：因为加了canCommit，所以如果在doCommit阶段协调者和一个参与者挂掉，其他参与者会已经知道第一阶段大家都已经决定要通过了，此时便可以直接通过）
- 尚未解决：
    - 参与者超时提交，协调者重新回到网络，带来新的数据不一致问题
    - 阻塞问题、单点问题、数据不一致依然存在
## 2.3 参考
[2PC和3PC中故障情况分析](https://blog.csdn.net/Lnho2015/article/details/78685503)  
# 3 paxos
- 概念：paxos是一个共识算法（区别于一致性算法）：在一个可能发生异常的分布式系统中，对某个数据的值达成一致  
[Paxos算法详解](https://zhuanlan.zhihu.com/p/31780743)
- 角色：Proposer、Acceptor、Leaner。一个进程可能充当不止一种角色
- 安全性
    - 只有被提出的提案，才能被选定
    - 只能有一个被选定
    - learners 只能获得被选定（chosen）的 value
- 算法内容
    - 通过加强上述约束（主要是第二个）获得了两个约束
    - P1：Acceptor必须接受（Accept）它所收到的第一个Proposal。
        - P1a（包含P1）：Acceptor可以接受（Accept）编号为 n 的提案(Proposal)当且仅当它没有回复过一个具有更大编号的Prepare消息。
    - P2：如果值为v的Proposal被选定（Chosen），则任何被选定（Chosen）的具有更高编号的Proposal值也一定为v
        - P2c（满足P2）：如果[n,v]被提出，那么存在一个由半数以上Acceptor组成的集合S，满足
            - S中不存在 accept 过小于编号n的Acceptor
            - 或S中所有 Acceptor accept 的编号小于n的提案，其中编号最大的值是v
- 提案产生
    - prepare 阶段
        - Proposer：选择一个提案编号n，并将Prepare请求发送给Acceptors中的一个多数派
        - Acceptor：收到Prepare消息后
            - 承诺不再回复小于n的提案
            - 如果n大于它已经回复的所有prepare消息，则Acceptor将**已经accept过的最大编号的提案**回复给Proposer
    - Accept 阶段
        - Proposer：在收到的超过一半Acceptors中取编号最大的Proposal的值，发出Accept请求。如果没有这样的值就任意选择
        - Acceptor：P1a：若未对编号大于n的Prepare请求做响应，则接受Proposal
- 活锁  
两个Proposer依次提出编号相互递增的提案
    - 解决方法：主Proposer
- Multi-Paxos
    - Basic Paxos的问题
        - 原始的Paxos算法（Basic Paxos）只能对**一个值**形成决议
        - 决议的形成至少需要**两次网络来回**，在高并发情况下可能需要更多的网络来回，极端情况下甚至可能形成活锁。
        - 因此Basic Paxos几乎只是用来做理论研究，并不直接应用在实际工程中。
    - 改进
        - 在所有Proposers中**执行一次Basic Paxos选举一个Leader**，由Leader唯一地提交Proposal给Acceptors进行表决。这样没有Proposer竞争，**解决了活锁问题**。
        - 在系统中仅有一个Leader进行Value提交的情况下，**Prepare阶段就可以跳过**，从而将两阶段变为一阶段，提高效率。 
    - 如何解决Leader崩溃？
- Fast-Paxos?
- 参考
[2PC到3PC到Paxos到Raft到ISR](https://segmentfault.com/a/1190000004474543)
# 4 ZK的基本概念、设计目标？
- 设计目标
    - **高性能**、高可用、写操作严格顺序性的分布式协调服务
    - 以读操作为主，3台读请求QPS12-13W
- 基本概念
    - 角色：Observer不参与选举和过半写
    - 会话：客户端和服务器之间的TCP长连接
        - 心跳检测
        - 发送请求，接收响应
        - 接受Watch事件通知
    - 节点
        - 持久
        - 临时：会话绑定
    - 版本：ZNode会维护一个Stat
        - version：ZNode版本
        - cversion：子节点版本
        - aversion：ACL版本
    - ACL：CREATE和DELETE针对子节点
    - ZXID：高32位是epoch，低32位自增
# 5 ZK的分布式一致性特性？
- **顺序一致性**：见[并发顺序一致性](../Java/并发.md#54-顺序一致性)客户端更新请求按顺序应用
    - 读顺序一致性：读操作不一定能读到最新的数据，但一定是按顺序的。
    - 写线性一致性：写操作是在最新的数据上进行写
- 原子性：更新操作要么成功要么失败，没有中间结果
- 单一视图：无论客户端连接的是哪个服务器，看到的数据一致，不会看到旧的视图
- 可靠性：保证一旦客户端接收到successful return code，则事务变更永久生效；对于返回失败的不保证事务的状态（可能成功或失败）
- 实时性：一定时间后能读到最新的视图，**客户端可以调用sync方法获取最新视图**
## 5.1 单一视图如何保证？
- 客户端每次请求响应，服务端都会将**最新的zxid作为消息头发送**给客户端，客户端会进行保存
- 客户端与服务器断开重连时会**发送本地保存的zxid**，服务端判断如果客户端lastZxid大于服务端zxid（说明服务端数据较老）则拒绝连接
# 6 ZAB(ZK Atomic Broadcast ZK原子消息广播协议)协议和Leader选举
## 6.1 Leader选举
- 状态变更：选举时非Observer机器为LOOKING状态
- 步骤：
    1. 每个Server 生成(myid,ZXID)投票投自己，并发给集群中所有机器
    2. 每台机器接收其他机器的选票，检验是否是本轮、是否来自LOOKING
    3. 进行选票PK，先比ZXID，再比myid。**如果需要变更**，则更新选票，再向所有机器发送
    4. 每次投票后，每台机器会进行计票，判断**是否已有一台机器接收到过半选票**。然后更新状态
- 总结：ZXID越新越有可能成为Leader
## 6.2 ZAB协议的任务？
1. 单一主进程接收并处理所有事务请求，并广播到Follower上
2. 保证顺序一致性：一个全局的变更序列被顺序应用
3. 处理崩溃恢复
## 6.3 ZAB两种基本模式 
消息广播过程类似于一个两阶段提交，但没有中断逻辑，而是加入崩溃恢复过程
- 消息广播：
    - 类似于两阶段提交，但没有中断逻辑，并且采用过半提交
    - 基于具有FIFO特性的TCP协议进行网络通信，保证消息广播过程中消息接收与发送的顺序性。Leader服务器会为每一个Follower服务器分配一个单独队列，并将事务Proposal根据FIFO发送
- 崩溃恢复：**Leader崩溃或者Leader失去和过半Follower联系，进入崩溃恢复**
    - 需要一个高效Leader选举算法
    - 并确保
        1. ZAB协议需要确保那些已经在Leader服务器上commit的事务最终被所有服务器都commit
        2. ZAB协议需要确保丢弃那些只在Leader服务器上被提出的事务
        - 综上Leader选举算法需要选举出拥有最大ZXID的Leader——一定拥有所有已提交的提案
    - 同步过程
        - 将Follower上未提交的事务发送给Follower
        - 对于Follower上应该丢弃的事务：采用ZXID(epoch,计数器)，当一个包含上个周期未提交事务的服务器启动时，其epoch不是最新，Leader会要求其回退
## 6.4 理论算法
主进程周期：启动或者崩溃恢复之后进入
- 发现
    - 选定epoch：Follower发送epoch给准Leader，准Leader在过半里面取最大的e'发送给Follower
    - Follower将epoch改为e'，并向准Leader返回ACK，并包含e'和历史事务集合Ie'
    - 选定最新历史事务集合：准Leader接收Ie'后选ZXID最大的（补齐相比Follower多数派缺失的事务）
- 同步
    - 准Leader将e'和Ie'发给Follower（各Follower再补齐相比准Leader缺失的状态）
    - 过半ACK则准Leader提交事务，成为Leader
- 广播：
    - 类似2PC的过程，但只需过半ACK就可以提交
## 6.5 所以ZAB和paxos区别？
- 为了**加快收敛速度避免活锁**引发的竞争引入了Leader角色，在正常情况下最多只有一个参与者扮演Proposer角色，其他参与者扮演Acceptor，类似multi paxos：在第一轮选举出Leader，在之后选定提案时就可以跳过Prepare阶段
- 崩溃恢复
# 7 原生客户端和ZKClient区别
1. 原生不支持递归创建
2. 原生节点内容只支持字节数组(byte[])
3. 构造时或get时不传入watcher，而是以API级别支持Watcher注册，通过注册事件监听方式进行。且**不是一次性**的，注册一次一直生效。
    - subscribeChildChanges(path,listener)：子节点新增、删除，本节点删除
    - subscribeDataChanges(path,listener)：节点数据变化、删除节点
# 8ZK的应用？
## 8.1 数据发布订阅
- 举例：机器列表、数据库配置
- 适用于：数据量小，运行时动态变化，集群中机器共享、配置一致
## 8.2 命名服务
使用顺序节点生成分布式环境下的全局唯一ID
## 8.3 分布式协调通知
- 热备：通过每台机器创建临时顺序节点，取序号最小。待命状态客户端注册子节点变更Watcher
- 冷备：也是创建临时顺序节点，如果自己不是最小删除节点，遍历下一个任务
- 心跳检测（创建临时节点）、工作进度汇报（创建临时节点并把任务进度写进去）
### 8.3.1 Master选举和分布式锁
- 排它锁：创建临时节点，没获取到锁的注册子节点变更Watcher
- 共享锁
    - 获取锁：
        - 读请求创建临时顺序节点：/shard_lock/R-seq
        - 写请求创建临时顺序节点：/shard_lock/W-seq
    - 判断读写顺序
        1. 获取所有节点
        2. 确认自己的顺序
            - 对于读请求：前面的节点都是读，则请求获取锁，否则等待
            - 对于写请求：不是最小序号就等待
        3. 注册Watcher
            - 读请求：向比自己序号小的最后一个写请求节点注册
            - 写请求：向比自己序号小的最后一个节点注册
        4. 接到通知重复1步骤
## 8.4 分布式队列和barrier
- FIFO：创建临时顺序节点，判断自己是否是最小节点，是则执行否则等待，同时向比自己序号小的注册Watcher
- barrier：节点内容赋为目标值，客户端在节点下创建子节点并判断子节点数量是否达到目标值，同时注册子节点变更Watcher
# 9 ZK的stat对象和版本号？
![](../picture/微服务/zk/1-stat.png)
version属性在setData时作为乐观锁实现机制：setData会传入version，不要求传入-1
# 10 Watcher机制？
## 10.1 工作机制
- 客户端注册  
对客户端请求request进行标记为*使用Watcher监听*，保存路径和Watcher映射到ZKWatcherManager.xxxWatches(Map\<String,Set\<Watcher\>>)。发送消息中**不含watcher对象（包含process方法）**，ZK只会把**request和requestHeader**两个属性序列化
- 服务器处理  
    - 存储：在WatcherManager中保存watchTable和watch2Paths两个结构。
    - Watcher触发：先封装为WatchedEvent（KeeperState、EventType、Path），从watchTable中找到Watcher，并从watchTable和watch2Paths中删除（**一次性**），向客户端返回。
- 客户端回调  
EventThread从WatcherManager中取出放到waitting Events队列，再取出进行**串行同步处理**
## 10.2 特性
一次性、客户端串行执行、轻量（没有传递Watcher对象开销低）
# 11 ACL？
- 格式：scheme:id:permission
- 不同scheme格式
	- ip:{ip}:cdrwa
	- digest:username:password(sha-1+BASE64):cdrwa
	- world:anymore:cdrwa
	Super:需要启动是配置属性支持，使用和digest一样
- Permission：CDRWA(admin：ACL设置权限)
# 12 客户端原理？
![](../picture/微服务/zk/2-客户端.png)
- 流程：建立TCP连接、建立会话  
- 服务器地址列表先随机再Round-robin
- ClientCnxn：网络I/O
    - Packet：协议层的封装
    - SendThread：维护客户端和服务端会话生命周期，定期心跳ping，如果TCP断开会自动重连
    - EventThread处理Watcher
    - ClientCnxnSocket：通信层
# 13 ZK如何进行会话管理？
会话状态有CONNECTING、CONNECTED、RECONNECTING、RECONNECTED、CLOSE    
- 会话激活：客户端发送请求就会触发一次会话激活，如果**超时时间1/3**内没进行通信，则会发送PING，触发会话激活
- 会话清理：服务端超时检查线程会对已过期会话进行清楚、删除临时节点
- 自动重连：自动从地址列表中选取地址重新连接，但是如果连上后服务端发现已超时会返回SESSION——EXPIRED
# 14 Leader、Follower、Observer的请求处理过程？
采用责任链模式
## 14.1 Leader
![](../picture/微服务/zk/3-Leader请求处理链.png)
- PreRequestProcessor：生成事务请求头（**包含ZXID**）
- ProposalRequestProcessor
    - 交给CommitProcessor
    - 对于**事务请求**：创建Proposal提议，发给所有followler，事务投票，交给SyncRequestProcessor进行事务日志记录
- SyncRequestProcessor
    - 记录事务日志
    - 记录快照
- CommitProcessor：对于事务请求：等待Proposal投票直到可提交
- FinalRequestProcessor：创建客户端请求的响应，将事务应用到**内存数据库**，并放入**commitProposal**（保存最近被提交的事务请求，便于进行数据同步）
- LearnerHandler：是ZooKeeper集群中Learner服务器的管理器，主要负责Follower/Observer服务器和Leader服务器之间的一系列**网络通信，包括数据同步、请求转发和Proposal提议的投票**等。Leader服务器中保存了**所有Follower/Observer对应的LearnerHandler**
## 14.2 Follower
- 处理非事务请求；转发事务请求给Leader，并继续走Commit流程
- 参与事务请求投票
- 参与Leader选举投票
![](../picture/微服务/zk/4-Follower请求处理链.png)  
[Zookeeper核心之 实现细节【Follower的请求处理流程】](https://blog.csdn.net/Coder_Boy_/article/details/109971650)
## 14.3 Observer
处理非事务请求，转发事务请求给Leader。  
- 不同于Leader和Follower之间的**PROPOSAL,ACK,COMMIT**消息，Leader会采用**Inform**消息通知Observer进行事务提交，其中携带了事务请求的内容。
# 15 ZK的数据与存储，及数据初始化和同步（启动过程）？
## 15.1 数据与存储
- ZKDatabase：ZooKeeper 的**内存数据库**，负责管理ZooKeeper 的所有会话、DataTree 存储和事务日志。ZKDatabase 会定时向磁盘dump 快照数据，同时在ZooKeeper 服务器启动的时候，会通过磁盘上的事务日志和快照数据文件恢复成一个完整的内存数据库。
    - DataTree
	    - **nodes**：ConcurrentHashMap<String,DataNode>
            - DataNode
                - data[]
                - acl
                - stat
                - parent引用
	            - children列表
	    - dataWatches
	    - childWatches
	    - ephemerals：ConcurrentHashMap<String,DataNode>
- 事务日志
- snapshot：在配置次事务日志记录后进行一次数据快照
## 16.2 初始化和同步
- 初始化：从快照文件中加载快照数据，并根据事务日志进行数据订正
- **过半**同步：完成Leader选举后，Learner向Leader注册后进行数据同步  
    - 主要依赖上文提到的**commitProposals**，根据Learner的ZXID是否处于commitProposals队列中，进行DIFF,TRUNC+DIFF,TRUNC,SNAP 4种方式的同步
## 16.3 启动过程
先创建ZkDataBase，然后恢复本地数据（**初始化**），然后**Leader选举**，最后**同步**
# 17 ZK运维（了解）
## 17.1 配置
- 基本配置
	**clientPort**、dataDir、tickTime（最小时间单元）
- 高级配置
	- dataLogdir：最好单独硬盘
	- server.id=host:**port:port**
		- 和leader通信和数据同步
		- leader选举
	- forceSync：事务提交强制刷盘
	- leaderServes：不接受客户端连接
## 17.2 四字命令
- 使用方式：nc、telnet  
![](../picture/微服务/zk/5-4字命令.png)
## 17.3 集群高可用
奇数台容灾能力和多一台一样（过半存活即可用）。搭建一个允许F台机器宕掉的集群，需要部署2F+1台。


https://juejin.cn/post/6844904002962849799