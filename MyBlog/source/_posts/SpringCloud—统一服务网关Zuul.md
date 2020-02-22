title: SpringCloud—统一服务网关Zuul
author: Loux

cover: /img/post/35.jpg

tags:

  - spring cloud
categories:
  - 微服务
date: 2020-02-18 14:10:00

---

在微服务的架构设计思想中，每个服务的实例个数都是动态变化的。对于外部应用程序来说，很难感知到这种变化。现在的项目基本上都是前后端分离的，对于前端工程师来说，要他们记住这么多服务的端口地址也是压力山大的，更何况这些访问地址可能还在变化当中。所以说，后端的服务往往不直接提供给使用方调用，而是有一个网关服务统一管理后端各个服务的uri。当前端调用时首先会到网关这一个服务，网关再根据url路由到相应微服务的api。

# Zuul简介

> Zuul是Netflix开源的一款提供路由、监控、安全的网关服务。Spring官方也将它整理到了Spring Cloud的组件库中，使得我们能在应用中方便、快速的搭建起系统的网关服务。

为什么提到了安全呢？因为Zuul中提供了很多过滤器，我们可以根据过滤器实现一些安全操作，例如权限控制、记录请求uri参数等信息、ip地址黑名单过滤等功能。

下面的使用中会对这些过滤器进行一些介绍。

# 搭建网关服务

在前面，我们已经搭建了用户服务与课程服务。接下来我们要搭建一个网关服务，来统一管理这两个服务的api接口。

首先新建一个服务，引入以下依赖

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <!-- 服务网关 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        </dependency>
```

对于启动类上，需要加上@EnableZuulProxy注解，代表这个服务支持Zuul的代理功能。

```java
@EnableZuulProxy
@SpringCloudApplication
public class ZuulApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZuulApplication.class,args);
    }
}
```

接下来，我们需要在配置文件中配置用户、课程两个服务在网关中的访问路劲

```yml
server:
  port: 9000
spring:
  application:
    name: homepage-zuul
eureka:
  client:
    service-url:
      defaultZone: http://server1:8000/eureka/

zuul:
  prefix: /wutong                               #统一访问路劲的前缀
  routes:
    course:                                     #自定义的名称
      path: /homepage-course/**                  
      serviceId: eureka-client-homepage-course  #服务名称
      strip-prefix: false
    user:
      path: /homepage-user/**
      serviceId: eureka-client-homepage-user
      strip-prefix: false
```

例如我们现在访问localhost:9000/wutong/homepage-course 就能映射到课程服务上去。

访问localhost:9000/wutong/homepage-user就能映射到用户服务上去。

搭建起来真的是非常的简单。如果只想要统一访问api的路劲的话。

# 使用Zuul提供的拦截器

可是刚刚我们说了，Zuul还为我们提供了很多种类的拦截器，通过这些拦截器我们可以实现很多的功能。

我们需要新建类来继承Zuul提供的拦截器。并重写它的一些方法。

我现在要实现一个功能，`对每个请求的uri和请求时间用日志进行打印。` 

首先，先来了解以下ZuulFilter中有哪些我们常用的类型

* PRE_TYPE：在请求被路由前执行
* POST_TYPE：在请求被路由后执行
* ERROR_TYPE：在请求发生错误时执行
*  ROUTE_TYPE：在请求被路由时执行

那么对于我们这个功能来说，需要用到两种Filter类型，一种是PRE_TYPE，一种是POST_TYPE。思路就是在PRE_TYPE的执行方法中记录当前的请求时间，然后在POST_TYPE执行的方法中对时间差进行日志打印，就是我们需要的时间。（POST_TYPE中需要设置filter的优先级,意为在请求相应之前执行）

```java
/**
 * 在过滤器中存储客户端存储请求的时间戳
 */
@Component
public class PreRequestFilter extends ZuulFilter {
    
    //执行filter的类型
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    //执行的优先级，越小执行优先级越高
    @Override
    public int filterOrder() {
        return 0;
    }

    //是否启用当前过滤器
    @Override
    public boolean shouldFilter() {
        return true;
    }

    //
    @Override
    public Object run() throws ZuulException {
        //用于在过滤器之间传递消息
        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.set("startTime",System.currentTimeMillis());
        return null;
    }
}
```

```java
/**
 * 自定义过滤器打印客户端请求时间
 */
@Component
public class AccessLogFilter extends ZuulFilter {

     private final Logger log = LoggerFactory.getLogger(AccessLogFilter.class);

    @Override
    public String filterType() {
        return FilterConstants.POST_TYPE;
    }

    @Override
    public int filterOrder() {
        return FilterConstants.SEND_RESPONSE_FILTER_ORDER-1;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        Long startTime = (Long)ctx.get("startTime");
        HttpServletRequest request = ctx.getRequest();
        String uri = request.getRequestURI();
        long duration = System.currentTimeMillis()-startTime;
        log.info("uri: {}, duration: {}ms",uri,duration/100);
        return null;
    }
}
```

接下来我们进行测试，用网关访问后台服务

![](/images/image-20200222152953354.png)

![](/images/image-20200222153110038.png)

可以看到，通过网关访问后台用户服务获取用户信息的api成功，并且在Zuul服务的控制台中输出了请求的地址，请求执行时间信息。