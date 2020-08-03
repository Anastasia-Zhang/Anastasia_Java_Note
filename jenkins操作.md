```shell
[root@centos7 ~]# firewall-cmd --zone=public --add-port=80/tcp --permanent

查询端口号80 是否开启：

[root@centos7 ~]# firewall-cmd --query-port=80/tcp

重启防火墙：

[root@centos7 ~]# firewall-cmd --reload

查询有哪些端口是开启的:

[root@centos7 ~]# firewall-cmd --list-port
```

```
java -jar jenkins.war --httpPort=9000
```

mf1932246 12345

mysql启动异常报错

```shell
[root@iZm5e6dai3e435hwmgadxpZ mysql]# service mysql start 
Starting MySQL.200218 01:54:22 mysqld_safe error: log-error set to '/var/log/mariadb/mariadb.log', however file don't exists. Create writable for user 'mysql'.
The server quit without updating PID file (/var/lib/mysql/i[FAILED]3e435hwmgadxpZ.pid).

[root@iZm5e6dai3e435hwmgadxpZ mysql]# mkdir /var/log/mariadb
[root@iZm5e6dai3e435hwmgadxpZ mysql]# touch /var/log/mariadb/mariadb.log

[root@iZm5e6dai3e435hwmgadxpZ mysql]# chown -R mysql:mysql /var/log/mariadb
[root@iZm5e6dai3e435hwmgadxpZ mysql]# service mysql start
Starting MySQL.                                            [  OK  ]


```

error: 'Can't connect to local MySQL server through socket '/tmp/mysqld.sock' (2)'
Check that mysqld is running and that the socket: '/tmp/mysqld.sock' exists!

```shell
ln -sf /var/lib/mysql/mysql.sock /tmp/mysqld.sock
chmod -R 777 /var/lib/mysql/mysql.sock
```

```shell
To do so, start the server, then issue the following commands:

  ./bin/mysqladmin -u root password 'new-password'
  ./bin/mysqladmin -u root -h iZm5e6dai3e435hwmgadxpZ password 'new-password'

Alternatively you can run:

  ./bin/mysql_secure_installation

```

navate 访问数据库

```shell
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;
Query OK, 0 rows affected (0.00 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)
```

cc1734abfe05dd20fc915a32b441e48d8f1a8541

```shell
#!bin/bash
export BUILD_ID=dontkillme
echo "start.sh is running"
kill -9 $(ps -ef|grep injian-1.0-SNAPSHOT.jar|grep -v grep|awk '{print $2}')
cd /root/.jenkins/workspaces/injian/target
nohup java -jar injian-1.0-SNAPSHOT.jar >log.txt &

echo "success"

```

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDBCNW1AKTQ9Rsd+09gzI1NWiocvbNVWzNCEmQU0KdxxwhHWVfaSYhdAxiklQnqR7JiaWTX+FuzmSifKR4Sin5uBg2+D6zJtii72Wxqph6VxXqS0p2aEdjvGWQNCT69cDEAlk6mWRJl/Zn4SKEH6LIVBrDMyhUtTf/qf2zLXW7DjCfndxUsnjMIFMg65LSd26hzYmgyb26IuMakvkCKbRUcMTcAjYlqT62tb0zroGkWSEj/Jz2jw0FCEM4yDCXXS0Cl/zyAMFOQ4M3E+TlUiJWTvB7kyvI4jk0NXD8GCeQPZ9JfxtmN+hMN45yYOUu2G3lhFU471yQOa7RS0XnY6tfz 1003814986@qq.com

D:\软件安装包\sonar-scanner-cli-4.2.0.1873-windows\sonar-scanner-4.2.0.1873-windows

 **ff06d856b56f99063d1317f95310025af4400ef8** 



### 下载Zookpeer

https://mirrors.cnnic.cn/apache/zookeeper/

![image-20200408130618977](E:\研究生学习\Work\技术笔记\jenkins操作.assets\image-20200408130618977.png)

```shell
[root@iZm5e6dai3e435hwmgadxpZ ~]# mv zookeeper-3.4.14 /usr/local/zookeeper
[root@iZm5e6dai3e435hwmgadxpZ ~]# mkdir -p /var/lib/zookeeper
[root@iZm5e6dai3e435hwmgadxpZ ~]# cat > /usr/local/zookeeper/conf/zoo.cfg << EOF
> tickTime=2000
> dataDir=/var/lib/zookeeper
> clientPort=2181
> EOF
[root@iZm5e6dai3e435hwmgadxpZ ~]# /usr/local/zookeeper/bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

```

```
[root@iZm5e6dai3e435hwmgadxpZ ~]# telnet localhost 2181
-bash: telnet: command not found

```

咦？？？

安装 telent ！

```shell
yum install telnet-server
yum install telnet*
```

再试一次~

```shell
[root@iZm5e6dai3e435hwmgadxpZ ~]# telnet localhost 2181
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
srvr
Zookeeper version: 3.4.14-4c25d480e66aadd371de8bd2fd8da255ac140bcf, built on 03/06/2019 16:18 GMT
Latency min/avg/max: 0/0/0
Received: 1
Sent: 0
Connections: 1
Outstanding: 0
Zxid: 0x0
Mode: standalone
Node count: 4
Connection closed by foreign host.
[root@iZm5e6dai3e435hwmgadxpZ ~]# 

```

下载kafka https://www.apache.org/dyn/closer.cgi?path=/kafka/2.3.0/kafka_2.11-2.3.0.tgz

```shell
[root@iZm5e6dai3e435hwmgadxpZ ~]# tar -zxf kafka_2.11-2.3.0.tgz
[root@iZm5e6dai3e435hwmgadxpZ ~]# mv kafka_2.11-2.3.0 /usr/local/kafka
[root@iZm5e6dai3e435hwmgadxpZ ~]# mkdir /tmp/kafka-logs
[root@iZm5e6dai3e435hwmgadxpZ ~]# vim /usr/local/kafka/config/server.properties
[root@iZm5e6dai3e435hwmgadxpZ ~]# 
```

启动kafka内存分配不够

```shell
if [ $# -lt 1 ];
then
        echo "USAGE: $0 [-daemon] server.properties [--override property=value]*"
        exit 1
fi
base_dir=$(dirname $0)

if [ "x$KAFKA_LOG4J_OPTS" = "x" ]; then
    export KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:$base_dir/../config/log4j.properties"
fi

if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
    export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
fi

EXTRA_ARGS=${EXTRA_ARGS-'-name kafkaServer -loggc'}

COMMAND=$1
case $COMMAND in
  -daemon)
    EXTRA_ARGS="-daemon "$EXTRA_ARGS
    shift
    ;;
  *)

```

![image-20200408182249209](E:\研究生学习\Work\技术笔记\jenkins操作.assets\image-20200408182249209.png)

```shell
/usr/local/kafka/bin/kafka-server-start.sh -daemon /usr/local/kafka/config/server.properties
/usr/local/kafka/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

[root@iZm5e6dai3e435hwmgadxpZ bin]# /usr/local/kafka/bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic test
Topic:test	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: test	Partition: 0	Leader: 1	Replicas: 1	Isr: 1


[root@iZm5e6dai3e435hwmgadxpZ bin]# /usr/local/kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
>Test Massage 1
>Test Massage 2

/usr/local/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning

```

Scrum是⼀个基于团队进⾏行行复杂系统和产品开发的框架 。Scrum 框架包括 Scrum 团队及其相关的角色、事件、工件和规则。框架中的每个模块都有⼀个特定的目的，对 Scrum 的成功和使用都至关重要。自上世