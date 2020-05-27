# Kafka 数据消费

### 重复消费的情况

![图片](E:\研究生学习\Work\技术笔记\Kafka数据消费.assets\图片.png)

### 如何保证幂等性

![图片](E:\研究生学习\Work\技术笔记\Kafka数据消费.assets\图片-1582886952177.png)

消费之后把数据放在内存里，看数据是否在内存中。消费之后去看内存中有没有这个数据，要发现有重复数据就丢失

（1）比如你拿个数据要写库，你先根据主键查一下，如果这数据都有了，你就别插入了，update一下好吧

（2）比如你是写redis，那没问题了，反正每次都是set，天然幂等性

（3）比如你不是上面两个场景，那做的稍微复杂一点，你需要让生产者发送每条数据的时候，里面加一个全局唯一的id，类似订单id之类的东西，然后你这里消费到了之后，先根据这个id去比如redis里查一下，之前消费过吗？如果没有消费过，你就处理，然后这个id写redis。如果消费过了，那你就别处理了，保证别重复处理相同的消息即可。

 

还有比如基于数据库的唯一键来保证重复数据不会重复插入多条，我们之前线上系统就有这个问题，就是拿到数据的时候，每次重启可能会有重复，因为kafka消费者还没来得及提交offset，重复数据拿到了以后我们插入的时候，因为有唯一键约束了，所以重复数据只会插入报错，不会导致数据库中出现脏数据



### Kafka 如何手动提交

### **一、概述**

在新消费者客户端中，消费位移是存储在Kafka内部的主题 __consumer_offsets 中。把消费位移存储起来（持久化）的动作称为 “提交” ，**消费者在消费完消息之后需要执行消费位移的提交**。

参考下图的消费位移，x 表示某一次拉取操作中此分区消息的最大偏移量，假设当前消费者已经消费了 x 位置的消息，那么我们就可以说消费者的消费位移为 x ，图中也用了 lastConsumedOffset 这个单词来标识它。

![img](E:\研究生学习\Work\技术笔记\Kafka数据消费.assets\p3b3dnrce2.jpeg)

不过需要非常明确的是，当前消费者需要提交的消费位移并不是 x ，而是 x+1 ，对应上图中的 position ，它表示下一条需要拉取的消息的位置。

KafkaConsumer 类提供了 partition(TopicPartition) 和 committed(TopicPartition) 两个方法来分别获取上面所说的 postion 和 committed offset 的值。这两个方法的定义如下所示：

- public long position(TopicPartition partition)
- public OffsetAndMetadata committed(TopicPartition partition)

可通过 TestOffsetAndPosition.java 来测试consumed offset、committed offset、position之间的关系。该 TestOffsetAndPosition.java 文件的地址为：

https://github.com/841809077/hdpproject/blob/master/src/main/java/com/hdp/project/kafka/consumer/TestOffsetAndPosition.java

### **二、offset 提交的两种方式**

#### **1、自动提交**

在 Kafka 中默认的消费位移的提交方式为自动提交，这个由消费者客户端参数 enable.auto.commit 配置，默认值为 true 。这个默认的自动提交不是每消费一条消息就提交一次，而是定期提交，这个定期的周期时间由客户端 auto.commit.interval.ms 配置，默认值为 5 秒，此参数生效的前提是 enable.auto.commit 参数为 true 。

在默认的配置下，消费者每隔 5 秒会将拉取到的每个分区中最大的消息位移进行提交。自动位移提交的动作是在 poll() 方法的逻辑里完成的，在每次真正向服务端发起拉取请求之前会检查是否可以进行位移提交，如果可以，那么就会提交上一次轮询的位移。

#### **2、手动提交**

Kafka 自动提交消费位移的方式非常简便，它免去了复杂的位移提交逻辑，但并没有为开发者留有余地来处理**重复消费**和**消息丢失**的问题。自动位移提交无法做到精确的位移管理，所以Kafka还提供了手动位移提交的方式，这样就可以使得开发人员对消费位移的管理控制更加灵活。**开启手动提交功能的前提是消费者客户端参数 enable.auto.commit 配置为 false 。**示例如下：

```javascript
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
```

手动提交又分为**同步提交**和**异步提交**，对应于 KafkaConsumer 中的 commitSync() 和 commitAsync() 两种类型的方法。

##### **2.1、同步提交**

消费者可以调用 commitSync() 方法，来实现位移的同步提交。

**commitSync() 方法会根据 poll() 方法拉取的最新位移来进行提交，只要没有发生不可回复的错误，它就会阻塞消费者线程直至位移提交完成。**

对于采用 commitSync() 的无参方法而言，它提交消费位移的频率和拉取批次消息、处理批次消息的频率是一样的。如果想寻求更细粒度的、更精准的提交，那么就需要使用 commitSync() 的另一个含参方法，具体定义如下：

```javascript
public void commitSync(Map<TopicPartition, OffsetAndMetadata> offsets)
```

该方法提供了一个 offsets 参数，用来提交指定分区的位移。

##### **2.2、异步提交**

**与 commitSync() 方法相反，异步提交的方式在执行的时候消费者线程不会被阻塞，可以在提交消费位移的结果还未返回之前就开始新一次的拉取操作**。异步提交可以使消费者的性能得到一定的增强。commitAsync() 方法有三个不同的重载方法：

```javascript
public void commitAsync()
public void commitAsync(OffsetCommitCallback callback)
public void commitAsync(Map<TopicPartition, OffsetAndMetadata> offsets, OffsetCommitCallback callback)   
```

第一个无参方法和第三个方法中的 offsets 都很好理解，对照 commitSync() 方法即可。关键是第二个方法与第三个方法的 callback 参数，它提供了一个异步提交的回调方法，当位移提交完成后会回调 OffsetCommitCallback 中的 onComplete() 方法。如下图所示：

![img](E:\研究生学习\Work\技术笔记\Kafka数据消费.assets\hfiquj4k3g.jpeg)

发送提交请求后可以继续做其它事情。如果提交失败，错误信息和偏移量会被记录下来。

### **三、同步和异步组合提交**

一般情况下，针对偶尔出现的提交失败，不进行重试不会有太大问题，因为如果提交失败是因为临时问题导致的，那么后续的提交总会有成功的。但如果这是发生在 关闭消费者 或 再均衡（分区的所属权从一个消费者转移到另一个消费者的行为） 前的最后一次提交，就要确保能够提交成功。

因此，在消费者关闭前一般会组合使用 commitAsync() 和 commitSync() 。使用 commitAsync() 方式来做每条消费信息的提交（因为该种方式速度更快），最后再使用 commitSync() 方式来做位移提交最后的保证。

```javascript
try {
    while (true) {
        // 消费者poll并且执行一些操作
        // ...

        // 异步提交，也可使用有回调函数的异步提交。较同步提交速度更快。
        consumer.commitAsync();
    }
} catch (Exception e) {
    logger.error("Unexpected error" , e);
} finally {
    try {
        // 同步提交，来做位移提交最后的保证。
        consumer.commitSync();
    } finally {
        consumer.close();
    }
}
```