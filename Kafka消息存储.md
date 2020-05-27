## 消息的文件存储机制

前面我们知道了一个 topic 的多个 partition 在物理磁盘上的保存路径，那么我们再来分析日志的存储方式。通过如下命令找到对应 partition 下的日志内容

```shell
[root@localhost ~]# ls /tmp/kafka-logs/firstTopic-1/
00000000000000000000.index 
00000000000000000000.log
00000000000000000000.timeindex leader-epochcheckpoint
```


kafka 是通过分段的方式将 Log 分为多个 LogSegment，LogSegment 是一个逻辑上的概念，一个 LogSegment 对应磁盘上的一个日志文件和一个索引文件，其中日志文件是用来记录消息的。索引文件是用来保存消息的索引。那么这个 LogSegment 是什么呢？

### LogSegment

假设 kafka 以 partition 为最小存储单位，那么我们可以想象当 kafka producer 不断发送消息，必然会引起 partition文件的无线扩张，这样对于消息文件的维护以及被消费的消息的清理带来非常大的挑战，所以 kafka 以 segment 为单位又把 partition 进行细分。**每个 partition 相当于一个巨型文件被平均分配到多个大小相等的segment数据文件中（每个 segment 文件中的消息不一定相等），这种特性方便已经被消费的消息的清理，提高磁盘的利用率。**

log.segment.bytes=107370 (设置分段大小),默认是1gb，我们把这个值调小以后，可以看到日志分段的效果

抽取其中 3 个分段来进行分析

![img](E:\研究生学习\Work\技术笔记\Kafka消息存储.assets\20190814182605945.png)

segment file 由 2 大部分组成，分别为 index file 和 datafile，此 2 个文件一一对应，成对出现，后缀**".index"和“.log”分别表示为 segment 索引文件、数据文件。**



### segment文件的命名规则（与offset的关系）

> segment 文件命名规则：partion 全局的第一个 segment从 0 开始，后续每个 segment 文件名为上一个 segment文件最后一条消息的 offset 值进行递增。数值最大为 64 位long 大小，20 位数字字符长度，没有数字用 0 填充

查看 segment 文件命名规则，通过下面这条命令可以看到 kafka 消息日志的内容 

```shell
sh kafka-run-class.sh kafka.tools.DumpLogSegments --files /tmp/kafka-logs/test0/00000000000000000000.log --print-data-log
```


输出结果为:

```shell
offset: 5376 position: 102124 CreateTime: 1531477349287
isvalid: true keysize: -1 valuesize: 12 magic: 2
compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: []
payload: message_5376
```



第一个 log 文件的最后一个 offset 为:5376,所以下一个segment 的文件命名为: 00000000000000005376.log。对 应的 index 为 00000000000000005376.index。

### segment 中 index 和 log 的对应关系

从所有分段中，找一个分段进行分析为了提高查找消息的性能，为每一个日志文件添加 2 个索引索引文件：OffsetIndex 和 TimeIndex，分别对应.index以及.timeindex,TimeIndex 索引文件格式：它是映射时间 戳和相对 offset。

查 看 索 引 内 容 ：

```shell
sh kafka-run-class.sh
kafka.tools.DumpLogSegments --files /tmp/kafkalogs/test-0/00000000000000000000.index
--print-datalog
```

![img](E:\研究生学习\Work\技术笔记\Kafka消息存储.assets\20190814182808437.png)

如图所示，index 中存储了索引以及物理偏移量。 log 存储了消息的内容。

索引文件的元数据执行对应数据文件中 message 的物理偏移地址。举个简单的案例来说，以[4053,80899]为例，在 log 文件中，对应的是第 4053 条记 录，物理偏移量（position）为 80899. position 是ByteBuffer 的指针位置

### 在 partition 中如何通过 offset 查找 message

当要访问偏移量为899的这条消息时，先去index文件中查找，找到了700和1000这个区间，根据700这个索引中的信息，找到log文件中700这条消息的具体位置，然后顺序往下查找，直到找到索引为899的这条消息为止。从这个实现中我们可以看到，Kafka并没有进行log文件的整个遍历，而是通过index中的稀疏索引，找到消息在log中的大概位置，然后顺序遍历找到消息，这样就大大提高了查找的效率，如图：

![img](E:\研究生学习\Work\技术笔记\Kafka消息存储.assets\1236645-20190629231456638-940164239.png)

#### 查找的算法是

1. **根据 offset 的值，查找 segment 段中的指定的 index 索引文件**。由于索引文件命名是以上一个文件的最后一个 offset 进行**命名的**，所以，使用**二分查找算法**能够根据offset 快速定位到指定的索引文件。

2. **找到索引文件后，根据 offset 进行定位，找到索引文件中的符合范围的索引**。（kafka 采用稀疏索引的方式来提高查找性能）

3. 得到 position 以后，再到对应的 log 文件中，从 position出开始查找 offset 对应的消息，**将每条消息的 offset 与目标 offset 进行比较，直到找到消息**。

   比如说，我们要查找 offset=2490 这条消息，那么先找到 00000000000000000000.index, 然后找到[2487,49111]这个索引，再到 log 文件中，根据 49111 这个 position 开始查找，比较每条消息的 offset 是否大于等于 2490。最后查找到对应的消息以后返回

 

### Log 文件的消息内容分析

前面我们通过 kafka 提供的命令，可以查看二进制的日志文件信息，一条消息，会包含很多的字段。

```shell
offset: 5371 position: 102124 CreateTime: 1531477349286
isvalid: true keysize: -1 valuesize: 12 magic: 2
compresscodec: NONE producerId: -1 producerEpoch: -
1 sequence: -1 isTransactional: false headerKeys: []
payload: message_5371
```


offset 和 position 这两个前面已经讲过了、createTime 表示创建时间、keysize 和 valuesize 表示 key 和 value 的大小、 compresscodec 表示压缩编码、payload:表示消息的具体内容

### 日志的清除策略以及压缩策略

#### 日志清除策略

前面提到过，日志的分段存储，一方面能够减少单个文件内容的大小，另一方面，方便 kafka 进行日志清理。日志的清理策略有两个

* **根据消息的保留时间**，当消息在 kafka 中保存的时间超过了指定的时间，就会触发清理过程

* **根据 topic 存储的数据大小**，当 topic 所占的日志文件大小大于一定的阀值，则可以开始**删除最旧的消息**。kafka会启动一个后台线程，定期检查是否存在可以删除的消息通过 log.retention.bytes 和 log.retention.hours 这两个参数来设置，当其中任意一个达到要求，都会执行删除。默认的保留时间是：7 天

### 日志压缩策略

Kafka 还提供了“日志压缩（Log Compaction）”功能，通过这个功能可以有效的减少**日志文件的大小，缓解磁盘紧张的情况**，在很多实际场景中，消息的 key 和 value 的值之间的对应关系是不断变化的，就像数据库中的数据会不断被修改一样，消费者只关心 key 对应的最新的 value。因此，我们可以开启 kafka 的日志压缩功能，服务端会在后台启动启动Cleaner 线程池，**定期将相同的 key 进行合并，只保留最新的 value 值**。日志的压缩原理是

![img](E:\研究生学习\Work\技术笔记\Kafka消息存储.assets\20190814182913970.png)



————————————————
版权声明：本文为CSDN博主「小马的学习笔记」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/madongyu1259892936/article/details/99594896