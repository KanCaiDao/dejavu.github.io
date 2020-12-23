---
title: "记一次客服软件挖掘RCE"
date: 2020-11-04T17:25:29+08:00
draft: false
tags: ['客服软件RCE']
categories: ['WEB漏洞']
---

项目中的一个站，因为目标是基于第三方正规平台，只能找一些别的点进行渗透，正好看到了这个客服软件
<!--more-->


## 前言
通过搜索引擎加上商务qq，得到了一个试用账号。下载客服软件，发现这个软件基于Electron 框架

## 介绍
Electron 是一款基于 Web 技术（HTML5 + Javascript + css）构建图形界面的开发框架，基于 nodejs 和 Chromium 开发。因为无痛兼容 nodejs 包管理（npm）的大量功能丰富的模块，相对于 native 实现降低了开发难度和迭代成本，受到了开发者的青睐。


## 解包


Electron的主文件在resources文件夹下，里面有个electron.asar，app.asar。electron.asar是自带的不用管，主要看app.asar，asar是压缩包，我们需要对它进行解压

### 安装asar
npm install -g asar

### 解压命令
asar extract app.asar <输出目录>




## electron挖掘点


electron常见的漏洞有2个，一个是**自定义协议**造成的命令注入，一个是开启**nodeIntegration**造成的RCE。
### 
### 自定义协议命令注入 (CVE-2018-1000006)


全局搜索setAsDefaultProtocolClient和registerSchemesAsPrivileged，并未发现关键字，说明未注册协议，这里就不用看了


### 开启**nodeIntegration造成的RCE**
### **
全局搜索nodeIntegration，如果值为True，说明开启了Node.js扩展,可以执行系统命令。这时候我们只需要找到一个XSS点,然后执行Node.js系统命令,即可造成RCE。而这个客服软件nodeIntegration为True,且有XSS。Nodejs执行系统命令的语句是require('child_process').exec('') 

## 利用过程


与客服建立通话，发送payload如下

![QQ截图20201104151601.png](https://sqlmap.wiki/images/68747470733a2f2f63646e2e6e6c61726b2e636f6d2f79757175652f302f323032302f706e672f313439383635322f313630343437363630363238342d34313366623965372d643831362d343737342d613831662d3831326366313464613766392e706e6723616c69676.png)
效果
![QQ截图20201104152237.png](https://sqlmap.wiki/images/QQ截图20201104152237.png)



## 漏洞分析


全局搜索{"type":"say", 定位到以下关键代码


```javascript
sendMessageBtn: function() {
                        var t = this,
                            e = this.textarea.replace(/&nbsp;/g, "").trim(),
                            a = !0;
                        if (this.externalLinkList.length && this.externalLinkList.forEach(function(i) { - 1 < e.indexOf(i.externalLink) && (t.$message.error("请勿发送包含外链的文本信息"), a = !1)
                        }), !a) return ! 1;
                        if ("" == this.chatInit.textarea.replace(/[ ]|[&nbsp;]/g, "")) return Object(v.Message)({
                            showClose: !0,
                            message: "发送的内容不能为空",
                            type: "error",
                            duration: 3e3
                        }),
                            !1;
                        this.chatTruth.showEmoji = !1;
                        var i = this.chatInit.textarea,
                            n = /\[(.+?)\]/g;
                        this.chatInit.textarea.match(n) && this.chatInit.textarea.match(n).map(function(t) {
                            for (var e in j.a) {
                                var a = "[" + j.a[e][0] + "]",
                                    n = new RegExp("\\[(" + j.a[e][0] + ")\\]", "ig");
                                t == a && (i = i.replace(new RegExp(n, "g"), ":" + e + ":"))
                            }
                        });
                        var o = {
                                type: "say",
                                sendtime: this.getTimes(),
                                agent_avatar: this.chatInit.agentAvatar,
                                sendname: this.chatInit.agentName,
                                from_uid: this.chatInit.agentId,
                                to_uid: this.chatItem.onlineVisitorEntity.visitorId,
                                role: "agent",
                                msg_type: "text",
                                text: i
                            },
                            r = {
                                messagesId: this.chatItem.messageId,
                                visitorId: this.chatItem.onlineVisitorEntity.visitorId,
                                msgType: "text",
                                talk_time: "0",
                                text: i
                            };
```


可以看到text只过滤了/把它替为空，其他的完全没过滤。所以造成了RCE
