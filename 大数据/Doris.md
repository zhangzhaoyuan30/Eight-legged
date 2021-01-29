<!-- TOC -->

- [1 数据划分？](#1-数据划分)
- [2 数据模型？](#2-数据模型)
    - [2.1数据模型局限性？](#21数据模型局限性)
    - [2.2数据模型的选择建议？](#22数据模型的选择建议)
- [3 ROLLUP表？](#3-rollup表)
- [4前缀索引？](#4前缀索引)

<!-- /TOC -->
# 1 数据划分？
- Tablet & Partition
- Doris 支持两层的数据划分。第一层是 Partition，仅支持 Range 的划分方式。第二层是 Bucket（Tablet），仅支持 Hash 的划分方式
    - 分桶列的选择，是在 查询吞吐 和 查询并发 之间的一种权衡
# 2 数据模型？
- Aggregate：SUM、REPLACE、MAX、MIN
- Uniq：Aggregate的特例，每个key都使用Replace聚合
- Duplicate 
## 2.1数据模型局限性？
针对 Aggregate 模型，count需要通过加一列聚合类型为 REPLACE，且值恒为 1的列来实现。否则是扫描所有key且聚合后得到的结果，当聚合列非常多时，扫描的数据量大[聚合模型的局限性](https://doris.apache.org/master/zh-CN/getting-started/data-model-rollup.html#%E8%81%9A%E5%90%88%E6%A8%A1%E5%9E%8B%E7%9A%84%E5%B1%80%E9%99%90%E6%80%A7)
## 2.2数据模型的选择建议？
1. Aggregate 模型可以通过预聚合，极大地降低聚合查询时所需扫描的数据量和查询的计算量，非常适合有固定模式的报表类查询场景。但是该模型对 count(*) 查询很不友好。同时因为固定了 Value 列上的聚合方式，在进行其他类型的聚合查询时，需要考虑语意正确性。
2. Uniq 模型针对需要唯一主键约束的场景，可以保证主键唯一性约束。但是无法利用 ROLLUP 等预聚合带来的查询优势（因为本质是 REPLACE，没有 SUM 这种聚合方式）。
3. Duplicate 适合任意维度的 Ad-hoc 查询。虽然同样无法利用预聚合的特性，但是不受聚合模型的约束，可以发挥列存模型的优势（只读取相关列，而不需要读取所有 Key 列）。
# 3 ROLLUP表？
在 Base 表之上，我们可以创建任意多个 ROLLUP 表。这些 ROLLUP 的数据是基于 Base 表产生的，并且在**物理上是独立存储**的。  
- ROLLUP 表的基本作用：
    - 在于在 Base 表的基础上，获得**更粗粒度的聚合**数据
    - Duplicate 模型中的 ROLLUP：作为调整列顺序，以命中前缀索引的作用
- 注意事项
    - 但是不能在查询中显式的指定查询某 ROLLUP。**是否命中 ROLLUP 完全由 Doris 系统自动决定**。
    - 查询能否命中 ROLLUP 的一个必要条件（非充分条件）是，查询所涉及的所有列（包括 select list 和 where 中的查询条件列等）都存在于该 ROLLUP 的列中
    - 可以通过 **EXPLAIN your_sql**; 命令获得查询执行计划，在执行计划中，查看是否命中 ROLLUP
    - 可以通过 DESC tbl_name ALL; 语句显示 Base 表和所有已创建完成的 ROLLUP
    - 命中条件
        - 查询或者子查询中涉及的所有列都存在一张独立的 Rollup 中。
        - 如果查询或者子查询中有 Join，则 Join 的类型需要是 Inner join。
# 4前缀索引？
- 不同于传统的数据库设计，Doris **不支持在任意列上创建索引**。Doris 这类 MPP 架构的 OLAP 数据库，通常都是通过提高并发，来处理大量数据的。
- 本质上，Doris 的数据存储在类似 SSTable（Sorted String Table）的数据结构中。该结构是一种有序的数据结构，可以按照指定的列进行排序存储。在这种数据结构上，以排序列作为条件进行查找，会非常的高效。
将一行数据的前 **36** 个字节 作为这行数据的前缀索引。当遇到 VARCHAR 类型时，前缀索引会直接截断
- 可以通过创建 ROLLUP 来人为的调整列顺序