---
title: guava 观察者模式(监听模式)
date: 2017-12-23 12:12:45
tags:
    - intforce
type: "categories"
---

## 1. 传统观察者模式
这是设计模式中的一种，举一个不恰当的例子，一个老师，盯着3个学生做作业。每当一个孩子做完作业，老师都要去及时的批改作业。

这其中就存在二个对象：

1，观察者对象：就是老师

2，被观察对象(受查者)：被监听的三个倒霉孩子


### 实现：

```
import java.util.Observable;
import java.util.Observer;

public class WatcherCls {
	class TeaCherWatcher implements Observer {

		@Override
		public void update(Observable o, Object arg) {
			StudentWatched b = (StudentWatched) o;
			System.out.println(String.format("%s,写完了作业,老师要进行批改了...", b.call()));
		}
	}

	class StudentWatched extends Observable {
		private String name;

		public StudentWatched(String name) {
			this.name = name;
		}

		public String call() {
			return name;
		}

		public void work(boolean isOk) {
			if (isOk) {//如果作业写完，通知观察者老师
				setChanged();
				notifyObservers();
			}
		}
	}

	public static void main(String[] args) {
		WatcherCls ws = new WatcherCls();
		StudentWatched nick = ws.new StudentWatched("nick");// 受查者
		StudentWatched devid = ws.new StudentWatched("devid");// 受查者
		StudentWatched ken = ws.new StudentWatched("ken");// 受查者

		TeaCherWatcher teatherWatcher = ws.new TeaCherWatcher();// 观察者
		nick.addObserver(teatherWatcher);
		devid.addObserver(teatherWatcher);
		ken.addObserver(teatherWatcher);
		nick.work(true);
		devid.work(false);
		ken.work(true);

	}
}
```

* 观察者(theather)重写Observer接口，被观察者(student) 继承Observable。

* 触发notifyObservers 父类方法通知观察者。setChanged方法，表明内容有变化，不调用则不会通知。为开关方法。

## 2. 监听模式

设计模式中的一种，还是已上面那个例子为准。老师盯着三个孩子写作业，每当孩子完成作业都要通知老师来检查。

定义：事件源经过事件的封装传给监听器，当事件源触发事件后，监听器接收到事件对象可以回调事件的方法


我们拆分一下对象，存在3个对象

1，事件源：老师

2，监听器：老师监听作业完成情况

3， 事件：学生

###实现：

```
import java.util.Enumeration;
import java.util.Vector;

public class EventCls2 {
	public class TeacherSource {
		private Vector<TeacherListener> repository = new Vector<TeacherListener>();// 监听自己的监听器队列

		public TeacherSource() {
		}

		public void addListener(TeacherListener dl) {
			repository.addElement(dl);
		}

		public void notifyEvent() {// 通知所有的监听器
			Enumeration<TeacherListener> em = repository.elements();
			while (em.hasMoreElements()) {
				TeacherListener dl = (TeacherListener) em.nextElement();
				dl.work(new StudentEvent(this));
			}
		}
	}

	public class StudentEvent extends java.util.EventObject {
		private static final long serialVersionUID = 1L;
		boolean isOk = false;

		public StudentEvent(Object source) {
			super(source);		}

		public void say(String name, boolean isok) {
			String rs = isok ? name + "作业写完了。。" : name + "作业没写完。。";
			isOk = isok;
			System.out.println(rs);
		}
	}

	public interface TeacherListener extends java.util.EventListener {
		public void work(StudentEvent dm);
	}

	public static void main(String[] args) {
		EventCls2 ec = new EventCls2();
		TeacherSource stSource = ec.new TeacherSource();
		stSource.addListener(new TeacherListener() {
			String name = "nick";

			@Override
			public void work(StudentEvent dm) {
				dm.say(name, false);
			}
		});
		stSource.addListener(new TeacherListener() {
			String name = "devide";

			@Override
			public void work(StudentEvent dm) {
				dm.say(name, true);
			}
		});
		stSource.addListener(new TeacherListener() {
			String name = "ken";

			@Override
			public void work(StudentEvent dm) {
				dm.say(name, false);
			}
		});
		stSource.notifyEvent();// 触发事件、通知监听器

	}
}
```

监听器模式是观察者模式的另一种形态，同样基于事件驱动模型。监听器模式更加灵活，可以对不同事件作出相应。但也是付出了系统的复杂性作为代价的，因为我们要为每一个事件源定制一个监听器以及事件，这会增加系统的负担。 监听器可以实现多个时间的监控，及抽象扩展性非常的灵活。

##guava 中的观察者模式

### 1.简单方法

还是一样的定义，我们实现老师监控学生写作业。

```
import java.util.HashSet;
import java.util.Set;

import com.google.common.eventbus.EventBus;
import com.google.common.eventbus.Subscribe;

public class EventCls3 {
	public class StudentEvent {
		String name;
		boolean isok;

		public StudentEvent(String name, boolean isok) {
			this.name = name;
			this.isok = isok;
		}

		public boolean getStatus() {
			return isok;
		}

		public String getName() {
			return name;
		}
	}

	public class TeacherListener {
		Set<String> rs;

		public TeacherListener() {
			rs = new HashSet<String>();
		}

		@Subscribe
		public void listen(StudentEvent event) {
			String say = event.getStatus() ? event.getName() + ", 完成了作业.." : event.getName() + "没完成作业";
			rs.add(say);
		}

		public Set<String> rs() {
			return rs;
		}
	}

	public static void main(String[] args) {
		EventCls3 ec = new EventCls3();

		TeacherListener listener = ec.new TeacherListener();//观察者
		EventBus eventBus = new EventBus();
		//注册监听器
		eventBus.register(listener);

		StudentEvent nickEvent = ec.new StudentEvent("nick", true);//受查者
		StudentEvent devidEvent = ec.new StudentEvent("devid", false);//受查者
		StudentEvent kenEvent = ec.new StudentEvent("ken", true);//受查者
		//提交监听事件,并触发监听方法
		eventBus.post(nickEvent);
		eventBus.post(devidEvent);
		eventBus.post(kenEvent);
		//遍历返回信息
		for (String say : listener.rs()) {
			System.out.println(say);
		}
	}
}

```
1，相对于jdk中提供的观察者方法，guava中不需要继承接口，没有代码侵入性。干净且利于理解。
2，guava 使用注解@Subscribe可以自定义监听器的监听方法，可以同时实现多个事件的监听。




### 2, Event的继承

如果Listener A监听Event A, 而Event A有一个子类Event B, 此时Listener A将同时接收Event A和B消息，实例如下：

```
import java.util.HashSet;
import java.util.Set;

import com.google.common.eventbus.EventBus;
import com.google.common.eventbus.Subscribe;

public class EventCls3 {

	public class ParentEvent {
		protected String name;
		protected boolean isok;

		public ParentEvent(String name, boolean isok) {
			this.name = name;
			this.isok = isok;
		}

		public boolean getStatus() {
			return isok;
		}

		public String getName() {
			return name;
		}

		public String parentName() {
			return name;
		}
	}

	public class StudentEvent extends ParentEvent {

		public StudentEvent(String name, boolean isok) {
			super(name, isok);
		}

		@Override
		public String getName() {
			return super.getName() + "的儿子";
		}

	}

	public class TeacherListener {
		Set<String> rs;

		public TeacherListener() {
			rs = new HashSet<String>();
		}

		@Subscribe
		public void listenStudent(StudentEvent event) {
			String say = event.getStatus() ? event.getName() + ", 完成了作业.." : event.getName() + " ,没完成作业";
			rs.add(say);
		}

		@Subscribe
		public void listenParent(StudentEvent event) {
			String say = event.getStatus() ? event.parentName() + ", 你儿子没完成作业.." : event.parentName() + " ,你儿子没完成作业";
			rs.add(say);
		}

		public Set<String> rs() {
			return rs;
		}
	}

	public class TeacherListenerParent {

		Set<String> rs;

		public TeacherListenerParent() {
			rs = new HashSet<String>();
		}

		@Subscribe
		public void listenParent(StudentEvent event) {
			String say = event.getStatus() ? event.parentName() + ", 你儿子完成了作业" : event.parentName() + " ,你儿子没完成作业，来学校一趟";
			rs.add(say);
		}

		public Set<String> rs() {
			return rs;
		}
	}

	public static void main(String[] args) {
		EventCls3 ec = new EventCls3();

		TeacherListener listener = ec.new TeacherListener();
//		TeacherListenerParent listenerParent = ec.new TeacherListenerParent();

		EventBus eventBus = new EventBus("test");
		eventBus.register(listener);
//		eventBus.register(listenerParent);

		StudentEvent nickEvent = ec.new StudentEvent("nick", true);
		StudentEvent devidEvent = ec.new StudentEvent("devid", false);
		StudentEvent kenEvent = ec.new StudentEvent("ken", true);
		eventBus.post(nickEvent);
		eventBus.post(devidEvent);
		eventBus.post(kenEvent);
		for (String say : listener.rs()) {
			System.out.println(say);
		}
//		System.out.println("send parent.......");
//		for (String say : listenerParent.rs()) {
//			System.out.println(say);
//		}
	}
}

```

就写到这里，基本操作就是这样。这个设计模式在很多地方都可以用到，想到了一句话叫暗中观察。。。

