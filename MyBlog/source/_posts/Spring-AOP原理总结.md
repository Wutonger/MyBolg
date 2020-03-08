title: Spring AOP原理总结（基于注解）
author: Loux

cover: /img/post/34.jpg

tags:

  - spring
categories:
  - 源码分析
date: 2020-03-08 20:14:00

---

相信在工作中，我们都用过Spring的面向切面编程，即降低了代码的耦合性又提升了开发维护效率。

最近这段时间研究了一下AOP中的一些基本的原理，在此记录下来，方便以后查阅。

# AOP的使用

AOP是一种在程序运行期间将某段代码切入到指定方法位置的编程方式，底层为动态代理。

通知方式有以下几种：

* 前置通知(@Before)   在目标方法运行之前运行
* 后置通知(@After)     在目标方法运行结束之后运行,无论正常结束还是异常结束
* 返回通知(@AfterReturning)     在目标方法正常返回之后运行
* 异常通知(@AfterThrowing)    在目标方法出现异常以后运行
* 环绕通知(@Around)

同时，需要将被切入的业务逻辑类与切面类都加入进Spring容器中，切面类上加入@Aspect注解告诉Spring这是切面类。

在Spring容器配置类中需要加入@EnableAspectJAutoProxy注解，启动注解模式的Aop功能 (Spring boot中不需要添加，默认开启该功能)

# AOP基本原理

主要分为两部分，第一部分为Spring容器的创建，第二部分为业务逻辑方法的执行

**一、Spring容器创建**

1) @EnableAspectJAutoProxy中利用@Import注入了一个AspectJAutoProxyRegistrar类型的bean ;

2) 而AspectJAutoProxyRegistrar实现了ImportBeanDefinitionRegistrar接口，利用registerBeanDefinitions()向容器中注入其它类型组件;

3)registerBeanDefinitions()执行过程中通过registerBeanPostProcessors()注册了AnnotationAwareAspectJAutoProxyCreator(后置处理器)注：AnnotationAwareAspectJAutoProxyCreator的父类实现了SmartInstantiationAwareBeanPostProcessor接口，所以该类本身也是一个postProcessor;

 4)容器创建其它Bean（业务逻辑Bean,切面Bean）;

5)AnnotationAwareAspectJAutoProxyCreator会拦截Bean的创建过程（初始化完成之后），判断组件是否需要增强;

 6)如果需要增强，会将切面通知方法包装为Advisor(增强器)，给业务逻辑对象创建代理对象(cglib or jdk动态代理方式)，并将代理对象保存在spring容器中;

**二、方法执行**

1)利用CglibAopProxy.intercept()拦截目标方法；

2)根据ProxyFactory对象获取将要执行目标方法的拦截器链(将增强器包装为拦截器MethodInterceptor)；

3)根据拦截器链式机制，依次进入每一个拦截器执行目标方法 前置通知->目标方法 -> 后置通知 -> 返回通知 or 异常通知；