## 简介

1. **列式存储格式。**Druid使用面向列的存储，这意味着它仅需要加载特定查询所需的确切列。这极大地提高了仅命中几列的查询的速度。此外，每列都针对其特定数据类型进行了优化存储，从而支持快速扫描和聚合。
2. **可扩展的分布式系统。**Druid通常部署在数十到数百台服务器的群集中，并且可以提供每秒数百万条记录的接收速率，数万亿条记录的保留以及亚秒级到几秒的查询延迟。
3. **大规模并行处理。**Druid可以在整个集群中并行处理查询。
4. **实时或批量摄取。**德鲁伊可以实时（批量获取被查询的数据）或批量提取数据。
5. **自我修复，自我平衡，易于操作。**作为操作员，要扩展或扩展集群，只需添加或删除服务器，集群就会在后台自动重新平衡自身，而不会造成任何停机。如果任何Druid服务器出现故障，系统将自动绕过损坏，直到可以更换这些服务器为止。Druid设计为24/7全天候运行，而无需出于任何原因而导致计划内停机，包括配置更改和软件更新。
6. **云原生的容错架构，不会丢失数据。**一旦Druid摄取了数据，副本就安全地存储在[深度存储](https://druid.apache.org/docs/latest/design/architecture.html#deep-storage)（通常是云存储，HDFS或共享文件系统）中。即使每台Druid服务器发生故障，也可以从深度存储中恢复数据。对于仅影响少数Druid服务器的有限故障，复制可确保在系统恢复时仍可进行查询。
7. **用于快速过滤的索引。**Druid使用[Roaring](https://roaringbitmap.org/)或 [CONCISE](https://arxiv.org/pdf/1004.0403)压缩的位图索引来创建索引，以支持快速过滤和跨多列搜索。
8. **基于时间的分区。**德鲁伊首先按时间对数据进行分区，然后可以根据其他字段进行分区。这意味着基于时间的查询将仅访问与查询时间范围匹配的分区。这将大大提高基于时间的数据的性能。
9. **近似算法。**德鲁伊包括用于近似计数区别，近似排名以及近似直方图和分位数计算的算法。这些算法提供有限的内存使用量，通常比精确计算要快得多。对于精度比速度更重要的情况，德鲁伊还提供了精确的计数区别和精确的排名。
10. **摄取时自动汇总。**Druid可选地在摄取时支持数据汇总。这种汇总会部分地预先聚合您的数据，并可以节省大量成本并提高性能。

### 应用场景

Druid适合场景：

- 插入率很高，但更新很少。
- 您的大多数查询都是聚合查询和报告查询（“分组依据”查询）。您可能还具有搜索和扫描查询。
- 您将查询等待时间定为100毫秒到几秒钟。
- 您的数据具有时间成分（Druid包括与时间特别相关的优化和设计选择）。
- 您可能有多个表，但是每个查询仅命中一个大的分布式表。查询可能会击中多个较小的“查找”表。
- 您具有高基数数据列（例如URL，用户ID），并且需要对其进行快速计数和排名。
- 您要从Kafka，HDFS，平面文件或Amazon S3之类的对象存储中加载数据。

情况下，您可能会*不*希望使用德鲁伊包括：

- 您需要使用主键对*现有*记录进行低延迟更新。Druid支持流插入，但不支持流更新（使用后台批处理作业完成更新）。
- 您正在构建一个脱机报告系统，其中查询延迟不是很重要。
- 您想执行“大”联接（将一个大事实表连接到另一个大事实表），并且可以花很长时间来完成这些查询。

