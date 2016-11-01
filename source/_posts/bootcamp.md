---
title: bootcamp
date: 2016-07-18 18:03:48
tags: MySQL
---


# mysql index and best practice handout

## syntax
```sql
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] tbl_name
    (create_definition,...)
    [table_options]
    [partition_options]

CREATE [TEMPORARY] TABLE [IF NOT EXISTS] tbl_name
    [(create_definition,...)]
    [table_options]
    [partition_options]
    [IGNORE | REPLACE]
    [AS] query_expression

CREATE [TEMPORARY] TABLE [IF NOT EXISTS] tbl_name
    { LIKE old_tbl_name | (LIKE old_tbl_name) }

create_definition:
    col_name column_definition
  | [CONSTRAINT [symbol]] PRIMARY KEY [index_type] (index_col_name,...)
      [index_option] ...
  | {INDEX|KEY} [index_name] [index_type] (index_col_name,...)
      [index_option] ...
  | [CONSTRAINT [symbol]] UNIQUE [INDEX|KEY]
      [index_name] [index_type] (index_col_name,...)
      [index_option] ...
  | {FULLTEXT|SPATIAL} [INDEX|KEY] [index_name] (index_col_name,...)
      [index_option] ...
  | [CONSTRAINT [symbol]] FOREIGN KEY
      [index_name] (index_col_name,...) reference_definition
  | CHECK (expr)

column_definition:
    data_type [NOT NULL | NULL] [DEFAULT default_value]
      [AUTO_INCREMENT] [UNIQUE [KEY] | [PRIMARY] KEY]
      [COMMENT 'string']
      [COLUMN_FORMAT {FIXED|DYNAMIC|DEFAULT}]
      [STORAGE {DISK|MEMORY|DEFAULT}]
      [reference_definition]
  | data_type [GENERATED ALWAYS] AS (expression)
      [VIRTUAL | STORED] [UNIQUE [KEY]] [COMMENT comment]
      [NOT NULL | NULL] [[PRIMARY] KEY]

data_type:
    BIT[(length)]
  | TINYINT[(length)] [UNSIGNED] [ZEROFILL]
  | SMALLINT[(length)] [UNSIGNED] [ZEROFILL]
  | MEDIUMINT[(length)] [UNSIGNED] [ZEROFILL]
  | INT[(length)] [UNSIGNED] [ZEROFILL]
  | INTEGER[(length)] [UNSIGNED] [ZEROFILL]
  | BIGINT[(length)] [UNSIGNED] [ZEROFILL]
  | REAL[(length,decimals)] [UNSIGNED] [ZEROFILL]
  | DOUBLE[(length,decimals)] [UNSIGNED] [ZEROFILL]
  | FLOAT[(length,decimals)] [UNSIGNED] [ZEROFILL]
  | DECIMAL[(length[,decimals])] [UNSIGNED] [ZEROFILL]
  | NUMERIC[(length[,decimals])] [UNSIGNED] [ZEROFILL]
  | DATE
  | TIME[(fsp)]
  | TIMESTAMP[(fsp)]
  | DATETIME[(fsp)]
  | YEAR
  | CHAR[(length)] [BINARY]
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | VARCHAR(length) [BINARY]
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | BINARY[(length)]
  | VARBINARY(length)
  | TINYBLOB
  | BLOB
  | MEDIUMBLOB
  | LONGBLOB
  | TINYTEXT [BINARY]
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | TEXT [BINARY]
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | MEDIUMTEXT [BINARY]
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | LONGTEXT [BINARY]
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | ENUM(value1,value2,value3,...)
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | SET(value1,value2,value3,...)
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | JSON
  | spatial_type

index_col_name:
    col_name [(length)] [ASC | DESC]

index_type:
    USING {BTREE | HASH}

index_option:
    KEY_BLOCK_SIZE [=] value
  | index_type
  | WITH PARSER parser_name
  | COMMENT 'string'

reference_definition:
    REFERENCES tbl_name (index_col_name,...)
      [MATCH FULL | MATCH PARTIAL | MATCH SIMPLE]
      [ON DELETE reference_option]
      [ON UPDATE reference_option]

reference_option:
    RESTRICT | CASCADE | SET NULL | NO ACTION

table_options:
    table_option [[,] table_option] ...

table_option:
    ENGINE [=] engine_name
  | AUTO_INCREMENT [=] value
  | AVG_ROW_LENGTH [=] value
  | [DEFAULT] CHARACTER SET [=] charset_name
  | CHECKSUM [=] {0 | 1}
  | [DEFAULT] COLLATE [=] collation_name
  | COMMENT [=] 'string'
  | COMPRESSION [=] {'ZLIB'|'LZ4'|'NONE'}
  | CONNECTION [=] 'connect_string'
  | DATA DIRECTORY [=] 'absolute path to directory'
  | DELAY_KEY_WRITE [=] {0 | 1}
  | ENCRYPTION [=] {'Y' | 'N'}
  | INDEX DIRECTORY [=] 'absolute path to directory'
  | INSERT_METHOD [=] { NO | FIRST | LAST }
  | KEY_BLOCK_SIZE [=] value
  | MAX_ROWS [=] value
  | MIN_ROWS [=] value
  | PACK_KEYS [=] {0 | 1 | DEFAULT}
  | PASSWORD [=] 'string'
  | ROW_FORMAT [=] {DEFAULT|DYNAMIC|FIXED|COMPRESSED|REDUNDANT|COMPACT}
  | STATS_AUTO_RECALC [=] {DEFAULT|0|1}
  | STATS_PERSISTENT [=] {DEFAULT|0|1}
  | STATS_SAMPLE_PAGES [=] value
  | TABLESPACE tablespace_name
  | UNION [=] (tbl_name[,tbl_name]...)

partition_options:
    PARTITION BY
        { [LINEAR] HASH(expr)
        | [LINEAR] KEY [ALGORITHM={1|2}] (column_list)
        | RANGE{(expr) | COLUMNS(column_list)}
        | LIST{(expr) | COLUMNS(column_list)} }
    [PARTITIONS num]
    [SUBPARTITION BY
        { [LINEAR] HASH(expr)
        | [LINEAR] KEY [ALGORITHM={1|2}] (column_list) }
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

query_expression:
    SELECT ...   (Some valid select or union statement)
```

## 选择合适的数据类型
物尽其用。如果存储空间较小的数据类型能满足需求，就不要使用更大的类型。

### 整数
TINYINT(8),SMALLINT(16),MEDIUMINT(24),INT(32),BIGINT(64) <UNSIGNED>. 如果明确存储的值不为负数，加上unsigned可有效扩大值域范围。

### 实数
虽然mysql提供了FLOAT,DOUBLE,DECIMAL来存储带精度的小数，但是实际开发中通常的做法是将其转换成整数存储，避免出现各种因精度导致的问题。

### 字符串
#### varchar
可变长，适用于大多数情况。
#### char
选择合适的使用场景，如存储密码和手机号等等。

### 日期和时间

#### datetime

保存时间范围从1001-9999，精度为秒，是mysql中表示时间范围最大的类型，且与时区无关。空间占用8字节。

#### timestamp
unix timestamp，表示从1970-01-01午夜开始的时间。可覆盖的最大时间为2038年，与时区相关，可根据时区自动变化，空间占用4字节。

这两种日期时间类型都只支持秒级别的数据，如果需要存储毫秒可以考虑使用bigint来存储。

### 谨慎选择标识符类型
1. 整数类型。 整数是最理想的作为标识列的数据类型，比较很快，并且可以使用auto_increment. 顺序I/O的性能要比随机好很多。
2. 字符串类型。 实际工作中可能会遇到使用UUID,SHA1产生的随机字符串来作为标识列，用字符串来作为标识列不仅消耗大而且比较慢，此外会导致插入，查询都变成随机I/O，而且可能会产生大量页分裂的情况，对性能有很大的影响。可考虑将其转换成整数存储。

## 表设计注意事项

### 合理的列长度
同一个表中不要设计过多的列，否则会导致数据行在mysql server和数据库引擎之前转换时产生较大的开销。
### 合理使用null
前面说过尽量不要存在默认值为null的情况，但是如果实际情况确实为一个未知的值，也不能硬塞一个不合理的值充数，否则编写代码时还要对其进行特殊处理，更严重的是导致bug的产生。如数据库存在一个`-1`的默认值,在传递给iOS客户端时，如果客户端使用的类型是unsigned int，就会产生一个未知的整数，严重的导致应用崩溃，次之则是产生困惑的数据。

## null 比较查询注意事项
不要使用 = <> 比较null值。使用 is null ,is not null. 灵活使用IF(),IFNULL(). 除非有特殊需要，尽量避免使用null值，null值会使得mysql很难对查询进行优化，尤其是不要在有索引的列上允许null存在。

## 字段维护注意事项

1. 新增字段尽量要有默认值,且设置为NOT NULL,有利于数据检索优化。
2. 增加多个字段时，使用一条语句修改，避免多次单独创建。
3. 字段新增或结构变化应该先于应用上线，避免上线后报SQL错误。
4. 若是删除字段，应该等待应用上线运行稳定后再进行字段删除，避免误删。
5. 若字段类型更改导致不兼容时，可能会停机维护，一般可通过其他方式避免停机。
6. 若涉及索引变更，应在应用上线前增加新的索引，上线运行稳定之后再删除旧的索引。

## 风格
### 命名
* 采用26个英文字母(区分大小写)和0-9这十个自然数,加上下划线'_'组成,共63个字符.不能出现其他字符(注释除外).
* 外键名用`fk_开头，后面跟该外键所在的表名和对应的主表名（不含t_）`。子表名和父表名自己用下划线（_）分隔。外键名长度不能超过30个字符。如果过长，可对表名进行缩写。缩写规则同表名的缩写规则。外键名用小写的英文单词来表示。
唯一性索引用`uk_开头，后面跟字段名。一般性索引用idx_开头`，后面跟字段名
* 备份数据表名使用正式表名加上备份时间组成，如，`branch_user_20140326`

### 建表相关

* innodb建表，主键用无意义的自增主键。
* 禁用外键约束，由应用程序实现参照完整性
* 字段需设置为非空，需设置字段默认值。
* 用INNODB引擎建表。
* 表和每个字段都添加简短的comments
* 用尽量少的存储空间来存数一个字段的数据
    1. 能用int的就不用char或者varchar
    1. 能用tinyint的就不用int
    1. 能用varchar(20)的就不用varchar(255)
* 在建表语句后面加上使用的数据引擎和索引还有编码格式
```sql
CREATE TABLE `db`.`tbl_name` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `user_id` int(11)  NOT NULL DEFAULT '0' COMMENT '用户ID',
  `start_time` DATETIME DEFAULT NULL COMMENT '创建时间',
  PRIMARY KEY (`id`),
  UNIQUE INDEX `uk_user` (`user_id`)
)ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='清理浏览记录的时间’;
```
* 修改或者新建表的时候,不需要加库命，也不要写if not exist这种语句，建表之前请确定是否存在原表,比如
```sql
alter table `user_token` add column `version` varchar(64) DEFAULT NULL
COMMENT '版本号';
```
而不是写
```sql
alter table `db`.`user_token` add column `version` varchar(64) DEFAULT NULL COMMENT '版本号';
```

### 修改表结构
* 多步合一,禁止单表多个更改字段，用多次alter table命令 ,在操作时，为了尽可能减少影响和操作时间，对同一个表进行的多步操作进行合并。比如对同一个表既加字段、又加索引，那么就应该写成一条语句。减少复制临时表的时间。
* 如果只是改变一个字段的默认值，那么使用`alter table user alter column id set default 5;`这种语法.

## 基本命令
`show databases`,`show tables`,`desc tableName`,`show create table tableName`.

## 查询技巧
```
where 1=1

select
g1.id
from group g1
where
1 = (select count(g2.id) from group g2 where g1.user_id=g2.user_id and g1.created_time <= g2.created_time );
```

## 索引

### B Tree 索引
实现为B+ TREE.
### hash索引
索引中存放的hash值和指向数据行的指针。
### 聚簇索引。
聚簇索引严格来说不是一种索引类型，只是一种数据存储方式。主键就是一种聚簇索引，索引和数据行都存放在leaf上。
### 覆盖索引
一个索引包含所有查询字段的值，则称为覆盖索引。
优点:
1. 可避免回表查询，极大提高效率。
2. 完全顺序I/O。
缺点:
因为要存储列值，所以只适用于B TREE 索引。
### 索引合并
如果表中存在多个单列索引，在同时使用这几个单列索引时，mysql会采用一种索引合并的算法来同时使用这几个索引，这个过程会耗费大量的CPU和内存资源，出现这种情况可以说设计的索引就是一个失败的案例。 通过EXPALIN可以查看到是否使用了索引合并。
### 索引选择性
索引选择性=唯一值/总记录数。 索引选择性越高查询性能越好。可以据此判断索引设计的好坏。

