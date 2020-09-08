---
title: "连我mysql读你文件"
date: 2020-01-03T11:07:26+08:00
draft: flase
tags: ['连我mysql读你文件']
categories: ['web漏洞']
---


连我mysql读你文件
<!--more-->
## 原理
我们可以伪造一个 MySQL 的服务端，当有客户端连接上这个假服务端的时候，我们就可以任意读取客户端的一个文件，当然前提是运行客户端的用户具有读取该文件的权限。<br>

这个问题主要是出在LOAD DATA INFILE这个语法上，这个语法主要是用于读取一个文件的内容并且放到一个表中<br>
有两种语法，分别是:<br>
```mysql
1、load data infile “/data/data.csv” into table TestTable;
```
<br>

```mysql
2、load data local infile "/home/data.csv" into table TestTable;
```
第一个语法是读取本地文件，第二个语法是读取客户端(client)文件，我们要利用的就是第二个语法<br>
我们来看下这个语法的处理流程
![avatar] https://sqlmap.wiki/images/mysql1.png
从上图中，可以看到，第二步，其实是服务端命令客户端干事，也就是说，核心是服务端，服务端命令客户端干啥，客户端就干啥。如果我们把目标网站当成一个客户端，攻击者伪造一个服务端，然后把第二步操作改成读取客户端文件，这样就造成了一个任意文件读取漏洞

## 漏洞利用
1、下载利用工具 https://github.com/allyshka/Rogue-MySql-Server
用Python2运行。编辑该文件，下图的filelist就是我们要读取的文件<br>
![avatar](https://sqlmap.wiki/images/mysql2.png)
2、受害者连接我们伪造的服务端。
![avatar](https://sqlmap.wiki/images/mysql3.png)
3、受害者文件被被读取，并保存到攻击者mysql.log中
![avatar](https://sqlmap.wiki/images/mysql4.png)


## 出现场景
1、结合重装漏洞进行利用<br>
2、数据迁移等需要连接外部数据的功能点

### 结合重装漏洞
如果不了解重装漏洞的，可以参考公众号前面的文章。<br>
利用重装漏洞来到重装页面，数据库填上我们伪造的地址，即可读取任意文件
![avatar](https://sqlmap.wiki/images/mysql5.png)

### 数据迁移等功能点
这是某云的一个数据迁移功能，连接地址填上我们伪造的mysql服务端地址，即可读取任意文件
![avatar](https://sqlmap.wiki/images/mysql6.png)
![avatar](https://sqlmap.wiki/images/mysql7.png)

## 漏洞修复
- 禁掉load读取文件
- 使用加密链接 ssl-mode=VERIFY_IDENTITY

## 思考
可以看到，这个漏洞本质是受害者主动连接攻击者，我们把我们服务端当成蜜罐，客户端当成攻击者，这样是不是就能反击攻击者呢。

## 参考文章
https://y4er.com/post/mysql-read-client-file/
