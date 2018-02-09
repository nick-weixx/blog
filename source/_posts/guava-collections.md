---
title: guava 新集合类型
date: 2017-09-22 20:02:15
tags:
    - frame
type: "categories"
---

## Multiset
Guava提供了一个新集合类型 Multiset，它可以多次添加相等的元素。类似：Map&#60;String, Integer&#62;

举个例子：

```
	   final String[] names = new String[] { "nick", "davide", "ken", "ken", "nick", "nick" };
       System.out.println("hashmap:");
		Map<String, Integer> hashMap = new HashMap<String, Integer>();
		for (int i = 0; i < names.length; i++) {
			String name = names[i];
			if (hashMap.containsKey(name)) {
				hashMap.put(name, hashMap.get(name) + 1);
			} else {
				hashMap.put(name, 1);
			}
		}

		for (Map.Entry<String, Integer> entry : hashMap.entrySet()) {
			System.out.println("name: " + entry.getKey() + " ,count: " + entry.getValue());

		}

		System.out.println("\n" + "hashmultiset:");
		HashMultiset<String> hashMultiSet = HashMultiset.create();
		for (int i = 0; i < names.length; i++) {
			hashMultiSet.add(names[i]);
		}
		
		System.out.println(hashMultiSet);
		for (Multiset.Entry<String> name : hashMultiSet.entrySet()) {
			System.out.println("name: " + name.getElement() + " ,count: " + hashMultiSet.count(name.getElement()));
		}
	}
```
HashMap 在实现key值出现次数的统计时，必须拿到原来的值，并重新赋值会原来的集合。还需要判断当前key是否贼map中。

使用 HashMultiset ,我们不需要判断key是否已经存在于集合，也不用自己做值的累加。节省了代码量并且看着非常的直观。 

guava为我们提供了如下Multiset类型集合：

| map类型            | multiset实现            | 是否支持null  | 是否线程安全 | 含义         |
| ------------------|:-----------------------:| -----------:|------------| ---------    |
| HashMap           | HashMultiset            | Y           |N           |hash表map      |
| TreeMap           | TreeMultiset            | Y           |N           |具有排序的map   |
| LinkedHashMap     | LinkedHashMultiset      | Y           |N           |链表方式实现的map |
| ConcurrentHashMap | ConcurrentHashMultiset  | N           |Y           | 线程安全的hash 表map|
| ImmutableMap(guava)  | ImmutableMultiset       | N           |Y    | 不可变集合 |



## Multimap
我们经常使用Map&#60;K, List&#60;V&#62;&#62;或Map&#60;K, Set&#60;V&#62;&#62;，写业务代码，这个结构非常的笨拙。guava提供了，multimap已便我们用来实现业务。

存储的数据结构类似于：

a -> [1, 2, 4] b -> 3 c -> 5

举个例子：

```
       System.out.println("hashMap:");
		Map<String, Integer> hashMap = new HashMap<String, Integer>();
		hashMap.put("nick", 23);
		hashMap.put("nick", 22);
		hashMap.put("ken", 56);
		hashMap.put("ken", 58);
		hashMap.put("david", 78);
		hashMap.put("", 78);
		hashMap.put(null, 78);
		for (String demo : hashMap.keySet()) {
			System.out.println(demo + "\t" + hashMap.get(demo));
		}
		
		
		//k-set结构
		System.out.println("\n" + "hashMultimap:");
		HashMultimap<String, Integer> hashMultimap = HashMultimap.create();
		hashMultimap.put("nick", 23);
		hashMultimap.put("nick", 22);
		hashMultimap.put("ken", 56);
		hashMultimap.put("ken", 58);
		hashMultimap.put("david", 78);
		hashMultimap.put("", 78);
		hashMultimap.put(null, 78);
		for (String demo : hashMultimap.keySet()) {
			System.out.println(demo + "\t" + hashMultimap.get(demo));
		}
		
		//k-list 结构
		System.out.println("\n" + "arrayListMultiMap:");
		ArrayListMultimap<String, Integer> arrayListMultiMap =ArrayListMultimap.create();
		arrayListMultiMap.put("nick", 23);
		arrayListMultiMap.put("nick", 22);
		arrayListMultiMap.put("ken", 56);
		arrayListMultiMap.put("ken", 58);
		arrayListMultiMap.put("david", 78);
		arrayListMultiMap.put("", 78);
		arrayListMultiMap.put(null, 78);
		for (String demo : arrayListMultiMap.keySet()) {
			System.out.println(demo + "\t" + arrayListMultiMap.get(demo));
		}
		
		
	   System.out.println("\n" + "TreeMap:");
		TreeMap<String, Integer> treeMap = new TreeMap<String, Integer>();
		treeMap.put("nick", 23);
		treeMap.put("nick", 22);
		treeMap.put("david", 56);
		treeMap.put("david", 58);
		treeMap.put("ken", 78);
		treeMap.put("aaa", 23);
		treeMap.put("leo", 34);
		treeMap.put("luna", 87);
		treeMap.put("", 78);
		// treeMap.put(null, 78); //key不能为null
		for (String demo : treeMap.keySet()) {
			System.out.println(demo + "\t" + treeMap.get(demo));
		}

		System.out.println("\n" + "TreeMultimap:");
		TreeMultimap<String, Integer> treeMultimap = TreeMultimap.create();
		treeMultimap.put("nick", 23);
		treeMultimap.put("nick", 22);
		treeMultimap.put("david", 56);
		treeMultimap.put("david", 58);
		treeMultimap.put("ken", 78);
		treeMultimap.put("aaa", 23);
		treeMultimap.put("leo", 34);
		treeMultimap.put("luna", 87);
		treeMultimap.put("", 78);
		// treeMultimap.put(null, 78); //key不能为null
		for (String demo : treeMultimap.keySet()) {
			System.out.println(demo + "\t" + treeMultimap.get(demo));
		}
		
```

我们可以看出区别，HashMultimap&#60;String, Integer&#62; 其实类似于，HashMap&#60;String,Set&#60;Integer&#62;&#62;的数据结构。


guava为我们提供了如下MultiMap类型集合：

| 实现            | 键行为类似            | 值行为类似  | 是否可为null |是否线程安全| 含义         |
| ------------------|:-----------------------:| :-----------:| :-----------:|------------| ---------    |
| ArrayListMultimap | HashMap            | ArrayList         | Y |N           |hash表map      |
| HashMultimap      | HashMap            | HashSet           | Y |N           |hash表map   |
| LinkedListMultimap   | LinkedHashMap      | LinkedList      | Y   |N           |链表方式实现的map |
| LinkedHashMultimap | LinkedHashMap  | LinkedHashMap         | Y |N          | 链表方式实现的map |
| TreeMultimap  | TreeMap       | TreeSet         |Y |N    | 具有排序的map |
| ImmutableListMultimap  | ImmutableMap(guava)     | ImmutableList |       N    |Y    | 不可变集合 |
| ImmutableSetMultimap  | ImmutableMap(guava)      | ImmutableSet |     N     |Y    | 不可变集合 |