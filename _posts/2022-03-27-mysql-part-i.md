# MySQL随笔 - Part I

​	最近，在复习一些常用的技术栈，试着从文档和源代码两部分来分析这些软件。当然，也由于本人的言语表达能力、技术能力有限，所以一些文章会有个别错误及表达问题。希望包容指出～

​	然后关于这些知识的来源，主要是[官方技术文档](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/index.html)和源代码两个方面，同时尽量减少直接贴代码（除非非常必要及精彩的部分）。

## 关于MySQL

​	我们知道MySQL是一个**关系型数据库**，关系型数据库将数据存储在不同的表里。数据库里的主要元素有**行、列、数据库、表、视图**等元素，MySQL将这些元素的复杂度封装起来，对外以端口3306的形式提供给我们数据库服务，我们可以用它来做模型、数据的关联。常见的关联如一对一、一对多。

​	*tips: MySQL常用的数据查询语句是符合SQL规范的，何为SQL？它的全称其实是Structured Query Language，即结构化数据查询语句。*

​	MySQL是经典的c/s架构，client和server之间可以通过tcp/ip协议来进行数据交换，一个简单的架构图如下：

![cs](https://user-images.githubusercontent.com/38686456/160279756-ddcb90cd-d4c2-4eb3-8fc8-ea1fac193eee.png)

​	client端可以是jdbc、[go-sql-driver](https://github.com/go-sql-driver/mysql)和MySQL自带的客户端等。这部分后面会展开成一章节的通过抓包等方式来详细叙述其经过。

## 常用功能

​	MySQL本身具有以下几个主要功能：

- 丰富的数据类型支持，如[`FLOAT`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/floating-point-types.html), [`DOUBLE`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/floating-point-types.html), [`CHAR`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/char.html), [`VARCHAR`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/char.html), [`BINARY`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/binary-varbinary.html), [`VARBINARY`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/binary-varbinary.html), [`TEXT`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/blob.html), [`BLOB`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/blob.html), [`DATE`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/datetime.html), [`TIME`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/time.html), [`DATETIME`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/datetime.html), [`TIMESTAMP`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/datetime.html), [`YEAR`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/year.html), [`SET`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/set.html), [`ENUM`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/enum.html)
- 常用的函数支持，如[`COUNT()`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/aggregate-functions.html#function_count), [`AVG()`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/aggregate-functions.html#function_avg), [`STD()`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/aggregate-functions.html#function_std), [`SUM()`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/aggregate-functions.html#function_sum), [`MAX()`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/aggregate-functions.html#function_max), [`MIN()`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/aggregate-functions.html#function_min), and [`GROUP_CONCAT()`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/aggregate-functions.html#function_group-concat)
- 常用的用户名/密码的鉴权方式
- 丰富的生态、代码仓库

