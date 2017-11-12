---
title: jmap命令
date: 2017-10-12 22:17:49
tags:
    - frame
type: "categories"
---

## jmap

jmap是一个可以输出所有内存中对象的工具，甚至可以将VM 中的heap，以二进制输出成文本。打印出某个java进程（使用pid）内存内的，所有‘对象’的情况（如：产生那些对象，及其数量）。

### 1，options： 
* jmap [option] <pid> (to connect to running proces)通过进程id获取，获取VM的堆信息

* jmap [option] <executable <core> (to connect to a core file) 

* jmap [option] [server_id@]<remote server IP or hostname>(to connect to remote debug server) 远程debug服务的主机名或ip

### 2，基本参数：
* -heap 打印heap的概要信息，GC使用的算法，heap的配置及wise heap的使用情况
* -histo[:live]  打印每个class的实例数目,内存占用,类全名信息. VM的内部类名字开头会加上前缀”*”. 如果live子参数加上后,只统计活的对象数量. 

* -clstats 打印被加载的静态变量 
* -finalizerinfo 打印正等候回收的对象的信息
* -dump:<dump-options> 使用hprof二进制形式,输出jvm的heap内容到文件

例1：

```
## 打印pid为13816进程的，所有jedis对象
jmap -histo 20326 |grep jedis

```

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/jmap_1.png) 

备注： 编号，引用数(对象数量)，占用字节大小，类名

例2：

```
## -j-d64指64位操作系统,13816为进程id，file为输出堆文件
jmap -J-d64 -dump:format=b,file=dump_123.bin 20326

```

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/jmap_2.png) 


备注： 将堆内所有对象答应出来，生成快照文件。我们使用eclipse 的 mat 工具分析堆信息。
可以生成内存分析图，及占用内存最大的对象信息。


## 使用jmx实现远程监控

### 1，启动程序中添加如下参数

-Dcom.sun.management.jmxremote.port=8999
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false

### 2, 运行jconsole命令

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/jconsole_1.png) 

输入指定好的端口及ip

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/jconsole_2.png) 

通过Mbean 我们可以看到，JedisCluster对象，通过commons-pool2，实现了连接池。每一个pool对象对应一台机器的连接池。
通过这个工具我们可以监控到，堆内存占用大小，对象数量变化，及线程的活跃情况。这里简要进行介绍。以后遇到问题，会逐步完善次文档。
