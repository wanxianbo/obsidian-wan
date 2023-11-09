# Oracle学习笔记：窗口函数

SQL中的聚合函数，顾名思义是聚集合并的意思，是对某个范围内的数值进行聚合，聚合后的结果是一个值或是各个类别对应的值。直接聚合得到的结果是所有数据合并，分组聚合(group by)得到的结果是分组合并。

这种聚合函数得到的数据行数是小于基础数据行数的，但是我们经常会有这样的需求，就是既希望看基础数据同时也希望查看聚合后的数据，这个时候聚合函数就满足不了我们了，窗口函数就派上用场了。窗口函数就是既可以显示原始基础数据也可以显示聚合数据。

## 1.测试数据

学习当然不能凭空想象，需要大量的实践来提高学习效果。

先编排测试数据。

```sql
-- 创建测试表
create table temp_cwh_window
(
  shopname varchar(10),
  sales number,
  date2 date
);
-- 插入数据
insert into temp_cwh_window values('淘宝','50',to_date('20191013','yyyymmdd'));
insert into temp_cwh_window values('淘宝','35',to_date('20191014','yyyymmdd'));
insert into temp_cwh_window values('淘宝','63',to_date('20191015','yyyymmdd'));
insert into temp_cwh_window values('天猫','15',to_date('20191013','yyyymmdd'));
insert into temp_cwh_window values('天猫','59',to_date('20191014','yyyymmdd'));
insert into temp_cwh_window values('天猫','63',to_date('20191015','yyyymmdd'));
insert into temp_cwh_window values('京东','159',to_date('20191013','yyyymmdd'));
insert into temp_cwh_window values('京东','32',to_date('20191014','yyyymmdd'));
insert into temp_cwh_window values('京东','59',to_date('20191015','yyyymmdd'));
-- 查询
select * from temp_cwh_window;

```

![image-20220707105137894](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/202207071051521.png)

## 2.聚合函数+over()

`over()`的作用就是告诉SQL引擎：按区域对数据进行分区，然后累计每个切片的总额，再全部展示。

`over函数`指明在那些字段上做分析，其内跟`Partition by`表示对数据进行分组。注意`Partition by`可以有多个字段。

`over函数`可以和其它聚集函数、分析函数搭配，起到不同的作用。例如`sum`，还有诸如`rank`，`dense_rank`, `min`, `max`等。

- 平均销量

```sql
select shopname,
       sales,
       date2,
       avg(sales) over()
from temp_cwh_window;
```

## 3.partition by子句

使用`partition by`子句可以进行分组操作，类似于`group by`，需要与`over()`搭配使用。

```sql
select shopname,
       sales,
       date2,
       avg(sales) over(partition by shopname)
from temp_cwh_window;
```

## 4.order by子句

使用`order by`子句可以按照某一列数值进行排序，可与序列函数`ntile`、`row_number`、`lag`、`lead`、`first_value`、`last_value`等结合使用。

当 order by 与聚合函数一起使用时，是顺序聚合的。**类似于累计和的效果**，截止当前行的累计，这里需要特别注意。

```sql
select shopname,
       sales,
       date2,
       sum(sales) over(partition by shopname order by date2)
from temp_cwh_window;
```

```sql
-- 运行结果
1	京东	159	2019/10/13	159
2	京东	32	2019/10/14	191
3	京东	59	2019/10/15	250
4	淘宝	50	2019/10/13	50
5	淘宝	35	2019/10/14	85
6	淘宝	63	2019/10/15	148
7	天猫	15	2019/10/13	15
8	天猫	59	2019/10/14	74
9	天猫	63	2019/10/15	137
```

当order by与序列函数一起使用时用于排序。

## 5.序列函数

序列函数，就是可以将数据整理成一个有序的序列，然后可以在这个序列里面挑选我们想要的序列对应的数据。

### 5.1 分析函数之 ntile

`ntile`函数对一个数据分区中的有序结果集进行划分，将其分组为各个桶，并为每个小组分配一个唯一的组编号。默认是对表在不做任何操作之前进行切片分组的。

**注：**必须要加`order by`

- 随机切分为3组

```sql
select shopname,
       sales,
       date2,
       ntile(3) over(order by null)
from temp_cwh_window;
```

- 先分组排序再切分

```sql
select shopname,
       sales,
       date2,
       ntile(3) over(partition by shopname order by sales)
from temp_cwh_window;
```

- 空值排最后

```sql
select shopname,
       sales,
       date2,
       ntile(3) over(partition by shopname order by sales desc nulls last)
from temp_cwh_window;
```

### 5.2 分析函数之 row_number

`row_number()` 从 1 开始，按照顺序（注意这里是顺序不是排序）生成该条数据在分组内的对应的序列数，`row_number()` 的值不会存在重复，当排序的值相同时，按照表中记录的顺序进行排列。

`row_number()` 一般需要与 `order by` 进行结合使用。

`rank`是不连续排名函数(1,1,3,3,5)，`dense_rank` 是连续排名函数(1,1,2,2,3)。

```sql
select shopname,
       sales,
       date2,
       row_number() over(partition by shopname order by date2) as rank
from temp_cwh_window;
```

然后只需要将 `rank = 1` 部分数据取出即可。

### 5.3 分析函数之 lag、lead

`lag` 的英文意思是滞后，而 `lead` 的英文意思是超前。

对应的 `lag` 是让数据向后移动，而 `lead` 是让数据向前移动。

```sql
select shopname,
       sales,
       date2,
       lag(date2,1) over(partition by shopname order by date2)
from temp_cwh_window;
```

```sql
-- 结果
1	京东	159	2019/10/13	
2	京东	32	2019/10/14	2019/10/13
3	京东	59	2019/10/15	2019/10/14
4	淘宝	50	2019/10/13	
5	淘宝	35	2019/10/14	2019/10/13
6	淘宝	63	2019/10/15	2019/10/14
7	天猫	15	2019/10/13	
8	天猫	59	2019/10/14	2019/10/13
9	天猫	63	2019/10/15	2019/10/14
```

```sql
select shopname,
       sales,
       date2,
       lead(date2,1) over(partition by shopname order by date2)
from temp_cwh_window;
```

```sql
-- 结果
1	京东	159	2019/10/13	2019/10/14
2	京东	32	2019/10/14	2019/10/15
3	京东	59	2019/10/15	
4	淘宝	50	2019/10/13	2019/10/14
5	淘宝	35	2019/10/14	2019/10/15
6	淘宝	63	2019/10/15	
7	天猫	15	2019/10/13	2019/10/14
8	天猫	59	2019/10/14	2019/10/15
9	天猫	63	2019/10/15	
```

### 5.4 分析函数之 first_value、last_value

`first_value` 和 `last_value` 都是顾名思义，就是获取第一个值和最后一个值。

但是不是真正意义上的第一个或最后一个，而是`截至到当前行`的第一个或最后一个。

```sql
select shopname,
       sales,
       date2,
       first_value(date2) over(partition by shopname order by date2),
       last_value(date2) over(partition by shopname order by date2)
from temp_cwh_window;
```

```sql
-- 结果
1	京东	159	2019/10/13	2019/10/13	2019/10/13
2	京东	32	2019/10/14	2019/10/13	2019/10/14
3	京东	59	2019/10/15	2019/10/13	2019/10/15
4	淘宝	50	2019/10/13	2019/10/13	2019/10/13
5	淘宝	35	2019/10/14	2019/10/13	2019/10/14
6	淘宝	63	2019/10/15	2019/10/13	2019/10/15
7	天猫	15	2019/10/13	2019/10/13	2019/10/13
8	天猫	59	2019/10/14	2019/10/13	2019/10/14
9	天猫	63	2019/10/15	2019/10/13	2019/10/15
```

