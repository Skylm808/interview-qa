# PostgreSQL 基础与面经合集

> 目标：用 MySQL 作参照理解 PostgreSQL，而不是把 PostgreSQL 当成“另一种 SQL 语法”。两者都是成熟的关系型数据库；选择关键在数据类型、查询能力、扩展生态、团队经验和运维成本。

---

## 目录

1. [一句话理解 PostgreSQL](#1-一句话理解-postgresql)
2. [核心概念：和 MySQL 对照着看](#2-核心概念和-mysql-对照着看)
3. [PostgreSQL 的强项：类型、索引与查询](#3-postgresql-的强项类型索引与查询)
4. [事务、MVCC 与日常运维](#4-事务mvcc-与日常运维)
5. [最小 SQL 示例：JSONB、全文搜索与索引](#5-最小-sql-示例jsonb全文搜索与索引)
6. [什么时候选 PostgreSQL，什么时候选 MySQL](#6-什么时候选-postgresql什么时候选-mysql)
7. [和 ES、Milvus、pgvector 的边界](#7-和-esmilvuspgvector-的边界)
8. [高频面试问答](#8-高频面试问答)

---

## 1. 一句话理解 PostgreSQL

PostgreSQL 是功能很完整的开源对象关系型数据库：除标准关系表外，它特别擅长 JSONB、数组、范围、地理空间、全文搜索和自定义类型/扩展。MySQL 则在经典互联网 OLTP（在线事务处理）场景中生态成熟、团队普及度高、运维经验丰富。

不要把它理解成“PostgreSQL 适合复杂、MySQL 只适合简单”。两者都能做 ACID 事务、MVCC 并发控制、复制、高可用、索引和分区；真正的差别在于业务是否需要 PostgreSQL 的数据表达与查询能力，以及团队能否把它用好。

~~~text
订单创建、库存扣减、支付状态：
  MySQL 或 PostgreSQL 都可以做事实源和事务主库。

商品属性经常变化、需要在 JSON 内筛选：
  PostgreSQL 的 JSONB + GIN 索引通常更自然。

数亿文档的关键词检索、聚合、运营搜索：
  优先 Elasticsearch，不用强行让关系库承担搜索引擎工作。

大规模语义向量召回：
  先评估 pgvector；规模或吞吐上来再考虑 Milvus 等专用向量库。
~~~

---

## 2. 核心概念：和 MySQL 对照着看

| 维度 | PostgreSQL | MySQL（主要指 InnoDB） | 怎么理解 |
| --- | --- | --- | --- |
| 事务与 MVCC | 支持事务、快照读、行锁和多种隔离级别 | 同样支持事务、MVCC、行锁和隔离级别 | 两者都能承担核心 OLTP；不要仅因“有 MVCC”选 PostgreSQL。 |
| 存储与扩展 | 可通过 extension 扩展类型、函数、索引能力，如 PostGIS、pgvector | 常以存储引擎和内建能力为中心 | PostgreSQL 的可扩展性是明显特点，但扩展也增加升级和运维评估。 |
| JSON | jsonb 适合查询与索引，常配合 GIN | 支持 JSON、函数索引/生成列等方式 | 两者都能存 JSON；JSON 内复杂包含查询和索引是 PostgreSQL 常见优势。 |
| 复杂类型 | 原生数组、范围、枚举、网络地址、几何等 | 类型体系更偏常规业务字段 | 不是为了炫技；只有业务真的需要时才用。 |
| 索引 | B-tree、Hash、GIN、GiST、SP-GiST、BRIN 等 | 常见业务主要依赖 B+ 树，另有全文、空间等索引 | PostgreSQL 可按数据形态选索引，尤其适合 JSONB、全文、范围与地理类查询。 |
| 全文检索 | tsvector/tsquery + GIN，可做应用内中小规模检索 | FULLTEXT 可满足基础文本搜索 | 大规模召回、复杂排序聚合仍应优先 ES。 |
| SQL 与查询 | CTE、窗口函数、LATERAL、RETURNING、丰富函数/操作符 | 现代 MySQL 也支持 CTE、窗口函数等 | PostgreSQL 在复杂查询表达和分析型 SQL 体验上常更灵活。 |
| 运维关注点 | VACUUM、autovacuum、膨胀、WAL、长事务 | undo/redo、锁、慢 SQL、主从延迟等 | 两者都有各自的维护重点，迁移前必须补齐监控与故障演练。 |

**MySQL 表和 PostgreSQL 表的思路相同。** 例如用户表、订单表都可以用主键、联合索引和事务完成；差别不是“语法能不能写出来”，而是后续对 JSON、搜索、地理位置、扩展和复杂查询的需求。

---

## 3. PostgreSQL 的强项：类型、索引与查询

### 3.1 JSONB：半结构化属性也能被查询和索引

假设商品的基础字段稳定，但不同品类属性差异很大：

~~~text
手机：内存、屏幕尺寸、颜色
衣服：尺码、面料、版型
图书：作者、出版社、装帧
~~~

可以把稳定且高频过滤的字段建普通列，把低频、变化快的属性放进 JSONB。

~~~sql
CREATE TABLE product (
  id          BIGSERIAL PRIMARY KEY,
  category_id BIGINT NOT NULL,
  name        TEXT NOT NULL,
  attrs       JSONB NOT NULL DEFAULT '{}'::jsonb
);

CREATE INDEX idx_product_attrs_gin
  ON product USING GIN (attrs);

SELECT id, name
FROM product
WHERE attrs @> '{"color": "black"}'
  AND attrs @> '{"memory_gb": 256}';
~~~

这里不是鼓励把所有列塞进 JSONB。价格、库存、租户、状态等高频字段仍应单独建列和索引；JSONB 更适合“字段经常变化、但仍需要按内容过滤”的扩展属性。

### 3.2 索引类型：先看数据形态，再选索引

| 索引 | 常见场景 | 例子 |
| --- | --- | --- |
| B-tree | 等值、范围、排序 | 订单号、创建时间、状态 + 时间联合索引。 |
| GIN | JSONB 包含查询、全文搜索、数组成员 | 商品属性、标签、文档检索。 |
| GiST / SP-GiST | 范围、几何、相似度等 | 时间范围是否重叠、地理空间。 |
| BRIN | 超大、物理写入顺序与时间高度相关的表 | 日志、事件流按时间追加的分区表。 |

索引越多不是越好：每个索引都会增加写入、更新、存储和维护成本。和 MySQL 一样，应从真实 WHERE、ORDER BY、JOIN 和数据分布出发，用 EXPLAIN ANALYZE 验证执行计划。

### 3.3 全文搜索和地理能力

PostgreSQL 的全文搜索适合“业务数据就在库中，检索规模中小、功能不复杂”的场景。它可以将文本转换为 tsvector，并用 GIN 加速；需要分词、同义词、复杂聚合、海量文档召回时，再引入 ES。

PostGIS 是 PostgreSQL 的常见空间扩展，可表达点、线、面与空间关系。附近门店、配送范围、围栏判断等需求很适合，但要明确坐标系、索引和精度要求；它不是“存两个经纬度字段”那么简单。

---

## 4. 事务、MVCC 与日常运维

### 4.1 MVCC 到底解决什么？

MVCC（多版本并发控制）让读操作可以在很多情况下读取一致快照，不必因普通写入长期阻塞。PostgreSQL 和 InnoDB 都有这类能力；写写冲突、唯一键竞争和显式锁仍会发生，不能把 MVCC 理解成“没有锁”。

例如扣库存仍要使用条件更新：

~~~sql
UPDATE inventory
SET available = available - 1
WHERE sku_id = $1
  AND available > 0;
~~~

受影响行数为 1 才表示成功。这个模式与 MySQL 完全相通：正确性依赖条件更新、唯一约束和事务边界，而非只依赖应用内锁。

### 4.2 PostgreSQL 特别要关注：长事务和 VACUUM

PostgreSQL 更新或删除后会留下旧行版本，后续由 VACUUM 回收。若长事务长时间不结束，旧版本不能及时清理，表和索引可能膨胀，查询和写入都会变慢。

日常至少监控：

- 最长事务年龄、空闲但仍在事务中的连接。
- autovacuum 是否跟得上，表/索引膨胀是否异常。
- 慢 SQL、锁等待、连接数、WAL 产生量、复制延迟、磁盘空间。
- 大批量更新/删除是否需要分批、分区或重建索引。

不要把 VACUUM 当成故障后的临时命令；autovacuum 是正常维护机制，参数应结合写入量和表大小调优。

---

## 5. 最小 SQL 示例：JSONB、全文搜索与索引

### 5.1 JSONB 查询

~~~sql
-- attrs 中的 brand 等于 Apple
SELECT id, name
FROM product
WHERE attrs ->> 'brand' = 'Apple';

-- 对经常按 brand 筛选的表达式建索引
CREATE INDEX idx_product_brand
  ON product ((attrs ->> 'brand'));
~~~

### 5.2 全文搜索

~~~sql
CREATE TABLE article (
  id      BIGSERIAL PRIMARY KEY,
  title   TEXT NOT NULL,
  body    TEXT NOT NULL,
  search  TSVECTOR GENERATED ALWAYS AS
    (to_tsvector('simple', coalesce(title, '') || ' ' || coalesce(body, ''))) STORED
);

CREATE INDEX idx_article_search ON article USING GIN (search);

SELECT id, title
FROM article
WHERE search @@ plainto_tsquery('simple', 'redis hot key')
ORDER BY ts_rank(search, plainto_tsquery('simple', 'redis hot key')) DESC;
~~~

### 5.3 和 MySQL 的迁移提醒

从 MySQL 迁来时，最容易踩的不是表结构，而是语义差异：大小写/排序规则、时间类型与时区、JSON 函数、分页、自动递增、事务隔离、NULL 比较和 SQL 方言都要逐项回归。不要只让 ORM 自动建表后就切生产流量；应先双写或回放验证，再灰度切读写。

---

## 6. 什么时候选 PostgreSQL，什么时候选 MySQL？

| 场景 | 更优先考虑 | 原因 |
| --- | --- | --- |
| 常规电商、订单、账户、支付，团队已有 MySQL 体系 | MySQL | 迁移收益小，熟悉的监控、容灾和调优更重要。 |
| B2B SaaS，租户属性和业务对象字段变化很快，仍需可靠过滤 | PostgreSQL | JSONB、GIN、复杂查询能减少大量临时扩展表或应用层过滤。 |
| 地图、配送范围、空间关系 | PostgreSQL + PostGIS | 空间类型与查询能力更强。 |
| 中小规模、与业务数据强关联的全文搜索 | PostgreSQL | 可在同一事务和 SQL 体系内完成，减少一套同步链路。 |
| 海量文档搜索、复杂聚合、运营检索 | Elasticsearch | 不要把关系库当搜索引擎横向扩展。 |
| 业务表加少量向量检索，规模还不大 | PostgreSQL + pgvector | SQL、事务、权限过滤与向量列放在一个库，开发和运维简单。 |
| 大规模高吞吐语义/图片/推荐召回 | Milvus 等专用向量库 | 向量索引和查询资源可以独立扩展。 |

**选型顺序：** 先看已有基础设施、团队经验和可接受运维成本；再用真实数据验证 p99、吞吐、索引体积和查询效果；最后才是产品特性列表。引入 PostgreSQL、ES 或 Milvus 都意味着新的备份、监控、权限和同步链路，不是“技术更先进”就一定更好。

---

## 7. 和 ES、Milvus、pgvector 的边界

### 7.1 pgvector 适合什么？

pgvector 是 PostgreSQL 扩展，让表能够存 vector、halfvec、sparsevec 等向量列。默认可精确最近邻查询，也可用 HNSW 或 IVFFlat 做近似检索。它适合“向量只是业务库的一个能力”：例如 10 万到数百万条知识库 chunk，查询同时需要 tenant、状态、时间和 SQL Join。

但近似索引加过滤时，要特别压测候选不足问题。pgvector 文档说明，HNSW/IVFFlat 可能先扫描近似候选、再应用过滤；过滤比例高时需结合普通字段索引、分区、部分索引或 iterative scan 调整 recall。不要只看 SQL 能执行，就以为结果和全量精确检索一样。

### 7.2 四种系统怎么协作？

~~~text
MySQL / PostgreSQL：事实数据、事务、权限、实时库存
Elasticsearch：关键词、过滤、聚合、运营检索、混合搜索
pgvector：中小规模向量能力与 SQL 查询同库
Milvus：大规模或高吞吐的独立向量召回层
~~~

它们不是互斥关系。一个 RAG 系统可以由 PostgreSQL 保存文档和权限事实，ES 负责关键词检索，Milvus 负责语义候选；但系统越多，CDC/Outbox 同步、版本、删除和权限过滤的复杂度也越高。能用一套系统达标时，不要为了“架构完整”而盲目多引组件。

---

## 8. 高频面试问答

### 8.1 🟢 PostgreSQL 和 MySQL 的核心区别是什么？

**回答：** 两者都能做关系型事务主库。PostgreSQL 的特点是类型系统、扩展能力和复杂查询更丰富，例如 JSONB、数组、范围、GIN/GiST、全文搜索和 PostGIS；MySQL 在经典 OLTP 场景生态成熟。选型不能只背功能表，要看业务数据形态、查询方式、团队经验和运维成本。

### 8.2 🟡 为什么 JSONB 不应该替代正常列？

**Situation（背景）：** 商品属性变化很快，团队想把所有字段都放 JSONB。  
**Task（任务）：** 既要灵活扩展，又要保证高频查询和约束可维护。  
**Action（行动）：** 把价格、库存、租户、状态等高频字段建普通列和索引；低频、差异大的扩展属性放 JSONB，并对真实查询条件建立 GIN 或表达式索引。  
**Result（结果）：** 模型可扩展，同时避免常用过滤退化成大范围 JSON 扫描。  
**Tradeoff（取舍）：** JSONB 灵活但弱化结构约束，写入和索引也有成本；稳定字段仍应规范化。  

### 8.3 🟡 PostgreSQL 的 MVCC 是否意味着不需要锁？

**回答：** 不意味着。MVCC 主要降低读写互相阻塞，但写写冲突、唯一键竞争、行锁和死锁仍存在。扣库存仍要用条件更新或行锁，支付回调仍要靠唯一键和状态机保证幂等；长事务还会阻碍旧版本回收，导致膨胀。

### 8.4 🔴 什么时候会从 PostgreSQL + pgvector 拆到 Milvus？

**Situation（背景）：** 向量检索从业务的辅助功能变为 RAG/推荐主链路，数据量、QPS、索引构建和查询资源持续增长。  
**Task（任务）：** 保住向量召回的延迟和 recall，同时不影响事务库。  
**Action（行动）：** 以 PostgreSQL 为事实源，通过 CDC/Outbox 异步同步向量与版本到 Milvus；用真实评测集比较 recall、p99、过滤后候选数和成本，再灰度切流。  
**Result（结果）：** 向量检索能独立扩缩容，业务事务与索引构建相互隔离。  
**Tradeoff（取舍）：** 多了一套集群和同步链路，必须处理删除、版本、权限和短暂最终一致；规模未到瓶颈时留在 pgvector 往往更简单。  

---

## 官方资料

- [PostgreSQL 并发控制](https://www.postgresql.org/docs/current/mvcc.html)
- [PostgreSQL JSON 类型与 JSONB 索引](https://www.postgresql.org/docs/current/datatype-json.html)
- [PostgreSQL 全文搜索](https://www.postgresql.org/docs/current/textsearch.html)
- [PostgreSQL 索引类型](https://www.postgresql.org/docs/current/indexes-types.html)
- [pgvector README：HNSW、IVFFlat 与过滤](https://github.com/pgvector/pgvector)
