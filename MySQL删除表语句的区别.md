## delete drop truncate 的区别

- drop：删除内容和定义，释放空间。（表结构和数据一同删除）【drop语句将删除表的结构，被依赖的约束（constrain),触发器（trigger)索引（index);依赖于该表的存储过程/函数将被保留，但其状态会变为：invalid。】
- truncate：删除内容，释放空间，但不删除定义。（表结构还在，数据删除）【truncate table 权限默认授予表所有者、sysadmin 固定服务器角色成员、db_owner 和 db_ddladmin 固定数据库角色成员且不可转让。】
- delete：删除内容，不删除定义，也不释放空间。
- 三者的执行速度，一般来说：drop > truncate > delete


 

### 释放空间可以体现在：

通过delete删除的行数据是不释放空间的，如果表id是递增式的话，那么表数据的id就可能不是连续的；而通过truncate删除数据是释放空间的，如果表id是递增式的话，新增数据的id又是从头开始，而不是在已删数据的最大id值上递增。

 

### delete from user 与 truncate table user ，虽然同样是删除user表的数据，但却有很大的区别：

1、delete from user 在删除数据时，是一行数据一行数据地删除，每删除一行数据，就在事务日志中为删除的那行数据做一项记录，因此可对delete操作进行回滚（roll back）；
truncate table user 在删除数据时，是通过释放存储表数据所用的数据页来删除数据，并且只在事务日志中记录页的释放；
因此，truncate 的执行速度比 delete 快，且使用的系统和事务日志资源少（在表数据越多的情况下对比更明显）； 不过，在执行回滚命令时，delete 将被撤销，而 truncate 则不会别撤销。

2、delete 可以删除部分行数据， truncate 只能删除表中所有数据。

3、delete语句是数据库操作语言(dml)，这个操作会放到 rollback segement 中，事务提交之后才生效；如果有相应的触发器（trigger），执行的时候将被激活。
      truncate、drop 是数据库定义语言(ddl)，操作立即生效，原数据不放到 rollback segment 中，不能回滚，操作不激活 触发器（trigger）。

4、truncate 在删除表中的所有行后，表的结构及其列、约束、索引等保持不变。新行标识所用的计数值重置为该列的种子。如果想保留标识计数值，请改用delete。

5、truncate 将重新设置高水平线和所有的索引。在对整个表和索引进行完全浏览时，经过 truncate 操作后的表比Delete操作后的表要快得多。

6、当表被清空后表和表的索引讲重新设置成初始大小，而delete则不能。

7、truncate 不能清空父表。

8、truncate 不能用于参与了索引视图的表。

 

注：在什么情况下，使用 delete 而不使用 truncate ?

1、当只要删除表中部分数据时，使用 delete ，因为 truncate 只能清空表中所有数据；

2、当要删除数据的表中存在外键（foreign key）约束时，应使用 delete，不能使用 truncate，因为 truncate 操作不能激活 触发器（trigger）。
