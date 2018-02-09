---
title: 你好，spring boot
date: 2018-01-22 20:02:15
tags:
    - frame
type: "categories"
---

## 一，spring boot 介绍
Spring Boot使您可以轻松创建独立的生产级基于Spring的应用程序，您可以“just run”。我们对Spring平台和第三方库进行了自己的封装，所以你可以非常容易的开始。大多数Spring Boot应用程序只需要很少的Spring配置。

特性：

 *  创建独立的Spring应用程序
 *  直接嵌入Tomcat，Jetty或Undertow（无需部署WAR文件）
 *  提供POM来简化您的Maven配置
 *  尽可能自动配置Spring
 *  提供生产就绪功能，如指标，运行状况检查和外部配置
 *  绝对不会生成代码，也不需要XML配置
 
 
spring boot 配合 spring cloud 非常适合于开发微服务，简单快速，灵活,实现SOA，服务注册发现。
  
 
## 二，spring boot 程序生成

###  1. 工具推荐

spring boot 程序可以通过 <a href="http://write.blog.csdn.net/postlist" target="_blank">Spring Boot CLI</a> 快速的生成原型，也可以直接创建maven项目，引用对应的spring boot 包。但是推荐大家使用<a href "https://spring.io/tools/sts/all" >sts(Spring Tool Sulte)</a>开发工具，里面集成了所有spring boot的程序模板，非常的方便，而且它是在eclipse 的基础上开发而来，容易上手。

### 2. 写一个hello word

#### (1). 创建项目

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/springboot-helloword-1.png)

创建项目名称，配置maven的group id ,artifactId，版本等


 ![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/springboot-helloword-2.png)
 
 定义为web程序
 
  ![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/springboot-helloword-3.png)
 
 生成的项目结构图，是一个标准的maven项目。
 
#### (2). 结构简介

##### 1. pom.xml 文件分析

  
```
   <parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
	</parent>
    <dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web-services</artifactId>
	</dependency>
	<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
	</dependency>
```

* 引用spring boot parent服务
* 引用sprint-boot-xxxx-web-services定义为web程序
* 引用sprint-boot-xxx-test进行测试

##### 2. 生成程序启动入口，配置控制器信息

```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
@SpringBootApplication
public class HellowordDemoApplication {
	@RequestMapping("/")
	@ResponseBody
	String home() {
		return "Hello World!";
	}
	public static void main(String[] args) {
		SpringApplication.run(HellowordDemoApplication.class, args);
	}
}
```
 
##### 3. application.properties 文件，在这里面可以设置程序参数，如启动端口,数据库连接等.


```
  ## 服务访问端口
 server.port=9527
 
```

### 3. 程序效果 

![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/springboot-helloword-4.png)

spring boot 集成了嵌入式的tomcat,jetty 容器，非常方便引用即可用。降低了程序开发的难度。将所有用到的框架，梳理封装成了一个整体。不在需要程序员再像原来一样一点点在maven文件里引用，排包冲突等，是一套完整的解决方案。如果说以前实现程序开发，像是自己组装零件的话，那spring boot 就像是直接给了你n套半成品机械，只需要编写业务即可。好处很多，但是坏处还是有两点的，第一，出现时间太短，在国内也不是那么的流行。第二，封装程度过高，出现底层问题，靠自己排查会非常的困难。但相信经过时间的积累，它会越发的稳定，且被大众接受，成为一套标准。