

# Redis 哨兵模式

**主从切换技术的方法是：当主服务器宕机后，需要手动把一台从服务器切换为主服务器，这就需要人工干预，费事费力，还会造成一段时间内服务不可用。**这不是一种推荐的方式，更多时候，我们优先考虑**哨兵模式**。



## 一、哨兵模式概述

哨兵模式是一种特殊的模式，首先Redis提供了哨兵的命令，哨兵是一个独立的进程，作为进程，它会独立运行。其原理是**哨兵通过发送命令，等待Redis服务器响应，从而监控运行的多个Redis实例。**

![img](https:////upload-images.jianshu.io/upload_images/11320039-57a77ca2757d0924.png?imageMogr2/auto-orient/strip|imageView2/2/w/507/format/webp)



**哨兵的核心功能是主节点的自动故障转移。**下面是Redis官方文档对于哨兵功能的描述：

- 监控（Monitoring）：哨兵会不断地检查主节点和从节点是否运作正常。
- 自动故障转移（Automatic failover）：当主节点不能正常工作时，哨兵会开始自动故障转移操作，它会将失效主节点的其中一个从节点升级为新的主节点，并让其他从节点改为复制新的主节点。
- 配置提供者（Configuration provider）：客户端在初始化时，**通过连接哨兵来获得当前Redis服务的主节点地址。**
- 通知（Notification）：**哨兵可以将故障转移的结果发送给客户端**。

其中，**监控和自动故障转移功能，使得哨兵可以及时发现主节点故障并完成转移**；而配置提供者和通知功能，则需要在与客户端的交互中才能体现。



## 二、Redis 实现高可用的技术

在介绍哨兵之前，首先从宏观角度回顾一下Redis实现高可用相关的技术。它们包括：持久化、复制、哨兵和集群，其主要作用和解决的问题是：

- 持久化：持久化是最简单的高可用方法(有时甚至不被归为高可用的手段)，主要作用是数据备份，即将数据存储在硬盘，保证数据不会因进程退出而丢失。
- 复制：复制是高可用Redis的基础，哨兵和集群都是在复制基础上实现高可用的。复制主要实现了数据的多机备份，以及对于读操作的负载均衡和简单的故障恢复。缺陷：故障恢复无法自动化；写操作无法负载均衡；存储能力受到单机的限制。
- 哨兵：在复制的基础上，**哨兵实现了自动化的故障恢复**。缺陷：写操作无法负载均衡；存储能力受到单机的限制。
- 集群：通过集群，Redis解决了写操作无法负载均衡，以及存储能力受到单机限制的问题，实现了较为完善的高可用方案。



## 三、哨兵的主要功能原理

![img](E:\研究生学习\Work\技术笔记\Redis哨兵.assets\1174710-20180908182924632-1069251418.png)



通过发送命令，让Redis服务器返回监控其运行状态，包括主服务器和从服务器。

当哨兵监测到master宕机，会自动将slave切换成master，然后通过**发布订阅模式**通知其他的从服务器，修改配置文件，让它们切换主机。

然而一个哨兵进程对Redis服务器进行监控，可能会出现问题，为此，我们可以使用多个哨兵进行监控。各个哨兵之间还会进行监控，这样就形成了多哨兵模式。

用文字描述一下**故障切换（failover）**的过程。假设主服务器宕机，哨兵1先检测到这个结果，系统并不会马上进行failover过程，仅仅是哨兵1主观的认为主服务器不可用，这个现象成为**主观下线**。当后面的哨兵也检测到主服务器不可用，并且数量达到一定值时，那么哨兵之间就会进行一次投票，投票的结果由一个哨兵发起，进行failover操作。切换成功后，就会通过发布订阅模式，让各个哨兵把自己监控的从服务器实现切换主机，这个过程称为**客观下线**。这样对于客户端而言，一切都是透明的。

### 基本原理

关于哨兵的原理，关键是了解以下几个概念。

（1）定时任务：每个哨兵节点维护了3个定时任务

* 定时任务的功能分别如下：
  * 通过向主从节点发送info命令获取最新的主从结构；
  * 通过发布订阅功能获取其他哨兵节点的信息；
  * 通过向其他节点发送ping命令进行心跳检测，判断是否下线

（2）**主观下线**：在心跳检测的定时任务中，如果其他节点超过一定时间没有回复，哨兵节点就会将其进行主观下线。顾名思义，主观下线的意思是**一个哨兵节点“主观地”判断下线**；与主观下线相对应的是客观下线。

（3）**客观下线**：哨兵节点在对主节点进行主观下线后，会通过sentinel is-master-down-by-addr命令**询问其他哨兵节点该主节点的状态；如果判断主节点下线的哨兵数量达到一定数值（一般为一半），则对该主节点进行客观下线**。

> （光我自己一个人检测到不行，万一我检测错了呢，我得问问其他的哨兵，是不是你们也发现那个主节点挂掉了，如果其他的小伙伴也认为是主服务器不行了，那肯定他不行了）

**需要特别注意的是，客观下线是主节点才有的概念；如果从节点和哨兵节点发生故障，被哨兵主观下线后，不会再有后续的客观下线和故障转移操作。**

（4）**选举领导者哨兵节点**：当主节点被判断客观下线以后，各**个哨兵节点会进行协商，选举出一个领导者哨兵节点，并由该领导者节点对其进行故障转移操作。**

> (他不行了，为了抢救整个集群我得选举个leader，我的小伙伴们都相当leader咋整？先到先得呗！Raft算法：一个小伙伴首提出想当leader ，我接受到消息之前并发现其他人告诉我他们的想法，他是第一个告诉我的，好了，就你了！) 

监视该主节点的所有哨兵都有可能被选为领导者，选举使用的算法是Raft算法；Raft算法的基本思路是先到先得：即在一轮选举中，哨兵A向B发送成为领导者的申请，如果B没有同意过其他哨兵，则会同意A成为领导者。选举的具体过程这里不做详细描述，一般来说，哨兵选择的过程很快，谁先完成客观下线，一般就能成为领导者。

（5）**故障转移**：选举出的领导者哨兵，开始进行故障转移操作，该操作大体可以分为3个步骤：

- **在从节点中选择新的主节点**：选择的原则是，首先过滤掉不健康的从节点；然后选择优先级最高的从节点(由slave-priority指定)；如果优先级无法区分，则选择复制偏移量最大的从节点；如果仍无法区分，则选择runid最小的从节点。
- **更新主从状态**：通过slaveof no one命令，让选出来的从节点成为主节点；并通过slaveof命令让其他节点成为其从节点。
- **将已经下线的主节点(即6379)设置为新的主节点的从节点**，当6379重新上线后，它会成为新的主节点的从节点。



## 参考

* https://www.jianshu.com/p/06ab9daf921d
  