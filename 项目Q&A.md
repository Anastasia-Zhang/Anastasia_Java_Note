# 实习项目

我叫张心宇，目前就读于南京大学，是软件工程专业的研究生。我在2019年的4月-8月在浪潮有过一次实习经历，主要做医疗商城的后端开发工作，使用的是SpringBoot、MyBatis框架。研究生期间主要做电子邮件的文本挖掘，我的主要方向是事件挖掘。我喜欢编程热爱新技术，自己学习过大数据的基本知识，研究过基于Ptri网的日志模型生成算法；擅长沟通，在本科和实习时经常和团队一起做项目，养成了良好的团队沟通能力

## 研究生项目

![image-20200317155759307](E:\研究生学习\Work\技术笔记\项目Q&A.assets\image-20200317155759307.png)

* **项目定位**：希望能做出一个电子邮件分析的系统，用户通过登录自己的邮箱号，可以看到给自己邮件的分析结果，主要包括：邮件主题提取、用户关系网络、邮件事件提取三个基本功能。我主要负责邮件的事件提取
* 数据：自己邮件 + 清华开源邮件 + 其他文本
* **目前的问题**：数据集数量不够，现有的数据集质量不高，
* **解决方案：**找一些类似的文本去做技术上的基础
* **现在的工作：**阅读论文，实现相关的功能
* 数据来源：github 上有很多自然语言处理的相关 的开源的数据集 包括百度知识的、新闻类数据集等



## 实习项目

#### 简单介绍一下你的实习的项目：

这个项目是医疗销售平台的开发，用的是SpringBoot 和 MyBatis 框架，原来 JSP+Spring ，现在前后端分离，后端整个重构+升级。

我在整项目中主要参与负责了四项内容：商品模块的重构、订单功能的开发、数据库的适配及优化还有前端页面的编写。其中我参与较多的是订单功能的开发。数据库的适配工作。

### 商品模块的重构

![image-20200302212828992](E:\研究生学习\Work\技术笔记\项目Q&A.assets\image-20200302212828992.png)

参与商品模块的重构：梳理商品模块的业务逻辑，优化数据库表结构的设计；完成保存商品快照功能；优化相关业务逻辑代码，提高了代码的可复用性。

#### Q1 如何做的商品价格数据库的表结构设计（如何优化数据库表结构设计）？

* 删掉实际线上项目没有用到的列（上线的功能用到的不多的列）二次开发项目。**如何确定这个字段没有用到？不仅是业务需求没有用到、在确定重构方案之前也会把相关的代码逻辑走一遍进而确定这个字段没有被用到**
* （测试、停服、在所有的业务代码里都发现这个字段没有用到）
* 商品价格都保存到eb_item_sku中，主要的字段有price、market_price、sku_whse_qty、status、sku_qty_warn、sku_sale_qty。同时，根据不同sku数据的price，计算出最大价格和最小价格，保存到eb_item_iteminfo表的max_price和min_price字段。
* 原先商品的 iteminfo 表中的商品价格，有 price min_price max_price 三个字段，price 字段 和 min_price 字段是一样的，具体设计到也业务场景也多，主要是作为商品列表的展示价格，因此把price字段去掉了
* 商品的各个与销售有关的价格都保存到了 sku 表中
* 索引

#### Q2 线上的数据如何做迁移？

数据库的重构一般只改动了不常用的列比较多，原先的数据改动比较少。

具体的数据如何迁移我们没有涉及到。一般为 停服务 备份 迁移

不过我认为的可行方法是：修改好后我们组用测试环境测试数据测试没有什么问题了，还要进行一个线上环境的测试，尝试用线上的数据测试没问题了，或者将更改的内容同步到正是库，或者新建一个库，库的表和字段是和新修改的数据库一样，把正式库的数据迁移到这个新建的库。（一般在晚上业务量不大的时候，这个时候上游的服务是要停掉的）

#### Q1.2  数据库字段删除需要考虑的风险

#### Q3 商品库存，为啥不单独搞一个库存？

 eb_item_sku表保存每个规格的销量、库存，同时eb_item_iteminfo表保存商品的总销量总库存（每个规格的总和）。在修改商品的库存和销量时，要同时修改eb_item_sku和eb_item_iteminfo。

？？？单独的库存（本宝宝不知道）

#### Q4 商品快照功能？

面向B端，商家修改商品信息后可以追溯到自己修改的商品信息

追溯商品更改信息，商品更改的时候需要先下架，以防更改商品的时候有人下单。**下架之后用户突然搜不到商品了？？ESs**

商品基本信息（eb_item_iteminfo中的数据）保存到eb_item_snap；eb_item_snap的time_stamp是保存快照时的时间戳，和item_id一起，用来标记商品快照的版本。下架修改商品信息的时候，不需要保存新商品快照功能，

> * 更新商品基本信息：更新eb_item_iteminfo表。
>
> * 更新商品价格：将eb_item_sku中旧的数据都删除，然后insert新的sku数据。
>
> * 更新商品详情：更新eb_item_itemdetail_info表。
>
> * 更新图片：更新图片，先查询商品的所有图片，然后循环处理，对新添加的图片执行insertItemImg操作，不需要的图片执行deleteItemImg操作，要修改图片执行updateItemImg操作。
>
> （1）  商品发布后不允许修改类目。
>
> （2）  如果商品已经上架，需要修改商品信息，需要先下架，以防在修改时有买家下单。
>
> （3）  可以单独提供一个补货的接口，只修改商品的库存，不需要下架和审核。
>
> （4）  提供修改库存和销量的接口，用于有订单创建的时候，库存减，销量加。
>
> （5）  如果后续运营发现，商家修改价格比较频繁，可以提供一个修改价格的接口，修改前先下架，修改完之后自动上架。

**商品快照**

- 当商品下架时，需要保存商品快照，以便追溯商品修改记录。

- 商品基本信息（eb_item_iteminfo中的数据）保存到eb_item_snap，具体字段见1.5节；eb_item_snap的time_stamp是保存快照时的时间戳，和item_id一起，用来标记商品快照的版本。

- sku数据（properties、props_name、price、market_price、sort_order、outer_sku_id等，详见1.6节），保存到eb_item_sku_snap。eb_item_sku_snap的time_stamp和eb_item_snap的time_stamp值相同。

- 商品详情（detail_desc、mob_detail_desc）以键值对的方式存放到eb_item_snap的detail字段。




### 订单模块的重构

![image-20200302222043307](E:\研究生学习\Work\技术笔记\项目Q&A.assets\image-20200302222043307.png)

#### Q1：俩接口啥区别

先后顺序关系：（淘宝）订单确认是提交商品信息确认商品信息、订单提交是填写收货地址，真正的提交订单入库

#### Q3：商品优惠券

优惠券批价？优惠券直接满减。直接从总价减优惠券

#### Q4：如何防止重复下单

* 订单确认之后接口会生成随机数，返回给前端
* 订单提交的接口的参数会带有之前订单确认的随机数，在执行订单提交之前会从redis里先检测有没有这个随机数
* 如果有的话就是重复提交订单，没有的话就把这个随机数存入redis，处理完订单提交流程之后再把redis删除

#### Q5: kafka 消息回溯

消费过的消息再消费一遍？？可以把消费过的消息的信息存入redis里面

kafka 会把消费过的消息的offset会提交给 zookpeer 或者 broker ，消费者下次消费消息就会从当前的offset中读取最大的继续消费。

#### Q2: Kafka 如何保证信息不丢失不重复

**发送端：同步发送等待所有的副本都同步成功之后返回成功，消费端关掉自动提交位移，消息处理完之后手动提交位移**

* **消费端**：如果在消息处理完成前就提交了offset，那么就有可能造成数据的丢失。由于Kafka consumer默认是自动提交位移的，所以在后台提交位移前一定要保证消息被正常处理了，因此不建议采用很重的处理逻辑，如果处理耗时很长，则建议把逻辑放到另一个线程中去做。为了避免数据丢失，现给出两点建议：
  enable.auto.commit=false  关闭自动提交位移
  在消息被完整处理之后再手动提交位移
  ![image-20200303110519056](E:\研究生学习\Work\技术笔记\项目Q&A.assets\image-20200303110519056.png)

* **生产者**

  * **kafka同步生产者**：这个生产者写一条消息的时候，它就立马发送到某个分区去。follower还需要从leader拉取消息到本地，follower再向leader发送确认，leader再向客户端发送确认。由于这一套流程之后，客户端才能得到确认，所以很慢。

    同步发送消息后会产生一个发送消息的记录

    ![image-20200303112158803](E:\研究生学习\Work\技术笔记\项目Q&A.assets\image-20200303112158803.png)

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



**数据重复消费**的情况，如果处理
（1）**去重：将消息的唯一标识保存到外部介质中，每次消费处理时判断是否处理过；**
（2）不管：大数据场景中，报表系统或者日志信息丢失几条都无所谓，不会影响最终的统计分析结

### 数据库的适配

![image-20200303121450230](E:\研究生学习\Work\技术笔记\项目Q&A.assets\image-20200303121450230.png)

#### Q1 有哪些SQL问题？

主要是mysql 的内置函数在其他数据库中不支持

* SYSDATE mysql 是 SYSDATE()  其他是 SYSDATE 

* DATE_FORMAT() (函数用于以不同的格式显示日期/时间数据) 其他数据库改为 to_char()

* GROUP_CONCAT() 改为WM_CONCAT() group_contact 把分组之后同一组的列值相链接起来

  `group_concat([DISTINCT] 要连接的字段 [Order BY ASC/DESC 排序字段] [Separator '分隔符'])`

![image-20200303133749183](E:\研究生学习\Work\技术笔记\项目Q&A.assets\image-20200303133749183.png)

* order by 的字段必须在 select 当中

```sql
1、SYSDATE
MYSQL    SYSDATE()
其他     SYSDATE


imall:
<select id="getBaseVend" parameterType="Map" databaseId="mysql">
	SELECT
		VEND_APPLY.APPLY_ID,
		VEND.VEND_ID,
		VEND.COM_ID,
		COM_NAME,
		to_char(COM_CRT_TIME,'YY-mm-dd') AS COM_CRT_TIME
	FROM
		EB_BASE_VEND VEND  	
	WHERE 
		BEGIN_DATE &lt;SYSDATE()			
</select>
<select id="getBaseVend" parameterType="Map" databaseId="AQKK">
		SELECT
		VEND_APPLY.APPLY_ID,
		VEND.VEND_ID,
		VEND.COM_ID,
		COM_NAME,
		to_char(COM_CRT_TIME,'YY-mm-dd') AS COM_CRT_TIME
	FROM
		EB_BASE_VEND VEND  
	WHERE
		BEGIN_DATE &lt;SYSDATE	  	
</select>


2、DATE_FORMAT()
imall:
<select id="getBaseVend" parameterType="Map" databaseId="mysql">
	DATE_FORMAT(BONUS.CREATE_TIME, '%Y-%m-%d %H:%i:%s')
		AS CREATE_TIME,			
</select>
<select id="getBaseVend" parameterType="Map" databaseId="AQKK">
		to_char(BONUS.CREATE_TIME, '%Y-%m-%d %H:%i:%s')
		AS CREATE_TIME, 	
</select>

4、GROUP_CONCAT()
将GROUP_CONCAT()改为WM_CONCAT()

imall:
<select id="getPic" parameterType="Map" databaseId="mysql">
	SELECT
		GROUP_CONCAT(T.PATH) PICPATH,
		T.PIC_ID
	FROM
		EB_ITEM_PIC_UPLOAD T
	GROUP BY
		T.PIC_ID			
</select>
<select id="getPic" parameterType="Map" databaseId="AQKK">
	SELECT
		WM_CONCAT(T.PATH) PICPATH,
		T.PIC_ID
	FROM
		EB_ITEM_PIC_UPLOAD T
	GROUP BY
		T.PIC_ID  	
</select>

5、SELECT DISTINCT ........ ORDER BY ..
EX:
SELECT DISTINCT
	CAT_NAME,
	PARENT_CID,
	IS_LEAF,
	CAT_TYPE,	
	STATUS,
	NOTE,
	COM_ID,
	CAT_PATH
FROM
	EB_ITEM_ITEMCAT
WHERE
	STATUS = '1'
AND PARENT_CID = '-1'
AND COM_ID = 'hdds'
ORDER BY
	SORT_ORDER
改为:
SELECT DISTINCT
	CAT_NAME,
	PARENT_CID,
	IS_LEAF,
	CAT_TYPE,
	SORT_ORDER,
	STATUS,
	NOTE,
	COM_ID,
	CAT_PATH
FROM
	EB_ITEM_ITEMCAT
WHERE
	STATUS = '1'
AND PARENT_CID = '-1'
AND COM_ID = 'hdds'
ORDER BY
	SORT_ORDER

```

#### Q2 项目中"重构查询方式，将多表连接查询分解 成单表查询，在业务代码中关联，使缓存更高效，提高了数据的可分离性", 能介绍一下吗?  如果不关联, 如何处理Join查询分页问题呢? 

比如查询订单详情，以前的查询时基于订单表基本表、订单商品表、订单收货地址表三个表连接查询而来，现在把这一个SQL语句改成三个SQL语句，在Service分别调用这三种查询并合成一个结果，这样拆分后的SQL语句还可以被其他的功能使用。拆分的时候没有遇到分页的问题。

这样有助做的分库和分表做，分库分表使用join链接多表会降低性能。分解查询时也可以减少锁的竞争。这样也使得单个的sql更好的进行维护。还有一个出发点就是如过某个表很少改变，那么查询的时候直接走缓存就可以，使缓存的利用率更高。
当时我做的时候没有遇到需要分页的Sql语句，像大的分页问题如商品页搜索和商家订单的搜索还是从ES查的。具体的分页问题如果在业务代码中合并数据不复杂的话在应用层分页，数据量不是很大的话，可以一次返回全部数据在前端分页。

**使用场景**：

1. 分页查询还是用join

2. 简单分页，不涉及到分页的，如查询某一某个详细信息，可以单表查询业务逻辑组合

3. 缓存效率： 
   数据库是存在缓存机制的，当一条SQL执行之后，再次执行相同的SQL，数据库会把缓存的结果返回出去，而不会重新查询数据库。单查询的可重用性较高，所以缓存效率相较之联合查询会更高。 使用第三方redis等缓存，key（组合更少更单一）和value使用也相应减少

4. 锁竞争 
       熟悉多线程的同学相信对锁机制一定不陌生。为了保证数据库的数据同步，在数据库进行读写时，数据库会用锁机制，限制其他连接对其操作。读写越快，数据库的并发性越高。由于联合查询查询速度比单个查询要慢很多，这样联合查询会增加锁的竞争关系，所以用单查询会更好些。

5. 查询结果有效使用率 
       相较于联合查询，单查询的查询结果有效利用率要高很多，也就是说联合查询会浪费一些时间在查询无用的数据上。例如后台管理的列表界面，通常都会分页显示，关联查询的结果集，只有当前页的数据被使用，其他都是无用的，但数据库需要消耗额外资源得到全部结果集，再从中得到当前页数据。 

   单表查询结果放redis等缓存中使用效率更高 ？？？

6. 大数量的表推荐使用单表，小数据量的表推荐使用组合查询

7. 单表SQL简单容易理解，而且做分库等改动较小

   

#### Q3 为什么不用ES, 直接从ES查? ES用来干什么了, 怎么往ES同步的数据?

ES主要用到的是商铺的订单和商品查询、用户的商品查询。

**ES数据同步**：(在实习的项目中如何进行的ES同步不太了解，但是我理解的可以进行以下的数据同步）（1）同步双写：数据修改到数据库中，同时写到ES中。（可能有双写失败风险，ES和mysql业务逻辑强耦合）（2）异步双写：从数据写到数据库中后，再使用MQ （3）使用多线程异步调用es的保存操作。

### 前端页面

![image-20200303161127783](E:\研究生学习\Work\技术笔记\项目Q&A.assets\image-20200303161127783.png)

#### Q 如何配置的 Nginx

* 主要用于商品的静态页面 
* 反向代理

```shell

#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;


    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;
		


        location / {
            #root   html;
			root E:\opt\apps\www;
            index  index.html index.htm;
        }
		
		location ^~ /ecweb {
            proxy_pass        http://localhost:18081/ecweb;
			proxy_set_header HOST $http_host; 
            proxy_set_header X-Real-IP $remote_addr; 
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
			location ^~ /ecwebdev {
            proxy_pass        http://localhost:38080/ecwebdev;
			proxy_set_header HOST $host; 
			proxy_set_header X-Real-IP $remote_addr; 
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
        location ^~ /ecv6 {
            proxy_pass        http://localhost:8083/ecv6;
			proxy_set_header   Host $http_host;			
			#proxy_set_header HOST $host; 
			proxy_set_header X-Real-IP $remote_addr; 
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;	
        }
        location ^~ /bspm/ {
            proxy_pass        http://localhost:8380/bspm/;
        }
        location ^~ /admin/ {
            proxy_pass        http://localhost:8380/admin/;
        }

			location ^~ /healthmall {
            proxy_pass        http://localhost:8080/healthmall;
			proxy_set_header HOST $host; 
			proxy_set_header X-Real-IP $remote_addr; 
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			client_max_body_size 20M;
        }
		
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}

```



### Github上冲突问题

* 我提交的时候我本地代码的版本是要比当前分支上的最新版本还要老的时候，并且我要提交的代码和远程库上最新版本的代码不一样的时候，这个时候提交代码会有冲突



### 你平时在项目中用到了什么设计模式

**策略模式：**策略模式的主要目的是将算法的定义与使用分开，将算法的定义放在专门的策略类中，每一个策略类封装了一种实现算法，由环境类（content）负责决定使用不同的策略。策略模式将要实现的策略定义成接口，具体的策略类实现策略接口；Content类与具体的策略类组合，来实现根据情况下使用不同策略

我使用基于SpringIOC 实现策略模式：Spring提供了根据类型注入bean的特性：如果依赖注入的是集合类或者Map类型的，容器将自动装配所有与声明值类型匹配的bean，对于Map ，key将解析为相应的bean名称，value为对象实现bean实例。

1. 定义一个商品的活动类接口当做策略接口；定义两个接口继承商品活动接口当做具体的策略（如商品折扣活动，商品抢购活动，商品组团活动等）；这几个具体的策略有各自的实现类 Service。
2. 具体的实现类Service在@Service注解上指定value值  如 `@Service("discountActivityService")`
3. 定义活动类型枚举，封装各种类型的活动，并设置活动key、活动描述、活动value值，即对应的Service实现类的value（@service注解上的指定的value值）
4. 在 Controller 里根据不同的情况调用不同的Service类，也就是不同的策略。此时Controller相当于 Content 类。
5. 在Controller 类里注入 Map，key 代表具体service实现类的value值，value 代表活动接口
6. 这样我们可以根据不同的活动类型，去活动枚举类中得到对应活动的具体 service 的value值
7. 将活动的value值当做map的key，去 map 中根据key 就可以获得具体的活动实例 

**装饰者模式：**在不改变原有对象的基础之上，将功能附加到对象上。他通过组合被装饰者对象，可以对被装饰者对象的“行为”进行扩展。

装饰者模式定义了一个组件（被装饰者）作为接口，装饰者类继承组件接口，具体的装饰者类实现装饰者类的接口；装饰者通过组合组件类，来动态的扩展组件的行为。

在项目中我主要是使用的是包装器业务异常类来统一管理在项目运行时抛出的异常。业务首先定义一个异常类接口，包括异常的错误码和错误信息等。然后自定义一个业务异常类BusinessException和异常枚举EmBusinessException实现异常类接口。EmBusinessException中定义项目规定好的错误码和描述信息，BusinessException实现对枚举异常类EmBusinessException的包装：可以使用定义好的枚举类当做构造函数的参数来构造自定义业务异常类，也可以自定义错误码来构建业务异常。最终可以在业务代码中抛出使用自定义的业务异常类。