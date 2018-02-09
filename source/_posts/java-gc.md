---
title: java 错误及gc问题排查
date: 2018-01-28 20:02:15
tags:
    - frame
type: "categories"
---

最近参加了一次公司的分享，讲的非常的好。都是实战加方法论总结出来的东西。怕自己忘记，希望通过写博客的方式记录下来，顺便整理消化成自己的东西。

 
## 一，问题分类

### 1. jvm问题

#### (1) 分类

* 内存占用过大，内存溢出等
* 线程数量，线程死锁等
* GC问题
* ClassLoader/Class 问题(Class 未找到，转化失败) 

#### (2) 方法论

* 首先需要确认自己程序，是内存密集型还是cpu密集型服务。
* 在自己重要业务上一定要将日志加全，好在后期排查问题。
* 依赖外部开源监控程序，实时监控自己的业务。

#### （3）内存问题排查

排查其他系统占用

* 首先 top (shift+m) ,按内存排序，查看进程所占用的内存比率
* vmstat 3 (3秒一次)查看 Swap下的si(每秒交换回内存)，so(每秒交换出内存).这两个值越大证明系统越繁忙。

排查外部内存泄露

* ? 

确定为自己程序问题

* 如果还是有问题，查看GC，jstat -gcutil pid 5s 60 (每隔5秒，执行60次)，查看 FGC(full GC)情况。如果老年代内存一直不释放，且一直在做GC。内存没有释放，打印堆栈查看

* 首先 jmap -histo pid 打印出程序jvm class的实例数目,内存占用,类全名信息。对象分布情况。如果是web程序，需要查看nio的direct buffer，它通过jni申请了堆外内存，非jvm内存。 查看自己的连接是否未释放等。


#### (4) cpu 过高

* top (shift+p) 查看cpu使用率
* 超过预期值，使用jstack 查看当前线程再做什么操作
* 线程死锁，查看是否程序一直在GC，而导致的这个问题。
* 另一种可能就是当前程序某些线程有问题，使用 jstack 查看当前所有线程在干什么。

```
1. top (shift+p)       查看cpu使用率
2. jstat -gcutil  pid  查看gc情况
3. jmap –heap pid/jmap –histo pid 查看堆对象存储快照
4. top -Hp pid         查看当前进程的线程使用资源情况
5. printf 0x%x tid     线程id转化为16进制id
6. jstack pid | grep nid –A 10  使用jstack 查看 当前线程在做什么操作，后10行信息 
```

### 2. 问题发现点

#### (1). cpu 异常

* 复杂的CPU运算* 死循环* 频繁的fullgc#### (2). 内存异常* 堆内存* metaspace/thread stack* 非堆内存#### (3). IO异常* 流量异常，如频繁的远程数据库I/O* time\_wait/close\_wait

### 3. 工具

#### （1）linux

* top、free、vmstat、sysstat(iostat/sar)、netstat、iftop#### （2）jvm

* jps、jinfo、jmap、jstat、jstack#### （3） jmx

* jvisualvm及其相关插件、jmc、jfr，MXBean

#### （4） 其他

* mat
* google-perftools* btrace
* show-busy-java-threads

### 4. 博客推荐

* 陈皓 https://coolshell.cn * 飒然Hang http://www.rowkey.me * 你假笨 http://lovestblog.cn * 江南白衣 http://calvin1978.blogcn.com

注： 上面内容还没有完全整理完毕，会抽取时间不断完善。 
