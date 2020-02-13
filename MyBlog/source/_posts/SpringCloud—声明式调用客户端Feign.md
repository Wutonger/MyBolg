title: SpringCloud—声明式调用客户端Feign
author: Loux
cover: /img/post/32.jpg
tags:
  - spring cloud
categories:
  - 微服务
date: 2020-02-06 20:40:00

---

自从去年中旬写了Eureka的文章之后，已经半年多没有更新Springcloud组件的文章了。说来真是惭愧啊，主要的原因就是太懒了。。。。最近由于疫情影响，被迫在家上班，摸鱼的时间更多了，决定继续写完Springcloud常用的组件的使用。

# 什么是Feign

Feign是一个声明式WebService客户端。使用Feign能让微服务之间的调用更加简单。而简单的原因正是因为它的“声明式”，在使用feign的时候只用定义一些接口，接口上加上Feign定义的注解，即可实现跨服务之间的调用。比使用RestTemplate或其它远程调用工具更加的方便，受到了越来越多开发者的喜爱。Spring Cloud对Feign进行了封装,使其支持了Spring MVC标准注解，同时还在其中自动集成了负载均衡Ribbon，我们只需要引入一个Feign就能实现微服务调用和负载均衡两个功能。

# 开始使用

先说明，这里我使用的Springcloud的版本是`Greenwich.RELEASE`

我们首先创建一个用户服务名为homepage-user，在服务中，应当加入以下依赖

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
```

进入spring-cloud-starter-openfeign的pom文件，我们可以看到，它集成了ribbon

![image-20200213141359630](/images/image-20200213141359630.png)

同时在服务的启动类上应当加上`@EnableFeignClients`注解，例如这样

```java
@SpringbootApplication
@@EnableDiscoveryClient  //使得该服务能被注册中心发现并注册
@EnableFeignClients      //加上这个注解，才能使用feign
@EnableJpaAuditing
public class HomePageUserApplication {

    public static void main(String[] args) {
        SpringApplication.run(HomePageUserApplication.class,args);
    }
}
```

这里我有一个课程服务homepage-course。它有一些api如下：

```java
@Slf4j
@RestController
public class HomePageCourseController {

    @Value("${server.port}")
    private String port;

    @Autowired
    private ICourseService courseService;

    @GetMapping("get/port")
    public String getPort(){
       return port;
    }

    @GetMapping("get/course")
    public CourseInfo getCourseInfo(Long id){
        log.info("<homepage-course>: get course -> {}", id);
        return courseService.getCourseInfo(id);
    }

    @PostMapping("get/courses")
    public List<CourseInfo> getCourseInfos(@RequestBody CourseInfosRequest request){
        log.info("<homepage-course>: get courses -> {}" , JSON.toJSONString(request));
        return courseService.getCourseInfos(request);
    }
}
```

课程服务的配置文件中关键内容大概如下：

```yml
server:
  port: 7001
  servlet:
    context-path: /homepage-course
spring:
  application:
    name: eureka-client-homepage-course
eureka:
  client:
    service-url:
      defaultZone: http://server1:8000/eureka/
```

一切准备完成，接下来我们要在用户服务中调用课程服务的api，根据Feign的官方定义，它是以接口方式工作的，我们应该创建一个接口

```java
/**
 * 通过Feign访问课程微服务
 */
@FeignClient(value = "eureka-client-homepage-course")  //指定服务名
public interface CourseClient {

    @GetMapping(value = "/homepage-course/get/port")
    String getPort();

    @GetMapping(value = "/homepage-course/get/course")
    CourseInfo getCourseInfo(Long id);

    @PostMapping(value = "/homepage-course/get/courses")
    List<CourseInfo> getCourseInfos(@RequestBody CourseInfosRequest request);

}
```

在接口上，添加了@FeignClient注解，标识这个接口是一个Feign客户端。value的值也就是需要调用的服务名称。在这里我们课程微服务的名称为eureka-client-homepage-course。

同时接口定义的方法，需要和课程微服务提供的api相同，并在上面加上api的访问地址，即完成了课程微服务的调用。

同时Fegin自动集成了Ribbon，实现了负载均衡。默认的策略是轮询机制，即若被调用的微服务部署在服务器A,B,C上，则第一次调用时会调用A、第二次会调用B、第三次会调用C、第四次会调用A......依次循环。

当然如果我们想用其它的负载均衡策略也可以在配置文件中设置,在配置文件中新添加以下内容：

```yml
eureka-client-homepage-course:
  ribbon:
    #NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #配置规则 随机
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule #配置规则 轮询
    #NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RetryRule #配置规则 重试
    #NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule #配置规则 响应时间权重
    #NFLoadBalancerRuleClassName: com.netflix.loadbalancer.BestAvailableRule #配置规则 最空闲连接策略
    ConnectTimeout: 500 #请求连接超时时间
    ReadTimeout: 1000 #请求处理的超时时间
    OkToRetryOnAllOperations: true #对所有请求都进行重试
    MaxAutoRetriesNextServer: 2 #切换实例的重试次数
    MaxAutoRetries: 1 #对当前实例的重试次数
```

其中eureka-client-homepage-course即是被调用方的服务名称，即该设置对eureka-client-homepage-course这个服务是有效的。我们可以看到，除了轮询之外，还有随机、重试、相应时间权重、最空闲连接策略这几种方式。根据名称我们都能理解其含义。

接下来，我们测试一下效果，先在用户服务中编写一个Controller，获取课程服务的端口号，代码如下：

```java
@Slf4j
@RestController
public class HomepageUserController {
    
    /** 注入刚刚编写的Feign客户端*/
    @Autowired
    private CourseClient courseClient;

    @GetMapping("/get/coursePort")
    public String getCoursePort(){
        return courseClient.getPort();
    }
}
```

我们在7000端口启动用户服务，分别在7001，7002端口上启动两个课程服务。

访问路劲`http://localhost:7000/homepage-user/get/coursePort`

第一次访问结果：

![](/images/image-20200213144606970.png)

第二次访问结果：

![](/images/image-20200213144636956.png)

从结果可以看到，我们使用feign调用课程微服务的api成功了，同时还有负载均衡策略（这里我们使用的是轮询策略）。