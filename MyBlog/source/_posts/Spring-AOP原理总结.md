title: Spring AOP原理总结（基于注解）
author: Loux
cover: /img/post/34.jpg
tags:
  - spring
categories:
  - Spring系列
  - ''
date: 2020-03-08 20:14:00
---

相信在工作中，我们都用过Spring的面向切面编程，即降低了代码的耦合性又提升了开发维护效率。用起来爽爽的。

以前只知道它底层使用了代理模式，不知道具体的原理是什么。最近这段时间刚好在梳理Spring框架系列的知识，于是深入了解了一下Spring AOP的基本实现过程。

# AOP的使用

Spring AOP到底是什么呢，或者我们应该怎样准确的描述该技术呢？

> Spring AOP是一种在程序运行期间将某段代码切入到指定方法位置的编程方式，底层为动态代理。

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

### Spring容器创建过程

第一步：@EnableAspectJAutoProxy中利用@Import注入了一个`AspectJAutoProxyRegistrar`类型的bean ;

![](/images/image-20200318144104557.png)

第二步：而`AspectJAutoProxyRegistrar`实现了`ImportBeanDefinitionRegistrar`接口，利用`registerBeanDefinitions()`向容器中注入其它类型组件;

![](/images/image-20200318144225107.png)

第三步：registerBeanDefinitions()注册了`AnnotationAwareAspectJAutoProxyCreator`类型的Bean,而它的父类实现了SmartInstantiationAwareBeanPostProcessor（一种后置处理器）接口，所以该类本身也是一个postProcessor，我们可以看到该类的继承结构

![](/images/image-20200318145456881.png)

第四步：在完成了PostProcessor的注册后，容器创建其它Bean（业务逻辑Bean,切面Bean）

第五步：AnnotationAwareAspectJAutoProxyCreator会拦截Bean的创建过程，在**Bean实例化之前**执行其父类AbstractAutoProxyCreator的`postProcessAfterInitialization()`方法来判断是否需要增强。该方法源码如下：

```java
    public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) 
    {
        if (bean != null) {
            //根据bean的类类型与beanName构建出cacheKey
            Object cacheKey = this.getCacheKey(bean.getClass(), beanName);
            //根据cacheKey判断是否需要创建指定的bean
            if (!this.earlyProxyReferences.contains(cacheKey)) {
                return this.wrapIfNecessary(bean, beanName, cacheKey);
            }
        }

        return bean;
    }

    protected Object getCacheKey(Class<?> beanClass, @Nullable String beanName) {
        if (StringUtils.hasLength(beanName)) {
            return FactoryBean.class.isAssignableFrom(beanClass) ? "&" + beanName : beanName;
        } else {
            return beanClass;
        }
    }
```

可以看到，重要的代码在这里`return this.wrapIfNecessary(bean, beanName, cacheKey);`，接下来我们看看wrapIfNecessary()方法的源码

```java
    protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
        //如果该bean已经被处理过，直接返回
        if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
            return bean;
        //如果该bean不需要进行AOP增强，直接返回    
        } else if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
            return bean;
        //如果该bean是基础设施类或者配置了跳过自动代理，则将cachekey添加到advisedBeans中
        //否则执行else if中的主逻辑
        } else if (!this.isInfrastructureClass(bean.getClass()) && !this.shouldSkip(bean.getClass(), beanName)) {
            //获取到增强器（增强器为对切面方法的包装）
            Object[] specificInterceptors = this.getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, (TargetSource)null);
            //如果增强器不为空，则根据beanName、增强器等创建代理对象返回
            if (specificInterceptors != DO_NOT_PROXY) {
                this.advisedBeans.put(cacheKey, Boolean.TRUE);
                //核心方法：创建代理对象并返回
                Object proxy = this.createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
                this.proxyTypes.put(cacheKey, proxy.getClass());
                return proxy;
            } else {
                this.advisedBeans.put(cacheKey, Boolean.FALSE);
                return bean;
            }
        } else {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return bean;
        }
    }
```

第六步：如果需要增强，会将切面通知方法包装为Advisor(增强器)，根据Advisor创建代理对象(cglib or jdk动态代理方式)，并将代理对象保存在spring容器中

获取增强器的流程在其子类AbstractAdvisorAutoProxyCreator中，源码如下：

```java
    protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
        //找到可用的增强器并返回，如果为空，返回空集合
        List<Advisor> advisors = this.findEligibleAdvisors(beanClass, beanName);
        return advisors.isEmpty() ? DO_NOT_PROXY : advisors.toArray();
    }

    protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {	
        //获取到所有的增强器
        List<Advisor> candidateAdvisors = this.findCandidateAdvisors();
        //根据beanClass与beanName在所有的增强器中找到该bean可用的增强器
        List<Advisor> eligibleAdvisors = this.findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
        this.extendAdvisors(eligibleAdvisors);
        //如果该bean可用的增强器不为空，将它进行排序
        if (!eligibleAdvisors.isEmpty()) {
            eligibleAdvisors = this.sortAdvisors(eligibleAdvisors);
        }

        return eligibleAdvisors;
    }
```



### 方法执行流程

第一步：在执行目标方法时，CglibAopProxy类的intercept()方法会先于目标方法执行

```java
		@Override
		public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			Object oldProxy = null;
			boolean setProxyContext = false;
			Class<?> targetClass = null;
			Object target = null;
			try {
				if (this.advised.exposeProxy) {
					// Make invocation available if necessary.
					oldProxy = AopContext.setCurrentProxy(proxy);
					setProxyContext = true;
				}
				target = getTarget();
				if (target != null) {
					targetClass = target.getClass();
				}
                  //获取目标方法将要执行的拦截器链
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				Object retVal;
                  //如果拦截器链为空，则通过反射执行目标方法
				if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = methodProxy.invoke(target, argsToUse);
				}
				else {
					// We need to create a method invocation...
                        //根据代理对象、目标对象、目标方法、参数、拦截器链等信息构建CglibMethodInvocation对象
                        //并执行对象的proceed()方法，即执行拦截器链
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				retVal = processReturnType(proxy, target, method, retVal);
				return retVal;
			}
			finally {
				if (target != null) {
					releaseTarget(target);
				}
				if (setProxyContext) {
					// Restore old proxy.
					AopContext.setCurrentProxy(oldProxy);
				}
			}
		}
```

第二步：根据ProxyFactory对象获取将要执行目标方法的拦截器链(将增强器包装为拦截器MethodInterceptor[])，有些增强器实现了MethodInterceptor接口，而有些则需要适配器转换

第三步：根据拦截器链式机制—递归调用，依次进入每一个拦截器。每一个拦截器总是等待下一个拦截器执行完成之后，再执行自身的方法，AOP精髓。

以图来表示该流程

![链式执行流程](/images/image-20200318232656888.png)

需要注意的是11那里，当执行返回通知异常时，就会被AspectJAfterThrowingAdvice 捕获，并在catch中执行异常通知。

`proceed()`方法源码如下：

```java
	public Object proceed() throws Throwable {
        //currentInterceptorIndex默认值为-1
		//如果没有拦截器，则直接执行目标方法
        //或者拦截器的索引与拦截器数量减1相等，也是执行目标方法
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}
        
		Object interceptorOrInterceptionAdvice =
		this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				return proceed();
			}
		}
		else {
			//递归调用
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
```

