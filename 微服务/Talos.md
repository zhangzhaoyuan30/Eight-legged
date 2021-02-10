# 1 Talos
Talos Streaming Data Platform，简称Talos Platform，是小米针对大数据时代的流式数据平台。
- 【注】：Partition个数的指定，请用户参照如下的标准来设置，每个partition允许最大1MB/s的写，最大2MB/s的读，请用户根据业务的数据量来合理设置partition个数；

# 2 Why Talos?
- 多租户：Talos 提供完善的认证/授权机制，细粒度的权限访问控制，保证**多租户下的数据安全/隔离**，解决 Kafka 数据不安全的痛点；隐私合规更符合规范；
- 高可靠：Talos 底层基于小米沉淀已久的 **HDFS，每一次写都是强制三副本**，可以在保持最优性能的同时保证数据不丢，解决 Kafka 使用单副本可能丢数据的隐患
- 稳定：Talos 消费端提供更优的 rebalance 机制（基于最小损失的均衡算法），解决内部用户使用 Kafka 遇到的消费 failover 导致 rebalance 不稳定的痛点
- 快速恢复：Talos 在宕机（故障转移）与数据迁移（负载均衡）时，无状态的设计会让其很快恢复，解决 Kafka 运维复杂、出现故障恢复时间长的痛点（存储计算耦合需要数据恢复）

# 3 语义
- producer：at least once
- consumer：都可以

https://juejin.cn/post/6844904166049972238
https://www.infoq.cn/article/p9oj2ns8t1ueacbleltc