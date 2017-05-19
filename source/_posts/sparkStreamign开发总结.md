---
title: sparkStreamign开发总结
date: 2017-03-12 23:16:51
tags: SparkStreaming
---

>本文档只是针对现目前在碰见的一些在SparkStreming开发中遇到的一些问题和现象做一些汇总，都是个人总结，有可能有一些不正确的地方，这里说的都是针对spark1.6 目前最新的版本2.0都对有些地方进行了优化。持续更新

## kafka
目前公司开发SparkStreaming的数据来源只有kafka，所以kafka的使用直接决定SparkStreaming处理任务的效率。

<!--more-->
### 磁盘数
Kafka其实对内存的要求没有多高，但是对磁盘的要求比较高，并且最好是多磁盘，这样能够加快kafka的对数据的读写效率，kafka的核心思想就是使用磁盘而非内存。这样做的原因，from internet：

 1. 磁盘缓存由Linux系统维护，减少了程序员的不少工作。
 2. 磁盘顺序读写速度超过内存随机读写。
 3. JVM的GC效率低，内存占用大。使用磁盘可以避免这一问题。
 4. 系统冷启动后，磁盘缓存依然可用。

### topic分区数

数据放入到kafka中，然后就是消费者从kafka中进行消费，而消费者对topic进行消费的效率很大一部分取决与topic的分区数，topic有几个分区直接决定了有多少个线程能够同时从kafka获取数据，举个栗子，如果topic分区数为3但是你起了4个线程从kafka中消费数据，那么有一个线程肯定是消费不到数据的，除非其中一个线程挂掉。
那么是不是topic的分区数数是不是越多越好呢？topic的分区多了虽然可以加快消费这消费的效率，但是topic的分区数还和磁盘的数量有关，最理想的情况是一块磁盘一个分区。所以上文强调磁盘的重要，所以最好kafka的磁盘是多磁盘而不是一块很大的磁盘。

## SparkStreaming

### executor的数量
上文说了topic的分区数，那么sparkStreaming作为kafka的consumer，所以一般topic有几个分区就启动几个executor的数量，那当然不是这样，如果spark的集群比kafka的集群小，那就不用说了，配置最多的executor数量吧。  
如果spark集群的数量远远大于kafka集群数量，那么如果kafka的分区数少配置的executor的数量也少的话，数据量大的话肯定是不能达到spark处理数据的效果的，只能造成消息堆积，所以executor数肯定需要配置的多一些  

### executor 数大于topic分区数  == > repartition

假如：executor的数量为8，topic的分区数为3  

那么streaming任务启动后，肯定只有3个executor能获取到数据的，所以默认这些数据只能在这3个executor中计算。除非到了后面有shuffle操作才会将数据分布到其他节点，那么这样就会造成开始的计算分布不均，所以只有让数据从kafka消费出来就进行一次数据重新分布，那么就做repartition吧，做了之后会把数据分布到其他节点上。但是repartition是有代价的，做repartiton又会增加网络等开销，数据量大也会话费比较长的时间。下图是在sparkUI上查看

![](/img/sparkStreaming1.png)


 
### task数量

Spark之所以能够对那么大的数据进行计算就是考的executor上能够起多个task并发执行，所以你的executor上的task数量多也是加快spark执行效率的原因，那么task是如何确定的呢，一个spark任务中能够同时启动的task数量就是executor数乘以executor的核数，spark中还有个参数是``spark.default.parallelism``，这个参数是设置每个stage的task数量，这个对应的sparkSql中的参数是：``spark.sql.shuffle.partitions``，如果你设置了这些东西task的数量都没有起来很多的话，有很大的原因就是上面说的partition数量不够，做了repartition后可以提高task的数量，task的数据量多少还跟你的任务具体业务来判定，如果需要汇聚计算的比较多那肯定是不适合多太多task的，下图是在sparkUI上查看task的图实例：

![](/img/sparkStreaming2.png)

 

### 从stage里面查找任务执行缓慢的原因	

![](/img/sparkStreaming3.png)

 上图中是一个stage执行过程中每个executor执行过程的汇总的结果，从途中可以很明显的看出有executor上的执行只用了1、2s，而有的却用了将近1分钟时间，所以你一看就知道肯定有什么地方没对，要么数据机器问题（当然概率比较小），要么就是数据倾斜了



![](/img/sparkStreaming4.png)

 
上图是一张详细的stage的所有task执行过程图，需要说明的几列的意思：

Locality Levels：表示task执行时数据的来源是哪儿。
 1. PROCESS_LOCAL: 数据在同一个 JVM 中，即同一个 executor 上。这是最佳数据 locality。
 2. NODE_LOCAL: 数据在同一个节点上。比如数据在同一个节点的另一个 executor上；或在 HDFS 上，恰好有 block 在同一个节点上。速度比 PROCESS_LOCAL 稍慢，因为数据需要在不同进程之间传递或从文件中读取
 3. NO_PREF: 数据从哪里访问都一样快，不需要位置优先
 4. RACK_LOCAL: 数据在同一机架的不同节点上。需要通过网络传输数据及文件 IO，比 NODE_LOCAL 慢
 5. ANY: 数据在非同一机架的网络上，速度最慢

 GC: 这个task执行时gc花了多少时间，从上图可以看出，这些task的gc时间都花了很长的时间，这样出现在sparkStreming中肯定是不正常的，但是你可以对GC进行一些调优使GC的时间短一些，但是这时间太长了，肯定有问题，检查代码吧。    
倒数第二列和第三列可以看出落在这个task上的数据量大小，从而也可以发现一些数据倾斜的相关线索。  

### spark反压机制
``spark.streaming.kafka.maxRatePerPartition ``
这个参数可以配置spark每个批次接收到的数据量的大小n条  
每个批次接受到的数据量大小等于 batch time * n * topic 分区数  

### spark job 并行度
``spark.streaming.concurrentJobs``
此参数可以配置一个同时可以执行几个job，如下图，默认这个参数是1，也就是说你最多在下图中看到的active batches 处于processing的只会有一个，如果再来一个的话并且上一个还没处理完的话任务就会处于排队状态，这样就会只有等上一个批次处理完了再处理下一个，如果配置了这个参数可以多个批次同时处理。
![](/img/sparkStreaming5.png)

 
### spark GC打印
添加参数如：spark.executor.extraJavaOptions=" -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps”
当然还可以添加一些其他的gc参数。
参看executor的gc日志：
 
![](/img/sparkStreaming6.png)

## 总结

SparkStreming开发的时候需要根据实际情况对spark做相应的调整，task的数量也不一定非要很多，太多的话，可能造成shuffle的时候花费太多时间。这里有一个表象就是任务在执行的时候你在sparkUI上看老是在最后一步（一般都是action）花了很长时间，所以就容易产生误解，导致错误的修改，其实很多时候是前面的transfrom操作花了很多时间，但是你从界面上看起来老是在做最后一步操作。Spark里面每个action都能看到花了很多时间的，找耗时原因的时候需可以根据sparkUI上的job时间汇总进行具体的优化。如下图可以很清楚的看到在哪个action里花了多少时间，总之如果发现任务耗时很长的话一定要细细看时间到底花在了哪个地方

![](/img/sparkStreaming7.png)