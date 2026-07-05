---
title: Elasticsearch 深度解析
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - Elasticsearch
  - 搜索引擎
  - 倒排索引
  - 分布式
categories:
  - 存储
---

[TOC]

## 一、ES 基础概念

Elasticsearch（简称 ES）是一个基于 Lucene 构建的分布式、RESTful 风格的实时搜索与数据分析引擎。它不仅是一个全文搜索引擎，同时也具备数据存储和分析能力。

### 1.1 什么是 Elasticsearch

ES 的核心是 **Apache Lucene**——一个用 Java 编写的高性能全文搜索库。Lucene 本身仅是一个搜索库，不提供分布式能力、高可用和易用的 API。ES 在 Lucene 之上封装了分布式集群管理、REST API、数据聚合等能力，使其成为一个完整的搜索引擎产品。

**ES 不是数据库**，尽管它具备存储数据的能力。核心区别：

| 维度 | 关系型数据库 (MySQL) | Elasticsearch |
|------|---------------------|---------------|
| 核心功能 | 精确查询、事务、Join | 全文搜索、相关性排序、聚合分析 |
| 索引结构 | B+ Tree | 倒排索引 (Lucene) |
| 事务支持 | ACID | 无事务，最终一致性 |
| 数据更新 | 原地更新 | 标记删除 + 新增（不可变段） |
| JOIN 操作 | 支持 | 不原生支持（有限嵌套/父子） |
| 擅长场景 | OLTP 精确读写 | 全文检索、日志分析、时序数据 |

### 1.2 核心概念

ES 的概念与数据库概念存在天然映射关系，理解这个映射是入门的关键：

| Elasticsearch | 关系型数据库 | 说明 |
|---------------|-------------|------|
| Index（索引） | Database / Table | ES 7.x+ 一个索引等同于一张表；早期 Type 接近表的概念 |
| ~~Type~~（7.x 废弃） | Table | 7.x 仅允许单 Type (`_doc`)，8.x 彻底移除 |
| Document（文档） | Row | 一个 JSON 对象，相当于一行记录 |
| Field（字段） | Column | 文档中的键值对 |
| Mapping（映射） | Schema | 定义字段类型、分词器、索引选项等 |
| Shard（分片） | 分区 (Partition) | 水平拆分数据，每个分片是一个独立的 Lucene 实例 |
| Replica（副本） | 从库 | 分片的复制副本，提供高可用和读扩展 |

**Index（索引）**：ES 中"索引"有两个含义——作为名词，它是一类相似文档的集合（相当于 MySQL 的一张表）；作为动词，它指将文档写入并建立倒排索引的过程。每个索引由若干个分片组成。

**Shard & Replica**：分片是 ES 水平扩展的核心。创建索引时指定主分片数量（`number_of_shards`），一旦设定便不可修改（因为路由公式依赖它）。每个主分片可有 0 到多个副本分片（`number_of_replicas`），副本可在运行时动态调整。

```json
// 创建索引时指定分片数
PUT /my_index
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}
// 运行中动态修改副本数
PUT /my_index/_settings
{
  "number_of_replicas": 2
}
```

### 1.3 适用场景与不适用场景

**适用场景**：

- **全文搜索**：电商商品搜索、知识库检索、代码搜索（GitHub 使用 ES 支撑代码搜索）
- **日志与监控**：ELK Stack（Elasticsearch + Logstash + Kibana），集中存储和分析应用日志、系统指标
- **APM 数据存储**：应用性能监控，存储调用链和指标数据（Elastic APM）
- **数据聚合分析**：按时间维度聚合统计，用户行为分析，BI 报表数据源
- **搜索推荐**：基于用户输入实时联想和补全（Completion Suggester）
- **时序数据**：配合 Rollover + ILM，可作为时间序列数据库使用

**不适用场景**：

- **需要 ACID 事务的场景**：ES 不支持事务。订单系统、账户余额等需要强一致性的场景不适合用 ES 做主存储。
- **频繁更新的数据**：ES 的段不可变设计导致更新代价高，每次更新实质是删除旧文档 + 插入新文档。频繁更新会导致大量段合并开销。
- **复杂的关联查询**：多表 JOIN、子查询不是 ES 的强项。虽然有 nested query 和 parent-child 支持，但性能远不及关系型数据库。
- **精确、低延迟的单条记录查询**：根据主键查单条数据，ES 的延迟高于 KV 存储（如 Redis），因为仍有查询协调、分片路由、段读取开销。
- **数据需要长期离线归档且较少查询**：ES 作为长期冷数据存储的成本太高（需要集群运行），更适合用对象存储归档。

## 二、倒排索引

倒排索引是搜索引擎的基石。理解它的内部结构和压缩技术，才能写出高效的查询语句和在面试中回答"ES 为什么快"这类问题。

### 2.1 正向索引 vs 倒排索引

**正向索引 (Forward Index)**：以文档 ID 为键，记录该文档包含哪些词条。这是 MySQL 等数据库的默认索引方式。对于 `SELECT * FROM articles WHERE title LIKE '%elasticsearch%'` 这种查询，只能全表扫描文档，逐一匹配。

**倒排索引 (Inverted Index)**：以词条（Term）为键，记录该词条出现在哪些文档中。当用户搜索 "elasticsearch"，直接通过 Term Dictionary 找到该词条对应的文档列表，无需扫描任何无关文档。

```
// 原始文档
Doc1: "Elasticsearch is a search engine"
Doc2: "Elasticsearch is built on Lucene"
Doc3: "Lucene is a Java search library"

// 倒排索引
"Elasticsearch" → [Doc1, Doc2]
"is"            → [Doc1, Doc2, Doc3]
"a"             → [Doc1, Doc3]
"search"        → [Doc1, Doc3]
"engine"        → [Doc1]
"built"         → [Doc2]
"on"            → [Doc2]
"Lucene"        → [Doc2, Doc3]
"Java"          → [Doc3]
"library"       → [Doc3]
```

查询 "search engine" 时，分别查 `search → [Doc1, Doc3]` 和 `engine → [Doc1]`，取交集得到 `[Doc1]`。查询复杂度取决于词条数量，而不是文档总数——这就是 ES 在亿级数据中毫秒级响应的根本原因。

### 2.2 Term Dictionary → Term Index (FST)

倒排索引的核心数据结构分为三级：

```
Term Index (FST 前缀树, 内存)
    ↓ 定位到范围
Term Dictionary (排序的 Term 列表, 可部分在磁盘)
    ↓ 定位到 Term
Posting List (文档ID列表, 磁盘)
```

**Term Dictionary（词项字典）**：所有词项按字典序排序存储。由于数据量大，不可能全部加载到内存。ES 通过 Term Index 来加速查找。

**Term Index（词项索引）**：使用 **FST (Finite State Transducer，有限状态转换器)** 实现。FST 是一种前缀压缩的数据结构，它将所有词项的前缀合并为一个有向无环图（DAG），极大压缩存储空间。

FST 的结构示例（简化）：

```
         (s)
    ┌─────┴─────┐
    t            e
    │            │
    a(1)    ┌────┴────┐
            r          a
            │          │
            v         r(4)
            │
           e(2)
            │
           r(3)
```

从根到叶子节点的路径就是词项的全部字符。FST 可以在节点上存储输出值（Term 在 Term Dictionary 中的位置），因此查找一个词项只需 O(len(term)) 时间，并且 FST 自身高度压缩，可完全放入内存。

> **面试要点**：FST 是 Lucene 的核心加速结构。它本质是一个确定性有限状态自动机（DFA），查找一个词项的时间复杂度仅与词项长度相关，与词项总数无关。

### 2.3 Posting List（文档列表）

Posting List 存储了每个 Term 出现的文档 ID 列表，除了文档 ID 外还包含：

- **Term Frequency (TF)**：该词在当前文档中出现的次数，用于相关性评分
- **Position**：词在文档中的位置列表，用于 Phrase Query（短语匹配）
- **Offset**：词的起止字符偏移，用于高亮（Highlight）
- **Payload**：每个词条的自定义元数据（如权重加成）

```
// 简化的 Posting 结构
Term: "elasticsearch"
  DocID: 1, TF: 2, Positions: [3, 15], Offsets: [(20,33), (105,118)]
  DocID: 5, TF: 1, Positions: [0],    Offsets: [(0,13)]
  DocID: 8, TF: 3, Positions: [7,21,35], ...
```

Posting List 按文档 ID 升序排列，这个特性对于布尔查询时的交集合并（跳表/Skip List 加速）非常重要。

### 2.4 压缩算法

文档 ID 列表如果直接存 int 数组，在亿级文档下会占用大量磁盘和内存。Lucene 使用以下压缩技术：

**FOR (Frame of Reference，参照帧压缩)**：

核心思想：文档 ID 是递增的，存储差值（Delta）比存储绝对值数值小很多。

```
原始: [1, 5, 9, 11, 20, 101]   → 每个占 4 字节 = 24 字节
差值: [1, 4, 4,  2,  9,  81]   → 大部分值很小
```

FOR 将文档列表按 128 个一组分块，每块找出最大值，确定该块需要的位宽（比如最大差值为 9，每数只需 4 bit），然后整个块按该位宽紧凑存储。

```
块 [1, 4, 4, 2, 9] → max=9 → 需要 4 bit/数 → 5个数 × 4 bit = 2.5 字节
```

空间节省显著——原来 128 个 int（512 字节）可能压缩到几十字节。

**RBM (Roaring Bitmap，咆哮位图)**：

当文档列表非常密集时（如高频词 "the" 几乎每篇文档都有），FOR 的效率下降。Lucene 5.0+ 引入 Roaring Bitmap。

RBM 将 32 位文档 ID 按高 16 位分桶，每个桶内的低 16 位根据稀疏度选择不同容器：

- **Array Container**：桶内文档数 ≤ 4096 时，用 `short[]` 有序数组存储
- **Bitmap Container**：桶内文档数 > 4096 时，用 `long[1024]` 位图存储（覆盖 65536 个值）
- **Run Container**：桶内文档连续时，用游程编码 `[start, length]` 压缩

```java
// RoaringBitmap 使用示例
RoaringBitmap rb = new RoaringBitmap();
rb.add(1);
rb.add(100);
rb.add(10000);

RoaringBitmap rb2 = new RoaringBitmap();
rb2.add(1);
rb2.add(200);

RoaringBitmap result = RoaringBitmap.and(rb, rb2); // 交集 → [1]
```

Lucene 的过滤查询（Filter）大量使用 RoaringBitmap。多个过滤条件求交集时，在 RoaringBitmap 层面直接进行高效的位运算，无需解压整份文档列表。

### 2.5 分析流程 (Analysis)

文档在建立倒排索引之前，需要经过分析器（Analyzer）处理。一个 Analyzer 由三部分组成：

```
原始文本 → Character Filter → Tokenizer → Token Filter → 词项序列
```

**1. Character Filter（字符过滤器）**：在分词前对原始文本做预处理。如 `html_strip` 去掉 HTML 标签，`mapping` 字符替换（`&` → `and`），`pattern_replace` 正则替换。

**2. Tokenizer（分词器）**：将文本切分为词项（Token）。这是最核心的一步。常见的 tokenizer：

- **standard**：基于 Unicode 文本分段算法（UAX #29），按词边界切分，转小写，去掉标点
- **whitespace**：仅按空白字符切分
- **keyword**：整段文本作为一个词项（适用于 not_analyzed 字段）
- **pattern**：按正则表达式切分
- **ik_max_word / ik_smart**：中文分词（IK Analyzer 插件）

**3. Token Filter（词项过滤器）**：对切分后的词项进一步处理。常见的有：

- **lowercase**：转小写
- **stop**：去除停用词（的、是、the、a 等无实义词）
- **stemmer / snowball**：词干提取（`running` → `run`, `cars` → `car`）
- **synonym**：同义词扩展（`pc` → `电脑, 计算机`）
- **shingle**：生成 N-gram shingle
- **pinyin**：中文转拼音（`中国` → `zhongguo, zhong, guo`）

```json
// 自定义 Analyzer 示例
PUT /my_index
{
  "settings": {
    "analysis": {
      "char_filter": {
        "my_char_filter": {
          "type": "mapping",
          "mappings": ["& => and", "| => or"]
        }
      },
      "tokenizer": {
        "my_tokenizer": {
          "type": "pattern",
          "pattern": "[\\s,]+"
        }
      },
      "filter": {
        "my_stopwords": {
          "type": "stop",
          "stopwords": ["the", "a", "an"]
        }
      },
      "analyzer": {
        "my_analyzer": {
          "char_filter": ["my_char_filter"],
          "tokenizer": "my_tokenizer",
          "filter": ["lowercase", "my_stopwords"]
        }
      }
    }
  }
}
```

### 2.6 内置分词器与中文分词

**standard 分词器**（默认）：按词边界切分，对英文良好，对中文会将每个汉字作为一个词项。

```
"Elasticsearch 深度解析"
→ standard: ["elasticsearch", "深", "度", "解", "析"]
→ 问题：中文被切成单字，搜索"深度"找不到该文档
```

**IK 分词器**（中文最主流）：`ik_max_word` 做细粒度切分，`ik_smart` 做粗粒度切分。

```
"中华人民共和国国歌"
→ ik_smart:     ["中华人民共和国", "国歌"]
→ ik_max_word:  ["中华人民共和国", "中华人民", "中华", "华人", "人民共和国", "人民", "共和国", "国歌"]
```

IK 分词器支持自定义扩展词典和停用词典，配置文件位于 `{ES_HOME}/plugins/ik/config/`。热更新词典通常通过远程词库实现。

**拼音分词器**：`pinyin` 插件将中文转为拼音，解决用户输入拼音搜索中文内容的需求。

```json
PUT /products
{
  "settings": {
    "analysis": {
      "analyzer": {
        "ik_pinyin_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["pinyin_filter", "lowercase"]
        }
      },
      "filter": {
        "pinyin_filter": {
          "type": "pinyin",
          "keep_full_pinyin": true,
          "keep_joined_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "lowercase": true
        }
      }
    }
  }
}
```

### 2.7 倒排索引的查询合并

布尔查询中，需要将多个 Term 的 Posting List 做交集（AND）、并集（OR）、差集（NOT）。Lucene 使用两种合并策略：

**直接合并（Conjunction）**：用于高频词（文档列表很短），直接遍历取交集即可。

**跳表加速（Skip List）**：对于低频词的超长文档列表，Lucene 在每个 Block（128 个文档 ID）上建立多层跳表，合并时可以在跳表层快速定位到公共文档 ID，跳过大量不必要的比较。

```
Posting List: 1, 5, 13, 21, 38, 52, ...  (每个 Block 128 文档)
Skip List L1: → 1 → 38 → 129 → ...        (每块第一个文档 ID)
Skip List L2: → 1 → 129 → 257 → ...        (每 8 块的第一个)
```

## 三、集群架构

ES 的分布式架构建立在多个节点角色和分片机制之上。理解集群架构对于故障排查和容量规划至关重要。

### 3.1 节点角色

从 ES 5.0 开始，节点角色细化，一个节点可以同时承担多种角色：

| 角色 | 配置参数 | 职责 |
|------|---------|------|
| Master (主节点) | `node.master: true` | 管理集群层面的变更（创建/删除索引，管理分片分配），维护集群状态 |
| Data (数据节点) | `node.data: true` | 存储数据，执行数据相关操作（CRUD、搜索、聚合） |
| Coordinating (协调节点) | `node.master: false, node.data: false, node.ingest: false` | 不存数据、不当主节点，仅负责路由请求、合并结果。类似负载均衡器 |
| Ingest (预处理节点) | `node.ingest: true` | 在索引文档之前执行预处理 Pipeline（字段加工、数据清洗） |
| MLAI (机器学习节点) | `node.ml: true` | 运行机器学习作业（异常检测、数据帧分析）（基础版不支持） |
| Transform (转换节点) | `node.transform: true` | 运行 Transform 作业（将聚合结果写入目标索引） |

**生产最佳实践**：

```yaml
# elasticsearch.yml — 专用 Master 节点
node.master: true
node.data: false
node.ingest: false

# elasticsearch.yml — 专用 Data 节点
node.master: false
node.data: true
node.ingest: false

# elasticsearch.yml — Coordinating-Only 节点
node.master: false
node.data: false
node.ingest: false
```

**原因**：Master 节点负责集群元数据管理和分片分配决策，这些操作需要全局一致性，并发冲突风险高（本质是单线程批处理）。如果 Master 节点同时存大量数据，GC 停顿可能导致该节点失联，触发不必要的重新选举。**线上必须配置 3 个及以上专用 Master 候选节点**。

### 3.2 分片分配与感知 (Allocation Awareness)

ES 通过分片感知（Shard Allocation Awareness）确保数据在不同物理机架上分布，避免单机架故障导致数据不可用：

```yaml
# 在每个节点上配置
node.attr.rack_id: rack1

# 在集群配置中启用感知
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.awareness.attributes": "rack_id",
    "cluster.routing.allocation.awareness.force.rack_id.values": "rack1,rack2,rack3"
  }
}
```

ES 会确保主分片和其副本分片不会被分配到同一个 `rack_id` 上。

更细粒度的分片分配过滤通过 `index.routing.allocation` 实现，例如将热数据索引只分配到 SSD 节点：

```json
PUT /hot_index/_settings
{
  "index.routing.allocation.include.box_type": "hot"
}
```

### 3.3 路由公式

文档写入或查询时，ES 通过以下公式决定将请求发送到哪个分片：

```
shard_num = hash(_routing) % num_primary_shards
```

- `_routing` 默认是文档的 `_id`，也可以在写入或查询时显式指定。**指定 routing 可以将相同 routing 值的文档路由到同一分片**，这对某些需要文档间关联的场景非常有用。
- 取模的分母是主分片数，**这就是为什么主分片数创建后不能修改**——改了分片数，取模的结果就变了，已有的文档将找不到。

```java
// Java REST Client 指定 routing
IndexRequest request = new IndexRequest("orders")
    .id("order_123")
    .routing("user_456")  // 同一用户的订单路由到同一分片
    .source("user_id", "user_456", "amount", 99.9);

// 查询时也必须指定相同的 routing
SearchRequest searchRequest = new SearchRequest("orders")
    .routing("user_456");
```

**自定义 routing 的代价与收益**：

- **收益**：同一 routing 值的文档在同一分片，可以进行聚合、排序，或利用父子文档关系。查询时只需搜索一个分片，减少分散请求。
- **代价**：数据可能分布不均（数据倾斜），某些分片远超其他分片，产生热点问题。

### 3.4 Zen Discovery 与节点发现

ES 7.x 使用 Zen Discovery 机制进行节点发现和集群组建：

**1. 单播发现 (Unicast Ping)**：

节点尝试连接 `discovery.seed_hosts` 列表中配置的其他节点。一旦与任一节点建立连接，该节点会返回它所知道的所有节点的列表。新节点据此连接所有已知节点，获取完整集群状态。

```yaml
# elasticsearch.yml — ES 7.x 配置
discovery.seed_hosts:
  - es-node1:9300
  - es-node2:9300
  - es-node3:9300
```

**2. Master 选举**：

当一个 Master 候选节点发现当前没有活跃的 Master 或 Master 下线，它会开始选举。ES 7.x 使用类 Bully 算法的改进版——基于集群状态版本和节点 ID 的优先顺序进行投票。

**3. 脑裂 (Split Brain) 与 quorum**：

当网络分区发生时，如果原集群被分割成多个子网，每个子网都以为自己是一个独立集群，就会选举自己的 Master，出现两个 Master 同时写入——这就是脑裂。

**ES 7.x 的 solution**：使用 `discovery.seed_hosts` 配合基于法定人数（quorum）的选举机制。Master 候选节点数为 N，则需要至少 N/2+1 个候选节点参与才能选举成功。如果子网中的候选节点数不足半数，该子网不会选举 Master，拒绝写入。

```yaml
# 建议配置 3 个 Master 候选节点，N/2+1 = 2
# 任意 2 个 Master 节点组成的子网可以选举；1 个 Master 节点的子网无法选举
cluster.initial_master_nodes:
  - es-master1
  - es-master2
  - es-master3
```

> **ES 8.x 的演变**：ES 8.x 中，Zen Discovery 被更简化的配置替代。Master 选举中引入了一个特殊的 Voting-Only 节点角色（`node.voting_only: true`），它参与投票但不参与 Master 竞争，用于增加可用 Master 候选节点数而不需要全功能 Master。

**4. Cluster State（集群状态）**：

集群状态是 ES 最核心的元数据，包含：索引元数据（Mapping、Setting）、节点列表、分片路由表（Routing Table）、集群配置。集群状态是全局唯一的，由 Master 节点维护和发布。

集群状态更新的流程：
1. Master 接收状态变更请求
2. Master 计算变更后的目标状态
3. Master 将新的集群状态发布给所有节点（两阶段提交： Commit → Apply）
4. 大多数 Master 候选节点确认后，状态生效

节点数量超过数百时，集群状态的分发可能成为瓶颈——这就是为什么需要限制单个集群规模。

```bash
# 查看集群状态大小（通常应 < 100MB）
GET /_cluster/state/_all/human=true
```

## 四、写入流程

ES 的写入流程体现了其分布式、近实时的设计哲学。深入理解写入路径，是优化写入性能和排查数据丢失问题的基础。

### 4.1 写入路径详解

一个文档从客户端写入到可被搜索，经历了以下路径：

```
Client → 协调节点 → 主分片 → 副本分片
                      ↓
                   Translog (持久化, 顺序写)
                   Buffer (内存) → Refresh → Segment (可搜索)
                                              ↓
                                        Flush → 磁盘 (commit)
                   Segment → Merge → 大段
```

**分步详解**：

**Step 1. 请求路由**：客户端发送索引请求到任意节点（协调节点）。协调节点根据路由公式计算目标分片：

```
shard = hash(_routing) % num_primary_shards
```

**Step 2. 主分片写入**：协调节点将请求转发到主分片所在的数据节点。主分片执行以下操作：

(a) 将文档写入 **内存 buffer**（此时还不可被搜索）
(b) 将文档追加写入 **Translog**（顺序写，速度很快，类似于 MySQL binlog/redo log）
(c) 检查 Translog 是否需要 fsync（由 `index.translog.durability` 控制）

**Step 3. 副本分片同步**：主分片写入成功后，并行将请求发送到所有副本分片所在的节点。副本分片执行与主分片相同的 (a)(b)(c) 步骤。当所有副本分片都成功响应后，主分片向协调节点返回成功。

**Step 4. 返回客户端**：协调节点收集到所有必需分片的成功响应后，向客户端返回成功。

```
时间线:
Client ──→ Coordinating ──→ Primary Shard ──→ [Replica1, Replica2]
   ↑                                      ↑
   └────── Wait for all acks ←───────────┘
```

**写一致性级别** (`wait_for_active_shards`)：

```json
PUT /my_index/_doc/1?wait_for_active_shards=all
{
  "title": "Elasticsearch Guide"
}
```

- `1`（默认）：仅主分片写入成功即返回
- `all`：所有副本也必须写入成功才返回（最高可靠性）
- `N`：至少 N 个分片（含主分片）写入成功

### 4.2 Near Real-Time (近实时搜索)

ES 写入的文档并非立即可被搜索。从写入到可被查询，中间有一个 **Refresh** 过程：

```
内存 Buffer → Refresh → 新 Segment (内存中，fsync 前也可搜索)
```

**Refresh 间隔默认 1 秒**（`index.refresh_interval: 1s`）。每秒钟 ES 将内存 Buffer 中的数据写入一个新的 Segment 文件，该 Segment 存储在文件系统缓存（OS Page Cache）中，虽然还没有 fsync 到磁盘，但已经可以对外提供搜索。

**为什么这不是实时而是近实时？**

- Refresh 不是每次写入都触发（代价太高），而是按固定的间隔周期性执行
- Segment 在 Refresh 后才对搜索可见
- 写入到可搜索的延迟 ≈ Refresh 间隔 = 默认 1 秒

**生产调优**：

```json
// 大批量写入时，关闭 Refresh 以减少开销
PUT /my_index/_settings
{
  "index.refresh_interval": "-1"
}

// 写入完成后恢复
PUT /my_index/_settings
{
  "index.refresh_interval": "30s"
}
```

对于极高吞吐的日志写入场景，可以将 `refresh_interval` 设为 30s 甚至 60s，大幅减少 Segment 碎片化，降低 Merge 压力。

### 4.3 Translog（事务日志）

Translog 是 ES 防止数据丢失的关键机制。类比 MySQL InnoDB 的 Redo Log：

```
写入时: 文档 → 内存 Buffer (可能丢失) + Translog (持久化, 可恢复)
```

**Translog 的两个持久化级别**：

```json
PUT /my_index/_settings
{
  "index.translog.durability": "request",  // 默认
  "index.translog.sync_interval": "5s"
}
```

| 设置 | 说明 | 可靠性 | 性能 |
|------|------|--------|------|
| `request` (默认) | 每次写入后 fsync 到磁盘 | 高（类似 MySQL sync_binlog=1） | 较低 |
| `async` | 每 `sync_interval` (默认 5s) fsync 一次 | 中（最多丢 5s 数据） | 高 |

**Translog 何时被清空？**

当 ES 执行 **Flush** 操作时，会做以下事情：
1. 将所有内存中的 Segment 持久化（fsync）到磁盘
2. 创建一个新的 Commit Point 文件
3. 清空（或截断）旧的 Translog

Flush 自动触发条件：Translog 达到 `index.translog.flush_threshold_size`（默认 512MB），或经过指定时间（默认 30 分钟）。

**故障恢复**：节点宕机重启后，ES 从最后一个 Commit Point 恢复 Segment，然后重放 Translog 中尚未 Flush 的操作，恢复丢失的文档。这个机制与 InnoDB 的 crash recovery 原理完全一致。

### 4.4 段合并 (Segment Merge)

ES/Lucene 的一个核心设计理念是段（Segment）的不可变性——一旦写入，不再修改。删除文档并非物理删除，而是在 `.del` 文件中标记为已删除；更新文档实质是删除旧文档 + 写入新文档。

随着写入增加，Segment 数量不断增长，会导致：
- 每次搜索需要遍历所有 Segment
- 文件描述符（FD）开销增大
- 标记删除的文档占用空间

**自动合并**：ES 在后台自动执行 Merge，将多个小 Segment 合并为大 Segment，同时物理删除已标记删除的文档。Merge 策略由 TieredMergePolicy 控制。

```json
// 查看当前 Segment 状态
GET /_cat/segments/my_index?v

// 手动触发 Force Merge (仅用于不再写入的索引)
POST /my_index/_forcemerge?max_num_segments=1
```

**Force Merge 注意事项**：
- 仅用于不再写入的只读索引（如归档索引）
- 会产生大量 I/O，应在低峰期执行
- 不会释放已分配的内存（Segment 被加载到 Page Cache）
- `max_num_segments=1` 将所有 Segment 合并为 1 个，查询性能最优

### 4.5 写入性能优化

**1. Bulk API 批量写入**：

```java
// Java High Level Client 批量写入
BulkRequest bulkRequest = new BulkRequest();
for (int i = 0; i < 10000; i++) {
    bulkRequest.add(new IndexRequest("my_index")
        .id(String.valueOf(i))
        .source(XContentType.JSON, "field1", "value" + i, "field2", i));
}
// 每批 5000-10000 条，字节数控制在 5-15MB
BulkResponse response = client.bulk(bulkRequest, RequestOptions.DEFAULT);
if (response.hasFailures()) {
    // 处理失败
}
```

**2. 调整 Refresh 间隔**：

```json
PUT /my_index/_settings
{
  "index.refresh_interval": "30s"
}
```

**3. 调整 Translog**：

```json
PUT /my_index/_settings
{
  "index.translog.durability": "async",
  "index.translog.sync_interval": "30s",
  "index.translog.flush_threshold_size": "1024mb"
}
```

**4. 减少副本（写入时）**：

```json
// 大批量导入前关闭副本，导入完再开启
PUT /my_index/_settings
{
  "index.number_of_replicas": 0
}
```

**5. 自动生成文档 ID**：让 ES 自动生成 ID 比自定义 ID 更快，因为 ES 可以跳过 ID 唯一性检查。

**6. 使用 SSD**：磁盘 I/O 是写入的关键路径（Translog 需要 fsync），SSD 延时是 HDD 的 1/100 ~ 1/1000。

**7. 合理设置分片数**：单个分片大小建议 10-50GB（日志场景可到 100GB）。分片数过多，单次请求需要协调过多分片；分片数过少，无法充分利用集群并行能力。

## 五、查询流程

查询是 ES 最核心的功能。理解查询的完整链路、评分机制和分页策略，是写出高效查询的基础。

### 5.1 Query vs Filter

ES 的查询分为两个基本类型：

| 维度 | Query (查询) | Filter (过滤) |
|------|-------------|--------------|
| 评分 | 计算相关性评分 (_score) | 不评分 |
| 缓存 | 不缓存 | 自动缓存 (Filter Cache) |
| 速度 | 相对慢 | 快 |
| 典型场景 | 全文搜索、需要排序 | 条件筛选、精确匹配 |
| DSL 位置 | `query` 块 | `filter` 块，必须在 `bool` 之内 |

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "手机" } }        // Query: 需要评分
      ],
      "filter": [
        { "term": { "brand": "Apple" } },       // Filter: 精确过滤
        { "range": { "price": { "gte": 2000, "lte": 8000 } } }
      ]
    }
  }
}
```

**Filter Cache 机制**：

ES 将 Filter 条件对应的文档集合用 RoaringBitmap 紧凑存储，缓存在节点级别的 Filter Cache 中。下次相同 Filter 直接命中缓存，无需再次遍历倒排索引。

- 缓存键 = Filter 条件的序列化表示
- 缓存值 = 匹配文档 ID 的 RoaringBitmap
- 淘汰策略：LRU，受 `indices.queries.cache.size`（默认 10% heap）限制
- 仅在 Segment 级别缓存，Segment 合并后缓存自动失效

**生产建议**：将不需要评分的条件一律放入 `filter` 而非 `must`。`bool` 查询中，`filter` 子句先于 `must` 子句执行，先缩小范围再评分，性能更优。

### 5.2 相关性评分：TF-IDF 与 BM25

相关性评分决定了搜索结果中哪些文档排在前面。Lucene 最初使用 TF-IDF，后来（Lucene 6.0+, ES 5.0+）默认改为 BM25。

**TF-IDF (Term Frequency - Inverse Document Frequency)**：

```
score(D, Q) = Σ TF(t,D) × IDF(t) × fieldNorm(D)
```

- **TF (词频)**：Term 在文档 D 中出现的次数。次数越多越相关。
- **IDF (逆文档频率)**：log(总文档数/含该词的文档数)。出现在越少文档中的词越重要。
- **fieldNorm**：文档长度归一化。短文档中出现的词权重更高。

TF-IDF 的问题是：TF 线性增长没有上限，长文档的重复词会获得不合理的高分。

**BM25 (Best Match 25)**：

```
BM25(D, Q) = Σ IDF(qi) × [f(qi,D) × (k1+1)] / [f(qi,D) + k1 × (1 - b + b × |D|/avgdl)]
```

其中：
- **k1**（默认 1.2）：控制词频饱和度。TF 值增长到一定程度后对评分的影响趋于饱和
- **b**（默认 0.75）：控制文档长度归一化程度。0 = 不归一化，1 = 完全归一化
- **|D|/avgdl**：文档长度与平均文档长度的比值

BM25 的核心改进：
- **非线性词频饱和**：TF 从 1 到 2 的影响很大，从 10 到 11 的影响几乎为零。这更符合直觉——一篇文档中出现 10 次 "Elasticsearch" 和出现 100 次，相关性差异并不大。
- **文档长度归一化更合理**：通过参数 `b` 调节长度惩罚的程度，不同场景可以灵活配置。

```json
// 使用 BM25 参数调优 (ES 5.0+)
PUT /my_index
{
  "settings": {
    "index": {
      "similarity": {
        "my_bm25": {
          "type": "BM25",
          "k1": 1.2,
          "b": 0.75
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "content": {
        "type": "text",
        "similarity": "my_bm25"
      }
    }
  }
}
```

> **面试要点**：BM25 相比 TF-IDF 的核心改进是**非线性词频饱和度**——TF 对评分的影响在达到阈值后递减，避免了长文档中重复词的评分膨胀。

### 5.3 Search Type：query_then_fetch

ES 的搜索流程分为两个阶段，这也是默认的 `search_type` —— `query_then_fetch`：

**Phase 1: Query 阶段**

```
Coordinating Node
  │
  ├─→ Shard 1 (Primary/Replica): Query → 返回 [docIds + sort values]
  ├─→ Shard 2 (Primary/Replica): Query → 返回 [docIds + sort values]
  └─→ Shard 3 (Primary/Replica): Query → 返回 [docIds + sort values]
      每个分片返回 top N (from+size) 个文档的 ID 和排序值

Coordinating Node: 合并排序 → 选出全局 top N 个文档 ID
```

**Phase 2: Fetch 阶段**

```
Coordinating Node
  │
  ├─→ Shard 1: Fetch [docId1, docId5, docId8] → 返回完整 _source
  ├─→ Shard 2: Fetch [docId3, docId4]          → 返回完整 _source
  └─→ Shard N: ...

Coordinating Node: 组装最终结果集，返回客户端
```

注意：Query 阶段每个分片返回 from+size 个结果，而非仅 size 个。例如查询 `from=90, size=10`，每个分片返回 100 个文档 ID，协调节点合并排序后再取第 91-100 个。

### 5.4 分页策略

**1. from + size（浅分页）**：

默认分页方式，适用于查看前几页结果的场景。原理如上所述，每个分片返回 from+size 个文档，协调节点合并排序。

**性能问题**：`from=10000, size=10` 时，每个分片需要返回 10010 个文档，在协调节点排序。分片越多、翻页越深，内存和 CPU 开销越大。ES 默认限制 `from + size ≤ 10000`（由 `index.max_result_window` 控制）。

**2. Scroll（深度分页 / 遍历）**：

Scroll 相当于在内存中生成一个当前数据快照（Snapshot），与后续写入的数据隔离。适合数据导出、全量拉取等场景。

```json
// 创建 Scroll
POST /my_index/_search?scroll=1m
{
  "size": 1000,
  "query": { "match_all": {} }
}
// 后续页通过 scroll_id 获取
POST /_search/scroll
{
  "scroll": "1m",
  "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAD4WYm9laVYtZndUQlNsdDcwakFMNjU1QQ=="
}
// 清除 Scroll (释放资源)
DELETE /_search/scroll
{
  "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAD4WYm9laVYtZndUQlNsdDcwakFMNjU1QQ=="
}
```

**Scroll 的代价**：占用宝贵的 JVM 堆内存和文件句柄。每个 Scroll 上下文是一个时间点快照，大量并发 Scroll 会消耗 Search Context 资源。务必在完成后显式清除 Scroll。

**3. search_after（实时深度分页）**：

ES 5.0+ 引入，基于上一页最后一条文档的排序值获取下一页，是深度分页的推荐方案。

```json
// 第一页
GET /my_index/_search
{
  "size": 10,
  "query": { "match_all": {} },
  "sort": [
    { "date": "desc" },
    { "_id": "asc" }      // _id 作为 tiebreaker（必须唯一）
  ]
}
// Response: 最后一条的 sort 值为 [1609459200000, "abc123"]

// 第二页：使用上一页最后一条的 sort 值
GET /my_index/_search
{
  "size": 10,
  "query": { "match_all": {} },
  "sort": [
    { "date": "desc" },
    { "_id": "asc" }
  ],
  "search_after": [1609459200000, "abc123"]
}
```

相比 Scroll，search_after 不创建快照，无额外资源开销。缺点是无法跳页（只能顺序翻页）。

**4. Point in Time (PIT) + search_after**：

ES 7.10+ 引入，是 search_after 的增强版。PIT 为搜索创建一个轻量级的时间点视图，保证翻页过程中数据一致性，比 Scroll 更轻量。

```json
// 创建 PIT
POST /my_index/_pit?keep_alive=5m
// Response: { "id": "pxA0...AbBz" }

// 分页搜索
GET /_search
{
  "size": 10,
  "query": { "match_all": {} },
  "pit": { "id": "pxA0...AbBz", "keep_alive": "5m" },
  "sort": [{"@timestamp": "desc"}, {"_shard_doc": "asc"}],
  "search_after": [1609459200000, 123]
}

// 关闭 PIT
DELETE /_pit
{
  "id": "pxA0...AbBz"
}
```

### 5.5 聚合 (Aggregation)

聚合是 ES 除了全文搜索之外最强大的功能，能替代部分 OLAP 场景。

**Bucket Aggregation（分桶聚合）**：按条件将文档分组，类似 SQL 的 `GROUP BY`。

- **terms**：按字段值聚合（最常用）
- **range / date_range**：按数值/日期范围分组
- **histogram / date_histogram**：按固定间隔分组
- **filter / filters**：按过滤条件分组
- **nested / reverse_nested**：处理嵌套文档

**Metric Aggregation（指标聚合）**：对分组内文档做数值计算。

- **avg / max / min / sum**：统计值
- **stats / extended_stats**：基本统计 + 方差/标准差
- **cardinality**：近似去重统计（基于 HyperLogLog 算法）
- **percentiles**：百分位数（基于 T-Digest 算法）
- **top_hits**：每个桶内取 top N 文档

**Pipeline Aggregation（管道聚合）**：对聚合结果进行二次计算。

- **derivative**：差值（环比）
- **moving_avg**：移动平均
- **cumulative_sum**：累计求和
- **bucket_script**：自定义桶间计算

```json
// 综合聚合示例: 按商品分类统计销量与均价
GET /orders/_search
{
  "size": 0,
  "query": {
    "range": { "date": { "gte": "2026-01-01", "lte": "2026-06-30" } }
  },
  "aggs": {
    "by_category": {
      "terms": { "field": "category_id", "size": 10 },
      "aggs": {
        "total_sales": { "sum": { "field": "amount" } },
        "avg_price": { "avg": { "field": "unit_price" } },
        "sales_trend": {
          "date_histogram": {
            "field": "date",
            "calendar_interval": "month"
          },
          "aggs": {
            "monthly_sales": { "sum": { "field": "amount" } }
          }
        }
      }
    }
  }
}
```

**聚合性能调优点**：

- 聚合非常消耗内存，`size: 0` 关闭返回文档，只返回聚合结果
- `terms` 聚合的 `size` 不宜太大，`shard_size` 影响精度但随之而来是内存开销
- `cardinality` 使用 HyperLogLog++ 算法，可接受 1-6% 误差，内存占用固定
- 尽可能在 `filter` 中缩小范围再进行聚合

### 5.6 查询性能分析

**Profile API**：分析查询各部分耗时：

```json
GET /my_index/_search
{
  "profile": true,
  "query": { "match": { "title": "elasticsearch" } }
}
```

输出会展示每一步的耗时（`time_in_nanos`）、Lucene 查询内部细节（如使用了什么合并算法）、查询结果数量等，是定位慢查询的第一步。

**Explain API**：解释某文档为何匹配及评分：

```json
GET /my_index/_explain/1
{
  "query": { "match": { "title": "elasticsearch" } }
}
```

输出详细的 TF、IDF、fieldNorm 各分量值，帮助理解 BM25 的评分过程。

**查询优化建议**：

1. 用 `filter` 替代 `must` 中不需要评分的条件
2. 合理设置 `size` 和 `from`，避免深度分页
3. 查询字段使用合适的类型，`keyword` 查询比 `text` 快
4. 利用 `routing` 将查询限制在特定分片
5. 避免在高基数 `keyword` 字段上做 `terms` 聚合（`shard_size` 设小）
6. 使用 `index sorting` 预排序（写入时排序，查询时利用排序信息加速）
7. 合并只读索引为 1 个 segment（`_forcemerge`）

## 六、索引优化

索引设计直接影响写入速度、查询性能和存储成本。好的设计在初期就能避免大量后期迁移工作。

### 6.1 Mapping 设计

Mapping 定义了索引中字段的类型、索引方式和分析器。不当的 Mapping 是生产问题的重灾区。

**核心字段类型选择**：

| 类型 | 使用场景 | 注意事项 |
|------|---------|---------|
| `keyword` | 精确匹配、聚合、排序 | 不会分词，整体作为一个 Term |
| `text` | 全文搜索 | 会被分析器分词，不适合聚合排序 |
| `long / integer / short / byte` | 整数数值 | 选最小的够用的类型 |
| `float / double / half_float / scaled_float` | 浮点数 | `scaled_float` 内部用 long 存，适合价格类 |
| `boolean` | 布尔值 | `true` / `false` |
| `date` | 日期时间 | 支持多种格式，底层存 long (epoch millis) |
| `keyword` vs `text` | 字符串 | 不确定时两个都定义，用 `fields` 子字段 |
| `geo_point` | 地理位置 | 经纬度坐标 |
| `ip` | IP 地址 | 支持 CIDR 查询 |
| `binary` | 二进制 | Base64 编码存储 |

**Dynamic Mapping 控制**：

默认情况下 ES 会自动推断新字段类型，但推断结果不一定符合预期。例如日期字符串 `"2026-07-05"` 会被推断为 `date` 类型，可能导致后续写入不兼容。

```json
// 控制 dynamic mapping 行为
PUT /my_index
{
  "mappings": {
    "dynamic": "strict",     // strict: 拒绝未定义字段
                             // true: 自动创建字段 (默认)
                             // false: 忽略未定义字段
                             // runtime: 动态创建 runtime fields
    "properties": {
      "id": { "type": "keyword" },
      "name": {
        "type": "text",
        "fields": {          // 子字段: 同时支持全文搜索和聚合
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      },
      "created_at": { "type": "date" }
    }
  }
}
```

**常见的 Mapping 参数**：

- `index: true/false`：是否建索引（false 则该字段不可搜索，仅可存储）
- `doc_values: true/false`：是否开启列存储（用于聚合/排序，默认 true）
- `enabled: true/false`：是否索引该对象（false 则整个 JSON 对象不可搜索）
- `eager_global_ordinals: true`：预加载全局序数（加速聚合，但增加内存）
- `norms: false`：关闭评分归一化信息（`keyword` 默认 false，节省空间）
- `ignore_above`：超过该长度的字符串不被索引（但可存储）
- `copy_to`：合并到另一个字段做搜索

```json
// 多字段搜索优化：将多个字段合并到一个字段
PUT /products
{
  "mappings": {
    "properties": {
      "name":    { "type": "text", "copy_to": "all_fields" },
      "brand":   { "type": "text", "copy_to": "all_fields" },
      "description": { "type": "text", "copy_to": "all_fields" },
      "all_fields": { "type": "text" }
    }
  }
}
```

### 6.2 索引模板 (Index Template)

索引模板用于自动为新创建的索引应用预定义的 Setting 和 Mapping。在日志场景（每天一个新索引）中必不可少。

```json
// Component Template (可复用组件)
PUT /_component_template/logs_mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "level":      { "type": "keyword" },
        "message":    { "type": "text" },
        "host":       { "type": "keyword" }
      }
    }
  }
}

PUT /_component_template/logs_settings
{
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index.refresh_interval": "30s"
    }
  }
}

// Composable Index Template (ES 7.8+)
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "priority": 100,
  "composed_of": ["logs_mappings", "logs_settings"],
  "template": {
    "settings": {
      "index.lifecycle.name": "logs_policy"
    }
  }
}
```

### 6.3 索引别名 (Index Alias)

别名是 ES 中的一个核心概念，它提供一个间接层，使得可以无缝切换底层索引而不影响应用。

```json
// 创建别名指向索引
POST /_aliases
{
  "actions": [
    { "add": { "index": "orders-2026-07", "alias": "orders" } }
  ]
}

// 原子切换：移除旧索引别名 + 添加新索引别名
POST /_aliases
{
  "actions": [
    { "remove": { "index": "orders-2026-06", "alias": "orders" } },
    { "add":    { "index": "orders-2026-07", "alias": "orders" } }
  ]
}

// 应用始终对 orders 别名读写，不知底层是哪个具体索引
```

别名支持过滤器和路由：

```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "users",
        "alias": "vip_users",
        "filter": { "term": { "is_vip": true } },
        "routing": "user_id"
      }
    }
  ]
}
```

### 6.4 Rollover 与 ILM

ILM (Index Lifecycle Management) 是 ES 索引生命周期管理的核心功能，实现 Hot-Warm-Cold 三层架构：

```
Hot (热)          →   Warm (温)       →   Cold (冷)       →   Delete (删除)
当前写入的索引        查询频率降低         几乎不查询           数据过期
SSD, 高性能         降低副本, ForceMerge   迁移到慢存储         彻底删除
```

**Rollover**：当索引达到一定条件时自动创建新索引：

```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "logs-000001",
        "alias": "logs-write",
        "is_write_index": true
      }
    },
    {
      "add": {
        "index": "logs-000001",
        "alias": "logs-read"
      }
    }
  ]
}

POST /logs-write/_rollover
{
  "conditions": {
    "max_age": "7d",
    "max_docs": 10000000,
    "max_size": "50gb",
    "max_primary_shard_size": "25gb"
  }
}
```

**ILM Policy 完整示例**：

```json
PUT /_ilm/policy/logs_policy
{
  "phases": {
    "hot": {
      "min_age": "0ms",
      "actions": {
        "rollover": {
          "max_age": "1d",
          "max_size": "50gb"
        },
        "set_priority": {
          "priority": 100
        }
      }
    },
    "warm": {
      "min_age": "3d",
      "actions": {
        "forcemerge": { "max_num_segments": 1 },
        "shrink": { "number_of_shards": 1 },
        "allocate": {
          "require": { "box_type": "warm" }
        },
        "set_priority": { "priority": 50 }
      }
    },
    "cold": {
      "min_age": "30d",
      "actions": {
        "allocate": {
          "require": { "box_type": "cold" },
          "number_of_replicas": 0
        },
        "freeze": {},
        "set_priority": { "priority": 0 }
      }
    },
    "delete": {
      "min_age": "90d",
      "actions": {
        "delete": {}
      }
    }
  }
}
```

### 6.5 冷热分离硬件配置

实现冷热分离的核心是将不同生命周期的索引分配到不同的硬件节点上：

```yaml
# Hot 节点 — SSD, 高性能
node.attr.box_type: hot
node.data: true
node.master: false

# Warm 节点 — HDD, 大容量
node.attr.box_type: warm
node.data: true
node.master: false

# Cold 节点 — 廉价存储
node.attr.box_type: cold
node.data: true
node.master: false
```

在 ILM Policy 的 `allocate` action 中通过 `require.box_type` 控制索引迁移。冷节点可将副本数设为 0，甚至使用可搜索快照（Searchable Snapshot）将索引存储在对象存储（S3/MinIO）中进一步降低成本。

### 6.6 索引性能优化总结

1. **Mapping 设计前置**：直接使用 `dynamic: strict`，强制显式定义字段类型
2. **避免过多字段**：单个索引字段数控制在 1000 以内。字段爆炸（如动态 key）会导致 mapping 膨胀，集群状态过大
3. **使用 `_source` 过滤**：只返回需要的字段，节省网络带宽和反序列化开销
4. **禁用默认不必要特性**：`_all`（7.x 已移除）、`_field_names`
5. **控制分片大小**：10-50GB/分片为黄金值。过大则迁移恢复慢，过小则协调开销大
6. **Close/Freeze 不常用索引**：减少打开的分片数，降低集群堆内存压力

## 七、分布式一致性

ES 是一个分布式系统，需要在 CAP 中做权衡。理解它的数据一致性和集群协调机制，对于处理生产故障至关重要。

### 7.1 数据一致性模型

ES 采用 **Primary-Backup** 模型保证数据一致性：

**写入路径**（同步写）：

1. 协调节点将写请求路由到主分片
2. 主分片写入成功后，将请求转发给所有 In-Sync 副本分片
3. 主分片等待所有 In-Sync 副本的确认
4. 全部确认后才向协调节点返回成功

**读请求**（可从主或副本读）：

ES 的读请求可以从主分片或副本分片读取。默认情况下协调节点采用轮询（Round-Robin）选择分片副本，以均衡负载。

**潜在的不一致风险**：

由于 Refresh 间隔（默认 1s），写入成功的文档在主分片上可能尚未被 Refresh，因此在副本分片上同样不可见。这是 ES 的**近实时特性**，与数据安全性是两回事——Translog 保证了持久性，Refresh 只影响可见性。

**写一致性**可以通过 `wait_for_active_shards` 参数控制。极端场景下（网络分区），如果主分片被隔离且写入继续，而少数副本所在的子网选举了新 Master 并将副本提升为主分片，就会产生数据不一致（split-brain 写入）。

### 7.2 脑裂处理

**脑裂的成因**：集群中部分节点网络断开，分成多个子网。每个子网认为 Master 已经宕机，各自选举新 Master，出现多个 Master 并行写入——这就是脑裂。

**ES 7.x 的预防机制**：

1. **法定人数（Quorum）**：Master 选举需要至少 N/2+1 个 Master 候选节点参与。配置 3 个 Master 候选节点时，任何一个子网必须有至少 2 个 Master 候选节点才能选举。网络分裂成两个子网时，最多只有一个子网拥有 ≥2 个 Master 节点，因此最多只有一个子网能选举成功。

2. **`discovery.zen.minimum_master_nodes`**（ES 7.x 中已废弃，由 cluster coordination 替代）：这个参数确保了一个节点在加入集群时至少能看到 `minimum_master_nodes` 个 Master 候选节点，否则不会加入。

**ES 7.x/8.x 的新协调层**（基于 Raft 协议的改进版）：

ES 7.x 重写了集群协调（Cluster Coordination）层，引入了一个正式的容错共识机制：
- 使用基于 Paxos 的变体，不依赖外部服务（如 ZooKeeper）
- 自动配置 `initial_master_nodes` 在第一次集群启动时
- Voting-Only 节点参与投票但不参与 Master 竞争，用于更好地分配投票权重
- 集群变更经过一个两阶段提交过程，确保一致性

```yaml
# 启动时指定初始 Master 候选节点
cluster.initial_master_nodes:
  - es-master1
  - es-master2
  - es-master3
```

### 7.3 集群状态变更过程

集群状态变更（比如创建索引、分配分片、节点加入/离开）遵循以下流程：

1. **提交变更**：Master 节点接收变更请求，放入待处理队列
2. **计算目标状态**：Master 根据当前状态 + 变更请求，计算新的集群状态
3. **发布状态**：Master 将新集群状态广播给所有节点（两阶段提交）
   - **Commit Phase**：Master 将新状态发送给所有节点。节点接收后验证并确认可接受该状态
   - **Apply Phase**：当大多数 Master 候选节点确认后，Master 发送 Apply 指令，所有节点应用新状态
4. **确认返回**：各节点应用完毕后向 Master 发送 ACK。大多数节点 ACK 后，Master 认为该变更已完成

**分片分配延迟**：节点加入退出后，Master 不会立即重新分配分片，而是有一个延迟（`cluster.routing.allocation.node_concurrent_recoveries` 和 `index.unassigned.node_left.delayed_timeout`，默认 1 分钟）。这是为了防止网络抖动导致不必要的分片迁移——短暂的节点失联后恢复，无需重新分配整个分片。

### 7.4 数据一致性问题的实际影响

**场景 1：主分片写入成功，副本同步失败**

假设 `wait_for_active_shards=1`（默认），主分片写入成功即返回。如果此时主分片所在节点宕机，刚写入的文档还未同步到副本。系统会从副本中选举一个 In-Sync 副本提升为主分片——但该副本没有最新数据，文档丢失。

**解决方案**：对数据一致性要求高的场景，设置 `wait_for_active_shards=all`，确保所有 In-Sync 副本都写入成功才返回。

**场景 2：客户端认为写入失败，但 ES 已写入成功**

客户端发送写入请求，主分片已写入成功，但在返回响应的网络传输中包丢失，客户端收到超时。客户端可能重试，导致重复写入。

**解决方案**：使用外部唯一 ID，ES 按 ID 覆盖是幂等的（索引操作用 PUT + 相同 ID）。或者使用 `_version` 实现乐观锁（Optimistic Concurrency Control）。

```java
// 乐观锁: 只有文档当前版本为 3 时才更新
UpdateRequest request = new UpdateRequest("my_index", "1")
    .doc(XContentType.JSON, "field", "new_value");
request.setIfSeqNo(3L);
request.setIfPrimaryTerm(1L);
```

## 八、生产实践

这一章聚焦日常运维、监控、排障和容量规划，内容来自生产环境中的常见场景。

### 8.1 集群规划

**内存**：

- ES 依赖 JVM 堆内存和 OS Page Cache。**堆内存上限设置为物理内存的 50%，且不超过 32GB**（32GB 是 JVM 指针压缩的阈值，超过后 64 位指针占双倍空间）
- `Xms` 和 `Xmx` 设为相同值，防止堆扩容引发 GC 时间长
- 剩余 50% 物理内存留给 OS Page Cache 缓存 Lucene 段数据（Lucene 大量依赖 Page Cache 来加速磁盘 I/O）

```yaml
# jvm.options
-Xms16g
-Xmx16g

# 堆内存不超过 32GB（压缩 OOP 阈值）
# 也不宜低于 8GB（除非数据量极小）
```

**磁盘**：

- **SSD 是生产必备**。Translog 需要顺序写入，HDD 尚可；但 Segment Merge 和查询的随机读必须依赖 SSD
- 每个数据节点的磁盘容量建议 ≤ 4TB（单节点挂载过多数据会拖慢恢复速度）
- 使用 RAID 0（追求性能）或不做 RAID（ES 自身有副本容错）
- 磁盘占用率建议 ≤ 85%（ES 有水位线机制 `disk.watermark.low=85%`, `disk.watermark.high=90%`）

**CPU**：

- 中等规模集群（10 节点以内）建议 16 核以上
- ES 的线程模型是每个分片一个搜索线程，CPU 核心数直接决定并发处理能力
- 聚合查询（特别是高基数 `terms` 聚合）消耗大量 CPU

**网络**：

- 低延迟网络（节点间传输时间 < 10ms）
- 避免跨机房部署同一集群（网络延迟会导致集群状态同步缓慢、脑裂风险增加）
- 9300 端口（节点间通信）和 9200 端口（HTTP API）须开放

### 8.2 监控指标

**集群级别**：

```bash
# 集群健康状态: green / yellow / red
GET /_cluster/health

# 集群状态概览
GET /_cluster/stats?human=true

# 待处理任务
GET /_cluster/pending_tasks
```

| 状态 | 含义 | 影响 |
|------|------|------|
| **Green** | 所有主分片和副本分片均已分配 | 正常 |
| **Yellow** | 所有主分片已分配，但有副本分片未分配 | 有单点故障风险，但读写正常 |
| **Red** | 有主分片未分配 | 数据不完整，部分搜索和索引操作失败 |

**节点级别**：

```bash
# 节点统计信息
GET /_nodes/stats?human=true
GET /_nodes/stats/jvm,os,fs,indices

# 节点热点线程
GET /_nodes/hot_threads
```

关键指标：
- **JVM Heap Usage**：正常 50%-75%，GC 频繁且占 CPU ≥ 15% 时需要关注
- **OS CPU Usage**：持续 ≥ 80% 需要扩容
- **Disk Free Space**：低于水位线会触发分片迁移
- **Segment Count**：单分片 Segment 数 > 100 说明 Merge 跟不上写入

**索引级别**：

```bash
# 索引状态
GET /_cat/indices?v&s=store.size:desc

# 索引统计信息
GET /my_index/_stats?human=true
```

监控重点：
- **Store Size vs Primary Size**：含副本的存储大小 vs 主分片存储大小
- **Docs Count / Docs Deleted**：已删除文档占比 > 30% 需要 Force Merge
- **Search / Index Rate**：查询速率和写入速率，发现异常峰谷
- **Query / Fetch Time**：查询延迟分布

### 8.3 常见问题排查

**1. 集群状态 Red**

根本原因：有主分片未分配。

```bash
# 查看未分配的分片及原因
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason

# 查看分片分配解释
GET /_cluster/allocation/explain
```

常见原因：
- `NODE_LEFT`：主分片所在节点宕机，且没有完整的 In-Sync 副本可提升为主分片
- `CLUSTER_RECOVERED`：旧分片版本与当前集群状态不一致
- `ALLOCATION_FAILED`：分配尝试失败（磁盘不足、分片数超过限制）
- `INDEX_CREATED`：索引刚创建，尚未完成分片分配

**修复 Red**：

```bash
# 方案1: 如果副本完整，手动提升副本为主分片
POST /_cluster/reroute?retry_failed=true

# 方案2: 如果数据不可恢复且可接受丢失，分配空主分片（慎用！）
POST /_cluster/reroute
{
  "commands": [
    {
      "allocate_empty_primary": {
        "index": "my_index",
        "shard": 0,
        "node": "es-node3",
        "accept_data_loss": true
      }
    }
  ]
}
```

**2. 集群状态 Yellow**

未分配的是副本分片，主分片正常。影响：无写风险，但缺少副本可能导致读性能下降或单点故障。

```bash
# 查看未分配原因
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason

# 常见原因: 节点数少于副本数 (NODE_LEFT / INDEX_CREATED)
# 解决方案: 增加数据节点，或减少副本数
PUT /my_index/_settings { "index.number_of_replicas": 1 }
```

**3. JVM 内存溢出 / GC 超时**

```bash
# 查看 GC 时间
GET /_nodes/stats/jvm?human=true

# 查看占用内存最多的前 N 个索引
GET /_cat/indices?v&s=store.size:desc&h=index,pri,rep,docs.count,store.size
```

常见原因和解决方案：

- **查询聚合深度过大**：减少 `terms` 聚合的 `size`，使用 `execution_hint: map` 限制内存
- **Field Data 过大**：text 字段被聚合/排序时会加载到内存。限制 Field Data 缓存大小：`indices.fielddata.cache.size: 20%`
- **并发 Query 太多**：设置 `thread_pool.search.queue_size` 限制
- **Segment 过多**：每个 Segment 占用一定的堆内存，触发 Force Merge
- **堆内存设得过大**：超过 32GB 导致指针膨胀

**4. 慢查询排查**

```bash
# 设置慢查询阈值并查看日志
PUT /my_index/_settings
{
  "index.search.slowlog.threshold.query.warn": "1s",
  "index.search.slowlog.threshold.query.info": "500ms",
  "index.search.slowlog.threshold.fetch.warn": "500ms"
}

# 使用 Profile API 定位慢查询中的瓶颈
GET /my_index/_search
{
  "profile": true,
  ...
}
```

**5. 节点失联 / 离线**

```bash
# 查看节点列表
GET /_cat/nodes?v

# 查看节点离开的原因
GET /_cluster/health?level=indices
```

常见原因：
- GC 停顿过长导致 Master 认为节点失联（延长 `discovery.zen.fd.ping_timeout`）
- 网络分区（检查防火墙/交换机/网络丢包）
- 磁盘满导致节点退出（检查磁盘水位线）
- OOM Kill（Linux OOM Killer 杀掉 ES 进程）

### 8.4 数据备份与迁移

**Snapshot & Restore**：

ES 官方推荐的备份方案。Snapshot 是增量备份，仅存储自上次备份以来变化的部分。

```bash
# 1. 注册 Snapshot Repository
PUT /_snapshot/my_backup
{
  "type": "fs",
  "settings": {
    "location": "/mount/backups/elasticsearch",
    "compress": true
  }
}

# 2. 创建快照 (支持通配符匹配索引)
PUT /_snapshot/my_backup/snapshot_20260705?wait_for_completion=true
{
  "indices": "logs-*,metrics-*",
  "ignore_unavailable": true,
  "include_global_state": false
}

# 3. 查看快照状态
GET /_snapshot/my_backup/snapshot_20260705/_status

# 4. 恢复 (目标集群或当前集群)
POST /_snapshot/my_backup/snapshot_20260705/_restore
{
  "indices": "logs-2026-07-*",
  "rename_pattern": "logs-(.+)",
  "rename_replacement": "restored-logs-$1"
}
```

**Reindex（同构迁移）**：

当需要修改 Mapping 且无法直接更新时（如修改字段类型），需要创建新索引并 Reindex：

```json
POST /_reindex
{
  "source": { "index": "old_index" },
  "dest": { "index": "new_index" }
}

// 带数据处理和过滤的 Reindex
POST /_reindex
{
  "source": {
    "index": "old_index",
    "query": {
      "range": { "@timestamp": { "gte": "2026-01-01" } }
    }
  },
  "dest": { "index": "new_index" },
  "script": {
    "source": "ctx._source.new_field = ctx._source.old_field.toUpperCase()"
  }
}
```

**跨集群迁移（CCR）**：

ES 6.7+ 支持跨集群复制（Cross Cluster Replication），用于灾备和多数据中心部署。Follower 索引自动从 Leader 索引同步数据。

```json
PUT /_ccr/auto_follow/cluster2_follow
{
  "remote_cluster": "cluster2",
  "leader_index_patterns": ["logs-*"],
  "follow_index_pattern": "{{leader_index}}-copy"
}
```

### 8.5 安全最佳实践

1. **启用 X-Pack Security**：设置用户认证和 RBAC 权限控制
2. **网络隔离**：9200/9300 端口绑定内网 IP，不要暴露公网
3. **HTTPS 加密**：节点间和客户端通信启用 TLS
4. **定期备份**：结合 Snapshot & Restore + ILM 制定自动化备份策略
5. **版本升级**：滚升升级（Rolling Upgrade），逐个节点进行
6. **关闭 dynamic scripting**：禁止动态 Groovy/Painless 脚本注入风险
7. **监控告警**：Watcher 设置集群健康状态为 Yellow/Red 时自动告警

### 8.6 容量规划速查表

| 数据规模 | 节点配置 | 分片建议 | 典型场景 |
|---------|---------|---------|---------|
| < 100GB | 3 节点 (混合), 8C 32G | 1 分片 + 1 副本 | 小型搜索引擎 |
| 100GB - 1TB | 3-5 节点, 16C 64G, SSD | 1-3 分片 + 1 副本 | 中型电商搜索 |
| 1TB - 10TB | 5-10 节点, 32C 128G, SSD | 3-5 分片 + 1 副本 | 日志平台 ELK |
| > 10TB | 10-30+ 节点, 冷热分离 | 按天 Rollover + 冷热分离 | 大规模日志/监控 |

> 数据节点内存：每 TB 数据 ≈ 4-8GB 堆 + OS 留给 Page Cache 的内存。

### 8.7 面试高频题总结

1. **ES 写入为什么是近实时的？** Refresh 间隔（默认 1s）将内存 Buffer 刷为 Segment 后才可搜索。写入的文档先到内存 Buffer，经过 Refresh 才转为可搜索的 Segment。

2. **ES 为什么快？** 倒排索引将搜索复杂度从 O(N) 降到 O(M)（M = 词条数）；FST 快速定位词项；Skip List 加速文档列表合并；Filter Cache 用 RoaringBitmap 缓存过滤结果；分布式并行搜索。

3. **为什么主分片数不能改？** 路由公式 `hash(routing) % num_primary_shards` 依赖主分片数。改分片数后，旧文档按照新分片数取模，路由到的分片不同，导致找不到已有文档。

4. **深度分页怎么解决？** `search_after` 替代 `from+size`，或使用 Scroll（数据导出场景）。`search_after` 基于排序值翻页，无内存开销。

5. **数据丢失场景与防护？** Translog 保证写入持久化（类似 redo log）；`wait_for_active_shards=all` 保证副本同步；Snapshot 定期备份做最后防线。

6. **ES 和 MySQL 怎么协同工作？** 典型的读写分离模式：MySQL 做为主存储（ACID 保证），通过 Canal/Debezium 监听 binlog 变更同步到 ES，ES 提供搜索服务。ES 不承担数据可靠性的最终责任。
