---
title: spark2Gp
date: 2016-10-12 23:11:25
tags: [Spark,GreenPlum]
---

　　从Spark将数据写入到GreenPlum集群中，亲测有两种方式比较快，两种方式都是通过csv，而不是通过JDBC直接连上GP集群一条一条的写。

<!--More-->

## Gpfdist

这种方式是配合外表表的使用，首先在Gp中创建外部表，然后通过Spark程序将数据以CSV的方式写入到Hdfs上，然后可以在Gp中写一个从hdfs上拷贝到本地的程序，将数据实时的拷贝到本地并加载到Gp集群。加载到集群之后再将数据以``insert into select * from ....``这种方式加载到正式表中，最后将外部表数据删除。再将hdfs上的数据删除，这样保证了数据唯一性，外部表只是起了一个过渡的作用，加载到GP集群后数据可以均匀分配到集群，不会给节点造成压力，这种速度也是比较快的，一秒钟10万行是没有问题的。

```
nohup gpfdist –d /home/gpadmin –p 1234 >/tmp/gpfdist.log &
DROP external TABLE IF EXISTS "externals"."slimedecommerce";
CREATE external TABLE "externals"."slimedecommerce" (
order_id           text, 
isp                text 
)
location('gpfdist://localhost:1234/data/csv_*/*SlimedECommerce_*')
format 'CSV' (DELIMITER as E'\002' null as '')
LOG ERRORS INTO med_ord_pgm_d_err SEGMENT REJECT LIMIT 10000;
```
## Copy

这种方式比上述方式要简单点，过程少了几部，这里主要运用``PGConnection``的``copyin``
，本质上还是些csv，只是将csv写入到内存，然后直接copy到GP，前提是要获取JDBC连接。
``
conn.unwrap(classOf[PGConnection]).getCopyAPI.copyIn(s"COPY $tableName FROM STDIN DELIMITER as E'\\002' null as '' LOG ERRORS INTO med_ord_pgm_d_err SEGMENT REJECT LIMIT 10000", reader)
``

