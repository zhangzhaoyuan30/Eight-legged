# 1 特点
Pegasus是一个分布式Key-Value存储系统
- 数据**模型简单**, 可以接受Key-Value模型存储
- 数据**存储量大**, 例如100GB~10TB的存储规模, 使用Redis等内存存储价格高昂, 希望通过相对更低廉的磁盘(SSD)来访问
- 数据有强一致, **持久化**的需求。例如存储用户消息, 用户访问历史等等, 不希望数据丢失
- 数据访问需要较低的延迟, Pegasus 通常能够提供P99在**10ms级别**的稳定读写延迟 (注: 3KB的单条数据读写)

设计考虑：
- 高扩展性：通过哈希分片实现分布式横向扩展。
- **强一致性**：通过PacificA一致性协议保证。每一份数据写入都将分别复制到3个不同ReplicaServer上
- 高性能：底层使用RocksDB作为存储引擎。
- 简单易用：基于Key-Value的良好接口。

# 2 与Redis：
- Redis性能比Pegasus好不少，**Redis是在几十或者几百微妙**级别，Pegasus是在**毫秒级别**。
- 数据模型：两者都是Key-Value模型，但是Pegasus支持(HashKey + SortKey)的二级键。
- 接口：Redis的接口更丰富，支持List、Set、Map等容器特性；Pegasus的接口相对简单，功能更单一。
- 伸缩性：Pegasus伸缩性更好，可以很**方便地增减机器节点，并支持自动的负载均衡**；Redis的分布式方案在增减机器的时候比较麻烦，需要较多的运维介入。
- 可用性：Pegasus数据总是持久化的，系统架构保证其较高的可用性；Redis在机器宕机后需要较长时间恢复，**可用性不够好**，还可能丢掉最后一段时间的数据。
- 成本：Pegasus使用SSD，Redis主要使用内存，从成本上考虑，Pegasus显然更划算。
# 3 数据模型
- HashKey  
字节串，限制为64KB。类似于 DynamoDB 中的 partition key 概念，**HashKey 用于计算数据属于哪个分片**。Pegasus 使用一个特定的 hash 函数，对HashKey 计算出一个hash值，然后对分片个数取模，就得到该数据对应的 Partition ID 。HashKey 相同的数据总是存储在同一个分片中。
- SortKey  
字节串，长度无限制。类似于 DynamoDB 中的 sort key 概念，SortKey 用于数据在分片内的排序。HashKey 相同的数据放在一起，并且按照 **SortKey 的字节序排序**。实际上，在内部存储到RocksDB时，我们将 **HashKey 和 SortKey 拼在一起作为 RocksDB 的 key**。
- Value  
字节串，长度无限制。