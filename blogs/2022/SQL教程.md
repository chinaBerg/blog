# SQL入门

> 愣锤 2022/04/07

- 数据库（Database）是按照数据结构来组织、存储和管理数据的仓库
- 关系型数据库，是建立在关系模型基础上的数据库，借助于集合代数等数学概念和方法来处理数据库中的数据
- 关系数据库管理系统(Relational Database Management System)的特点：
    - 数据以表格的形式出现
    - 每行为各种记录名称
    - 每列为记录名称所对应的数据域
    - 许多的行和列组成一张表单
    - 若干的表单组成database

### 基本命令

- 创建数据库

```bash
CREATE DATABASE 数据库名;
```

注意的是，`SQL`的关键词大小写不敏感。

- 查看所有数据库

```bash
# 注意末尾有分号
SHOW DATABASES;
```

![image](https://note.youdao.com/yws/res/20989/5C4E1AF1DE5543E2983BF5A1189BD299)

- 删除数据库

```bash
DROP DATABASE 数据库名;
```

- 选择数据库

```bash
USE 数据库名;
```

- 查看指定数据库的所有表

```bash
# 前提是先使用USE先选择数据库
SHOW TABLES;
```

### 数据库表

- 创建数据库表

如果user表不存在则创建创建user表，`AUTO_INCREMENT`值自增；`PRIMARY KEY`指定主键的字段；`NOT NULL`表示在操作数据库时不允许字段为`null`，否则报错；

```bash
CREATE TABLE IF NOT EXISTS `user`(
   `uid` INT UNSIGNED AUTO_INCREMENT,
   `username` VARCHAR(100) NOT NULL,
   PRIMARY KEY ( `uid` )
)ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

- 删除数据库表

```bash
DROP TABLE 表名;
```

### 插入数据

- 往表中插入数据

```bash
INSERT INTO 表名 (field1, field2, ...fieldN) VALUES (value1, value2,...valueN);
```

多个数据字段使用逗号隔开，如果值是字符串，则使用单引号或双引号包裹。自增的值不需要赋值。

### 查询数据

- 查询表所有列数据

`*`表示所有列字段

```bash
SELECT * FROM user;
```

- 查询表中指定列的所有数据

```bash
# 查询user表的username列的所有数据
SELECT username FROM user;

# 查询user表的多个列的所有数据
SELECT username, otherColumn1, otherColumn2 FROM user;
```

- `WHERE`关键字可以用于指定查询条件

```bash
# 查询user表中username为make的这条记录所有字段信息
SELECT * FROM user WHERE username='make';
```
![image](https://note.youdao.com/yws/res/21027/ED48DE085BF048AE9D6484ABC9C31C9E)

- `AND`和`OR`可以在`WHERE`后面使用多个表达式，表示`与`和`或`的逻辑

```bash
# 选取books表中type为666且author为施耐庵的记录数据集合
SELECT * FROM books WHERE type=666 AND author='施耐庵';

# 选取books表中type为666或者author为make的记录数据集合
SELECT * FROM books WHERE type=666 OR author='make';
```

- 表达式操作符可以为`=`、`!=`、`>`、`<`、`>=`、`<=`。

```bash
# 选取books表中id大于2的数据记录集合
SELECT * FROM books WHERE id > 2;
```

- `LIMIT`指定查询数据，`OFFSET`指定查询位置

现有一张`books`表的数据如下

![image](https://note.youdao.com/yws/res/21031/A012AD928D934875A378A71B987C9528)

```bash
# 从books表中下标位置为2开始，查询2条记录的数据
SELECT * FROM books LIMIT 2 OFFSET 2;
```

![image](https://note.youdao.com/yws/res/21038/D6DFC76C68B94C70A2DB94546B23DC15)

- `LIKE`关键词模糊查找

```bash
# LIKE语法
SELECT field1, field2,...fieldN 
FROM table_name
WHERE field1 LIKE condition1 [AND [OR]] filed2 = 'somevalue'
```

`LIKE`后面的表达式中 `%` 表示任意字符，有点类似于`linux`中的`*`。

例如我们查询`books`表中所有`bookname`为`西游`开头的记录:

```bash
SELECT * FROM books WHERE username LIKE '西游%';
```

- `UNION`连接多个SELECT表达式

```bash
# 查询level表中level字段小于2和大于5的记录
SELECT * FROM level WHERE level < 2 UNION SELECT * FROM level WHERE level > 5;
```

![image](https://note.youdao.com/yws/res/21085/9899D71B203D43D8986D7D75A6AF6E2E)

默认情况下，UNION得到的交集会去除重复的数据记录，也可以显示的指定是否去除：

```bash
# DISTINCT字段表示去除重复记录，也是默认值
SELECT * FROM level WHERE level < 2 UNION DISTINCT SELECT * FROM level WHERE level > 5;

# ALL字段表示不去除重复记录
SELECT * FROM level WHERE level < 2 UNION ALL SELECT * FROM level WHERE level > 5;
```

### 更新数据

- 将指定列的所有数据进行更新

```bash
UPDATE 表名 SET field1=value1, field2=value2;
```

- `WHERE`根据筛选条件进行更新

```bash
UPDATE 表名 SET field1=value1, field2=value2 WHERE 表达式;
```

### 删除数据

- 删除表中符合条件的记录数据

```bash
# 从books表中删除id为6的记录
DELETE FROM books WHERE id=6;
```

- 删除表中全部数据

```bash
# 删除user表中全部记录
DELETE FROM user;
```

### 查询结果排序

- 升序，`ORDER BY`默认是升序，默认值是`ASC`

```bash
# 查询level表中所有记录，按field1升序排序
SELECT * FROM level ORDER BY field1;
```

- 降序，`ORDER BY DESC`指定降序排序

```bash
# 查询level表中所有记录，按field1降序排序
SELECT * FROM level ORDER BY field1 DESC;
```

### 分组

`GROUP BY`可以将数据表按指定字段分组。例如下面将数据表按照`name`分组并统计每个`name`有多少条数据：

```bash
SELECT name, COUNT(*) FROM level  GROUP BY name;
```

![image](https://note.youdao.com/yws/res/21111/4F1749C50B1647AE98F9DF6E29A5E39F)

类似`COUNT()`将在后面函数部分讲解。

### 多表联查

- `JOIN`可以用于在多张表中查询数据，获取两个表中字段匹配关系的记录。类似于交集。

```bash
# 查询存在于user表中且在level表中有相同level值的记录
# 查询返回的fields为user.uid、user.author、user.level、level.name、level.description
select a.uid, a.author, a.level, b.name as levelName, b.description as levelDesc  from user a join level b on a.level = b.level;
```

需要注意的是`AS`关键词可以对返回的`field`重命名，即起的别名

![image](https://note.youdao.com/yws/res/21133/1BC17426D7314A02A6C6E7CCF9B16C20)

- `LEFT JOIN` 获取左表符合条件的所有记录，即使右表没有对应匹配的记录

```bash
# 对于在b表中没有查到的记录的数据，则以null填充
select a.uid, a.author, a.level, b.name as levelName  from user a left join level b on a.level = b.level;
```

![image](https://note.youdao.com/yws/res/21136/45CADB7E36594C1C9A8CB42A2CA948C3)

- `RIGHT JOIN` 获取右表符合条件的所有记录，即使左表没有对应匹配的记录

```bash
select a.uid, a.author, a.level, b.name as levelName  from user a right join level b on a.level = b.level;
```

![image](https://note.youdao.com/yws/res/21142/43A9D0492CE24E68AFCF0D8326BBD7B7)

### 条件表达式使用正则

除了使用`LIKE`关键词进行模糊匹配，`WHERE`的条件表达式也可以使用正则表达式进行查询，关键词是`REGEXP`:

```bash
# 查询books表中bookname是西游开头的所有记录
select * from books where bookname regexp '^西游';
```