---
title: flume源码分析1
date: 2017-12-24 12:40:03
tags:
- frame
type: "categories"
---


### 入口类Application解析

#### 1.1 类图
![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/flume_1.png)


. 1. components 对象，存储了解析flume-conf.properties 的实现方法

. 2. supervisor 对象,生命周期管理者，管理所有继承与LifecycleAware接口的所有服务的启停及状态监控

. 3. materializedConfiguration对象，存储了flume传入的配置信息。启动source,channel,sink时负责传入参数

. 4. monitorServer 对象，用与启动http监控或则Ganglia监控。监控服务端信息

. 5. main(String[] args) 主程序方法，命令参数校验，加载配置文件，加载监控等。

. 6. start() 启动配置文件加载类

. 7. stop() 停止monitorServer 监控和supervisor管理服务.

. 8. main() 观察者方法，用于启动source,channel,sink服务


#### 1.3 mian方法

. 使用了 commons-cli.jar 中的Options，校验入参的完整性。及解析参数信息。

```
main(){....

      Options options = new Options();
      //具有参数
      Option option = new Option("n", "name", true, "the name of this agent");
      //强制参数，必须得有
      option.setRequired(true);
      options.addOption(option);

      option = new Option("f", "conf-file", true,
          "specify a config file (required if -z missing)");
      option.setRequired(false);
      options.addOption(option);
      ...
      ...
      option = new Option("h", "help", false, "display help text");
      options.addOption(option);
      
      CommandLineParser parser = new GnuParser();
      CommandLine commandLine = parser.parse(options, args);
      
      String agentName = commandLine.getOptionValue('n');
      File configurationFile = new File(commandLine.getOptionValue('f'));

```

Option(String opt, String longOpt, boolean hasArg, String description) 
 参数名称，完整名称，是否接受参数，参数描述。 比如命令n ，具有参数且必须传入参数.如果步传入这个参数，则直接抛出异常。
 
 命令h ，不具有参数，只提供帮助信息打印


 根据配置文件解析程序

. 参数模拟：-f /Users/playcrab/dev/java_code/ws3/flume/conf/flume-conf.properties -n pagent

flume-conf.properties内容 

```
pagent.sources = file_source
pagent.channels = file_channel
pagent.sinks = kafka_sink

......
```

加载配置文件解析方法

```
main(){....
          EventBus eventBus = new EventBus(agentName + "-event-bus");
          PollingPropertiesFileConfigurationProvider configurationProvider =
            new PollingPropertiesFileConfigurationProvider(
              agentName, configurationFile, eventBus, 30);
          components.add(configurationProvider);
          application = new Application(components);
          //订阅事件
          eventBus.register(application);
```

观察者方法

```
@Subscribe
  public synchronized void handleConfigurationEvent(MaterializedConfiguration conf) {
    stopAllComponents();
    startAllComponents(conf);
  }
  
```

被观察者 PollingPropertiesFileConfigurationProvider，中的FileWatcherRunnable线程

```
 run()...
      eventBus.post(getConfiguration());
  
```

. 使用监听模式，guava 的EventBus，被观察者通知观察者是否已解析完配置文件。如果解析完成，观察者运行handleConfigurationEvent()方法，启动所有的channel,source,sink

### PollingPropertiesFileConfigurationProvider 配置文件解析线程

#### 2.1 类图

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/flume_2.png)

* 1. 解析flume配置文件，启动线程定期检测配置文件更新情况，并且重新进行加载。

* 2. 注册到生命周期管理，实现线程管理。

#### 2.2 start 方法

```
  @Override
  public void start() {
    LOGGER.info("Configuration provider starting");

    Preconditions.checkState(file != null,
        "The parameter file must not be null");
    //周期任务调度器,启动单线程定时执行任务
    executorService = Executors.newSingleThreadScheduledExecutor(
            new ThreadFactoryBuilder().setNameFormat("conf-file-poller-%d")
                .build());

    FileWatcherRunnable fileWatcherRunnable =
        new FileWatcherRunnable(file, counterGroup);
    // */30 * * * * 秒调度一次,timeunit 时间转化器
    executorService.scheduleWithFixedDelay(fileWatcherRunnable, 0, interval,
        TimeUnit.SECONDS);

    lifecycleState = LifecycleState.START;

    LOGGER.debug("Configuration provider started");
  }
```
重写了接口LifecycleAware的start方法，方法内单独启动一个线程，30秒执行一次。调用FileWatcherRunnable，对配置文件进行解析。

```
  public class FileWatcherRunnable implements Runnable {

    private final File file;
    private final CounterGroup counterGroup;

    private long lastChange;

    public FileWatcherRunnable(File file, CounterGroup counterGroup) {
      super();
      this.file = file;
      this.counterGroup = counterGroup;
      this.lastChange = 0L;
    }

    @Override
    public void run() {
      LOGGER.debug("Checking file:{} for changes", file);
      //检查次数
      counterGroup.incrementAndGet("file.checks");

      long lastModified = file.lastModified();

      if (lastModified > lastChange) {
        LOGGER.info("Reloading configuration file:{}", file);
        //加载次数
        counterGroup.incrementAndGet("file.loads");

        lastChange = lastModified;

        try {
        	//提交消息
          eventBus.post(getConfiguration());
        } catch (Exception e) {
          LOGGER.error("Failed to load configuration data. Exception follows.",
              e);
        } catch (NoClassDefFoundError e) {
          LOGGER.error("Failed to start agent because dependencies were not " +
              "found in classpath. Error follows.", e);
        } catch (Throwable t) {
          // caught because the caller does not handle or log Throwables
          LOGGER.error("Unhandled error", t);
        }
      }
    }
  }
```

这个才是配置文件处理的真实线程，当文件被更新。则重新解析配置文件，并且告诉观察者 Application的 handleConfigurationEvent()方法。重新加载source,channel,sink。

####  2.3 AbstractConfigurationProvider

是PollingPropertiesFileConfigurationProvider 的抽象父类，实现了getConfiguration()方法，解析配置文件。

```
	protected abstract FlumeConfiguration getFlumeConfiguration();
	public MaterializedConfiguration getConfiguration() {
		MaterializedConfiguration conf = new SimpleMaterializedConfiguration();
		FlumeConfiguration fconfig = getFlumeConfiguration();
		....
```


