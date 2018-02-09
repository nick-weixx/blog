---
title: springboot mybatis集成及dubbo集成
date: 2018-02-08 20:42:03
tags:
    - frame
type: "categories"
---

这里创建了一个小demo，沿用上期blog需求。但是底层查询数据的架构使用了mybatis，同时使用dubbo 实现了 provider与cusumer。 provider 用于blog文章的数据提供，cusumer 消费provider并且通过restful 形式的接口提供给前端调用。

## 一. mybatis 集成

### 1. maven依赖

添加 mybatis-spring-boot-starter 与 com.alibaba连接池

```

<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>1.3.1</version>
</dependency>
	<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>druid</artifactId>
			<version>1.0.18</version>
	</dependency>

```

### 2. 编写 POJO对象及实现数据访问层访问

#### (1) POJO对象

和之前的hibernate 对象没有区别，只是将注解去掉了。

```
public class Article implements Serializable {
	private static final long serialVersionUID = -1491292595468157592L;
	// 标明自增id，及列名
	private int articleId;

	// 标明文章标题
	private String title;

	// 标明文章类别
	private String category;

	public int getArticleId() {
		return articleId;
	}

	public void setArticleId(int articleId) {
		this.articleId = articleId;
	}

	public String getTitle() {
		return title;
	}

	public void setTitle(String title) {
		this.title = title;
	}

	public String getCategory() {
		return category;
	}

	public void setCategory(String category) {
		this.category = category;
	}

	@Override
	public String toString() {
		return "Article [articleId=" + articleId + ", title=" + title + ", category=" + category + "]";
	}

}
```

#### (2) 数据访问层实现


```
@Mapper
public interface IArticleDao {

	@Select("SELECT article_id,title,category FROM articles")
	@Results({ @Result(property = "articleId", column = "article_id", javaType = Integer.class),
			@Result(property = "title", column = "title"), @Result(property = "category", column = "category")

	})
	List<Article> getAllArticles();
	
	@Select("SELECT article_id,title,category FROM articles where article_id=#{articleId}")
	@Results({ @Result(property = "articleId", column = "article_id", javaType = Integer.class),
			@Result(property = "title", column = "title"), @Result(property = "category", column = "category")
	})
	Article getArticleById(int articleId);
	
	@Insert("INSERT INTO articles(title,category) VALUES(#{title},#{category})")
	int addArticle(Article article);
	@Update("UPDATE articles SET category=#{category} WHERE article_id=#{articleId}")
	void updateArticle(Article article);
	
	@Delete("DELETE FROM articles WHERE article_id=#{articleId}")
	void deleteArticle(@Param("articleId") int articleId);
	
	@Select("SELECT article_id FROM articles WHERE title=#{title} AND category=#{category}")
	Integer articleExists(@Param("title") String title, @Param("category") String category);
}
```

* 全部使用注解完成，接口上使用@Mapper 标明为访问层mapper代码实现
* 通过@Result 注解，标明映射关系
* @SELECT 注解标明为查询语句，@DELETE 标明为删除语句，@INSERT 标明为插入语句

#### (3) hibernate vs mybatis

因为我们做了分层，下面的数据库交互框架的变动并不影响业务层和mvc的使用。所以上层并不需要修改。到此访问层架构的提供已完成。非常简单。

这里多说几句，hibernate与mybatis都用过，这两种框架，各有各的优劣，没有绝对的好与坏。选型需要很多因素的评估，团队的整体水平，程序开发周期时长，底层使用的数据库，使用的场景，对性能的要求都是制约的条件。


##### Mybatis优势

* MyBatis可以进行更为细致的SQL优化，可以减少查询字段。
* MyBatis容易掌握，而Hibernate门槛较高。
* 轻量级，可控性高。

#####  Hibernate优势

* Hibernate的DAO层开发比MyBatis简单，Mybatis需要维护SQL和结果映射。
* Hibernate对对象的维护和缓存要比MyBatis好，对增删改查的对象的维护要方便。
* Hibernate数据库移植性很好，MyBatis的数据库移植性不好，不同的数据库需要写不同SQL。
* Hibernate有更好的二级缓存机制，可以使用第三方缓存。MyBatis本身提供的缓存机制不佳。

感兴趣的可以看一下：https://www.zhihu.com/question/21104468


## 二. provider 开发

### 1. maven引入


```

<dependency>
	<groupId>io.dubbo.springboot</groupId>
	<artifactId>spring-boot-starter-dubbo</artifactId>
	<version>1.0.0</version>
</dependency>

```

### 2. provider 实现

```

@Service(version = "1.0.0")
public class ArticleDubboServiceImpl implements IArticleDubboService {
	@Autowired
	private IArticleService articleService;

	@Override
	public Article getArticleById(Integer id) {
		return articleService.getArticleById(id);
	}

}

```

* 使用注解的方式标明服务版本，@Service(version = 1.0.0)
* 调用之前实现的服务层代码

### 3. 文件配置

application.properties 文件汇中配置

```

server.port=9999

spring.datasource.url=jdbc:mysql://xxxxxxx:3306/spring_boot
spring.datasource.username=xxxx
spring.datasource.password=xxxx
spring.datasource.driver-class-name=com.mysql.jdbc.Driver

spring.dubbo.application.name=blog_provider
spring.dubbo.registry.address=zookeeper://127.0.0.1:2181
spring.dubbo.protocol.name=dubbo
spring.dubbo.protocol.port=20880
spring.dubbo.scan=com.nick.demo2.dubbo

```


### 4. 项目结构图

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/springboot-dubbo-1.jpg)

* com.nick.demo2.entity POJO实现
* com.nick.demo2.dao  数据访问层实现
* com.nick.demo2.service 业务层数显
* com.nick.demo2.controller 控制器实现
* com.nick.demo2.dubbo dubbo provider 实现


### 4. 注册服务

运行程序的 Application，将服务注册到zookeeper

效果图：

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/springboot-dubbo-2.png)

## 三，编写消费者


### 1. 结构图

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/springboot-dubbo-3.png)

* com.nick.dubbo.consumer.controller restful 风格控制器实现
* com.nick.demo2.entity 接口传输的对象
* com.nick.demo2.dubbo 消费者实现

### 2. 消费者实现

```

@Component
public class DubboService {
	@Reference(version = "1.0.0")
	public IArticleDubboService articleDubbo;
}

```

* 非常简单指定消费接口就行

* <font color='red'> 注意：</font> 
消费者中用到的接口包路径及model必须与，服务提供者一致。其实很好理解，因为要序列化传输，所以两边的对象必须一致。所以 才会出现那种不太正常的包结构(com.nick.demo2.dubbo)。 这个可以单独抽取成一个jar包，供两边实现。很简单。

### 3。 配置

application.properties 文件汇中配置

```
server.port=9998

spring.dubbo.application.name=consumer
spring.dubbo.registry.address=zookeeper://127.0.0.1:2181
spring.dubbo.scan=com.nick.demo2.dubbo

```


### 4. 启动消费者及演示

启动 CosumerApplication类

#### 效果图

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/springboot-dubbo-4.png)

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/springboot-dubbo-5.png)


dubbo配置详细说明：

官网： http://dubbo.io/books/dubbo-user-book/preface/background.html


他每做一件小事的时候，都好像抓住一根救命稻草，到最后你才发现，他抱住的已经是一棵参天大树了。--《士兵突击》
