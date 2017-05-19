---
title: scala操作hdfs之parquet
date: 2016-05-07 00:12:00
tags: [scala,parquet,hdfs]
---

&nbsp;&nbsp;&nbsp;&nbsp;现在我们集群上的数据存放格式群是parquet格式，其实用spark操作parquet也是很方便的，但是如果你在已经有了一个sparkjob，你需要在这个job中再起一个job来处理parquet文件的话，就不行了，如果可以的话估计只有上jobserver了。这里记录一下直接使用scala来操作hdfs上的parquet文件
```scala
def main(args: Array[String]): Unit = {
    System.setProperty("HADOOP_USER_NAME", "wangchangkun")
//多个parquet文件
    val paths = Array("hdfs://ip:8020/user/**.parquet","hdfs://ip:8020/user/**.parquet")
//建立一个parquetreader的集合
    val parquetReaders: mutable.Set[ParquetReader[Record]] = scala.collection.mutable.Set()
    val parquetWriter: AvroParquetWriter[Record] = new AvroParquetWriter[Record](new Path("hdfs://ip:8020/user/***.parquet"), PreParsedLog.SCHEMA$)
    val readSupport: AvroReadSupport[Record] = new AvroReadSupport[Record]
    paths.foreach(file => {
      val path: Path = new Path(file)
      val parquetReader: ParquetReader[Record] = ParquetReader.builder[Record](readSupport, path).build()
      parquetReaders.add(parquetReader)
    })
    parquetReaders.foreach(reader => {
      var flag = true
      while (flag) {
        val preParsedLog = reader.read()
        if (preParsedLog == null) {
          flag = false
        } else {
          parquetWriter.write(preParsedLog)
        }
      }
      reader.close()
    })
    parquetWriter.close()
  }
```

<!--more-->

&nbsp;&nbsp;&nbsp;&nbsp; 上面需要注意的是PreParsedLog.SCHEMA和Recoder，本来按理来说你传PreParserLog的schema那么你的AvroReadeSupport、ParquetReader和ParquetWriter的泛型都应该是PreParserLog而不是Record，而且你这么写了编译的时候也是不会报错的，但是你在运行的时候就不行了，这里会抛出一个类型强转的异常，意思就是父类转子类不能强转，特别是这段代码如果不在主线程中你根本看不到他抛出来的异常