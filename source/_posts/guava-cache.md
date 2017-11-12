---
title: guava LoadingCache 缓存
date: 2017-08-30 15:50:14
tags:
    - frame
type: "categories"
---

Guava 工程包含了若干被 Google 的 Java 项目广泛依赖 的核心库，我们希望通过此文档为 Guava 中最流行和最强大的功能，提供更具可读性和解释性的说明。先从缓存开始学习，记录下点点滴滴，满满的进步。

## 适用性

缓存在很多场景下都是相当有用的。例如，计算或检索一个值的代价很高，并且对同样的输入需要不止一次获取值的时候，就应当考虑使用缓存。

Guava Cache 与 ConcurrentMap 很相似，但也不完全一样。最基本的区别是 ConcurrentMap 会一直保存所有添加的元素，直到显式地移除。相对地，Guava Cache 为了限制内存占用，通常都设定为自动回收元素。在某些场景下，尽管 LoadingCache 不回收元素，它也是很有用的，因为它会自动加载缓存。

通常来说，Guava Cache 适用于：

你愿意消耗一些内存空间来提升速度。
你预料到某些键会被查询一次以上。
缓存中存放的数据总量不会超出内存容量。（Guava Cache 是单个应用运行时的本地缓存。它不把数据存放到文件或外部服务器。如果这不符合你的需求，请尝试 Memcached 这类工具）
如果你的场景符合上述的每一条，Guava Cache 就适合你。

如同范例代码展示的一样，Cache 实例通过 CacheBuilder 生成器模式获取，但是自定义你的缓存才是最有趣的部分。

注：如果你不需要 Cache 中的特性，使用 ConcurrentHashMap 有更好的内存效率——但 Cache 的大多数特性都很难基于旧有的 ConcurrentMap 复制，甚至根本不可能做到。

## 自动加载

在使用缓存前，首先问自己一个问题：有没有合理的默认方法来加载或计算与键关联的值？如果有的话，你应当使用 CacheLoader。如果没有，或者你想要覆盖默认的加载运算，同时保留"获取缓存-如果没有-则计算"[get-if-absent-compute]的原子语义，你应该在调用 get 时传入一个 Callable 实例。缓存元素也可以通过 Cache.put 方法直接插入，但自动加载是首选的，因为它可以更容易地推断所有缓存内容的一致性。

### CacheLoader

```
LoadingCache<String, RatioItem> pctmapping = CacheBuilder.newBuilder().maximumSize(100 * 1000).expireAfterWrite(1, TimeUnit.MINUTES)
			.build(new CacheLoader<String, RatioItem>() {
				@Override
				public RatioItem load(String key) throws Exception {
					List<RatioItem> ratioItem = expressionDao.getExpressByAlgorithm(key);
					if (ratioItem.size() > 0) {
						return ratioItem.get(0);
					}else{
						RatioItem item =new RatioItem();
						item.setTargetName("UNKNOW");
						return item;
					}
				}
			});

```

这个例子中，我们通过key从数据库加载数据对象，当调用get方法，发现pctmapping 不存在这个值的时候，会自动加载load方法，寻找数据库中是否包含 map key所对应的value。

注意：LoadingCache 的返回值是不接受null值的，会报错。如果不存在请在加载逻辑中做判断。返回UNKONW对象信息。

配置项：

maximumSize： 最大添加数量，设置为10W

expireAfterAccess：1分钟内没有被重写，则清理

## Callable
所有类型的Guava Cache，不管有没有自动加载功能，都支持get(K, Callable<V>)方法。这个方法返回缓存中相应的值，或者用给定的Callable运算并把结果加入到缓存中。在整个加载方法完成前，缓存项相关的可观察状态都不会更改。这个方法简便地实现了模式"如果有缓存则返回；否则运算、缓存、然后返回"。

```
			try {
				pctmapping.get(column, new Callable<RatioItem>() {
					@Override
					public RatioItem call() throws Exception {
						RatioItem ratioItem=new RatioItem();
						ratioItem.setTargetName("UNKNOW");
						return ratioItem;
					}
				});
			} catch (ExecutionException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
```

显式插入(不推荐)

使用cache.put(key, value)方法可以直接向缓存中插入值，这会直接覆盖掉给定键之前映射的值。使用Cache.asMap()视图提供的任何方法也能修改缓存。但请注意，asMap视图的任何方法都不能保证缓存项被原子地加载到缓存中。进一步说，asMap视图的原子运算在Guava Cache的原子加载范畴之外，所以相比于Cache.asMap().putIfAbsent(K,V)，Cache.get(K, Callable<V>) 应该总是优先使用。


## 缓存回收

cache提供了三种基本的缓存回收方式：基于容量回收、定时回收和基于引用回收

### 基于容量的回收（size-based eviction）

如果要规定缓存项的数目不超过固定值，只需使用CacheBuilder.maximumSize(long)。缓存将尝试回收最近没有使用或总体上很少使用的缓存项。

* 警告：在缓存项的数目达到限定值之前，缓存就可能进行回收操作——通常来说，这种情况发生在缓存项的数目逼近限定值时。


### 定时回收（Timed Eviction）

(1) CacheBuilder提供两种定时回收的方法：

* expireAfterAccess(long, TimeUnit)：缓存项在给定时间内没有被读/写访问，则回收。请注意这种缓存的回收顺序和基于大小回收一样。

* expireAfterWrite(long, TimeUnit)：缓存项在给定时间内没有被写访问（创建或覆盖），则回收。如果认为缓存数据总是在固定时候后变得陈旧不可用，这种回收方式是可取的。


(2) 测试定时回收

对定时回收进行测试时，不一定非得花费两秒钟去测试两秒的过期。你可以使用Ticker接口和CacheBuilder.ticker(Ticker)方法在缓存中自定义一个时间源，而不是非得用系统时钟。

### 基于引用的回收（Reference-based Eviction）

通过使用弱引用的键、或弱引用的值、或软引用的值，Guava Cache可以把缓存设置为允许垃圾回收：

* CacheBuilder.weakKeys()：使用弱引用存储键。当键没有其它（强或软）引用时，缓存项可以被垃圾回收。因为垃圾回收仅依赖恒等式（==），使用弱引用键的缓存用==而不是equals比较键。

* CacheBuilder.weakValues()：使用弱引用存储值。当值没有其它（强或软）引用时，缓存项可以被垃圾回收。因为垃圾回收仅依赖恒等式（==），使用弱引用值的缓存用==而不是equals比较值。

* CacheBuilder.softValues()：使用软引用存储值。软引用只有在响应内存需要时，才按照全局最近最少使用的顺序回收。考虑到使用软引用的性能影响，我们通常建议使用更有性能预测性的缓存大小限定（见上文，基于容量回收）。使用软引用值的缓存同样用==而不是equals比较值。

### 显式清除

任何时候，你都可以显式地清除缓存项，而不是等到它被回收：

* 个别清除：Cache.invalidate(key)
* 批量清除：Cache.invalidateAll(keys)
* 清除所有缓存项：Cache.invalidateAll()

### 移除监听器

通过CacheBuilder.removalListener(RemovalListener)，你可以声明一个监听器，以便缓存项被移除时做一些额外操作。缓存项被移除时，RemovalListener会获取移除通知[RemovalNotification]，其中包含移除原因[RemovalCause]、键和值。

```
//已数据库连接池为例，load 根据key加载连接。RemovalListener监听移除时，关闭连接。
CacheLoader<Key, DatabaseConnection> loader = new CacheLoader<Key, DatabaseConnection> () {
    public DatabaseConnection load(Key key) throws Exception {
        return openConnection(key);
    }
};

RemovalListener<Key, DatabaseConnection> removalListener = new RemovalListener<Key, DatabaseConnection>() {
    public void onRemoval(RemovalNotification<Key, DatabaseConnection> removal) {
        DatabaseConnection conn = removal.getValue();
        conn.close(); // tear down properly
    }
};
 
return CacheBuilder.newBuilder()
    .expireAfterWrite(2, TimeUnit.MINUTES)
    .removalListener(removalListener)
    .build(loader);
```
