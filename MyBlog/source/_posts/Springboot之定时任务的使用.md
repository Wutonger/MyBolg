title: Springboot之定时任务的使用
author: Loux
tags:
  - 后端知识
categories:
  - 后端知识
date: 2019-12-26 14:53:00

---

在日常的开发中，由于业务的需要，我们经常需要使用定时任务去处理一些逻辑。例如mysql的备份、redis缓存的更新等等。

Quartz、JDK自带的Timer都能够实现定时任务，但是目前Springboot被使用的越来越多，Springboot就自带了处理定时任务的工具Spring Task。能够不引入其它依赖并且能实现比Timer更强大的功能，岂不是美滋滋？

主流的使用方式有以下两种：

* 基于@Scheduled注解
* 实现SchedulingConfigurer接口

首先，无论是哪种方式，我们都要让Springboot知道我们需要使用定时任务模块的功能。在应用程序的启动类上加上`@EnableScheduling注解`

![](/images/image-20191226152559499.png)

# 基于@Scheduled注解

首先，贴出一个简单的例子说明@Scheduled的使用有多简单

```java
@Component  //注入进Spring容器
public class ScheduleTask {

    @Scheduled(cron = "0/5 * * * * ?")
    public void doTask(){
        System.out.println("现在的时间是："+ LocalDateTime.now());
    }
}
```



Springboot应用启动后，观察控制台，我们会发现输出

![](/images/image-20191226153614340.png)

我们可以看到，每隔5秒钟，doTask()方法就会自动执行一次。

接下来我们来说一下`@Scheduled`注解有哪些参数

## cron

cron参数接收一个cron表达式。cron表达式是一个字符串，以5个或者6个空格隔开，分开共6个或7个域，每一个域代表不同的涵义。

cron表达式大致如下：

[秒] [分] [小时] [日] [月] [周] [年]，其中“年”这个参数是可以省略的

cron表达式的语法这里就不详细介绍了。有人说我不会写cron表达式怎么办？简单，网上有自动生成cron表达式的网站，只要输入你需要执行的时间就能自动生成。网址为<a href="http://cron.qqe2.com/">cron表达式在线生成器</a>。

## zone

zon参数接收一个时区参数(java.util.TimeZone)，cron表达式会根据该时区参数来解析。一般我们不填这个参数，它会默认按照服务器的时区来解析。

## fixedDelay

上一次执行完毕后隔多长时间执行,如：

`@Scheduled(fixedDelay = 1000) //上一次执行完毕时间点之后1秒再执行`

## fixedDelayString

与fixedDelay一样，只是该接收的参数为字符串，可以获取配置文件中的属性,如：

`@Scheduled(fixedDelay = "1000") //上一次执行完毕时间点之后1秒再执行`

`@Scheduled(fixedDelayString = "${time.fixedDelay}")`

