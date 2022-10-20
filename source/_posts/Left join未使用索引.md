---
title: Left join未使用索引
date: 2021-05-13 19:25:00
img: https://gitee.com/Qzjp/pics/raw/master/2022/sea.png  #设置本地图片
summary: Mysql数据库字符集差异导致索引失效
tags:
  - left join
  - 索引
  - Mysql
  - 字符集
---

## Mysql字符集不同导致索引失效

先上SQL语句：

```sql
SELECT a.fuhrq AS fuhrq
	,a.busiType AS busiType
	,a.zhangh AS zhangh
	,b.DOCUMENT_NUMBER AS createNumber
	,b.HUM AS hum
	,c.ORGANNAME AS organname
	,a.yinjkbh AS yinjkbh
	,a.channelId AS channelId
	,a.localSerialNum AS localSerialNum
	,a.warehousingStatus AS warehousingStatus
	,a.qiyrq AS qiyrq
	,a.ruky AS ruky
FROM yinj_serial_status a
LEFT JOIN zhanghb b ON a.zhangh = b.zhangh
LEFT JOIN organarchives c ON a.JIGH = c.ORGANNUM
WHERE a.serialStatus = '6'
	AND a.fuhrq >= '2020-10-01'
	AND a.fuhrq <= '2020-12-16'
	AND a.jigh IN (
		SELECT o.ORGANNUM
		FROM organarchives o
		WHERE LEVELFLAG LIKE '6iv%'
		)
ORDER BY a.fuhrq DESC;
```

其中a表（8000+条记录）中索引信息如下：

| 名             | 字段           | 索引类型 | 索引方法 |
| :------------- | -------------- | -------- | -------- |
| localSerialNum | localSerialNum | Unique   | BTREE    |
| idx_zhangh     | idx_zhangh     | Normal   | BTREE    |
| idx_fuhrq      | idx_fuhrq      | Normal   | BTREE    |

b表（30W+条记录）索引如下

| 名                  | 字段            | 索引类型 | 索引方法 |
| :------------------ | --------------- | -------- | -------- |
| idx_document_number | DOCUMENT_NUMBER | Normal   | BTREE    |
| idx_zhangh          | ZHANGH          | Normal   | BTREE    |

c表（400+条记录）索引如下

| 名                        | 字段          | 索引类型 | 索引方法 |
| :------------------------ | ------------- | -------- | -------- |
| ORGANARCHIVES_N_PARENTNUM | N_PARENTNUM   | Normal   | BTREE    |
| idx_levelflag             | idx_levelflag | Normal   | BTREE    |

该SQL跑了190S依然没有将结果查询出来，查看了一下该SQL的执行计划没有使用a表的idx_zhangh索引查询，而是全表扫描：

![](https://gitee.com/Qzjp/pics/raw/master/2022/SqlPlan1.png)

看了一下SHOW WARNINGS

```sql
--WARNING! ERRORS ENCOUNTERED DURING SQL PARSING!
/* select#1 */
SELECT `localhost`.`a`.`fuhrq` AS `fuhrq`
	,`localhost`.`a`.`busiType` AS `busiType`
	,`localhost`.`a`.`zhangh` AS `zhangh`
	,`localhost`.`b`.`DOCUMENT_NUMBER` AS `createNumber`
	,`localhost`.`b`.`HUM` AS `hum`
	,`localhost`.`c`.`ORGANNAME` AS `organname`
	,`localhost`.`a`.`yinjkbh` AS `yinjkbh`
	,`localhost`.`a`.`channelId` AS `channelId`
	,`localhost`.`a`.`localSerialNum` AS `localSerialNum`
	,`localhost`.`a`.`warehousingStatus` AS `warehousingStatus`
	,`localhost`.`a`.`qiyrq` AS `qiyrq`
	,`localhost`.`a`.`ruky` AS `ruky`
FROM `localhost`.`yinj_serial_status` `a` semi
JOIN (`localhost`.`organarchives` `o`)
LEFT JOIN `localhost`.`zhanghb` `b` ON ((`localhost`.`a`.`zhangh` = convert(`localhost`.`b`.`ZHANGH` using utf8mb4)))
LEFT JOIN `localhost`.`organarchives` `c` ON ((`localhost`.`a`.`jigh` = convert(`localhost`.`c`.`ORGANNUM` using utf8mb4)))
WHERE (
		(`localhost`.`a`.`serialStatus` = '6')
		AND (`localhost`.`a`.`fuhrq` >= '2020-10-01')
		AND (`localhost`.`a`.`fuhrq` <= '2020-12-16')
		AND (`localhost`.`o`.`LEVELFLAG` LIKE '6iv%')
		AND (`localhost`.`a`.`jigh` = convert(`localhost`.`o`.`ORGANNUM` using utf8mb4))
		)
ORDER BY `localhost`.`a`.`fuhrq` DESC

```

可以看到在LEFT JOIN的ON关联条件中做了convert(`localhost`.`b`.`ZHANGH` using utf8mb4)、convert(`localhost`.`c`.`ORGANNUM` using utf8mb4)、convert(`localhost`.`c`.`ORGANNUM` using utf8mb4)分别对zhanghao、organnum做了字符集的转换，导致不适用表上的索引，而是全表扫描。

看了一下几张表的字符集，发现yinj_serial_status表字段的字符集为utf8mb4、utf8mb4_general_ci，另外两张表为utf8、utf8_general_ci，当yinj_serial_status表作为驱动表时，需要将被驱动表的字符集转为utf8mb4

```
SHOW FULL COLUMNS FROM yinj_serial_status;
SHOW FULL COLUMNS FROM zhanghb;
SHOW FULL COLUMNS FROM organarchives;
```

yinj_serial_status表

![](https://gitee.com/Qzjp/pics/raw/master/2022/yinj_serial_status.png)

zhanghb表

![](https://gitee.com/Qzjp/pics/raw/master/2022/zhanghb.png)

organarchives表

![](https://gitee.com/Qzjp/pics/raw/master/2022/organarchives.png)

修改了一下yinj_serial_status表的字符集

```sql
ALTER TABLE yinj_serial_status CONVERT TO CHARACTER SET utf8 COLLATE utf8_general_ci;
```

再看一下该SQL的执行计划走了相应的索引，而idx_fuhrq因为时间范围覆盖了全表，走全表扫描也是正常。

![](https://gitee.com/Qzjp/pics/raw/master/2022/SqlPlan2.png)

整条SQL执行完成只需0.5s，再看一下SHOW WARNINGS没有再进行字符集的转换

```sql
/* select#1 */
SELECT `localhost`.`a`.`fuhrq` AS `fuhrq`
	,`localhost`.`a`.`busiType` AS `busiType`
	,`localhost`.`a`.`zhangh` AS `zhangh`
	,`localhost`.`b`.`DOCUMENT_NUMBER` AS `createNumber`
	,`localhost`.`b`.`HUM` AS `hum`
	,`localhost`.`c`.`ORGANNAME` AS `organname`
	,`localhost`.`a`.`yinjkbh` AS `yinjkbh`
	,`localhost`.`a`.`channelId` AS `channelId`
	,`localhost`.`a`.`localSerialNum` AS `localSerialNum`
	,`localhost`.`a`.`warehousingStatus` AS `warehousingStatus`
	,`localhost`.`a`.`qiyrq` AS `qiyrq`
	,`localhost`.`a`.`ruky` AS `ruky`
FROM `localhost`.`organarchives` `o`
JOIN `localhost`.`yinj_serial_status` `a`
LEFT JOIN `localhost`.`zhanghb` `b` ON ((`localhost`.`b`.`ZHANGH` = `localhost`.`a`.`zhangh`))
LEFT JOIN `localhost`.`organarchives` `c` ON ((`localhost`.`a`.`jigh` = `localhost`.`c`.`ORGANNUM`))
WHERE (
		(`localhost`.`a`.`serialStatus` = '6')
		AND (`localhost`.`a`.`fuhrq` >= '2020-10-01')
		AND (`localhost`.`a`.`fuhrq` <= '2020-12-16')
		AND (`localhost`.`o`.`LEVELFLAG` LIKE '6iv%')
		AND (`localhost`.`a`.`jigh` = `localhost`.`o`.`ORGANNUM`)
		)
ORDER BY `localhost`.`a`.`fuhrq` DESC
```
