---
title: 数据库设计
date: 2024-05-27 11:22:23
tags: 设计
---

## 1.数据库设计

### 1.1 存储引擎
+ 如无特殊情况，`MySQL`必须使用`InnoDB`引擎，更换引擎需要与DBA讨论。



### 1.2. 可以考虑反范式设计
+ 适合的冗余设计，减少`JOIN`。
+ 可以不创建物理外键，由程序控制数据的一致性，但是外键的字段必须覆盖索引。



### 1.3. 字段设计
+ 频繁读写的`InnoDB`表，使用`UNSIGNED`自增列作为主键。
+ 自增主键创建：
+ 
    - `MySQL`使用`int / bigint unsigned AUTO_INCREMENT`。
    - `PgSQL`使用 `serial / bigserial`。
    - `MSSQL`使用 `int/bigint identity`。



其他注意事项：



+ 不同表之间用于关联的外键类型需保持一致，避免隐式转换带来的问题。
+ 尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。
+ MySQL使用`UNSIGNED`存储非负数值，标志和状态等枚举类型的字段根据具体情况使用`tinyint/smallint/int/bigint`，所有的数值类型不必定义长度，使用默认长度即可。
+ MySQL 5.6以后的版本，建议用`DATETIME`取代`TIMESTAMP`，因为它的可用范围比`TIMESTAMP`更大，物理存储上仅比`TIMESTAMP`多1个字节，整体性能上的损失并不大。（5.5及之前建议用`TIMESTAMP`）如果只是精确到日期，使用`DATE`即可。
+ 对于`text`，`blob`字段，读写频繁的表建议进行拆分，分隔冷热数据。
+ MySQL应使用`INT UNSIGNED`存储IPV4地址，用`INET_ATON()`、`INET_NTOA()`进行转换，基本上没必要使用`CHAR(15)`来存储。
+ PgSQL有专门保存ip地址的类型Inet。
+ MSSQL对于字符串类型的字段如果内容含有中文，选择`nvarchar` ,`nchar`比`varchar`,`char`更好，能够确定长度的情况下`char`，`nchar`比`varchar`，`nvarchar`的效率更高。
+ 表必备三字段：`id, CreateDate/create_time, EditDate/edit_time`。`id`为主键，`CreateDate/create_time`为创建时间，`EditDate/edit_time`为修改时间。



### 1.4. 字符集
+ MySQL字符集使用`utf8mb4`，校对规则选择`utf8mb4_general_ci`。
+ MySQL存放Emoji表情必须用`utf8mb4`字符集。
+ 禁止指定列的字符集以及校对规则，需要统一指定表级别的字符集以及校对规则。
+ 如需指定特殊的字符集以及相应的校对规则，需要与DBA讨论。
+ MSSQL禁止指定列以及表的校对规则统一按照数据库的系统校对规则。



### 1.5. 避免使用NULL字段
+ 很难进行查询优化，`NULL`列加索引，需要额外空间。
    - c1 varchar(16) default null -- 不合规。
    - c1 varchar(16) default not null -- 不合规。
    - c1 varchar(16) not null default '' -- 合规。
    - c1 int not null default 0 -- 合规。
    - c1 double(18,4) not null default 0 -- 合规。
    - c1 datetime not null default  '1970-01-01' -- 合规。





### 1.6. 索引
+ 所有表必须有显式主键，一般必须要带有唯一的命名为`id`的自增主键。
+ 减少对索引列更新。
+ 尽量使用短，自增的重复率低列做索引。
+ 复合索引的建立以及使用需要注意字段顺序（`order by`字段以及`group by`字段顺序）。
+ 物理外键可以不建立，但是外键的字段都应该覆盖索引。
+ 避免冗余和重复索引（如：一张表内既存在a列的索引，又存在a,b列的复合索引）。
+ `MySQL`需要注意`union all `引起执行计划索引失效的情况。



### 1.7. 做好数据评估
+ 对预估数据量大的表可以预留冷热拆分、水平拆分或者分区方案。
+ 尽量不要使用分区表，在表变大后，进行DDL、SHARDING、恢复都变得更加困难。因此应业务端进行SHARDING。



### 1.8. MySQL以及PostgreSQL命名规范
+ 对象名（表名、列名、函数名、视图名、序列名、等对象名称）规范，对象名务必只使用小写字母，下划线，数字。不要以数字开头，不要使用保留字段。具体系统保留字段请参考：
+ 
    - MySQL：[http://dev.MySQL.com/doc/refman/5.7/en/keywords.html](http://dev.MySQL.com/doc/refman/5.7/en/keywords.html)
    - PostgreSQL: [https://www.postgresql.org/docs/current/static/sql-keywords-appendix.html](https://www.postgresql.org/docs/current/static/sql-keywords-appendix.html)
+ 表名命名,对相关功能的表应当使用相同前缀,其中前缀通常为这个表的模块或依赖主实体对象的名字。（如：sms_status_rpt、sms_subject）
+ 视图应以"v*"开头，存储过程应以"sp*"开头，用户自定义函数应以"fn*"开头，触发器应以"tr*"开头。
+ 公共审计字段命名规范creator、editor、create_time、edit_time、is_del、version
+ 其他非公共字段，应当使用简单单词命名，单词之间必须使用下划线分隔。（例如：oss_id）
+ 多表中的相同列，必须保证列名一致，数据类型一致。
+ 字段应当有注释，描述该字段的用途及可能存储的内容，如枚举值则必须将该字段中使用的内容都定义出来
+ 索引命名，索引名字前缀为"idx*"，唯一索引前缀为"uniq*"（MySQL ：表中索引以及外键之类的约束命名应该明确区分，如果索引与约束命名一样有可能会引起MySQLdump生成的备份，恢复的时候报错。（截止到5.7.15依旧存在该问题） PG：索引名称与约束名称必须全实例唯一）。



### 1.9. MSSQL命名规范
+ 对象名（表名、列名、函数名、视图名、序列名、等对象名称）规范，对象名务必只使用字母，下划线，数字。单词首字母必须大写，不要以数字开头，不要使用保留字段，不要用下划线来连接每个单词，禁止使用中文命名！禁止名称中留有空格！（具体系统保留字段请参考： MSSQL : [https://msdn.microsoft.com/zh-cn/library/ms189822.aspx）](https://msdn.microsoft.com/zh-cn/library/ms189822.aspx）)
+ 公共审计字段命名规范Creator、Editor、CreateDate、EditDate、Deleted、Version。
+ 视图应以"v"开头，存储过程应以"sp"开头，用户自定义函数应以"fn"开头，触发器应以"tr"开头。
+ 索引命名，索引名字前缀为"IDX*"，唯一约束前缀为"UK*"，检查约束前缀为"CK_"。
+ 字段应当有注释，描述该字段的用途及可能存储的内容，如枚举值则必须将该字段中使用的内容都定义出来。
+ 编写函数文本如视图、P函数、触发器、存储过程以及其他数据对象时，必须为每个函数增加适当注释。该注释以多行注释为主，主要结构如下：



-- Author: --作者

-- Create Date: --创建时间

-- Description: --功能描述

-- Input : --输入参数 (有传入参数的话需要填写，以","隔开)

-- Output :  --输出参数 (有传出参数的话需要填写，以","隔开)

-- Update Log: --函数更改信息(包括作者、时间、更改内容等)

(如：2016.05.24 by LQB : 添加解冻操作科目)

---

### 1.10. 数据加密规范原则
+ 采用加密字符串存储密码，并保证密码不可解密。
+ 对于四要素信息（姓名，手机号，银行卡号，身份证号），应该加密后再入库，保证数据的安全性。（注：具体加解密方法与规范请联系架构组和代码评审委员会了解）。

### 1.11. 审计字段使用规范
| 字段名 | 描述 |
| --- | --- |
| Creator/creator | 数据创建来源，应在插入的时候带入。不进行修改。此字段不与业务数据相关，仅作为标识数据来源用途，如:'jd_system','sys' |
| CreateDate/create_time | 数据创建时间，应在插入的时候带入。不进行修改 |
| Editor/editor | 数据修改来源，首次更新的时候带入，后续更新的时候修改。此字段不与业务数据相关，仅作为标识数据更新来源用途，如:'jd_system','sys' |
| EditDate/edit_time | 数据修改时间，首次更新的时候带入，后续更新的时候修改。 |
| Deleted/deleted | 软删除标识。只有0,1两个值。1为删除。 |
| Version/version | 行版本号，每次更新的时候+1。更新的情况下带上该字段为条件能有效减少“不可重复读”引起的问题。例如：   Select @version=version from table1 where id = 123;   Update table1 set version = version + 1 where id = 123 and version = @version; |


## 2. SQL规范
### 2.1. 避免类型转型
+ 数字对数字，字符对字符 ，避免类型转换导致查询效率低下以及索引失效。



### 2.2. 不在索引列进行数学运算或函数运算
+ where cast(create_time as date)  =  '2016-01-01'-- 不合规。
+ where  create_time  >=  '2016-01-01 00:00:00' and create_time <=  '2016-01-02 23:59:59' -- 合规。



### 2.3. 注重where条件
+ `delete、update、select`必须有`where `条件。
+ `where`条件的字段，区别度高的字段，注意建索引。
+ `like`不要出前以%开头的查询，使用不了索引，导致全表扫描。
+ 对于出现子查询的SQL，要确认上线的MySQL版本，利用`explain`确认查询是否符合预期。
+ 避免负向查询，如`NOT、!=、<>、!<、!>、NOT EXISTS、NOT IN、NOT LIKE`等。
+ 对于连续的值，用`between`代替`in`。
+ 对于依赖外部表的字段，用`exists`代替`in`。
+ 同一字段，将`or`改写为`in()`，注意控制IN的个数，建议小于200。
+ 不同字段，可以将`or`改为`union`。



### 2.4. 明确需要的字段
+ 尽量避免`SELECT *`，方便调整字段列表，还可减少不必要的I/O读（保留意见，表字段少或者真的需要查询全部列的时候可以使用`SELECT *` ）。
+ INSERT要明确字段写入。



### 2.5. 大量数据操作
+ 当插入大量数据的时候使用CSV平面文件导入会是一个不错的选择（例如：MySQL ：load data，PG：copy from ,MSSQL：BCP）。
+ 全表删除数据时，`truncate`比`delete`的效率快得多（只适用于测试环境，生产环境一般用软删除，如果需要物理删除则要提前备份）。



### 2.6. 优化join
+ 小表驱动大表，多表`join`时，要把过滤性最大的表选为驱动表，使用字典， 常用表，其它表排序。
+ 控制`join`后面`where `条件选择的行数，尽量在1000行以下。
+ 使用`union all`代替`union`。
+ 减少临时表出现。
+ 减少`outer join`的滥用（`left join ,right join,full outer join`）。



（例子：`select a.name,b.type,b.create_date from a left outer join b on a.id =b.prjid where b.create_date >= '2016-01-01'`这里的`left join`并没有任何意义，与`inner join`是等价。）



### 2.7. 线上的操作
+ 分批多次操作。
+ 大事务拆成多个事务分区间操作。
+ 线上对大表进行加字段并且该字段需要赋默认值的时候应该分四步走:
1. 
    1. 加字段。
    2. set default。
    3. 更新新字段的值（非业务繁忙期，拆分小规模更新）。
    4. 添加非空约束。
+ 对同一个表的多次alter操作必须合并为一次操作。
+ 对于更新或者修复的DDL，DML脚本需要统一包裹在事务里面执行，必要的时候可以进行回滚。(PgSQL、MSSQL支持DDL事务，MySQL暂不支持DDL事务)。



### 2.8. 减少数据的交互次数
+ 数据插入时如果要避免重复数据以及减少数据的交互次数,可以利用唯一索引或下列语句处理：
+ 
    - MySQL使用insert ignore、replace into、insert...on duplicate key update。
    - PgSQL使用 insert...on conflict do update set (9.5以上版本支持)。
+ 对于需要查询的Insert,Update,Delete影响的数据可以利用下列的字句：
+ 
    - MySQL、PgSQL使用RETURNING。
    - MSSQL使用OUTPUT。



### 2.9. 不要在数据库中存放过多业务逻辑
+ 尽量少使用存储过程、触发器、视图、自定义函数等。不应该把过多的业务逻辑耦合到数据库，这样会造成数据库的DDL、SCALE OUT、SHARDING等变得更加困难。



### 2.10. 其他注意
+ 尽量使用标准的SQL语法。（如：MySQL: select col1 ,col2,sum(amount) from tb1 group by col2 这类不标准的SQL用法不要使用。DBA组这边也会在以后新的项目里面添加sql_mode的控制）。
+ MySQL中避免出现now()、rand()、sysdate()、current_user()等不确定结果的函数。
+ MySQL 5.5以及之前的版本避免使用子查询。
+ MySQL 5.6开启GTID特性后临时表不支持事务。
+ 不在程序使用PgSQL、MySQL的create...table...as select...以及MSSQL的Select....into。
+ 不要滥用悲观锁（排他锁），如果不了解锁的作用域，慎用如：MySQL、PgSQL的select...for update，MSSQL的 select ...from ...(XLOCK)/(TABLOCKX)/(UPDLOCK)。
+ PgSQL需要注意不要滥用""，添加双引号的对象会大小写敏感。上线脚本数据库对象统一不用双引号引用。
+ 生产环境一般使用逻辑删除，不使用物理删除。
+ 非sql语句结束部分，不要使用分号（特别是注释）。

## 3. 其他注意事项
### 3.1 MySQL禁止在查询条件使用函数
重点说多几次：

+ MySQL的查询SQL中的where 条件中禁止使用函数，因为一旦使用函数容易导致索引失效，导致慢SQL拖垮整个MySQL服务。
+ MySQL的查询SQL中的where 条件中禁止使用函数，因为一旦使用函数容易导致索引失效，导致慢SQL拖垮整个MySQL服务。
+ MySQL的查询SQL中的where 条件中禁止使用函数，因为一旦使用函数容易导致索引失效，导致慢SQL拖垮整个MySQL服务。

where 条件之外可以使用函数，例如

```sql
SELECT DATE_FORMAT(create_time, '%Y-%m-%d') FROM $tableName WHERE 1= 1 GROUP BY DATE_FORMAT(create_time, '%Y-%m-%d')
```

