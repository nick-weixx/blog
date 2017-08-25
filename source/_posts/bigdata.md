---
title: 技术框架介绍(大数据篇)
tags:
    - frame 
type: "categories" 
---
这是我第一篇博客文章，千里之行始于脚下，希望自己养成良好的习惯，记录下工作生活中的点点滴滴。每天进步一点，积少成多，沉淀升华。 整理了一下从业需要的了解的相关技术，想慢慢的一点点细化这些架构，组合运用，可以做出一些好玩的东西出来。


## hadoop

hadoop 是一个开发和运行大规模数据的软件平台，属于appach组织，是一个开源项目，主要由java编写实现。核心架构包含hdfs(分布式存储)与mapreduce(分布式计算)组成。目前最新版本为3.0.0。它最早设计用于搜索引擎，为搜索存储海量数据，并进行索引建立。

More info: [官网](http://hadoop.apache.org/)

### hdfs

hdfs是一个分布式存储系统，源自于Google的GFS论文，简单理解为google实现的克隆版。它是一个高度容错的系统，能检测和应对硬件故障，用于在低成本的通用硬件上运行。

![Alt text](http://img.blog.csdn.net/20140222161019968?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd29zaGl3YW54aW4xMDIyMTM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
Client：切分文件；访问HDFS；与NameNode交互，获取文件位置信息；与DataNode交互，读取和写入数据。

NameNode：Master节点，在hadoop1.X中只有一个，管理HDFS的名称空间和数据块映射信息，配置副本策略，处理客户端请求。

DataNode：Slave节点，存储实际的数据，汇报存储信息给NameNode。

Secondary NameNode：辅助NameNode，分担其工作量；定期合并fsimage和fsedits，推
送给NameNode；紧急情况下，可辅助恢复NameNode，但Secondary NameNode并非NameNode的热备
### mapreduce
源自于google的MapReduce论文，Hadoop MapReduce是google MapReduce 克隆版。是一个分布式计算框架，mapreduce 实现了算法迁移，写好的程序会被分配到各个datanode节点执行。mapreduce会频繁的读写本地文件，进行排序，归类。

![Alt text](http://img.blog.csdn.net/20140222162536546?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd29zaGl3YW54aW4xMDIyMTM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

input: 要读取的文件列表，获取文件大小位置信息

splitting: 对文件进行切割,分片到各个节点

mapping: 详细的读取逻辑，组合成 k-v 形式，存储文件

shuffing:  根据key合并结果集，将数据分发到不同的reduce

reducing: 合并后结果，具体的处理逻辑实现

final result: 输出计算结果


## hive

由facebook开源，解决了海量数据结构化查询的问题。运行在mapredece之上，记性了sql化的封装。适用于离线数据的计算，隔离了mapreduce编写，大大提升了效率。

![Alt text](http://img.blog.csdn.net/20140222161841875?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd29zaGl3YW54aW4xMDIyMTM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

More info: [官网](http://hive.apache.org/)

## zookeeper

源自Google的Chubby论文,是Hadoop和Hbase的重要组件。提供的功能包括：配置维护、域名服务、分布式同步、组服务等。可以理解为分布式的监听模式实现。

More info: [官网](http://zookeeper.apache.org/)

## spark
spark 为迭代式数据处理提供更好的支持。每次迭代的数据可以保存在内存中，而不是写入文件.相对于mapreduce框架，频繁的将文件读写到磁盘，性能上有极大的提升。而且spark提供了一套完整的解决方案，包括spark sql (交互封装),spark streaming (流式计算),MLlib(机器学习), GraphX(图计算)等一套完整方案 。它将动作封装成了一个个RDD，分为转化和动作两类。

More info: [官网](http://spark.apache.org/)

## oozie

是一个工作流框架，可以将mapreduce任务，hive任务，shell脚本，数据库访问。。。。,根据业务封装成工作流。工作流必须标记start,end节点，可以通过kill节点杀死任务，也可以通过fork&join节点并行的执行任务。同时提供web端的任务流查询，是一个非常优秀的一个框架。

More info: [官网](http://oozie.apache.org/)

## hbase

源自Google的Bigtable论文,由facebook开源,是一个键值对数据库，具有可伸缩，过性能，分布式的特点。可以对大规模数据进行随机访问，实时读写，是一个横向无限大的表。

![Alt text](http://www.bitstech.net/wp-content/uploads/2015/09/hbase.jpg?_=5571193)

More info: [官网](http://hbase.apache.org/)


## mahout
Mahout 是一个很强大的数据挖掘工具，是一个分布式机器学习算法的集合，包括：被称为Taste的分布式协同过滤的实现、分类、聚类等。Mahout最大的优点就是基于hadoop实现，把很多以前运行于单机上的算法，转化为了MapReduce模式，这样大大提升了算法可处理的数据量和处理性能。

More info: [官网](http://mahout.apache.org/)



