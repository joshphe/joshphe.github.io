---
title: MySQL数据库设计开发规范
date: 2021-01-22 20:25:00
img: https://joshphe-pics.oss-cn-shanghai.aliyuncs.com/pics/mokuc.png  #设置本地图片
summary: Mysql数据库设计约束
tags:
  - MySQL
  - 开发规范
---

### Mysql 数据库设计约束：

#### 一、建表约束

【强制】 日期、时间相关的字段，根据精度要求不同，使用date或timestamp类型，不要使用char或varchar类型，如果字段只包含日期不包含具体时间，使用date型，把时分秒设置为00:00:00

**解读：**使用char或varchar类型可能造成优化器估算不准，影响数据处理效率。

【强制】 应用用户没有授予管理员权限

**解读：**严格的权限管理能有效避免不必要的操作失误。

【强制】建表时需要对表和字段增加注释

**解读：**规范的建表和注释便于后续维护和管理。

【强制】所有数据库对象名称禁止使用mysql保留关键字

**解读：**使用Mysql保留关键字会导致SQL报错。

【建议】高并发OLTP交易系统尽量避免使用大字段（BLOB、CLOB、LONG）

**解读：**因LOB字段结构特点，高并发系统中LOB字段时长出现导致系统运行缓慢的问题。被删除或更新的BLOB字段所占用空间不会自动批量回收，若表存在大量删除、更新操作时，BLOB所在Segments会迅速耗尽空间，新的INSERT需要空间时，会在高水位线上加锁后，回收曾使用但已经过期的BLOB空间，因该操作效率很低，数据库会有大量的‘enq:HW - contention’等待，相关SQL会由于该等待而串行执行，业务受影响十分严重。

【建议】建议控制单表数据量在1000万以内。单表数据文件控制在10G。单行数据建议不超过8K

**解读：**表记录数过多对数据库性能会有影响，若单行记录仅仅超出一点，还会产生较大的空间浪费。

【建议】不建议使用Mysql分区表

**解读：**Mysql分区表在物理上表现为多个文件，在逻辑上表现为一张表，谨慎选择分区键，跨分区查询效率可能会更低，建议采用物理分表的方式管理大数据。

【建议】不建议在数据库中存储图片，文件等大的二进制数据

**解读：**通常文件很大，会短时间内造成数据量快速增长，数据库进行数据库读取时，会进行大量的随机IO操作，文件很大时，IO操作很耗时。这类二进制数据建议存储于文件服务器，数据库只存储文件地址信息。

------

#### 二、SQL约束

【强制】where条件里的过滤字段上禁止使用函数

**解读：**在sql的where子句中带有索引的列使用函数时，优化器会忽略掉索引导致索引失效。建议将函数应用在条件上，索引是可以生效的。

```sql
select * from staff where trunc(birthdate) = '01-MAY-82';    //不会用到索引

select * from staff where birthdate < (to_date('01-MAY-82') + 0.9999);    //会使用索引
```

【强制】禁止使用LIKE '%_'进行模糊匹配查询

**解读：**对字段使用like模糊匹配时，如果不从字段的第一位开始，将无法使用索引，禁止使用。

```sql
select * from student where name like 'aaa%';    //'aaa%' 会用到索引

select * from student where name like '%aaa';    //'%aaa' 或者 '_aaa' 不会使用索引
```

【强制】绑定变量和字段数据类型需要保持一致

**解读：**字段定义和SQL绑定变量传进来的值的类型需要保持一致，否则会导致隐式转换无法使用索引。

```sql
例如：dept_id 是一个varchar2型的字段，但在查询时把该字段作为number类型以where条件传给Oracle,这样会导致索引失效。
select * from temp where dept_id = 33333333333 ; //错误的例子

select * from temp where dept_id ='33333333333'; //正确的例子
```

【强制】在查询中指定所需的列，不要直接使用 ‘*’ 返回所有的列

**解读：**

（1）在查询中使用select * 时，需要多做一步查询内部系统目录以便把表中的所有字段名找出来。

（2）明确字段名，将来在表增加字段的时候，不会导致程序报错。

（3）读取不需要的列会增加CPU、IO、NET消耗。

（4）无法有效的利用覆盖索引。

【强制】变量和参数在类型和长度必须与表定义的列类型和长度匹配

**解读：**减少存储空间和IO资源的浪费，避免隐式转换而导致不使用索引的情况。

【强制】SQL语句中的表关联查询不允许超过5个，需要join的字段，数据类型必须绝对一致；多表关联查询时，保证被关联的字段需要有索引

**解读：**优化器对与多表连接查询的处理效果不佳，应尽可能减少多表连接的情况，关联字段需要有索引避免不必要的全表扫描，提高执行效率。

【强制】对于很大的表进行全表删除时，不允许使用全表delete，应用truncate或drop重建

**解读：**delete语句是dml，该操作会放到rollback segment中，事务提交后才生效；delete语句不影响表所占用的extent，高水线（high watermark）保持原位置不动；而truncate会将高水线复位（回到最开始）。

另外删除速度一般来说：drop > truncate > delete

【强制】大批量UPDATE、DELETE、INSERT操作，要分批多次操作

**解读：**大批量操作可能会造成严重的主从延迟，binlog日志为row格式时会产生大量的日志，应避免产生大事务操作。

【强制】禁止使用order by rand() 进行随机排序

**解读：**将rand()放在order by子句中会被多次执行，效率极低，禁止使用。

【强制】受制于GTID复制的限制，不使用create table …… select语句建立表、不可以在事务中使用create temporary table建立临时表、不可使用关联更新，同时更新事务表与非事务表。

**解读：**受制于GTID复制的限制。

【建议】对应同一列进行or判断时，使用in代替or

**解读：**in或or在字段有添加索引的情况下，查询速度近似，但在没有添加索引的情况下，连接的字段越多，in比or的效率高很多。

【建议】尽量避免使用全表扫描

**解读：**对于小表或代码表的访问，全表扫描也许是合理的，其他情况下应尽量避免全表扫描，提高执行效率。

【建议】明显不会有重复值时使用UNION ALL而不是UNION

**解读：**union：对两个结果集进行并集操作，不包括重复行，同时进行默认规则的排序；union all：对两个结果集进行并集操作，包括重复行，不进行排序。所以在已知不会有重复值的情况下使用union all效率要高很多，因为union需要进行排重。

------

#### 三、索引约束

【强制】 Mysql每个InnoDB表必须要有主键，推荐使用自增列作为主键

**解读：**递增主键在写入时可以提高插入性能，避免Page分裂，减少表碎片，提升内存和空间的使用。InnoDB默认会给没有主键的表创建一个不可见的、长度为6字节的row_id，且InnoDB维护了一个全局的dictsys.row_id，所有未定义主键的表都共用这个row_id。无主键的表删除时，使用row模式的主从架构会导致备份表夯住。

【强制】 分区表上如无特别原因不使用全局索引

**解读：**分区表使用global index或local index因根据实际的业务需要来设计，应尽量将业务需要统计的信息放在同一个分区中，使用local index使分区表的性能最大化。若需要使用global index，需说明原因，对分区进行维护后全局索引会失效，应特别注意。

【强制】 OLTP系统每张表索引不超过5个

**解读：**控制索引数量可以减少不必要的索引空间维护和使用，对于经常插入、更新、删除数据的表应减少索引数量，过多的索引会导致索引开销大，影响性能。

【强制】 复合索引字段不超过3个，必须将区分度更高的字段放在左边

**解读：**复合索引的设计应综合考虑前缀性和可选性，索引字段不宜过多，应尽可能将区分度高，查询使用较频繁的字段放在左边，这样能够一开始就有效的过滤掉无用数据。在写SQL时，若WHERE条件中包含多个条件，应看表上是否有线程的联合索引可使用，注意各个条件的顺序应尽量和复合索引字段顺序一致。

【强制】 没有重复索引（如a和a,b列都建了索引）

**解读：**避免在表上建立重复的索引如复合索引(a,b)和单列索引a，降低索引的维护成本。

【强制】 外键的子表关联字段上必须有索引

**解读：**如果使用了外键约束，子表的关联字段上必须创建索引，否则主表的每一条记录删除，都会导致对子表一次全表扫描。

【强制】 避免在更新频繁、区分度不高的列上单独建立索引

**解读：**区分度不高的列单独创建索引的优化效果很小，但是较为频繁的更新则会让索引的维护成本更高。

------

#### 四、存储过程约束

【强制】禁止使用存储过程

**解读：**MySQL数据库在使用存储过程中会有一些限制，如优化器无法使用关键字来优化单个查询调用存储过程、优化器无法评估存储过程函数的执行成本等。而使用存储过程处理复杂业务逻辑时，在性能上没有太大提升，并且处理简单逻辑的语句时，是否使用存储过程性能差别并不大，同时对于存储过程的管理也相对复杂，所以不允许使用存储过程。

------

#### 五、触发器约束

【强制】禁止使用触发器

**解读：**考虑到降低数据库的复杂度，提高可维护性，减少不可预知的风险，禁止使用触发器。

------

#### 六、视图约束

------

#### 七、统计信息

Mysql有自动统计信息收集机制，当表的修改超过10%时触发。统计信息收集为异步模式，因此对短时间内有大量数据变化的表，需要手工收集统计信息。

------

#### 八、系统级别约束

【强制】在Mysql5.5版本后，Innodb成为默认存储引擎，所有表必须使用该存储引擎

**解读：**Innodb存储引擎支持ACID事务、支持行级锁、具备更好的恢复性以及更好的并发性。

【强制】MySQL数据库支持多种字符集，如无特殊要求，使用utf8mb4

**解读：**MySQL数据库支持多种字符集，如无特殊要求，使用utf8mb4，注意日常操作导出建表语句的操作中字符集的设置，不同的字符集可能引起索引失效等问题。

