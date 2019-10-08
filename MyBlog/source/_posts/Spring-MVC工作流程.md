title: Spring MVC工作流程
author: Loux
tags:
  - Spring MVC
categories:
  - 后端知识
date: 2019-02-10 21:51:00
cover: /img/post/5.jpg
---
Spring
MVC框架算是最近使用的比较多的控制层的MVC框架，它以轻量级，效率高，简单易使用等特点迅速得到了开发者们广泛的使用，它和Struts2相比如下所示。

-   **核心控制器**：Spring MVC的核心控制器是Servlet，而Struts2则是Filter。

-   **控制器实例**：Spring
    MVC中Controller是基于方法设计的，每次执行请求只需执行Controller对应的方法即可。而Struts2中的Action则是基于对象进行设计的，每次执行请求都会实例化一个Action对象并注入属性。从这一点来看理论上Spring
    MVC的效率是要高于Struts2的。

-   **参数传递**：Spring
    MVC中是通过方法中的参数来进行参数接受的，Struts2中自身提供多种参数接受，其实都是通过ValueStack进行传递和赋值的。

当我们使用Spring MVC时，首先要引入它的核心控制器DispatcherServlet。那么Spring
MVC的工作流程是怎样的呢？网络上很流行这样一张图，贴在这里方便理解。

![Spring MVC工作流程图1](/images/springMVC1.png)

1.  用户发送的请求被DispatcherServlet(前端控制器)拦截

2.  DispatcherServlet根据url请求handlerMapping(处理器映射器)去查找handler(处理器),找到后向前端控制器返回一条执行链(Handler
    ExecutionChain对象)，执行链中包含了handlerInterceptor对象(如果使用了Spring
    MVC拦截器才有该对象)和handler对象。

3.  DispatcherServlet接受到了执行链后调用HandlerAdapter(处理器适配器)去执行handler

4.  handler执行完成以后向handlerAdapter返回ModelAndView对象(返回页面的情况下)，handlerAdapter又将其返回给DispatcherServlet

5.  DispatcherServlet请求ViewResolver(视图解析器)进行视图解析，解析后视图解析器向DispatcherServlet返回View

6.  DispatcherServlet根据View对视图进行渲染后，将结果返回到浏览器，用户看见了响应结果

特别关键是handlerMapping和handlerAdapter。一个是根据url去找到处理器(也就是我们所说的Controller)；另一个是用来执行handler，也就是执行Controller中对应的方法。网上找到的这张图，感觉对于帮助理解来说特别好。

![Spring MVC工作流程图2](/images/springMVC2.png)