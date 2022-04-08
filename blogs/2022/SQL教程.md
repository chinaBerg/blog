# SQL入门

> 愣锤 2022/04/08

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

### 索引

### 导入导出

- 导出某个表的数据

```bash
# 将books表中所有数据导出到./books.sql文件中
select * from books into outfile './books.sql';
```

命令运行后你可能遇到如下错误导致无法导出：

```bash
ERROR 1290 (HY000): The MySQL server is running with the --secure-file-priv option so it cannot execute this statement
```

![image](https://note.youdao.com/yws/res/21185/F3F6AD2F9E0A4038BEACE9777A960F6C)

这个错误是说数据库的`secure-file-priv`设置不允许导入导出数据。

解决版本，我们先查看一下我们的数据库`secure-file-priv`配置:

```bash
# 终端运行
show variables like '%secure%';
```

![image](https://note.youdao.com/yws/res/21181/9F270583388F47D3AE711AB7B5163C54)

`secure-file-priv`值为`NULL`说明不允许导入导出，没有具体值时表示不对导入导出做限制，有具体值表示只允许在指定路径导入导出。

- 导出整个数据库

注意，导出的是整个数据库所有表的表结构，而不是导出全部数据。导出的表结构可以在其他地方导入直接创建表。

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
