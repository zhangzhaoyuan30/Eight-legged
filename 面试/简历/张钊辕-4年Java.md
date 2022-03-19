# 个人信息
- 张钊辕/男/1996<img style="float:right" src="./picture/1-照片.jpeg" width = "100" height = "128"/>
- 工作年限：3年
- 期望职位：Java后台开发
- 手机/微信：15600603715
- Email：zhangzhaoyuan123@foxmail.com

# 教育经历
- 2014.9-2018.7&emsp;&emsp;&emsp;&emsp;本科&emsp;&emsp;&emsp;&emsp;哈尔滨工业大学&emsp;&emsp;&emsp;&emsp;土木工程

# 技术栈
- Java基础扎实，熟悉集合、并发，了解 JVM 原理
- 熟悉 Spring/SpringBoot 开发，了解其实现原理
- 熟悉 Mysql/Redis 存储
- 熟悉 Kafka，了解其使用场景和原理
- 熟悉 Thrift，了解RPC原理、服务发现、服务治理技术
- 熟悉 Zookeeper，了解Paxos 协议、ZAB 协议
- 有 Spark/Flink/Druid 等大数据处理经验

# 工作经历
- 2021.6-至今&emsp;&emsp;&emsp;&emsp;快手用户增长&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;Java工程师
- 2019.8-2021.6&emsp; &emsp;&emsp;小米商业化（广告部门）&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;Java工程师
- 2018.7-2019.8&emsp;&emsp;&emsp;&emsp;数码视讯&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;Java工程师

# 项目经历
##  SEM落地页性能优化
- **背景**：
落地页平均220ms，p99 400ms，用户体验？
- **解决难点**：
    1. 热门视频55ms，横滑视频155ms（如何降低到2ms，50ms）？并行化，线程池大小？
    2. 利用缓存，本地缓存+redis，缓存线程池异步更新，缓存过期时间？使用redis+本地两级缓存，并对全部数据进行缓存，请求到达时先查询本地缓存，然后查询redis缓存，对于没有命中的再调rpc查询，并异步更新redis缓存，最后距离上次更新超过一定阈值的数据，提交异步更新缓存任务。（**数量、大小，单个4kb**）
    3. sentinel为什么要熔断降级隔离降级（不会因下游超时而超时，线程池打满），线程池加锁？
    4. 横滑获取视频数据源重构（策略模式）
## SEM 智能运营规则引擎
## RTA
## 全网匹配
##  oCPX 广告投放系统开发
- **背景**：
游戏广告需要提升广告收入，而广告主对付费回收考核严格，因此搭建整套oCPX系统，采用oCPX手段进行游戏广告效果的优化
- **承担角色**：
初期参与开发，后期作为游戏oCPX功能Owner
- **解决难点**：
    1. 搭建广告实时归因服务：归因结果用于OLAP和cvr模型训练，形成投放效果优化闭环 (SpringBoot/Kafka/Pegasus/MySql/Druid/Pivot)
    2. 游戏 oCPX 广告 DSP 服务搭建 (Spring/Thrift)
    3. 使用 PID 控制算法稳定广告成本(Redis/Flink)
    4. 先后支持多种广告出价方式，使用策略模式重构出价模块
    5. 搭建策略服务，模型实验和策略实验分层，便于策略组优化 (SpringBoot/Thrift)
## EVE 广告诊断平台
- **背景**：
依托于广告平台的数据能力，提供收入波动根因分析、广告出价诊断、广告素材诊断等功能，集前端、后端、数据处理一体的平台。目的是降低广告销售、运营排查问题的门槛。
- **承担角色**：
负责收入诊断功能数据侧和后台开发
- **解决难点**：
    1. 引入多维度分层下钻分析算法完成广告收入根因分析功能
    2. 解决 Spark 任务数据倾斜问题，保证数据实时性
    3. 解决竞价数据 Flink 作业消费堆积问题
