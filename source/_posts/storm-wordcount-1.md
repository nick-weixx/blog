---
title: storm wordcount样例详解
date: 2017-07-27 18:08:20
tags:
    - frame
type: "categories"
---
最近在研究storm，一点点记录希望可以对工作有所帮助。

## 1,storm分组介绍
1, shuffleGrouping

将流分组定义为混排。这种混排分组意味着来自Spout的输入将混排，或随机分发给此Bolt中的任务。shuffle grouping对各个task的tuple分配的比较均匀。

2, fieldsGrouping

这种grouping机制保证相同field值的tuple会去同一个task，这对于WordCount来说非常关键，如果同一个单词不去同一个task，那么统计出来的单词次数就不对了。

3, All grouping

广播发送， 对于每一个tuple将会复制到每一个bolt中处理。

4, Global grouping

Stream中的所有的tuple都会发送给同一个bolt任务处理，所有的tuple将会发送给拥有最小task_id的bolt任务处理。

5, None grouping

不关注并行处理负载均衡策略时使用该方式，目前等同于shuffle grouping,另外storm将会把bolt任务和他的上游提供数据的任务安排在同一个线程下。

6, Direct grouping

由tuple的发射单元直接决定tuple将发射给那个bolt，一般情况下是由接收tuple的bolt决定接收哪个bolt发射的Tuple。这是一种比较特别的分组方法，用这种分组意味着消息的发送者指定由消息接收者的哪个task处理这个消息。 只有被声明为Direct Stream的消息流可以声明这种分组方法。而且这种消息tuple必须使用emitDirect方法来发射。消息处理者可以通过TopologyContext来获取处理它的消息的taskid (OutputCollector.emit方法也会返回taskid)

## 2, wordcount 事例分析

### 2.1 创建spout 模拟数据发送

```
import org.apache.storm.spout.SpoutOutputCollector;
import org.apache.storm.task.TopologyContext;
import org.apache.storm.topology.OutputFieldsDeclarer;
import org.apache.storm.topology.base.BaseRichSpout;
import org.apache.storm.tuple.Fields;
import org.apache.storm.tuple.Values;
import org.apache.storm.utils.Utils;

public class RandomSentenceSpout extends BaseRichSpout {
  SpoutOutputCollector _collector;
  Random _rand;


  @Override
  public void open(Map conf, TopologyContext context, SpoutOutputCollector collector) {
    _collector = collector;
    _rand = new Random();
  }

  @Override
  public void nextTuple() {
    Utils.sleep(100);
    String[] sentences = new String[]{ "the cow jumped over the moon", "an apple a day keeps the doctor away",
        "four score and seven years ago", "snow white and the seven dwarfs", "i am at two with nature" };
    String sentence = sentences[_rand.nextInt(sentences.length)];
    _collector.emit(new Values(sentence));
  }

  @Override
  public void ack(Object id) {
  }

  @Override
  public void fail(Object id) {
  }

  @Override
  public void declareOutputFields(OutputFieldsDeclarer declarer) {
    declarer.declare(new Fields("word"));
  }

}
```
说明： 必须继承BaseRichSpout类,重写方法定义
    
 1，open()方法，既初始化方法，会传入storm的配置信息，输出类SpoutOutputCollector.我们在这个方法中初始化随机数Random类，接收SpoutOutputCollector.
 
 2, nextTuple() 方法，既写入tuple的数据流方法，这里面我们随机取字符串，发射数据。如果要确保数据处理是否成功，需要传送一个msgId,例:collector.emit(values, msgId),msgId需要自己定义。
 
 3，ack(Object id)方法,既锚定方法.如果数据被处理成功会调用此方法。未处理
 
 4,fail(Object id)方法，如果数据处理失败,则会调用fail方法。未处理
 
 5, declareOutputFields 方法，用于描述输出流的数据名称定义。
 
### 2.2 创建SplitSentence类，分割输入信息
 
```
import org.apache.storm.Config;
import org.apache.storm.LocalCluster;
import org.apache.storm.StormSubmitter;
import org.apache.storm.task.ShellBolt;
import org.apache.storm.topology.BasicOutputCollector;
import org.apache.storm.topology.IRichBolt;
import org.apache.storm.topology.OutputFieldsDeclarer;
import org.apache.storm.topology.TopologyBuilder;
import org.apache.storm.topology.base.BaseBasicBolt;
import org.apache.storm.tuple.Fields;
import org.apache.storm.tuple.Tuple;
import org.apache.storm.tuple.Values;
import org.apache.storm.starter.spout.RandomSentenceSpout;

 public class SplitSentence extends BaseBasicBolt {

		@Override
		public void declareOutputFields(OutputFieldsDeclarer declarer) {
			declarer.declare(new Fields("word"));
		}

		@Override
		public Map<String, Object> getComponentConfiguration() {
			Config conf = new Config();
			conf.put(conf.TOPOLOGY_TICK_TUPLE_FREQ_SECS, 10);
			return conf;

		}

		@Override
		public void execute(Tuple input, BasicOutputCollector collector) {
			// TODO Auto-generated method stub
			if (input.getSourceComponent()
					.equals(Constants.SYSTEM_COMPONENT_ID)
					&& input.getSourceStreamId().equals(
							Constants.SYSTEM_TICK_STREAM_ID)) {
				System.out.println("每10s，执行定时器");
			}
			String[] val = input.getString(0).split(" ");
			for (int i = 0; i < val.length; i++) {
				collector.emit(new Values(val[i]));
			}
		}
	} 
```

说明：必须继承BaseBasicBolt类，重写其方法

1, execute(Tuple input, BasicOutputCollector collector)方法，在这里编写输出流逻辑。我们直接在这里切割了语句，将单次作为一行数据发送出去。

2, declareOutputFields(OutputFieldsDeclarer declarer)方法，定义输出数据的名称。

3, getComponentConfiguration方法，可以设置对于单个bolt 的配置。例:
bolt 每个10秒执行一次,可以用于清理缓存信息等.

### 2.3 创建WordCountBolt类，统计单次个数

```
import java.util.HashMap;
import java.util.Map;

import org.apache.storm.task.OutputCollector;
import org.apache.storm.task.TopologyContext;
import org.apache.storm.topology.OutputFieldsDeclarer;
import org.apache.storm.topology.base.BaseRichBolt;
import org.apache.storm.tuple.Fields;
import org.apache.storm.tuple.Tuple;
import org.apache.storm.tuple.Values;

public class WordCountBolt extends BaseRichBolt {

	private OutputCollector collector;
	private Map<String, Long> counts = null;
	/**
	 * 
	 */
	private static final long serialVersionUID = 1000003L;

	public void prepare(Map stormConf, TopologyContext context,
			OutputCollector collector) {
		this.collector = collector;
		counts = new HashMap<String, Long>();
	}

	public void execute(Tuple input) {
		String word = input.getStringByField("word");
		Long count = this.counts.get(word);
		if (count == null) {
			count = 0L;
		}
		count++;
		this.counts.put(word, count);
		this.collector.emit(new Values(word, count));
	}

	public void declareOutputFields(OutputFieldsDeclarer declarer) {
		declarer.declare(new Fields("word", "count"));
	}

}
```
说明：

BaseRichBolt与BaseBasicBolt的区别，下次再说(TODO)

1,通过counts 统计每个单次的出现次数,输出两列，分别为 (单词，出现数量)


### 2.4 创建ReportBolt类，打印报告

```
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.apache.storm.task.OutputCollector;
import org.apache.storm.task.TopologyContext;
import org.apache.storm.topology.OutputFieldsDeclarer;
import org.apache.storm.topology.base.BaseRichBolt;
import org.apache.storm.tuple.Tuple;

public class ReportBolt extends BaseRichBolt {

	/**
	 * 
	 */
	private static final long serialVersionUID = 100000004L;
	private Map<String, Long> counts;

	public void prepare(Map stormConf, TopologyContext context,
			OutputCollector collector) {
		this.counts = new HashMap<String, Long>();
	}

	public void execute(Tuple input) {
		String word=input.getStringByField("word");
		Long count =input.getLongByField("count");
		this.counts.put(word, count);
	}

	public void declareOutputFields(OutputFieldsDeclarer declarer) {
		
	}

	@Override
	public void cleanup() {
		System.out.println("---FINAL COUNTS---");
		List<String> keys =new ArrayList<String>();
		keys.addAll(counts.keySet());
		Collections.sort(keys);
		for (String key : keys) {
			System.out.println(key+" : "+this.counts.get(key));
		}
		System.out.println("-----------------");
	}
	
	

}

```

说明：

1，execute方法，缓存所有单词信息
2，cleanup方法，整个程序结束时调用的清理方法，我们在这里打印统计信息


### 2.5 创建WordCountTopology类，串联任务

```
import org.apache.storm.Config;
import org.apache.storm.LocalCluster;
import org.apache.storm.StormSubmitter;
import org.apache.storm.generated.AlreadyAliveException;
import org.apache.storm.generated.AuthorizationException;
import org.apache.storm.generated.InvalidTopologyException;
import org.apache.storm.generated.StormTopology;
import org.apache.storm.topology.TopologyBuilder;
import org.apache.storm.tuple.Fields;

public class WordCountTopology {
	private static final String SENTENCE_SPOUT_ID = "sentence-spout";
	private static final String SPLIT_BOLT_ID = "split-bolt";
	private static final String COUNT_BOLT_ID = "count-bolt";
	private static final String REPORT_BOLT_ID = "report-bolt";
	private static final String TOPOLOGY_NAME = "word-count-topology";

	public static void main(String[] args) {
		RandomSentenceSpout spout = new RandomSentenceSpout();
		SplitSentenceBolt splitBolt = new SplitSentenceBolt();
		WordCountBolt countBolt = new WordCountBolt();
		ReportBolt reportBolt = new ReportBolt();
		TopologyBuilder builder = new TopologyBuilder();
		builder.setSpout(SENTENCE_SPOUT_ID, spout, 2);

		builder.setBolt(SPLIT_BOLT_ID, splitBolt, 2).setNumTasks(4)
				.shuffleGrouping(SENTENCE_SPOUT_ID);
		//  相同的word发送到同一个 bolt
		builder.setBolt(COUNT_BOLT_ID, countBolt, 4).fieldsGrouping(
				SPLIT_BOLT_ID, new Fields("word"));
		builder.setBolt(REPORT_BOLT_ID, reportBolt).globalGrouping(
				COUNT_BOLT_ID);

		Config config = new Config();
		config.setNumWorkers(2);
		try {
			submit(false, TOPOLOGY_NAME, config, builder.createTopology());
		} catch (AlreadyAliveException e) {
			System.out.println("此任务已存在");
			e.printStackTrace();
		} catch (InvalidTopologyException e) {
			e.printStackTrace();
			System.out.println("初始化错误");
		} catch (AuthorizationException e) {
			e.printStackTrace();
			System.out.println("权限问题");
		}

	}

	private static void debug(String topologyName, Config config,
			StormTopology topology) {
		LocalCluster cluster = new LocalCluster();

		try {
			cluster.submitTopology(topologyName, config, topology);
			Thread.sleep(50 * 1000);
		} catch (Exception e) {
			e.printStackTrace();
		}

		cluster.killTopology(TOPOLOGY_NAME);
		cluster.shutdown();
	}

	private static void submit(boolean isbug, String topologyName,
			Config config, StormTopology topology)
			throws AlreadyAliveException, InvalidTopologyException,
			AuthorizationException {
		StormSubmitter.submitTopology(topologyName, config, topology);

	}
```

说明：

1，创建TopologyBuilder类，串联各个任务

2，设置RandomSentenceSpout spout任务，并行两个Executor(Thread)线程处理

3,设置切割任务SplitSentenceBolt,并行两个Executor(Thread)线程,每个线程处理2个任务。shuffleGrouping 分组，平均分配给各个blot。

4,设置WordCountBolt任务，并行4个Executor(Thread)线程,每个线程处理1个任务。根据字段名称分组(fieldsGrouping)相同的进入一组。

5,设置ReportBolt任务，全局分组(globalGrouping)。所有输出分到一个blot做处理。

6，config.setNumWorkers(2) ，设置两个jvm worker并行。










