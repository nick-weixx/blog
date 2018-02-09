---
title: guava 不可变集合
date: 2017-11-11 20:02:15
tags:
    - frame
type: "categories"
---

## 不可变集合

* 规避了线程竞争条件，以保证线程的安全。
* 不必支持可变性，也就是不需要根据原子自动扩容空间。也就不需要初始化时预留空间位置。从而节省了内存空间。
* 可以作为常量使用

### 1. Collections.unmodifiableSet 的问题


```
		//Collections.unmodifiableSet
        Set<String> colors=new HashSet<String>();                                                                               
        colors.add("red");
		colors.add("green");
		colors.add("blue");
        
        Set<String> finalColors=Collections.unmodifiableSet(colors);
        System.out.println("变化前："+finalColors);
        colors.add("test");
        System.out.println("变化后："+finalColors);
```


* 它用起来笨拙繁琐你不得不在每个防御性编程拷贝的地方用这个方法
* 它不安全：如果有对象reference原始的被封装的集合类，这些方法返回的集合也就不是正真的不可改变。
* 效率低：因为它返回的数据结构本质仍旧是原来的集合类，所以它的操作开销，包括并发下修改检查，hash table里的额外数据空间都和原来的集合是一样的。


### 2. immutable集合

```
		// copyof 创建
		Set<String> colors = Sets.newHashSet();
		colors.add("red");
		colors.add("green");
		colors.add("blue");
		ImmutableSet<String> finalColors = ImmutableSet.copyOf(colors);
		System.out.println(finalColors);

		// build 创建
		finalColors = ImmutableSet.<String> builder().add("red", "green", "blue", "t1").build();
		System.out.println(finalColors);
		// of 创建
		finalColors = ImmutableSet.of("red", "green", "blue", "t2");
		System.out.println(finalColors);

		// finalColors.add("test");//报错，不能添加
		System.out.println("all: " + finalColors.asList());
		String isRul = finalColors.contains("purple") ? "有" : "没有";
		System.out.println("有紫色吗? " + isRul);
```

* immtable拒绝接受空值(Null),会报错。
* 使用简单方便。
* 创建只有独立与引用对象，实现了真正的不可变。

### 3. immtable集合和集合关系
|可变集合类型|可变集合源 (JDK or Guava)|不可变集合|说明|
|---|---|---|---|
|Collection |	JDK	| ImmutableCollection|集合|
|List|	JDK	 | ImmutableList| 集合 arraylist,linedlist|
|Set|	JDK	 | ImmutableSet| hashset|
|SortedSet/NavigableSet|	JDK	 | ImmutableSortedSet|---|
|Map|	JDK	 | ImmutableMap|hashmap等|
|SortedMap|	JDK	 | ImmutableSortedMap|有序map|
|Multiset|	Guava | ImmutableMultiset| 容许重复，但是没有顺序，类 Map&#60;String, Integer&#62;|
|SortedMultiset|	Guava | ImmutableSortedMultiset|---|
|Multimap|	Guava | ImmutableMultimap|Map&#60;K, Set&#62;|
|ListMultimap|	Guava |	ImmutableListMultimap|Map&#61;K, List&#62;|
|SetMultimap|	Guava	| ImmutableSetMultimap|--?--|
|BiMap |	Guava|	ImmutableBiMap|--?|
|ClassToInstanceMap|	Guava |	ImmutableClassToInstanceMap|--?|
|Table|	Guava	| ImmutableTable|Map&#60;String,Map&#60;String,integet&#62;&#62;|


