# SQL入门

> 愣锤 2022/04/11

> version 8.0.16

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

如果user表不存在则创建创建user表，`AUTO_INCREMENT`表示值自增；`PRIMARY KEY`指定主键的字段；`NOT NULL`表示在操作数据库时不允许字段为`null`，否则报错；

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

- 查看创建的数据表结构

```bash
desc 表名;
```

![image](https://note.youdao.com/yws/res/21364/9FF8611776D343959BDE7A1C8DD1DFD1)

### 数据类型

`MySQL` 支持所有标准 `SQL` 数值数据类型，像`日期时间`、`字符串`、`数值`等。

类型 | 说明 | 范围
---|---|---|
CHAR | 定长字符串 | `0-255`字节
VARCHAR | 变长字符串 | `0-65535`字节
TINYTEXT | 短文本字符串 | `0-255`字节
TEXT | 长文本字符串 | `0-65535`字节
TINYINT | 小整数值 | `(-128，127)`
INT | 大整数值 | `(-2 147 483 648，2 147 483 647)`
DATE | 日期值 | `1000-01-01/9999-12-31`
DATETIME | 日前时间值 | `1000-01-01 00:00:00/9999-12-31 23:59:59`


需要注意的是`char(n)` 和 `varchar(n)` 括号中的 `n` 代表字符的个数，不是字节数。

其他数据类型可以参考[文档](https://www.runoob.com/mysql/mysql-data-types.html)

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

### 事务

- 在 MySQL 中只有使用了 Innodb 数据库引擎的数据库或表才支持事务。
- 事务处理可以用来维护数据库的完整性，保证成批的 SQL 语句要么全部执行，要么全部不执行。
- 事务用来管理 insert,update,delete 语句

事务的主要作用是保证一组数据库操作都成功执行，不然如果有一步数据操作出错了会导致不完整性。举个例子，当你删除一个用户时，同时需要用户的文章信息、登录数据等等时，要确保删除用户时的其他删除操作也必须成功。

使用使用主要是三步，`BEGIN;`开始一个事务，`ROLLBACK;`可以回滚事物，`COMMIT;`事务确认。

```bash
# 开始事务
begin;

# 一些数据库操作
# 此时所有数据操作并未被真正写入数据库
insert into books (bookname, type, author) values ('深入浅出Vue.js', 123, '刘博文');
insert into books (bookname, type, author) values ('new book', 123, 'make');
insert into books (bookname, type, author) values ('new book2', 123, 'make');

# 事物确认，此时所有数据才被全部写入
commit;
```

![image](https://note.youdao.com/yws/res/21166/653ADE7E5B2248069D5C85A60ABED259)


### ALTER更新数据表

`ALTER`的作用是修改数据表名或者字段。

- 向数据表增加列

```bash
# 向user表中新增newField1列，类似为INT
alter table user add newField1 int;
```

- 修改数据表字段的名称和类型

`MODIFY`作用是修改字段类型，`CHANGE`可以同时修改名称和类型。

```bash
# 将user表的newField1字段的类型修改为varchar(64)
alter table user modify newField1 varchar(64);

# 将user表的newField1字段名称修改为new_field1，类型修改为varchar(128)
alter table user change newField1 new_field1 varchar(128);
```

注意：`alter`修改类型时覆盖操作，不是增量覆盖，因此修改类型时要加上原先所有的：

```bash
# 举个例子，原先的uid字段如下
`uid` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '用户ID',

# 在使用alter修改类型时，
# 哪怕仅仅改一个数据类型为int，也需要把后面的NOT NULL等类型携带上
alter table user modify uid int NOT NULL AUTO_INCREMENT COMMENT '用户ID'
```

- `DROP`移除字段

```bash
# 移除user表的new_field1字段
alter table user drop new_field1;
```

- 设置/删除字段的默认值

`alter`操作时如果没有指定默认值，则默认值是`NULL`，可以通过下面的例子删除/设置字段的默认值:

```bash
# 设置user表的new_field字段的默认值为100
# 注意，alter是用了两次
alter table user alter new_field set default 100;

# 移除user表的new_field字段的默认值
alter table user alter new_field drop default;
```

- 修改数据表的名称

```bash
# 我们先创建一个数据表
create table new_table(uid int);

# 修改new_table数据表的名称为new_table2
alter table new_table rename to new_table2;
```

![image](https://note.youdao.com/yws/res/21313/95641B27DC014252ACD5936C463CB7ED)


### 索引

索引分为`主键索引`、`普通索引`、`唯一索引`、`全文索引`，查看数据表的索引情况的命令是`show index from 数据表名`，下图展示了我的user表的索引情况：

![image](https://note.youdao.com/yws/res/21320/CB389C7D40AF4248B41651F01DC547EB)

- 主键索引

主键索引是一种特殊的唯一索引，不允许有NULL值。一般在创建表的时候指定主键索引，一个表只能有一个主键。

```bash
# 创建demo_table表，
# 通过PRIMARY KEY指定为主键
# 通过unique指定为唯一索引
# 通过index可以指定为普通索引
create table demo_table(
  id int not null primary key,
  username varchar(64) unique
) engine=innodb;
```

![image](https://note.youdao.com/yws/res/21338/9F09B705041B488787B69BA37EF92747)

- 唯一索引

唯一索引对应的列值必须唯一，但可以为NULL，如果是组合索引，则列值的组合必须唯一。

可以通过`alter table 表名 add unique (列名);`添加唯一索引。`alter table 表名 add unique (列1, 列2);`创建唯一组合索引。

- 普通索引

普通索引是基本的索引，没有限制。创建普通索引的方式是`alter table 表名 add index 索引名称 (列名);`，创建普通组合索引的方式是`alter table 表名 add index 索引名称 (列1, 列2);`

- 全文索引

```bash
# 给demo_table2先增加一列text_field1
alter table demo_table2 add text_field1 text;

# demo_table2表的text_field1字段添加全文索引
alter table demo_table2 add fulltext(text_field1);
```

- 删除索引

索引一经创建便不可修改，如果要修改则需要删除重建。

```bash
drop index 索引名称 on 表名;
```

注意：索引是一种数据结构，索引可以提升检索速度，但是会额外占用磁盘工具，降低写的速度。因此，不要过度索引。

[参考文章：MySQL索引优化看这篇文章就够了！作者：良月柒](https://zhuanlan.zhihu.com/p/61687047)

### 导出

- 导出某个表的数据

```bash
# 将books表中所有数据导出到./books.sql文件中
select * from books into outfile '/tmp/books.sql';
```

命令运行后你可能遇到如下错误导致无法导出：

```bash
ERROR 1290 (HY000): The MySQL server is running with the --secure-file-priv option so it cannot execute this statement
```

![image](https://note.youdao.com/yws/res/21185/F3F6AD2F9E0A4038BEACE9777A960F6C)

这个错误是说数据库的`secure-file-priv`设置不允许导入导出数据。我们先查看一下我们的数据库`secure-file-priv`配置情况:

```bash
# 终端运行
show variables like '%secure%';
```

![image](https://note.youdao.com/yws/res/21181/9F270583388F47D3AE711AB7B5163C54)

`secure-file-priv`值为`NULL`说明不允许导入导出，没有具体值时表示不对导入导出做限制，有具体值表示只允许在指定路径导入导出。

解决办法大家可以参考[这篇文章](https://zhuanlan.zhihu.com/p/476381922)

问题解决后，再次运行导出数据的命令，可以看到在`/tmp`目录下已经生成了`books.sql`文件，文件内容如下:

```bash
1	资本论	123	make
2	西游记	123	吴承恩
3	三国演义	123	罗贯中
4	水浒传	123	施耐庵
5	红楼梦	123	曹雪芹
7	深入浅出Node.js	123	朴灵
8	深入浅出Vue.js	123	刘博文
9	深入浅出Vue.js2	123	刘博文
10	深入浅出Vue.js3	123	刘博文
11	深入浅出Vue.js4	123	刘博文
```

- 导出整个数据库

注意，导出的是整个数据库所有表的表和数据。导出的表结构可以在其他地方导入直接创建表和数据。

```bash
# 将指定数据库所有表的格式导出到express-blog.sql文件
# 直接终端输入，不需要先mysql登录数据库
mysqldump -u root -p 要导出的数据库名 > express-blog.sql
```

紧接着会要求输入数据库密码，输入完毕后，可以看到导出了一个`express-blog.sql`文件，其内容如下:

```sql
-- MySQL dump 10.13  Distrib 8.0.16, for macos10.14 (x86_64)
--
-- Host: localhost    Database: express-blog
-- ------------------------------------------------------
-- Server version	8.0.16

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
 SET NAMES utf8mb4 ;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

--
-- Table structure for table `books`
--

DROP TABLE IF EXISTS `books`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
 SET character_set_client = utf8mb4 ;
CREATE TABLE `books` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `bookname` varchar(100) NOT NULL,
  `type` varchar(50) NOT NULL,
  `author` varchar(50) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=12 DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `books`
--

LOCK TABLES `books` WRITE;
/*!40000 ALTER TABLE `books` DISABLE KEYS */;
INSERT INTO `books` VALUES (1,'资本论','123','make'),(2,'西游记','123','吴承恩'),(3,'三国演义','123','罗贯中'),(4,'水浒传','123','施耐庵'),(5,'红楼梦','123','曹雪芹'),(7,'深入浅出Node.js','123','朴灵'),(8,'深入浅出Vue.js','123','刘博文'),(9,'深入浅出Vue.js2','123','刘博文'),(10,'深入浅出Vue.js3','123','刘博文'),(11,'深入浅出Vue.js4','123','刘博文');
/*!40000 ALTER TABLE `books` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `level`
--

DROP TABLE IF EXISTS `level`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
 SET character_set_client = utf8mb4 ;
CREATE TABLE `level` (
  `name` varchar(255) DEFAULT NULL,
  `level` int(64) DEFAULT NULL,
  `description` varchar(255) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `level`
--

LOCK TABLES `level` WRITE;
/*!40000 ALTER TABLE `level` DISABLE KEYS */;
INSERT INTO `level` VALUES ('青铜',0,'倔强青铜等级'),('白银',1,'白银等级'),('黄金',2,'闪耀黄金等级'),('铂金',3,'铂金等级'),('钻石',4,'璀璨钻石等级'),('大师',5,'超凡大师等级'),('王者',6,'无上王者等级'),('王者',7,'闪耀等级'),('王者',8,'闪耀等级');
/*!40000 ALTER TABLE `level` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `user`
--

DROP TABLE IF EXISTS `user`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
 SET character_set_client = utf8mb4 ;
CREATE TABLE `user` (
  `uid` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `author` varchar(100) NOT NULL,
  `level` int(10) unsigned DEFAULT NULL,
  PRIMARY KEY (`uid`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `user`
--

LOCK TABLES `user` WRITE;
/*!40000 ALTER TABLE `user` DISABLE KEYS */;
INSERT INTO `user` VALUES (1,'make',1),(2,'berg',3),(3,'emei',11);
/*!40000 ALTER TABLE `user` ENABLE KEYS */;
UNLOCK TABLES;
/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;

/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;

-- Dump completed on 2022-04-08 23:00:40
```

从文件内容可以看出，实际上数据库导出的是一系列创建数据表和插入数据的命令。

- 导出指定的表结构

```bash
mysqldump -u root -p 数据名 数据表名 > ./xx.sql
```

### 导入

- 将导出的数据表数据导入到数据表

```bash
# 将'/tmp/books.sql'中的数据导入到books表中
LOAD DATA INFILE '/tmp/books.sql' INTO TABLE books;
```

- 将导出的数据库（或数据表）导入

```bash
# 根据导出./books.sql文件在express-blog数据库中创建表和表的数据
mysql -u root -p express-blog < ./books.sql
```