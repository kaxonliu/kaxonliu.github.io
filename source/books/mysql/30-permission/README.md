# 权限管理

MySQL 权限管理是数据库安全和管理中的核心部分。它主要涉及创建用户、授予权限、撤销权限和删除用户等操作。



## 核心概念

1. **用户账号**：由 `'用户名'@'主机名'` 唯一标识。`'root'@'localhost'` 和 `'root'@'%'` 是两个不同的用户。
2. **权限 (Privileges)**：定义了用户可以对数据库对象（如数据库、表、列）执行的操作，如 `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `DROP` 等。
3. **授权表 (Grant Tables)**：MySQL 在 `mysql` 系统数据库中存储用户和权限信息，主要表包括：
   - `user`：用户账户和全局权限。
   - `db`：数据库级权限。
   - `tables_priv`：表级权限。
   - `columns_priv`：列级权限。
   - `procs_priv`：存储过程和函数权限。

所有修改用户或权限的操作，最终都是修改这些表。**强烈建议使用专门的 SQL 命令（如 `CREATE USER`, `GRANT`）而不是直接修改这些表**来管理用户。



## 用户管理

#### 1. 查看现有用户

~~~mysql
SELECT User, Host FROM mysql.user;
~~~

`'root'@'localhost'` 和 `'root'@'%'` 是两个不同的用户。



#### 2. 创建用户

使用 `CREATE USER` 语句创建新用户。创建用户后，默认没有任何权限。

~~~sql
CREATE USER '用户名'@'主机名' IDENTIFIED BY '密码';
~~~

**主机名**：指定用户可以从哪里连接到数据库，有如下值可供选择。

- `'localhost'`：只能从本机连接，和 `127.0.0.1` 不同。
- `'%'`：可以从任何主机连接。（**慎用，有安全风险**）
- `'192.168.1.%'`：可以从 `192.168.1.0/24` 网段连接。
- `'example.com'`：可以从 `example.com` 域名的主机连接。

如果指定的主机名是 `127.0.0.1`，那么登陆使用 `-h 127.0.0.1`。



#### 3. 删除用户

使用 `DROP USER` 语句删除用户。删除用户会同时移除该用户的所有权限。和新建用户类似，删除时也需要指定主机名。

```mysql
DROP USER '用户名'@'主机名';
```



#### 4. 修改密码

~~~sql
ALTER USER '用户名'@'主机名' IDENTIFIED BY '新密码';
~~~

#### 5. 修改用户名

~~~sql
RENAME USER '旧用户名'@'旧主机名' TO '新用户名'@'新主机名';
~~~



## 授权

使用 `GRANT` 语句为用户分配权限。

语法如下：

~~~sql
GRANT 权限列表 ON 数据库名.表名 TO '用户名'@'主机名';
~~~

**权限列表**：

- 特定权限：`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `DROP`, `ALTER`, `INDEX`, `ALL PRIVILEGES` (所有权限) 等。
- 用逗号分隔多个权限，例如 `GRANT SELECT, INSERT, UPDATE ON ...`。
- 除了授权之外的所有权限，`ALL`。

**数据库名.表名**：

- `*.*`：所有数据库的所有表（全局权限）。
- `数据库名.*`：指定数据库的所有表。
- `数据库名.表名`：指定数据库的指定表。

好习惯：使用 `FLUSH PRIVILEGES;` 命令使新授予的权限立即生效。



#### 1. 授权示例

~~~sql
-- 示例1
-- 授予 liuxu 对 所有库及其表拥有所有权限
-- 权限保存在 mysql.user 表中
grant all on *.* to 'liuxu'@'localhost';
FLUSH PRIVILEGES;


-- 示例2
-- 授予 tom 对 db1库中的所有表拥有所有权限
-- 权限保存在 mysql.db 表中
grant all on db1.* to 'tom'@'localhost';
FLUSH PRIVILEGES;


-- 示例3
-- 授予 lili 对 db1库中t1表拥有select和update权限
-- 权限保存在表 mysql.tables_priv
grant SELECT, INSERT on db1.t1 to 'lili'@'localhost';
FLUSH PRIVILEGES;


-- 示例4
-- 授予 lili 对 db1库的t3表，拥有id age字段的select权限，age字段的update权限
-- 权限保存在表 mysql.columns_priv
grant select (id,age),update(age) on db1.t3 to 'lili'@'localhost';
~~~



#### 权限超级管理员权限

`all` 可以代表除了 `grant` 之外的所有权限，配合使用 `with grant options`  表示授权管理员权限。

~~~sql
grant all on *.* to 'liuxu'@'localhost' with grant options;
~~~



#### 撤销权限

属权使用 `grant` 和 `to`

撤销权限使用 `revoke` 和 `from`

~~~sql
-- 把 liuxu 的全部权限撤销
revoke all on *.* from liuxu@localhost;

-- 撤销 tom 对 db1库中所有表的update权限
revoke update on db1.* from tom@localhost;
~~~



#### 查看用户权限

使用 `SHOW GRANTS` 查看特定用户的权限。

~~~sql
SHOW GRANTS FOR 'liuxu'@'localhost';
~~~



#### 扩展权限

max_queries_per_hour：一个用户每小时可发出的查询数量
max_updates_per_hour：一个用户每小时可发出的更新数量
max_connections_per_hour：一个用户每小时可连接到服务器的次数
max_user_connections：允许同时连接数量

~~~sql
-- liuxu 在一个小时内剋发出一次查询请求
grant all on *.* to 'liuxu'@'%'  with max_queries_per_hour 1;
FLUSH PRIVILEGES;
~~~

