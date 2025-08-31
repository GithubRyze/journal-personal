---
title: elasticsearch 学习
author: 刘Sir
date: 2025-08-28 10:10:00 +0800
categories: [技术]
tags: [redis]
render_with_liquid: false
---
## 客户端写 es 的过程
https://www.elastic.co/docs/deploy-manage/distributed-architecture/reading-and-writing-documents

对于学习 es，那么除了理解它能做什么之外，最终重要的是需要清楚一条数据经过哪些过程写入 es，这样不仅可以对 es 有更深入的了解，并且可以学习 es 是如何处理数据的。
1. 路由：经过协调节点（coordinating node），计算分片（默认文档 id % 分片数量），然后转发到分片的节点
2. 写主分片：主分片进行分词，建立倒排表，列存储（doc values），原数据等，写入 translog 和 内存 buffer （不会写入磁盘）
3. 同步数据到副本：主分片会把信息同步到副本分片，副本写入成功后 响应 ack
4. 刷新（refresh）：默认 1 秒 刷新一次，内存 buffer 中的数据会生成 segment 并写入文件系统缓存，新的文档才会对搜索可见
5. 持久化（Flush）： translog 周期性落盘（默认 30分，512mb），segment 文件写到磁盘，保证数据不会丢失。

## 客户端查询 es 过程
https://www.elastic.co/docs/deploy-manage/distributed-architecture/reading-and-writing-documents

1. 查询路由：协调节点（coordinating node）定位分片的节点位置，把查询请求广播到相关的节点
2. 分片本地查询（每个分片是一个 lence 实例）：检索倒排索引，找到文档 id, 计算相关性分值，生成本地前 N 条结果 （此阶段不会返回具体的文档数据，只是文档元数据，位置，相关分值）
3. 合并结果：协调节点（coordinating node）收到其他节点分片执行后的结果进行汇总，取值。
4. 抓取文档：协调节点（coordinating node）再根据文档元数据，位置信息再向分片节获取具体的文档信息（类似回表），最终返回

## Elasticsearch 是什么？和传统数据库有什么区别？
es 是定位为一个文档型的数据库，以 json 的数据格式存储数据，适合大量文本检索的场景（文本搜索、聚合分析、日志查询）。对比传统的数据有以下不同
- http/https 处理请求
- 倒排索引（Lucene），能有效应对大量文档检索（（文本搜索、聚合分析、日志查询））
- 文档存储（json），非列式存储
- 支持分布式部署，通过数据分片 + 副本 方式高可用

## Index、Type、Document、Shard、Replica 的关系。
index： 数据存储索引名称（可以理解是一个表）
type： 数据中的字段类型（text，int，keyword，long 等）
document：具体的数据内容（可以理解是一行数据）
shard （primary）：主分片，es 会把 index 的数据通过内置分片策略，分配到不同的实例上
Replica：分片的副本数量，es 会复制分片做到高可用，分担查询压力

## Elasticsearch 的倒排索引是什么，如何工作？
- 正排索引：存储每个文档包含哪些词
- 倒排索引：存储词在哪些文档中
工作机制：
1. 数据写入的时候，会对写入的数据进行分词（Tokenizer），会把文本分成一个个单独的词（token）
2. 标准化，把写入的词按照 es 标准进行格式化 小写化、去掉停用词（stop words）、做词干处理（stemming）等
3. 建立倒排表：为每个词建立文档列表，记录文档 ID 和位置等信息
4. 查询：检索时候查询倒排表，定位包含词的文档，然后根据条件进行过滤（filter，bool，评分）等返回结果
通过以上过程即可不扫描文档，通过关键字结合查询条件快速查找数据（模糊查询、前缀查询、短语查询）

## Elasticsearch 的分片（Shard）和副本（Replica）是如何工作的？为什么需要它们？
es shard（primary）主分片会把数据通过内置的分片策略分到不同的 es 实例上（差不多说分表的概念），在通过复制分片的副本（Replica）到不同的 es 实例上
- shard（primary）：接收写请求，拆成多份分片，分布在不同节点上，解决单机存储和性能瓶颈
- Replica：是主分片的完全拷贝，存储在不同节点上（只读）。节点宕机时，副本可以顶替主分片继续工作，查询请求可以发送到主分片或副本分片，提高并发查询能力。

写多读少：可以配置多些主分片，减少副分片，提升写的效率

读多：配置多些副分片，减少主分片，提升查询的效率

## Elasticsearch 的集群架构和节点角色有哪些？
https://www.elastic.co/docs/deploy-manage/distributed-architecture/clusters-nodes-shards/node-roles

一个 Elasticsearch 集群是由一个或多个节点组成的集合，每个集群有一个唯一的名称，集群内部节点协作来存储数据和处理查询。
- Master 节点（主节点）：管理集群元数据（metadata），如索引信息、分片分配、集群状态
- data：存储数据分片（primary & replica），执行索引和查询操作
- ingest：处理预处理管道（ingest pipeline），如数据清洗、转换等操作
- Coordinating 节点（协调节点）：收客户端请求，分发请求到各个数据节点，汇总结果返回客户端。
- remote-eligible： 跨集群沟通节点
- machine leaning：执行机器学习任务，如异常检测、趋势分析等
- transform：数据预处理节点
一个 es 集群至少配置 3 个节点，因为 master 选举策略是 zen discovery（大于半数）

## Elasticsearch 中的索引分词器（Analyzer）是什么？它的作用有哪些？
Analyzer 主要是把目标文本进行分词，拆分成不同的词（token）
- 索引阶段：拆分文本，建立倒排表
- 检索阶段：拆分查询的关键字，确保匹配
- 提高检索效率：通过词项化、去停用词、大小写规范化等优化搜索匹配
 
过程：字符过滤器（char filter，文本预处理）-》分词器（Tokenizer）：将文本拆分成一个个词项 -》词项过滤器（Token Filter）：对拆分后的词项进行处理（如小写化、同义词替换、去停用词）。

常见的分词器：
- ik_max_word：最大化拆分（多粒度分词），适合搜索。
- ik_smart：智能分词，粒度较粗，适合分析和统计。
- standard：标准分词器，分词规则基于 Unicode 文本分割。
- simple：按非字母字符拆分，并转小写。
- keyword：不拆分整个字段，通常用于精确匹配。
- whitespace：按空格拆分，不做其他处理。

不同的字段可以配置不同的分词器，索引分词器和查询分词器
```
PUT my_index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "fields": {
          "ik": { "type": "text", "analyzer": "ik_max_word" },
          "keyword": { "type": "keyword" }
        }
      }
    }
  }
}

```

## Elasticsearch 支持哪些类型的查询？match、term、range，bool query、must、should、filter、must_not 有什么区别？
- term：精确匹配，不分词。适合keyword、数值、日期 等字段，如查询状态
- match：字段匹配，会分词，再根据倒排索引查找。适合 text 类型字段（全文检索）。
- mutil_match: 多字段匹配
- bool: 复合查询，搭配 must，must not，should
- fliter: 不参与算分，结果可缓存，加速查询。
- range: 范围查询，适合数值、日期
- must: 必须匹配 and，会影响相关性评分
- should:  部分匹配，影响相关性评分。可以配置 should 的数量
- must not: 必须不匹配 not

## Elasticsearch 写入和查询性能如何优化？
https://www.elastic.co/docs/deploy-manage/production-guidance/optimize-performance

写优化
1. 
2. 配置合理索引分片数量，读多（主分片可少，副分片可以多），写多（主分片多，写分片少）
3. 配置合理的索引 ilm 策略，hot，warm，cold，delete 等策略，热数据放性能好的节点，冷数据放低成本节点
4. 批量写入 bulk insert
5. 配置合理的 refresh 时间
6. 避免过多的索引和小分片
读优化
1. 尽可能使用 filter 进行过滤查询，可缓存结果，并且不影响评分
2. 避免深度分页：from + size 在大 offset 时非常慢，可使用 search_after 和 scroll api
3. 数据字段设计合理，无需分词的字段设置 keyword，数值使用 int，long，date 等类型
4. 聚合查询启用 request cache（"index.requests.cache.enable": true）
5. 查询时指定 routing，避免广播到所有分片（_id 哈希决定文档落在哪个分片）

集群优化：
- 配置正节点角色，比如 master， data，Coordinating 等节点
- 合理的节点硬件配置：ES 足够的内存（建议 heap 占物理内存的 50%，最大不超过 32GB），使用 ssd 高速磁盘
- 监控 heap usage、GC、query latency，实时调整分片、副本和 refresh 策略

## 如果文档量非常大（上亿级别），你如何设计索引结构？
其主要问题在以下几个方面：
- 写入压力大（索引构建、倒排索引膨胀）。
- 存储成本高（磁盘占用，冷热数据）。
- 查询延迟（全量扫描 or 分片过多）。
- 集群稳定性（分片均衡、rebalance 代价）。

1. 拆分索引，避免单一索引膨胀。ilm 配置索引按时间，大小进行拆分。再配合 hot，warm，cold，delete 等策略
2. 分片设计：不要设置太多分片 （1 亿数据可能 5–10 个分片就够，不要上百个），主分片的数量影响写，副分片过多增加磁盘占用，并且分片过多查询 routing 需要回表计算
3. 映射 & 字段优化：关闭不必要的_source、_all、_id 字段存储，节省磁盘，过滤字段使用 keyword。避免使用 nested 结果，配置 mapping 控制字段数量
4. 冷热数据分层，热数据放 ssd，温数据放普通磁盘，冷数据用 snapshot 存到对象存储
5. 写入时用 bulk API、调高 refresh interval，查询时用 filter 和 routing 优化范围

引用：每个分片本质上是一个 Lucene 实例，需要独立的文件句柄、内存索引缓存、线程池 → 分片过多会挤占 JVM heap，导致 GC 压力大
## 如何处理 Elasticsearch 的热点分片问题？
热点分片问题：热点分片问题是指数据写入过度集中在少数分片，导致该分片成为性能瓶颈。
解决：
1. 调整 routing 策略，增加 hash 打散数据
2. 使用索引模板 + （ILM）
3. hot，warm，cold 数据分层存储
```
# 热节点（负责高并发写入和实时查询，通常用 SSD）
node.roles: [data_hot]

# 温节点（写入少，读多，存放次新数据，通常用 SATA/SSD 混合）
node.roles: [data_warm]

# 冷节点（很少访问，只做存档查询，通常用大容量 HDD）
node.roles: [data_cold]

# 冻结节点（Frozen，低成本，利用共享存储，几乎不查询）
node.roles: [data_frozen]

```
4. 批量写入
## 如何实现日志或指标数据的滚动索引（Index Rollover）？
索引模版 + ilm
## 如何解决 ES 内存溢出（OutOfMemoryError）问题？
1. 热，温，冷数据分层存储： 配置节点角色，索引模版，ilm 策略
2. 拆分索引，避免单个索引过大：索引模版，ilm 策略
3. 控制 fielddata / doc_values（聚合查询需要加载所有的 doc values到内存）
    -  text 字段默认不能用于聚合/排序，否则会触发 fielddata 加载到 heap，容易 OOM。
4. 查询优化
    - 避免 match_all + 大分页（from+size 很大）
    - 使用 scroll / search_after 来替代深分页
    - 多用 filter（可缓存）替代 must（需要算分）
5. 避免 Mapping 设计问题：动态 mapping 可能生成大量 field（field explosion
6. 集群层面：设置分片数（太多分片会占用大量 heap）