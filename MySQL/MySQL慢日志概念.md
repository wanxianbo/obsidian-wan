#### **MySQL慢日志概念**

​		MySQL的慢查询日志是MySQL提供的一种日志记录，它用来记录在MySQL中响应时间超过阀值的语句，具体指运行时间超过long_query_time值的SQL，则会被记录到慢查询日志中。long_query_time的默认值为10，意思是运行10S以上的语句。默认情况下，Mysql数据库并不启动慢查询日志，需要我们手动来设置这个参数，当然，如果不是调优需要的话，一般不建议启动该参数，因为开启慢查询日志会或多或少带来一定的性能影响。慢查询日志支持将日志记录写入文件，也支持将日志记录写入数据库表。



#### **慢查询日志相关参数**

MySQL 慢查询的相关参数解释：

slow_query_log  ：是否开启慢查询日志，1表示开启，0表示关闭。

log-slow-queries ：旧版（5.6以下版本）MySQL数据库慢查询日志存储路径。可以不设置该参数，系统则会默认给一个缺省的文件host_name-slow.log

slow-query-log-file：新版（5.6及以上版本）MySQL数据库慢查询日志存储路径。可以不设置该参数，系统则会默认给一个缺省的文件host_name-slow.log

long_query_time ：慢查询阈值，当查询时间多于设定的阈值时，记录日志。

log_queries_not_using_indexes：未使用索引的查询也被记录到慢查询日志中（可选项）。

log_output：日志存储方式。log_output='FILE'表示将日志存入文件，默认值是'FILE'。log_output='TABLE'表示将日志存入数据库，这样日志信息就会被写入到mysql.slow_log表中。MySQL数据库支持同时两种日志存储方式，配置的时候以逗号隔开即可，如：log_output='FILE,TABLE'。日志记录到系统的专用日志表中，要比记录到文件耗费更多的系统资源，因此对于需要启用慢查询日志，又需要能够获得更高的系统性能，那么建议优先记录到文件。



#### 慢查询日志配置

默认情况下slow_query_log的值为OFF，表示慢查询日志是禁用的，可以通过设置slow_query_log的值来开启，如下所示：

```mysql
mysql> show variables like '%slow_query_log%';
+---------------------+-------------------------------------------------+
| Variable_name       | Value                                           |
+---------------------+-------------------------------------------------+
| slow_query_log      | OFF                                             |
| slow_query_log_file | /var/lib/mysql/iZwz92kcyvmrtxo5bkhyfoZ-slow.log |
+---------------------+-------------------------------------------------+
2 rows in set (0.04 sec)
mysql> set global slow_query_log = 1;
Query OK, 0 rows affected (0.03 sec)

mysql> show variables like '%slow_query_log%';
+---------------------+-------------------------------------------------+
| Variable_name       | Value                                           |
+---------------------+-------------------------------------------------+
| slow_query_log      | ON                                              |
| slow_query_log_file | /var/lib/mysql/iZwz92kcyvmrtxo5bkhyfoZ-slow.log |
+---------------------+-------------------------------------------------+
2 rows in set (0.04 sec)
```

