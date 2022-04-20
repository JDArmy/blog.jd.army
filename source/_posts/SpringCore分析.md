---
title: Spring-Core RCE分析
tags: [Spring,漏洞]
categories: 漏洞分析
date: 2022-04-11 10:08:16
---

迟来的SpringMVC 框架RCE分析。本文章简单介绍了SpringMVC框架请求处理流程，并以此对漏洞进行了分析与复现。
<!-- more -->
## 框架浅析

`SpringMVC`其本质上是一个`Servlet`，它的请求处理主要是在`DispatcherServlet`中，这里大概有四步：

1. 根据`Request`找到`Handler`
2. 根据`Handler`找到`HandlerAdapter`
3. 用`HandlerAdapter`调用`Handler`处理请求
4. 处理结果并渲染输出给用户

借用一张图来看下这个流程

![屏幕快照 2019-10-17 下午8.49.43](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgPing_Mu_Kuai_Zhao_-2019-10-17-_Xia_Wu_8.png)

`Handler`是用来处理请求，`SpringMVC`内置了大量的`Handler`，我们重点关注下其中对参数进行处理的，主要是`HandlerMethodArgumentResolver`和`HandlerMethodReturnValueHandler`，前者表示一个参数解析器，后者除了解析参数之外还可以处理相应类型的返回值。以下是`HandlerMethodArgumentResolver`的实现类

![image-20220410133631042](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220410133631042.png)

它们基本上都实现了:

```java
public boolean supportsParameter(MethodParameter parameter) //和 
public Object  resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory)
```

当`RequestMapping`对应参数符合`supportsParameter`会使用`resolveArgument`解析请求，并最终得到参数的值传入`RequestMapping`，这里以`RequestParamMapMethodArgumentResolver`简单介绍下

```java
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		RequestParam requestParam = parameter.getParameterAnnotation(RequestParam.class);
		return (requestParam != null && Map.class.isAssignableFrom(parameter.getParameterType()) &&
				!StringUtils.hasText(requestParam.name()));
	}

	@Override
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        
        ......
      else {
				Map<String, String[]> parameterMap = webRequest.getParameterMap();
				Map<St	ring, String> result = CollectionUtils.newLinkedHashMap(parameterMap.size());
				parameterMap.forEach((key, values) -> {
					if (values.length > 0) {
						result.put(key, values[0]);
					}
				});
				return result;
			}
    }
```

首先看其支持类型，需要有`RequestParam`注解，且参数类型为`Map`，所以可以定义如下接口：

```java
    @ResponseBody
    @RequestMapping("/mvc/world")
    public String world(@RequestParam HashMap<String, String> map) {
        return "successfuladd";
    }
```

该接口就会被`RequestParamMapMethodArgumentResolver`处理，很容易看出这里简单的做了个类型转换，这里的`result`就是我们需要的参数了

![image-20220410151644951](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220410151644951.png)

有趣的是这里如果两个相同参数的请求，其只会取第一个的值，而如果是`RequestParamMethodArgumentResolver`进行处理时会把两个参数值通过`,`进行连接。

部分解析器及其作用：

![image-20220410154419251](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220410154419251.png)

## 漏洞分析

前面扯了那么多，现在终于是进入正题了，先来搭建下漏洞环境：

- JDK：11.0.14
- Tomcat：9.0.60
- Spring 5.3.17

主要代码如下：

```java
@Controller
public class TestController {

    @ResponseBody
    @RequestMapping("/mvc/hello")
    public String hello(User user) {
        System.out.println(user.getName());
        return "success";
    }
}

//User
public class User {
  private String name;

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }
}
```

PoC:`class.module.classLoader.resources.context.parent.pipeline.first.pattern=***`

单独看`PoC`可能会疑惑这个参数是怎么来的，所以这里要结合着环境进行分析。可以看到`hello`的参数`User`，这是一个没有注释的非通用类型参数，而上文中有提到不同参数类型的解析器也不一样，现在的情况会由`ModelAttributeMethodProcessor`进行处理，跟进其`resolveArgument`方法，它会尝试从当前请求中获取值并绑定到`user`上

![image-20220410174059024](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220410174059024.png)

一路跟进`bindRequestParameters`函数直到`org.springframework.validation#applyPropertyValues`。

![image-20220410181059133](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220410181059133.png)

这里经过`getPropertyAccessor()`我们实际上获取到了一个`User`对象的`BeanWrapper`实例

![image-20220410181408547](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220410181408547.png)



在这里我们补充下`BeanWrapper`相关的内容，在`Spring`中，`BeanWrapper`接口是对`Bean`的包装，定义了对包装对象的属性值的访问与修改的接口，`BeanWrapperImpl`则是对`BeanWrapper`的默认实现，`BeanWrapperImpl`类有多个设置`bean`属性值的重载方法，其中就有`public void setPropertyValue(PropertyValue pv)`，`PropertyValue `以对象的方式存储键值对，比`Map`使用起来要灵活，通过`BeanWrapperImpl`设置属性值：

```java
public class BeanWrapperTest {
    public static void main(String[] args) {
        User user=new User();

        BeanWrapper bw= PropertyAccessorFactory.forBeanPropertyAccess(user);
        bw.setPropertyValue(new PropertyValue("name","bean"));

        System.out.println(user.getName());
    }
}
```

也可以通过`getPropertyDescriptors`获取所有属性值：

```java
public class BeanWrapperTest {
    public static void main(String[] args) {
        User user=new User();

        BeanWrapper bw= PropertyAccessorFactory.forBeanPropertyAccess(user);
        for (PropertyDescriptor p :
            bw.getPropertyDescriptors()) {
            System.out.println(p.getName());
        }
    }
}

//output
class
name
```

可以看到除了除了`name`之外还会有一个`class`，那这是不是说明`class`也可以被我们修改呢？看一下`setPropertyValue`的代码，它会进入`getPropertyAccessorForPropertyPath`，它支持两种方式的属性值，一种是直接用`name`进行操作，一种则是`user.name`的形式进行递归逐步获取到`user`后对`name`进行操作，这里对第二种情况进行分析：

添加新类`God`:

```java
public class God {
    private String name;

    public String getName(){
        return name;
    }

    public void setName(String name){
        this.name = name;
    }
}
```

同时在`User`中加入:

```java
    private God god = new God();


    public God getGod(){
        return god;
    }
```



最后运行

```java
public class BeanWrapperTest {
    public static void main(String[] args) {
        User user = new User();

        BeanWrapper bw= PropertyAccessorFactory.forBeanPropertyAccess(user);
        bw.setPropertyValue(new PropertyValue("god.name","bean"));
        System.out.println(user.getGod().getName());
    }
}
```

第一次解析`god`，如果之前未解析过`bean`类，首先会对该类进行分析并缓存，使用的方法是`CachedIntrospectionResults.forClass`，在获取到所有`get,set`方法后循环判断了该类为`Class`的同时属性是不是`classLoader`，防止了直接`class.classLoader`来进一步获取值

![image-20220410201409567](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220410201409567.png)

缓存之后就开始获取属性值了，如果该属性可读的话就会在`getValue`时执行其`get`方法，这里的`Value`就是`God`实例

![image-20220410201835796](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220410201835796.png)



最后会以该实例生成一个新的`nestedPa`返回并进入第二次循环

![image-20220410202130985](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220410202130985.png)

不过第二次时已经没有`.`了，所以直接返回`this`，也就是`god`，并以此知道要设置的值为`god.name`，所以后续就进入了设置属性值的流程，只有当该属性值存在且可写的情况下才可以继续往下执行

![image-20220410202831939](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220410202831939.png)

至此整个流程就结束了，让我们回到漏洞

`setPropertyValues(mpvs, isIgnoreUnknownFields(), isIgnoreInvalidFields())`其实也调用了`setPropertyValue(PropertyValue pv)`

![image-20220410203054865](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220410203054865.png)

那么结合上文对`setPropertyValue`流程的分析，其实我们已经大致理解了`payload`的格式，包括为什么用`class.module.classLoader`而不是直接`class.classLoader`。在`Tomcat`中是`ParallelWebappClassLoader`，而且其有一个属性`getResources`，就这样层层递归，最终操作日志，达成任意文件写入，从而实现`RCE`，在`SpringBoot`的`LaunchedURLClassLoader`中并不存在`getResources`所以直接使用`SpringBoot`的情况下上述`Payload`是不起作用的。

## 修复方案

针对该漏洞`Spring`以及 `Tomcat`都做出了修复

`Spring:` `Class`类仅可以获取`name`相关的值了，而且对没有写操作权限的`ClassLoader`以及`ProtectionDomain`做了限制

![image-20220410204849484](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220410204849484.png)

`Tomcat`则是直接把`getResources`返回为空了

![image-20220410205319618](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220410205319618.png)

## 参考文章

- https://paper.seebug.org/1877/
- https://blog.csdn.net/zhiweianran/article/details/7919129
- http://rui0.cn/archives/1158