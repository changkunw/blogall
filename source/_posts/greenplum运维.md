---
title: greenplum运维
date: 2016-12-12 13:29:57
tags: [GreenPlum,运维]
---

Greenplum挂了，再怎么启动都启动不来，始终报错如下：
```
file "/data1/gpdata/gpmaster/gpseg-1/postmaster.pid" does not exist
```
其实跟这个文件没有半毛钱的关系，查看日志也没有任何可以查的出来的，全是LOG没有失败的日志，最后发现pg_hba.conf这个文件中有几行不符合规范，注释掉不符合规范的那几行就可以启动的起来。这个报错也是醉了。
