---
title: springboot hibernate集成及RESULFUL风格接口
date: 2018-02-04 10:47:54
tags:
    - frame
type: "categories"
---

我们创建一个简单的例子，这个例子中使用springboot+hibernate 实现了与mysql数据库的交互。同时使用RESULFUL风格http接口，实现数据访问展示，使用json格式数据进行交互。

假定的需求场景是博客的文章管理，对图书进行CRUD操作，这是一个非常简单的场景。

## 一，设计数据库

```
CREATE TABLE `articles` (
  `article_id` int(5) NOT NULL AUTO_INCREMENT COMMENT '自增的文章编号',
  `title` varchar(200) NOT NULL DEFAULT '' COMMENT '标题',
  `category` varchar(100) NOT NULL DEFAULT '' COMMENT '类别',
  PRIMARY KEY (`article_id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8;

INSERT INTO `articles` (`article_id`, `title`, `category`) VALUES
	(1, 'Java Concurrency', 'Java'),
	(2, 'Hibernate HQL ', 'Hibernate'),
	(3, 'Spring MVC with Hibernate', 'Spring'); 
```

这张表非常的简单，我们只添加了标题，与文章类别。同事插入了三行数据。


## 二，maven 及项目结构划分

### 1. maven 文件

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.nick</groupId>
	<artifactId>spring-boot-demo1</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>demo-1</name>
	<description>Demo project for Spring Boot</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<relativePath /> <!-- lookup parent from repository -->
	</parent>
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>


</project>

```

配置描述：

* spring-boot-starter-parent 用于依赖管理的父POM
* spring-boot-starter-web：构建Web，REST应用程序的入口。它使用tomcat服务器作为默认的嵌入式服务器。 
* spring-boot-starter-data-jpa：启动Spring数据JPA管理
* spring-boot-devtools：它提供了开发工具,代码热部署等
* spring-boot-maven-plugin：用于创建应用程序的可执行JAR。


### 2. 项目结构

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/springboot-jpa-1.png)

结构描述：

* entity 数据库映射model
* dao  数据访问层实现
* service 业务层实现
* controller 控制器实现

## 二， 数据访问层实现

### 1. 编写entity

我们使用的是注解的方式，配置entity。

```
import java.io.Serializable;
import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Table;
@Entity //标明为Entity
@Table(name="articles") //标明映射的表名
public class Article implements Serializable { 
	private static final long serialVersionUID = 1L;
	@Id
	@GeneratedValue(strategy=GenerationType.AUTO)
	@Column(name="article_id")//1.标明自增id，及列名
        private int articleId;  
	@Column(name="title") //2. 标明文章标题
        private String title;
	@Column(name="category")//3. 标明文章类别
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
}  
```
非常简单,很容易理解。

### 2. 编写访问层实现

```
@Transactional
@Repository
public class ArticleDAOImpl implements IArticleDAO {

	@PersistenceContext	
	private EntityManager entityManager;	
	@Override
	public Article getArticleById(int articleId) {
		return entityManager.find(Article.class, articleId);
	}
	
	/**
	 * 获取所有
	 */
	@SuppressWarnings("unchecked")
	@Override
	public List<Article> getAllArticles() {
		String hql = "FROM Article as atcl ORDER BY atcl.articleId";
		return (List<Article>) entityManager.createQuery(hql).getResultList();
	}	
	
	/**
	 * 添加
	 */
	@Override
	public void addArticle(Article article) {
		entityManager.persist(article);
	}
	.....
```

* 类上面标记了@Transactional 受事务控制，同时标了@Repository 标注数据访问组件。
* EntityManager 上面标@PersistenceContext,实体管理器，用于执行持久化操作。

## 三，业务层与控制器实现

### 1. 业务层实现

```
@Service
public class ArticleService implements IArticleService {

	@Autowired
	private IArticleDAO articleDAO;

	@Override
	public Article getArticleById(int articleId) {
		Article obj = articleDAO.getArticleById(articleId);
		return obj;
	}

	@Override
	public List<Article> getAllArticles() {
		return articleDAO.getAllArticles();
	}

	@Override
	public synchronized boolean addArticle(Article article) {
		if (articleDAO.articleExists(article.getTitle(), article.getCategory())) {
			return false;
		} else {
			articleDAO.addArticle(article);
			return true;
		}
	}

```

* 使用@Service标明，为服务层。
* IArticleDAO 上面标 @Autowired，引入数据层实现


### 2. 控制器实现

```
@Controller
@RequestMapping("user")
public class ArticleController {
	@Autowired
	private IArticleService articleService;

	/**
	 * 获取文章信息,设备get方式
	 * @param id
	 * @return
	 */
	@GetMapping("article/{id}")
	public ResponseEntity<Article> getArticleById(@PathVariable("id") Integer id) {
		Article article = articleService.getArticleById(id);
		return new ResponseEntity<Article>(article, HttpStatus.OK);
	}
	
	/**
	 * 获取所有
	 * @return
	 */
	@GetMapping("articles")
	public ResponseEntity<List<Article>> getAllArticles() {
		List<Article> list = articleService.getAllArticles();
		return new ResponseEntity<List<Article>>(list, HttpStatus.OK);
	}
```

* 类上面@Controller，@RequestMapping("user") 标明为控制器，且mapping为/user
* getArticleById 方法上标明@GetMapping("article/{id}"),既后面的参数id 为查询的文章编号

## 四，application.properties 配置

```
##server
server.port=8888
##db 
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://103.37.163.105:51000/spring_boot
spring.datasource.username=spring
spring.datasource.password=123
spring.datasource.tomcat.max-wait=20000
spring.datasource.tomcat.max-active=50
spring.datasource.tomcat.max-idle=20
spring.datasource.tomcat.min-idle=15
##jpa
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect = org.hibernate.dialect.MySQLDialect
spring.jpa.properties.hibernate.id.new_generator_mappings = false
spring.jpa.properties.hibernate.format_sql = true
##log
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE 
```

## 五，演示

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/springboot-jpa-2.png)

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/springboot-jpa-3.png)


spring boot 帮助文档：https://docs.spring.io/spring-boot/docs/current/reference/html/

demo git地址： https://github.com/nick-weixx/springboot-demo1
