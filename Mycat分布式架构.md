# Mycat分布式架构

### 一、入门概述

#### 	1、读写分离

![](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/image-20200621165433375.png)



#### 2、数据分片

垂直拆分(分库)、水平拆分(分表)、垂直拆分+水平拆分（分库分表）

![image-20200621171100263](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/image-20200621171100263.png)

#### 3、多数据源整合

### 二、原理

​	Mycat的原理中最重要的一一个动词是“拦截”，它拦截了用户发送过来的SQL语句,首先对SQL
​	语句做了一些特定的分析:如分片分析、路由分析、读写分离分析、缓存分析等，然后將此sQL发.
​	往后端的真实数据库，并将返回的结果做适当的处理，最终再返回给用户。

### 三、安装启动

#### 3.1 安装

1. 解压后即可使用:解压缩文件拷贝到Linux 下/usr/local

2. 三个配置文件：

   ①schema.xml:定义逻辑库，表、分片节点等内容。
   ②rule.xml:定义分片规则。
   ③server.xml:定义用户以及系统相关变量，如端口等。
   
3. 启动命令

   `mysql -umycat -pmycat -h192.168.42.120 -P8066`

### 四、搭建读写分离

​		我们通过Mycat和MySQL的主从复制配合搭建数据库的读写分离，实现MySQL的高可用性。我们将搭建：一主一从、双主双从两种读写分离模式。

#### 4.1 一主一从

​	

​	一台主机用于处理所有写请求，一台从主机负责处理所有读请求，架构图如下：

![image-20200623151625568](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/image-20200623151625568.png)

1、搭建MySQL数据库主从复制

①	MySQL主从复制原理（**接入点开始复制**）

![image-20200622160838059](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/image-20200622160838059.png)

②	主机配置

```properties
修改配置文件：vim /etc/my.cnf
#主服务器唯一ID
server-id=1
#启动二进制日志
log-bin=mysql-bin
#设置不要复制的数据库（可设置多个）
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
#设置需要复制的数据库
binlog-do-db=需要复制的主数据库名字
#设置logbin的格式
binlog_format=STATEMENT/ROW/MIXED
```

​	binlog 日志三种格式

![image-20200623231404619](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/image-20200623231404619.png)

③	从机配置

```mysql
修改配置文件：vim/etc/my.cnf
#从机服务器唯一ID
server-id=2
#启用中继日志
reloy-log=mysql-relay
```

④	主机、从机重启mysql服务

⑤	主机从机都关闭防火墙

⑥	在主机建立用户并授权slave

```mysql
#在主机MySQL里执行授权命令
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%' IDENTIFIED BY 'root';
show master status
#记录FILE和POS值，不在操作主机
```

⑦	在从机上配置需要复制的主机

```mysql
#复制主机的命令
CHANGE MASTER TO MASTER_HOST='192.168.42.120',
MASTER_PORT=33305,
MASTER_USER='slave',
MASTER_PASSWORD='root',
MASTER_LOG_FILE='mysql-bin.000002',
MASTER_LOG_POS=435;
#启动从机服务器复制功能
start slave;
#查看从服务器状态
show slave status\G;
#停止从机服务器
stop slave;
#重新配置主从关系
stop slave;
reset master;
```

#### 4.2 双主双从

一个主机m1用于处理所有写请求,它的从机s1和另一台主机m2还有它的从机s2负责所有读请求。当m1主机宕机后，m2主机负责写请求，m1、m2互为备机。架构图如下。

![image-20200705155630985](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/image-20200705155630985.png)

①	双主机配置

<font color='red'>Master1配置</font>

```mysql
修改配置文件：vim /etc/my.cnf
#主服务器唯一ID
server-id=1
#启动二进制日志
log-bin=mysql-bin
#设置不要复制的数据库（可设置多个）
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
#设置需要复制的数据库
binlog-do-db=需要复制的主数据库名字
#设置logbin的格式
binlog_format=STATEMENT/ROW/MIXED
#在作为从数据库的时候，有写入操作也要更新二进制日志文件。
log-slave-updates
# 表示自增长字段每次递增的量，指自增字段的起始值，其默认值是1，取值范围是1...6535
auto-increment-increment=2
# 表示自增长字段从哪个数开始，指字段一次递增多少，他的取值范围是1.. 65535。
auto-increment-offset=1
```

<font color='red'>Master2配置</font>

```mysql
修改配置文件：vim /etc/my.cnf
#主服务器唯一ID
server-id=3
#启动二进制日志
log-bin=mysql-bin
#设置不要复制的数据库（可设置多个）
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
#设置需要复制的数据库
binlog-do-db=需要复制的主数据库名字
#设置logbin的格式
binlog_format=STATEMENT/ROW/MIXED
#在作为从数据库的时候，有写入操作也要更新二进制日志文件。
log-slave-updates
# 表示自增长字段每次递增的量，指自增字段的起始值，其默认值是1，取值范围是1...6535
auto-increment-increment=2
# 表示自增长字段从哪个数开始，指字段一次递增多少，他的取值范围是1.. 65535。
auto-increment-offset=2
```

②	双从机配置

Slave1配置

```mysql
修改配置文件：vim/etc/my.cnf
#从机服务器唯一ID
server-id=2
#启用中继日志
reloy-log=mysql-relay
```

Slave2配置

```mysql
修改配置文件：vim/etc/my.cnf
#从机服务器唯一ID
server-id=4
#启用中继日志
reloy-log=mysql-relay
```

③	主机、从机重启mysql服务

④	主机从机都关闭防火墙

⑤	Master1在主机建立用户并授权slave

```mysql
#连接mysql
mysql -uroot -proot -h192.168.42.120 -P33305
#停止slave
stop slave;
reset master;
#建立用户并授权slave
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%' IDENTIFIED BY 'root';
show master status
#记录FILE和POS值，不在操作主机
```

⑥	 在从机Slave1上配置需要复制的主机

```mysql
#连接mysql
mysql -uroot -proot -h192.168.42.120 -P33304
#复制主机的命令
CHANGE MASTER TO MASTER_HOST='192.168.42.120',
MASTER_PORT=33305,
MASTER_USER='slave',
MASTER_PASSWORD='root',
MASTER_LOG_FILE='mysql-bin.000002',
MASTER_LOG_POS=435;
#启动从机服务器复制功能
start slave;
#查看从服务器状态
show slave status\G;
#停止从机服务器
stop slave;
#重新配置主从关系
stop slave;
reset master;
```

⑦	Master2在主机建立用户并授权slave

```mysql
#连接mysql
mysql -uroot -proot -h192.168.42.120 -P33303
#停止slave
stop slave;
reset master;
#建立用户并授权slave
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%' IDENTIFIED BY 'root';
show master status
#记录FILE和POS值，不在操作主机
```

⑧	在从机Slave2上配置需要复制的主机

```mysql
#连接mysql
mysql -uroot -proot -h192.168.42.120 -P33302
#复制主机的命令
CHANGE MASTER TO MASTER_HOST='192.168.42.120',
MASTER_PORT=33305,
MASTER_USER='slave',
MASTER_PASSWORD='root',
MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=154;
#启动从机服务器复制功能
start slave;
#查看从服务器状态
show slave status\G;
#停止从机服务器
stop slave;
#重新配置主从关系
stop slave;
reset master;
```

⑨	Master1复制到Master2，Master2复制到Master1

```mysql
#复制主机1的命令
CHANGE MASTER TO MASTER_HOST='192.168.42.120',
MASTER_PORT=33305,
MASTER_USER='slave',
MASTER_PASSWORD='root',
MASTER_LOG_FILE='mysql-bin.000002',
MASTER_LOG_POS=435;
#启动从机服务器复制功能
start slave;
#查看从服务器状态
show slave status\G;
#复制主机2的命令
CHANGE MASTER TO MASTER_HOST='192.168.42.120',
MASTER_PORT=33303,
MASTER_USER='slave',
MASTER_PASSWORD='root',
MASTER_LOG_FILE='mysql-bin.000002',
MASTER_LOG_POS=435;
#启动从机服务器复制功能
start slave;
#查看从服务器状态
show slave status\G;
```

⑩	创建testMycat和表user，添加数据到user表

```
#创建testMycat库
create database testMycat default charset utf8 collate utf8_general_ci;
#创建user表
create table user(
	id int not null auto_increment primary key,
    name varchar(16));
#插入数据
insert into user values(1000,'张三');
```

```tex
#balance="1":全部的readHost与stand by writeHost参与select语句的负载均衡。。
#writeType="0":所有写操作发送到配置的第一个writeHost，第一个挂了切到还生存的第二个。
#writeType="1",所有写操作都随机的发送到配置的writeHost, 1.5 以后废弃不推荐。
#writeHost,重新启动后以切换后的为准，切换记录在配置文件中:dnindex.properties。。
#switchType="1": 1默认值，自动切换
				-1表示丕自动切换。
				2基于MySQL主从同步的状态决定是否切换。
```

### 五、垂直拆分——分库

​		一个数据库由很多表的构成，每个表对应着不同的业务,垂直切分是指按照业务将表进行分类,分布到不同的数据库上面，这样也就将数据或者说压力分担到不同的库上面，如下图:

![image-20200706155727081](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/image-20200706155727081.png)

#### 5.1 如何划分表

​	分库的原则：有紧密关联关系的表应该在一个库里，相互没有关联的表可以分到不同的库里。6000

```mysql
#客户表	rows:20万
create table customer(
	id int auto_increment primary key,
	name varchar(200)
);
#订单表	rows：600万
create table orders(
	id int auto_increment primary key,
	order_type int,
	customer_id int,
	amount decimal(10,2)
);
#订单详情表	rows：600万
create table orders_detail(
	id int auto_increment primary key,
	detail varchar(2000),
	order_id int
);
#订单状态字典表	rows：20万
create table dict_order_type(
	id int auto_increment primary key,
	order_type varchar(200)
);
```

客户表分在一个数据库，另外三张表都需要关联查询，分在另外一个数据库。

#### 5.2 配置分库

​	1、修改schema配置文件

```xml
...
		<schema name="TESTDB" checkSQLschema="true" sqlMaxLimit="100" dataNode="dn1">
                <table name="customer" dataNode="dn2"></table>
        </schema>
        <dataNode name="dn1" dataHost="host1" database="orders" />
        <dataNode name="dn2" dataHost="host2" database="orders" />
        <dataHost name="host1" maxCon="1000" minCon="10" balance="0"
                          writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <!-- can have multi write hosts -->
                <writeHost host="hostM1" url="jdbc:mysql://192.168.42.120:33305" user="root"
                         password="root">
                </writeHost>
        </dataHost>
        <dataHost name="host2" maxCon="1000" minCon="10" balance="0"
                          writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <!-- can have multi write hosts -->
                <writeHost host="hostM2" url="jdbc:mysql://192.168.42.120:33303" user="root"
                         password="root">
                </writeHost>
        </dataHost>
```

​	2、新增两个空白的数据库

​			分库操作不是在原来的老数据库上进行操作，需要准备两台机器分别安装新的数据库。

```mysql
#在数据节点dn1，dn2上分别创建数据库orders
CREATE DATABASE orders DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
```

​	3、在mycat上创建四个表

​		dn1有orders表、orders_detail表、dict_order_type表三张表，而dn2只有customer表

### 六、水平拆分——分表

​		相对于垂直拆分，水平拆分不是将表做分类,而是按照某个字段的某种规则来分散到多个库之中，每个表中含一部分数据。简单来说，我们可以将数据的水平切分理解为是按照数据行的切分，就是将表中的某些行切分到一个数据库，而另外的某些行又切分到其他的数据库中，如图：

![image-20200706164327703](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/image-20200706164327703.png)

#### 6.1 配置分表

1、选择要拆分的表。
		MySQL单表存储数据条数是有瓶颈的，单表达到1000万条数据就达到了瓶颈，会影响查询效率，需要进行水平拆分(分表)进行优化。例如:例子中的orders、orders_ _detail 都已经达到600万行数据，需要进行分表优化。

2、分表字段

以orders表为例，可以根据不同字段进行分表

| 编号 | 分表字段              | 效果                                                         |
| ---- | --------------------- | ------------------------------------------------------------ |
| 1    | id（主键或创建时间）  | 查询订单注重时效，历史订单被查询的次数少，如此分片会造成一个接待你访问多，一个节点访问少，不平均，浪费资源。 |
| 2    | customer_id（客户id） | 根据客户id去分，两个接待你访问平均，一个客户的订单都在同一个节点上。 |

3、修改schema.xml配置文件

```xml
#为orders表设置数据节点为dn1、dn2，并指定分片规则为mod_rule（自定义的名字）
<table name="orders" dataNode="dn1,dn2" rule="mod_rule"></table>
```

4、修改配置文件rule.xml

```xml
#在rule配置文件里新增分片规则mod_rule， 并指定规则适用字段为customer_id.
#还有选择分片算法mod-long (对字段求模运算)，customer_id对两个节点求模，根据结果分片。
#配置算法mod-long参数count为2，两个节点。
		<tableRule name="mod_rule">
                <rule>
                        <columns>customer_id</columns>
                        <algorithm>mod_long</algorithm>
                </rule>
        </tableRule>
		<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
                <!-- how many data nodes -->
                <property name="count">2</property>
        </function>
```

5、在数据节点dn2上建orders表

6、重启Mycat，让配置生效

7、访问Mycat实现分片

```mysql
#在mycat里向orders表插入数据,字段不能省略
insert into orders(id,order_type,customer_id,amount) values(1,101,100,100100);
insert into orders(id,order_type,customer_id,amount) values(2,101,100,100300);
insert into orders(id,order_type,customer_id,amount) values(3,101,101,120000);
insert into orders(id,order_type,customer_id,amount) values(4,101,101,103000);
insert into orders(id,order_type,customer_id,amount) values(5,102,101,100400);
insert into orders(id,order_type,customer_id,amount) values(6,102,100,100020);
```

#### 6.2 Mycat的分片join

​	orders订单表已经进行分表操作，和它关联的order_detail订单详细表如何进行join查询。

我们要对orders_detail也要进行分片操作。join的原理如下图：

![image-20200706175706360](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/image-20200706175706360.png)

1、ER表

​		Mycat借鉴了NewSQL 领域的新秀Foundation DB的设计思路，Foundation DB创新性的提出了Table Group的概念，其将子表的存储位置依赖于主表，并且物理上紧邻存放，因此彻底解决了join的效率和性能问题，根据这一思路， 提出了基于E-R 关系的数据分片策略，子表的记录与所关联的父表记录存放在同一个数据分片上。

```yml
#修改schema.xml文件
<table name="orders" dataNode="dn1,dn2" rule="mod_rule">
	<childTable name="orders_detail" primaryKey="id" joinKey="order_id" parentKey="id" /></table>
#再dn2创建orders_detail表
#重启mycat
#访问Mycat向orders_ detail 表插入数据。
INSERT INTO orders_detail(id,detail,order_id) values(1,'detail1',1);
INSERT INTO orders_detail(id,detail,order_id) VALUES(2,'detail1',2);
INSERT INTO orders_detail(id,detail,order_id) VALUES(3,'detail1',3);
INSERT INTO orders_detail(id,detail,order_id) VALUES(4,'detail1',4);
INSERT INTO orders_detail(id,detail,order_id) VALUES(5,'detail1',5);
INSERT INTO orders_detail(id,detail,order_id) VALUES(6,'detail1',6);
INSERT INTO orders_detail(id,detail,order_id) VALUES(7,'detail1',7);

#在mycat、dn1、 dn2中运行两个表join语句。
select o.*,od.detail from orders o inner join orders_detail od on o.id= od.order_id;
```

3、全局表可以跟任何一个表进行JOIN操作。将字典表或者符合字典表特性的一-些表定义为全局表，则从另外-一个方面，很好的解决了数据JOIN的难题。通过全局表+基于E-R关系的分片策略，Mycat可以满足80%以上的企业应用开发。

```xml
<table name= ="orders" dataNode= ="dn1 ,dn2" rule= ="mod_rule" >。
    <childTable name= "orders_detail" primaryKey="id" joinKey="order_id" parentKey="id"/></table>
<table name="dict_order_type" dataNode="dn1,dn2" type="global"></table>
#在dn2创建dict_order_type表
#重启mycat
#向dict_order_type表插入两条数据
insert into dict_order_type(id,order_type) values(101,'type1');
insert into dict_order_type(id,order_type) values(102,'type2');
#在mycat、dn1、dn2忠查询表数据
select * from dict_order_type;
```

#### 6.3 常用的分片规则

​	**1、取模**

​	此规则为对分片字段求摸运算。也是水平分表最常用规则。配置分表中, orders 表采用了此规则。

​	**2、分片枚举**

​		通过在配置文件中配置可能的枚举id，自己配置分片，本规则适用于特定的场景，比如有些业务需要按照省份或区县来做保存，而全国省份区县固定的，这类业务使用本条规则。

```mysql
#(1)修改schema.xml配置文件。
<table name="orders_ware_info" dataNode="dn1,dn2" rule="sharding_by_intfile"></table>
#(2)修改rule.xml配置文件。
<tableRule name="sharding_by_ intfile">
	<rule>
		<columns>areacode</columns>。
		<algorithm>hash-int</algorithm>
	</rule>
</tableRule>
<function name= "hash-int" class="io.mycatroute.function.PartitionByFileMap">
	<property name="mapFile">partition-hash-int.txt </property>v
	<property name="type">1<property>o
	<property name="defaultNode">0</property>
</function>
# columns: 分片字段，algorithm: 分片函数。
# mapFile:标识配置文件名称，type: 0为int型、非0为String。
# defaultNode: 默认节点:小于0表示不设置默认节点，大于等于0表示设置默认节点，
# 设置默认节点如果碰到不识别的枚举值，就让它路由到默认节点，如不设置不识别就报错
# (3)修改partition-hash-int.txt配置文件
	110=0
	120=1

# (4)重启Mycat
# (5)访问Mycat创建表
# 订单归属区域信息表
create table orders_ware_info
(
	`id` int auto_increment comment '编号',
	`order_id` int comment '订单编号',
	`address` varchar(200) comment '地址',
	`areacode` varchar(20) comment '区域编号',
	primary key(id)
)
# 插入数据
insert into orders_ware_info(id,order_id,address,areacode) values(1,1,'北京','110');
insert into orders_ware_info(id,order_id,address,areacode) values(2,2,'天津','120');
# (7) 查询mycat、dn1、dn2可以看到数据分片效果

```

​	**3、范围约定**

​			此分片适用于，提前规划好分片字段某个范围属于哪个分片。

```mysql
#(1)修改schema.xml配置文件。
<table name="payment_info" dataNode="dn1,dn2" rule="auto_sharding_long"></table>
#(2)修改rule.xml配置文件。
<tableRule name="auto_sharding_long">
	<rule>
		<columns>order_id</columns>。
		<algorithm>rang-long</algorithm>
	</rule>
</tableRule>
<function name="rang-long" class="io.mycat.route.function.AutoPartitionByLong">
	<property name="mapFile">autopartition-long.txt</property>.
	<property name="defaultNode">0</property>
</function>
# columns: 分片字段，algorithm: 分片函数。
# mapFile:标识配置文件名称，type: 0为int型、非0为String。
# defaultNode: 默认节点:小于0表示不设置默认节点，大于等于0表示设置默认节点，
# 设置默认节点如果碰到不识别的枚举值，就让它路由到默认节点，如不设置不识别就报错
# (3)修改autopartition-long.txt配置文件
0-102=0
103-200=1
# (4)重启Mycat
# (5)访问Mycat创建表
# 支付信息表
CREATE TABLE payment_info
(
	`id` INT AUTO INCREMENT comment '编号',
	`order_id` INT comment '订单编号',
	`payment_status` INT comment '支付状态',
	PRIMARY KEY(id)
);
# (6)插入数据
INSERT INTO payment_info (id,order_id,payment_status) VALUES (1,101,0);
INSERT INTO payment_info (id,order_id,payment_status) VALUES (2,102,1);
INSERT INTO payment_info (id,order_id,payment_status) VALUES (3,103,0);
INSERT INTO payment_info (id,order_id,payment_status) VALUES (4,104,1);
# (7)查询Mycat、dn1、dn2可以看到数据分片效果。
```

​	**4、按日期(日)分片**

​	此规则为按天分片。设定时间格式、范围。

```mysql
#(1)修改schema.xml配置文件。
<table name="payment_info" dataNode="dn1,dn2" rule="auto_sharding_long"></table>
#(2)修改rule.xml配置文件。
<tableRule name="sharding_by_date">
	<rule>
		<columns>login_date</columns>
		<algorithm>shardingByDate</algorithm>
	</rule>
</tableRule>
<function name= ="shardingByDate" class= ="io.mycat.route.function.PartitionByDate">
	<property name="dateFormat">yyy-MM-dd</property>
    <property name="sBeginDate">2019-01-01</property>
    <property name="sEndDate">2019-01-04</property>
    <property name="sPartionDay">2</property>
</function>
# columns:分片字段，algorithm: 分片函数。
#dateFormat: 日期格式。
#sBeginDate: 开始日期。
#sEndDate: 结束日期，则代表数据达到了这个日期的分片后循环从开始分片插入。
#sPartionDay: 分区天数，即默认从开始日期算起，分隔2天一个分区。
# (4)重启Mycat
# (5)访问Mycat创建表
# 支付信息表
CREATE TABLE login_info
(
	id INT AUTO_INCREMENT comment '编号',
	user_id INT comment '用户编号',
	login_date date comment '登录日期',
	PRIMARY KEY(id)
);
# (6)插入数据
INSERT INTO login_info(id,user_id,login_date) VALUES (1,101,'2019-01-01');
INSERT INTO login_info(id,user_id,login_date) VALUES (2,102,'2019-01-02');
INSERT INTO login_info(id,user_id,login_date) VALUES (3,103,'2019-01-03');
INSERT INTO login_info(id,user_id,login_date) VALUES (4,104,'2019-01-04');
INSERT INTO login_info(id,user_id,login_date) VALUES (5,103,'2019-01-05');
INSERT INTO login_info(id,user_id,login_date) VALUES (6,104,'2019-01-06');
```

#### 6.4 全局序列

**1、本地文件**

​		此方式Mycat将sequence 配置到文件中，当使用到sequence 中的配置后，Mycat会更下classpath中的sequence_ conf.properties 文件中sequence 当前的值。
​	①	优点:本地加载，读取速度较快。
​	②	缺点:抗风险能力差，Mycat所在主机宣机后，无法读取本地文件。

2、数据库方式。
		利用数据库-一个表来进行计数累加。但是并不是每次生成序列都读写数据库，这样效率太低。Mycat会预加载一部分 号段到Mycat的内存中，这样大部分读写序列都是在内存中完成的。如果内存中的号段用完了Mycat 会再向数据库要一次。

​	问:那如果mycat崩溃了，那内存中的序列岂不是都没了?

​	是的。如果是这样，那么mycat启动后会向数据库申请新的号段，原有号段会弃用。
​	也就是说如果mycat重启，那么损失是当前的号段没用完的号码，但是不会因此出现主键重复。

​	①	建序列表

```mysql
#在dnl上创建全局序列表。
drop table if exists MYCAT_SEQUENCE;
CREATE TABLE MYCAT_SEQUENCE (NAME VARCHAR(50) NOT NULL,current_value INT NOT NULL,increment INT NOT NULL DEFAULT 100, PRIMARY KEY(NAME)) ENGINE=INNODB;
#创建全局序列所需函数
DROP FUNCTION IF EXISTS mycat_seq_currval;
DELIMITER $$
CREATE FUNCTION mycat_seq_currval(seq_name VARCHAR(50)) RETURNS VARCHAR(64) CHARSET utf8
DETERMINISTIC
BEGIN
DECLARE retval VARCHAR(64);
SET retval = "-999999999,null";
SELECT CONCAT(CAST(current_value AS CHAR),",",CAST(increment AS CHAR)) INTO retval FROM
MYCAT_SEQUENCE WHERE NAME = seq_name;
RETURN retval;
END $$
DELIMITER ;

DROP FUNCTION IF EXISTS mycat_seq_setval;
DELIMITER $$
CREATE FUNCTION mycat_seq_setval(seq_name VARCHAR(50),VALUE INTEGER) RETURNS VARCHAR(64) CHARSET utf8
DETERMINISTIC
BEGIN
UPDATE MYCAT_SEQUENCE
SET current_value = VALUE
WHERE NAME = seq_name;
RETURN mycat_seq_currval(seq_name);
END $$
DELIMITER ;

DROP FUNCTION IF EXISTS mycat_seq_nextval;
DELIMITER $$
CREATE FUNCTION mycat_seq_nextval(seq_name VARCHAR(50)) RETURNS VARCHAR(64) CHARSET UTF8
DETERMINISTIC
BEGIN
UPDATE MYCAT_SEQUENCE
SET current_value = current_value + increment WHERE NAME = seq_name;
RETURN mycat_seq_currval(seq_name);
END $$
DELIMITER ;
#初始化序列表记录
INSERT INTO MYCAT_SEQUENCE(NAME,current_value,increment) VALUES ('ORDERS',400000,100);
```

​	②	修改mycat配置

```shell
#修改sequence_db_conf.properties
vim sequence_db_conf properties
#意思是ORDERS这个序列在dn1这个节点上，具体dn1节点是哪台机子，请参考schema.xml

#修改server.xml
vim server.xml
<property name="sequenceHandlerType">1</property>
#全局序列类型: 0-本地文件，1-数据库方式，2-时间戳方式。此处应该修改成1.
#重启mycat
```

​	③	验证全局序列e

```mysql
#登录mycat,插入数据
#在1.6之后插入语法
insert into orders(id,amount,customer_id,order_type) values('next value for MYCATSEQ_ORDERS',1000,101,102);

#查询数据
```

### 七、基于HA机制的Mycat的高可用

​	在实际项目中，Mycat服务也需要考虑高可用性，如果Mycat所在服务器出现宣机，或Mycat服务故障，需要有备机提供服务，需要考虑Mycat集群。

#### 7.1 高可用方案

​		我们可以使用HAProxy + Keepalived配合两台Mycat搭起Mycat集群实现高可用性。HAProxy实现了MyCat多节点的集群高可用和负载均衡，而HAProxy自身的高可用则可以通过Keepalived来实现。

![image-20200710221153234](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/image-20200710221153234.png)

#### 7.2 安装配置HAProxy

```shell
#1、下载压缩包
https://github.com/haproxy/haproxy
#2、解压到/usr/local/src
tar -zxvf haproxy-2.2.0.tar.gz -C /usr/local/src
#3、进入解压后的目录，查看内核版本，进行编译
cd /usr/local/src/haproxy-2.2.0
#查看内核版本
uname -r
make TARGET=linux310 PREFIX=/usr/local/haproxy ARCH=x86_64
make TARGET=linux-glibc USE_OPENSSL=1 USE_SYSTEMD=1 USE_PCRE=1 USE_ZLIB=1 PREFIX=/usr/local/haproxy && make install PREFIX=/usr/local/haproxy
# ARGET=linux310,内核版本，使用uname -r查看内核，如:3.10.0-1127.13.1.el7.x86_64 此时该参数就为linux310;
# ARCH=x86_64，系统位数;
# PREFIX=/usr/local/haprpxy 
# /usr/loca/haprpxy，为haprpxy安装路径。

#5安装完成后，创建目录、创建HAProxy配置文件。
mkdir -p /usr/datal haproxy/
vim /usr/local/haproxy/haproxy.conf

#6向配置文件中插入以下配置信息，并保存
global
	#设置日志
	log 127.0.0.1 local3 info
	#用户与用户组
	user nobody
	group nobody
	#守护进程启动
	daemon
	#最大连接数
	maxconn 4000
	pidfile /var/run/haproxy.pid

#默认配置
defaults
	log global
	mode http
	option httplog
	option dontlognull
	timeout connect 5000
	timeout client 50000
	timeout server 50000

#前端配置，http_front名称可自定义
frontend http_front
	#发起http请求到10080端口，会被转发到设置的ip及端口
	bind *:10080
	#haproxy的状态管理页面，通过/haproxy?stats来访问
	stats uri /haproxy?stats
	default_backend http_back

#后端配置，http_back名称可自定义
backend http_back
	#负载均衡方式
	#source 根据请求源IP
	#static-rr 根据权重
	#leastconn 最少连接者先处理
	#uri 根据请求的uri
	#url_param 根据请求的url参数
	#rdp-cookie 据据cookie(name)来锁定并哈希每一次请求
	#hdr(name) 根据HTTP请求头来锁定每一次HTTP请求
	#roundrobin 轮询方式
	balance roundrobin
	#设置健康检查页面
	option httpchk GET /index.html
	#传递客户端真实IP
	option forwardfor header X-Forwarded-For
	# inter 2000 健康检查时间间隔2秒
	# rise 3 检测多少次才认为是正常的
	# fall 3 失败多少次才认为是不可用的
	# weight 30 权重
	# 需要转发的ip及端口
	server web1 192.168.79.129:80 check inter 2000 rise 3 fall 3 weight 30
	server web2 192.168.79.131:80 check inter 2000 rise 3 fall 3 weight 30
listen admin_stats
        bind 0.0.0.0:9188
        mode http
        stats enable
        option httplog
        maxconn 10
        stats refresh 30s
        stats uri /haproxy-status
        stats realm welcome login\ Haproxy
        stats auth admin:admin123
        stats hide-version
        stats admin if TRUE
#1启动HAProxy。
/usr/local/haproxy/sbin haproxy -f /usr/local/haproxy/haproxy.conf
#2查看HAProxy进程。
ps -efgrep haproxy
#3打开浏览器访问
http://192.168.42.120:9188/haproxy-status
#在弹出框输入用户名: admin密码: admin123
#查看启动进程
ps -ef|grep haproxy
#关闭haproxy
kill -9 进程号
#4验证负载均衡，通过HAProxy访 问Mycat。
mysql -unycat -pmycat h 192.168.42.120 -P 48066
```

7.3 安装配置Keppalived

```shell
#1准备好Keepalived安装包，传到/opt目录下
#2解压到usr/local/src
tar -zxvf keepalived-2.1.5.tar.gz -C /usr/local/src
#3安装依赖插件
yum install -y gecc openssl -devel popt-devel
#3进入解压后的目录，进行配置，进行编译。
cd /usr/local/src/keepalived-2.1.5
#3安装依赖插件。
yum install -y gcc openssl-devel popt-devel
#3进入解压后的目录，进行配置，进行编译。
cd /usr/local/src/keepalived-2.1.5
/configure --prefix-/usr/local/keepalived
#4进行编译，完成后进行安装
make && make install
#5运行前配置。
cp /usr/local/srckeepalived-2.1.5/keepalived/etc/init.dkeepalived /et/init.d/
mkdir /etc/keepalived
cp /usr/local/keepalived/etclkeepalivedkeepalived.conf /etclkeepalived/
cp /usr/local/src/keepalived-2.1.5/keepalivld/etc/sysconfig/keepalived /etc/sysconfig/
cp /usr/local/keepalived/sbin/keepalived /usr/sbin/
#6修改配置文件
vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     1036700294@qq.com
   }
   notification_email_from keepalived@showjoy.com
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.42.200
    }
}

virtual_server 192.168.42.200 48066 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.42.120 48066 {
        weight 1
        HTTP_GET {
            url {
              path /
              digest ff20ad2481f97b1754ef3e12ecd3a9cc
            }
            url {
              path /mrtg/
              digest 9b3a0c85a887a256d6939da88aabd8cd
            }
}
#1启动Keepalived.
service keepalived start
mysql -umycat -pmycat -h192.168.42.200 -P48066
```

### 八、Mycat安全设置

#### 8.1 权限配置

​	①用户权限

```xml
#设置可读权限，默认为false
<user name="user">
	<property name="readOnly">true</property>
</user>
```

​	②表权限

```
<privileges check="false">
	<schema name="TESTDB" dml="0110" >
    	<table name="tb01" dml="0000"></table>
		<table name="tb02" dml="1111"></table>
     </schema>
</privileges>   
```

​	配置说明

| DML权限 | 增加（insert） | 更新（update） | 查询（select） | 删除（delete） |
| ------- | -------------- | -------------- | -------------- | -------------- |
| 0000    | 禁止           | 禁止           | 禁止           | 禁止           |
| 0010    | 禁止           | 禁止           | 可以           | 禁止           |
| 1110    | 可以           | 可以           | 可以           | 禁止           |
| 1111    | 可以           | 可以           | 可以           | 可以           |

#### 8.2 SQL拦截

​	firewall来设置白黑名单

```
<firewall>
       <whitehost>
              <host host="1*7.0.0.*" user="root"/>
       </whitehost>
       <blacklist check="false">
       </blacklist>
</firewall>
```

可以设置的黑名单SQL拦截功能列表

| 配置项           | 缺省值 | 描述                          |
| ---------------- | ------ | ----------------------------- |
| selectAllow      | true   | 是否允许执行 SELECT 语句      |
| deleteAllow      | true   | 是否允许执行 DELETE 语句      |
| updateAllow      | true   | 是否允许执行 UPDATE 语句      |
| insertAllow      | true   | 是否允许执行 INSERT 语句      |
| createTableAllow | true   | 是否允许创建表                |
| setAllow         | true   | 是否允许使用 SET 语法         |
| alterAllow       | true   | 是否允许执行 ALTER TABLE 语句 |
| dropAllow        | true   | 是否允许修改表                |
| commitAllow      | true   | 是否允许执行 commit 操作      |
| rollbackAllow    | true   | 是否允许执行 rollback 操作    |

### 九、Mycat监控工具

​		Mycat-web是Mycat可视化运维的管理和监控平台，弥补了Mycat 在监控上的空白。帮Mycat分

担统计任务和配置管理任务。Mycat-web 引入了ZooKeeper 作为配置中心，可以管理多个节点。

Mycat-web主要管理和监控Mycat的流量、连接、活动线程和内存等，具备IP白名单、邮件警告等模

块，还可以统计SQL并分析慢SQL和高频SQL等。为优化SQL提供依据。

#### 	9.1 mycat-web简介

![image-20200715221322512](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/image-20200715221322512.png)

#### 	9.2 mycat-web配置使用

```
netstat -ant | grep 8082
```



#### 	9.3 mycat性能监控指标