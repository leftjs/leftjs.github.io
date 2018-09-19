---
title: InfluxDB 存储方案
date: 2018-09-19 16:18:43
tags:
  - 分布式系统
  - 存储引擎
  - 数据库
  - InfluxDB
categories: 数据库
---

# InfluxDB

InfluxDB 在 DB-Engines 的时序数据库类别里排名第一，实至名归，从它的功能丰富性、易用性以及底层实现来看，都有很多的亮点，值得大篇幅来分析。

首先简单归纳下它的几个比较重要的特性：

1.  **极简架构**：单机版的 InfluxDB 只需要安装一个 binary，即可运行使用，完全没有任何的外部依赖。相比来看几个反面例子，OpenTSDB 底层是 HBase，拖家带口就得带上 ZooKeeper、HDFS 等，如果你不熟悉 Hadoop 技术栈，一般运维起来是有一定的难度，这也是其被人抱怨最多的一个点。KairosDB 稍微好点，它依赖 Cassandra 和 ZooKeeper，单机测试可以使用 H2。总的来说，依赖一个外部的分布式数据库的 TSDB，在架构上会比完全自包含的 TSDB 复杂一点，毕竟一个成熟的分布式数据库本身就很复杂，当然这一点在云计算这个时代已经完全消除。
2.  **TSM Engine**：底层采用自研的 TSM 存储引擎，TSM 也是基于 LSM 的思想，提供极强的写能力以及高压缩率，在后面的章节会对其做一个比较详细的分析。
3.  **InfluxQL**：提供 SQL-Like 的查询语言，极大的方便了使用，数据库在易用性上演进的终极目标都是提供 Query Language。
4.  **Continuous Queries**: 通过 CQ 能够支持 auto-rollup 和 pre-aggregation，对常见的查询操作可以通过 CQ 来预计算加速查询。
5.  **TimeSeries Index**: 对 Tags 会进行索引，提供高效的检索。这一项功能，对比 OpenTSDB 和 KairosDB 等，在 Tags 检索的效率上提升了不少。OpenTSDB 在 Tags 检索上做了不少的查询优化，但是受限于 HBase 的功能和数据模型，所以然并卵。不过目前稳定版中的实现采用的是 memory-based index 的实现方式，这种方案在实现上比较简单，查询上效率最高，但是带来了不少的问题，在下面的章节会详细描述。
6.  **Plugin Support**: 支持自定义插件，能够扩展到兼容多种协议，如 Graphite、collectd 和 OpenTSDB。

# TSM

InfluxDB 底层的存储引擎经历了从 LevelDB 到 BlotDB，再到选择自研 TSM 的过程，整个选择转变的思考可以在其官网文档里看到。整个思考过程很值得借鉴，对技术选型和转变的思考总是比平白的描述某个产品特性让人印象深刻的多。

我简单总结下它的整个存储引擎选型转变的过程，第一阶段是 LevelDB，选型 LevelDB 的主要原因是其底层数据结构采用 LSM，对写入很友好，能够提供很高的写入吞吐量，比较符合时序数据的特性。在 LevelDB 内，数据是采用 KeyValue 的方式存储且按 Key 排序，InfluxDB 使用的 Key 设计是 SeriesKey+Timestamp 的组合，所以相同 SeriesKey 的数据是按 timestamp 来排序存储的，能够提供很高效的按时间范围的扫描。

不过使用 LevelDB 的一个最大的问题是，InfluxDB 支持历史数据自动删除（Retention Policy），在时序数据场景下数据自动删除通常是大块的连续时间段的历史数据删除。LevelDB 不支持 Range delete 也不支持 TTL，所以要删除只能是一个一个 key 的删除，会造成大量的删除流量压力，且在 LSM 这种数据结构下，真正的物理删除不是即时的，在 compaction 时才会生效。各类 TSDB 实现数据删除的做法大致分为两类：

1.  **数据分区**：按不同的时间范围划分为不同的分区（Shard），因为时序数据写入都是按时间线性产生的，所以分区的产生也是按时间线性增长的，写入通常是在最新的分区，而不会散列到多个分区。分区的优点是数据回收的物理删除非常简单，直接把整个分区删除即可。缺点是数据回收的精细度比较大，为整个分区，而回收的时间精度取决于分区的时间跨度。分区的实现可以是在应用层提供，也可以是存储引擎层提供，例如可以利用 RocksDB 的 column family 来作为数据分区。InfluxDB 采用这种模式，默认的 Retention Policy 下数据会以 7 天时间跨度组成为一个分区。
2.  **TTL**：底层数据引擎直接提供数据自动过期的功能，可以为每条数据设定存储时间（time to live），当数据存活时间到达后存储引擎会自动对数据进行物理删除。这种方式的优点是数据回收的精细度很高，精细到秒级及行级的数据回收。缺点是 LSM 的实现上，物理删除发生在 compaction 的时候，比较不及时。RocksDB、HBase、Cassandra 和阿里云表格存储都提供数据 TTL 的功能。

InfluxDB 采用的是第一种策略，会按 7 天一个周期，将数据分为多个不同的 Shard，每个 Shard 都是一个独立的数据库实例。随着运行时间的增长，shard 的个数会越来越多。而由于每个 shard 都是一个独立的数据库实例，底层都是一套独立的 LevelDB 存储引擎，这时带来的问题是，每个存储引擎都会打开比较多的文件，随着 shard 的增多，最终进程打开的文件句柄会很快触及到上限。LevelDB 底层采用 level compaction 策略，是文件数多的原因之一。实际上 level compaction 策略不适合时序数据这种写入模式，这点原因 InfluxDB 没有提及。

由于遇到大量的客户反馈文件句柄过多的问题，InfluxDB 在新版本的存储引擎选型中选择了 BoltDB 替换 LevelDB。BoltDB 底层数据结构是 mmap B+树，其给出的选型理由是：1.与 LevelDB 相同语义的 API；2.纯 Go 实现，便于集成和跨平台；3.单个数据库只使用一个文件，解决了文件句柄消耗过多的问题，这条是他们选型 BoltDB 的最主要理由。但是 BoltDB 的 B+树结构与 LSM 相比，在写入能力上是一个弱势，B+树会产生大量的随机写。所以 InfluxDB 在使用 BoltDB 之后，很快遇到了 IOPS 的问题，当数据库大小达到几个 GB 后，会经常遇到 IOPS 的瓶颈，极大影响写入能力。虽然 InfluxDB 后续也采用了一些写入优化措施，例如在 BoltDB 之前加了一层 WAL，数据写入先写 WAL，WAL 能保证数据是顺序写盘，但是最终写入 BoltDB 还是会带来比较大的 IOPS 资源消耗。

InfluxDB 在经历了几个小版本的 BoltDB 后，最终决定自研 TSM，TSM 的设计目标一是解决 LevelDB 的文件句柄过多问题，二是解决 BoltDB 的写入性能问题。TSM 全称是 Time-Structured Merge Tree，思想类似 LSM，不过是基于时序数据的特性做了一些特殊的优化。来看下 TSM 的一些重要组件：

1.  **Write Ahead Log(WAL)** : 数据会先写入 WAL，后进入 memory-index 和 cache，写入 WAL 会同步刷盘，保证数据持久化。Cache 内数据会异步刷入 TSM File，在 Cache 内数据未持久化到 TSM File 之前若遇到进程 crash，则会通过 WAL 内的数据来恢复 cache 内的数据，这个行为与 LSM 是完全类似的。
2.  **Cache**: TSM 的 Cache 与 LSM 的 MemoryTable 类似，其内部的数据为 WAL 中未持久化到 TSM File 的数据。若进程发生 failover，则 cache 中的数据会根据 WAL 中的数据进行重建。Cache 内数据保存在一个 SortedMap 中，Map 的 Key 为 TimeSeries+Timestamp 的组成。所以可以看到，在内存中数据是按 TimeSeries 组织的，TimeSeries 中的数据按时间顺序存放。
3.  **TSM Files**: TSM File 与 LSM 的 SSTable 类似，TSM File 由四个部分组成，分别为：header, blocks, index 和 footer。其中最重要的部分是 blocks 和 index：
    - **Block**：每个 block 内存储的是某个 TimeSeries 的一段时间范围内的值，即某个时间段下某个 measurement 的某组 tag set 对应的某个 field 的所有值，Block 内部会根据 field 的不同的值的类型采取不同的压缩策略，以达到最优的压缩效率。
    - **Index**：文件内的索引信息保存了每个 TimeSeries 下所有的数据 Block 的位置信息，索引数据按 TimeSeries 的 Key 的字典序排序。在内存中不会把完整的 index 数据加载进去，这样会很大，而是只对部分 Key 做索引，称之为 indirectIndex。indirectIndex 中会有一些辅助定位的信息，例如该文件中的最小最大时间以及最小最大 Key 等，最重要的是保存了部分 Key 以及其 Index 数据的文件 offset 信息。若想要定位某个 TimeSeries 的 Index 数据，会先根据内存中的部分 Key 信息找到与其最相近的 Index Offset，之后从该起点开始顺序扫描文件内容再精确定位到该 Key 的 Index 数据位置。
4.  **Compaction**: compaction 是一个将 write-optimized 的数据存储格式优化为 read-optimized 的数据存储格式的一个过程，是 LSM 结构存储引擎做存储和查询优化很重要的一个功能，compaction 的策略和算法的优劣决定了存储引擎的质量。在时序数据的场景下，基本很少发生 update 或者 delete，数据都是按时间顺序生成的，所以基本不会有 overlap，Compaction 起到的作用主要在于压缩和索引优化。

    - **LevelCompaction**: InfluxDB 将 TSM 文件分为 4 个层级(Level 1-4)，compaction 只会发生在同层级文件内，同层级的文件 compaction 后会晋升到下一层级。从这个规则看，根据时序数据的产生特性，level 越高数据生成时间越旧，访问热度越低。由 Cache 数据初次生成的 TSM 文件称为 Snapshot，多个 Snapshot 文件 compaction 后产生 Level1 的 TSM 文件，Level1 的文件 compaction 后生成 level2 的文件，依次类推。低 Level 和高 Level 的 compaction 会采用不同的算法，低 level 文件的 compaction 采用低 CPU 消耗的做法，例如不会做解压缩和 block 合并，而高 level 文件的 compaction 则会做 block 解压缩以及 block 合并，以进一步提高压缩率。我理解这种设计是一种权衡，compaction 通常在后台工作，为了不影响实时的数据写入，对 compaction 消耗的资源是有严格的控制，资源受限的情况下必然会影响 compaction 的速度。而 level 越低的数据越新，热度也越高，需要有一种更快的加速查询的 compaction，所以 InfluxDB 在低 level 采用低资源消耗的 compaction 策略，这完全是贴合时序数据的写入和查询特性来设计的。
    - **IndexOptimizationCompaction**: 当 Level4 的文件积攒到一定个数后，index 会变得很大，查询效率会变的比较低。影响查询效率低的因素主要在于同一个 TimeSeries 数据会被多个 TSM 文件所包含，所以查询不可避免的需要跨多个文件进行数据整合。所以 IndexOptimizationCompaction 的主要作用就是将同一 TimeSeries 下的数据合并到同一个 TSM 文件中，尽量减少不同 TSM 文件间的 TimeSeries 重合度。
    - **FullCompaction**: InfluxDB 在判断某个 Shard 长时间内不会再有数据写入之后，会对数据做一次 FullCompaction。FullCompaction 是 LevelCompaction 和 IndexOptimization 的整合，在做完一次 FullCompaction 之后，这个 Shard 不会再做任何的 compaction，除非有新的数据写入或者删除发生。这个策略是对冷数据的一个规整，主要目的在于提高压缩率。

# TimeSeries Index

时序数据库除了支撑时序数据的存储和计算外，还需要能够提供多维度查询。InfluxDB 为了提供更快速的多维查询，对 TimeSeries 进行了索引。关于数据和索引，InfluxDB 是这么描述自己的：

> InfluxDB actually looks like two databases in one: a time series data store and an inverted index for the measurement, tag, and field metadata.

在 InfluxDB 1.3 之前，TimeSeries Index(下面简称为 TSI)只支持 Memory-based 的方式，即所有的 TimeSeries 的索引都是放在内存内，这种方式有好处但是也会带来很多的问题。而在最新发布的 InfluxDB 1.3 版本上，提供了另外一种方式的索引可供选择，新的索引方式会把索引存储在磁盘上，效率上相比内存索引差一点，但是解决了内存索引存在的不少问题。

## Memory-based Index

```go
// Measurement represents a collection of time series in a database. It also
    // contains in memory structures for indexing tags. Exported functions are
    // goroutine safe while un-exported functions assume the caller will use the
    // appropriate locks.
    type Measurement struct {
     database string
     Name     string `json:"name,omitempty"`
     name     []byte // cached version as []byte

     mu         sync.RWMutex
     fieldNames map[string]struct{}

     // in-memory index fields
     seriesByID          map[uint64]*Series              // lookup table for series by their id
     seriesByTagKeyValue map[string]map[string]SeriesIDs // map from tag key to value to sorted set of series ids

     // lazyily created sorted series IDs
     sortedSeriesIDs SeriesIDs // sorted list of series IDs in this measurement
    }

    // Series belong to a Measurement and represent unique time series in a database.
    type Series struct {
     mu          sync.RWMutex
     Key         string
     tags        models.Tags
     ID          uint64
     measurement *Measurement
     shardIDs    map[uint64]struct{} // shards that have this series defined
    }
```

如上是 InfluxDB 1.3 的源码中对内存索引数据结构的定义，主要有两个重要的数据结构体：

**Series**: 对应某个 TimeSeries，其内存储 TimeSeries 相关的一些基本属性以及它所属的 Shard

- Key：对应 measurement + tags 序列化后的字符串。
- tags: 该 TimeSeries 下所有的 TagKey 和 TagValue
- ID: 用于唯一区分的整数 ID。
- measurement: 所属的 measurement。
- shardIDs: 所有包含该 Series 的 ShardID 列表。

**Measurement**: 每个 measurement 在内存中都会对应一个 Measurement 结构，其内部主要是一些索引来加速查询。

- seriesByID：通过 SeriesID 查询 Series 的一个 Map。
- seriesByTagKeyValue：双层 Map，第一层是 TagKey 对应其所有的 TagValue，第二层是 TagValue 对应的所有 Series 的 ID。可以看到，当 TimeSeries 的基数变得很大，这个 map 所占的内存会相当多。
- sortedSeriesIDs：一个排序的 SeriesID 列表。

全内存索引结构带来的好处是能够提供非常高效的多维查询，但是相应的也会存在一些问题：

- 能够支持的 TimeSeries 基数有限，主要受限于内存的大小。若 TimeSeries 个数超过上限，则整个数据库会处于不可服务的状态。这类问题一般由用户错误的设计 TagKey 引发，例如某个 TagKey 是一个随机的 ID。一旦遇到这个问题的话，也很难恢复，往往只能通过手动删数据。
- 若进程重启，恢复数据的时间会比较长，因为需要从所有的 TSM 文件中加载全量的 TimeSeries 信息来在内存中构建索引。

## Disk-based Index

针对全内存索引存在的这些问题，InfluxDB 在最新的 1.3 版本中提供了另外一种索引的实现。得益于代码设计上良好的扩展性，索引模块和存储引擎模块都是插件化的，用户可以在配置中自由选择使用哪种索引。

![disk-based index structure](/images/2018/09/disk-based-index-structure.png)

InfluxDB 实现了一个特殊的存储引擎来做索引数据的存储，其结构也与 LSM 类似，如上图就是一个 Disk-based Index 的结构图，详细的说明可以参见设计文档。
索引数据会先写入 Write-Ahead-Log，WAL 中的数据按 LogEntry 组织，每个 LogEntry 对应一个 TimeSeries，包含 Measurement、Tags 以及 checksum 信息。写入 WAL 成功后，数据会进入一个内存索引结构内。当 WAL 积攒到一定大小后，LogFile 会 Flush 成 IndexFile。IndexFile 的逻辑结构与内存索引的结构一致，表示的也是 Measurement 到 TagKey，TagKey 到 TagValue，TagValue 到 TimeSeries 的 Map 结构。InfluxDB 会使用 mmap 来访问文件，同时文件中对每个 Map 都会保存 HashIndex 来加速查询。

当 IndexFile 积攒到一定数量后，InfluxDB 也提供 compaction 的机制，将多个 IndexFile 合并为一个，节省存储空间以及加速查询。
