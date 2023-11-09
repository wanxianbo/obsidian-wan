### MySQL用到的函数

`FIND_IN_SET(str,strlist)`

str 要查询的字符串

strlist 字段名 参数以”,”分隔 如 (1,2,6,8,10,22)

查询字段(strlist)中包含(str)的结果，返回结果为null或记录

例子1：

> SELECT FIND_IN_SET('b','a,b,c');

结果：2

> select FIND_IN_SET('2', '1，2'); 返回2
> select FIND_IN_SET('6', '1'); 返回0 strlist中不存在str，所以返回0

**find_in_set()和in的区别：**

弄个测试表来说明两者的区别

```mysql
CREATE TABLE `tb_test` (
 `id` int(8) NOT NULL auto_increment,
 `name` varchar(255) NOT NULL,
 `list` varchar(255) NOT NULL,
 PRIMARY KEY (`id`)
);
INSERT INTO `tb_test` VALUES (1, 'name', 'daodao,xiaohu,xiaoqin');
INSERT INTO `tb_test` VALUES (2, 'name2', 'xiaohu,daodao,xiaoqin');
INSERT INTO `tb_test` VALUES (3, 'name3', 'xiaoqin,daodao,xiaohu');
```

原来以为mysql可以进行这样的查询：

```mysql
SELECT id,name,list from tb_test WHERE 'daodao' IN(list); -- (一)
```

![img](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2021/01/20210120180110.png)

实际上这样是不行的， 这样只有当list字段的值等于'daodao'时（和IN前面的字符串完全匹配），查询才有效，否则都得不到结果，即使'daodao'真的在list中。

再来看看这个：

```mysql
SELECT id,name,list from tb_test WHERE 'daodao' IN ('libk', 'zyfon', 'daodao'); -- (二)
```

![img](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2021/01/20210120180133.png)

这样是可以的。

这两条到底有什么区别呢？为什么第一条不能取得正确的结果，而第二条却能取得结果。原因其实是（一）中 (list) list是变量， 而（二）中 ('libk', 'zyfon', 'daodao')是常量。

所以如果要让（一）能正确工作，需要用

```mysql
find_in_set():

SELECT id,name,list from tb_test WHERE FIND_IN_SET('daodao',list); -- (一)的改进版
```

**总结：
**

所以如果list是常量，则可以直接用IN， 否则要用find_in_set()函数。

也就是这两个sql是查询的效果是相同的：

```mysql
SELECT * from C_PURCHASINGMASTERDATA where FIND_IN_SET(EKGRP,'C54,C02,C14,C60,C06,C61,C53,C51,C12,C08,C03,C07')
SELECT * from C_PURCHASINGMASTERDATA where EKGRP in ('C54','C02','C14','C60','C06','C61','C53','C51','C12','C08','C03','C07')
```

但是如果第二句sql里面的值是传入sql的一个变量字段，那么第二句sql就不好使了。要以实际情况决定用in还是用 find_in_set()函数 。

**find_in_set()和like的区别：**

主要的区别就是like是广泛的模糊查询，而 find_in_set() 是精确匹配，并且字段值之间用‘,'分开。

现在想查询拥有角色编号为2的用户，用like关键字查询：

```mysql
SELECT userid,username,userrole 角色 FROM `user` WHERE userrole LIKE '%2%';
```

结果：

![img](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2021/01/20210120180303.png)

用 find_in_set() 查询：

```mysql
SELECT userid,username,userrole 角色 FROM `user` WHERE find_in_set('2',userrole)
```

结果：

![img](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2021/01/20210120180325.png)

显然用 `find_in_set() `查询得到的结果才是我们想要的结果。所以他俩的

主要的区别就是like是广泛的模糊查询；而 `find_in_set() `是精确匹配，并且字段值之间用‘,'分开，Find_IN_SET查询的结果要小于like查询的结果。
#### 三、CONVERT 用法

##### 	3.1 CONVERT(s USING cs)  函数将字符串 s 的字符集变成 sc

```mysql
SELECT CHARSET('ABC')
->utf-8    
# CHARSET() 查看字符串的字符集
SELECT CHARSET(CONVERT('ABC' USING gbk))
->gbk
```

​	3.2 例子 使用中文拼音进行排序

```mysql
select
    *
from
    `user` 
order by
    convert(trim(`name`) using gbk) collate gbk_chinese_ci asc
```

```mysql
# 实现中文首字母排序
DELIMITER $$
CREATE FUNCTION `fristPinyin`(P_NAME VARCHAR(255)) RETURNS varchar(255) CHARSET utf8  
DETERMINISTIC
BEGIN 
    DECLARE V_RETURN VARCHAR(255);
    DECLARE V_BOOL INT DEFAULT 0;
          DECLARE FIRST_VARCHAR VARCHAR(1);
 
    SET FIRST_VARCHAR = left(CONVERT(P_NAME USING gbk),1);
    SELECT FIRST_VARCHAR REGEXP '[a-zA-Z]' INTO V_BOOL;
    IF V_BOOL = 1 THEN
      SET V_RETURN = FIRST_VARCHAR;
    ELSE
      SET V_RETURN = ELT(INTERVAL(CONV(HEX(left(CONVERT(P_NAME USING gbk),1)),16,10),   
          0xB0A1,0xB0C5,0xB2C1,0xB4EE,0xB6EA,0xB7A2,0xB8C1,0xB9FE,0xBBF7,   
          0xBFA6,0xC0AC,0xC2E8,0xC4C3,0xC5B6,0xC5BE,0xC6DA,0xC8BB,  
          0xC8F6,0xCBFA,0xCDDA,0xCEF4,0xD1B9,0xD4D1),   
      'A','B','C','D','E','F','G','H','J','K','L','M','N','O','P','Q','R','S','T','W','X','Y','Z');  
    END IF;
    RETURN V_RETURN;
END$$
DELIMITER;


#例子
SELECT fristPinyin(`name`) AS fristPinyin,id,age,`name`
FROM `user`
ORDER BY fristPinyin
```

