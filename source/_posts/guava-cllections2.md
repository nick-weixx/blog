---
title: guava 新集合类型2
date: 2017-09-22 20:02:15
tags:
    - frame
type: "categories"
---

## Table
Guava为此提供了新集合类型Table，它有两个支持所有类型的键：”行”和”列”。替代了类似Map&#60;String, Map&#60;String,Integer&#62;&#62;的实现，这种方式很丑陋,使用不友好的数据结构。

举个例子：

```
      System.out.println("java hashmap...\n");
		Map<String, Map<String, Integer>> t1 = new HashMap<String, Map<String, Integer>>();
		Map<String, Integer> nick = new HashMap<String, Integer>();
		nick.put("math", 100);
		nick.put("chinese", 99);
		nick.put("en", 98);
		nick.put("en", 100);
		Map<String, Integer> david = new HashMap<String, Integer>();
		david.put("math", 101);
		david.put("chinese", 102);
		david.put("en", 102);
		t1.put("nick", nick);
		t1.put("david", david);
		System.out.println("row: " + t1.get("nick"));

		System.out.println("\n guava HashBasedTable...\n");
		Table<String, String, Integer> table = HashBasedTable.create();
		table.put("nick", "math", 100);
		table.put("nick", "chinese", 99);
		table.put("nick", "en", 98);
		table.put("nick", "en", 100);
		table.put("david", "math", 101);
		table.put("david", "chinese", 102);
		table.put("david", "en", 102);
		System.out.println("all col: " + table.columnMap());
		System.out.println("rowkeys: " + table.rowKeySet());
		System.out.println("vals: " + table.values());
		System.out.println("col: " + table.column("chinese"));
		System.out.println("row: " + table.row("nick"));
		System.out.println("is contains column:[math] " +  table.containsColumn("math"));
		System.out.println("is contains row:[nick] " + table.containsRow("nick"));
		
		//只读表
		System.out.println("\n guava ImmutableTable...\n");
		Table<String, String, Integer> table = HashBasedTable.create();
		table.put("nick", "math", 100);
		table.put("nick", "chinese", 99);
		table.put("nick", "en", 98);
		table.put("nick", "en", 100);
		table.put("david", "math", 101);
		table.put("david", "chinese", 102);
		table.put("david", "en", 102);
		ImmutableTable<String, String, Integer> copyImutable = ImmutableTable.copyOf(table);
		System.out.println("all col: " + copyImutable.columnMap());
		System.out.println("rowkeys: " + copyImutable.rowKeySet());
		System.out.println("vals: " + copyImutable.values());
		System.out.println("col: " + copyImutable.column("chinese"));
		System.out.println("row: " + copyImutable.row("nick"));
		System.out.println("is contains column:[math] " + copyImutable.containsColumn("math"));
		System.out.println("is contains row:[nick] " + copyImutable.containsRow("nick"));
		
		//二维数组
		Table<String, String, Integer> table = HashBasedTable.create();
		table.put("nick", "math", 100);
		table.put("nick", "chinese", 99);
		table.put("nick", "en", 98);
		table.put("nick", "en", 100);
		table.put("david", "math", 101);
		table.put("david", "chinese", 102);
		table.put("david", "en", 102);
		ArrayTable<String, String, Integer> arrTab = ArrayTable.create(table);
		System.out.println("row key: " + arrTab.rowKeyList());
		System.out.println("column key: " + arrTab.columnKeyList());
		
```

-  table.row("nick")： 用Map&#60;C, V&#62;返回给定”行”的所有列，对这个map进行的写操作也将写入Table中。
-  table.rowKeySet()：返回所有行键，Set&#60;R&#62; 格式
 
- cellSet()：用元素类型为Table.Cell&#60;R, C, V&#62;的Set表现Table&#60;R, C, V&#62;。Cell类似于Map.Entry，但它是用行和列两个键区分的

guava为我们提供了如下Multiset类型集合：

| 实现(TABLE)           | 是否重复值插入           | 是否支持null  | 是否线程安全 | 含义         |
| ------------------|:-----------------------:| -----------:|------------| ---------    |
| HashBasedTable       | Y            | Y           |N           |HashMap&#60;R, HashMap&#60;C, V&#62;&#62;实现      |
| TreeBasedTable           | Y            | Y           |N          |TreeMap&#60;R, TreeMap&#60;C,V&#62;&#62;实现   |
| ArrayTable     | Y      | Y           |N          |二维数组实现，与其他TABLE工作原理不同 |
| ImmutableTable     | N      | N          |Y          |只读table实现,ImmutableMap&#60;R, ImmutableMap&#60;C, V&#62;&#62; |



## RangeSet
RangeSet描述了一组不相连的、非空的区间。当把一个区间添加到可变的RangeSet时，所有相连的区间会被合并，空区间会被忽略。例如：

```
	   RangeSet<Integer> rangeSet = TreeRangeSet.create();
		rangeSet.add(Range.closed(1, 10));//[[1‥10]]  1<=x<=10
		System.out.println(rangeSet);
		rangeSet.add(Range.closedOpen(11, 15));//[[1‥10], [11‥15)] 11<=x<15
		System.out.println(rangeSet);
		rangeSet.add(Range.open(15, 20));//[[1‥10], [11‥15), (15‥20)] 15<x<20
		System.out.println(rangeSet);
		rangeSet.add(Range.openClosed(0, 0));//[[1‥10], [11‥15), (15‥20)] 空区间 0<x<0
		System.out.println(rangeSet);
		rangeSet.remove(Range.open(5, 10));// [[1‥5], [10‥10], [11‥15), (15‥20)] 移除区间 5<x<9
		System.out.println(rangeSet);
		//互补区间
		System.out.println("补集: "+rangeSet.complement());

		// 是否在区间内
		System.out.println("单点21是否在区间内：" + rangeSet.contains(21));// false
		System.out.println("单点9是否在区间内：" + rangeSet.contains(9));// true
		// 元素所在区间
		System.out.println("元素15所在区间：" + rangeSet.rangeContaining(15));
		System.out.println("元素7所在区间：" + rangeSet.rangeContaining(7));

		// 范围是否在区间内
		System.out.println("定义区间是否在区间内" + rangeSet.encloses(Range.closedOpen(18, 20)));
```


-  complement()：返回RangeSet的补集视图。complement也是RangeSet类型,包含了不相连的、非空的区间。
-  subRangeSet(Range&#60;C&#62;)：返回RangeSet与给定Range的交集视图。这扩展了传统排序集合中的headSet、subSet和tailSet操作。
-  asRanges()：用Set&#60;Range&#60;C&#62;&#62;表现RangeSet，这样可以遍历其中的Range。
-  asSet(DiscreteDomain&#60;C&#62;)（仅ImmutableRangeSet支持）：用ImmutableSortedSet&#60;C&#62;表现RangeSet，以区间中所有元素的形式而不是区间本身的形式查看。（这个操作不支持DiscreteDomain 和RangeSet都没有上边界，或都没有下边界的情况）


## RangeMap

RangeMap描述了”不相交的、非空的区间”到特定值的映射。和RangeSet不同，RangeMap不会合并相邻的映射，即便相邻的区间映射到相同的值。例如：

```
	RangeMap<Integer, String> rangeMap = TreeRangeMap.create();
	rangeMap.put(Range.closed(1, 10), "foo"); //{[1,10] => "foo"}
	rangeMap.put(Range.open(3, 6), "bar"); //{[1,3] => "foo", (3,6) => "bar", [6,10] => "foo"}
	rangeMap.put(Range.open(10, 20), "foo"); //{[1,3] => "foo", (3,6) => "bar", [6,10] => "foo", (10,20) => "foo"}
	rangeMap.remove(Range.closed(5, 11)); //{[1,3] => "foo", (3,5) => "bar", (11,20) => "foo"}

```

- asMapOfRanges()：用Map&#60;Range&#60;K&#62;, V&#62;表现RangeMap。这可以用来遍历RangeMap。
- subRangeMap(Range&#60;K&#62;)：用RangeMap类型返回RangeMap与给定Range的交集视图。这扩展了传统的headMap、subMap和tailMap操作。