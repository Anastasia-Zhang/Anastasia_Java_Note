## kafka弄丢了数据

 

这块比较常见的一个场景，就是kafka某个broker宕机，然后重新选举partiton的leader时。大家想想，**要是此时其他的follower刚好还有些数据没有同步，结果此时leader挂了，然后选举某个follower成leader之后，他不就少了一些数据？这就丢了一些数据啊。**

生产环境也遇到过，我们也是，之前kafka的leader机器宕机了，将follower切换为leader之后，就会发现说这个数据就丢了

所以此时一般是要求起码设置如下4个参数： 

* 给这个topic设置 `replication.factor` 参数：这个值必须大于1，**要求每个partition必须有至少2个副本**

* 在kafka服务端设置 `min.insync.replicas` 参数：这个值必须大于1，这个是要求一个leader至少感知到有至少一个follower还跟自己保持联系，没掉队，这样才能确保leader挂了还有一个follower吧

* 在producer端设置 `acks=all`：这个是要求每条数据**，必须是写入所有replica（副本）之后，才能认为是写成功了**
* 在producer端设置 `retries=MAX`（很大很大很大的一个值，无限次重试的意思）：这个是要求一旦写入失败，就无限重试，卡在这里了

 

我们生产环境就是按照上述要求配置的，这样配置之后，至少在kafka **broker端就可以保证在leader所在broker发生故障，进行leader切换时，数据不会丢失**

 

3）生产者会不会弄丢数据

如果按照上述的思路设置了ack=all，一定不会丢，要求是，你的leader接收到消息，所有的follower都同步到了消息之后，才认为本次写成功了。如果没满足这个条件，生产者会自动不断的重试，重试无限次。



### Kafka 如何保证信息不丢失不重复

**发送端：同步发送等待所有的副本都同步成功之后返回成功，消费端关掉自动提交位移，消息处理完之后手动提交位移**

* **消费端**：如果在消息处理完成前就提交了offset，那么就有可能造成数据的丢失。由于Kafka consumer默认是自动提交位移的，所以在后台提交位移前一定要保证消息被正常处理了，因此不建议采用很重的处理逻辑，如果处理耗时很长，则建议把逻辑放到另一个线程中去做。为了避免数据丢失，现给出两点建议：
  enable.auto.commit=false  关闭自动提交位移
  在消息被完整处理之后再手动提交位移
  ![image-20200303110519056](E:\研究生学习\Work\技术笔记\Kafka信息丢失.assets\image-20200303110519056.png)

* **生产者**

  * **kafka同步生产者**：这个生产者写一条消息的时候，它就立马发送到某个分区去。follower还需要从leader拉取消息到本地，follower再向leader发送确认，leader再向客户端发送确认。由于这一套流程之后，客户端才能得到确认，所以很慢。

    同步发送消息后会产生一个发送消息的记录

    ![image-20200303112158803](E:\研究生学习\Work\技术笔记\Kafka信息丢失.assets\image-20200303112158803.png)

  * **kafka异步生产者**：这个生产者写一条消息的时候，先是写到某个缓冲区，这个缓冲区里的数据还没写到broker集群里的某个分区的时候，它就返回到client去了。虽然效率快，但是不能保证消息一定被发送出去了。

客户端向topic发送数据分为两种方式：
producer.type=sync 同步模式 
producer.type=async 异步模式 

**两种模式下的可能造成的信息丢失问题：**

1）使用同步模式的时候，有3种状态保证消息被安全生产，在配置为1（只保证写入leader成功）的话，如果刚好leader partition挂了，数据就会丢失。
2）还有一种情况可能会丢失消息，就是使用异步模式的时候，当缓冲区满了，如果设置在还没有收到确认的情况下，缓冲池一满，就清空缓冲池里的消息，数据就会被立即丢弃掉。

**解决方案**：

1）在同步模式的时候，ack = -1，也就是让消息写入leader和所有的副本。
2）在异步模式下，如果消息发出去了，但还没有收到确认的时候，缓冲池满了，**在配置文件中设置成不限制阻塞超时的时间**，也就说让生产端一直阻塞，这样也能保证数据不会丢失。



消息队列的问题都要从源头找问题，就是生产者是否有问题。

讨论一种情况，如果数据发送成功，但是接受response的时候丢失了，机器重启之后就会重发。

重发很好解决，消费端增加去重表就能解决，但是如果生产者丢失了数据，问题就很麻烦了。

**数据重复消费**的情况，如果处理
（1）**去重：将消息的唯一标识保存到外部介质中，每次消费处理时判断是否处理过；**
（2）不管：大数据场景中，报表系统或者日志信息丢失几条都无所谓，不会影响最终的统计分析结





#### 具体分析

Kafka到底会不会丢数据(data loss)? 通常不会，但有些情况下的确有可能会发生。下面的参数配置及Best practice列表可以较好地保证数据的持久性(当然是trade-off，牺牲了吞吐量)。笔者会在该列表之后对列表中的每一项进行讨论，有兴趣的同学可以看下后面的分析。
没有银弹，如果想要高吞吐量就要能容忍偶尔的失败（重发漏发无顺序保证）。

```shell
block.on.buffer.full = true
acks = all
retries = MAX_VALUE
max.in.flight.requests.per.connection = 1
# 使用KafkaProducer.send(record, callback) callback逻辑中显式关闭producer：close(0) 
unclean.leader.election.enable=false
replication.factor = 3 
min.insync.replicas = 2
replication.factor > min.insync.replicas
enable.auto.commit=false
消息处理完成之后再提交位移
```


给出列表之后，我们从两个方面来探讨一下数据为什么会丢失：

#### Producer端

##### 消息丢失问题

目前比较新版本的Kafka正式替换了Scala版本的old producer，使用了由Java重写的producer。**新版本的producer采用异步发送机制。KafkaProducer.send(ProducerRecord)方法仅仅是把这条消息放入一个缓存中(即RecordAccumulator，本质上使用了队列来缓存记录)，同时后台的IO线程会不断扫描该缓存区，将满足条件的消息封装到某个batch中然后发送出去。**显然，这个过程中就有一个数据丢失的窗口：若IO线程发送之前client端挂掉了，累积在accumulator中的数据的确有可能会丢失。

##### 消息乱序问题

Producer的另一个问题是消息的乱序问题。假设客户端代码依次执行下面的语句将两条消息发到相同的分区
producer.send(record1);
producer.send(record2);
如果此时由于某些原因(比如瞬时的网络抖动)导致record1没有成功发送，同时Kafka又配置了重试机制和max.in.flight.requests.per.connection大于1(默认值是5，本来就是大于1的)，那么重试record1成功后，record1在分区中就在record2之后，从而造成消息的乱序。很多某些要求强顺序保证的场景是不允许出现这种情况的。发送之后重发就会丢失顺序

##### 解决方案

鉴于producer的这两个问题，我们应该如何规避呢？？对于消息丢失的问题，很容易想到的一个方案就是：既然异步发送有可能丢失数据， 我改成同步发送总可以吧？比如这样：

producer.send(record).get();
这样当然是可以的，但是性能会很差，不建议这样使用。因此特意总结了一份配置列表。个人认为该配置清单应该能够比较好地规避producer端数据丢失情况的发生：(特此说明一下，软件配置的很多决策都是trade-off，下面的配置也不例外：应用了这些配置，你可能会发现你的producer/consumer 吞吐量会下降，这是正常的，因为你换取了更高的数据安全性)

1. `block.on.buffer.full = true`  尽管该参数在0.9.0.0已经被标记为“deprecated”，但鉴于它的含义非常直观，所以这里还是显式设置它为true，**使得producer将一直等待缓冲区直至其变为可用。否则如果producer生产速度过快耗尽了缓冲区，producer将抛出异常。缓冲区满了就阻塞在那，不设置阻塞超时时间，不抛异常，也不要丢失数据**
2. `acks=all`  很好理解，所有follower都响应了才认为消息提交成功，即"committed"
3. `retries = MAX` 无限重试，直到你意识到出现了问题
4. `max.in.flight.requests.per.connection = 1` 限制客户端在单个连接上能够发送的未响应请求的个数。设置此值是1表示**kafka broker在响应请求之前client不能再向同一个broker发送请求**。注意：设**置此参数是为了避免消息乱序**
5. 使用KafkaProducer.send(record, callback)而不是send(record)方法   **自定义回调逻辑处理消息发送失败，比如记录在日志中，用定时脚本扫描重处理** 
6. callback逻辑中最好显式关闭producer：close(0) 注意：设置此参数是为了避免消息乱序（仅仅因为一条消息发送没收到反馈就关闭生产者，感觉代价很大）
7. unclean.leader.election.enable=false   关闭unclean leader选举，即不允许非ISR中的副本被选举为leader，以避免数据丢失
8. replication.factor >= 3   这个完全是个人建议了，参考了Hadoop及业界通用的三备份原
9. min.insync.replicas > 1 消息至少要被写入到这么多副本才算成功，也是提升数据持久性的一个参数。与acks配合使用，这个是要求一个leader至少感知到有至少一个follower还跟自己保持联系，没掉队，这样才能确保leader挂了还有一个follower吧
10. 保证replication.factor > min.insync.replicas  如果两者相等，当一个副本挂掉了分区也就没法正常工作了。通常设置replication.factor = min.insync.replicas + 1即可

####  Consumer端
consumer端丢失消息的情形比较简单：如果在消息处理完成前就提交了offset，那么就有可能造成数据的丢失。由于Kafka consumer默认是自动提交位移的，所以在后台提交位移前一定要保证消息被正常处理了，因此不建议采用很重的处理逻辑，如果处理耗时很长，则建议把逻辑放到另一个线程中去做。为了避免数据丢失，现给出两点建议：
enable.auto.commit=false  关闭自动提交位移
在消息被完整处理之后再手动提交位移