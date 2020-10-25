---
layout:     post
title:      PHP Phar 利用
subtitle:   很早之前就有接触一直没进行总结，趁机总结一下
date:       2020-08-06
author:     BY Diego
header-img: /img/wenzhang/post-bg-格式化.jpgimg/wenzhang/php-phar.jpg
catalog: true
tags:
    - 反序列化
    - PHP
---



# 一、 Phar  简介



phar扩展提供了一种将整个PHP应用程序放入称为“ phar”（PHP归档文件）的单个文件中的方法，以便于分发和安装。 类似于java 中的jar包

*phar* 这个词是*PHP*和 *Archive的缩写*，它宽松地基于Java开发人员熟悉的*jar*（Java Archive）。

别的也不说了 详情看文档 [phar文档](https://www.php.net/manual/en/intro.phar.php)



# 二、Phar用法 



构建也是类似于jar

前提php.ini

```
phar.readonly = 0
```

构建如下目录

![ayT8cq.png](https://s1.ax1x.com/2020/08/05/ayT8cq.png)



src 目录存放源码，build 用来存放构建好的phar文件 index.php用来构建phar

源码如下

/index.php

```php
<?php

$phar = new Phar("build/test.phar", 
  FilesystemIterator::CURRENT_AS_FILEINFO |       FilesystemIterator::KEY_AS_FILENAME, "test.phar");

$phar->startBuffering();

$phar->buildFromDirectory("src", '/\.php$/');

$phar->setStub($phar->createDefaultStub("index.php"));

$phar->stopBuffering();

```

/src/index.php（调用打包后的php文件）

```php
<?php
require_once "phar://test.phar/function.php";
require_once "phar://test.phar/function1.php";
?>
```

/src/function.php

```php
<?php
class Test {
	public $name;
	
	public function run(){
		echo("Test::run\n");
	}
}
function myfunc(){
	echo("myfunc\n");
}
?>
```

/src/function1.php

```php
<?php
class Test1 {
	public $name;
	
	public function run(){
		echo("Test1::run\n");
	}
}
function myfunc1(){
	echo("myfunc1\n");
}
?>
```



运行/index.php 会生成 /build/test.phar



![ayqGiF.png](https://s1.ax1x.com/2020/08/05/ayqGiF.png)



然后就是利用test.phar文件 ，新建一个php 文件 运行效果如下

```php
<?php

require 'phar://test.phar';

myfunc();
myfunc1();

$test = new Test();
$test->Run();

$test1 = new Test1();
$test1->Run();
?>
```




![ayqJG4.png](https://s1.ax1x.com/2020/08/05/ayqJG4.png)



这样就实现了简单的封装功能



# 三、Phar 文件格式



格式主要如下

![](https://cdn.nlark.com/yuque/0/2019/png/239570/1560660577305-69de6e9c-5fc0-4b98-8d6a-57edd056234d.png)

phar 文件的本质就是 php代码 + 数据 + 签名 直接`php phar文件` 就能运行，`stub` 简单来说就是当使用`phar协议`  执行phar 文件时如何去处理 类似序列化的`__wakeup`，也就是刚才说的php代码部分

这里还要说一下`__HALT_COMPILER()`函数的作用

![ayOtD1.png](https://s1.ax1x.com/2020/08/05/ayOtD1.png)



简单来说就是中制编译，后面的代码不管对错都不会去编译，因此 数据 + 签名放在 `__HALT_COMPILER()`之后

![ayOYuR.png](https://s1.ax1x.com/2020/08/05/ayOYuR.png)



继续看stub部分



```php
<?php

$phar = new Phar("test.phar",0,"test.phar");

$phar->startBuffering();

$phar->setStub("<?php phpinfo();__HALT_COMPILER();?>"); // 验证

$phar["aa"] = 123; //随便写点数据进去

$phar->stopBuffering();

?>
```



生成的phar文件内容

![ayjNtK.png](https://s1.ax1x.com/2020/08/05/ayjNtK.png)



使用phar 协议调用test.phar 触发了 stub 部分的代码，因此`__HALT_COMPILER()`必须要有否则会报错，之前的要符合php语法 及开始格式的`xxx<?php xxx;__HALT_COMPILER();`



![ayjUfO.png](https://s1.ax1x.com/2020/08/05/ayjUfO.png)



stub 也可以使用默认的，但他的操作就很多

```php
$phar->setDefaultStub();
```



代码太多就放开头和结尾一部分

![ayxlZR.png](https://s1.ax1x.com/2020/08/05/ayxlZR.png)



![ayxML9.png](https://s1.ax1x.com/2020/08/05/ayxML9.png)



# 四、Phar 漏洞利用 



## (1) 文件上传 + 文件包含 



因为phar文件 特性，可以在stub最开始添加任意字符，而后缀又没有任何限制，可以绕过某些文件上传限制，然后再利用文件包含去调用phar中的内容



举个很早以前的题目，图片存储的题目





扫一下后台


![a4gM5D.png](https://s1.ax1x.com/2020/08/08/a4gM5D.png)

url 存在f参数，

![a4glPe.png](https://s1.ax1x.com/2020/08/08/a4glPe.png)



伪协议读取源码 **php://filter/read/convert.base64-encode/resource=**

然后分别读取到了 **index.php upload.php show.php**

- **index.php**



```php
<?php
error_reporting(0);

@session_start();
// posix_setuid(1000);

$f = empty($_GET['f']) ? 'fail' : $_GET['f'];
if(preg_match('/\.\./',$f))
{
	die('Be a good student!');
}
if(preg_match('/rm/i',$_SERVER["QUERY_STRING"]))
{
	die();
}
foreach ($_POST as $key => $value) {
	if(preg_match('/rm/i',$value))
	{
		die();
	}
	if(preg_match('/mv/i',$value))
	{
		die();
	}
	if(preg_match('/cp/i',$value))
	{
		die();
	}
	if(preg_match('/touch/i',$value))
	{
		die();
	}
	if(preg_match('/>/i',$value))
	{
		die();
	}
}

?>

<!DOCTYPE html>
<html>
	<head>
		<title>图像系统</title>
		<meta charset="utf-8">
	</head>
	<body>
		<div class="container">
			<div class="jumbotron">
				<h1>图片存储</h1>
				<p class="lead">请在这里上传您的图片,我们将为您保存:)</p>
				<form action="?f=upload" method="POST" id="form" enctype="multipart/form-data">
					<input type="file" id="image" name="image" class="btn btn-lg btn-success" style="margin-left: auto; margin-right: auto;">
					<input type="submit" id="submit" name="submit" class="btn btn-lg btn-success" role="button" value="上传图片">
				</form>
			</div>
	   	</div>
	</body>
</html>
<?php
if($f !== 'fail')
{
	if(!(include($f.'.php')))
	{
		?>
		<div class="alert alert-danger" role="alert">NOPE</div>
		<?php
			exit;
	}
}
?>
```



- **uploads.php**



```php
<?php  
if(isset($_POST['submit']) && !empty($_FILES['image']['tmp_name']))
{	
	$name = $_FILES['image']['tmp_name'];
	$type = $_FILES['image']['type'];
	$size = $_FILES['image']['size'];

	if(!is_uploaded_file($name))
	{
		?>
		<div class="alert alert-danger" role="alert">图片上传失败,请重新上传:)</div>
		<?php
			exit;
	}	

	if($type !== 'image/png')
	{
		?>
		<div class="alert alert-danger" role="alert">只能上传PNG图片</div>
		<?php
			exit;
	} 	

	if($size > 10240)
	{
		?>
		<div class="alert alert-danger" role="alert">图片大小超过10KB</div>
		<?php
			exit;	
	}
	function create_imageid()
	{
		return sha1($_SERVER['REMOTE_ADDR'] . $_SERVER['HTTP_USER_AGENT'] . time() . mt_rand());
	}
	$imageid = create_imageid();
	move_uploaded_file($name,"uploads/$imageid.png");

	echo "<script>location.href='?f=show&imageid=$imageid'</script>";
}
?>
```



- **show.php**



```php
<?php  
@$imageid = $_GET['imageid'];
if(empty($imageid))
{
	echo "<br>请输入imageid";
}
else
{
	echo <<<EOF
	<div class="alert alert-success" role="alert">
	图像上传成功,<a href="uploads/$imageid.png" class="alert-link">点此查看</a>
	</div>
EOF;
}
?>
```



构造phar 文件

![acZ9G4.png](https://s1.ax1x.com/2020/08/06/acZ9G4.png)



shell.php

![acVzIU.png](https://s1.ax1x.com/2020/08/06/acVzIU.png)



修改后缀然后上传

![acZse0.png](https://s1.ax1x.com/2020/08/06/acZse0.png)





![acZywV.png](https://s1.ax1x.com/2020/08/06/acZywV.png)





## (2) Phar反序列化



只有通过`phar::setMetadata();` 设置的值才会被序列化放在stub 之后 通过直接赋值 无法序列化

![acl6G6.png](https://s1.ax1x.com/2020/08/06/acl6G6.png)



需要添加额外数据 或者`$phar->addFromString()`添加任意文件才能生成phar文件

![ac8Qqe.png](https://s1.ax1x.com/2020/08/06/ac8Qqe.png)



调用后会进行反序列化， 利用的时候构造pop链 然后触发反序列化

![ac8MrD.png](https://s1.ax1x.com/2020/08/06/ac8MrD.png)





# 五、参考

[File Operation Induced Unserialization via the “phar://” Stream Wrapper](https://cdn2.hubspot.net/hubfs/3853213/us-18-Thomas-It's-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-....pdf)

[利用 phar 拓展 php 反序列化漏洞攻击面](https://paper.seebug.org/680/)

[PHP开发常识：什么是Phar?](https://www.webhek.com/post/packaging-your-php-apps-with-phar.html)

