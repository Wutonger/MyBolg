title: Spring cloud—注册中心Eureka
author: Loux
cover: /img/post/6.jpg
tags:
  - spring cloud
categories:
  - 微服务
date: 2019-08-11 16:41:00
---
#### 为什么使用Spring cloud
* Spring cloud是Spring封装的一系列微服务解决方案，它是一站式的，使用和配置都很方便。
* 身后有着Spring团队强大的支持
* 社区活跃，技术热度很高，遇到棘手的问题能更快速的在网上寻求解决  

#### 什么是Eureka
Eureka是Netflix开发的一款服务的发现与注册框架，它本身是一个基于Rest的服务。Spring将它集成在了子项目spring-cloud-netflix中。  
> Eureka采用了C-S的设计架构，Eureka Server作为服务注册与发现的服务器，它是整个系统的注册中心。  
系统中的其它微服务使用Eureka Client与注册中心连接并维持心跳(默认周期为30s)，这样Eureka就知道那些服务是正常的，哪些服务出现了问题。如果Eureka Server在多个心跳的周期内没有收到某个服务的心跳，Eureka Server将会把它从注册中心里移除。  

下图为Eureka的基本工作流程

![Eureka基本工作流程](/images/pasted-16.png)

#### 搭建注册中心
##### 创建项目
使用idea快速创建一个Empty Project,在Project中创建一个Springboot Module，等待初始化完成。
##### 更改pom文件
```
    ......省略
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Finchley.M7</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    .......省略   
```
##### 更改application.yml配置文件
```
server:
  port: 8100
eureka:
  instance:
    hostname: eureka1
    prefer-ip-address: false
  client:
    service-url:
      defaultZone: http://eureka1:8100/eureka/
    register-with-eureka: false  #由于本身为注册中心，不需要将自己注册进注册中心
    fetch-registry: false  #由于本身为注册中心，不需要搜索发现服务
spring:
  application:
    name: microservice-eureka
```
这里的eureka.instance.hostname我在hosts文件中做了映射，即eureka1 -> 127.0.0.1。
##### 编写启动类
```
/**
 * server 服务端
 */
@EnableEurekaServer
@SpringBootApplication
public class SpringcloudEurekaApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringcloudEurekaApplication.class, args);
    }

}
```
##### 启动项目
正常启动后访问localhost:8100,出现以下界面说明配置中心搭建成功。

![启动成功](/images/pasted-17.png)
因为目前没有任何微服务注册进Eureka，所以我们可以看到列表中的服务为空。
#### 服务的注册
##### 创建Moudule
再创建一个Springboot Module,作为一个服务注册到注册中心去。
##### 更改pom文件
pom文件和上一个唯一的区别就是
```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```
##### 更改application配置文件
```
server:
  port: 8000
spring:
  application:
    name: microservice-member  #指定服务的名称，在Eureka中通过该名称去搜索服务
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8100/eureka
    register-with-eureka: true
    fetch-registry: true
```
##### 编写启动类
```
/**
 * client 客户端
 */
@EnableEurekaClient
@SpringBootApplication
public class SpringcloudMemberApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringcloudMemberApplication.class, args);
    }
}
```
##### 启动该服务
启动后Eureka图形界面如下

![服务列表](/images/pasted-18.png)
我们可以发现服务列表中成功的出现了该服务的基本信息，且状态为UP,说明该服务成功的注册到了Eureka注册中心。
#### 搭建高可用Eureka集群
 在分布式系统中，任何的地方存在单点，整个体系就不是高可用的，Eureka 也一样。在实际的项目中，如果Eureka 服务器只有一个，那么当该主机宕机后会导致整个系统的注册中心挂掉，其它的业务服务无法被注册，也无法被发现，导致整个系统的瘫痪，所以说我们必须采取一些措施来防止该情况的发生。使用Eureka集群可以有效的解决这一问题。即多台Eureka Server服务器，互相同步数据，当一台挂掉后，另外的服务器还能正常工作，以达到高可用的目标。
Eureka集群的搭建其实很简单，就是做到“你中有我，我中有你”。什么意思呢？例如两个Eureka Server服务，在启动时分别将其注册到对方的注册中心去，这样当挂掉一个Eureka Server后，另外一台服务器能够顶替它的工作。三台，四台集群的原理其实和两台也是一样的。下面具体来实现一个简单的双服务器集群。
##### 更改Eureka1与Eureka2的application.yml配置文件
<b>Eureka1</b>
```
server:
  port: 8100
eureka:
  instance:
    hostname: eureka1
    prefer-ip-address: false
  client:
    service-url:
      defaultZone: http://eureka2:9100/eureka/
    register-with-eureka: true
    fetch-registry: true
spring:
  application:
    name: microservice-eureka
```
<b>Eureka2</b>
```
server:
  port: 9100
eureka:
  instance:
    hostname: eureka2
    prefer-ip-address: false
  client:
    service-url:
      defaultZone: http://eureka1:8100/eureka/
    register-with-eureka: true
    fetch-registry: true
spring:
  application:
    name: microservice-eureka
```
<b>其它服务需要同时注册到两个Eureka1、Eureka2服务</b>
```
server:
  port: 8000
spring:
  application:
    name: microservice-member
eureka:
  client:
    service-url:
      defaultZone: http://eureka1:8100/eureka,http://eureka2:9100/eureka
    register-with-eureka: true
    fetch-registry: true
```
启动Eureka1、Eureka2两个服务，这里只贴出Eureka1的界面，其实Eureka2的界面与它类似
![集群启动](/images/pasted-19.png)
DS Replicas说明该注册中心从哪里同步信息，从图上可以看到Eureka1从Eureka2那里同步信息（8100端口为Eureka1,9100端口为Eureka2）