---
layout:     post
title:      Linux shell学习
subtitle:    shell学习 and 几个小发现
date:       2020-04-27
author:     BY Diego
header-img: /img/wenzhang/post-bg-linux-shell.jpg
catalog: true
tags:
    - Linux
    - Shell
    - 编程
---

## 一、Shell 简介
Shell 是一个用 C 语言编写的程序，它是用户使用 Linux 的桥梁。Shell 既是一种命令语言，又是一种程序设计语言。

Shell 是指一种应用程序，这个应用程序提供了一个界面，用户通过这个界面访问操作系统内核的服务。

常见的linux shell

* Bourne Shell（/usr/bin/sh或/bin/sh）
* Bourne Again Shell（/bin/bash）
* C Shell（/usr/bin/csh）
* K Shell（/usr/bin/ksh）
* Shell for Root（/sbin/sh）

## 二、语法

### ① 变量
```sh
## 定义
test=abc
${test} # 括号包含
$test  # 不含括号

## 删除变量
unset variable_name

## 只读变量
myUrl="http://www.google.com"
readonly myUrl
```

### ② 字符串
```bash
## 字符串拼接 (无需连接符，可以用来绕过限制)
string='a'"b"'c'"d"
## 求长度
echo ${#string}
## 截取字符串
echo ${string:2:4}
```
![JhMGw9.png](https://s1.ax1x.com/2020/04/27/JhMGw9.png)
![Jh3sjf.png](https://s1.ax1x.com/2020/04/27/Jh3sjf.png)

可以利用内置变量截取字符来拼接字符串
![Jh8A5d.png](https://s1.ax1x.com/2020/04/27/Jh8A5d.png)

类似内置变量可参考
http://blog.chinaunix.net/uid-24004458-id-2153236.html
### ③ 数组
```sh
## 定义 空格为分割符
array=(123 0.9999 "abc")
## 访问数组
echo ${array[0]}
## 输出数组所有元素
echo ${array[@]}
echo ${array[*]}
## 获取数组长度
echo ${#array[@]}
echo ${#array[*]}
```

![JhMTTs.png](https://s1.ax1x.com/2020/04/27/JhMTTs.png)

### ④ 注释
```sh
# 注释一

;<<EOF
注释二
EOF
```

### ⑤ shell 脚本传递参数
参数处理|	说明
-|-
\$#	|传递到脚本的参数个数
\$* |	以一个单字符串显示所有向脚本传递的参数。如"$*"用「"」括起来的情况、以"$1 $2 … $n"的形式输出所有参数。
\$$|	脚本运行的当前进程ID号
\$!|	后台运行的最后一个进程的ID号
\$@|	与\$*相同，但是使用时加引号，并在引号中返回每个参数。如"\$@"用「"」括起来的情况、以"$1" "$2" … "$n" 的形式输出所有参数。
\$-|	显示Shell使用的当前选项，与set命令功能相同。
\$?|	显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误。
\$n  | 传输到脚本的参数 大于9要用{}包裹

![Jh19Rx.png](https://s1.ax1x.com/2020/04/27/Jh19Rx.png)

上述变量存在一些空值，可以用这些空值或者存在的值绕过限制
如这里的\$! 或者用未定义的变量({}没被过滤的条件下)执行
```sh
cat  /et$!c/passwd
cat  /et${a}c/passwd
```

这里用php 的system函数执行一下
![Jh3aAH.png](https://s1.ax1x.com/2020/04/27/Jh3aAH.png)
![Jh3NHe.png](https://s1.ax1x.com/2020/04/27/Jh3NHe.png)

### ⑥ 运算符
shell 基本运算依靠于如 awk 和 expr
```sh
val=`expr 2 + 2`
echo "两数之和为 : $val"
```
这里 \`\`代表的是执行系统命令，因此 \`\`也可以用来绕过限制


运算符	|说明	|举例
-|-|-
-eq	|检测两个数是否相等，相等返回 true。|	[ \$a -eq \$b ] 返回 false。
-ne|	检测两个数是否不相等，不相等返回 true。|	[ \$a -ne \$b ] 返回 true。
-gt	|检测左边的数是否大于右边的，如果是，则返回 true。|	[ \$a -gt \$b ] 返回 false。
-lt	|检测左边的数是否小于右边的，如果是，则返回 true。	|[ \$a -lt \$b ] 返回 true。
-ge	|检测左边的数是否大于等于右边的，如果是，则返回 true。|	[ \$a -ge \$b ] 返回 false。
-le	|检测左边的数是否小于等于右边的，如果是，则返回 true。|	[ \$a -le\ $b ] 返回 true。
!	|非运算，表达式为 true 则返回 false，否则返回 true。|	[ ! false ] 返回 true。
-o|	或运算，有一个表达式为 true 则返回 true。|	[ \$a -lt 20 -o \$b -gt 100 ] 返回 true。
-a	|与运算，两个表达式都为 true 才返回 true。|	[ \$a -lt 20 -a \$b -gt 100 ] 返回 false。
=	|检测两个字符串是否相等，相等返回 true。|	[ \$a = \$b ] 返回 false。
!=	|检测两个字符串是否相等，不相等返回 true。|	[ \$a != \$b ] 返回 true。
-z	|检测字符串长度是否为0，为0返回 true。|	[ -z \$a ] 返回 false。
-n|	检测字符串长度是否不为 0，不为 0 返回 true。|	[ -n "\$a" ] 返回 true。
$	|检测字符串是否为空，不为空返回 true。|	[ $a ] 返回 true。

文件类
操作符	|说明	|举例
-|-|-
-b |file	检测文件是否是块设备文件，如果是，则返回 true。	|[ -b \$file ] 返回 false。
-c |file	检测文件是否是字符设备文件，如果是，则返回 true。	|[ -c $file ] 返回 false。
-d |file	检测文件是否是目录，如果是，则返回 true。|	[ -d \$file ] 返回 false。
-f |file	检测文件是否是普通文件（既不是目录，也不是设备文件），如果是，则返回 true。|	[ -f \$file ] 返回 true。
-g| file	检测文件是否设置了 SGID 位，如果是，则返回 true。|	[ -g \$file ] 返回 false。
-k |file	检测文件是否设置了粘着位(Sticky Bit)，如果是，则返回 true。|	[ -k \$file ] 返回 false。
-p |file	检测文件是否是有名管道，如果是，则返回 true。|	[ -p \$file ] 返回 false。
-u |file	检测文件是否设置了 SUID 位，如果是，则返回 true。|	[ -u \$file ] 返回 false。
-r |file	检测文件是否可读，如果是，则返回 true。|	[ -r \$file ] 返回 true。
-w |file	检测文件是否可写，如果是，则返回 true。|	[ -w \$file ] 返回 true。
-x |file	检测文件是否可执行，如果是，则返回 true。|	[ -x \$file ] 返回 true。
-s| file	检测文件是否为空（文件大小是否大于0），不为空返回 true。|	[ -s \$file ] 返回 true。
-e |file	检测文件（包括目录）是否存在，如果是，则返回 true。|	[ -e \$file ] 返回 true。

### ⑦ 输出

echo -e 开启转义
printf 类似于c语言
列举一下不同的地方
```sh
printf %s abc def
printf "%s %s %s\n" a b c d e f g h i j
```

![JhzdXT.png](https://s1.ax1x.com/2020/04/28/JhzdXT.png)

### ⑧ 流程控制

if
```sh
if condition
then
    command1
    command2
    ...
    commandN
fi
## 或者
if [ $(ps -ef | grep -c "ssh") -gt 1 ]; then echo "true"; fi
```

for
```sh
for var in item1 item2 ... itemN
do
    command1
    command2
    ...
    commandN
done
## 或者

for var in item1 item2 ... itemN; do command1; command2… done;
```

while
```sh
while condition
do
    command
done
```

case
```sh
case 值 in
模式1)
    command1
    command2
    ...
    commandN
    ;;
模式2|模式3|...)
    command1
    command2
    ...
    commandN
    ;;
esac
```

### ⑨ 自定义函数

定义
```sh
[ function ] funname [()]
{
    action;
    [return int;]
}
```

例子
```sh
funWithParam(){
    echo "第一个参数为 $1 !"
    echo "第二个参数为 $2 !"
    echo "第十个参数为 $10 !"
    echo "第十个参数为 ${10} !"
    echo "第十一个参数为 ${11} !"
    echo "参数总数有 $# 个!"
    echo "作为一个字符串输出所有参数 $* !"
}
funWithParam 1 2 3 4 5 6 7 8 9 34 73
```

## 三、参考

[Shell 教程](https://www.runoob.com/linux/linux-shell.html)

[shell中的内置变量](http://blog.chinaunix.net/uid-24004458-id-2153236.html)
