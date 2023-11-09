# MySQL JSON 特性

## 一、常用函数

引用MySQL 文档：https://dev.mysql.com/doc/refman/5.7/en/json.html

![img](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/04/1684605-20190712145547380-1472901871.png)

## 二、开始基础使用

1.该[`JSON_TYPE()`](https://dev.mysql.com/doc/refman/5.7/en/json-attribute-functions.html#function_json-type)函数需要一个 JSON 参数并尝试将其解析为 JSON 值。如果有效，则返回值的 JSON 类型，否则产生错误：

```mysql
mysql> SELECT JSON_TYPE('["a", "b", 1]');
+----------------------------+
| JSON_TYPE('["a", "b", 1]') |
+----------------------------+
| ARRAY                      |
+----------------------------+

mysql> SELECT JSON_TYPE('"hello"');
+----------------------+
| JSON_TYPE('"hello"') |
+----------------------+
| STRING               |
+----------------------+

mysql> SELECT JSON_TYPE('hello');
ERROR 3146 (22032): Invalid data type for JSON data in argument 1
to function json_type; a JSON string or JSON type is required.
```

```mysql
mysql> SELECT JSON_ARRAY('a', 1, NOW());
+----------------------------------------+
| JSON_ARRAY('a', 1, NOW())              |
+----------------------------------------+
| ["a", 1, "2015-07-27 09:43:47.000000"] |
+----------------------------------------+
```

[`JSON_OBJECT()`](https://dev.mysql.com/doc/refman/5.7/en/json-creation-functions.html#function_json-object)获取一个（可能为空的）键值对列表并返回一个包含这些对的 JSON 对象：

```mysq
mysql> SELECT JSON_OBJECT('key1', 1, 'key2', 'abc');
+---------------------------------------+
| JSON_OBJECT('key1', 1, 'key2', 'abc') |
+---------------------------------------+
| {"key1": 1, "key2": "abc"}            |
+---------------------------------------+
```

[`JSON_MERGE()`](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-merge)接受两个或多个 JSON 文档并返回组合结果：

```mysql
mysql> SELECT JSON_MERGE('["a", 1]', '{"key": "value"}');
+--------------------------------------------+
| JSON_MERGE('["a", 1]', '{"key": "value"}') |
+--------------------------------------------+
| ["a", 1, {"key": "value"}]                 |
+--------------------------------------------+
```

2.JSON 值可以分配给用户定义的变量：

```mysql
mysql> SET @j = JSON_OBJECT('key', 'value');
mysql> SELECT @j;
+------------------+
| @j               |
+------------------+
| {"key": "value"} |
+------------------+
```

但是，用户定义的变量不能是 `JSON`数据类型，因此虽然 `@j`在前面的示例中看起来像 JSON 值并且具有与 JSON 值相同的字符集和排序规则，但它没有 *数据*`JSON`类型。相反，结果 from [`JSON_OBJECT()`](https://dev.mysql.com/doc/refman/5.7/en/json-creation-functions.html#function_json-object)在分配给变量时被转换为字符串。

通过转换 JSON 值生成的字符串有一个字符集`utf8mb4`和一个排序规则 `utf8mb4_bin`：

```mysql
mysql> SELECT CHARSET(@j), COLLATION(@j);
+-------------+---------------+
| CHARSET(@j) | COLLATION(@j) |
+-------------+---------------+
| utf8mb4     | utf8mb4_bin   |
+-------------+---------------+
```

区分大小写也适用于 JSON `null`、`true`和 `false`文字，它们必须始终以小写形式编写：

```mysql
mysql> SELECT JSON_VALID('null'), JSON_VALID('Null'), JSON_VALID('NULL');
+--------------------+--------------------+--------------------+
| JSON_VALID('null') | JSON_VALID('Null') | JSON_VALID('NULL') |
+--------------------+--------------------+--------------------+
|                  1 |                  0 |                  0 |
+--------------------+--------------------+--------------------+

mysql> SELECT CAST('null' AS JSON);
+----------------------+
| CAST('null' AS JSON) |
+----------------------+
| null                 |
+----------------------+
1 row in set (0.00 sec)

mysql> SELECT CAST('NULL' AS JSON);
ERROR 3141 (22032): Invalid JSON text in argument 1 to function cast_as_json:
"Invalid value." at position 0 in 'NULL'.
```

JSON 文字的区分大小写与 SQL `NULL`、`TRUE`和 `FALSE`文字的区分大小写不同，后者可以用任何字母大小写：

```mysql
mysql> SELECT ISNULL(null), ISNULL(Null), ISNULL(NULL);
+--------------+--------------+--------------+
| ISNULL(null) | ISNULL(Null) | ISNULL(NULL) |
+--------------+--------------+--------------+
|            1 |            1 |            1 |
+--------------+--------------+--------------+
```

将其作为 JSON 对象插入到 `facts`表中的一种方法是使用 MySQL [`JSON_OBJECT()`](https://dev.mysql.com/doc/refman/5.7/en/json-creation-functions.html#function_json-object)函数。在这种情况下，您必须使用反斜杠转义每个引号字符，如下所示：

```mysql
mysql> INSERT INTO facts VALUES
     >   (JSON_OBJECT("mascot", "Our mascot is a dolphin named \"Sakila\"."));
```

如果您将值作为 JSON 对象文字插入，则这不会以相同的方式工作，在这种情况下，您必须使用双反斜杠转义序列，如下所示：

```mysql
mysql> INSERT INTO facts VALUES
     >   ('{"mascot": "Our mascot is a dolphin named \\"Sakila\\"."}');
```

## 三、搜索和修改 JSON 值

JSON 路径表达式在 JSON 文档中选择一个值。

路径表达式对于提取部分 JSON 文档或修改 JSON 文档的函数很有用，以指定在该文档中的哪个位置进行操作。例如，以下查询从 JSON 文档中提取具有 `name`键的成员的值：

```mysql
mysql> SELECT JSON_EXTRACT('{"id": 14, "name": "Aztalan"}', '$.name');
+---------------------------------------------------------+
| JSON_EXTRACT('{"id": 14, "name": "Aztalan"}', '$.name') |
+---------------------------------------------------------+
| "Aztalan"                                               |
+---------------------------------------------------------+
```

路径语法使用一个前导`$`字符来表示所考虑的 JSON 文档，可选地后跟选择器，这些选择器依次指示文档的更具体部分：

- 后跟键名的句点使用给定键命名对象中的成员。如果不带引号的名称在路径表达式中不合法（例如，如果它包含空格），则必须在双引号内指定键名称。

- `[*`N`*]`附加到*`path`*选择一个数组的 a 名称的值在*`N`* 数组中的位置。数组位置是从零开始的整数。如果*`path`*不选择数组值，则*`path`*[0] 的计算结果与 相同 *`path`*：

  ```mysql
  mysql> SELECT JSON_SET('"x"', '$[0]', 'a');
  +------------------------------+
  | JSON_SET('"x"', '$[0]', 'a') |
  +------------------------------+
  | "a"                          |
  +------------------------------+
  1 row in set (0.00 sec)
  ```

- 路径可以包含`*`或 `**`通配符：
  - `.[*]`计算 JSON 对象中所有成员的值。
  - `[*]`计算 JSON 数组中所有元素的值。
  - `*`prefix`****`suffix`*` 计算以命名前缀开头并以命名后缀结尾的所有路径。
- 文档中不存在的路径（评估为不存在的数据）评估为`NULL`。

让我们`$`用三个元素来引用这个 JSON 数组：

```mysql
[3, {"a": [5, 6], "b": 10}, [99, 100]]
```

然后：

- `$[0]`评估为`3`。
- `$[1]`评估为`{"a": [5, 6], "b": 10}`。
- `$[2]`评估为`[99, 100]`。
- `$[3]`计算结果为`NULL` （它指的是第四个数组元素，它不存在）。

因为`$[1]`和`$[2]` 计算为非标量值，它们可以用作选择嵌套值的更具体的路径表达式的基础。例子：

- `$[1].a`评估为`[5, 6]`。
- `$[1].a[1]`评估为 `6`。
- `$[1].b`评估为 `10`。
- `$[2][0]`评估为 `99`。

如前所述，如果未引用的键名在路径表达式中不合法，则必须引用命名键的路径组件。让我们`$`参考这个值：

```mysql
{"a fish": "shark", "a bird": "sparrow"}
```

这两个键都包含一个空格，并且必须用引号引起来：

- `$."a fish"`评估为 `shark`。
- `$."a bird"`评估为 `sparrow`。

使用通配符的路径评估为可以包含多个值的数组：

```mysql
mysql> SELECT JSON_EXTRACT('{"a": 1, "b": 2, "c": [3, 4, 5]}', '$.*');
+---------------------------------------------------------+
| JSON_EXTRACT('{"a": 1, "b": 2, "c": [3, 4, 5]}', '$.*') |
+---------------------------------------------------------+
| [1, 2, [3, 4, 5]]                                       |
+---------------------------------------------------------+
mysql> SELECT JSON_EXTRACT('{"a": 1, "b": 2, "c": [3, 4, 5]}', '$.c[*]');
+------------------------------------------------------------+
| JSON_EXTRACT('{"a": 1, "b": 2, "c": [3, 4, 5]}', '$.c[*]') |
+------------------------------------------------------------+
| [3, 4, 5]                                                  |
+------------------------------------------------------------+
```

在以下示例中，路径`$**.b` 计算为多个路径 (`$.a.b`和 `$.c.b`) 并生成匹配路径值的数组：

```mysql
mysql> SELECT JSON_EXTRACT('{"a": {"b": 1}, "c": {"b": 2}}', '$**.b');
+---------------------------------------------------------+
| JSON_EXTRACT('{"a": {"b": 1}, "c": {"b": 2}}', '$**.b') |
+---------------------------------------------------------+
| [1, 2]                                                  |
+---------------------------------------------------------+
```

