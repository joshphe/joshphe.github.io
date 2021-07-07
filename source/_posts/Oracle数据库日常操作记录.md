---
title: Oracle数据库日常操作
image: https://gitee.com/Qzjp/pics/raw/master/titlepic/sea.png  #设置本地图片
keywords: Oracle, 日常

---

Oracle数据库日常操作

<!--more-->

## Oracle数据库日常操作

##### 一、查询数据库表的字段、类型、长度、是否为空

```sql
select
       a.TABLE_NAME as TableName,
       case when (select count(*) from user_views v where v.VIEW_NAME =a.TABLE_NAME )>0 then 'V' else 'U'end as "TableType",
       a.COLUMN_NAME as ColumnName,
       A.COLUMN_ID as ColumnIndex,
       a.DATA_TYPE as DataType,
       case  
         when a.DATA_TYPE = 'NUMBER' then
           case when a.Data_Precision is null then
             a.Data_Length
             else 
               a.Data_Precision
               end 
         else
          a.Data_Length
       end as Length,
       case when a.nullable = 'N' then  '0' else '1' end as IsNullable,
       b.comments as "Description", 
       case
          when (select count(*) from user_cons_columns c 
            where c.table_name=a.TABLE_NAME and c.column_name=a.COLUMN_NAME and c.constraint_name=
                (select d.constraint_name from user_constraints d where d.table_name=c.table_name and d.constraint_type   ='P')
                 )>0 then '1' else '0'end as IsPK
  from USER_TAB_COLS a,
       sys.user_col_comments b
 where a.table_name = b.table_name      
       and b.COLUMN_NAME = a.COLUMN_NAME       
       order by a.TABLE_NAME, a.COLUMN_ID;
```

##### 二、查询表的约束

```sql
select
	a.TABLE_NAME,
	a.CONSTRAINT_NAME,
	a.CONSTRAINT_TYPE,
	wm_concat (b.COLUMN_NAME) as CONSTRAINT_COLUMNS,
	a.R_CONSTRAINT_NAME
from
	USER_CONSTRAINTS a
	left join USER_CONS_COLUMNS b on a.CONSTRAINT_NAME = b.CONSTRAINT_NAME
group by
	a.TABLE_NAME,
	a.CONSTRAINT_NAME,
	a.CONSTRAINT_TYPE,
	a.R_CONSTRAINT_NAME
order by
	a.TABLE_NAME,
	a.CONSTRAINT_NAME;
```

##### 三、查询表的索引

```sql
select
	a.table_name,
	a.index_name,
	a.index_type,
	b.column_name,
	a.uniqueness as isuniqueness,
	a.partitioned as ispartition,
	a.tablespace_name
from
	user_indexes a,
	user_ind_columns b
where
	a.index_name = b.index_name
order by
	table_name,
	index_name;
```

##### 四、查询表分区情况

```sql
select
	table_name,
	partitioned as ispartitioned,
	'非分区' as partitioning_type
from
	user_tables
where
	table_name not in(
		select
			table_name from user_part_tables)
union
select
	t.table_name,
	t.partitioned as ispartitioned,
	p.partitioning_type
from
	user_tables t,
	user_part_tables p
where
	t.table_name = p.table_name;
```

##### 五、查询全表扫描的语句

```sql
--查找SQL_ID
SELECT *
FROM v$sql_plan v
WHERE v.OPERATION = 'TABLE ACCESS'
	AND v.OPTIONS = 'FULL'
	AND v.OBJECT_OWNER = 'XXX';--查询指定用户

--根据SQL_ID查找SQL_TEXT
SELECT p.SQL_TEXT
	,p.SQL_ID
	,q.TIMESTAMP
FROM v$sqlarea p
	,v$sql_plan q
WHERE p.SQL_ID = q.SQL_ID
	AND q.OPERATION = 'TABLE ACCESS'
	AND q.OPTIONS = 'FULL'
	AND q.OBJECT_OWNER = 'XXX'--查询指定用户
	AND q.TIMESTAMP > sysdate - 1;
```

##### 五、查询全表扫描的语句

```sql
--表统计信息收集
begin
	dbms_stats.gather_table_stats('USER','TABLE_NAME',cascade => true);
end;
/
```

​    