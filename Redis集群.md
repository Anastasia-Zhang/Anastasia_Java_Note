# Redis 集群

## 集群的概念

所谓的集群，就是通过添加服务器的数量，提供相同的服务，从而让服务器达到一个稳定、高效的状态。

###  使用redis集群的必要性

问题：我们已经部署好了redis，并且能启动一个redis，实现数据的读写，为什么还要学习redis集群？

（1）单个redis存在不稳定性。当redis服务宕机了，就没有可用的服务了。

（2）单个redis的读写能力是有限的。

## redis的集群架构

###  Redis 主从复制模型

  主从复制模型中，有多个redis节点。

  其中，**有且仅有**一个为主节点Master。从节点Slave可以有多个。

 

只要网络连接正常，Master会一直将自己的数据更新同步给Slaves，保持主从同步。

 ![img](E:\研究生学习\Work\技术笔记\Redis集群.assets\1666036-20190715225123379-428287419.png)

### redis cluster

* **支撑N个redis master node**，每个master node都可以挂载多个slave node

* **读写分离的架构，对于每个master来说，写就写到master，然后读就从mater对应的slave去读**

* 高可用，因为每个master都有salve节点，那么如果mater挂掉，redis cluster这套机制，**就会自动将某个slave切换成master**
* redis cluster（多master + 读写分离 + 高可用）

我们只要基于redis cluster去搭建redis集群即可，不需要手工去搭建replication复制+主从架构+读写分离+哨兵集群+高可用

 

## 分布式数据存储算法

讲解分布式数据存储的核心算法，数据分布的算法

**hash算法 -> 一致性hash算法（memcached） -> redis cluster，hash slot算法**

用不同的算法，就决定了在多个master节点的时候，数据如何分布到这些节点上去，解决这个问题

### 1、redis cluster介绍

redis cluster

（1）自动将数据进行分片，每个master上放一部分数据
（2）提供内置的高可用支持，部分master不可用时，还是可以继续工作的

在redis cluster架构下，每个redis要放开两个端口号，比如一个是6379，另外一个就是加10000的端口号，比如16379

16379端口号是用来进行节点间通信的，也就是cluster bus的东西，集群总线。cluster bus的通信，用来进行故障检测，配置更新，故障转移授权

cluster bus用了另外一种二进制的协议，主要用于节点间进行高效的数据交换，占用更少的网络带宽和处理时间

### 2、最老土的hash算法和弊端（大量缓存重建）

![最老土的hash算法以及弊端](E:\研究生学习\Work\技术笔记\Redis集群.assets\最老土的hash算法以及弊端.png)

### 3、一致性hash算法（自动缓存迁移）+虚拟节点（自动负载均衡）

![一致性hash算法的讲解和优点](E:\研究生学习\Work\技术笔记\Redis集群.assets\一致性hash算法的讲解和优点.png)

![一致性hash算法的虚拟节点实现负载均衡](E:\研究生学习\Work\技术笔记\Redis集群.assets\一致性hash算法的虚拟节点实现负载均衡.png)

### 4、redis cluster的hash slot算法

![redis cluster hash slot算法](E:\研究生学习\Work\技术笔记\Redis集群.assets\redis cluster hash slot算法.png)

redis cluster有固定的16384个hash slot，对每个key计算CRC16值，然后对16384取模，可以获取key对应的hash slot

redis cluster中每个master都会持有部分slot，比如有3个master，那么可能每个master持有5000多个hash slot

hash slot让node的增加和移除很简单，增加一个master，就将其他master的hash slot移动部分过去，减少一个master，就将它的hash slot移动到其他master上去

移动hash slot的成本是非常低的

客户端的api，可以对指定的数据，让他们走同一个hash slot，通过hash tag来实现



## 主从复制

### redis replication的核心机制

（1）redis采用**异步方式复制数据到slave节点**，不过redis 2.8开始，slave node会周期性地确认自己每次复制的数据量
（2）一个master node是可以配置多个slave node的
（3）slave node也可以连接其他的slave node
（4）slave node做复制的时候，是不会block master node的正常工作的
（5）slave node在做复制的时候，**也不会block对自己的查询操作，它会用旧的数据集来提供服务; 但是复制完成的时候，需要删除旧数据集，加载新数据集，这个时候就会暂停对外服务了**
（6）slave node**主要用来进行横向扩容，做读写分离**，扩容的slave node可以提高读的吞吐量slave，高可用性，有很大的关系

#### master持久化对于主从架构的安全保障的意义

如果采用了主从架构，那么建议必须开启master node的持久化！

不建议用slave node作为master node的数据热备，因为那样的话，如果你关掉master的持久化，可能在master宕机重启的时候数据是空的，然后可能一经过复制，salve node数据也丢了

* master -> RDB和AOF都关闭了 -> 全部在内存中

* master宕机，重启，是没有本地数据可以恢复的，然后就会直接认为自己IDE数据是空的

* master就会将空的数据集同步到slave上去，所有slave的数据全部清空

* 100%的数据丢失

master节点，必须要使用持久化机制

第二个，master的各种备份方案，要不要做，万一说本地的所有文件丢失了; 从备份中挑选一份rdb去恢复master; 这样才能确保master启动的时候，是有数据的

即使采用了后续讲解的高可用机制，slave node可以自动接管master node，但是也可能sentinal还没有检测到master failure，master node就自动重启了，还是可能导致上面的所有slave node数据清空故障

### 主从架构复制的核心原理

1. 当启动一个slave node的时候，它会发送一个PSYNC命令给master node

2. 如果这是slave node重新连接master node，那么master node仅仅会复制给slave部分缺少的数据; 否则如果是slave node第一次连接master node，那么会触发一次full resynchronization

3. 开始full resynchronization的时候，master会启动一个后台线程，开始生成一份RDB快照文件，同时还会将从客户端收到的所有写命令缓存在内存中。RDB文件生成完毕之后，master会将这个RDB发送给slave，slave会先写入本地磁盘，然后再从本地磁盘加载到内存中。然后master会将内存中缓存的写命令发送给slave，slave也会同步这些数据。

4. slave node如果跟master node有网络故障，断开了连接，会自动重连。master如果发现有多个slave node都来重新连接，仅仅会启动一个rdb save操作，用一份数据服务所有slave node。

#### 主从复制的断点续传

从redis 2.8开始，就支持主从复制的断点续传，如果主从复制过程中，网络连接断掉了，那么可以接着上次复制的地方，继续复制下去，而不是从头开始复制一份

master node会在内存中常见一个backlog，master和slave都会保存一个replica offset还有一个master id，offset就是保存在backlog中的。如果master和slave网络连接断掉了，slave会让master从上次的replica offset开始继续复制

但是如果没有找到对应的offset，那么就会执行一次resynchronization

#### 无磁盘化复制

master在内存中直接创建rdb，然后发送给slave，不会在自己本地落地磁盘了

基于磁盘：RDB创建完成 slave 排队去数据

不基于磁盘：一个传输开始 slove 排队取数据，新的传输将会在旧的传输开始之前取数据

`repl-diskless-sync`
`repl-diskless-sync-delay`，等待一定时长再开始复制，因为要等更多slave重新连接过来

#### 过期key处理

slave不会过期key，只会等待master过期key。如果master过期了一个key，或者通过LRU淘汰了一个key，那么会模拟一条del命令发送给slave。



### 复制的完整流程及细节

（1）slave node启动，仅仅保存master node的信息，包括master node的host和ip，但是复制流程没开始

master host和ip是从哪儿来的，redis.conf里面的slaveof配置的

（2）slave node内部有个定时任务，每秒检查是否有新的master node要连接和复制，如果发现，就跟master node建立socket网络连接
（3）slave node发送ping命令给master node
（4）口令认证，如果master设置了requirepass，那么salve node必须发送masterauth的口令过去进行认证
（5）master node第一次执行全量复制，将所有数据发给slave node
（6）master node后续持续将写命令，异步复制给slave node

### 数据同步相关的核心机制

指的就是第一次slave连接msater的时候，执行的全量复制，那个过程里面你的一些细节的机制

#### （1）master和slave都会维护一个offset

* master会在自身不断累加offset，slave也会在自身不断累加offset
* **slave每秒都会上报自己的offset给master，同时master也会保存每个slave的offse**t

这个倒不是说特定就用在全量复制的，**主要是master和slave都要知道各自的数据的offset**，才能知道互相之间的数据不一致的情况，以及断点续传

#### （2）backlog

* master node有一个backlog，默认是1MB大小
* master node给slave node复制数据时，也会将数据在backlog中同步写一份
* **backlog主要是用来做全量复制中断候的增量复制的**

#### （3）master run id

info server，可以看到master run id
如果根据host+ip定位master node，是不靠谱的，如果master node重启或者数据出现了变化，**那么slave node应该根据不同的run id区分，run id不同就做全量复制**
如果需要不更改run id重启redis，可以使用redis-cli debug reload命令

#### （4）psync

* 从节点使用psync从master node进行复制，`psync runid offset`
* master node会根据自身的情况返回响应信息，可能是FULLRESYNC runid offset触发全量复制，可能是CONTINUE触发增量复制

### 全量复制

主节点生成 rdb

（1）master执行bgsave，在本地生成一份rdb快照文件
（2）master node将rdb快照文件发送给salve node，如果rdb复制时间超过60秒（repl-timeout），那么slave node就会认为复制失败，可以适当调节大这个参数
（3）对于千兆网卡的机器，一般每秒传输100MB，6G文件，很可能超过60s
（4）master node在生成rdb时，会将所有新的写命令缓存在内存中，在salve node保存了rdb之后，再将新的写命令复制给salve node
（5）client-output-buffer-limit slave 256MB 64MB 60，如果在复制期间，内存缓冲区持续消耗超过64MB，或者一次性超过256MB，那么停止复制，复制失败
（6）slave node接收到rdb之后，清空自己的旧数据，然后重新加载rdb到自己的内存中，同时基于旧的数据版本对外提供服务
（7）如果slave node开启了AOF，那么会立即执行BGREWRITEAOF，重写AOF

rdb生成、rdb通过网络拷贝、slave旧数据的清理、slave aof rewrite，很耗费时间

如果复制的数据量在4G~6G之间，那么很可能全量复制时间消耗到1分半到2分钟

#### 增量复制

（1）如果全量复制过程中，master-slave网络连接断掉，那么salve重新连接master时，会触发增量复制
（2）master直接从自己的backlog中获取部分丢失的数据，发送给slave node，默认backlog就是1MB
（3）msater就是根据slave发送的psync中的offset来从backlog中获取数据的

### heartbeat

主从节点互相都会发送heartbeat信息

master默认每隔10秒发送一次heartbeat，salve node每隔1秒发送一个heartbeat

### 异步复制

master每次接收到写命令之后，现在内部写入数据，然后异步发送给slave node

### 参考

https://www.jianshu.com/p/84dbb25cc8dc