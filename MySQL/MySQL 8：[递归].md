# MySQL 8：[递归] 

MySQL 中的通用表表达式 (CTE)，第四部分 - 深度优先或广度优先遍历、传递闭包、循环避免

## 深度优先或广度优先

我们假设我们有一张具有父子关系的人员表：

```mysql
CREATE TABLE tree (person CHAR(20), parent CHAR(20));
INSERT INTO tree VALUES
('Robert I', NULL),
('Thurimbert', 'Robert I'),
('Robert II', 'Thurimbert'),
('Cancor', 'Thurimbert'),
('Landrade', 'Thurimbert'),
('Ingramm', 'Thurimbert'),
('Robert III', 'Robert II'),
('Chaudegrand', 'Landrade'),
('Ermengarde', 'Ingramm');
```

这是公元 8 世纪左右一些古代法国国王的家谱的一部分。在一张图片中：

```txt
Robert I -> Thurimbert -+-> Robert II -> Robert III
                        |
                        +-> Cancor
                        |
                        +-> Landrade  -> Chaudegrand
                        |
                        +-> Ingramm   -> Ermengarde
```

现在我们想知道 Thurimbert 的所有后代：

```mysql
WITH RECURSIVE descendants AS
(
SELECT person
FROM tree
WHERE person='Thurimbert'
UNION ALL
SELECT t.person
FROM descendants d, tree t
WHERE t.parent=d.person
)
SELECT * FROM descendants;
+-------------+
| person      |
+-------------+
| Thurimbert  |
| Robert II   |
| Cancor      |
| Landrade    |
| Ingramm     |
| Robert III  |
| Chaudegrand |
| Ermengarde  |
+-------------+
```

*广度优先排序*意味着我们希望首先看到所有孩子（直系后代），将它们组合在一起，然后才会出现他们自己的孩子。这是查询返回的内容，但您不应该依赖它（根据 SQL 中的常规规则：“没有指定 ORDER BY？没有可预测的顺序！”）。由于直接后代都出现在第一次迭代中，我们只需要在一个新的“级别”列中计算迭代次数，并按它进行排序：

```mysql
WITH RECURSIVE descendants AS
(
SELECT person, 1 as level
FROM tree
WHERE person='Thurimbert'
UNION ALL
SELECT t.person, d.level+1
FROM descendants d, tree t
WHERE t.parent=d.person
)
SELECT * FROM descendants ORDER BY level;
+-------------+-------+
| person      | level |
+-------------+-------+
| Thurimbert  |     1 |
| Robert II   |     2 |
| Cancor      |     2 |
| Landrade    |     2 |
| Ingramm     |     2 |
| Robert III  |     3 |
| Chaudegrand |     3 |
| Ermengarde  |     3 |
+-------------+-------+
```

*深度优先排序*意味着我们希望看到孩子们立即分组在他们的父母之下。为此，我们构建了一个“路径”列并按其排序：

```mysql
WITH RECURSIVE descendants AS
(
SELECT person, CAST(person AS CHAR(500)) AS path
FROM tree
WHERE person='Thurimbert'
UNION ALL
SELECT t.person, CONCAT(d.path, ',', t.person)
FROM descendants d, tree t
WHERE t.parent=d.person
)
SELECT * FROM descendants ORDER BY path;

+-------------+---------------------------------+
| person      | path                            |
+-------------+---------------------------------+
| Thurimbert  | Thurimbert                      |
| Cancor      | Thurimbert,Cancor               |
| Ingramm     | Thurimbert,Ingramm              |
| Ermengarde  | Thurimbert,Ingramm,Ermengarde   |
| Landrade    | Thurimbert,Landrade             |
| Chaudegrand | Thurimbert,Landrade,Chaudegrand |
| Robert II   | Thurimbert,Robert II            |
| Robert III  | Thurimbert,Robert II,Robert III |
+-------------+---------------------------------+
```

我已经在[这里](https://dev.mysql.com/blog-archive/mysql-8-0-labs-recursive-common-table-expressions-in-mysql-ctes-part-two-how-to-generate-series/)提到了 CAST 的必要性：“路径”列的类型由初始 SELECT 确定，并且它必须足够长以适应树中最深叶子的路径：CAST 确保了这一点。正如我们在上面看到的，很容易在 MySQL 中使用“级别”或“路径”列进行模拟。

## 用简单的循环避免计算传递闭包

在 2300 年，地球人满为患，人们被鼓励通过登上这些太空火箭移民到附近的星球：

```mysql
CREATE TABLE rockets
(origin CHAR(20), destination CHAR(20), trip_time INT);
INSERT INTO rockets VALUES
('Earth', 'Mars', 2),
('Mars', 'Jupiter', 3),
('Jupiter', 'Saturn', 4);
```

请注意，地球的统治者故意没有设置任何从这些行星返回地球的方式。

让我们列出所有可以从地球到达的目的地：我们首先找到可以直接到达的行星，然后我们找到从这些第一颗行星可以直接到达的地方，依此类推：

```mysql
WITH RECURSIVE all_destinations AS
(
SELECT destination AS planet
FROM rockets
WHERE origin='Earth' 
UNION ALL
SELECT r.destination
FROM rockets r, all_destinations d
WHERE r.origin=d.planet
)
SELECT * FROM all_destinations;
+---------+
| planet  |
+---------+
| Mars    |
| Jupiter |
| Saturn  |
+---------+
```

现在是 2400 年，地球人口减少了很多，统治者决定带一些移民回来，所以他们安装了从土星到地球的新火箭：

```mysql
INSERT INTO rockets VALUES ('Saturn', 'Earth', 9);
```

让我们重复查询以列出所有可以从地球到达的目的地：

```mysql
WITH RECURSIVE all_destinations AS
(
SELECT destination AS planet
FROM rockets
WHERE origin='Earth' 
UNION ALL
SELECT r.destination
FROM rockets r, all_destinations d
WHERE r.origin=d.planet
)
SELECT * FROM all_destinations;

Recursive query aborted after 1001 iterations. Try increasing @@cte_max_recursion_depth to a larger value.
```

查询似乎永远不会完成，我不得不手动中断它（请注意，从 MySQL 8.0.3 开始，查询将自动中止，这要归功于[@@cte_max_recursion_depth](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_cte_max_recursion_depth)）。为什么会这样？我们的数据有一个循环：从地球开始，我们添加火星，然后是木星，然后是土星，然后是地球（因为新火箭），所以我们又回到了起点——地球。然后我们永远添加火星，依此类推。

设置[max_execution_time](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_max_execution_time)肯定会确保这种失控的查询被自动杀死，但是……我们如何修改查询才能正常工作，即*不*失控？

一个简单的解决方案是，如果行星已经存在，则不将其添加到结果中。这种重复的消除是通过使用 UNION DISTINCT 而不是 UNION ALL 完成的：

```mysql
WITH RECURSIVE all_destinations AS
(
SELECT destination AS planet
FROM rockets
WHERE origin='Earth' 
UNION DISTINCT
SELECT r.destination
FROM rockets r, all_destinations d
WHERE r.origin=d.planet
)
SELECT * FROM all_destinations;
+---------+
| planet  |
+---------+
| Mars    |
| Jupiter |
| Saturn  |
| Earth   |
+---------+
```

## 更复杂的循环避免

所以我们有一个使用 UNION DISTINCT 的很好的查询。让我们稍微扩展一下，以额外显示每次旅行所花费的时间（很天真的想法，对吧？）：

```mysql
WITH RECURSIVE all_destinations AS
(
SELECT destination AS planet, trip_time AS total_time 
FROM rockets
WHERE origin='Earth' 
UNION DISTINCT
SELECT r.destination, d.total_time+r.trip_time
FROM rockets r, all_destinations d
WHERE r.origin=d.planet
)
SELECT * FROM all_destinations;
^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
```

什么？再次它永远不会结束？实际上，请考虑第一个生成的行必须是什么：

- 初始选择：*火星，2*
- 迭代 1：木星，5 (2+3)
- 迭代 2：土星，9 (5+4)
- 迭代 3：地球，18 (9+9)
- 迭代 4：*火星，20* (18+2)

地球->火星->木星->土星->地球->火星行程时间为20，地球->火星行程时间为2；这两次旅行现在形成了两个不同的行，因为它们的不同时间是行的一部分：(Mars, 2) 和 (Mars, 20)。UNION DISTINCT 不再有助于消除循环，因为它只消除*所有*列都匹配的行……因此，将 (Mars, 20) 放入结果中，并导致 (Jupiter, 23) 等等。

SQL 标准为此提供了一个可选的 CYCLE 子句，我们可以很容易地模拟它：解决方案是构建一个“路径”列（如上面的深度/广度第一段），并在我们看到我们正在添加一些东西时中断这已经在路径中。我们不再需要使用 DISTINCT，所以我们使用 ALL 来避免（无意义的）重复消除的开销：

```mysql
WITH RECURSIVE all_destinations AS
(
SELECT destination AS planet, trip_time AS total_time,
CAST(destination AS CHAR(500)) AS path
FROM rockets
WHERE origin='Earth' 
UNION ALL
SELECT r.destination, d.total_time+r.trip_time,
CONCAT(d.path, ',', r.destination)
FROM rockets r, all_destinations d
WHERE r.origin=d.planet
AND FIND_IN_SET(r.destination, d.path)=0
)
SELECT * FROM all_destinations;
```

如果在您的特定用例中，循环被认为是您想要寻找的异常，我们查询的变体将标记它们。我们只需要允许循环输入我们的结果并宣传自己（使用新的“is_cycle”列），但我们仍然必须通过不迭代 is_cycle=1 的行来防止查询永远运行：

```mysql
WITH RECURSIVE all_destinations AS
(
SELECT destination AS planet, trip_time AS total_time,
CAST(destination AS CHAR(500)) AS path, 0 AS is_cycle
FROM rockets
WHERE origin='Earth' 
UNION ALL
SELECT r.destination, d.total_time+r.trip_time,
CONCAT(d.path, ',', r.destination),
FIND_IN_SET(r.destination, d.path)!=0
FROM rockets r, all_destinations d
WHERE r.origin=d.planet
AND is_cycle=0
)
SELECT * FROM all_destinations;
+---------+------------+--------------------------------+----------+
| planet  | total_time | path                           | is_cycle |
+---------+------------+--------------------------------+----------+
| Mars    |          2 | Mars                           |        0 |
| Jupiter |          5 | Mars,Jupiter                   |        0 |
| Saturn  |          9 | Mars,Jupiter,Saturn            |        0 |
| Earth   |         18 | Mars,Jupiter,Saturn,Earth      |        0 |
| Mars    |         20 | Mars,Jupiter,Saturn,Earth,Mars |        1 |
+---------+------------+--------------------------------+----------+
```

“is_cycle”列中的“1”将我们指向循环。



## 附录

https://dev.mysql.com/blog-archive/mysql-8-0-1-recursive-common-table-expressions-in-mysql-ctes-part-four-depth-first-or-breadth-first-traversal-transitive-closure-cycle-avoidance/

```mysql
CREATE TABLE tree (person CHAR(20), parent CHAR(20));
INSERT INTO tree VALUES
('Robert I', NULL),
('Thurimbert', 'Robert I'),
('Robert II', 'Thurimbert'),
('Cancor', 'Thurimbert'),
('Landrade', 'Thurimbert'),
('Ingramm', 'Thurimbert'),
('Robert III', 'Robert II'),
('Chaudegrand', 'Landrade'),
('Ermengarde', 'Ingramm');

alter table tree add index idx_person(person);
alter table tree add index idx_parent(parent);

desc tree;
SELECT * FROM tree;
-- 广度优先排序
WITH RECURSIVE descendants AS
(
SELECT person
FROM tree
WHERE person='Thurimbert'
UNION ALL
SELECT t.person
FROM descendants d, tree t
WHERE t.parent=d.person
)
SELECT * FROM descendants;

WITH recursive descendants AS
(
	SELECT person, 1 as `level` FROM tree WHERE person = 'Thurimbert'
	UNION ALL
	SELECT
		t.person, d.level+1
	FROM
		descendants d, tree t
	WHERE
		d.person = t.parent
)
SELECT * FROM descendants;

-- 深度优先排序意味着我们希望看到孩子们立即分组在他们的父母之下。为此，我们构建了一个“路径”列并按其排序：
WITH RECURSIVE descendants AS
(
SELECT person, CAST(person AS CHAR(500)) AS path
FROM tree
WHERE person='Thurimbert'
UNION ALL
SELECT t.person, CONCAT(d.path, ',', t.person)
FROM descendants d, tree t
WHERE t.parent=d.person
)
SELECT * FROM descendants ORDER BY path;

-- 用简单的循环避免计算传递闭包CREATE TABLE rockets
CREATE TABLE rockets
(origin CHAR(20), destination CHAR(20), trip_time INT);
ALTER table rockets add index idx_origin(origin);
ALTER table rockets add index idx_destination(destination);
INSERT INTO rockets VALUES
('Earth', 'Mars', 2),
('Mars', 'Jupiter', 3),
('Jupiter', 'Saturn', 4);

-- 让我们列出所有可以从地球到达的目的地：我们首先找到可以直接到达的行星
WITH RECURSIVE all_destinations AS
(
SELECT destination AS planet
FROM rockets
WHERE origin='Earth' 
UNION ALL
SELECT r.destination
FROM rockets r, all_destinations d
WHERE r.origin=d.planet
)
SELECT * FROM all_destinations;
-- 有加了一条了
INSERT INTO rockets VALUES ('Saturn', 'Earth', 9);

WITH RECURSIVE all_destinations AS
(
SELECT destination AS planet
FROM rockets
WHERE origin='Earth' 
UNION ALL
SELECT r.destination
FROM rockets r, all_destinations d
WHERE r.origin=d.planet
)
SELECT * FROM all_destinations;
-- 查询似乎永远不会完成，我不得不手动中断它（请注意，从 MySQL 8.0.3 开始，查询将自动中止，这要归功于@@cte_max_recursion_depth）

-- 使用 UNION DISTINCT 而不是 UNION ALL
WITH RECURSIVE all_destinations AS
(
SELECT destination AS planet
FROM rockets
WHERE origin='Earth' 
UNION DISTINCT
SELECT r.destination
FROM rockets r, all_destinations d
WHERE r.origin=d.planet
)
SELECT * FROM all_destinations;

-- 所以我们有一个使用 UNION DISTINCT 的很好的查询。让我们稍微扩展一下，以额外显示每次旅行所花费的时间
WITH RECURSIVE all_destinations AS
(
SELECT destination AS planet, trip_time AS total_time 
FROM rockets
WHERE origin='Earth' 
UNION DISTINCT
SELECT r.destination, d.total_time+r.trip_time
FROM rockets r, all_destinations d
WHERE r.origin=d.planet
)
SELECT * FROM all_destinations;
-- SQL 标准为此提供了一个可选的 CYCLE 子句，我们可以很容易地模拟它
WITH RECURSIVE all_destinations AS
(
SELECT destination AS planet, trip_time AS total_time,
CAST(destination AS CHAR(500)) AS path
FROM rockets
WHERE origin='Earth' 
UNION ALL
SELECT r.destination, d.total_time+r.trip_time,
CONCAT(d.path, ',', r.destination)
FROM rockets r, all_destinations d
WHERE r.origin=d.planet
AND FIND_IN_SET(r.destination, d.path)=0
)
SELECT * FROM all_destinations;

WITH RECURSIVE all_destinations AS
(
SELECT destination AS planet, trip_time AS total_time,
CAST(destination AS CHAR(500)) AS path, 0 AS is_cycle
FROM rockets
WHERE origin='Earth' 
UNION ALL
SELECT r.destination, d.total_time+r.trip_time,
CONCAT(d.path, ',', r.destination),
FIND_IN_SET(r.destination, d.path)!=0
FROM rockets r, all_destinations d
WHERE r.origin=d.planet
AND is_cycle=0
)
SELECT * FROM all_destinations;

```

