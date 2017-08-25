---
title: solr 语法简介
date: 2017-06-25 17:39:50
tags:
    - frame 
type: "categories" 
---
## 一、 查询参数说明
 
 
| 参数名        | 含义          | 例子  |
| ------------- |:-------------:| -----:|
| q      | 查询字符串,必须 | q=name=wxx and sex=man(查询姓名是wxx并且性别男) |
| fq      | filter query,过滤查询,作用：在q查询符合结果中同时是fq查询符合的(可多个) | q=uid:xxx&fq=log_date:[2017-02-09 TO 2017-02-10]   |
| fl | 指定返回那些字段内容，用逗号或空格分隔多个      | q=uid=xxx&fl=action,platform |
| start,rows |start 从第几条开始返回,rows 返回多少条，可用于分页|q=uid:xxx&start=0&rows=10|
| sort | 排序 desc asc| q=uid=xxx&sort=uid+desc |
| wt | 指定输出格式( xml, json, php, python,csv...)|wt=json|
| q.op |表示q 中 查询语句的 各条件的逻辑操作 AND(与) OR(或) |---|
|hl |是否高亮 ,如hl=true||
|hl.fl |高亮field,字段必须出现在查询条件中 |hl.fl=uid,platform|
|hl.snippets|默认是1,这里设置为3个片段|---|
|hl.simple.pre  |高亮前面的格式|---|
|hl.simple.post  |高亮后面的格式|---|
|facet  |是否启动统计|---|
|facet.field  |统计field|---|


## 二、 Solr运算符
| 运算符       | 含义          | 例子  |
| ------------- |:-------------:| -----:|
|“:”|指定字段查指定值|如返回所有值*:*|
|“?”|表示单个任意字符的通配|---|
| “~” |表示模糊检索|如检索拼写类似于”wxx~”的项这样写：wxx~将找到形如wxx和wxxx的单词；wxx~0.8，检索返回相似度在0.8以上的记录|
|"^"|控制相关度检索|如检索相隔10个单词的”apache”和”jakarta”，”jakarta apache”~10|
|AND,\|\|,OR,&&,NOT,! |布尔操作符|---|
|()|用于构成子查询|---|
|[]|包含范围检索|如检索某时间段记录，不包含头尾
date:{200707 TO 200710}|
|/|转义操作符，特殊字符包括|---|


 注：①“+”和”-“表示对单个查询单元的修饰，and 、or 、 not 是对两个查询单元是否做交集或者做差集还是取反的操作的符号
 
比如:AB:china +AB:america ,表示的是AB:china忽略不计可有可无，必须满足第二个条件才是对的,而不是你所认为的必须满足这两个搜索条件

如果输入:AB:china AND AB:america ,解析出来的结果是两个条件同时满足，即+AB:china AND +AB:america或+AB:china +AB:america


## 三、 Solr查询语法
1.最普通的查询，比如查询姓魏的人（ Name:张）,如果是精准性搜索相当于SQL SERVER中的LIKE搜索这需要带引号（""）,比如查询含有北京的（Address:"北京"）

2.多条件查询，注：如果是针对单个字段进行搜索的可以用（Name:搜索条件加运算符(OR、AND、NOT) Name：搜索条件）,比如模糊查询（ Name:张 OR Name:李 ）单个字段多条件搜索不建议这样写，一般建议是在单个字段里进行条件筛选，如（ Name:张 OR 李），多个字段查询（Name:张 + Address:北京 ）
