---
title: lily hbase index 配置整理
date: 2017-05-22 12:29:03
tags:
    - frame 
type: "categories" 
---

注明：因为这部分内容涉及到工作，所以笔者会将真实的业务逻辑隐藏，呈现的例子不具有很强的业务性，如果数据场景不恰当还请见谅，就当一个解决方案看一下就好。

## 背景

CDH5.3.2中的Key-Value Indexer使用的是Lily Hbase NRT Indexer服务，Lily HBase Indexer是一款灵活的、可扩展的、高容错的、事务性的，并且近实时的处理HBase列索引数据的分布式服务软件。它是NGDATA公司开发的Lily系统的一部分，已开放源代码。Lily HBase Indexer使用SolrCloud来存储HBase的索引数据，当HBase执行写入、更新或删除操作时，Indexer通过HBase的replication功能来把这些操作抽象成一系列的Event事件，并用来保证写入Solr中的HBase索引数据的一致性。并且Indexer支持用户自定义的抽取，转换规则来索引HBase列数据。

## 配置

### 1. 创建hbase表

```
##创建名称为basicdata 的空间，类似于库的概念
create_namespace 'basic_data'
##basic_data库中创建user_info表，指定列族名称为data，只留一个版本，采用lzo压缩，并开启同步功能。
create 'basic_data:user_info',{NAME => 'data', VERSIONS => 1,COMPRESSION => 'lzo',REPLICATION_SCOPE => 1}
```
创建完成之后可以通过，http://xxxx.xxx.xx:60010 查看表是否创建情况

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/lily_index_1.png)

### 2. 创建SolrCloud 集合
(1） 使用 instancedir generate 命令,生成solr的基本配置文件。
```
solrctl instancedir --generate ~/weixingxin/basic_data:user_info
```

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/lily%20index.png)

(2) 修改~/weixingxin/basic_data:user_info/conf/schema.xml 文件，配置需要索引的列信息及规则。

```
   <field name="user_id" type="string" indexed="true" stored="true" required="true" multiValued="false" />
   <field name="user_name" type="string" indexed="true" stored="true" required="false" multiValued="false" />
   <field name="sex" type="string" indexed="true" stored="true" required="false" multiValued="false" />
   <field name="age" type="long" indexed="true" stored="true" required="false" multiValued="false" />

```

(3)创建collection 名为basic_data_user_info

```
solrctl instancedir --create basic_data_user_info ~/weixingxin/basic_data:user_info
```
运行命令后，solr会将文件同步到各个zookpeer节点上，实现分布式配置。

(4) 创建basic_data_user_info 的分片数为4

```
solrctl collection --create basic_data_user_info -s 4

```

(5)其他命令
```
##查看已经创建好的所有集合
solrctl instancedir --list
##删除创建好的solr索引
solrctl collection --delete basic_data_user_info
##删除集合配置
solrctl instancedir --delete basic_data_user_info

```

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/lily_index_2.png)

### 3. lily index 配置
(1) 配置Morphlines文件，此文件映射了hbase的列与solr的关系

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/lily_index_3.png)

```
SOLR_LOCATOR : {
# Name of solr collection
collection : basic_data_user_info
# ZooKeeper ensemble
zkHost : "$ZK_HOST"
}
morphlines : [
{
	id : basic_data_user_info_Map
	importCommands : ["org.kitesdk.**", "com.ngdata.**"]
	commands : [
		{
			extractHBaseCells {
				mappings : [
				{
				inputColumn : "data:user_id"
				outputField : "user_id"
				type : string
				source : value
				},
				{
				inputColumn : "data:user_name"
				outputField : "user_name"
				type : long
				source : value
				},
				{
				inputColumn : "data:sex"
				outputField : "sex"
				type : long
				source : value
				},
				{
				inputColumn : "data:age"
				outputField : "age"
				type : string
				source : value
				}]
			}
		}
		{ logDebug { format : "output record: {}", args : ["@{}"] } }
	]
}]

```

(2) 配置morphline-hbase-mapper.xml 文件，将Morphlines 与 hbase basic_data:user_info表 关联

```
<?xml version="1.0"?>
 <!-- table：需要索引的HBase表名称-->
 <!-- mapper：用来实现和读取指定的Morphline配置文件类，固定为MorphlineResultToSolrMapper-->
<indexer table="basic_data:user_info" read-row="never"  mapper="com.ngdata.hbaseindexer.morphline.MorphlineResultToSolrMapper">
   <!--param中的name参数用来指定当前配置为morphlineFile文件 -->
   <!--value用来指定morphlines.conf文件的路径，绝对或者相对路径用来指定本地路径，如果是使用Cloudera Manager来管理morphlines.conf就直接写入值morphlines.conf"-->
   <param name="morphlineFile" value="morphlines.conf"/>
      <!-- The optional morphlineId identifies a morphline if there are multiple morphlines in morphlines.conf -->
   <param name="morphlineId" value="basic_data_user_info_Map"/>
</indexer>

```

(3) 同步morphline-hbase-mapper.xml配置文件

```
hbase-indexer add-indexer \
 --name cloudIndexer \
 --indexer-conf \
~/weixingxin/solr_config/morphline-hbase-mapper.xml \
 --connection-param solr.zk=hadoops15:2181/solr \
 --connection-param solr.collection=basic_data_user_info \
 --zookeeper hadoops15:2181
 
```

查看更多命令参数：hbase-indexer add-indexer --help


-----------------------------------------------
到此lily index 的配置完成，下一篇导入数据，测试索引。。。

