# SQL 其他操作

## 视图

视图（View）是一个虚拟表，本质为根据 SQL 语句动态获取结果集。把这个 SQL 命名，用户只需要使用视图名即可获取结果。

#### 1. 创建视图

~~~sql
CREATE VIEW view_name AS
SELECT column1, column2, ...
FROM table_name
WHERE condition;CREATE VIEW v_employee_with_location AS
SELECT
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee_name,
    e.department,
    d.location
FROM employees e
JOIN departments d ON e.department = d.dept_name;
~~~



#### 2. 使用视图

~~~sql
seect * from view_name;
~~~

#### 3. 删除视图

~~~sql
drop view view_name;
~~~



## 触发器

触发器（Trigger）是数据库中的一种特殊对象，它能在特定的数据库事件（如 `INSERT`, `UPDATE`, `DELETE`）**之前**或**之后**自动执行一段预定义的 SQL 代码。发器是一个强大的工具，但应谨慎使用。通常用于实现跨表的强制一致性约束、审计跟踪、透明日志记录等核心数据治理需求，而不是实现复杂的业务逻辑。

#### 1. 创建触发器

~~~sql
CREATE TRIGGER trigger_name
{BEFORE | AFTER} {INSERT | UPDATE | DELETE}
ON table_name FOR EACH ROW
[trigger_order]
BEGIN
    -- 触发器要执行的 SQL 语句
END;
~~~

**关键部分解析：**

- `trigger_name`: 触发器的名称，必须在数据库中唯一。
- `{BEFORE | AFTER}`: 指定触发器在事件**之前**还是**之后**执行。
- `{INSERT | UPDATE | DELETE}`: 指定监听哪种 DML 事件。
- `ON table_name`: 指定触发器关联在哪张表上。
- `FOR EACH ROW`: 这是一个关键短语，表示这是一个**行级触发器**。意味着受事件影响的每一行数据都会触发一次代码执行。MySQL 只支持行级触发器。
- `trigger_order`: （可选）MySQL 5.7+ 支持，用于定义具有相同触发事件和时间的多个触发器的执行顺序。例如：`FOLLOWS other_trigger_name` 或 `PRECEDES other_trigger_name`。
- `BEGIN ... END;`: 包裹触发器要执行的核心逻辑。如果逻辑只有一条语句，可以省略。但为了清晰，建议总是使用。

### 2. 访问新旧数据：`NEW` 和 `OLD`

在触发器体内，你可以访问到受事件影响的数据行。

| 触发器类型      | `OLD` 的含义           | `NEW` 的含义           |
| :-------------- | :--------------------- | :--------------------- |
| `INSERT` 触发器 | 无效（为 `NULL`）      | **将要插入**的新行数据 |
| `UPDATE` 触发器 | **更新前**的原始行数据 | **更新后**的新行数据   |
| `DELETE` 触发器 | **删除前**的原始行数据 | 无效（为 `NULL`）      |

**如何使用：**

- `OLD.column_name`: 引用某列变化前的值。
- `NEW.column_name`: 引用某列变化后的值。
- 在 `BEFORE INSERT` 或 `BEFORE UPDATE` 触发器中，你可以修改 `NEW.column_name` 的值。

#### 3. 示例

在更新一行记录时，自动将 `last_modified` 字段设置为当前时间

```sql
DELIMITER //

CREATE TRIGGER before_employee_update
BEFORE UPDATE
ON employees FOR EACH ROW
BEGIN
    SET NEW.last_modified = NOW();
END;
//

DELIMITER ;
```

使用 `DELIMITER` 临时更改语句分隔符，以便在触发器体内使用分号 `;`。



#### 4. 删除触发器

~~~sql
drop TRIGGER <trigger_name>
~~~



#### 5. 查看触发器

~~~sql
-- 列表所有
SHOW TRIGGERS;

-- 查看单个
SHOW CREATE TRIGGER trigger_name;
~~~



## 函数

#### 1. 字符串函数 

| 函数                                                | 描述                                                     | 示例                                                         |
| :-------------------------------------------------- | :------------------------------------------------------- | :----------------------------------------------------------- |
| `CONCAT(str1, str2, ...)`                           | 连接多个字符串                                           | `SELECT CONCAT('Hello', ' ', 'World');` -> `'Hello World'`   |
| `LENGTH(str)` / `CHAR_LENGTH(str)`                  | 返回字符串的**字节**长度 / **字符**长度                  | `SELECT LENGTH('中国');` -> `6` (UTF8), `CHAR_LENGTH('中国')` -> `2` |
| `UPPER(str)` / `LOWER(str)`                         | 将字符串转换为大写/小写                                  | `SELECT UPPER('mysql');` -> `'MYSQL'`                        |
| `TRIM([{BOTH|LEADING|TRAILING} [remstr] FROM] str)` | 去除字符串首尾的指定字符（默认为空格）                   | `SELECT TRIM(' hello ');` -> `'hello'`                       |
| `SUBSTRING(str, pos [, len])` / `SUBSTR()`          | 从位置 `pos` 开始提取子字符串，提取 `len` 个字符         | `SELECT SUBSTRING('MySQL', 3, 2);` -> `'SQ'`                 |
| `REPLACE(str, from_str, to_str)`                    | 将字符串中的 `from_str` 替换为 `to_str`                  | `SELECT REPLACE('www.mysql.com', 'mysql', 'oracle');` -> `'www.oracle.com'` |
| `LEFT(str, len)` / `RIGHT(str, len)`                | 返回字符串左边/右边开始的 `len` 个字符                   | `SELECT LEFT('MySQL', 2);` -> `'My'`                         |
| `LOCATE(substr, str [, pos])`                       | 返回子串 `substr` 在字符串 `str` 中第一次出现的位置      | `SELECT LOCATE('SQL', 'MySQL');` -> `3`                      |
| `FORMAT(X, D [, locale])`                           | 将数字 `X` 格式化为 `'#,##,##.##'` 样式，保留 `D` 位小数 | `SELECT FORMAT(1234567.456, 2);` -> `'1,234,567.46'`         |



#### 2. 数值函数

| 函数                        | 描述                                        | 示例                                        |
| :-------------------------- | :------------------------------------------ | :------------------------------------------ |
| `ROUND(X [, D])`            | 将 `X` 四舍五入，保留 `D` 位小数            | `SELECT ROUND(123.4567, 2);` -> `123.46`    |
| `CEILING(X)` / `CEIL(X)`    | 返回大于等于 `X` 的最小整数（向上取整）     | `SELECT CEIL(123.456);` -> `124`            |
| `FLOOR(X)`                  | 返回小于等于 `X` 的最大整数（向下取整）     | `SELECT FLOOR(123.456);` -> `123`           |
| `ABS(X)`                    | 返回 `X` 的绝对值                           | `SELECT ABS(-10);` -> `10`                  |
| `RAND()` / `RAND(N)`        | 返回一个 0~1 之间的随机浮点数（`N` 为种子） | `SELECT RAND();` -> `0.123456...` (随机)    |
| `POW(X, Y)` / `POWER(X, Y)` | 返回 `X` 的 `Y` 次方                        | `SELECT POW(2, 3);` -> `8`                  |
| `MOD(N, M)`                 | 返回 `N` 除以 `M` 的余数（取模）            | `SELECT MOD(10, 3);` -> `1`                 |
| `TRUNCATE(X, D)`            | 将数字 `X` 截断（非四舍五入）到 `D` 位小数  | `SELECT TRUNCATE(123.4567, 2);` -> `123.45` |



#### 3. 日期和时间函数

| 函数                                           | 描述                                             | 示例                                                         |
| :--------------------------------------------- | :----------------------------------------------- | :----------------------------------------------------------- |
| `NOW()` / `CURDATE()` / `CURTIME()`            | 返回当前**日期和时间** / **日期** / **时间**     | `SELECT NOW();` -> `'2023-10-27 15:30:45'`                   |
| `DATE(date)` / `TIME(date)`                    | 从日期时间值中提取**日期**部分 / **时间**部分    | `SELECT DATE(NOW());` -> `'2023-10-27'`                      |
| `YEAR(date)` / `MONTH(date)` / `DAY(date)`     | 从日期中提取**年** / **月** / **日**             | `SELECT YEAR('2023-10-27');` -> `2023`                       |
| `HOUR(time)` / `MINUTE(time)` / `SECOND(time)` | 从时间中提取**时** / **分** / **秒**             | `SELECT HOUR('15:30:45');` -> `15`                           |
| `DATE_FORMAT(date, format)`                    | 按指定格式显示日期/时间                          | `SELECT DATE_FORMAT(NOW(), '%W, %M %e, %Y');` -> `'Friday, October 27, 2023'` |
| `DATEDIFF(expr1, expr2)`                       | 返回两个日期之间相差的**天数** (`expr1 - expr2`) | `SELECT DATEDIFF('2023-10-30', '2023-10-27');` -> `3`        |
| `DATE_ADD(date, INTERVAL expr unit)`           | 日期加法                                         | `SELECT DATE_ADD(NOW(), INTERVAL 1 DAY);` -> 明天的此时      |
| `DATE_SUB(date, INTERVAL expr unit)`           | 日期减法                                         | `SELECT DATE_SUB(NOW(), INTERVAL 1 MONTH);` -> 一个月前的此时 |
| `DAYNAME(date)`                                | 返回日期是星期几                                 | `SELECT DAYNAME('2023-10-27');` -> `'Friday'`                |
| `LAST_DAY(date)`                               | 返回给定日期所在月份的最后一天                   | `SELECT LAST_DAY('2023-02-15');` -> `'2023-02-28'`           |

**常用 `DATE_FORMAT` 格式符：**

- `%Y`：四位年份 (e.g., 2023)
- `%y`：两位年份 (e.g., 23)
- `%m`：月份 (01..12)
- `%d`：日 (00..31)
- `%H`：24小时制 (00..23)
- `%i`：分钟 (00..59)
- `%s`：秒 (00..59)
- `%W`：星期名 (Sunday..Saturday)
- `%M`：月份名 (January..December)



#### 4. 聚合函数

对一组值执行计算，并返回单个汇总值。**通常与 `GROUP BY` 子句一起使用。**

| 函数                 | 描述                                                         |
| :------------------- | :----------------------------------------------------------- |
| `COUNT(expr)`        | 返回记录的行数（`COUNT(*)` 包括NULL，`COUNT(column)` 不包括NULL） |
| `SUM(expr)`          | 返回表达式的总和                                             |
| `AVG(expr)`          | 返回表达式的平均值                                           |
| `MAX(expr)`          | 返回表达式中的最大值                                         |
| `MIN(expr)`          | 返回表达式中的最小值                                         |
| `GROUP_CONCAT(expr)` | 将一组中的字符串连接起来                                     |

**示例：**

```sql
SELECT 
    department,
    COUNT(*) AS emp_count,
    AVG(salary) AS avg_salary,
    MAX(salary) AS max_salary,
    GROUP_CONCAT(name SEPARATOR ', ') AS employees
FROM employees
GROUP BY department;
```



#### 5. 流程控制函数

根据条件执行不同的操作。

| 函数                                                         | 描述                                                         | 示例                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `IF(expr, v1, v2)`                                           | 如果表达式 `expr` 为真(不为0且不为NULL)，返回 `v1`，否则返回 `v2` | `SELECT IF(salary > 5000, 'High', 'Low') FROM employees;`    |
| `IFNULL(v1, v2)`                                             | 如果 `v1` 不为 NULL，则返回 `v1`；否则返回 `v2`              | `SELECT IFNULL(bonus, 0) FROM employees;` (将 NULL bonus 显示为 0) |
| `CASE value WHEN compare_value THEN result ... [ELSE result] END` | 复杂的条件判断，类似于编程中的 switch-case                   | 见下方示例                                                   |

**`CASE` 表达式示例：**

```mysql
SELECT 
    name,
    salary,
    CASE 
        WHEN salary < 5000 THEN 'Junior'
        WHEN salary BETWEEN 5000 AND 10000 THEN 'Mid'
        ELSE 'Senior'
    END AS level
FROM employees;
```



#### 6. 其他实用函数

| 函数                                  | 描述                                             | 示例                                                         |
| :------------------------------------ | :----------------------------------------------- | :----------------------------------------------------------- |
| `COALESCE(val1, val2, val3, ...)`     | 返回参数列表中的第一个**非 NULL** 值             | `SELECT COALESCE(NULL, NULL, 'Third', 'Fourth');` -> `'Third'` |
| `CAST(expr AS type)`                  | 将表达式转换为任何其他数据类型                   | `SELECT CAST('123' AS UNSIGNED);` -> `123` (数字)            |
| `CONVERT(expr, type)`                 | 同上，`CAST()` 的另一种语法                      | `SELECT CONVERT('2023-10-27', DATE);`                        |
| `MD5(str)` / `SHA1(str)`              | 返回字符串的 MD5 / SHA1 哈希值（常用于密码加密） | `SELECT MD5('password');`                                    |
| `DATABASE()` / `USER()` / `VERSION()` | 返回当前数据库名 / 当前用户 / MySQL 版本信息     | `SELECT DATABASE();`                                         |

#### 如何使用

这些函数可以用于 `SELECT` 列表、`WHERE` 子句、`ORDER BY` 子句、`GROUP BY` 子句以及 `INSERT` 和 `UPDATE` 语句的值中。

```mysql
-- 在 SELECT 列表中使用
SELECT name, UPPER(department), ROUND(salary, 0) 
FROM employees;

-- 在 WHERE 子句中使用
SELECT * FROM orders 
WHERE YEAR(order_date) = 2023 
AND MONTH(order_date) = 10;

-- 在 ORDER BY 子句中使用
SELECT * FROM products 
ORDER BY IFNULL(stock_quantity, 0) DESC;

-- 在 UPDATE 语句中使用
UPDATE employees 
SET last_login = NOW() 
WHERE id = 1001;
```



## 存储过程

存储过程（Stored Procedure）是一组为了完成特定功能的 SQL 语句集，经编译后存储在数据库中，用户通过指定存储过程的名字并给出参数（如果需要）来调用执行。

**主要优势：**

1. **高性能**：预编译一次，多次调用，减少网络传输和解析开销。
2. **代码复用**：一次编写，多处调用，避免代码冗余。
3. **业务逻辑封装**：将复杂业务逻辑封装在数据库层，保证数据操作的一致性。
4. **安全性**：可以授权用户执行存储过程，而不直接操作底层表。



#### 创建存储过程

```mysql
DELIMITER //

CREATE PROCEDURE procedure_name ([IN | OUT | INOUT] parameter_name data_type, ...)
[characteristic ...]
BEGIN
    -- 存储过程体：SQL语句和逻辑控制
END //

DELIMITER ;
```

**关键部分解析：**

- `DELIMITER //`：临时更改语句结束符，以便在过程体内使用分号 `;`。
- `CREATE PROCEDURE`：创建存储过程的关键字。
- `procedure_name`：存储过程的名称。
- **参数模式**：
  - `IN` (默认)：输入参数，调用者传入值（传入值在过程体内是只读的）。
  - `OUT`：输出参数，存储过程通过它返回值给调用者（传入的变量值会被忽略）。
  - `INOUT`：输入输出参数，调用者传入值，存储过程可以修改并返回新值。
- `BEGIN ... END`：包裹存储过程的主体逻辑。
- `DELIMITER ;`：将结束符恢复为分号。



#### 执行存储过程

~~~mysql
-- 无参数
call proc_name();

-- 有参数
call proc_name(1,2);
~~~



#### 删除储存过程

~~~mysql
drop procdure proc_name;
~~~

#### 查看储存过程定义

~~~sql
SHOW CREATE PROCEDURE procedure_name;
~~~

#### 查看所有存储过程

~~~sql
SHOW PROCEDURE STATUS WHERE Db = DATABASE();
~~~



## 查看数据库连接

**查看允许的最大连接数**

~~~sql
mysql> show variables like 'max_connections';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 151   |
+-----------------+-------+
1 row in set (0.02 sec)
~~~

**查看当前连接数**

~~~sql
mysql> show status like 'Threads_connected';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_connected | 1     |
+-------------------+-------+
1 row in set (0.00 sec)
~~~

