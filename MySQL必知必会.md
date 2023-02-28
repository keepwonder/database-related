# MySQL必知必会

## 实践篇

### 01 | 存储：一个完整的数据存储过程是怎样的？

```mysql
-- 创建数据库
CREATE DATABASE demo;

-- 删除数据库
DROP DATABASE DEMO;

-- 查看数据库
SHOW DATABASES;

-- 创建数据表
CREATE TABLE demo.test
(
	bracode text,
    goodsname text,
    price int
);

-- 查看表结构
DESCRIBE demo.test;
DESC demo.test;

-- 查看所有表
SHOW TABLES;

-- 添加主键
ALTER TABLE demo.test
ADD COLUMN itemnuber int PRIMARY KEY AUTO_INCREMENT;

-- 向表中添加数据
INSERT INTO demo.test(barcode, goodsname, price) VALUES ('0001', '本', 3);
```



`面试题`: 

select  count(*) from t;  t中有id(主键)，name，age, sex 4个字段。假设数据10条，对sex添加索引。用explain 查看执行计划发现用了sex索引，为什么不是主键索引呢?主键索引应该更快的

`答案`:  

- MySQL Innodb的主键索引是一个B+树，数据存储在叶子节点上，10条数据，就有10个叶子节点。

- sex索引是辅助索引，也是一个B+树，不同之处在于，叶子节点存储的是主键值，由于sex只有2个可能的值：男和女，因此，这个B+树只有2个叶子节点，比主键索引的B+树小的多 

- 这个表有主键，因此不存在所有字段都为空的记录，所以COUNT(*)只要统计所有主键的值就可以了，不需要回表读取数据*

- SELECT COUNT(*) FROM t，使用sex索引，只需要访问辅助索引的小B+树，而使用主键索引，要访问主键索引的那个大B+树，明细工作量大，这就是为什么，优化器使用辅助索引的原因

```mysql
-- 创建表
CREATE TABLE demo.q1
(
	id int primary key auto_increment,
    name varchar(10),
    age int,
    sex int
);
-- 修改字段类型
ALTER TABLE demo.q1
MODIFY COLUMN sex char(1);

-- 添加数据
insert into demo.q1(name, age, sex) values ('张三1','20', '男');
insert into demo.q1(name, age, sex) values ('丽丽1','18', '女');
insert into demo.q1(name, age, sex) values ('张三2','23', '男');
insert into demo.q1(name, age, sex) values ('丽丽2','16', '女');
insert into demo.q1(name, age, sex) values ('张三3','21', '男');
insert into demo.q1(name, age, sex) values ('丽丽3','19', '女');
insert into demo.q1(name, age, sex) values ('张三4','22', '男');
insert into demo.q1(name, age, sex) values ('丽丽4','20', '女');
insert into demo.q1(name, age, sex) values ('张三5','29', '男');
insert into demo.q1(name, age, sex) values ('丽丽5','25', '女');

-- explain 执行计划 
mysql> explain select count(*) from q1;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | q1    | NULL       | index | NULL          | PRIMARY | 4       | NULL |   10 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+

-- 添加 sex 索引
ALTER TABLE demo.q1 ADD INDEX sex_index (sex);

-- explain 查询计划
mysql> explain select count(*) from q1;
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra       
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+------------+
|  1 | SIMPLE      | q1    | NULL       | index | NULL          | sex_index | 5       | NULL |   10 |   100.00 | Using index 
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+------------+
```

`思考题`：假设用户现在要销售商品，你能不能帮它设计一个销售表，把销售信息（商品名称、价格、数量、金额等）都保存起来？

```mysql
CREATE TABLE demo.sales
(
	goodsname text,
	salesprice decimal(10,2),
	quantity decimal(10,3),
	salesvalue decimal(10,3)
);
```







### 02 | 字段：这么多字段类型，该怎么定义？

数据类型

- 整数类型

  - TINYINT
  - SMALLINT
  - MEDIUMINT
  - INT(INTEGER)
  - BIGINT

- 浮点数类型和定点数类型

  - FLOAT

  - DOUBLE 

  - REAL 默认DOUBLE，如果SQL模式设定启用为“REAL_AS_FLOAT”,那么默认就是FLOAT，可以通过一下SQL语句实现

    - ```mysql
      SET sql_mode = "REAL_AS_FLOA"
      ```

  - FLOAT和DOUBLE的区别：其实就是，FLOAT 占用字节数少（4字节），取值范围小；DOUBLE 占用字节数多（8字节），取值范围也大。

  - 浮点数类型有个缺陷，就是不精准，举例：

    - ```mysql
      -- 建表
      CREATE TABLE demo.goodsmaster
      (
      	barcode TEXT,
      	goodsname TEXT,
      	price DOUBLE,
      	itemnumber INT PRIMARY KEY AUTO_INCREMENT
      );
      
      -- 插入数据
      INSERT INTO demo.goodsmaster(barcode, goodsname, price) values ('0001', '书', 0.47);
      INSERT INTO demo.goodsmaster(barcode, goodsname, price) values ('0002', '笔', 0.44);
      INSERT INTO demo.goodsmaster(barcode, goodsname, price) values ('0003', '胶水', 0.19);
      
      
      -- 查询数据
      mysql> select * from goodsmaster;
      +---------+-----------+-------+------------+
      | barcode | goodsname | price | itemnumber |
      +---------+-----------+-------+------------+
      | 0001    | 书        |  0.47 |          1 |
      | 0002    | 笔        |  0.44 |          2 |
      | 0003    | 胶水      |  0.19 |          3 |
      +---------+-----------+-------+------------+
      3 rows in set (0.00 sec)
      
      mysql> select sum(price) from goodsmaster;
      +--------------------+
      | sum(price)         |
      +--------------------+
      | 1.0999999999999999 |
      +--------------------+
      1 row in set (0.00 sec)
      
      MySQL 用 4 个字节存储 FLOAT 类型数据，用 8 个字节来存储 DOUBLE 类型数据。无论哪个，都是采用二进制的方式来进行存储的。比如 9.625，用二进制来表达，就是 1001.101，或者表达成 1.001101×2^3。看到了吗？如果尾数不是 0 或 5（比如 9.624），你就无法用一个二进制数来精确表达。怎么办呢？就只好在取值允许的范围内进行近似（四舍五入）。
      ```

      

  - 定点数 DECIMAL： 浮点数类型是把十进制数转换成二进制数存储，DECIMAL 则不同，它是把十进制数的整数部分和小数部分拆开，分别转换成十六进制数，进行存储。这样，所有的数值，就都可以精准表达了，不会存在因为无法表达而损失精度的问题。

  - MySQL 用 DECIMAL（M,D）的方式表示高精度小数。其中，M 表示整数部分加小数部分，一共有多少位，M<=65。D 表示小数部分位数，D<M。

  - 验证，把字段“price”的数据类型修改为 DECIMAL(5,2)：

    - ```mysql
      -- 修改字段类型
      ALTER TABLE demo.goodsmaster
      MODIFY COLUMN price DECIMAL(5,2);
      
      mysql> ALTER TABLE demo.goodsmaster
          -> MODIFY COLUMN price DECIMAL(5,2);
      Query OK, 3 rows affected (0.02 sec)
      Records: 3  Duplicates: 0  Warnings: 0
      
      mysql> select sum(price) from demo.goodsmaster;
      +------------+
      | sum(price) |
      +------------+
      |       1.10 |
      +------------+
      1 row in set (0.00 sec)
      ```

    - 

  - 总结：浮点类型取值范围大，但是不精准，适用于需要取值范围大，又可以容忍微小误差的科学计算场景（比如计算化学、分子建模、流体动力学等）；定点数类型取值范围相对小，但是精准，没有误差，适合于对精度要求极高的场景（比如涉及金额计算的场景）。

- 文本类型

  - CHAR(M)：固定长度字符串。CHAR(M) 类型必须预先定义字符串长度。如果太短，数据可能会超出范围；如果太长，又浪费存储空间。
  - VARCHAR(M)：可变长度字符串。VARCHAR(M) 也需要预先知道字符串的最大长度，不过只要不超过这个最大长度，具体存储的时候，是按照实际字符串长度存储的。
  - TEXT：字符串。系统自动按照实际长度存储，不需要预先定义长度。由于实际存储的长度不确定，MySQL 不允许 TEXT 类型的字段做主键。遇到这种情况，你只能采用 CHAR(M)，或者 VARCHAR(M)。只要不是主键字段，就可以按照数据可能的最大长度，选择这几种 TEXT 类型中的的一种，作为存储字符串的数据类型。
    - TINYTEXT：255 字符（这里假设字符是 ASCII 码，一个字符占用一个字节，下同）。
    - TEXT：65535 字符。
    - MEDIUMTEXT：16777215 字符。
    - LONGTEXT：4294967295 字符（相当于 4GB）。
  - ENUM：枚举类型，取值必须是预先设定的一组字符串值范围之内的一个，必须要知道字符串所有可能的取值。
  - SET：是一个字符串对象，取值必须是在预先设定的字符串值范围之内的 0 个或多个，也必须知道字符串所有可能的取值。

- 日期与时间类型

  | 类型      | 日期格式            | 占用字节数 |
  | --------- | ------------------- | ---------- |
  | YEAR      | YYYY                | 1          |
  | TIME      | HH:MM:SS            | 3          |
  | DATE      | YYYY-MM-DD          | 3          |
  | DATETIME  | YYYY-MM-DD HH:MM:SS | 8          |
  | TIMESTAMP | YYYY-MM-DD HH:MM:SS | 4          |

  为了确保数据的完整性和系统的稳定性，优先考虑使用 DATETIME 类型。因为虽然 DATETIME 类型占用的存储空间最多，但是它表达的时间最为完整，取值范围也最大。

  



### 03 | 表：怎么创建和修改数据表？

#### 创建表

```mysql
-- 建表
CREATE TABLE demo.importhead
(
	listnumber INT,
	supplierid INT,
	stocknumber INT,
	importtype INT DEFAULT 1,
	quantity DECIMAL(10,3),
	importvalue DECIMAL(10,2),
	recorder INT,
	recordingdata DATETIME
);

-- 插入数据查看默认值约束 importtype 字段默认1
INSERT INTO demo.importhead(listnumber,supplierid,stocknumber,quantity,importvalue,recorder,recordingdata) values
(3456, 1,1,10,100,1,'2020-12-10');

mysql> select * from demo.importhead;
+------------+------------+-------------+------------+----------+-------------+----------+---------------------+
| listnumber | supplierid | stocknumber | importtype | quantity | importvalue | recorder | recordingdata       |
+------------+------------+-------------+------------+----------+-------------+----------+---------------------+
|       3456 |          1 |           1 |          1 |   10.000 |      100.00 |        1 | 2020-12-10 00:00:00 |
+------------+------------+-------------+------------+----------+-------------+----------+---------------------+
1 row in set (0.00 sec)
```

#### 约束

- 主键约束
- 外键约束

- 默认值约束：置了默认约束，插入数据的时候，如果不明确给字段赋值，那么系统会把设置的默认值自动赋值给字段。

- 非空约束：非空约束表示字段值不能为空，如果创建表的时候，指明某个字段非空，那么添加数据的时候，这个字段必须有值，否则系统就会提示错误。
- 唯一性约束：唯一性约束表示这个字段的值不能重复，否则系统会提示错误。跟主键约束相比，唯一性约束要更加弱一些。
- 自增约束：自增约束可以让 MySQL 自动给字段赋值，且保证不会重复

#### 修改表

- 添加字段

- 修改字段

  ```mysql
  -- 赋值importhead表结构
  CREATE TABLE demo.importheadhist
  LIKE demo.importhead;
  
  -- 添加字段
  ALTER TABLE demo.importheadhist
  ADD confirmer INT;
  
  ALTER TABLE demo.importheadhist
  ADD confirmdate DATETIME;
  
  - -查看表结构
  mysql> desc demo.importheadhist;
  +---------------+---------------+------+-----+---------+-------+
  | Field         | Type          | Null | Key | Default | Extra |
  +---------------+---------------+------+-----+---------+-------+
  | listnumber    | int           | YES  |     | NULL    |       |
  | supplierid    | int           | YES  |     | NULL    |       |
  | stocknumber   | int           | YES  |     | NULL    |       |
  | importtype    | int           | YES  |     | 1       |       |
  | quantity      | decimal(10,3) | YES  |     | NULL    |       |
  | importvalue   | decimal(10,2) | YES  |     | NULL    |       |
  | recorder      | int           | YES  |     | NULL    |       |
  | recordingdata | datetime      | YES  |     | NULL    |       |
  | confirmer     | int           | YES  |     | NULL    |       |
  | confirmdate   | datetime      | YES  |     | NULL    |       |
  +---------------+---------------+------+-----+---------+-------+
  10 rows in set (0.00 sec)
  
  -- 修改字段 quantity 改成 importquantity 类型改为DOUBLE
  ALTER TABLE demo.importheadhist
  CHANGE quantity importquantity DOUBLE;
  
  mysql> desc demo.importheadhist;                                      
  +----------------+---------------+------+-----+---------+-------+     
  | Field          | Type          | Null | Key | Default | Extra |     
  +----------------+---------------+------+-----+---------+-------+     
  | listnumber     | int           | YES  |     | NULL    |       |     
  | supplierid     | int           | YES  |     | NULL    |       |     
  | stocknumber    | int           | YES  |     | NULL    |       |     
  | importtype     | int           | YES  |     | 1       |       |     
  | importquantity | double        | YES  |     | NULL    |       |     
  | importvalue    | decimal(10,2) | YES  |     | NULL    |       |     
  | recorder       | int           | YES  |     | NULL    |       |     
  | recordingdata  | datetime      | YES  |     | NULL    |       |     
  | confirmer      | int           | YES  |     | NULL    |       |     
  | confirmdate    | datetime      | YES  |     | NULL    |       |     
  +----------------+---------------+------+-----+---------+-------+     
  10 rows in set (0.00 sec)    
  
  
  -- 只改面字段类型
  ALTER TABLE demo.importheadhist
  MODIFY importquantity DECIMAL(10,3);
  
  mysql> desc demo.importheadhist;                                     
  +----------------+---------------+------+-----+---------+-------+    
  | Field          | Type          | Null | Key | Default | Extra |    
  +----------------+---------------+------+-----+---------+-------+    
  | listnumber     | int           | YES  |     | NULL    |       |    
  | supplierid     | int           | YES  |     | NULL    |       |    
  | stocknumber    | int           | YES  |     | NULL    |       |    
  | importtype     | int           | YES  |     | 1       |       |    
  | importquantity | decimal(10,3) | YES  |     | NULL    |       |    
  | importvalue    | decimal(10,2) | YES  |     | NULL    |       |    
  | recorder       | int           | YES  |     | NULL    |       |    
  | recordingdata  | datetime      | YES  |     | NULL    |       |    
  | confirmer      | int           | YES  |     | NULL    |       |    
  | confirmdate    | datetime      | YES  |     | NULL    |       |    
  +----------------+---------------+------+-----+---------+-------+    
  10 rows in set (0.00 sec)                                            
  
  
  -- 添加字段，指定在表中的位置
  ALTER TABLE demo.importheadhist
  ADD suppliername TEXT AFTER supplierid;
  
  mysql> desc demo.importheadhist;                                     
  +----------------+---------------+------+-----+---------+-------+    
  | Field          | Type          | Null | Key | Default | Extra |    
  +----------------+---------------+------+-----+---------+-------+    
  | listnumber     | int           | YES  |     | NULL    |       |    
  | supplierid     | int           | YES  |     | NULL    |       |    
  | suppliername   | text          | YES  |     | NULL    |       |    
  | stocknumber    | int           | YES  |     | NULL    |       |    
  | importtype     | int           | YES  |     | 1       |       |    
  | importquantity | decimal(10,3) | YES  |     | NULL    |       |    
  | importvalue    | decimal(10,2) | YES  |     | NULL    |       |    
  | recorder       | int           | YES  |     | NULL    |       |    
  | recordingdata  | datetime      | YES  |     | NULL    |       |    
  | confirmer      | int           | YES  |     | NULL    |       |    
  | confirmdate    | datetime      | YES  |     | NULL    |       |    
  +----------------+---------------+------+-----+---------+-------+    
  11 rows in set (0.00 sec)                                            
  ```

  

  `思考题`：将表 demo.goodsmaster 中的字段“salesprice”改成不能重复，并且不能为空。

  ```mysql
  ALTER TABLE demo.goodsmaster modify salsprice decimal(10,3) unique not null;
  ```

  

### 04 | 增删改查：如何操作表中的数据？

```mysql
-- 增
INSERT INTO 表名 [(字段名 [,字段名] ...)] VALUES (值的列表);

-- 插入查询结果
INSERT INTO 表名 （字段名）
SELECT 字段名或值
FROM 表名
WHERE 条件

-- 删
DELETE FROM 表名
WHERE 条件

-- 改
UPDATE 表名
SET 字段名=值
WHERE 条件

--查
SELECT *|字段列表
FROM 数据源
WHERE 条件
GROUP BY 字段
HAVING 条件
ORDER BY 字段
LIMIT 起始点，行数
```

