[TOC]

## 概念

​  分区将一张表分为几个分区进行存储，本质上还是一张表。分区键用于根据某个区间值、特定值列表或hash函数值执行数据的聚集，让数据更具分布规则分布在不同的分区中，让一个大对象变成一些小对象。

分区的优点：

- 可以在一个表中存储比在单个磁盘或文件系统分区上能保存的更多的数据。

- 可以轻松的移除或添加分区达到删除数据或新增存储数据的空间。

- 一些查询可以得到很好的优化。

- 跨多个磁盘分散数据查询，获得更大的吞吐量

## 分区表的创建

```mysql
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] tbl_name
    (create_definition,...)
    [table_options]
    [partition_options]
partition_options:
    PARTITION BY
        { [LINEAR] HASH(expr)
        | [LINEAR] KEY(column_list)
        | RANGE{(expr) | COLUMNS(column_list)}
        | LIST{(expr) | COLUMNS(column_list)} }
    [PARTITIONS num]
    [SUBPARTITION BY
        { [LINEAR] HASH(expr)
        | [LINEAR] KEY(column_list) }
      [SUBPARTITIONS num]
    ]
    [(partition_definition [, partition_definition] ...)]
    
partition_definition:
    PARTITION partition_name
        [VALUES
            {LESS THAN {(expr | value_list) | MAXVALUE}
            |
            IN (value_list)}]
        [[STORAGE] ENGINE [=] engine_name]
        [COMMENT [=] 'comment_text' ]
        [DATA DIRECTORY [=] 'data_dir']
        [INDEX DIRECTORY [=] 'index_dir']
        [MAX_ROWS [=] max_number_of_rows]
        [MIN_ROWS [=] min_number_of_rows]
        [TABLESPACE [=] tablespace_name]
        [NODEGROUP [=] node_group_id]
        [(subpartition_definition [, subpartition_definition] ...)]
subpartition_definition:
    SUBPARTITION logical_name
        [[STORAGE] ENGINE [=] engine_name]
        [COMMENT [=] 'comment_text' ]
        [DATA DIRECTORY [=] 'data_dir']
        [INDEX DIRECTORY [=] 'index_dir']
        [MAX_ROWS [=] max_number_of_rows]
        [MIN_ROWS [=] min_number_of_rows]
        [TABLESPACE [=] tablespace_name]
        [NODEGROUP [=] node_group_id]
```

分区选项一定要放在最后。

指定表的分区数`PARTITIONS num`时，必须将其表示为不带前导零的非零正整数，并且不是0.8E + 01或6-2等表达式，即使它的计算结果为整数值。不允许使用小数部分。

要么分区表上没有主键/唯一键，要么使用主键/唯一键都必须包含分区键。

**分区的名字不区分大小写。**

当NO_DIR_IN_CREATE服务器SQL模式生效时（没有指定该模式也不可以），分区定义中不允许使用DATA DIRECTORY和INDEX DIRECTORY选项。从MySQL 5.5.5开始，在定义子分区时也不允许使用这些选项。实际操作可知：即使你添加了这两个选项，再创建表的时候也会忽略它们。

## 查看分区

使用`SHOW CREATE TABLE`查看分区表的分区子句。

使用`SHOW TABLE STATUS`确认是否是分区表。

查询`INFORMATION_SCHEMA.PARTITIONS`表：

```mysql
SELECT * FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_NAME = 'table_name';
-- 简化版
SELECT PARTITION_NAME,TABLE_ROWS FROM information_schema.PARTITIONS WHERE TABLE_NAME = 'hash_par';
```

使用`EXPLAIN PARTITIONS select_statement`查看使用了哪些分区，和标准的[`EXPLAIN SELECT`](https://dev.mysql.com/doc/refman/5.5/en/explain.html)语句用法一样：

```mysql
EXPLAIN PARTITIONS select_statement;
```

EXPLAIN PARTITIONS 存在一些限制：

- 不能同时使用PARTITIONS和EXTENDED关键字。
- 如果用于解释非分区表，`partitions` 总是为`NULL`。

## 删除分区

和删除表一样，使用 `TRUNCATE` 清空分区（比`DELETE`要快），使用`DROP`删除分区：

```mysql
ALTER TABLE table_name TRUNCATE PARTITION partition_name;
ALTER TABLE table_name DROP PARTITION partition_name;
```

## 分区类型

通过`KYE`或`LINEAR KEY`分区时，可以使用`DATE,TIME,DATETIME`类型的字段作为分区字段。

通过`RANGE COLUMNS`和`LIST COLUMNS`分区时，可以使用`DATE,DATETIME`类型的字段作为分区字段。

其它的分区类型在要求一个返回整型数值或NULL的分区表达式。

### RANGE 分区

按范围分区的表的分区方式是：每个分区包含分区表达式值位于给定范围内的行。

范围应该是**连续但不重叠的**，并且使用`VALUES LESS THAN`运算符定义。

- NULL会被当作最小值来处理
- 支持使用`DATE,DATETIME`类型的字段作为分区键
- 支持使用`UNIX_TIMESTAMP,YEAR`等函数进行分区

```mysql
CREATE TABLE `range_test` (
  `ID` int(11) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8
/*!50100 PARTITION BY RANGE (ID)
(PARTITION P0 VALUES LESS THAN (5) ENGINE = InnoDB,
 PARTITION P1 VALUES LESS THAN (10) ENGINE = InnoDB,
 -- MAXVALUE表示最大的可能的整数值
 PARTITION P3 VALUES LESS THAN MAXVALUE ENGINE = InnoDB) */
-- /*!...*/ 是MySQL的扩展特性。是一种特殊的注释，其他的数据库产品当然不会执行。mysql特殊处理，会选择性的执行。特别注意 50100，它表示5.01.00 版本或者更高的版本，才执行。
-- 使用DATE类型的字段作为分区键
CREATE TABLE members (
    username VARCHAR(16) NOT NULL,
    email VARCHAR(35),
    joined DATE NOT NULL
)
PARTITION BY RANGE COLUMNS(joined) (
    PARTITION p0 VALUES LESS THAN ('1960-01-01'),
    PARTITION p1 VALUES LESS THAN MAXVALUE
);
-- 使用UNIX_TIMESTAMP函数进行分区
CREATE TABLE quarterly_report_status (
    report_id INT NOT NULL,
    report_updated TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
)
PARTITION BY RANGE ( UNIX_TIMESTAMP(report_updated) ) (
    PARTITION p0 VALUES LESS THAN ( UNIX_TIMESTAMP('2008-01-01 00:00:00') ),
    PARTITION p1 VALUES LESS THAN (MAXVALUE)
);
-- 使用YEAR等函数进行分区
CREATE TABLE employees (
    id INT NOT NULL,
    separated DATE NOT NULL DEFAULT '9999-12-31',
    store_id INT
)
PARTITION BY RANGE ( YEAR(separated) ) (
    PARTITION p0 VALUES LESS THAN (1991),
    PARTITION p1 VALUES LESS THAN MAXVALUE
);
```

`RANGE`分区类型特别适合以下几种情况：

- 需要删除旧数据时。可以很轻松地通过删除分区来删除旧数据。`ALTER TABLE employees DROP PARTITION p0;`
- 使用包含日期或时间值的列，或包含来自其他一些系列的值。
- 经常运行包含分区键的查询。

### LIST 分区

`LIST`分区时建立离散的值列表告诉数据库特定的值属于哪个分区。

与RANGE分区的区别：是LIST分区从属于一个枚举列表的值的集合，RANGE分区是从属于一个连续区间值的集合。

与RANGE分区不同，没有类似于*MAXVALUE*的值来匹配所有的情况；**所有被期望的值**都应该被包含进`PARTITION ... VALUES IN (...)`子句。

使用单个INSERT语句插入多行时，行为取决于表是否使用事务存储引擎：

- 对于支持事务的存储引擎，整个插入语句会被当做一个单一的事务。
- 对于不支持事务的存储引擎，在包含未匹配值的之前的记录会被插入，自己及其之后的不会。

可以使用`IGNORE`关键字忽略此类错误，如此一来，包含未匹配的值的记录不会插入，其余的都会被插入到数据库并且不会发出错误。

```mysql
MariaDB [MYISAM_TEST]> CREATE TABLE IF NOT EXISTS list_test (ID INT) MAX_ROWS=8 
PARTITION BY LIST (ID) (
    PARTITION P0 VALUES IN (1,2,3,4,5),
    PARTITION P1 VALUES IN (6,7,8)
);
MariaDB [MYISAM_TEST]> INSERT INTO list_test VALUES (1),(3),(9);
ERROR 1526 (HY000): Table has no partition for value 9
MariaDB [MYISAM_TEST]> INSERT IGNORE INTO list_test VALUES (1),(3),(9);
Query OK, 2 rows affected, 1 warning (0.00 sec)
Records: 3  Duplicates: 1  Warnings: 1
MariaDB [MYISAM_TEST]> SELECT * FROM list_test;
+------+
| ID   |
+------+
|    1 |
|    3 |
+------+
```

### COLUMNS 分区

列分区是`RANGE`和`LIST`分区的变种。COLUMNS分区允许在分区键中使用多个列。

`RANGE COLUMNS` 分区和`LIST COLUMNS` 分区都支持非整型（non-integer）字段作为值的范围或列的成员，以下是运行的数据类型：

- 所有的整数类型：TINYINT, SMALLINT, MEDIUMINT, INT (INTEGER), BIGINT
- 部分时间类型：DATE 和 DATETIME
- 部分字符串类型：CHAR, VARCHAR, BINARY,  VARBINARY

#### RANGE COLUMNS 分区

这种分区类型可以使用整数类型以外的类型列来定义范围。

RANGE COLUMNS 与 RANGE 分区的不同：

- RANGE COLUMNS 不接受表达式，只接受字段名。
- RANGE COLUMNS 接受一个或多个字段组合的列表。
  - RANGE COLUMNS 划分基于元组之间的比较
- RANGE COLUMNS 不受限于整数类型。其它类型也可以用作分区字段。

```mysql
CREATE TABLE table_name
PARTITIONED BY RANGE COLUMNS(column_list) (
    PARTITION partition_name VALUES LESS THAN (value_list)[,
    PARTITION partition_name VALUES LESS THAN (value_list)][,
    ...]
);
column_list:
    column_name[, column_name][, ...]
value_list:
    value[, value][, ...]
```

创建举例：

```mysql
mysql> CREATE TABLE rcx (
    ->     a INT,
    ->     b INT,
    ->     c CHAR(3),
    ->     d INT
    -> )
    -> PARTITION BY RANGE COLUMNS(a,d,c) (
    ->     PARTITION p0 VALUES LESS THAN (5,10,'ggg'),
    ->     PARTITION p1 VALUES LESS THAN (10,20,'mmm'),
    ->     PARTITION p2 VALUES LESS THAN (15,30,'sss'),
    ->     PARTITION p3 VALUES LESS THAN (MAXVALUE,MAXVALUE,MAXVALUE)
    -> );
Query OK, 0 rows affected (0.15 sec)
```

分区的定义必须遵从递增的顺序，否则会报错：

```mysql
-- 正确
CREATE TABLE rc4 (
    a INT,
    b INT,
    c INT
)
PARTITION BY RANGE COLUMNS(a,b,c) (
    PARTITION p0 VALUES LESS THAN (0,25,50),
    PARTITION p1 VALUES LESS THAN (10,20,100),
    PARTITION p2 VALUES LESS THAN (10,30,50)
    PARTITION p3 VALUES LESS THAN (MAXVALUE,MAXVALUE,MAXVALUE)
 );
-- 错误
mysql> CREATE TABLE rcf (
    ->     a INT,
    ->     b INT,
    ->     c INT
    -> )
    -> PARTITION BY RANGE COLUMNS(a,b,c) (
    ->     PARTITION p0 VALUES LESS THAN (0,25,50),
    ->     PARTITION p1 VALUES LESS THAN (20,20,100),
    ->     PARTITION p2 VALUES LESS THAN (10,30,50),
    ->     PARTITION p3 VALUES LESS THAN (MAXVALUE,MAXVALUE,MAXVALUE)
    ->  );
ERROR 1493 (HY000): VALUES LESS THAN value must be strictly increasing for each partition
```

##### RANGE COLUMNS 划分基于元组之间的比较的解释

举例说明：

```mysql
-- 创建分区表
MariaDB [MYISAM_TEST]> CREATE TABLE rc1 (
    ->     a INT,
    ->     b INT
    -> )
    -> PARTITION BY RANGE COLUMNS(a, b) (
    ->     PARTITION p0 VALUES LESS THAN (5, 12),
    ->     PARTITION p3 VALUES LESS THAN (MAXVALUE, MAXVALUE)
    -> );
    
-- 插入数据
MariaDB [MYISAM_TEST]> INSERT INTO rc1 VALUES (5, 2), (5, 10), (5, 12);
-- 查看数据分布
MariaDB [MYISAM_TEST]> SELECT PARTITION_NAME,TABLE_ROWS FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_NAME = 'rc1';
+----------------+------------+
| PARTITION_NAME | TABLE_ROWS |
+----------------+------------+
| p0             |          2 |
| p3             |          1 |
+----------------+------------+
```

多行分区对于分区键的比较是对元组进行比较，类似于这样：

```mysql
mysql> SELECT (5,10) < (5,12), (5,11) < (5,12), (5,12) < (5,12);
+-----------------+-----------------+-----------------+
| (5,10) < (5,12) | (5,11) < (5,12) | (5,12) < (5,12) |
+-----------------+-----------------+-----------------+
|               1 |               1 |               0 |
+-----------------+-----------------+-----------------+
1 row in set (0.00 sec)
```

#### LIST COLUMNS 分区

这种分区类型可以使用多个字段作为分区键，并使用整数类型以外的类型列来定义范围。

与RANGE COLUMNS 类似，参考上文说明。

多个字段作为分区键的写法：

```mysql
CREATE TABLE lc (
    a INT NULL,
    b INT NULL
)
PARTITION BY LIST COLUMNS(a,b) (
    PARTITION p0 VALUES IN( (0,0), (NULL,NULL) ),
    PARTITION p1 VALUES IN( (0,1), (0,2), (0,3), (1,1), (1,2) ),
    PARTITION p2 VALUES IN( (1,0), (2,0), (2,1), (3,0), (3,1) ),
    PARTITION p3 VALUES IN( (1,3), (2,2), (2,3), (3,2), (3,3) )
);
```

### HASH 分区

通过`HASH`进行分区主要用于确保在预定分区数量之间均匀地分布数据。

```mysql
CREATE TABLE employees (
    ...
)
PARTITION BY HASH(expr)
PARTITIONS num;
```

分区键*expr*是整型的字段或者是返回整型的表达式，同时需要指定分区数量*num*。

```mysql
CREATE TABLE employees (
    id INT NOT NULL,
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
)
PARTITION BY HASH( YEAR(hired) )
PARTITIONS 4;
```

*expr* 必须返回非常量非随机数的整型值。非常复杂的表达式可能会导致性能问题。也就是说，表达式越接近它所基于的列的值，MySQL就越有效地使用表达式进行散列分区。通过**取模运算**计算存储的分区号：*N* = MOD(*expr*, *num*)。

常规HASH分区让每个分区管理的数据都减少了，提高了查询效率；但是在分区管理上的代价太大，故提供了线性分区*LINEAR HASH Partitioning*来解决这个问题。

#### LINEAR HASH 分区

线性散列利用线性二次幂算法，而常规散列使用散列函数值的模数。

```mysql
CREATE TABLE employees (
    ...
)
PARTITION BY LINEAR HASH(expr)
PARTITIONS num;
```

分区号*N*通过以下算法计算：

1. 找到下一个大于等于*num*的2的幂*V*

```mysql
V = POWER(2, CEILING(LOG(2, num)))
```

2. 设置 *N = expr & (V - 1)*
3. While *N* >= *num*:
    - Set *V* = *V* / 2
    - Set *N* = *N* & (*V* - 1)

举例：

```mysql
CREATE TABLE t1 (col1 INT, col2 CHAR(5), col3 DATE)
    PARTITION BY LINEAR HASH( YEAR(col3) )
    PARTITIONS 6;
    
V = POWER(2, CEILING( LOG(2,6) )) = 8
N = YEAR('1998-10-19') & (8 - 1)
  = 1998 & 7
  = 6
(6 >= 6 is TRUE: additional step required)
N = 6 & ((8 / 2) - 1)
  = 6 & 3
  = 2
(2 >= 6 is FALSE: record stored in partition #2)
```

当线性分区的分区数是**2的N次幂**时，它与常规分区的结果是一致的。

优缺点：

- 优点：在分区管理（增加、删除、合并、拆分）时，处理更加迅速
- 缺点：相对于常规HASH分区，线性分区数据分布不太均匀。

### KEY 分区

类似于HASH分区，但是使用MySQL服务器提供的哈希函数。NDB Cluster 使用MD5()；其它的使用基于密码加密算法的哈希函数。

```mysql
CREATE TABLE employees (
    ...
)
PARTITION BY KEY(expr)
PARTITIONS num;
```

分区键*expr*是0或多个字段名，支持除BLOB或TEXT之外的其它类型作为分区键（查看上文**分区类型***）。用作分区键的任何列必须包含表的主键的部分或全部，如果表有一个。如果没有将列名指定为分区键，则使用表的主键（如果有）。在没有主键时，选择**唯一非空键**作为分区键。

注意：不能执行`ALTER TABLE DROP PRIMARY KEY`删除key类型的分区，但是NDB Cluster可以。

#### LINEAR KEY 分区

与线性HASH分区类似。

### 子分区

子分区（`Subpartitioning`）也称为复合分区，是对分区表中分区的进一步划分。

创建语法见上文。

支持对RANGE分区和LIST分区划分的表再划分，子分区又支持HASH 分区类型和KEY 分区类型。

例：

```mysql
-- 第一种写法
CREATE TABLE ts (id INT, purchased DATE)
    PARTITION BY RANGE( YEAR(purchased) )
    SUBPARTITION BY HASH( TO_DAYS(purchased) )
    SUBPARTITIONS 2 (
        PARTITION p0 VALUES LESS THAN (1990),
        PARTITION p1 VALUES LESS THAN (2000),
        PARTITION p2 VALUES LESS THAN MAXVALUE
    );
-- 第二种写法
CREATE TABLE ts (id INT, purchased DATE)
    PARTITION BY RANGE( YEAR(purchased) )
    SUBPARTITION BY HASH( TO_DAYS(purchased) ) (
        PARTITION p0 VALUES LESS THAN (1990) (
            SUBPARTITION s0,
            SUBPARTITION s1
        ),
        PARTITION p1 VALUES LESS THAN (2000) (
            SUBPARTITION s2,
            SUBPARTITION s3
        ),
        PARTITION p2 VALUES LESS THAN MAXVALUE (
            SUBPARTITION s4,
            SUBPARTITION s5
        )
    );
```

*ts*表有三个分区：*p0, p1, p2*，每个分区再进一步划分为两个分区，所有共有*2\*3 = 6*个分区。

**注意事项：**

- 每个分区必须有相同数量的子分区
- 不允许只对一部分分区定义子分区
- 在整个表中每个子分区的名称必须是唯一的
- 当NO_DIR_IN_CREATE服务器SQL模式生效时（不指定该模式也不可用），分区定义中不允许使用DATA DIRECTORY和INDEX DIRECTORY选项。从MySQL 5.5.5开始，在定义子分区时也不允许使用这些选项。实际操作可知：即使你添加了这两个选项，再创建表的时候也会忽略它们。

```mysql
-- 选项无效！
MariaDB [MYISAM_TEST]> CREATE TABLE sub_par_test (id INT) ENGINE=MYISAM PARTITION BY RANGE (id) SUBPARTITION BY HASH (id)
    -> (
    -> PARTITION P0 VALUES LESS THAN (10) (
    -> SUBPARTITION S01 DATA DIRECTORY = '/tmp/sub/data/s01' INDEX DIRECTORY = '/tmp/sub/index/s01',
    -> SUBPARTITION S02 DATA DIRECTORY = '/tmp/sub/data/s02' INDEX DIRECTORY = '/tmp/sub/index/s02'
    -> ),
    -> PARTITION P1 VALUES LESS THAN (MAXVALUE) (
    -> SUBPARTITION S03 DATA DIRECTORY = '/tmp/sub/data/s03' INDEX DIRECTORY = '/tmp/sub/index/s03',
    -> SUBPARTITION S04 DATA DIRECTORY = '/tmp/sub/data/s04' INDEX DIRECTORY = '/tmp/sub/index/s04'
    -> )
    -> );
MariaDB [MYISAM_TEST]> SHOW WARNINGS;
+---------+------+----------------------------------+
| Level   | Code | Message                          |
+---------+------+----------------------------------+
| Warning | 1618 | <DATA DIRECTORY> option ignored  |
| Warning | 1618 | <INDEX DIRECTORY> option ignored |
| Warning | 1618 | <DATA DIRECTORY> option ignored  |
| Warning | 1618 | <INDEX DIRECTORY> option ignored |
+---------+------+----------------------------------+
```

### MySQL分区如何处理NULL

!!! note
 `NULL`不是一个数字。
 MySQL的分区实现将NULL视为**小于任何非NULL值**，就像ORDER BY一样。

#### RANGE partitioning

如果计算过后分区键的值为NULL，将该记录插入到**最低的分区**中。

#### LIST partitioning

有一个分区的值**包含NULL**时，由LIST分区的表才允许NULL值。

#### HASH and KEY partitioning

NULL值被视为**零**。

## 分区管理

动作：添加、删除、重组（合并、分离）。使用`ALTER TABLE`语句带有的*partition_options* 子句执行，与创建分区的语法相同。这些动作会涉及到数据的迁移。

!!! note
 所有的分区都必须有相同数量的子分区，并且子分区一旦创建就不可改变。同一条语句中只有一个动作可以被执行。

### RANGE and LIST 的管理

#### 删除

```mysql
ALTER TABLE ... DROP PARTITION ...
```

```mysql
mysql> ALTER TABLE tr DROP PARTITION p2;
Query OK, 0 rows affected (0.03 sec)
```

 分区中的数据同时也被删除。`NDBCLUSTER`存储引擎不支持该操作。

#### 添加

```mysql
ALTER TABLE ... ADD PARTITION partition_definitions
```

RANGE只能在分区尾部添加新的分区，并且其范围比存在的分区都大。

```mysql
ALTER TABLE members ADD PARTITION (PARTITION p3 VALUES LESS THAN (2010));
ALTER TABLE tt ADD PARTITION (PARTITION p2 VALUES IN (7, 14, 21), PARTITION p3 VALUES IN (1));
```

LIST分区不能包含**重复的值**。

#### 重组

```mysql
ALTER TABLE table_name REORGANIZE PARTITION partition_names INTO (partition_definitions)
```

数据会存储在新的分区上。

分离分区：

```MySQL
ALTER TABLE members
    REORGANIZE PARTITION p0 INTO (
        PARTITION n0 VALUES LESS THAN (1970),
        PARTITION n1 VALUES LESS THAN (1980)
);
```

合并分区：

```MySQL
ALTER TABLE members REORGANIZE PARTITION s0,s1 INTO (
 PARTITION p0 VALUES LESS THAN (1970)
);
```

分离与合并的关系不必是一对多或多对一，可以是**多对多**：

```MySQL
ALTER TABLE members REORGANIZE PARTITION p0,p1,p2,p3 INTO (
    PARTITION m0 VALUES LESS THAN (1980),
    PARTITION m1 VALUES LESS THAN (2000)
);
```

!!! warning
 RANGE分区不能跨范围重组。例如：不能重组‘p0,p2’，而跳过p1。
 不能使用重组语句改变分区类型，以及改变分区键*expr*；使用`ALTER TABLE ... PARTITION BY ...`达到这个目的。

### HASH and KEY 的管理

这两个分区没有分区名，不能使用和分区名相关的语句进行管理。

#### 删除

不可删除指定分区，使用如下语句合并分区：

```MySQL
ALTER TABLE table_name COALESCE PARTITION number
```

*number*是要合并到其余分区的分区数，即删除的分区数。

```MySQL
MariaDB [MYISAM_TEST]> ALTER TABLE hash_par COALESCE PARTITION 2;
Query OK, 4 rows affected (0.11 sec)
Records: 4  Duplicates: 0  Warnings: 0
```

#### 添加

```MySQL
ALTER TABLE table_name ADD PARTITION PARTITIONS number 
```

*number*是添加的分区数。

#### 重组

通过删除和添加实现。

子分区会自动的进行相应的操作：

```MySQL
MariaDB [MYISAM_TEST]> SHOW CREATE TABLE sub_par_test \G;
CREATE TABLE `sub_par_test` (
  `id` int(11) DEFAULT NULL
) ENGINE=MyISAM DEFAULT CHARSET=utf8
/*!50100 PARTITION BY RANGE (id)
SUBPARTITION BY HASH (id)
(PARTITION P0 VALUES LESS THAN (10)
 (SUBPARTITION S01 ENGINE = MyISAM,
  SUBPARTITION S02 ENGINE = MyISAM),
 PARTITION P1 VALUES LESS THAN MAXVALUE
 (SUBPARTITION S03 ENGINE = MyISAM,
  SUBPARTITION S04 ENGINE = MyISAM)) */
MariaDB [MYISAM_TEST]> ALTER TABLE sub_par_test REORGANIZE PARTITION P0 INTO (PARTITION P00 VALUES LESS THAN (5), PARTITION P01 VALUES LESS THAN (10));
Query OK, 2 rows affected (0.01 sec)
Records: 2  Duplicates: 0  Warnings: 0
MariaDB [MYISAM_TEST]> SHOW CREATE TABLE sub_par_test \G;
CREATE TABLE `sub_par_test` (
  `id` int(11) DEFAULT NULL
) ENGINE=MyISAM DEFAULT CHARSET=utf8
/*!50100 PARTITION BY RANGE (id)
SUBPARTITION BY HASH (id)
(PARTITION P00 VALUES LESS THAN (5)
 (SUBPARTITION P00sp0 ENGINE = MyISAM,
  SUBPARTITION P00sp1 ENGINE = MyISAM),
 PARTITION P01 VALUES LESS THAN (10)
 (SUBPARTITION P01sp0 ENGINE = MyISAM,
  SUBPARTITION P01sp1 ENGINE = MyISAM),
 PARTITION P1 VALUES LESS THAN MAXVALUE
 (SUBPARTITION S03 ENGINE = MyISAM,
  SUBPARTITION S04 ENGINE = MyISAM)) */
```

### 分区的维护

维护动作：[`CHECK TABLE`](https://dev.mysql.com/doc/refman/5.5/en/check-table.html), [`OPTIMIZE TABLE`](https://dev.mysql.com/doc/refman/5.5/en/optimize-table.html), [`ANALYZE TABLE`](https://dev.mysql.com/doc/refman/5.5/en/analyze-table.html), and [`REPAIR TABLE`](https://dev.mysql.com/doc/refman/5.5/en/repair-table.html)。

语法：

```MySQL
ALTER TABLE table_name action PARTITION {partition_names | ALL}
```

- **重构分区**：丢弃（drop）分区所有的记录，然后重新插入数据。对于碎片整理很有用处。

```MySQL
ALTER TABLE t1 REBUILD PARTITION p0, p1;
```

- **优化分区**：若删除了一个分区的大量数据，或者在可变类型的字段做出了很多改变，可使用此语句来回收未使用的空间和优化分区数据文件。

```MySQL
ALTER TABLE t1 OPTIMIZE PARTITION p0, p1;
```

运行`OPTIMIZE PARTITION`与运行`CHECK PARTITION`, `ANALYZE PARTITION`, and `REPAIR PARTITION` 等价。

一些存储引擎（如InnoDB），不支持对单独的分区优化。在这种情况下，此语句将重构整个表。从MySQL 5.5.30开始，使用此语句将导致一个bug，使用`ALTER TABLE ... REBUILD PARTITION` and `ALTER TABLE ... ANALYZE PARTITION` 来避免整个问题。

- **分析分区**：读取并存储分区的key分布。

```MySQL
ALTER TABLE t1 ANALYZE PARTITION p3;
```

- **修复分区**：修复损坏的分区。

```MySQL
ALTER TABLE t1 REPAIR PARTITION p0,p1;
```

- **检查分区**：检查分区是否存在错误。其使用方式与在非分区表使用CHECK TABLE的方式非常相似。

```MySQL
ALTER TABLE trb3 CHECK PARTITION p1;
```

此语句将告诉表中的数据或索引是否已损坏。可以使用`REPAIR PARTITION`修复。

- **重建分区**：删除指定分区以及所有指定分区的数据并创建新分区。其使用方式与在非分区表使用TRUNCATE TABLE的方式非常相似。从MySQL 5.5.0开始。

```
ALTER TABLE hash_par TRUNCATE PARTITION ALL;
```

[**mysqlcheck**](https://dev.mysql.com/doc/refman/5.5/en/mysqlcheck.html) and [**myisamchk**](https://dev.mysql.com/doc/refman/5.5/en/myisamchk.html)不支持分区表。

`ANALYZE`, `CHECK`, `OPTIMIZE`, `REBUILD`, `REPAIR`, and `TRUNCATE`操作不支持子分区。
