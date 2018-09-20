---
title: Time-Series-Index(TSI)设计细节
date: 2018-09-20 12:41:59
tags:
  - 索引
  - TSI
  - 数据库
categories: 数据库
---

> 原文链接： [Time Series Index (TSI) details](https://docs.influxdata.com/influxdb/v1.6/concepts/tsi-details/)

# Time Series Index (TSI) description

当 InfluxDB 插入数据时，不仅需要存储 value 同时也需要为 measurement 和 tag 等做索引来加快查询速度。在之前的版本中，索引数据只存在内存中，这就要求大量的 RAM 并且需要设置机器所能承受的序列容量的上限，根据所用机器的不同，通常这个上限在 100-400 万之间。

时间序列索引（TSI）的设计就是为了超过容量上限。TSI 将索引存储在磁盘上，所以可以不再收到 RAM 的限制。TSI 使用操作系统的页面缓存将热数据提取到内存中，让冷数据驻留在磁盘上。

目前 TSM 和查询引擎中存在着一些限制。制约了一个节点所能存放的时间序列的数量。实际的存储上限通常为 3000 万左右。

# Tooling

## `influx_inspect dumptsi`

如果要解决索引问题，可以使用 `Influx_inspect dumptsi` 命令。此命令允许您打印索引、文件或一组文件的摘要统计信息。此命令一次仅适用于一个索引。
有关此命令的详细信息，请参阅 [Influx_inspect dumptsi](https://docs.influxdata.com/influxdb/v1.6/tools/influx_inspect/#influx-inspect-dumptsi)。

## `influx_inspect buildtsi`

如果要将现有分片从内存中索引转换为 TSI 索引，或者如果现有 TSI 索引已损坏，则可以使用`buildtsi`命令从基础 TSM 数据创建索引。如果您要重建现有 TSI 索引，请首先删除分片中的`index`目录。

此命令在服务器级别工作，但您可以选择添加数据库，保留策略和分片筛选器以仅应用于分片子集。

有关此命令的详细信息，请参阅[Influx inspect buildtsi](https://docs.influxdata.com/influxdb/v1.6/tools/influx_inspect/#influx-inspect-buildtsi)

# 理解 TSI

## 文件组织

TSI（时间序列索引）基于日志合并树(LSMT)，用于 InfluxDB 中存储时间序列。 TSI 由几个部分组成：

- **Index**：包含单个分片(shard)的整个索引数据集。
- **Partition**：包含分片数据的分片分区。
- **LogFile**：包含新写入序列并作为内存索引，并作为 WAL 保留。
- **IndexFile**：包含从单个 LogFile 构建或从两个连续索引文件合并的不可变内存映射索引。

还有一个 **SeriesFile**，它包含整个数据库中所有序列的 keys 的集合。数据库中的每个分片共享相同的 series 文件。

## Writes

写入到来时会发生以下情况：

- 将该 Series 添加到 Series 文件中，如果已存在则查找。返回一个自动递增 Series ID。
- 将该 Series 加到索引中。索引维护现有 Series IDs 的 [roaring bitmap](https://cloud.tencent.com/developer/article/1136054)，并忽略已创建的 Series。
- 该 Series 经过哈希处理后添加进相应的分区。
- 分区将该 Series 写为 LogFile 上的一条记录。
- LogFile 将该 Series 写到磁盘上的预写日志文件（WAL），并将该 Series 添加到内存索引中。

## Compaction

一旦 LogFile 超过阈值（5MB），就会创建一个新的活动日志文件，并且前一个文件开始压缩到 IndexFile 中。第一个索引文件位于 1 级（L1）。日志文件被视为级别 0（L0）。

也可以通过将两个较小的索引文件合并在一起来创建索引文件。例如，如果存在连续的两个 L1 索引文件，则它们可以合并到 L2 索引文件中。

## Reads

该索引提供了几个用于检索数据集的 API 调用，例如：

- MeasurementIterator()：返回 measurement names 的排序列表。
- TagKeyIterator()：返回 measurement 中 tag keys 的排序列表。
- TagValueIterator()：返回一个 tag key 的 tag values 的排序列表。
- MeasurementSeriesIDIterator()：返回 measurement 的所有 series ID 的排序列表。
- TagKeySeriesIDIterator()：返回 一个 tag key 的所有 series ID 的排序列表。
- TagValueSeriesIDIterator()：返回 一个 tag value 所有 series ID 的排序列表。

这些迭代器都可以使用多个合并迭代器进行组合。对于每种类型的迭代器（measurement, tag key, tag value, series id），有多个合并迭代器类型：

- 合并：对两个迭代器中的项进行重复数据删除。
- 相交：仅返回两个迭代器中同时存在的项。
- 不同：仅返回在第一个迭代器中存在，在第二个迭代器中不存在的项。

例如，具有区域！='us-west'的 WHERE 子句的查询将跨两个分片操作，将构造一组迭代器，如下所示：

例如，诸如跨两个分片的 WHERE 子句的查询 `region != 'us-west'`,该构造一组类似如下的迭代器:

```go
DifferenceSeriesIDIterators(
    MergeSeriesIDIterators(
        Shard1.MeasurementSeriesIDIterator("m"),
        Shard2.MeasurementSeriesIDIterator("m"),
    ),
    MergeSeriesIDIterators(
        Shard1.TagValueSeriesIDIterator("m", "region", "us-west"),
        Shard2.TagValueSeriesIDIterator("m", "region", "us-west"),
    ),
)
```

## Log File Structure

日志文件简单地构造为按顺序写入磁盘的 LogEntry 对象列表。一旦达到 5MB 则写入日志文件，然后将它们压缩为索引文件。日志中的条目对象可以是以下任何类型：

- AddSeries
- DeleteSeries
- DeleteMeasurement
- DeleteTagKey
- DeleteTagValue

日志文件中的内存索引内容如下

- name 对应多个 measurement
- measurement 对应多个 tag key
- tag key 对应多个 tag value
- measurement 对应多个 Series
- tag value 对应多个 Series
- 用于 series, measurements, tag keys, and tag values 的删除标记(Tombstones)

日志文件还维护用于表示 Series ID 存在和删除的 bitsets。这些 bitsets 与其他日志文件和索引文件合并，以在启动时重新生成完整的 index bitset。

## Index File Structure

索引文件是一个不可变的文件，它跟踪与日志文件类似的信息，但所有数据都被索引并写入磁盘，以便可以从内存映射中直接访问它。

索引文件包含以下部分：

- **TagBlocks**：维护一个索引，用于存放一个 tag key 的多个 tag values。
- **MeasurementBlock**：维护一个索引用于 measurements 和他们的 tag keys。
- **Trailer**：存储文件的偏移信息以及用于基数估计的 HyperLogLog skecthes。

## Manifest

**MANIFEST**文件存储在索引目录中，列出了属于索引的所有文件以及它们应被访问的顺序。每次发生压缩时都会更新此文件。任何没有在文件夹下面的索引文件都是正在压缩的索引文件

## FileSet

文件集是在 InfluxDB 进程运行时获取的清单的内存快照。这是提供某个时间点一致性索引视图所必需的。该文件集还便于对其所有文件进行引用计数，这样在文件的所有 readers 完成之前，不会通过压缩删除任何文件。
