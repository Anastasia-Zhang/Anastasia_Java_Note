# 5.全局锁&表锁&行锁

根据加锁的范围，MySQL里面的锁大致可以分成全局锁、表级锁和行锁三类。

## 5.1.全局锁

顾名思义，全局锁就是对整个数据库实例加锁。MySQL提供了一个加全局读锁的方法，命令是 **Flush tables with read lock** (FTWRL)。当你需要让整个库处于只读状态的时候，可以使用这个命令，之后其他线程的以下语句会被阻塞：数据更新语句（数据的增删改）、数据定义语句（包括建表、修改表结构等）和更新类事务的提交语句。

**全局锁的典型使用场景是，做全库逻辑备份。也就是把整库每个表都select出来存成文本。**

以前有一种做法，是通过FTWRL确保不会有其他线程对数据库做更新，然后对整个库做备份。注意，在备份过程中整个库完全处于只读状态。

但是让整库都只读，听上去就很危险：

- 如果你在主库上备份，那么在备份期间都不能执行更新，业务基本上就得停摆；
- 如果你在从库上备份，那么备份期间从库不能执行主库同步过来的binlog，会导致主从延迟。

### 为啥使用全局锁 ？

不加锁的话，备份系统备份的得到的库不是一个逻辑时间点，这个视图是逻辑不一致的。

说到视图你肯定想起来了，我们在前面讲事务隔离的时候，其实是有一个方法能够拿到一致性视图的，对吧？



#### 事务视图与全局锁

**可以利用可重复读这个数据隔离级别来得到一致性的视图：**官方自带的逻辑备份工具是mysqldump。当mysqldump使用参数–single-transaction的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。而由于MVCC的支持，这个过程中数据是可以正常更新的。

**有了这个功能，为什么还需要FTWRL呢？一致性读是好，但前提是引擎要支持这个隔离级别。比如，对于MyISAM这种不支持事务的引擎，如果备份过程中有更新，总是只能取到最新的数据，那么就破坏了备份的一致性。这时，我们就需要使用FTWRL命令了**。

所以，**single-transaction方法只适用于所有的表使用事务引擎的库**。如果有的表使用了不支持事务的引擎，那么备份就只能通过FTWRL方法。这往往是DBA要求业务开发人员使用InnoDB替代MyISAM的原因之一。



#### 只读与全局锁

既然要全库只读，**为什么不使用set global readonly=true的方式呢**？

确实readonly方式也可以让全库进入只读状态，但我还是会建议你用FTWRL方式，主要有两个原因：

* 一是，在有些系统中，readonly的值会被用来做其他逻辑，比如用来判断一个库是主库还是备库。因此，**修改global变量的方式影响面更大**，我不建议你使用。

* 二是，在异常处理机制上有差异。
  * 出现异常后锁会自动释放：**如果执行FTWRL命令之后由于客户端发生异常断开，那么MySQL会自动释放这个全局锁，整个库回到可以正常更新的状态。**
  * 出现异常后数据库会一直保持只读状态：**而将整个库设置为readonly之后，如果客户端发生异常，则数据库就会一直保持readonly状态，这样会导致整个库长时间处于不可写状态，风险较高**。

业务的更新不只是增删改数据（DML)，还有可能是加字段等修改表结构的操作（DDL）。不论是哪种方法，一个库被全局锁上以后，你要对里面任何一个表做加字段操作，都是会被锁住的。

但是，即使没有被全局锁住，加字段也不是就能一帆风顺的，因为你还会碰到接下来我们要介绍的表级锁。



## 5.2.表级锁

MySQL里面表级别的锁有两种：**一种是表锁，一种是元数据锁**（meta data lock，MDL)。

### 表锁

**表锁的语法是 lock tables … read/write**。与FTWRL类似，可以用unlock tables主动释放锁，也可以在客户端断开的时候自动释放。需要注意，**lock tables语法除了会限制别的线程的读写外，也限定了本线程接下来的操作对象**。

举个例子, 如果在某个线程A中执行lock tables t1 read, t2 write; 这个语句，则其他线程写t1、读写t2的语句都会被阻塞。同时，线程A在执行unlock tables之前，也只能执行读t1、读写t2的操作。连写t1都不允许，自然也不能访问其他表。

在还没有出现更细粒度的锁的时候，表锁是最常用的处理并发的方式。而对于InnoDB这种支持行锁的引擎，一般不使用lock tables命令来控制并发，毕竟锁住整个表的影响面还是太大。

### 元数据锁

**另一类表级的锁是MDL（metadata lock)**。MDL不需要显式使用，**在访问一个表的时候会被自动加上**。MDL的作用是，保证读写的正确性。你可以想象一下，如果一个查询正在遍历一个表中的数据，而执行期间另一个线程对这个表结构做变更，删了一列，那么查询线程拿到的结果跟表结构对不上，肯定是不行的。

因此，在MySQL 5.5版本中引入了MDL，**当对一个表做增删改查操作的时候，加MDL读锁；当要对表做结构变更操作的时候，加MDL写锁**。

- 读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。
- 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。

虽然MDL锁是系统默认会加的，但却是你不能忽略的一个机制。比如下面这个例子，我经常看到有人掉到这个坑里：给一个小表加个字段，导致整个库挂了。

![image](E:\研究生学习\Work\技术笔记\MySQL锁.assets\2019-03-24-135455.png)

* 假设session A先启动，这时候会对表t加一个MDL读锁。由于session B需要的也是MDL读锁，因此可以正常执行。
* 之后session C会被blocked，是因为session A的MDL读锁还没有释放，而session C需要MDL写锁，因此只能被阻塞。
* 如果只有session C自己被阻塞还没什么关系，但是之后所有要在表t**上新申请MDL读锁的请求也会被session C阻塞**。**前面我们说了，所有对表的增删改查操作都需要先申请MDL读锁，就都被锁住，等于这个表现在完全不可读写了。**
* 如果某个表上的查询语句频繁，而且客户端有重试机制，也就是说超时后会再起一个新session再请求的话，这个库的线程很快就会爆满。

你现在应该知道了，事务中的MDL锁，**在语句执行开始时申请，但是语句结束后并不会马上释放，而会等到整个事务提交后再释放**。

**基于上面的分析，我们来讨论一个问题，如何安全地给小表加字段**？

首先我们要解决长事务，事务不提交，就会一直占着MDL锁。在MySQL的information_schema 库的 innodb_trx 表中，你可以查到当前执行中的事务。如果你要做DDL变更的表刚好有长事务在执行，要考虑先暂停DDL，或者kill掉这个长事务。

但考虑一下这个场景。如果你要变更的表是一个热点表，虽然数据量不大，但是上面的请求很频繁，而你不得不加个字段，你该怎么做呢？

这时候kill可能未必管用，因为新的请求马上就来了。比较理想的机制是，**在alter table语句里面设定等待时间，如果在这个指定的等待时间里面能够拿到MDL写锁最好，拿不到也不要阻塞后面的业务语句**，先放弃。之后开发人员或者DBA再通过重试命令重复这个过程。

MariaDB已经合并了AliSQL的这个功能，所以这两个开源分支目前都支持DDL NOWAIT/WAIT n这个语法。

```
ALTER TABLE tbl_name NOWAIT add column ...
ALTER TABLE tbl_name WAIT N add column ... 
```



## 5.3.行锁

### 5.3.1.从两阶段锁说起

我先给你举个例子。在下面的操作序列中，事务B的update语句执行时会是什么现象呢？假设字段id是表t的主键。

![image](E:\研究生学习\Work\技术笔记\MySQL锁.assets\2019-03-24-142722.png)

这个问题的结论取决于事务A在执行完两条update语句后，持有哪些锁，以及在什么时候释放。你可以验证一下：实际上事务B的update语句会被阻塞，直到事务A执行commit之后，事务B才能继续执行。

知道了这个答案，你**一定知道了事务A持有的两个记录的行锁，都是在commit的时候才释放的**。

也就是说，在InnoDB事务中，**行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议**。

知道了这个设定，对我们使用事务有什么帮助呢？那就是，**如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放**。我给你举个例子。

假设你负责实现一个电影票在线交易业务，顾客A要在影院B购买电影票。我们简化一点，这个业务需要涉及到以下操作：

1. 从顾客A账户余额中扣除电影票价；
2. 给影院B的账户余额增加这张电影票价；
3. 记录一条交易日志。

也就是说，要完成这个交易，我们需要update两条记录，并insert一条记录。当然，为了保证交易的原子性，我们要把这三个操作放在一个事务中。那么，你会怎样安排这三个语句在事务中的顺序呢？

试想如果同时有另外一个顾客C要在影院B买票，那么这两个事务冲突的部分就是语句2了。因为它们要更新同一个影院账户的余额，需要修改同一行数据。

根据两阶段锁协议，不论你怎样安排语句顺序，所有的操作需要的行锁都是在事务提交的时候才释放的。所以，如果你把语句2安排在最后，比如按照3、1、2这样的顺序，那么影院账户余额这一行的锁时间就最少。这就最大程度地减少了事务之间的锁等待，提升了并发度。

好了，现在由于你的正确设计，影院余额这一行的行锁在一个事务中不会停留很长时间。但是，这并没有完全解决你的困扰。

如果这个影院做活动，可以低价预售一年内所有的电影票，而且这个活动只做一天。于是在活动时间开始的时候，你的MySQL就挂了。你登上服务器一看，CPU消耗接近100%，但整个数据库每秒就执行不到100个事务。这是什么原因呢？

这里，我就要说到死锁和死锁检测了。

### 5.3.2.死锁和死锁检测

> 死锁的四个条件：互斥，请求保持，不可剥夺，循环等待

当并发系统中不同线程出现循环资源依赖，涉及的线程都在等待别的线程释放资源时，就会导致这几个线程都进入无限等待的状态，称为死锁。这里我用数据库中的行锁举个例子。

![img](E:\研究生学习\Work\技术笔记\MySQL锁.assets\1245814-20190512223548013-342840533.png)

这时候，事务A在等待事务B释放id=2的行锁，而事务B在等待事务A释放id=1的行锁。 事务A和事务B在互相等待对方的资源释放，就是进入了死锁状态。当出现死锁以后，有两种策略：

- **一种策略是，直接进入等待，直到超时**。这个超时时间可以通过参数innodb_lock_wait_timeout来设置。
- **另一种策略是，发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。**将参数innodb_deadlock_detect设置为on，表示开启这个逻辑。

在InnoDB中，innodb_lock_wait_timeout的默认值是50s，意味着如果采用第一个策略，当出现死锁以后，第一个被锁住的线程要过50s才会超时退出，然后其他线程才有可能继续执行。对于在线服务来说，这个等待时间往往是无法接受的。

但是，我们又不可能直接把这个时间设置成一个很小的值，比如1s。这样当出现死锁的时候，确实很快就可以解开，但如果不是死锁，而是简单的锁等待呢？所以，超时时间设置太短的话，会出现很多误伤。

所以，正常情况下我们还是要采用第二种策略，即：主动死锁检测，而且innodb_deadlock_detect的默认值本身就是on。主动死锁检测在发生死锁的时候，是能够快速发现并进行处理的，但是它也是有额外负担的。

你可以想象一下这个过程：**每当一个事务被锁的时候，就要看看它所依赖的线程有没有被别人锁住，如此循环，最后判断是否出现了循环等待，也就是死锁**。

#### 如何检测死锁？

####  wait-for graph原理

 我们怎么知道上图中四辆车是死锁的？他们相互等待对方的资源，而且形成环路！我们将每辆车看为一个节点，当节点1需要等待节点2的资源时，就生成一条有向边指向节点2，最后形成一个有向图。我们只要检测这个有向图是否出现环路即可，出现环路就是死锁！这就是wait-for graph算法。

<img src="E:\研究生学习\Work\技术笔记\MySQL锁.assets\2efd7fac0ff4acd2d2c17f0496926181bfa15f9b.png" alt="img" style="zoom:50%;" />

　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　

 innodb将各个事务看为一个个节点，资源就是各个事务占用的锁，当事务1需要等待事务2的锁时，就生成一条有向边从1指向2，最后行成一个有向图。

那如果是我们上面说到的所有事务都要更新同一行的场景呢？

每个新来的被堵住的线程，都要判断会不会由于自己的加入导致了死锁，这是一个时间复杂度是O(n)的操作。假设有1000个并发线程要同时更新同一行，那么死锁检测操作就是100万这个量级的。虽然最终检测的结果是没有死锁，但是这期间要消耗大量的CPU资源。因此，你就会看到CPU利用率很高，但是每秒却执行不了几个事务。

根据上面的分析，我们来讨论一下，怎么解决由这种热点行更新导致的性能问题呢？问题的症结在于，**死锁检测要耗费大量的CPU资源。**

**一种头痛医头的方法，就是如果你能确保这个业务一定不会出现死锁，可以临时把死锁检测关掉**。但是这种操作本身带有一定的风险，因为业务设计的时候一般不会把死锁当做一个严重错误，毕竟出现死锁了，就回滚，然后通过业务重试一般就没问题了，这是业务无损的。而关掉死锁检测意味着可能会出现大量的超时，这是业务有损的。

**另一个思路是控制并发度**。根据上面的分析，你会发现如果并发能够控制住，比如同一行同时最多只有10个线程在更新，那么死锁检测的成本很低，就不会出现这个问题。一个直接的想法就是，在客户端做并发控制。但是，你会很快发现这个方法不太可行，因为客户端很多。我见过一个应用，有600个客户端，这样即使每个客户端控制到只有5个并发线程，汇总到数据库服务端以后，峰值并发数也可能要达到3000。

因此，这个并发控制要做在数据库服务端。如果你有中间件，可以考虑在中间件实现；如果你的团队有能修改MySQL源码的人，也可以做在MySQL里面。**基本思路就是，对于相同行的更新，在进入引擎之前排队。这样在InnoDB内部就不会有大量的死锁检测工作了**。

可能你会问，如果团队里暂时没有数据库方面的专家，不能实现这样的方案，能不能从设计上优化这个问题呢？

**你可以考虑通过将一行改成逻辑上的多行来减少锁冲突。还是以影院账户为例，可以考虑放在多条记录上，比如10个记录，影院的账户总额等于这10个记录的值的总和**。这样每次要给影院账户加金额的时候，随机选其中一条记录来加。这样每次冲突概率变成原来的1/10，可以减少锁等待个数，也就减少了死锁检测的CPU消耗。

这个方案看上去是无损的，但其实这类方案需要根据业务逻辑做详细设计。如果账户余额可能会减少，比如退票逻辑，那么这时候就需要考虑当一部分行记录变成0的时候，代码要有特殊处理。



## 悲观锁和乐观锁

mysql的并发操作时而引起的数据的不一致性（数据冲突）：

**丢失更新**：两个用户（或以上）对同一个数据对象操作引起的数据丢失。

　　　　解决方案：1.**悲观锁**，假设丢失更新一定存在；这是数据库的一种机制；比如全局锁、表锁、行锁

　　　　　　　　　2.**乐观锁**，假设丢失更新不一定发生。**update时候存在版本，更新时候按版本号进行更新**。

### 一、乐观锁

**乐观锁不是数据库自带的，需要我们自己去实现**。乐观锁是指操作数据库时(更新操作)，想法很乐观，认为这次的操作不会导致冲突，在操作数据时，并不进行任何其他的特殊处理（也就是不加锁），而在进行更新后，再去判断是否有冲突了。

通常实现是这样的**：**

* **在表中的数据进行操作时(更新)，先给数据表加一个版本(version)字段**，
* **每操作一次，将那条记录的版本号加1**。
* 在再更新操作前查询出那条记录，获取出version字段。
* **如果要对那条记录进行操作(更新)，则先判断此刻version的值是否与刚刚查询出来时的version的值相等**
* 如果相等，**则说明这段期间，没有其他程序对其进行操作，则可以执行更新，将version字段的值加1**；如果更新时发现此刻的**version值与刚刚获取出来的version的值不相等，则说明这段期间已经有其他程序对其进行操作了，则不进行更新操作**。

> 单操作包括3步骤：
>
> 1.查询出商品信息,包含version信息  select (status,status,**version**) from t_goods where id=#{id}
>
> 2.根据商品信息生成订单
>
> 3.修改商品status为2
>
> update t_goods  set status=2, version=version+1 where id=#{id} and version=#{version};



## 总结

* **全局锁**：常用场景是数据库备份的时候

* ##### 只读与全局锁的区别

  * 一是，在有些系统中，readonly的值会被用来做其他逻辑，比如用来判断一个库是主库还是备库。因此，**修改global变量的方式影响面更大**，我不建议你使用。

  * 二是，在异常处理机制上有差异。
    * 出现异常后锁会自动释放：**如果执行FTWRL命令之后由于客户端发生异常断开，那么MySQL会自动释放这个全局锁，整个库回到可以正常更新的状态。**
    * 出现异常后数据库会一直保持只读状态：**而将整个库设置为readonly之后，如果客户端发生异常，则数据库就会一直保持readonly状态，这样会导致整个库长时间处于不可写状态，风险较高**。

* **利用可重复读来实现“全局锁”**: 在导数据之前就会启动一个事务，来确保拿到一致性视图。而由于MVCC的支持，这个过程中数据是可以正常更新的

* **表锁**：显示加锁。除了会限制别的线程的读写外，也限定了本线程的操作对象。本线程只能访问加锁的这个表

* **元数据锁**：自动加锁。当对一个表做增删改查操作的时候，加MDL读锁；当要对表做结构变更操作的时候，加MDL写锁。

  * 读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。

  * 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。

  * **问题：**

    * 这个锁在语句执行开始时申请，但是语句结束后并不会马上释放，而会等到整个事务提交后再释放

    * 如果前面有一个事务获取到读锁，接下来一个事务想要获取写锁，就必须因为等待这个表的读锁而阻塞，而后的所有读锁写锁都会被阻塞。如果请求查询很频繁，再加上客户端有重试机制，可能线程就会爆满。

      >  读锁为啥也会被阻塞，不是说读锁之间不互斥吗？想想事务的一致性读，如果我这个事务在一个事务的更新之前读取数据，那么我读的数据就是老数据，就会又问题

    * 解决方法：在 DDL 之前解决长事务；DDL操作设置等待时间，在等待时间内获取到锁最好，获取不到就先放弃。之后再执行这个命令（再不影响后后续查的情况下）

* **行锁**：行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议；在设计中容易把冲突的语句往后放。

* **死锁**：死锁监测：有向闭环图，把事务看成结点，把资源之间的等待依赖关系看成边，如果有闭环就代表发生死锁。

  * 出现死锁的解决方案：等待超时回滚、死锁监测回滚其中的一个事务
  * **如何避免死锁监测造成的损耗**？多个并发执行的时候对大量并发线程进行死锁监测无疑是消耗CPU资源的。
    * 解决方案：服务端加入中间件（消息队列），限流作用、分表，把数据放到分开多张表中进行修改，减少竞争。但得采取一定的策略保证数据的一致性。