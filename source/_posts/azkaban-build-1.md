---
title: azkaban 环境搭建
date: 2019-08-30
tags:
    - frame
type: "categories"
---

azkaban 环境搭建，插件安装，简单demo，功能演示。

## 一. azkaban 环境搭建

### 1.  azkaban 代码编译

*  打开 azkaban [源码地址](https://github.com/azkaban/azkaban),将代码下载下来。

 ```
    ## 选择自己适合的版本，我选的 3.49.0版本，azkaban高版本与 azkaban插件(3.0)不兼容

   git clone https://github.com/azkaban/azkaban.git
   
   git tag  
   
   git checkout 3.49.0
 ```
 
*  编译代码(azkaban 用的是gradle进行的项目构建管理)

  ```
    ##没装gradle的，使用下面命令跳过测试构建环境
    ./gradlew clean build  -x test
     
  ```
  
* 编译包获取
![Alt text](https://raw.githubusercontent.com/nick-weixx/nick-weixx.github.io/master/img/azkaban-build-1.png)
