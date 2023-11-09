# spring boot 集成seata简单demo

## 二、seata服务端的安装

### 2.1 seata-server服务端

- 本文介绍的是window版本的安装；
- 当前最新版本为1.4.0,选择binary类型的点击下载即可；
- 下载路径： http://seata.io/zh-cn/blog/download.html ；
- 下载下来进行解压，目录结构如下：
  ![img](https://img2020.cnblogs.com/blog/2229317/202103/2229317-20210304115402724-215520671.png)

### 2.1.2 添加log文件（1.4.1版本后自带）

- 在根目录创建logs文件夹，里面创建一个seata_gc.log的文件，结构如下：![img](https://img2020.cnblogs.com/blog/2229317/202103/2229317-20210304115726767-659324378.png)

## 2.2 初始化数据库

### 2.2.1 seata库数据表

- 数据库名称就叫seata;
- branch_table

```mysql
CREATE TABLE `branch_table` (
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(128) NOT NULL,
  `transaction_id` bigint(20) DEFAULT NULL,
  `resource_group_id` varchar(32) DEFAULT NULL,
  `resource_id` varchar(256) DEFAULT NULL,
  `branch_type` varchar(8) DEFAULT NULL,
  `status` tinyint(4) DEFAULT NULL,
  `client_id` varchar(64) DEFAULT NULL,
  `application_data` varchar(2000) DEFAULT NULL,
  `gmt_create` datetime(6) DEFAULT NULL,
  `gmt_modified` datetime(6) DEFAULT NULL,
  PRIMARY KEY (`branch_id`),
  KEY `idx_xid` (`xid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

- global_table

  ```mysql
  CREATE TABLE `global_table` (
    `xid` varchar(128) NOT NULL,
    `transaction_id` bigint(20) DEFAULT NULL,
    `status` tinyint(4) NOT NULL,
    `application_id` varchar(32) DEFAULT NULL,
    `transaction_service_group` varchar(32) DEFAULT NULL,
    `transaction_name` varchar(128) DEFAULT NULL,
    `timeout` int(11) DEFAULT NULL,
    `begin_time` bigint(20) DEFAULT NULL,
    `application_data` varchar(2000) DEFAULT NULL,
    `gmt_create` datetime DEFAULT NULL,
    `gmt_modified` datetime DEFAULT NULL,
    PRIMARY KEY (`xid`),
    KEY `idx_gmt_modified_status` (`gmt_modified`,`status`),
    KEY `idx_transaction_id` (`transaction_id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
  ```

- lock_table

```mysql
CREATE TABLE `lock_table` (
  `row_key` varchar(128) NOT NULL,
  `xid` varchar(96) DEFAULT NULL,
  `transaction_id` bigint(20) DEFAULT NULL,
  `branch_id` bigint(20) NOT NULL,
  `resource_id` varchar(256) DEFAULT NULL,
  `table_name` varchar(32) DEFAULT NULL,
  `pk` varchar(36) DEFAULT NULL,
  `gmt_create` datetime DEFAULT NULL,
  `gmt_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`row_key`),
  KEY `idx_branch_id` (`branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 2.2.2 业务库表

- seata需要在每个业务库添加一张undo_log表
- undo_log表

```mysql
CREATE TABLE `undo_log` (
  `branch_id` bigint(20) NOT NULL COMMENT 'branch transaction id',
  `xid` varchar(100) NOT NULL COMMENT 'global transaction id',
  `context` varchar(128) NOT NULL COMMENT 'undo_log context,such as serialization',
  `rollback_info` longblob NOT NULL COMMENT 'rollback info',
  `log_status` int(11) NOT NULL COMMENT '0:normal status,1:defense status',
  `log_created` datetime(6) NOT NULL COMMENT 'create datetime',
  `log_modified` datetime(6) NOT NULL COMMENT 'modify datetime',
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='AT transaction mode undo table';
```

## 2.3 修改file.conf文件

### 2.3.1 信息

- 只需要修改模式为db和数据库链接信息即可，如下图：

## 2.4 修改registry.conf文件

### 2.4.1 信息

- 只需要修改注册类型和配置类型均为nacos和nacos的相关信息即可

2.5 将配置导入到nacos配置中心

1. [下载配置](https://github.com/seata/seata/tree/develop/script/config-center)

   - /config.txt
   - /nacos/nacos-config.sh

2. 将文件按要求放置，如图：

3. 修改config.txt配置：

   > server端参数配置说明：https://seata.io/zh-cn/docs/user/configurations.html

   3.1配置事务会话信息存储方式为db，并配置mysql连接信息

   - store.mode=db
   - store.db.url=jdbc:mysql://127.0.0.1:3306/seata?useUnicode=true
   - store.db.user=root
   - store.db.password=root

   3.2.新增集成seata的client的service.vgroupMapping（有几个业务服务就增加几个mapping，请记号这几个红字，要与client端的配置一一对应的）

   - （新增一行）service.vgroupMapping.demo-web-seata-service-group=default
   - （新增一行）service.vgroupMapping.demo-goods-seata-service-group=default

4. 上nacos创建新的命名空间：seata-server，并存好命名空间id：c38bf03a-2ee2-4ee6-834a-2db99dbec528；作为seata-server的配置中心

5. 执行导入脚本：

   ```shell
   sh nacos-config.sh -h 127.0.0.1 -p 8848 -g SEATA_GROUP -u nacos -w nacos -t c38bf03a-2ee2-4ee6-834a-2db99dbec528
   ```

## 三、安装seata服务端启动包

1. 在服务器上下载seata服务端（官网下载安装，有教程）

2. 修改配置registry.conf

   ```
   
   # 1.将registry下的type改为nacos
   # 2.改registry下的nacos的配置
   nacos {
       application = "seata-server"
       serverAddr = "127.0.0.1:8848"
       group = "SEATA_GROUP"
       namespace = "c38bf03a-2ee2-4ee6-834a-2db99dbec528"
       cluster = "default"
   }
    
   # 3.将config下的type改为nacos
   # 4.改config下的nacos的配置
    nacos {
       serverAddr = "127.0.0.1:8848"
       namespace = "c38bf03a-2ee2-4ee6-834a-2db99dbec528"
       group = "SEATA_GROUP"
     }
   ```

3. 到seata根目录启动服务

   ```shell
   bin/seata-server.bat(sh)
   ```

4. 查看日志是否启动成功

5. 检查nacos服务列表已有新的服务“seata-server”，则服务端ok

## 四、配置seata客服端集成

1. 创建数据库

   - 在各个业务数据库里增加UNDO_LOG表，如有demo-web库和demo-goods库，则俩库里都要创建这个undo_log表。

     ```mysql
     
     CREATE TABLE IF NOT EXISTS `undo_log`
     (
         `branch_id`     BIGINT(20)   NOT NULL COMMENT 'branch transaction id',
         `xid`           VARCHAR(100) NOT NULL COMMENT 'global transaction id',
         `context`       VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
         `rollback_info` LONGBLOB     NOT NULL COMMENT 'rollback info',
         `log_status`    INT(11)      NOT NULL COMMENT '0:normal status,1:defense status',
         `log_created`   DATETIME(6)  NOT NULL COMMENT 'create datetime',
         `log_modified`  DATETIME(6)  NOT NULL COMMENT 'modify datetime',
         UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
     ) ENGINE = InnoDB
       AUTO_INCREMENT = 1
       DEFAULT CHARSET = utf8 COMMENT ='AT transaction mode undo table';
     ```

2. 配置maven依赖

   ```xml
   <dependency>
               <groupId>io.seata</groupId>
               <artifactId>seata-spring-boot-starter</artifactId>
               <version>1.4.2</version>
           </dependency>
   
           <dependency>
               <groupId>com.alibaba.nacos</groupId>
               <artifactId>nacos-client</artifactId>
               <version>1.4.2</version>
           </dependency>
   ```

3. 在nacos中新建client配置

   在nacos命名空间“demo-web”和“demo-goods”都新建dataId=seata-client的配置：

   client端参数配置说明：https://seata.io/zh-cn/docs/user/configurations.html

   dataId=seata-client

   group=MIDDLEWARE_GROUP

   配置文件demo-web：

   ```yaml
   seata:
     enabled: true
     application-id: demo-web
     tx-service-group: demo-web-seata-service-group # 要与服务端nacos-config.txt中service.vgroup_mapping的后缀对应
     service:
       vgroup-mapping:
         demo-web-seata-service-group: default
     registry:
       type: nacos
       nacos:
         application: seata-server
         server-addr: "127.0.0.1:8848"
         group : "SEATA_GROUP"
         namespace: "c38bf03a-2ee2-4ee6-834a-2db99dbec528"
     config:
       type: nacos
       nacos:
         server-addr: "127.0.0.1:8848"
         group : "SEATA_GROUP"
         namespace: "c38bf03a-2ee2-4ee6-834a-2db99dbec528"
   ```

   配置文件demo-goods

   ```yaml
   seata:
     enabled: true
     application-id: demo-goods-seata
     tx-service-group: demo-goods-seata-service-group # 要与服务端nacos-config.txt中service.vgroup_mapping的后缀对应
     service:
       vgroup-mapping:
         demo-goods-seata-service-group: default
     registry:
       type: nacos
       nacos:
         application: seata-server
         server-addr: "127.0.0.1:8848"
         group : "SEATA_GROUP"
         namespace: "c38bf03a-2ee2-4ee6-834a-2db99dbec528"
     config:
       type: nacos
       nacos:
         server-addr: "127.0.0.1:8848"
         group : "SEATA_GROUP"
         namespace: "c38bf03a-2ee2-4ee6-834a-2db99dbec528"
   ```

4. 业务代码支持

![image-20210701170617326](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2021/07/20210701170619.png)

![image-20210701170635818](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2021/07/20210701170636.png)

