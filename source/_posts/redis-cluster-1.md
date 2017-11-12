---
title: redis cluster 集群搭建
date: 2017-10-10 20:02:15
tags:
    - frame
type: "categories"
---

## redis 集群搭建


### 1, 结构简述

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/redis_cluster_1.jpg)

Redis 集群是一个可以在多个 Redis 节点之间进行数据共享的设施（installation）。

Redis 集群不支持那些需要同时处理多个键的 Redis 命令， 因为执行这些命令需要在多个 Redis 节点之间移动数据， 并且在高负载的情况下， 这些命令将降低 Redis 集群的性能， 并导致不可预测的行为。

Redis 集群通过分区（partition）来提供一定程度的可用性（availability）： 即使集群中有一部分节点失效或者无法进行通讯， 集群也可以继续处理命令请求。

Redis 集群提供了以下两个好处：

* 将数据自动切分（split）到多个节点的能力。
* 当集群中的一部分节点失效或者无法进行通讯时， 仍然可以继续处理命令请求的能力。

Redis 集群使用数据分片（sharding）而非一致性哈希（consistency hashing）来实现： 一个 Redis 集群包含 16384 个哈希槽（hash slot）， 数据库中的每个键都属于这 16384 个哈希槽的其中一个， 集群使用公式 CRC16(key) % 16384 来计算键 key 属于哪个槽， 其中 CRC16(key) 语句用于计算键 key 的 CRC16 校验和 。

集群中的每个节点负责处理一部分哈希槽。 举个例子， 一个集群可以有三个哈希槽， 其中：

* 节点 A 负责处理 0 号至 5500 号哈希槽
* 节点 B 负责处理 5501 号至 11000 号哈希槽。
* 节点 C 负责处理 11001 号至 16384 号哈希槽。

这种将哈希槽分布到不同节点的做法使得用户可以很容易地向集群中添加或者删除节点。 比如说：

* 如果用户将新节点 D 添加到集群中， 那么集群只需要将节点 A 、B 、 C 中的某些槽移动到节点 D 就可以了。

* 与此类似， 如果用户要从集群中移除节点 A ， 那么集群只需要将节点 A 中的所有哈希槽移动到节点 B 和节点 C ， 然后再移除空白（不包含任何哈希槽）的节点 A 就可以了。

因为将一个哈希槽从一个节点移动到另一个节点不会造成节点阻塞， 所以无论是添加新节点还是移除已存在节点， 又或者改变某个节点包含的哈希槽数量， 都不会造成集群下线。

为了使得集群在一部分节点下线或者无法与集群的大多数（majority）节点进行通讯的情况下， 仍然可以正常运作， Redis 集群对节点使用了主从复制功能： 集群中的每个节点都有 1 个至 N 个复制品（replica）， 其中一个复制品为主节点（master）， 而其余的 N-1 个复制品为从节点（slave）。

在之前列举的节点 A 、B 、C 的例子中， 如果节点 B 下线了， 那么集群将无法正常运行， 因为集群找不到节点来处理 5501 号至 11000号的哈希槽。

另一方面， 假如在创建集群的时候（或者至少在节点 B 下线之前）， 我们为主节点 B 添加了从节点 B1 ， 那么当主节点 B 下线的时候， 集群就会将 B1 设置为新的主节点， 并让它代替下线的主节点 B ， 继续处理 5501 号至 11000 号的哈希槽， 这样集群就不会因为主节点 B 的下线而无法正常运作了。

不过如果节点 B 和 B1 都下线的话， Redis 集群还是会停止运作。

架构细节:

* 所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽.

* 节点的fail是通过集群中超过半数的节点检测失效时才生效.

* 客户端与redis节点直连,不需要中间proxy层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可

* redis-cluster把所有的物理节点映射到[0-16383]slot上,cluster 负责维护node<->slot<->value


### 2. 安装

#### (1) 安装包下载

* redis 安装包

```
wget http://download.redis.io/releases/redis-3.2.9.tar.gz
```

* ruby 安装包

```
 wget https://cache.ruby-lang.org/pub/ruby/2.2/ruby-2.2.8.tar.gz
```

#### (2) 安装redis

* 编译安装包

```
cd redis-3.2.9
make && make install

```

* 创建节点

```
cd redis-3.2.9
mkdir cluster/40000
mkdir cluster/40001
mkdir cluster/40002

```

* 复制配置文件到节点

```
cp redis.conf cluster/40000
cp redis.conf cluster/40001
cp redis.conf cluster/40002
cp redis.conf cluster/40003
cp redis.conf cluster/40004
cp redis.conf cluster/40005


```

* 更具端口及ip，修改各个节点的配置文件

```
##端口
port  40001
##本地ip
bind  192.168.0.1
##redis后台运行
daemonize    yes
##redis后台运行
pidfile  /var/run/redis_4001.pid
## 开启集群 
cluster-enabled  yes
## 集群的配置  配置文件首次启动自动生成
cluster-config-file  nodes_4001.conf
## 请求超时  默认15秒，可自行设置
cluster-node-timeout  15000
## aof日志开启  有需要就开启，它会每次写操作都记录一条日志
##appendonly  yes
##权限校验
##requirepass redis_password
##masterauth redis_password

```

* 复制修改后的程序到各个节点(略)

* 在每台服务器分别启动主节点

```
ssh s1
cd cluster/40000
redis-server redis.conf

ssh s2
cd cluster/40001
redis-server redis.conf

ssh s3
cd cluster/40002
redis-server redis.conf
```

* 驱动主节点

先不做从库备份，只做master节点。redis-trib.rb在一个节点运行即可

```
${redis_home}/src/redis-trib.rb create --replicas 0 s1_ip:40000 s1_ip: 40001 s1_ip: 40002

```

* 获取master节点id

```
redis-client 运行 , CLUSTER NODES 命令
 
```

* 添加并启动slave 节点

```
ssh s1
cd cluster/40003
redis-server redis.conf

ssh s2
cd cluster/40004
redis-server redis.conf

ssh s3
cd cluster/40005
redis-server redis.conf

redis-trib.rb add-node --slave --master-id s1主节点id s1_ip:41003 s1_ip:41000

redis-trib.rb add-node --slave --master-id s2主节点id s1_ip:41004 s1_ip:41001

redis-trib.rb add-node --slave --master-id s3主节点id s1_ip:41005 s1_ip:41002
```

#### （3） 安装ruby

```
tar -xvzf ruby-2.2.8.tar.gz    
cd ruby-2.2.8
 ./configure
```

使用 ruby -v 严重安装是否成功

安装 redis模块

```
gem install redis
```


### 3,注意

必须要3个或以上的主节点，否则在创建集群时会失败，并且当存活的主节点数小于总节点数的一半时，整个集群就无法提供服务了。
