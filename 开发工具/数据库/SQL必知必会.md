# 1.   了解 SQL

## 1.1  数据库基础

SQL 是用于和数据库打交道的语言。

### 1.1.1  数据库

数据库是以某种有组织的方式存储的数据集合。
- **数据库**是保存有数据的容器。
- 数据库管理系统（数据库软件）**DBMS**：用户通过 DBMS 创建和管理数据库。

---

<br/>


<br/>



### 1.1.2  表
表 Table 是数据库中结构化的文件，用于存储特定类型（不等于基本类型，而是不同的对象类型）的数据，例如，顾客清单、产品目录等。

> **表名**
> 数据库名和表名的组合使得表名成为唯一。有些数据库还是用数据库拥有者名字作为唯一名的一部分。

> **模式 Schema** 是描述表的特性（如何存储、存储什么数据、数据如何分解、各部分如何命名）的信息。

---

<br/>


<br/>



### 1.1.3  列和数据类型

表由列组成，类存储表中特定部分的信息。如客户清单中的，姓名、性别放在不同类中存储。

每类都有对应的数据类型。数据类型定义类列可以存储哪些数据种类。

> **数据类型兼容**
> 数据类型和名称是 SQL 不兼容的一个主要问题，许多高级数据类型没有一致的支持，相同的数据类型在不同的 DBMS 可能会有不同的名称。

---

<br/>


<br/>

### 1.1.4  行

行是表中的一个记录。

> **记录和行**
> 记录 record 和行基本可以互通，从技术上说，行才是正确的术语。

---

<br/>


<br/>

### 1.1.5  主键

表中每一行都应该有一列（或者多列）可以唯一标识自己，这些列就称为主键，主要有以下要求：
- 不同行的主键不能相同。
- 每一行都必须有主键。
- 主键的值不能修改或者更新。
-<span style="background:#fff88f"> 主键值不能复用。（列从行中删除之后，主键也不能赋给以后新的行）</span>。

> 不能复用，为了保证数据的唯一性。


---

<br/>


<br/>


<br/>



## 1.2  什么是 SQL

SQL **结构化查询语言（Structured Query Language）**，是一种专门用于和数据库沟通的语言。

- SQL 是通用语言。绝大多数 DBMS 支持 SQL，但是可能存在部分 SQL 方言。

---

<br/>


<br/>


<br/>




## 1.3  安装 MYSQL
> - Windows cmd 切换到盘：`D:`
> - 查看当前目录下文件和目录：`dir \b`

开始使用的安装教程如下：[MySQL 安装教程](https://blog.csdn.net/qq_59636442/article/details/123058454)。按照这个教程安装，遇到如了如下问题：
- [MYSQL 输入密码后闪退的解决方法](https://blog.csdn.net/weixin_44023658/article/details/105815797)
- MySQL 的安装路径在 C 盘，需要很高权限才能在其中操作。
- 默认数据库创建位置在根目录，C:\ProgarmData\  ，在资源管理器无法查看。
- 修改 ini 文件导致 MYSQL 无法启动。

以上问题的根本原因在于，教程中下载的 SQL 是配置好了的，灵活性太低，难以自行操作。[自定义安装教程](https://blog.51cto.com/u_16099178/7277203)。



本机的 MySQL 安装路径：
```
D:\Program Files\mysql-8.2.0-winx64\bin
```
默认数据创建路径：
```
D:\Program Files\mysql-8.2.0-winx64\data
```



**常用命令**
> - 安装 MySQL 服务：`mysqld --install`
> - 卸载 MySQL 服务：`mysqld --remove`

> - 启动服务：`net start mysql`（<span style="background:#fff88f">需要管理员权限</span>）；也可以手动启动：控制面板-系统和安全-管理空间-服务，找到 MYSQL 并启动。
> - 关闭服务：`net stop mysql`
> - **进入工作空间**：`mysql -u root -p`，输入用户密码即可。
> 	- 第一次进入工作空间可能需要修改密码，`alter user 'root'@'localhost' identified by '密码';`。
> - **退出工作空间**：`quit;`

>
> - 列出所有数据库：`show databases;`
> - 选中使用某个数据：`use database_name;`
> - 列出数据库中所有表：`show tables;`
> - 查看某个表的概要信息：`desc table_name;`
> - 创建数据库：`create database  database_name;`
> - 删除指定数据库：`drop database database_name;`

---


<br/>


<br/>


<br/>


<br/>


# 2.   检索数据

- 切换到指定数据库：`use database_name;`

## 2.1 SELECT 语句

使用 SELECT 语句去检索数据，至少需要给出两条信息（可以根据需求增加参数）：

- 选择索引哪些列
- 在哪个表中进行索引

---

<br/>


<br/>


## 2.2   检索单个列

```sql
SELECT prod_name
FROM Products;
```

以上语句的意思就是：在 Products 这个表中，检索 prod_name 这一列数据。

> **排序数据**
> 如果没有明确指出排序查询结果，则<span style="background:#fff88f">返回的数据没有特定的顺序</span>（可能是数据被添加到表中的顺序），只要返回相同数量的行就是正常的。

> **SQL 语句语法**
> - **SQL 语句不区分大小写**，但是许多开发人员喜欢对关键字使用大写。<span style="background:#fff88f">需要注意的是，表名、列名和值不一定不区分大小写</span>（取决于具体 DBMS）。
> - 在处理 SQL 语句时，**换行会被忽略**，可以写成多行，增加可读性，在语句最后添加分号。

---

<br/>


<br/>


<br/>


## 2.3  检索多个列

检索多个类，只需给出多个列名，在之间用逗号分开即可：
```sql
SELECT prod_id,prod_name,prod_price
FROM Products;
```

> **数据表示**
> SQL 语言一般返回原始的、无格式化的数据，不同 DBMS 和客户端显示数据方式略有不同，数据格式化是表示问题，不是检索问题。


----

<br/>


<br/>


<br/>



## 2.4  检索所有列

使用通配符 `*` 可以检索所有列：
```sql
SELECT *
FROM　Products;
```

---

<br/>


<br/>



<br/>



## 2.5  检索不同的值

以上语句都是之后返回表中某一类的数据，其中某些行在该列的数据可能会相同：
<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-10-29_152024.jpg" width = 300 /> </div>

如上图所以，显示了 9 行，但其中之后三种数据，如果只要显示**不同的值**，使用 `distinct` 关键字：
```sql
SELECT DISTINCT vend_id
FROM Products;
```

---

<br/>


<br/>


<br/>


## 2.6  限制结果

 `TOP` 限制最多返回多少行，如下：
 ```sql
 select top 5 prod_name
 from Products;
```
但是不同 DBMS 中存在不同的语法，**在 MYSQL 中，使用 `limit` 关键字**， 语法如下：
```sql
select prod_name
from Products
limit 5;
```

<br/>

**`offset` 指明从第几行开始检索数据**：
```sql
select prod_name
from Products
limit 5  offset 5;
```

> **第 0 行**
> <span style="background:#fff88f">第一个被检索的行是第 0 行</span>，如果从头开始检索，应写 `offset 0`

> **MySQL 捷径**
> <span style="background:#fff88f">从第 3 行开始，输出 4 行，可以简写为：`limit 3,4`。</span>这里的两个数字出现顺序和上面是相反的。

---

<br/>


<br/>


<br/>


## 2.7  注释

- 行内注释 `--`
``` sql
select prod_name -- 这是一条数据
from Products;
```

> <span style="background:#fff88f">注释内容不能紧贴 `-`，起码要隔开一个空格。</span>

也可以使用 `#`，但是这种形式某些 DBMS 不支持。

<br/>


- 多行注释：`/* */`


---

<br/>


<br/>


<br/>


<br/>



# 3.   排序检索数据

## 3.1  排序数据

如果不排序，数据一般一在表中出现的顺序显示（可能是数据最初添加到表中的顺序，但是进过更新或者删除之后，将会搜到影响），不应该依靠这个顺序。

使用 `order by` 子句指出根据哪些列进行排序。<span style="background:#fff88f">`order by` 子句必须是 `select` 语句中最后一条子句</span>，否则将会出错。

```sql
select prod_name
from Products
order by prod_name;
```

---


<br/>


<br/>


<br/>




## 3.2  按多个列排序

按多个列排序时，列名之间用逗号隔开。**排序规则类似于字典序排序**，先按第一个排序，第一个排序级别最高，在按之后的进行排序。

```sql
select prod_id,prod_price,prod_name
from Products
order by prod_price,prod_name;
```


---

<br/>


<br/>


<br/>


## 3.3  按位置排序

^c8cc01

```sql
select prod_id,prod_price,prod_name
from Products
order by 2,3;
```

跟 [[SQL必知必会#^c8cc01|按多个列排序]]是一样的，**用数字来代表不同的列**。<span style="background:#fff88f">数字代表的是 select 清单中出现的顺序，且第一个是 1 不是 0。在本例中，2 代表 prod_price。</span>

---

<br/>

<br/>

<br/>


## 3.4  指定排序方向

`order by` 子句**默认的排序是升序排序**，如果需要**降序排序，使用 `desc` 关键字。**

**使用方式**：
放在列名之后，直接作用于该一列。<span style="background:#fff88f">当需要多个列降序时，在每个列之后都要加关键字</span>。

```sql
select prod_id,prod_price,prod_name
from Products
order by prod_price desc,prod_name;
```

`desc` 关键字可以和 3.3 的按位置排序的用法结合。
```sql
order by 2 desc,3;
```

**与 `desc` 对应的关键字是 `asc`**。


---


<br/>


<br/>


<br/>


<br/>


# 4.   过滤数据

## 4.1  `where` 子句

**`where` 子句在 `from` 子句之后给出，但位于 `order by` 子句之前**。

> **SQL 过滤和应用过滤**
> 数据可以在应用层过滤——SQL 通过 select 语句给客户端检索出数据（可能超过实际所需），然后客户端代码对返回数据进行循环，提取出所需的行。
> 
> 这种做法不妥，**让客户端应用处理数据库将会极大影响应用性能，同时还浪费网络带宽。**
> 优化数据库可以快速有效对数据进行过滤。


---

<br/>


<br/>


<br/>


## 4.2  `where` 子句操作符

<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-10-31_222135.jpg" width = 600 /> </div>

**并不是所有 DBMS 都支持这些操作符**。

<br/>


### 4.2.1 范围值检查
使用方式：
```sql
select prod_name,prod_price
from Products
where prod_price between 5 and 10;
```

---

<br/>


<br/>


### 4.2.2  空值检查

表设计人员可以指定其中的列不包含值，此时包含空值 `null`。

**`null` 表示的是没有值，和 0、空字符串、空格不同。**

```sql
select prod_name,prod_price
from Products
where prod_price is null;
```


---

<br/>


<br/>


<br/>


<br/>



# 5.   高级数据过滤

## 5.1  组合 `where` 子句

可以给出多个 `where` 子句进行过滤，**子句用逻辑操作符 `and` 和 `or` 联结。**

### 5.1.1  `and` 操作符

```sql
selct prod_id,prod_price,prod_name
from Products
where vend_id='DLL01' and prod_price<=4;
```

---

<br/>


<br/>



### 5.1.2  `or` 操作符

和 `and` 的使用方式一样。**在许多 DBMS 中如果第一个条件得到满足就不再计算第二个条件。**

---

<br/>


<br/>



### 5.1.3 求值顺序

**`and` 的优先级比 `or` 的高。**

为了增加可读性，使用圆括号对操作符进行明确的分组，如下：
```sql
select prod_name,prod_price
from Products
where (vend_id ='DLL01' or vend_id= 'BRS01') and prod_price >= 10;
```

---

<br/>


<br/>


<br/>



## 5.2 `in` 操作符

`in` 操作符用于指定条件范围，范围中的每个条件都可以进行匹配。例子如下：

```sql
select prod_name,prod_price
from Products
where vend_id in ('DLL01','BRS01')
order by prod_name;
```

`in` 能做和 `or` 一样的事情，但是其的优点有：
- 语法更清晰，直观。
- 求值顺序更容易管理。
- **`in` 一般比一组 `or` 操作符执行的更快**。
- `in` 最大优点是**可以包含其他 `select` 语句**。

---

<br/>


<br/>


<br/>



## 5.3  `not` 操作符

例子如下：
```sql
select prod_name
from Products
where not vend_id="DLL01"
order by prod_name;
```

`not` 的优势在于——在更复杂的子句中，`not` 是非常有用的，在和 `in` 结合使用时，可以非常简单找出与条件列表不匹配的行。

---

<br/>


<br/>


<br/>


<br/>



# 6.  通配符进行过滤

## 6.1  `like` 操作符

之前所讲的数据过滤都是基于已知值进行的，例如 `prod_name='DLL01`。

<span style="background:#fff88f">通配符搜索只能用于字符。</span>

利用通配符可以匹配值的一部分。在搜索子句中，**使用通配符，必须使用 `like` 子句**，该子句指示 DBMS 后面的搜索模式不是简单的匹配比较。

> **搜索模式**
> 由字面值、通配符或者两者结合构成的搜索条件。

> **谓词**
> 从技术上来说，`like` 是谓词，而不是操作符。


<br/>


### 6.1.1  `%` 通配符


` % ` 表示**任何字符**出现**任何次数**（可以是 0 次）。

使用例子，如下：
```sql
select prod_id,prod_name
from Products
--  where prod_name like 'Fish%';           -- 以'Fish'开头
--  where prod_name like '%bean bag%';      --中间包含'bean bag'
--  where prod_name like 'F%y';             --以F开头，y结尾
```

> **区分大小写**
> 根据 DBMS 的不同及其配置，搜索可以区分大小写。

> **搜索电子邮件地址**
> 有一种情况吧通配符放在搜索模式中间是很有用的——根据邮件地址的一部分来查找电子邮件。例如 `where email like 'b%@forta.com';`

> **后面所跟的空格**
> 有些 DBMS 用空格来填补字段的内容，需要额外注意，更好的方式是用函数来去掉空格。

> `NULL`
> 通配符不能用来匹配 `NULL`。

---

<br/>


<br/>



### 6.1.2  `_` 通配符

`_` 使用方式和 `%` 一样，但是只能匹配单个字符（不能是 0 个，相当于一个占位符）。

> DB 2 不支持通配符 `_`。


---

<br/>


<br/>



### 6.1.3  `[]` 通配符

> 微软 SQL Server 支持集合，**但是 MySQL、Oracle、DB 2、SQLite 都不支持`[]`。**

`[]` 用来制定一个字符集，必须匹配指定位置的一个字符，例子如下：

```sql
select cust_contact
from Customers
where cust_contact like '[JM]%';
```

以上语句用于查找以字母 J 或者 M 开头的联系人。

可以用 `^` 进行否定，如下：
```sql
where cust_contact like '[^JM]%';
```

此时则是查找任意不以 J 和 M 开头的联系人，也可以用 `not` 实现同样的操作。


---

<br/>


<br/>


<br/>



## 6.2  使用通配符的技巧

- **不要过度使用通配符**。通配符比其他搜索要耗费更长的处理时间。
- **尽量不把通配符放在搜索模式开头的位置**，这样搜索起来是最慢。

---

<br/>


<br/>


<br/>


<br/>



# 7.   计算字段

## 7.1  计算字段

存储在数据表中的数据一般不是应用程序所需要的格式，一般需要把数据经过处理之后再输出。

> **字段 field**
> 基本上和列的意思相同，但是并**不是表中实际的列**，而是**经过处理之后输出的列**。

---

<br/>


<br/>


<br/>



## 7.2  拼接字段

不同的 DBMS 有不同的拼接语法，如下：

```sql
-- select vend_name + ' (' + vend_country + ') '      -- SQL Server
-- select vend_name || ' (' || vend_country || ') '   -- DB2  Oracle  PostgreSQL  SQLite
-- select Concat(vend_name,' (',vend_country,')')     -- MySQL  MariaDB
from Vendors;
```


许多数据库（不包含 MySQL）保存填充为列宽的文本值，拼接之后会输出多余的空格，可以使用以下函数来去掉空格：
- `rtrim()`：去掉字符串右边的空格
- `ltrim()`：去掉字符串左边的空格
- `trim()`：去掉两端的空格
```sql
select Concat(trim(vend_name),' (',trim(vend_country),')')     -- MySQL  MariaDB
from Vendors;
```


<br/>

**使用别名**

新计算的列实际上并没有名字，**未命名的类不能用于客户端**。使用 **`as` 关键字给计算字段命名**：
```sql
select vend_name,concat(vend_name,'(',vend_country,')') as vend_title
from Vendors
order by vend_name;
```

<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-11-01_161113.jpg" width = 600 /> </div>

> **别名其他用途**
> 实际的表列的名字中包含不合法字符时重命名。

> 别名的名字可以是一个单词 ，也可以是一个字符串，**如果是字符串，就应该括在引号中**。**多单词名字会给客户端带来各种问题，通常别名是一个单词**。

---

<br/>


<br/>


<br/>


## 7.3  执行算术计算

SQL 可以执行 `+ - * /` 四种基本运算，也可以用括号表明优先级，例子如下：
```sql
select prod_id,
	   quantity,
	   item_price,
	   quantity*item_price as expandec_price
from OrderItems
where order_num = 20008;
```

> **测试计算**
> **`select` 语句为测试、检验函数和计算提供了很好的方法。把 `from` 子句去掉就是简单访问和处理表达式**，例如 `select 2*3；` 返回计算结果。


---

<br/>


<br/>


<br/>


<br/>



# 8.  函数

## 8.1  函数

每一个 DBMS 都有特定的函数，只有少数几个函数被所有主流的 DBMS 等同的支持。

<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-11-04_221214.jpg" width = 600 /> </div>


---

<br/>


<br/>


<br/>



## 8.2  使用函数

大多数 SQL 实现支持以下类型的函数：
- 处理文本字符串（大小写转化等）
- 算术操作
- 处理日期和时间值，提取特定成分
- 格式化函数
- 返回特殊信息

### 8.2.1  文本处理函数

**`upper()`**：转化为大写
```sql
select vend_name,upper(vend_name) as vend_name_upcase
from Vendors
order by vend_name;
```


其他常用文本处理函数如下：
<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-11-04_221933.jpg" width = 600 /> </div>


> **SOUNDEX**
> 按英语发音进行匹配，即使字符串和搜索条件不一样，但是如果发音相似也会被检索出来。


---

<br/>


<br/>



### 8.2.2  日期和时间处理函数

**日期和时间处理函数可移植性最差**。

- **SQL Server**：检索 2020 年所有订单，实用 `date_part()`
```sql
select order_num
from Orders
-- where date_part(yy,order_date) = 2020;  -- SQL Server版本
-- where date_part('year',order_date) = 2020;  -- PostgreSQL版本
```

- **Oracle** 版本 使用 `extract()`，PostgreSQL 也支持该函数。
```sql
select order_num
from Orders
where extract(year from order_date) =2020;
```

- **MySQL 和 MariaDB** 可以用 `year()` 提取年份
```sql
where year(order_date) = 2020;
```

---

<br/>


<br/>



### 8.2.3  数值处理函数

**数值处理函数是最一致、最统一的函数**，常用数值处理函数如下：

<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-11-04_223603.jpg" width = 600 /> </div>


---

<br/>


<br/>


<br/>


<br/>



# 9.  汇总数据

## 9.1  聚集函数

**有时候需要汇总数据，而不需要检索出来**，此时使用聚集函数，如下：

<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-11-04_223735.jpg" width = 600 /> </div>

### 9.1.1  `avg()` 函数

avg 函数只能作用于**单个列**的某些或者全部行（可以使用 `where` 子句过滤），**也可以作用于计算字段**。

<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-11-22_181520.jpg" width = 600 /> </div>

avg 函数只能作用于**数值列**，作用于非数值列时，结果无效。

<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-11-22_181750.jpg" width = 600 /> </div>

> **avg 函数忽略 NULL 值**。

---

<br/>


<br/>


### 9.1.2   `count()` 函数

- **`count(*)`**：对表中行进行计数，**不会忽略 NULL**。
- `count(column)`：对特定列具有值的行进行计数，**忽略 NULL 值**。**也可作用于计算字段**。

---

<br/>


<br/>


### 9.1.3  `max()` 函数

要求指定列名，可以用于计算字段。忽略 NULL。

> 非数值数据使用 MAX
> max 函数**一般用于找出最大的数值或者日期值**。许多 DBMS 允许返回任意列的最大值。

---


<br/>


<br/>


### 9.1.4  `MIN()` 函数

和 `MAX()` 功能相反，其余特性一样。

---

<br/>


<br/>


### 9.1.5  `SUM()` 函数

可以使用计算字段来计算合计值。

---

<br/>


<br/>


<br/>


## 9.2  聚集不同值

以上 5 个聚集函数都可以使用：
- 对于所有行执行：指定 `all` 或者不指定参数（`all` 为默认参数）。
- 只对于不同值执行：指定 `distinct` 参数。

```sql
SELECT AVG(DISTINCT prod_price) AS avg_price
FROM Products
WHERE vend_id = 'DLL01';
```

> `DISTINCT` 不能用于 `COUNT(*)`。
> `DISTINCT` 必须使用列名，**不能用于计算或表达式**。**可以使用计算字段**。

<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-11-22_183623.jpg" width = 600 /> </div>


> 在使用 DISTINCT 关键字时，**不能直接使用计算或表达式来指定要去重的内容**，必须使用列名。 
>  
> 举个例子，假设有一个表格存储了商品的名称和价格，我们想要查询不同价格范围内的商品数量。我们可能会尝试以下查询语句：
> 
> ```sql
> SELECT DISTINCT CASE
>     WHEN price < 50 THEN 'Low'
>     WHEN price >= 50 AND price < 100 THEN 'Medium'
>     WHEN price >= 100 THEN 'High'
> END AS price_range,
> COUNT(*) AS count
> FROM products
> GROUP BY price_range;
> ```
> 
> 上述查询语句尝试使用计算表达式来定义要去重的内容，即根据价格范围来进行去重。然而，这是不合法的，因为 DISTINCT 关键字只能用于列名，而不能用于计算或表达式。 
>  
> 为了解决这个问题，我们需要使用子查询或临时表来先计算出价格范围，然后再使用 DISTINCT 关键字。以下是修正后的查询语句：
> ```sql
> SELECT price_range, COUNT(*) AS count
> FROM (
>     SELECT CASE
>         WHEN price < 50 THEN 'Low'
>         WHEN price >= 50 AND price < 100 THEN 'Medium'
>         WHEN price >= 100 THEN 'High'
>     END AS price_range
>     FROM products
> ) AS temp
> GROUP BY price_range;
> ```

---

<br/>


<br/>


<br/>


## 9.3  组合聚集函数

`select` 语句可以包含多个聚集函数：
```sql
SELECT COUNT(*) as num_items,
		MIN(prod_price) as price_min,
		MAX(prod_price) as price_max,
		AVG(prod_price) as price_avg
FROM Products;
```

---


<br/>


<br/>


<br/>


<br/>

# 10.   分组数据

## 10.1  数据分组

之前的计算都是在表的**所有数据**或者**特定的匹配数据**（`where` 子句过滤出来的行）上面进行的。

如果计算对象时表中某些**逻辑组**，就需要进行分组。

---

<br/>


<br/>


<br/>


## 10.2  创建分组

**`group by` 子句创建分组**：
```sql
select vend_id,count(*) as num_prods
from Products
group by vend_id;
```

<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-11-25_102505.jpg" width = 600 /> </div>

以上例子中，按照 vend_id 将表中所有行分成若干逻辑组，然后在每个逻辑组中，进行聚集函数操作。


<br/>


<span style="background:#fff88f">**`group by` 子句使用规则**</span>
- `group by` 子句可以**包含任意数目列**，可以对分组进行嵌套。
<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-11-25_103301.jpg" width = 600 /> </div>

- 每一列都必须是**检索列**，或者**有效表达式（但是不能是聚集函数）**。
- 大多数 SQL 实现不允许 group by 列带有长度可变的数据类型。
> 长度可变类型是指可以存储不同长度数据的数据类型，例如 `varchar` 和 `text` 。固定长度类型是指只能存储固定长度数据的数据类型，例如 `char` 和 `int` 。

- 除了聚集函数外，**select 语句中每一列都必须在 group by 子句中出现**。
- 分组列中具有 NULL 行，所有 NULL 行将分为一组。
- group by 子句必须在 where 子句之后，order by 之前。

> **根据相对位置指定列**
> `group by 2,1` 即根据 select 语句中第 2 个和第 1 个列分组，类似于 [[#^c8cc01|按位置排序]]。**并非所有 SQL 实现都支持，且容易出错。**


---

<br/>


<br/>


<br/>


## 10.3  过滤分组

`where` 过滤的是行，`having` 过滤分组。

```sql
select cust_id,count(*) as orders
from Orders
group by cust_id
having orders >= 2;
```

> **having 支持所有 where<span style="background:#fff88f"> 操作符</span>**
> 例如，通配符条件和带有多个操作符的子句。

> **having 和 where 差别**
> **where 在分组前进行过滤，而 having 在分组后进行过滤**。当一起使用时，被 where 剔除的数据，不会包括在分组中。
> 如果不指定分组，大多数 DBMS 同等对待两者，可以使用 having 替代 where。

两者一起使用的例子：
```sql
select vend_id,count(*) as num_prods
from Products
where prod_price >= 4
group by vend_id
having num_prods>=2;
```

---

<br/>


<br/>


<br/>


<br/>

# 11. 子查询


## 11.1  子查询

之前所学 select 语句都是简单查询，即从单个表中检索数据的单条语句。

> **查询**
> 任何 SQL 语句都是查询，但是次术语一般指 select 语句。

数据使用的**关系表**的关系如下：

<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/214b7a1fc2ae8b34913d9fb07af9d41.jpg" width = 700 /> </div>


---


<br/>


<br/>


<br/>


## 11.2  使用子查询进行过滤

假如需要列出订购物品 RGAN01 的所有顾客，步骤如下：
- OrderItems 表中检索包含物品 RGAN 01 的所有订单编号 order_num
- Orders 表中检索包含上一步订单编号的所有顾客 id  cust_id
- Customers 表中返回对应顾客 id 的信息

可以使用子查询进行嵌套，子查询总是**从内向外**：
```sql
select cust_name,cust_contact
from Customers
where cust_id in ( select cust_id
				   from Orders
				   where order_num in ( select order_num
										from OrderItems
										where prod_id='RGAN01'));
```

> 作为**子查询**的 select 语句只能是**查询单个列**。

> 子查询并不是这类数据检索最有效的方法。


---

<br/>


<br/>


<br/>


## 11.3  作为计算字段使用子查询

可以使用子查询来作为计算字段。

例如假如需要显示 Customers 表中每个顾客的订单总数：
```sql
select cust_name,
	   cust_state,
	   (select count(*)
	    from Orders
	    where Orders.cust_id = Customers.cust_id) as orders
from Customers
order by cust_name;
```

> **完全限定名**
> 当涉及的多个表具有相同的列名时，需要使用完全限定名进行区分。



---


<br/>


<br/>


<br/>


<br/>


# 12.  联结表

## 12.1  联结

### 12.1.1  关系表

**关系表**：把信息分解为多个表，一类数据一个表，各个表通过某些相同的值相互联系。
- 相同信息不用重复，不浪费空间。
- 信息发生改动，不同在同一个表中同时修改多行。

---

<br/>


<br/>

### 12.1.2  为何使用联结

**联结**是一种机制，用来在一条 select 语句中关联表。

联结不是物理实体，DBMS 根据需要建立联结，在查询执行期间一直存在。

---

<br/>


<br/>


<br/>

## 12.2  创建联结

例子如下：
```sql
select vend_name,prod_name,prod_price
from Vendors,Products
where Vendors.vend_id = Products.vend_id;
```

<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-11-25_155529.jpg" width = 600 /> </div>

### 12.2.1  where 子句的重要性

在联结两个表时，实际将第一个表中的每一行和第二个表中的每一行进行匹配，where 作为过滤条件值包含哪些匹配条件的行。

> 没有联结条件的表关系返回结果为笛卡儿积。


---

<br/>


<br/>

### 12.2.2 内联结

目前使用的联结为**等值联结**，也称为**内联结**。可以使用以下不同的语法：
```sql
select vend_name,prod_name,prod_price
from Vendors
inner join Products on Vendors.vend_id = Products.vend_id;
```


---

<br/>


<br/>

### 12.2.3  联结多个表

```sql
select prod_name,vend_name,prod_price,quantity
from OrderItems,Products,Vendors
where Products.vend_id = Vendors.vend_id
	  and  OrderItems.prod_id = Products.prod_id
	  and order_num = 20007;
```

> 联结处理可能非常耗费资源.


---


<br/>


<br/>


<br/>


<br/>


# 13.  高级联结

## 13.1 使用别名

可以给表名起别名：
- 缩短 SQL 语句
- 在一条 select 语句中多次使用相同的表

**表别名**可以作用于 select 的列表、order by 子句以及其他语句部分。


---

<br/>


<br/>


<br/>

## 13.2  使用不同类型的联结

### 13.2.1  自联结

自联结即在一个表中进行联结。

例如，需要在查找与 Jim Jones 同一公司的所有顾客的邮箱，使用子查询如下：
```sql
select cust_id,cust_name,cust_contact
from Customers
where cust_name = (select cust_name
				  from Customers
				  where cust_contact = 'Jim Jones');
```

结合 as 使用**内联结**如下：
```sql
select c1.cust_id,c1.cust_name,c1.cust_contact
from Customers as c1,Customers as c2
where c1.cust_name = c2.cust_name
	  and c2.cust_contact = 'Jim Jones';
```

---

<br/>


<br/>

### 13.2.2  自然联结

自然连接是 SQL 中一种特殊的联结，它会**自动连接两个表**，并在两个表中具有**相同名称的列**之间创建连接。
```sql
SELECT * 
FROM table1 
NATURAL JOIN table2;
```

---


<br/>


<br/>

### 13.2.3  外联结

外联结是 SQL 中一种联结类型，它允许您查询两个表中的数据，**即使两个表中没有匹配的记录**。 外联结有两种类型：左外联结和右外联结。 
 
- 左外联结会从**左表中返回所有记录**，即使右表中没有匹配的记录。
- 右外联结会从**右表中返回所有记录**，即使左表中没有匹配的记录。 

```sql
SELECT * 
FROM table1 
LEFT OUTER JOIN table2 ON table1.column1 = table2.column2;

SELECT * 
FROM table1 
RIGHT OUTER JOIN table2 ON table1.column1 = table2.column2;
```


---

<br/>


<br/>


<br/>


## 13.3  使用带聚集函数的联结

利用联结来实现在**多个表中进行汇总数据**。

例如，检索所有顾客，以及每个顾客所下订单数：
```sql
select Customers.cust_id,
	   count(Orders.order_num) as num_ord
from Customers
inner join Orders on Customers.cust_id = Orders.cust_id
group by Customers.cust_id;
```

<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2023-11-25_164802.jpg" width = 600 /> </div>

---


<br/>


<br/>


<br/>


<br/>


# 14. 组合查询

## 14.1  组合查询

执行多个查询，将结果作为一个查询结果返回。应用场景：
- 从不同表返回结构数据。
- 对一个表执行多次查询，按一个查询返回结果。

---

<br/>


<br/>


<br/>


## 14.2  创建组合查询

### 14.2.1  使用 UNION

```sql
select cust_name,cust_contact,cust_email
from Customers
where cust_state IN ('IL','IN','MI')

UNION

select cust_name,cust_contact,cust_email
from Customers
where cust_name = 'Fun4All';
```

---

<br/>


<br/>

### 14.2.2 UNION 规则

**如果结合 UNION 使用的 select 语句遇到不同的列名，会返回第一个名字。**

---

<br/>


<br/>

### 14.2.3  包含或者取消重复的行

默认从查询结果集中去除重复的行。可以使用 `UNION ALL` 来返回所有匹配行。


---

<br/>


<br/>


### 14.2.4  对组合查询结果排序

只能使用一条 order by 语句，且必须位于最后。order by 的列名必须是可见的，需要注意 14.2.2 UNION 规则。


---


<br/>


<br/>


<br/>


<br/>


# 15.  插入数据

## 15.1  数据插入

### 15.1.1  插入完整的行

```sql
insert into Customers
values(1000000006,
	   'Toy Land',
	   '123 Any Street',
	   'New York',
	   'NY',
	   '11111',
	   'USA',
	   NULL
	   NULL);
```
如果没有值，使用 NULL（假设运行填 NULL）。各类按照在表中定义中出现的次数填充。**这种语法，可移植差，高于依赖于表中列定义的次序，不推荐这种写法**。

建议使用以下写法：
```sql
insert into Customers(cust_id,
					 cust_name,
					 cust_address,
					 cust_city,
					 cust_state,
					 cust_zip,
					 cust_country,
					 cust_contact,
					 cust_email)
values(1000000006,
	   'Toy Land',
	   '123 Any Street',
	   'New York',
	   'NY',
	   '11111',
	   'USA',
	   NULL
	   NULL);
```

---

<br/>


<br/>

### 15.1.2  插入部分行

即在插入行时，某些列被省略了。
```sql
insert into Customers(cust_id,
					 cust_name,
					 cust_address,
					 cust_city,
					 cust_state,
					 cust_zip,
					 cust_country)
values(1000000006,
	   'Toy Land',
	   '123 Any Street',
	   'New York',
	   'NY',
	   '11111',
	   'USA');
```
以上例子中忽略最后两列，列可以被忽略的条件：
- 允许 NULL 值
- 给出了默认值

---

<br/>


<br/>

### 15.1.3  插入检索出的数据

顾名思义，将 select 语句结果插入表中：
```sql
insert into Customers(cust_id,
					 cust_name,
					 cust_address,
					 cust_city,
					 cust_state,
					 cust_zip,
					 cust_country)
select  cust_id,
	    cust_name,
		cust_address,
		cust_city,
		cust_state,
		cust_zip,
		cust_country
from CustNew;
```

select 的列名并**不需要和 insert into 对应的列名一样**，使用的是列的位置。

---

<br/>


<br/>


<br/>

## 15.2  从一个表复制到另一个表

```sql
create table CustCopy 
as select *
from Customers;
```

---

<br/>


<br/>


<br/>


<br/>


# 16.  更新和删除数据

## 16.1  更新数据

使用 `update` 选中 table，用 `set` 对特定列进行更新。**不要忽略 `where` 进行过滤**，否则就是所有行进行更新。

```sql
update Customers
set cust_contact = 'Sam Roberts',
    cust_email = 'sam@toyland.com'
where cust_id = 100000006;
```

如果需要删除某个值，可以将其设置为 NULL（如果可以设置为 NULL）。

---

<br/>


<br/>


<br/>


## 16.2  删除数据

使用 <span style="background:#fff88f">delete from </span>选中 table 进行删除操作，**不要忽略使用 `where` 子句进行过滤，否则就是删除所有行。**

**`from` 关键字是可选的**，但是为了保证可移植性，最好写上。

```sql
delete from Customers
where cust_id = 1000000006;
```

> **友好的外键**
> ```sql
> CREATE TABLE customers (
>   customer_id INT PRIMARY KEY,
>   name VARCHAR (255),
>   address VARCHAR (255),
>   phone_number VARCHAR (255)
> );
> 
> CREATE TABLE orders (
>   order_id INT PRIMARY KEY,
>   customer_id INT,
>   number_of_items INT,
>   order_date DATE,
>   FOREIGN KEY (customer_id) REFERENCES customers (customer_id)
> );
> ```
> 外键是指一个表中的列引用另一个表中的主键。外键可以用于确保数据的一致性，并防止数据丢失。 
> 例如，有一个  customers  表和一个  orders  表。 customers  表包含客户的姓名、地址和电话号码。 orders  表包含订单的数量、日期和客户的 ID。我们可以**将  customers  表中的  customer_id  列设置为  orders  表中的外键**。这意味着，当我们在  orders  表中插入一条新记录时，必须指定一个有效的  customer_id 。**如果尝试插入一个无效的  customer_id ，则 SQL 会拒绝该操作。** 
>  
> **外键可以用于确保数据的一致性**。如果尝试在 Customer 中删除一行，而在 orders 中有该顾客对应的订单，DBMS 会拒绝该操作。 如果我们确实需要删除该行，则必须先删除 orders 表中与该行相关的所有订单。这样，我们就可以确保数据的一致性。

^160d09

<br/>


**delete from 删除的是表的行**，而不是表本身。如果需要**删除表中所有行**：
```sql
truncate table table_name;
```
TRUNCATE TABLE 命令用于删除表中所有数据，但保留表的结构。 
TRUNCATE TABLE 命令与 DELETE FROM 命令的区别在于——<span style="background:#fff88f">DELETE FROM 命令会在删除数据之前将数据复制到一个临时表中</span>，而 <span style="background:#fff88f">TRUNCATE TABLE 命令则会直接删除数据，速度更快，但是不可恢复</span>。 


---


<br/>


<br/>


<br/>


<br/>

# 17.  创建和操纵表

## 17.1  创建表

### 17.1.1  表创建基础

创建表例子：

```sql
create table Products
(
	prod_id    char(10)    not null,
	vend_id    char(10)    not null,
	prod_name  char(254)   not null,
	prod_price decimal(8,2) not null,
	prod_desc  varchar(1000) null
)
```

---

<br/>


<br/>

### 17.1.2  使用 null 值

在不指定 not null 时，多数但不是全部 DBMS 认为默认指定 null。

允许 null 的列不能作为唯一标识。

---


<br/>


<br/>


### 17.1.3  指定默认值

可以在 not null 之后 使用关键字 `default` 指定一个默认值：
```sql
create table Order Items
(
	quantity integer  not null  default 1,
	-- ....
)
```

**默认值通常用于日期或者时间戳**。MySQL 指定 `default current date()`。


---

<br/>


<br/>


<br/>

## 17.2  更新表

- 理想情况下，不要对表中包含数据时对其进行更新。
- 许多 DBMS 允许给现有表增加列，但是对于数据类型有所限制。
- 许多 DBMS 不允许删除或者更改表中列。
- 多数 DBMS 允许重命名表中列。


**表增加列**
```sql
alter table Vendors
add vend_phone char(20);
```


<br/>

**表删除列**
```sql
alter table Vendors
drop column vend_phone;
```
并不是所有 DBMS 都支持这样的操作。**复杂表结果更改一般需要创建新表，复制数据到新表，然后删除旧表，重命名新表。**

---

<br/>


<br/>


<br/>

## 17.3  删除表

```sql
drop table CustCopy;
```


---

<br/>


<br/>


<br/>

## 17.4  重命名表

不同 DBMS 有不同要求。
- mysql 使用 `rename` 语句
- sqlite 使用 `alter table` 语句


---

<br/>


<br/>


<br/>


<br/>

# 18.  视图

## 18.1  视图

视图时虚拟的表，只包含使用时动态检索数据的查询。视图仅仅是用来查看存储在别处数据的一种设施，视图本身并不包含数据。

> **创建视图后，如果原始表中的数据发生变动，视图的数据也会相应地发生变动。** 
> 
> **视图是虚拟的表**，它基于查询定义，并从一个或多个基本表中检索数据。当查询视图时，它会**动态地从基本表中获取最新的数据**。 
>  
> 因此，如果在查询视图时修改了原始表的数据，视图将反映这些变化。无论是插入、更新还是删除操作，只要它们影响到视图的查询定义所涉及的基本表，视图将显示更新后的数据。 这种特性使得视图在提供简化和抽象数据访问的同时，保持与基本表之间的一致性。 

---

<br/>


<br/>


<br/>



## 18.2  创建视图

### 18.2.1  利用视图简化复杂度联结

使用 `create view` 创建视图，例子如下：
```sql
create view ProductCustomers as
select cust_name,cust_contact,prod_id
from Customers,Orders,OrderItems
where Customers.cust_id = Orders.cust_id
	and OrderItems.order_num = Orders.order_num;
```

视图创建之后，可以多次从重复使用，简化了 SQL 语句。

> **创建可以重用的视图**
> 创建视图时，不绑定特定数据，需要使用时，使用 where 过滤即可。


---

<br/>


<br/>

### 18.2.2  用视图重新格式化检索出的数据

在 [[#7.2 拼接字段]]中有一个格式化输出的例子：
```sql
-- select vend_name + ' (' + vend_country + ') '      -- SQL Server
-- select vend_name || ' (' || vend_country || ') '   -- DB2  Oracle  PostgreSQL  SQLite
-- select Concat(vend_name,' (',vend_country,')')     -- MySQL  MariaDB
from Vendors;
```

**假如需要多次使用这种格式，可以创建一个视图：**
```sql
create view VendorLocations as
select Concat(vend_name,' (',vend_country,')') -- MySQL MariaDB from Vendors;
from Vendors;
```

接下使用即可：
```sql
select * from VendorLocations;
```


---

<br/>


<br/>

### 18.2.3  使用视图过滤数据

可以在创建视图时，使用 `where` 子句过滤数据，先过滤掉不需要的数据。之后使用时，也可以根据要求对于视图，使用 `where` 子句。


---


<br/>


<br/>


<br/>


<br/>


# 19.  存储过程

## 19.1  存储过程

存储过程是一组预先编写好的 **SQL 语句集合**，它们被保存在数据库中并可以被重复调用。存储过程通常用于执行特定的任务或操作，**可以接受参数并返回结果**。

> SQLite 不支持存储过程。

---


<br/>


<br/>


<br/>


<br/>


# 20.  事务处理

## 20.1  事务处理

事务处理是指在数据库操作中，**将一系列操作作为一个原子单元进行处理**，要么全部执行成功，要么全部回滚到操作之前的状态，以确保数据的一致性和完整性。
- 事物：一组 sql 语句。
- 回退：撤销指定 sql 语句。
- 提交：将未存储的 sql 语句写入数据库表。
- 保留点：设置临时占位符，可以发布回退。

---

<br/>


<br/>


<br/>

## 20.2  控制事物处理

**不同 DBMS 实现有不同语法。**

在 mysql 中使用 `start transaction`：
```sql
start transaction
-- ....
```

### 20.2.1  使用回退

`rollback` 用于撤销语句。

---

<br/>


<br/>

### 20.2.2  提交

一般的 sql 语句是针对数据库表直接执行和编写的。但是在事物处理块中，提交一般不会隐式进行，需要使用 `commit` 语句。


---

<br/>


<br/>

### 20.2.3  保留点

使用简单的 `rollback` 和 `commit` 语句可以写入或者撤销这个事物，但是复杂度事物可能需要部分提交或回退。使用保留点，可以回退到某个保留点。

不同 DBMS 实现有不同的语法。

在 mysql 中设置保留点：
```sql
savepoint delete1;
```

在 mysql 中进行回退：
```sql
rollback to delete1;
```


---

<br/>


<br/>


<br/>


<br/>

# 21.  游标

## 21.1  游标

sql 检索操作返回的数据为结果集。简单 sql 语句没法获得结果集的某些特定行。游标是存储在 DBMS 服务上的数据库查询，可以对结果集进行滚动或者浏览。


> SQLite 支持的游标称为步骤。

- 可以执行定向操作（向前、向后、第一、最后一行）
- 可以将游标标志为只读。

---


<br/>


<br/>


<br/>

## 21.2  使用游标

### 21.2.1  创建游标

不同 DBMS 实现有不同语法。在 mysql 中：
```sql
declare CustCurosr cursor
for
select * from Customers
where cust_email is null;
```


### 21.2.2  使用游标

先打开游标，结束使用之后需要关闭，此外 DBMS 需要明确使用游标所占资源。

**打开游标**：
```sql
open cursor CustCursor;
```

**关闭游标** ：
```sql
close CustCursor;
```

---

<br/>


<br/>


<br/>


<br/>


# 22.  高级 SQL 特性

## 22.1  约束

### 22.1.1  主键

- 任意两行主键必须不同。
- 每行都必须有主键（不允许为 NULL）。
- 大部分 DBMS 对于主键列，从不修改或更新。
- **主键值不能复用（即使被删除，其主键值也不能分配给新的行）**。

> **SQLite 只允许在创建表时定义主键。**

定义主键例子：
```sql
create table Vendors
(
	vend_id    char(10)    not null primary key,
	-- ........
);
```

---

<br/>


<br/>

### 22.1.2  外键

[[#^160d09|参考此处]]

外键定义例子，以下声明说明，**本表中任何 cust_id 的值都是 Customers 的 cust_id 的值。**
```sql
create table Orders
(
	order_num    integer    not null primary key,
	-- ........
	cust_id      char(10)   not null references Customer(cust_id)
);
```


在表创建之后，添加外键的例子：
```sql
alter table Orders
add constraint
foreign key (cust_id) references Customers (cust_id);
```


---

<br/>


<br/>

### 22.1.3  唯一约束

用于保证某一列的数据时唯一的。和主键区别如下：
- 可以包含多个唯一约束列。
- 唯一约束可以包含 null 值。
- 唯一约束列可以修改、更新、重复使用（删除之后分配给新的行）。
- **唯一约束不能用于定义外键。**

唯一约束定义方式——在创建表时，在行尾使用 `unique` 关键字。



---

<br/>


<br/>

### 22.1.4  检查约束

用于保证一类中的数据满足条件，常见用途：
- 检查最小或者最大值。
- 指定范围。
- 只允许特定的值。

使用关键字 `check` +条件，进行定义。 例子如下：
```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(255),
    age INT,
    salary DECIMAL(10, 2),
    CONSTRAINT check_age     CHECK (age >= 18),
    CONSTRAINT check_salary  CHECK (salary > 0)
);
```

---

<br/>


<br/>


<br/>


## 22.2  索引

用来排序数据以加快搜索和排序操作对速度。可以在一个或多个列上定义索引，使得 **DBMS 保存到是一个排序之后的表**。
- 改善检索操作性能。但是降低数据插入和修改和删除的速度。
- 索引数据可能占用大量存储空间。

**创建索引列**：
```sql
create index prod_name_ind
on Products(prod_name);
```
**使用索引检索**：
```sql
select *
from Products
where prod_name = 100;
```
此时，在进行对于 prod_name 的检索时，会使用索引 prod_name_ind 来加快搜索。

---

<br/>


<br/>


<br/>

## 22.3 触发器

触发器是**特殊的存储过程**。在特定数据库互动发生时自动执行，可以和特定表上的 insert、updata、delete 操作关联，当操作发生时自动执行。

> [[#22.1 约束]]处理比触发器快，如果可以，尽量使用约束。


---