---
title: "Jwt的几种攻击手法"
date: 2020-01-07T14:34:57+08:00
draft: false
tags: ['jwt']
categories: ['WEB漏洞']
---

jwt的几种攻击手法
<!--more-->
## 简介

JWT的全称是Json Web Token。它遵循JSON格式，将用户信息加密到token里，服务器不保存任何用户信息，只保存密钥信息，通过使用特定加密算法验证token，通过token验证用户身份。基于token的身份验证可以替代传统的cookie+session身份验证方法。

## 结构
jwt由三个部分组成：header.payload.signature,以小数点分割，所以在实战中，我们看到这样的数据的时候，可以以此为依据

### Header
头部主要的作用是声明token加密用的啥算法
```
{
        "alg" : "HS256"
        "typ" : "jwt"
}
```
typ指明类型为jwt

### Payload
payload部分主要是传递一些用户数据
![avatar](https://sqlmap.wiki/images/jwt1.png)

### Signature
签名哈希部分是对上面两部分数据签名，通过指定的算法生成哈希，以确保数据不会被篡改。<br>
首先，需要指定一个密码（secret）。该密码仅仅为保存在服务器中，并且不能向用户公开。然后，使用标头中指定的签名算法（默认情况下为HMAC SHA256）根据以下公式生成签名
```
signature = HMAC-SHA256(base64urlEncode(header) + '.' + base64urlEncode(payload), secret_key)
```
也就是说，如果我们没有这个密钥，就篡改不了header和payload中的数据

## 实战利用
这里利用webgoat这个靶场来进行演示几种攻击手法

### 空加密算法
前面说了，header的alg用来声明token用的加密算法，而我们把值改为None，这样token就不会进行加密，当然后面的Signature得为空。进入到webgoat>a2>jwt tokens第4关
![avatar](https://sqlmap.wiki/images/jwt2.png)

一个投票页面，有几个角色。如果我们想删除这个页面，必须是管理员角色，但是这里的用户都不是管理员角色。
![avatar](https://sqlmap.wiki/images/jwt3.png)

我们先随便选择一个用户，然后点击删除按钮，用burpsuite抓包。
![avatar](https://sqlmap.wiki/images/jwt4.png)
看到这个access_token,以点号作为分割，就是我们说的jwt，程序就是以jwt作为标准，判断你有没有权限删除这个页面。因为header和payload这两个部分是base64加密，我们可以解密出来。这里利用burpsuite的 Json Web Tokens插件，burp商店就能下载。
![avatar](https://sqlmap.wiki/images/jwt5.png)

可以看到，这个token用的算法是HS512(sha512),payload的第一个参数，iat代表的是签发时间，admin为false，说明这个用户权限不是admin，user名为Jerry。这里我们尝试把alg改为None，然后把payload的admin值改为true。

![avatar](https://sqlmap.wiki/images/jwt6.png)
![avatar](https://sqlmap.wiki/images/jwt7.png)

最后把2个段重新以点号进行拼接，记得第三段Signature虽然删除了，但是点号不能删除。重新进行发包
![avatar](https://sqlmap.wiki/images/jwt8.png)
可以发现，已经删除成功了。这就是把算法改成None的一个操作。

### 爆破密钥
如果None设置不成功，最后可以试下爆破密钥。但是局限性很大
进入webgoat>a2>jwt tokens 第5关
![avatar](https://sqlmap.wiki/images/jwt9.png)

给了一个jwt，然后生成一个新的jwt。这时候只能进行爆破key了。推荐一个工具<a href=https://github.com/Ch1ngg/JWTPyCrack>jwt爆破工具</a>，有2个模块，一个是算法None模块，一个是爆破密钥，我们选择爆破密钥模块。-s参数指定jwt，--kf指定字典爆破成功则会提示found key

![avatar](https://sqlmap.wiki/images/jwt10.png)
进入网站jwt.io,把我们的key和jwt填进去，生成新的jwt。
![avatar](https://sqlmap.wiki/images/jwt11.png)
爆破密钥可能是迫不得已的手段。看大佬说的，可以去前端找算法中找key，可能是通用的。

![avatar](https://sqlmap.wiki/images/jwt12.png)

kid参数可能存在的漏洞
kid是jwt header中的一个可选参数，全称是key ID，它用于指定加密算法的密钥的位置，因为这个kid是由用户输入的，所以可能也会存在漏洞，如SQL注入、命令注入、任意文件读取
进入webgoat>a2>jwt tokens 第8关
![avatar](https://sqlmap.wiki/images/jwt13.png)

题目目的应该就是删除这个杰瑞或者汤姆，但是得有他们的token。我们先抓个包看下信息。
![avatar](https://sqlmap.wiki/images/jwt14.png)

kid存着webgoat key，我们白盒看下代码。
![avatar](https://sqlmap.wiki/images/jwt15.png)
发现kid是可控的，并且把它带进数据库进行查询，这里构造相应的payload然后base64加密就好了，这里就不演示了。

