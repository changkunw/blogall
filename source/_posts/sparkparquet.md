---
title: spark读写parquet
date: 2016-05-07 00:40:07
tags: [spark,parquet]
---

&nbsp;&nbsp;&nbsp;&nbsp;spark基于parquet的读写都提供了Api，所以是很简单的，这里只是需要说明一点的是，sc.parquetfile（path：String*）方法里面传入的是parquet文件的路径，意思是可以传很多路径，这是对的，变参嘛，在java里面可以直接传一个数组比如new String[]{path1，path2.。。。}，而且你在写程序的时候你也应该是写成数据然后传给他，只传一个文件进去的话肯定就没有什么问题，如果你用scala的话，按理说应该传一个Array(path,path1...),但是这样的话编译都过不了，scala在传变参的时候如果是一个数据的话应该这样写成
```
Array(path1,path2):_*
```
<!--more-->

mark 下代码

```
def dealFiles(): Unit = {
    val conf: SparkConf = new SparkConf().setAppName(classOf[CombineParquet].getName)
      .set("spark.executor.extraJavaOptions", "-XX:-OmitStackTraceInFastThrow")
      .set("spark.shuffle.consolidateFiles", "true")
    val fileSystem: FileSystem = FileSystem.get(new URI("hdfs://ip"), new Configuration())
    val sc: SparkContext = new SparkContext(conf)
    val sqlContext: SQLContext = new SQLContext(sc)
    val status: Array[FileStatus] = fileSystem.listStatus(new Path("/user/...."))
    val fileDir: mutable.Set[FileStatus] = scala.collection.mutable.Set[FileStatus]()
    for (file <- status) {
      if (file.isDirectory && file.getPath.getName.contains("_")) {
        val successFile: RemoteIterator[LocatedFileStatus] = fileSystem.listFiles(file.getPath, false)
        while (successFile.hasNext) {
          val scFile: LocatedFileStatus = successFile.next()
          if (scFile.getPath.getName == "_SUCCESS") {
            fileDir.add(file)
            logInfo(" found finish parquet file :" + file.getPath.getName)
          }
        }
      }
    }
    val parFiles: mutable.Set[String] = scala.collection.mutable.Set[String]()
    for (dir <- fileDir) {
      val files: RemoteIterator[LocatedFileStatus] = fileSystem.listFiles(dir.getPath, false)
      while (files.hasNext) {
        val parFile: LocatedFileStatus = files.next()
        if (parFile.getPath.getName.endsWith("parquet")) {
          parFiles.add(parFile.getPath.toString)
          logInfo("current parquet file is :" + parFile.getPath.getName)
        }
      }
    }
    val hdfsParFiles: mutable.Set[String] = parFiles.map(path => path.substring(path.indexOf("/user")))
    sqlContext.parquetFile(hdfsParFiles.toArray: _*)
      .repartition(1)
      .saveAsParquetFile("/user/..../allparquet_" + System.currentTimeMillis())
    fileDir.foreach(dir => {
      fileSystem.delete(dir.getPath, true)
    })
    sc.stop()
  }
```

上面这段代码的主要意思其实是想合并很多的parquet小文件
