# Kafka：基本介绍

#### 为什么要使用消息队列？

解耦、异步、限流削峰

### 简介

Kafka 是一个分布式的消息发布订阅系统；与传统的消息系统相比，他有以下优点：

* 分布式系统，易于向外扩展
* 同时为发布和订阅提高吞吐量
* 支持多订阅者，当失败时能自动平衡消费者
* 将消息持久化到磁盘，可用于批量消费
* 可以用来当分布式的消息队列和分布式提交日志



### 基本概念

**Producer**：发布消息的对象成为生产者

**Consumer**：订阅消息并处理发布的消息种子

**Topic** ：将消息进行分类，每一类消息成为话题

**Partition**：将消息进行分区，为了提高话题的吞吐量

**Broker**：已发布的消息保存在一组服务器中，称之为Kafka集群。集群中的每一个服务器都是一个代理(Broker). 消费者可以订阅一个或多个话题，并从Broker拉数据，从而消费这些已发布的消息。broker 有 leader 和 follower，他们之间进行消息复制

![Kafka集群](E:\研究生学习\Work\技术笔记\Kafaka.assets\producer_consumer.png)

**Consumer Group**：

一般来讲消息模型分两钟：队列和发布-订阅式

* **队列**：一组消费者从服务器读取消息，一条消息只有其中一个消费者来处理
* **发布-订阅**：消息被 **广播** 给所有的消费者，收到信息的消费者都可以处理此消息

消费者用一个消费者组名标记自己。 一个发布在Topic上消息被分发给此消费者组中的一个消费者。也就是说一个分区的数据只能被一个消费者组中的一个消费者所使用。

- 假如所有的消费者都在一个组中，那么这就变成了queue模型。
- 假如所有的消费者都在不同的组中，那么就完全变成了发布-订阅模型。

![A two server Kafka cluster hosting four partitions (E:\研究生学习\Work\技术笔记\Kafaka.assets\consumer-groups.png) with two consumer groups. Consumer group A has two consumer instances and group B has four](https://colobu.com/2014/08/06/kafka-quickstart/consumer-groups.png)

上图中有两个Server节点，有一个Topic被分为四个分区（P0-P4)分别被分配在两个节点上，另外还有两个消费者组（GA，GB），其中GA有两个消费者实例，GB有四个消费者实例。

从图中我们可以看出，首先订阅Topic的单位是消费者组，另外我们发现Topic中的消息根据一定规则将消息推送给具体消费者，主要原则如下：

- 若消费者数小于partition数，且消费者数为一个，那么它就消费所有分区消息；
- 若消费者数小于partition数，假设消费者数为N，partition数为M，那么每个消费者能消费的分区数为M/N或M/N+1；
- 若消费者数等于partition数，那么每个消费者都会均等分配到一个分区的消息；
- 若消费者数大于partition数，则将会出现部分消费者得不到消息分区，出现空闲的情况；

总的来说，Kafka会根据消费者组的情况均衡分配消息，比如有消息着实例宕机，亦或者有新的消费者加入等情况

#### Partition 和 Topic 

生产者往 Topic 丢数据，一个 Topic 会分为多个 Partition，实际上 Partition 会分布都在不同的Broker中。

![log-anatomy](E:\研究生学习\Work\技术笔记\Kafaka.assets\log-anatomy.png)

**其中每个partition中的消息是有序的，多个partition之间不一定有序**。**Topic分区中的信息只能由消费者组的找那个的唯一一个消费者处理**。只能保证一个分区内的消息按照先后顺序处理。若Topic有多个partition，生产者的消息可以指定或者由系统根据算法分配到指定分区，若你需要所有消息都是有序的，那么你最好只用一个分区。另外partition支持**消息位移或者偏移量（offset）**读取。偏移量是不断递增的整数值，创建消息时。Kafka会把他添加到消息里。在给定的分区里，每个消息的偏移量都是唯一的。消费者把每个分区最后读取的偏移量保存在Zookeeper 或者 Kafka 上，如果消息者关闭或者重启，他的读取状态不会丢失。

#### 分布式

数据分布在不同的 Partition 中，Partition 中备份在不同的Broker中。这就说明他天生就是分布式的。生产者和消费者主要是和主分区（leader）做交互，备份分区（followers）仅仅用作于备份，不做读写。如果某个 Broker 宕机，则会选出其他的 Partition 作为主分区（leader），这就实现了**高可用**。

#### Partition 持久化

Kafka是将partition的数据写在**磁盘**的(消息日志)，不过Kafka只允许**追加写入**(顺序访问)，避免缓慢的随机 I/O 操作。Kafka也不是partition一有数据就立马将数据写到磁盘上，它会先**缓存**一部分，等到足够多数据量或等待一定的时间再批量写入(flush)。

### 四大核心接口

Kafka为了拥有更强大的功能，提供了四大核心接口：

- Producer API允许了应用可以向Kafka中的topics发布消息；
- Consumer API允许了应用可以订阅Kafka中的topics,并消费消息；
- Streams API允许应用可以作为消息流的处理者，比如可以从topicA中消费消息，处理的结果发布到topicB中；
- Connector API提供Kafka与现有的应用或系统适配功能，比如与数据库连接器可以捕获表结构的变化；

### Kafka 保证

kafka作为一个高水准的系统，提供了以下的保证：

- 消息的添加是有序的，生产者越早向订阅的Topic发送的消息，会更早的被添加到Topic中，当然它们可能被分配到不同的分区；
- 消费者在消费Topic分区中的消息时是有序的；
- 对于有N个复制节点的Topic，系统可以最多容忍N-1个节点发生故障，而不丢失任何提交给该Topic的消息丢失；

### Kafka 与 传统的消息系统的区别

1.消息队列

| 特性     | 描述                                                         |
| :------- | :----------------------------------------------------------- |
| 表现形式 | 一组消费者从消息队列中获取消息，消息会被推送给组中的某一个消费者 |
| 优势     | 水平扩展，可以将消息数据分开处理                             |
| 劣势     | 消息队列不是多用户的，当一条消息记录被一个进程读取后，消息便会丢失 |

2.发布/订阅

| 特性     | 描述                                                         |
| :------- | :----------------------------------------------------------- |
| 表现形式 | 消息会广播发送给所有消费者                                   |
| 优势     | 可以多进程共享消息                                           |
| 劣势     | 每个消费者都会获得所有消息，无法通过添加消费进程提高处理效率 |

Kafka的优化点主要体现在以下两方面：

- 通过Topic方式来达到消息队列的功能
- 通过消费者组这种方式来达到发布/订阅的功能

### Kafka的存储功能

存储消息也是消息系统的一大功能，Kafka相对普通的消息队列存储来说，它的表现实在好的太多，

* 首先Kafka支持写入确认，保证消息写入的正确性和连续性
* 同时Kafka还会对写入磁盘的数据进行复制备份，来实现容错
* 另外Kafka对磁盘的使用结构是一致的，不管你的服务器目前磁盘存储的消息数据有多少，它添加消息数据的效率是相同的。

Kafka的存储机制很好的支持消费者可以随意控制自身所需要读取的数据，在很多时候你也可以将Kafka作为一个高性能，低延迟的分布式文件系统。

### Kafka 的流数据处理功能

很多时候原始的数据并不是我们想要的，我们想要的是经过处理后的数据结果，比如通过一天的搜索数据得出当天的搜索热点等，你可以利用Streams API来实现自己想要的功能，比如从输入Topic中获取数据，然后再发布到具体的输出Topic中。

Kafka的流处理可以解决诸如处理无序数据、数据的复杂转换等问题。



### Kafaka的优点

* #### 多个生产者

  支持多个生产者，不管客户端在使用单个主题或者多个主题，可以用来从多个前端系统手机数据

* #### 多个消费者

* #### 基于磁盘的数据存储

  可以非实时的读取信息持久化的数据可以保证数据不被丢失

* #### 伸缩性

  可以不断调整broker的数量

* #### 高性能

  可以横向扩展 生产者，消费者，broker 处理巨大的信息流

  

### 使用场景

#### 1. 活动跟踪

比如前端应用程序生成的用户活动相关的信息。可以生成前端的报告，为机器学习系统提供数据。

#### 2. 传递消息

发送邮件、异步更新、发送消息。一些公共应用程序读取消息。（消息队列）

#### 3. 度量指标和日志记录

应用程序定期吧度量指标发布到Kafka主题上，监控系统或者告警系统读取这些消息

#### 4. 提交日志

把数据库的更新发布到 Kafka 上，应用程序通过监控事件流来接收数据库的实时更新

#### 5. 流处理

处理实时数据流，有点像 Hadoop 处理的数据

### 几种消息队列的对比

| 特性                     | ActiveMQ                              | RabbitMQ                                           | RocketMQ                                                     | Kafka                                                        |
| ------------------------ | ------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 单机吞吐量               | 万级，比 RocketMQ、Kafka 低一个数量级 | 同 ActiveMQ                                        | 10 万级，支撑高吞吐                                          | 10 万级，高吞吐，一般配合大数据类的系统来进行实时数据计算、日志采集等场景 |
| topic 数量对吞吐量的影响 |                                       |                                                    | topic 可以达到几百/几千的级别，吞吐量会有较小幅度的下降，这是 RocketMQ 的一大优势，在同等机器下，可以支撑大量的 topic | topic 从几十到几百个时候，吞吐量会大幅度下降，在同等机器下，Kafka 尽量保证 topic 数量不要过多，如果要支撑大规模的 topic，需要增加更多的机器资源 |
| 时效性                   | ms 级                                 | 微秒级，这是 RabbitMQ 的一大特点，延迟最低         | ms 级                                                        | 延迟在 ms 级以内                                             |
| 可用性                   | 高，基于主从架构实现高可用            | 同 ActiveMQ                                        | 非常高，分布式架构                                           | 非常高，分布式，一个数据多个副本，少数机器宕机，不会丢失数据，不会导致不可用 |
| 消息可靠性               | 有较低的概率丢失数据                  | 基本不丢                                           | 经过参数优化配置，可以做到 0 丢失                            | 同 RocketMQ                                                  |
| 功能支持                 | MQ 领域的功能极其完备                 | 基于 erlang 开发，并发能力很强，性能极好，延时很低 | MQ 功能较为完善，还是分布式的，扩展性好                      | 功能较为简单，主要支持简单的 MQ 功能，在大数据领域的实时计算以及日志采集被大规模使用 |

## 参考

* https://scala.cool/2018/03/learning-kafka-1/
* https://colobu.com/2014/08/06/kafka-quickstart/