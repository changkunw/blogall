---
title: druid简介
date: 2017-01-01 22:36:16
tags: druid-io
---

## 什么是druid
　　首先说一下druid这个名字，可能大家都知道阿里巴巴有个开源的框架也加druid，但是我这里说的druid并不是阿里巴巴的druid,那个是数据库连接池，这里的druid是druid-io。
　　druid是一个为大型冷数据集上实时探索查询而设计的开源数据分析和存储系统，提供极具成本效益并且永远在线的实时数据摄取和任意数据处理。所以它主要是一个olap系统。
　　如果你现在在寻找一个实时的olap框架，那么druid可以纳入你的考虑范围之内。使用druid可以加载离线数据，也可以加载实时数据，加载数据之前，需要提前配置好指标和维度，用它的主要原因是由于它可以处理海量级数据，并且较快。
　　druid使用上其实还是不复杂，但是配置繁多，使用的时候要小心点。扩展集群比较方便，它依赖于zookeeper，所以添加的节点只需要在zookeeper中注册即可。

<!--more-->

## druid组件

### Overlord Node (Indexing Service)

Overlord会形成一个加载批处理和实时数据到系统中的集群，同时会对存储在系统中的数据变更（也称为索引服务）做出响应。另外，还包含了Middle Manager和Peons，一个Peon负责执行单个task，而Middle Manager负责管理这些Peons。意思就是这个组件主要是来建立索引的。

### Coordinator Node

监控Historical节点组，以确保数据可用、可复制，并且在一般的“最佳”配置。它们通过从MySQL读取数据段的元数据信息，来决定哪些数据段应该在集群中被加载，使用Zookeeper来确定哪个Historical节点存在，并且创建Zookeeper条目告诉Historical节点加载和删除新数据段。

### Historical Node

是对“historical”数据（非实时）进行处理存储和查询的地方。Historical节点响应从Broker节点发来的查询，并将结果返回给broker节点。它们在Zookeeper的管理下提供服务，并使用Zookeeper监视信号加载或删除新数据段。

### Broker Node 

接收来自外部客户端的查询，并将这些查询转发到Realtime和Historical节点。当Broker节点收到结果，它们将合并这些结果并将它们返回给调用者。由于了解拓扑，Broker节点使用Zookeeper来确定哪些Realtime和Historical节点的存在。

### Real-time Node

实时摄取数据，它们负责监听输入数据流并让其在内部的Druid系统立即获取，Realtime节点同样只响应broker节点的查询请求，返回查询结果到broker节点。旧数据会被从Realtime节点转存至Historical节点。

### ZooKeeper

为集群服务发现和维持当前的数据拓扑而服务；

###  Deep Storage
即数据的存放，可以选在存在本地(需要共享目录)，也可以放在hdfs或者s3

### Mysql
元数据存放的位置的地方。


## 数据查询

数据查询都是通过查询broker节点，它提供了restApi,还是比较方便，但是有很多其他框架支持druid的查询，比如plyql（sql）、pivot（druid比较好的数据展示ui）、superset（更炫酷的UI）

## 架构

![](/img/druidstruct.jpg)