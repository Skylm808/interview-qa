# ClickHouse 详细介绍：定位、索引原理、为什么比 MySQL 快

> 参考视角：这篇按「工程师看系统设计」的方式写，不只讲概念，而是重点解释 ClickHouse 为什么快、它的索引到底怎么工作、它和 MySQL 的根本差异在哪里。

> 面试导读：ClickHouse 面试通常不会只问“是什么”，而是会顺着「适用场景 -> 存储模型 -> MergeTree -> 主键/分区/索引 -> 为什么快 -> 为什么不适合频繁更新」一路追问。下面这篇保留原理讲解，同时在末尾补了高频问题清单，方便临场复习。

---

## 目录

1. [ClickHouse 是什么](#1-clickhouse-是什么)
2. [ClickHouse 适合什么场景](#2-clickhouse-适合什么场景)
3. [ClickHouse 和 MySQL 的根本区别](#3-clickhouse-和-mysql-的根本区别)
4. [为什么 ClickHouse 通常比 MySQL 做分析更快](#4-为什么-clickhouse-通常比-mysql-做分析更快)
5. [ClickHouse 的存储引擎：MergeTree](#5-clickhouse-的存储引擎mergetree)
6. [ClickHouse 的索引到底是什么](#6-clickhouse-的索引到底是什么)
7. [ClickHouse 主键索引具体怎么实现](#7-clickhouse-主键索引具体怎么实现)
8. [ClickHouse 的稀疏索引和 MySQL B+Tree 索引的区别](#8-clickhouse-的稀疏索引和-mysql-btree-索引的区别)
9. [ClickHouse 如何减少读取数据量](#9-clickhouse-如何减少读取数据量)
10. [ClickHouse 为什么不擅长频繁 update/delete](#10-clickhouse-为什么不擅长频繁-updatedelete)
11. [ClickHouse 的写入过程](#11-clickhouse-的写入过程)
12. [ClickHouse 的查询执行过程](#12-clickhouse-的查询执行过程)
13. [ClickHouse 常见表设计要点](#13-clickhouse-常见表设计要点)
14. [一个完整例子：用户行为日志分析](#14-一个完整例子用户行为日志分析)
15. [ClickHouse、Hive、Spark、Flink、MySQL 的关系](#15-clickhousehivesparkflinkmysql-的关系)
16. [一句话总结](#16-一句话总结)

---

## 1. ClickHouse 是什么

ClickHouse 是一个开源的、列式的、面向 OLAP 的分析型数据库。

它最核心的目标是：

> 在海量数据上，快速完成分析查询。

典型查询包括：

- `count(*)`
- `sum(amount)`
- `avg(duration)`
- `group by city`
- `group by event_type`
- 最近 7 天趋势
- 用户行为分析
- 日志检索
- 报表看板
- 多维分析

它不是为了替代 MySQL 这类业务数据库而生的。

MySQL 更适合：

- 用户注册
- 下单
- 支付
- 库存更新
- 改订单状态
- 事务处理

ClickHouse 更适合：

- 统计近 7 天订单量
- 分城市统计成交额
- 查日志错误率
- 分析用户行为路径
- 支撑 BI 看板

一句话：

> MySQL 管业务交易，ClickHouse 管分析查询。

---

## 2. ClickHouse 适合什么场景

ClickHouse 适合典型 OLAP 场景。

### 2.1 日志分析

比如服务日志表：

```sql
SELECT
    toStartOfMinute(timestamp) AS minute,
    level,
    count(*) AS cnt
FROM app_logs
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY minute, level
ORDER BY minute;
```

这类查询需要扫描大量日志，然后按时间、级别聚合。ClickHouse 很擅长。

### 2.2 用户行为分析

比如埋点事件表：

```sql
SELECT
    event_name,
    count(*) AS pv,
    uniq(user_id) AS uv
FROM user_events
WHERE event_time >= today() - 7
GROUP BY event_name
ORDER BY pv DESC;
```

ClickHouse 很适合这种大规模事件表分析。

### 2.3 实时报表 / BI 看板

比如：

- 实时成交额
- 每小时订单数
- 各渠道转化率
- 各地区活跃用户数

ClickHouse 能承担很多秒级或亚秒级查询场景。

### 2.4 明细筛选

比如从几十亿条行为日志里筛选：

```sql
SELECT *
FROM user_events
WHERE event_time >= now() - INTERVAL 1 DAY
  AND user_id = 12345
ORDER BY event_time DESC
LIMIT 100;
```

如果表设计合理，ClickHouse 也能比较快地筛明细。

---

## 3. ClickHouse 和 MySQL 的根本区别

### 3.1 设计目标不同

| 维度 | MySQL | ClickHouse |
|---|---|---|
| 类型 | OLTP 数据库 | OLAP 数据库 |
| 主要目标 | 业务交易正确 | 海量分析查询快 |
| 典型操作 | 单行增删改查 | 大范围扫描、聚合 |
| 是否重事务 | 是 | 不是核心能力 |
| 是否适合频繁更新 | 适合 | 不适合 |
| 是否适合报表聚合 | 数据量大时吃力 | 很适合 |

OLTP 是 Online Transaction Processing，联机事务处理。

OLAP 是 Online Analytical Processing，联机分析处理。

MySQL 负责把每一笔业务处理正确。

ClickHouse 负责从大量业务数据中快速算出结论。

---

### 3.2 存储方式不同：行存 vs 列存

MySQL 通常是行式存储。

ClickHouse 是列式存储。

假设有一张表：

| id | user_id | event_name | event_time | city |
|---|---|---|---|---|
| 1 | 1001 | click | 2026-06-23 10:00:00 | 北京 |
| 2 | 1002 | pay | 2026-06-23 10:00:01 | 上海 |
| 3 | 1003 | click | 2026-06-23 10:00:02 | 杭州 |

MySQL 更像按行存：

```text
第 1 行: 1,1001,click,2026-06-23 10:00:00,北京
第 2 行: 2,1002,pay,2026-06-23 10:00:01,上海
第 3 行: 3,1003,click,2026-06-23 10:00:02,杭州
```

ClickHouse 更像按列存：

```text
id 列:         1, 2, 3
user_id 列:    1001, 1002, 1003
event_name 列: click, pay, click
event_time 列: ...
city 列:       北京, 上海, 杭州
```

这个差别非常关键。

如果你只查询：

```sql
SELECT event_name, count(*)
FROM user_events
GROUP BY event_name;
```

ClickHouse 主要读取 `event_name` 这一列就够了。

MySQL 行存则更容易读到整行中很多无关字段。

---

## 4. 为什么 ClickHouse 通常比 MySQL 做分析更快

ClickHouse 快，不是因为它“全面比 MySQL 高级”，而是因为它为 OLAP 查询做了非常多针对性设计。

### 4.1 列式存储：只读需要的列

分析查询往往只需要少数列。

例如一张日志表有 100 个字段，但查询只需要：

```sql
SELECT status_code, count(*)
FROM access_logs
WHERE event_time >= now() - INTERVAL 1 DAY
GROUP BY status_code;
```

ClickHouse 只需要重点读取：

- `status_code`
- `event_time`

不用把 100 个字段都读出来。

MySQL 行存更适合按主键拿整行，但对这种“只用少数列，扫描很多行”的分析查询没那么占优势。

---

### 4.2 压缩率高

列式存储让同一列的数据放在一起。

同一列的数据类型和分布通常很相似，比如：

- 时间戳列
- 状态码列
- 城市列
- 事件类型列
- 布尔列

这类数据放在一起更容易压缩。

压缩带来两个好处：

1. 磁盘占用更少
2. 查询时需要从磁盘读取的数据更少

很多 OLAP 查询的瓶颈不是 CPU，而是 I/O。少读数据，就会明显变快。

---

### 4.3 向量化执行

ClickHouse 通常不是一行一行处理数据，而是一批一批处理。

它会把数据按 block 读取，然后进行批量计算。

这叫向量化执行。

好处是：

- 减少函数调用开销
- 更好利用 CPU cache
- 更容易利用 SIMD 等底层优化
- 聚合计算效率高

MySQL 更偏传统 OLTP 执行路径，适合处理点查和事务，不是为大规模列式批量计算设计的。

---

### 4.4 数据按排序键组织，可以跳过大量数据

ClickHouse 的 MergeTree 表会按照 `ORDER BY` 指定的排序键组织数据。

比如：

```sql
ORDER BY (event_date, user_id)
```

如果查询条件是：

```sql
WHERE event_date = '2026-06-23'
```

ClickHouse 可以利用排序信息，只读取相关数据块，而不是全表扫。

这不是 MySQL 那种逐条定位的 B+Tree 索引，而是一种适合 OLAP 的“跳过大量无关数据块”的机制。

---

### 4.5 并行执行能力强

ClickHouse 查询天然会并行化：

- 多线程读数据
- 多线程解压
- 多线程过滤
- 多线程聚合
- 分布式集群还能多机器并行

OLAP 查询通常数据量大，并行能力非常重要。

---

### 4.6 聚合函数高度优化

ClickHouse 内置了很多为分析优化的聚合函数，例如：

```sql
count()
sum()
avg()
uniq()
uniqExact()
countIf()
sumIf()
groupArray()
topK()
quantile()
```

比如 UV 统计：

```sql
SELECT uniq(user_id)
FROM user_events;
```

ClickHouse 对这类聚合有专门优化。

---

## 5. ClickHouse 的存储引擎：MergeTree

ClickHouse 里最核心、最常用的表引擎是 MergeTree 系列。

常见建表：

```sql
CREATE TABLE user_events
(
    event_time DateTime,
    event_date Date,
    user_id UInt64,
    event_name String,
    city String,
    device String,
    amount Float64
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, event_name, user_id);
```

这里最重要的几个点：

- `ENGINE = MergeTree`
- `PARTITION BY`
- `ORDER BY`

---

### 5.1 Partition：分区

`PARTITION BY` 决定数据如何分区。

比如：

```sql
PARTITION BY toYYYYMM(event_date)
```

表示按月份分区。

数据可能分成：

```text
202601
202602
202603
...
```

如果查询：

```sql
WHERE event_date >= '2026-06-01'
  AND event_date < '2026-07-01'
```

ClickHouse 可以只读 `202606` 这个分区。

这叫分区裁剪。

---

### 5.2 Order By：排序键

ClickHouse 的 `ORDER BY` 很重要。

它不是普通 SQL 最后结果排序的意思，而是底层数据存储顺序。

比如：

```sql
ORDER BY (event_date, event_name, user_id)
```

表示数据会按照这个 key 排序存储。

如果查询经常按 `event_date`、`event_name` 过滤，那么这个排序键就很有效。

---

### 5.3 Part：数据片段

ClickHouse 写入数据时，不是每插入一行就原地更新一个大文件。

它会把每次写入的数据形成一个或多个 part。

每个 part 内部是有序的、列式存储的。

后台会不断把小 part 合并成大 part。

这也是 MergeTree 名字里 “Merge” 的来源。

---

## 6. ClickHouse 的索引到底是什么

很多人把 MySQL 的索引概念直接套到 ClickHouse 上，容易理解错。

ClickHouse 的核心索引不是为了“精确定位某一行”，而是为了：

> 在大范围分析查询中，尽量跳过不需要读的数据块。

ClickHouse 里常见和“索引”相关的机制包括：

1. 分区裁剪
2. 主键索引 / 稀疏索引
3. 排序键
4. Granule / Mark
5. 数据跳过索引，也叫 skipping index
6. MinMax 索引
7. Bloom Filter 索引

其中最核心的是 MergeTree 的主键稀疏索引。

---

## 7. ClickHouse 主键索引具体怎么实现

### 7.1 Primary Key 和 Order By 的关系

在 MergeTree 表里，经常看到：

```sql
ORDER BY (event_date, event_name, user_id)
```

有时还会看到：

```sql
PRIMARY KEY (event_date, event_name)
ORDER BY (event_date, event_name, user_id)
```

如果不单独指定 `PRIMARY KEY`，默认主键就是 `ORDER BY`。

但要注意：

> ClickHouse 的 Primary Key 不像 MySQL Primary Key 那样保证唯一性。

ClickHouse 的主键主要用于索引和数据跳过，不负责唯一约束。

也就是说，ClickHouse 主键不是“唯一标识一行”的核心机制。

---

### 7.2 Granule：索引粒度

ClickHouse 不是每一行都建索引。

它会按照一定粒度给数据建索引。

默认通常是每 8192 行形成一个 granule。

可以粗略理解成：

```text
第 1 个 granule: 第 1 行 ~ 第 8192 行
第 2 个 granule: 第 8193 行 ~ 第 16384 行
第 3 个 granule: 第 16385 行 ~ 第 24576 行
...
```

ClickHouse 会为每个 granule 记录一条主键索引信息。

比如表按：

```sql
ORDER BY (event_date, user_id)
```

排序，那么每个 granule 的索引里会记录该 granule 起始位置的 key 值。

类似：

```text
granule 1 起始 key: (2026-06-01, 1001)
granule 2 起始 key: (2026-06-01, 9800)
granule 3 起始 key: (2026-06-02, 1200)
granule 4 起始 key: (2026-06-03, 3000)
```

这就是稀疏索引。

---

### 7.3 Mark：列文件里的定位点

ClickHouse 是列式存储，每一列都有自己的数据文件。

为了能快速跳到某个 granule 对应的位置，ClickHouse 会维护 mark 文件。

mark 可以理解成：

> 某个 granule 在某一列文件里的偏移位置。

查询时先用主键索引判断哪些 granule 可能有用，再通过 mark 定位到对应列文件的区域读取。

---

### 7.4 查询时如何利用主键索引

假设表按：

```sql
ORDER BY (event_date, user_id)
```

查询：

```sql
SELECT count(*)
FROM user_events
WHERE event_date = '2026-06-23'
  AND user_id = 10086;
```

ClickHouse 会大致做：

1. 根据分区条件，先排除无关分区
2. 在相关 part 中，根据主键稀疏索引定位可能包含 `event_date = '2026-06-23'` 的 granule 范围
3. 只读取这些 granule 对应的列数据
4. 在读取出来的数据里继续做精确过滤

注意：

ClickHouse 的稀疏索引只能帮你缩小读取范围，不保证直接定位到唯一一行。

它更多是“少读很多数据”，不是“像 MySQL 一样精确找到一条记录”。

---

## 8. ClickHouse 的稀疏索引和 MySQL B+Tree 索引的区别

### 8.1 MySQL B+Tree 索引

MySQL 常见索引是 B+Tree。

它适合：

- 按主键查一行
- 按唯一索引查一行
- 小范围查询
- 排序
- join
- 高并发 OLTP 查询

例如：

```sql
SELECT * FROM orders WHERE order_id = 123;
```

MySQL 可以通过 B+Tree 很快定位到这条记录。

MySQL 的索引更像：

> 为了快速找到某一行或少量行。

---

### 8.2 ClickHouse 稀疏索引

ClickHouse 的主键索引不是每行一条。

它是每隔一批行记录一个 key。

它适合：

- 大范围扫描
- 按排序键范围过滤
- 跳过不相关数据块
- 聚合分析

ClickHouse 的索引更像：

> 为了快速排除大量不相关的数据块。

---

### 8.3 对比表

| 维度 | MySQL B+Tree 索引 | ClickHouse 稀疏索引 |
|---|---|---|
| 索引粒度 | 通常精确到行 | 通常精确到 granule |
| 目标 | 找到少量行 | 跳过大量数据块 |
| 适合场景 | OLTP 点查 | OLAP 范围扫描 |
| 主键是否唯一 | 通常唯一 | 不要求唯一 |
| 查询方式 | 定位记录 | 裁剪读取范围 |
| 存储成本 | 相对更高 | 相对更低 |

---

### 8.4 为什么 ClickHouse 不用 MySQL 那种索引

因为 OLAP 查询的主要矛盾不同。

MySQL 常见查询：

```sql
SELECT * FROM user WHERE id = 1;
```

核心是找到一行。

ClickHouse 常见查询：

```sql
SELECT city, count(*)
FROM user_events
WHERE event_date >= '2026-06-01'
GROUP BY city;
```

核心是扫描很多行后聚合。

对于 OLAP 来说，逐行索引不是最优解。

原因：

1. 数据量巨大，每行建索引成本高
2. 查询经常要读很多行，即使有行级索引也得大量回表/读取
3. 分析更关心顺序读取、压缩、批处理、跳过无关块
4. 列存和批量扫描比随机点查更重要

所以 ClickHouse 选择的是适合 OLAP 的稀疏索引和数据跳过机制。

---

## 9. ClickHouse 如何减少读取数据量

ClickHouse 快的一个核心就是：

> 能不读的坚决不读。

主要靠几层机制。

---

### 9.1 分区裁剪

如果表按月分区：

```sql
PARTITION BY toYYYYMM(event_date)
```

查询只查 2026 年 6 月：

```sql
WHERE event_date >= '2026-06-01'
  AND event_date < '2026-07-01'
```

那其他月份分区可以直接不读。

这是第一层粗粒度裁剪。

---

### 9.2 主键稀疏索引裁剪 granule

在相关分区里，再用主键索引判断哪些 granule 可能符合条件。

如果排序键是：

```sql
ORDER BY (event_date, event_name, user_id)
```

查询：

```sql
WHERE event_date = '2026-06-23'
  AND event_name = 'pay'
```

ClickHouse 可以跳过大量不在这个日期、不属于这个事件类型的数据块。

---

### 9.3 只读取需要的列

如果查询只用：

```sql
event_date, event_name, user_id
```

ClickHouse 就不读其他列，比如：

- device
- app_version
- extra_json
- network_type
- page_url

这在宽表场景很有优势。

---

### 9.4 数据跳过索引

ClickHouse 支持额外的数据跳过索引。

比如 MinMax 索引：

```sql
INDEX idx_amount amount TYPE minmax GRANULARITY 4
```

它会记录某些数据块里 `amount` 的最小值和最大值。

如果查询：

```sql
WHERE amount > 10000
```

某个数据块的最大值只有 200，那么这个数据块可以直接跳过。

---

### 9.5 Bloom Filter 索引

对一些高基数字段，比如 user_id、trace_id、request_id，可以用 Bloom Filter。

它可以快速判断：

> 某个数据块里大概率有没有这个值。

如果 Bloom Filter 判断肯定没有，就跳过。

但 Bloom Filter 有误判可能：

- 判断没有：一定没有
- 判断有：可能有，也可能没有

所以它适合减少读取，但不能代替精确过滤。

---

## 10. ClickHouse 为什么不擅长频繁 update/delete

ClickHouse 是为追加写入和分析查询设计的，不是为频繁修改单行设计的。

### 10.1 MySQL 的数据是“活记录”

MySQL 里一条订单状态经常变化：

```sql
UPDATE orders
SET status = 'paid'
WHERE order_id = 123;
```

这很正常。

MySQL 支持事务、锁、MVCC、B+Tree 索引，很适合频繁修改。

---

### 10.2 ClickHouse 的数据更像“事实日志”

ClickHouse 更适合写入不可变或少变的事实数据：

- 用户点击事件
- 支付事件
- 服务日志
- 监控指标
- 行为明细

这类数据通常是追加写入：

```text
发生一条事件 -> 写一条新记录
```

而不是频繁修改旧记录。

---

### 10.3 ClickHouse 的 update/delete 本质较重

ClickHouse 虽然支持 mutation 类型的 update/delete，但通常是后台异步改写数据 part。

它不是像 MySQL 那样立即精确改一行的轻量操作。

大规模 mutation 会消耗很多资源。

所以 ClickHouse 使用原则是：

> 能追加就追加，少做频繁 update/delete。

---

## 11. ClickHouse 的写入过程

以 MergeTree 为例，ClickHouse 写入大致是：

1. 客户端批量插入数据
2. ClickHouse 按列组织数据
3. 根据分区规则写入对应 partition
4. 在 partition 下形成新的 part
5. part 内部按 `ORDER BY` 排序
6. 为 part 生成主键稀疏索引、mark 文件、列数据文件
7. 后台 merge 线程把小 part 合并成大 part

### 11.1 为什么推荐批量写入

如果每次只写一行，会产生大量小 part。

小 part 太多会导致：

- 查询时 part 数量多
- 后台 merge 压力大
- 元数据负担重
- 写入和查询都变慢

所以 ClickHouse 推荐批量写。

例如一批几千行、几万行甚至更多，通常比一行一行写好得多。

---

### 11.2 后台 Merge 是什么

ClickHouse 会不断把小 part 合并成更大的 part。

比如：

```text
part_1 + part_2 + part_3 -> part_merged
```

合并时会重新排序、压缩、生成索引和 mark。

这个过程对查询性能和存储组织很重要。

---

## 12. ClickHouse 的查询执行过程

假设有查询：

```sql
SELECT
    event_name,
    count(*) AS pv,
    uniq(user_id) AS uv
FROM user_events
WHERE event_date >= '2026-06-01'
  AND event_date <= '2026-06-23'
GROUP BY event_name
ORDER BY pv DESC;
```

ClickHouse 大致执行过程：

1. 解析 SQL
2. 分析查询需要哪些列：`event_name`、`user_id`、`event_date`
3. 根据分区条件裁剪无关 partition
4. 根据主键索引裁剪无关 granule
5. 通过 mark 定位列文件中的读取位置
6. 读取需要的列数据
7. 解压数据块
8. 执行过滤条件
9. 批量聚合计算
10. 多线程合并聚合结果
11. 排序返回

这个流程里有几个关键优化点：

- 不读无关分区
- 不读无关 granule
- 不读无关列
- 顺序读压缩数据
- 批量向量化执行
- 多线程并行聚合

这些叠加起来，就是 ClickHouse 分析查询快的根源。

---

## 13. ClickHouse 常见表设计要点

ClickHouse 性能很依赖表设计。

### 13.1 分区不要太细

常见按天或按月分区。

例如：

```sql
PARTITION BY toYYYYMM(event_date)
```

或：

```sql
PARTITION BY event_date
```

不要轻易按 user_id 这种高基数字段分区，否则分区数量会爆炸。

---

### 13.2 ORDER BY 要贴合查询条件

如果查询经常按日期、事件类型、用户过滤：

```sql
WHERE event_date = xxx
  AND event_name = xxx
  AND user_id = xxx
```

可以考虑：

```sql
ORDER BY (event_date, event_name, user_id)
```

排序键前缀越能匹配常见查询条件，跳过效果越好。

---

### 13.3 低基数字段可以用 LowCardinality

如果某列取值种类很少，比如：

- city
- device
- app_version
- event_name
- channel

可以考虑：

```sql
event_name LowCardinality(String)
```

这有助于减少存储和提升查询效率。

---

### 13.4 避免把 ClickHouse 当 MySQL 用

不建议在 ClickHouse 中做：

- 高频单行 update
- 高频 delete
- 强事务操作
- 按主键频繁点查并修改
- 大量小批量插入

ClickHouse 的正确使用姿势是：

- 批量写入
- 追加写入
- 按时间分区
- 按查询模式设计排序键
- 少数列扫描
- 聚合分析

---

## 14. 一个完整例子：用户行为日志分析

### 14.1 建表

```sql
CREATE TABLE user_events
(
    event_time DateTime,
    event_date Date,
    user_id UInt64,
    event_name LowCardinality(String),
    page String,
    city LowCardinality(String),
    device LowCardinality(String),
    app_version LowCardinality(String),
    amount Float64
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, event_name, user_id);
```

这个表的设计意图：

- `PARTITION BY toYYYYMM(event_date)`：按月分区，方便按时间裁剪
- `ORDER BY (event_date, event_name, user_id)`：常见查询按日期、事件类型、用户过滤
- `LowCardinality(String)`：适合低基数字符串字段

---

### 14.2 查询 PV/UV

```sql
SELECT
    event_date,
    event_name,
    count(*) AS pv,
    uniq(user_id) AS uv
FROM user_events
WHERE event_date >= '2026-06-01'
  AND event_date <= '2026-06-23'
GROUP BY event_date, event_name
ORDER BY event_date, pv DESC;
```

ClickHouse 优势：

1. 只读 6 月相关分区
2. 利用排序键跳过部分数据块
3. 只读 `event_date`、`event_name`、`user_id` 相关列
4. 批量聚合 `count` 和 `uniq`
5. 多线程执行

---

### 14.3 查询某用户行为轨迹

```sql
SELECT
    event_time,
    event_name,
    page,
    city,
    device
FROM user_events
WHERE event_date >= '2026-06-01'
  AND event_date <= '2026-06-23'
  AND user_id = 10086
ORDER BY event_time
LIMIT 1000;
```

因为排序键是：

```sql
ORDER BY (event_date, event_name, user_id)
```

这个查询对 `event_date` 有帮助，但对 `user_id` 的定位不是最理想，因为 `user_id` 在第三位，而且中间有 `event_name`。

如果系统大量查询单用户轨迹，可能需要重新考虑排序键，例如：

```sql
ORDER BY (user_id, event_time)
```

或者单独建一张面向用户轨迹查询的表。

这说明 ClickHouse 表设计非常依赖查询模式。

---

## 15. ClickHouse、Hive、Spark、Flink、MySQL 的关系

可以用一句话记：

> MySQL 管业务，Hive 管离线数仓，Spark 跑批，Flink 跑流，ClickHouse 快查。

### 15.1 MySQL

负责线上业务交易。

例如：

- 订单
- 用户
- 支付
- 库存

### 15.2 Hive

负责离线数仓表体系。

例如：

- ODS 原始层
- DWD 明细层
- DWS 汇总层
- ADS 应用层

### 15.3 Spark

负责大规模离线批处理。

例如：

- 每天凌晨跑昨天数据
- 清洗 ODS 到 DWD
- 汇总 DWD 到 DWS

### 15.4 Flink

负责实时流处理。

例如：

- 消费 Kafka 事件
- 实时计算成交额
- 实时风控
- 实时特征

### 15.5 ClickHouse

负责快速 OLAP 查询。

例如：

- 实时报表
- 明细分析
- 日志检索
- BI 看板

### 15.6 典型链路

```text
MySQL / 日志 / Kafka
        ↓
Hive ODS 原始层
        ↓ Spark 离线加工
Hive DWD / DWS / ADS
        ↓
ClickHouse 快速查询
```

实时链路可能是：

```text
Kafka
  ↓ Flink 实时计算
ClickHouse
  ↓
实时看板 / 查询平台
```

---

## 16. 一句话总结

ClickHouse 快的本质不是“它比 MySQL 全面高级”，而是它为 OLAP 分析查询选择了完全不同的设计：

- 列式存储
- 高压缩率
- 向量化执行
- 多线程并行
- 分区裁剪
- 稀疏主键索引
- granule/mark 定位
- 数据跳过索引
- 追加写入和后台 merge

MySQL 适合快速、准确地处理一条条业务记录。

ClickHouse 适合从海量记录里快速计算统计结果。

最稳的记法：

> MySQL 负责把每一笔业务做对。  
> ClickHouse 负责从很多业务记录里快速算出结论。

---

## 17. ClickHouse 面试高频问题

### 17.1 ClickHouse 是什么，适合什么场景？

ClickHouse 是面向 OLAP 的分析型数据库，更适合日志分析、埋点分析、实时报表、BI 看板、用户行为统计这类“读多、聚合多、扫描多”的场景。

面试时可以直接说：

> ClickHouse 不是拿来替代 MySQL 做业务交易的，而是拿来在海量数据上快速做统计分析的。

### 17.2 为什么 ClickHouse 通常比 MySQL 更适合分析查询？

核心原因是它的设计目标不同。

- ClickHouse 是列式存储，更适合只读少数列
- MergeTree 适合高吞吐写入和海量数据
- 主键只维护到 granule 级别，不是逐行索引
- 分区裁剪、跳数索引可以减少读取的数据量
- 适合聚合、分组、排序、扫描类查询

### 17.3 MergeTree 的核心特点是什么？

MergeTree 是 ClickHouse 最常用的表引擎，典型特点是：

- 插入会先形成 data parts
- 后台再异步 merge 这些 parts
- `ORDER BY` 决定排序键
- 主键索引对应的是 granule，而不是单行
- 支持分区、复制、TTL、采样和跳数索引

### 17.4 `ORDER BY`、`PRIMARY KEY`、`PARTITION BY` 有什么区别？

- `ORDER BY`：排序键，决定数据在 part 内的排序
- `PRIMARY KEY`：主键，可选；不写时通常和排序键一致
- `PARTITION BY`：分区键，主要用于分区裁剪和数据管理，不是越细越好

面试里常追问的一点是：

> 分区不是为了直接提速查询，真正影响查询性能的核心仍然是排序键和索引裁剪。

### 17.5 granule 是什么？

granule 是 ClickHouse 读取数据时的最小不可再拆分单位。  
官方文档里给出的默认粒度是 8192 行左右一个 granule。

这也是面试常问点：ClickHouse 的主键不是一行一行索引，而是按 granule 做定位。

### 17.6 ClickHouse 为什么快？

常见回答可以按这几个点说：

- 列式存储，少读无关列
- 数据分成 parts 和 granules，减少扫描范围
- 分区裁剪，先排除整批不相关数据
- 跳数索引、统计信息帮助跳过不可能命中的块
- 适合批量分析和大范围聚合

### 17.7 ClickHouse 为什么不适合频繁 update / delete？

因为它的设计偏向追加写和后台合并，不是 OLTP 那种频繁行级修改模型。  
面试里可以回答成：

> ClickHouse 更擅长写入新数据和后台 merge，不擅长高频行级更新删除；如果业务需要强事务和频繁修改，通常还是 MySQL 更合适。

### 17.8 ClickHouse 和 Hive 有什么区别？

- ClickHouse：偏在线分析查询，响应更快，适合交互式分析
- Hive：偏离线数仓和批处理，适合大规模 ETL、离线汇总

### 17.9 ClickHouse 和 HBase 有什么区别？

- ClickHouse：分析型数据库，擅长聚合、统计、报表
- HBase：分布式 KV / wide-column 存储，擅长低延迟随机读写

### 17.10 如果面试官问“怎么设计 ClickHouse 表”，你怎么答？

可以按这个顺序答：

1. 先确认查询模式
2. `ORDER BY` 优先放高频过滤条件
3. 分区一般按天或月，不要过细
4. 只保留分析必要字段，避免超宽表乱放
5. 对高频过滤条件考虑跳数索引
6. 控制更新删除，尽量走追加写

### 17.11 参考资料

- [ClickHouse MergeTree 官方文档](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree)
- ClickHouse 官方文档主页：https://clickhouse.com/docs
- 整理说明：正文以 ClickHouse 官方文档和常见 OLAP 面试追问为基础，按“介绍 -> 原理 -> 高频问题”重组
