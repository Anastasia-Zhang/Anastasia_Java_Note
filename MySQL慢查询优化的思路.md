### 一条查询的执行过程

![image](E:\研究生学习\Work\技术笔记\MySQL慢查询优化的思路.assets\687474703a2f2f636c7361612d6269672d646174612d6e6f7465732d313235323033323136392e636f7373682e6d7971636c6f75642e636f6d2f323031392d30312d32302d3033323231332e706e67.png)



### 查询过程中如何“照顾好用户的情绪”

* 使用索引
* 分页
* 重构数据库
* 优化查询的逻辑
* 数据库表的设计
* 提供进度条、预估截止时间、提供结果集大小
* 异步处理
* 查询频率较高使用缓存
* 预加载（表达式中预加载）
* 主从读写分离

### join 和 exists 

```mysql
SELECT DISTINCT film.film_id
FROM sakila.film
INNER JOIN sakila.film_actor USING(film_id);
```

```java
SELECT film_id
FROM sakila.film
WHERE EXISTS(
SELECT * FROM sakila.film_actor
WHERE film.film_id = film_actor.film_id);
```

* 他们在film.id 和 language.id 上做了索引，查询的时候都会用的到
* 对于distinct，用到了临时表，先把select film.film_id 的结果存入临时表，然后扫描film_actor,用索引查找并和join_buffer的内容做对比，符合join条件的，作为结果集返回。最后再做distinc操作
* 对于exist，他只负查询 select 是否在子查询。核心表是 film 表，子查询和外层表关心紧密，外层查询先对film表进行遍历，然后做 exisit 存在性的测试，如果非空，那么film_id 就是需要的结果，否则删除。他只需要找到符合条件的一条语句即可。

### left join 慢的原因

* 两个表对应连接字段的字符集排序规则不一致会导致连接表的字段不走索引

#### 原因

1. 首先t2 left join t1决定了t2是驱动表，这一步相当于执行了select * from t2 where t2.name = ‘dddd’，取出code字段的值，这里为’8a77a32a7e0825f7c8634226105c42e5’;（utf8mb4）

2. 然后拿t2查到的code的值根据join条件去t1里面查找，这一步就相当于执行了select * from t1 where t1.code = ‘8a77a32a7e0825f7c8634226105c42e5’ （utf8）;

3. 但是由于第（1）步里面t2表取出的code字段是utf8mb4字符集，而t1表里面的code是utf8字符集，这里需要做字符集转换，**字符集转换遵循由小到大的原则**，因为utf8mb4是utf8的超集，所以这里把utf8转换成utf8mb4，即把t1.code转换成utf8mb4字符集，转换了之后，由于t1.code上面的索引仍然是utf8字符集，所以这个索引就被执行计划忽略了，然后t1表只能选择全表扫描。更糟糕的是，如果t2筛选出来的记录不止1条，那么t1就会被全表扫描多次，性能之差可想而知。
4. **主表的字符集是从表字符集的超集，因此从表的字符集要转化为主表的字符集。这样的转化会使执行计划忽略从表的索引。**

#### 解决方案

将一方的索引转换成松散的一方，比如将t2表code的字符集转化成t1表的字符集。

**返过来可以吗？**t1的字符集转化为t2的字符集 （t1的字符集是t2字符集的超集）

如果t1表中的code包含多种类型的数据，而t2表中的code只有一种类型，当由t1转换为t2类型时，可能t1表中的数据有可能会丢失（精度损失）

#### 几种测试情况

```mysql
CREATE TABLE `table_a` (
 `id` int(10) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增ID',
 `code` varchar(20) NOT NULL COMMENT '编码',
 PRIMARY KEY (`id`),
 KEY `code` (`code`)
) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8mb4

CREATE TABLE `table_b` (
 `code` varchar(20) NOT NULL COMMENT '编码',
 `name` varchar(20) NOT NULL COMMENT '名称',
 KEY `code` (`code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```



```mysql
EXPLAIN SELECT * FROM `table_a` a LEFT JOIN table_b b ON a.code = b.code where a.code = '1001';
```

```mysql
EXPLAIN SELECT * FROM `table_a` a LEFT JOIN table_b b ON a.code = b.code
```

* b 表索引失效，原因如上

```mysql
EXPLAIN SELECT * FROM `table_a` a LEFT JOIN table_b b ON a.code = b.code where b.code = '1001';
```

*  a b 表都用了索引 因为先从a表取出数据，然后和满足条件的b表数据做比配，因为b表有where子句，因此索引会用到，先使用索引查询b表中满足条件的记录。可能再做字符转换

```mysql
EXPLAIN SELECT * FROM `table_b` b LEFT JOIN table_a a ON a.code = b.code where b.code = '1001';
```

* 都使用了索引，猜测不存在字符集转化

```mysql
EXPLAIN SELECT * FROM `table_b` b LEFT JOIN table_a a ON a.code = b.code
```

```mysql
EXPLAIN SELECT * FROM `table_b` b LEFT JOIN table_a a ON a.code = b.code where a.code = '1001'
```

* b表没有勇索引，因为b表无论咋样都是全表扫描，必然用不到索引

## 常见慢查询优化策略

### （1）数据库中设置SQL慢查询

方式一：

   修改配置文件  在 my.ini 增加几行:  主要是慢查询的定义时间（超过2秒就是慢查询），以及慢查询log日志记录（ slow_query_log）

![img](E:\研究生学习\Work\技术笔记\MySQL慢查询优化的思路.assets\20180921145213710.png)

方法二：通过MySQL数据库开启慢查询:

![img](E:\研究生学习\Work\技术笔记\MySQL慢查询优化的思路.assets\201809211454073.png)

### （2）分析慢查询日志         

​       直接分析mysql慢查询日志 ,利用explain关键字可以模拟优化器执行SQL查询语句，来分析sql慢查询语句

**EXPLAIN：显示SQL如何使用索引的执行计划。**

执行计划的参数：

* **table** 显示这一行的数据是关于哪张表的

* **type** 显示连接使用了何种类型。从最好到最差的连接类型为const、eq_reg、ref、range、indexhe和ALL

* **possible_keys** 显示可能应用在这张表中的索引。如果为空，没有可能的索引。可以为相关的域从WHERE语句中选择一个合适的语句 

* **key** 实际使用的索引。如果为NULL，则没有使用索引。很少的情况下，MYSQL会选择优化不足的索引。这种情况下，可以在SELECT语句中使用USE INDEX（indexname）来强制使用一个索引或者用IGNORE INDEX（indexname）来强制MYSQL忽略索引 

* **key_len** 使用的索引的长度。在不损失精确性的情况下，长度越短越好 

* **ref** 显示索引的哪一列被使用了，如果可能的话，是一个常数

* **rows** 扫描请求数据的行数 

* **Extra** 关于MYSQL如何解析查询的额外信息

### （3）常见的慢查询优化

####  （1）索引没起作用的情况

* 使用LIKE关键字的查询语句

  在使用LIKE关键字进行查询的查询语句中，如果匹配字符串的第一个字符为“%”，索引不会起作用。只有“%”不在第一个位置索引才会起作用。**索引的最左前缀匹配原则**

* 使用多列索引查询语句

   MySQL可以为多个字段创建索引。一个索引最多可以包括16个字段。对于多列索引，只有查询条件使用了这些字段中的第一个字段时，索引才会被使用。

####   （2）优化数据库结构

​        合理的数据库结构不仅可以使数据库占用更小的磁盘空间，而且能够使查询速度更快。数据库结构的设计，需要考虑数据冗余、查询和更新的速度、字段的数据类型是否合理等多方面的内容。

1. 将字段很多的表分解成多个表 

    对于字段比较多的表，如果有些字段的使用频率很低，可以将这些字段分离出来形成新表。因为当一个表的数据量很大时，会由于使用频率低的字段的存在而变慢。

2. 增加中间表

    对于需要经常联合查询的表，可以建立中间表以提高查询效率。通过建立中间表，把需要经常联合查询的数据插入到中间表中，然后将原来的联合查询改为对中间表的查询，以此来提高查询效率。

####  （3）分解关联查询

​    将一个大的查询分解为多个小查询是很有必要的。

  很多高性能的应用都会对关联查询进行分解，就是可以对每一个表进行一次单表查询，然后将查询结果在应用程序中进行关联，很多场景下这样会更高效，例如：       

```mysql
 SELECT * FROM tag 
        JOIN tag_post ON tag_id = tag.id
        JOIN post ON tag_post.post_id = post.id
        WHERE tag.tag = 'mysql';
```

```mysql
分解为：

SELECT * FROM tag WHERE tag = 'mysql';
SELECT * FROM tag_post WHERE tag_id = 1234;
SELECT * FROM post WHERE post.id in (123,456,567);
```
#### （4）优化LIMIT分页

* 在系统中需要分页的操作通常会使用limit加上偏移量的方法实现，同时加上合适的order by 子句。如果有对应的索引，通常效率会不错，否则MySQL需要做大量的文件排序操作。

*  一个非常令人头疼问题就是当偏移量非常大的时候，例如**可能是limit 10000,20这样的查询，这是mysql需要查询10020条然后只返回最后20条，前面的10000条记录都将被舍弃，这样的代价很高。**

*  优化此类查询的一个最简单的方法是尽可能的使用**索引覆盖扫描，而不是查询所有的列。然后根据需要做一次关联操作再返回所需的列**。对于偏移量很大的时候这样做的效率会得到很大提升。

  对于下面的查询：

  select id,title from collect limit 90000,10;

  该语句存在的最大问题在于limit M,N中偏移量M太大（我们暂不考虑筛选字段上要不要添加索引的影响），导致每次查询都要先从整个表中找到满足条件 的前M条记录，之后舍弃这M条记录并从第M+1条记录开始再依次找到N条满足条件的记录。如果表非常大，且筛选字段没有合适的索引，且M特别大那么这样的代价是非常高的。 试想，如我们下一次的查询能从前一次查询结束后标记的位置开始查找，找到满足条件的100条记录，并记下下一次查询应该开始的位置，以便于下一次查询能直接从该位置 开始，这样就不必每次查询都先从整个表中先找到满足条件的前M条记录，舍弃，在从M+1开始再找到100条满足条件的记录了。

#### 方法一：虑筛选字段（title）上加索引

​       title字段加索引  （此效率如何未加验证）

#### 方法二：先查询出主键id值

select id,title from collect where id>=(select id from collect order by id limit 90000,1) limit 10;

原理：先查询出90000条数据对应的主键id的值，然后直接通过该id的值直接查询该id后面的数据。

#### 方法三：“关延迟联”

如果这个表非常大，那么这个查询可以改写成如下的方式：

  Select news.id, news.description from news inner join (select id from news order by title limit 50000,5) as myNew using(id);

​    这里的“关延迟联”将大大提升查询的效率，**它让MySQL扫描尽可能少的页面**，获取需要的记录后再根据关联列回原表查询需要的所有列。这个技术也可以用在优化关联查询中的limit。**先不查询这多列，先查询有索引的列，**再去关联上其他的列

#### 方法四：建立复合索引 acct_id和create_time

​    select * from acct_trans_log WHERE  acct_id = 3095  order by create_time desc limit 0,10

 注意sql查询慢的原因都是:引起filesort

#### （5）分析具体的SQL语句

 **1、两个表选哪个为驱动表，表面是可以以数据量的大小作为依据，但是实际经验最好交给mysql查询优化器自己去判断。**

```mysql
 select * from a where id in (select id from b );     
```

​     对于这条sql语句它的执行计划其实并不是先查询出b表的所有id,然后再与a表的id进行比较。
mysql会把in子查询转换成exists相关子查询，所以它实际等同于这条sql语句：

```mysql
select * from a where exists(select * from b where b.id = a.id );
```

​    而exists相关子查询的执行原理是: 循环取出a表的每一条记录与b表进行比较，比较的条件是a.id=b.id . 看a表的每条记录的id是否在b表存在，如果存在就行返回a表的这条记录。

**exists查询有什么弊端？**
      **由exists执行原理可知，a表(外表)使用不了索引，必须全表扫描，因为是拿a表的数据到b表查**。而且必须得使用a表的数据到b表中查（外表到里表中），顺序是固定死的。**对外表的索引具有依赖性**

**如何优化？**
      建索引。但是由上面分析可知，要建索引只能在b表的id字段建，不能在a表的id上，mysql利用不上。

**这样优化够了吗？还差一些。**
      由于exists查询它的执行计划只能拿着a表的数据到b表查（外表到里表中），虽然可以在b表的id字段建索引来提高查询效率。
但是并不能反过来拿着b表的数据到a表查，exists子查询的查询顺序是固定死的。

**为什么要反过来？**
       因为首先可以肯定的是反过来的结果也是一样的。这样就又引出了一个更细致的疑问：在双方两个表的id字段上都建有索引时，到底是a表查b表的效率高，还是b表查a表的效率高？

该如何进一步优化？
       把查询修改成inner join连接查询：

```mysql
select * from a inner join b on a.id = b.id; 
```

**为什么不用left join 和 right join？**
       这时候表之间的连接的顺序就被固定住了，比如左连接就是必须先查左表全表扫描，然后一条一条的到另外表去查询，右连接同理。仍然不是最好的选择。

**为什么使用inner join就可以？**
       inner join中的两张表，如： a inner join b，但实际执行的顺序是跟写法的顺序没有半毛钱关系的，最终执行也可能会是b连接a，顺序不是固定死的。如果on条件字段有索引的情况下，同样可以使用上索引。

**那我们又怎么能知道a和b什么样的执行顺序效率更高？**
       你不知道，我也不知道。谁知道？mysql自己知道。让mysql自己去判断（查询优化器）。具体表的连接顺序和使用索引情况，mysql查询优化器会对每种情况做出成本评估，最终选择最优的那个做为执行计划。

​    在inner join的连接中,mysql会自己评估使用a表查b表的效率高还是b表查a表高，如果两个表都建有索引的情况下，mysql同样会评估使用a表条件字段上的索引效率高还是b表的。

利用explain字段查看执行时运用到的key（索引）
       而我们要做的就是：把两个表的连接条件的两个字段都各自建立上索引，然后explain 一下，查看执行计划，看mysql到底利用了哪个索引，最后再把没有使用索引的表的字段索引给去掉就行了
