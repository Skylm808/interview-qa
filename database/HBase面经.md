# HBase 面经整理：介绍、数据模型与高频问题

> 面试导读：HBase 面试常围绕“为什么它适合随机读写”“数据模型怎么设计”“Region/RegionServer 怎么工作”“MemStore 和 HFile 的写入链路”来问。下面先给一个简洁介绍，再整理常见追问。

## 1. HBase 是什么

HBase 是 Apache 生态里的分布式、可版本化的非关系型数据库，适合在大量行数据上做低延迟随机读写。

它更像一个面向海量数据的宽表 KV 存储，而不是传统关系型数据库。

### 1.1 典型场景

- 用户画像
- 订单明细索引
- 日志明细查询
- 实时检索
- 需要按 rowkey 快速定位的数据访问

### 1.2 核心特点

- 分布式
- 版本化
- 非关系型
- 低延迟随机访问
- 适合海量 row / column 组织数据

## 2. HBase 面试高频问题

### 2.1 HBase 的数据模型是什么？

HBase 的基本数据模型可以理解成：

- Row Key
- Column Family
- Column Qualifier
- Timestamp
- Cell Value

面试里通常会追问：

> 为什么 HBase 只强调 Column Family，而不是像 MySQL 那样强调完整的表结构？

### 2.2 Region 是什么？

Region 是 HBase 里表分布和可用性的基本单元。  
随着数据增长，Region 会拆分、迁移，由 RegionServer 承载。

这类问题常问：

- Region 和 RegionServer 的关系是什么
- 为什么要设计成 Region 级别分片
- Region split 会带来什么影响

### 2.3 HBase 为什么适合高并发随机读写？

可以从这几个点回答：

- 数据按 rowkey 定位，适合点查
- Region 分片后可以横向扩展
- 写入先进入 MemStore
- 落盘后变成 HFile
- 读取时结合内存和文件索引定位数据

### 2.4 MemStore 和 HFile 分别是什么？

- MemStore：内存里的写缓冲区
- HFile：落到 HDFS 上的持久化文件

写入先到 MemStore，flush 后落盘成 HFile；后台再通过 compaction 合并文件。

### 2.5 为什么 HBase 的 rowkey 设计很重要？

rowkey 决定：

- 数据如何排序
- 如何分片
- 是否会热点
- 查询是否能直接命中

常见追问：

- rowkey 能不能改
- 怎么避免热点 rowkey
- 复合 rowkey 怎么设计

### 2.6 HBase 和 MySQL 有什么区别？

- HBase 适合海量随机读写和宽表访问
- MySQL 适合事务、关联查询、复杂 SQL
- HBase 没有 MySQL 那种完整关系型能力

### 2.7 HBase 为什么说不是数据库，更像数据存储？

官方文档里会强调，HBase 缺少很多关系型数据库能力，比如二级索引、触发器和高级查询语言。  
面试中你可以理解为：

> HBase 更像可横向扩展的分布式存储系统，而不是传统 RDBMS。

### 2.8 HBase 的数据为什么不会只存在内存里？

因为它依赖 HDFS 持久化，数据最终要落成 HFile。  
MemStore 只是写入缓冲，不是最终存储。

### 2.9 HBase 和 Hive 有什么区别？

- HBase：随机读写、在线访问
- Hive：离线数仓、SQL 分析、批处理

### 2.10 参考资料

- [Apache HBase 官方 Reference Guide](https://hbase.apache.org/book.html)
- HBase 官方文档主页：https://hbase.apache.org/
- 整理说明：正文以 HBase 官方文档和常见存储引擎面试追问为基础
