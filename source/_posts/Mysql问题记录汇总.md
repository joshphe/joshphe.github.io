---
title: MySQL问题记录汇总
date: 2021-09-24 10:06:00
img: https://cdn.jsdelivr.net/gh/joshphe/blogImage@main/img/cloud.png  #设置本地图片
summary: Mysql问题记录汇总
tags:
  - MySQL
  - 日常
---

### Mysql 问题记录汇总：

#### 1、应用疲劳性压测

多并发情况下同时对数据库进行 **select、insert、update** 操作，6小时左右出现连接池连接耗尽，连接数据库失败。

**show full processlist** 查看数据库中存在大量 **insert、update** 操作积压。通过问题排查，定位为 **binlog** 写满导致磁盘空间不足。

**疑问1：应用使用的阿里Druid连接池并设置了超时时间60秒，在数据库操作执行超时的情况下，应用侧未捕获的超时异常。**

