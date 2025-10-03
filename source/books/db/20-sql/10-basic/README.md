# SQL 基本操作

## 数据库

### 1. 新建数据库

使用 `charset` 指定字符编码，如果没有指定则使用 mysqld 配置的字符编码。

~~~sql
create datebase db1 charset utf8;
~~~

### 2. 修改数据库

主要是修改字符编码（CHARACTER）和排序规则（COLLATE）

~~~sql
-- 将数据库 mydb 的字符集修改为 utf8mb4，排序规则修改为 utf8mb4_unicode_ci
ALTER DATABASE mydb 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;
~~~

### 3. 查看数据库

~~~sql
-- 列出来所有数据库
show databases;

-- 查看数据库的创建 SQL 语句
show create database db1;

-- 查看当前使用了哪个数据库
select database();
~~~

#### 4. 使用库

~~~sql
use db1;
~~~



### 5. 删除数据库

~~~sql
drop database db1;
~~~



## 表

### 1. 创建表

创建表需要定义表名、表的类型（存储引擎）、表中的字段以及字段的类型和约束条件，其中约束条件是可选的。MySQL 支持多种存储引擎，默认的存储引擎是 `InnoDB`。使用命令 ` show engines;` 查看所有支持的存储引擎。

~~~sql
create table student(
    id int primary key auto_increment,
    name varchar(64) not null,
    birthday datetime not null
);
~~~



### 2. 查看表

~~~sql
-- 查看当前库中所有的表
show tables;

-- 查看指定表的创建 SQL 语句
show create table student;

-- 查看表结构
describe student;
desc student;
~~~



### 3. 修改表结构

~~~sql
-- 修改表名字
alter table student rename stu;

-- 删除字段
alter table student drop age;

-- 增加字段
alter table student add age int not null;
alter table student add sex tinyint not null first;
alter table student add height int after name;

-- 修改字段
-- modify 只能修改字段类型和约束条件
alter table student modify name varchar(32);
-- change 可以修改字段名字、类型、约束条件
alter table student change name student_name varchar(64);
~~~



### 4. 复制表

~~~sql
-- 复制表结构和数据
-- 注意：不会复制 key
create table new_student select * from student;

-- 仅复制表结构
create table new_student select * from student where 1=2;
~~~



## 记录

### 1. 插入数据

~~~sql
-- 指定字段
insert into student(name,height,birthday) values('liuxu','175','1999-11-11');

-- 省略字段
insert into student values(3, 'jack','175','1999-11-11');

-- 批量插入
insert into student(name,height,birthday) values
('liuxu','175','1999-11-11'),
('liuxu','175','1999-11-11'),
('liuxu','175','1999-11-11');
~~~



### 2. 更新数据

必须使用 `where` 限定修改的范围，否则就全表更新。

~~~sql
update student set height=180 where id=3;
~~~



### 3. 删除数据

必须使用 `where` 限定修改的范围，否则就全表删除。

~~~sql
delete from student where id=3;
~~~



### 4. 清空重置表

~~~sql
truncate student;
~~~



### 5. 单表查询

#### 语法顺序

~~~sql
SELECT DISTINCT <字段列表> 
FROM <表名>
WHERE <条件>
GROUP BY <分组字段>
HAVING <二次筛选>
ORDER BY <排序字段>
LIMIT <限制条数>
~~~

#### 执行顺序

1. from，先确定使用哪张表
2. where，初次筛选数据
3. group by，分组
4. having，分组结果二次筛选
5. select，查询字段值
6. distinct，去重
7. order by，排序
8. limit，限制显示条数



#### where 筛选条件

~~~sql
-- 比较运算符
< > <= >= = != 

-- 区间范围
between 10 and 89

-- 指定数据集
in (10,20,12,32)

-- 字符串模糊, % 任意多个字读，_ 任意一个字符
name like '国%'
name kile '国_'

-- 逻辑运算符 and or not

-- 正则表达式
where name REGEXP '^ja'
~~~



#### 分组

分组之后只能使用 分组字段、组内的聚合数据。

可以使用的聚合函数

~~~sql
-- 组内成员的个数
COUNT(*)

-- 组内的用户名都有哪些
GROUP_CONCAT(name)

-- 组内的统计值
MAX(age)
MIN(age)
AVG(age)
SUM(age)
~~~

>补充，MySQL 5.7 之后默认开启了 `ONLY_FULL_GROUP_BY` 模式，不能在分组后使用分组之外的普通字段。c
>
>~~~sql
>select @@global.sql_mode;
>~~~





#### 排序

支持一个字段排序，也支持多个字段排序。支持升序、降序。默认是升序。

~~~sql
-- 按照年龄降序排序，同时按照身高升序排序。
select * from student order by age desc, height asc
~~~



#### 限制个数

两种玩法：1. 只限制个数；2. 控制起点的同时再限制个数（分页）。默认起点为0。

~~~sql
select * from student limit 3;
select * from student limit 3, 3;
~~~



### 6. 联表查询

联表之后再基于单表查询的逻辑，使用筛选、分组等功能。

~~~sql
select a.*, b.* from a inner join b on a.name = b.name
select a.*, b.* from a left join b on a.name = b.name
select a.*, b.* from a right join b on a.name = b.name
~~~



### 7. 子查询

把一次查询的结果，当作另一个查询的条件。

第一次查询的结果需要使用 `()` 括起来，并且不能带 `;`

条件判断时可以可以使用的命令：`in`、`all`、`any` 等，还可以使用 `=`、`!=` `>` 等。

~~~sql
select * from student where class_id in (select id from class);
~~~

需要注意：`all`、`any` 需要配合动态的查询结果集，不能是手动设置的结果集。不过 `in` 可以是动态的也可以使用手动指定的。

