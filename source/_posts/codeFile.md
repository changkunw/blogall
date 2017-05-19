---
title: scala 代码段
date: 2016-10-16 21:20:23
tags: [scala,代码段]
---

　　本人觉得像scala这样的语言，最适合的就是用来装逼，让别人看起来，代码写的很6的样子，下面是一个代码段，比如在用scala写spark程序的时候，有个rdd，然后你突然看到rdd调用了一个abc方法，如``rdd.abc()``这个方法肯定不是Rdd这个类里面的，看到这样的，让你云里雾里的时候你一定要想到隐士转换这个东西，话不多说，上代码先。


<!--More-->
```
package com.kun.example.sparkstreaming

import org.apache.spark.SparkContext
import org.apache.spark.rdd.RDD

object SparkDemo {
  var sc: SparkContext = new SparkContext()
  var rdd: RDD[String] = sc.textFile("/abc")

  import com.kun.example.util.UtilRdd._

  rdd.abc()
}

package com.kun.example.util

import org.apache.spark.rdd.RDD

import scala.reflect.ClassTag

class UtilRdd[T: ClassTag](rdd: RDD[T]) {
  def abc(): Unit = {
    rdd.foreach(println)
  }
}

object UtilRdd {
  implicit def createRdd[T: ClassTag](rdd: RDD[T]): UtilRdd[T] = new UtilRdd[T](rdd)
}

```
