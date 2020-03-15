title: Spring Bean的注入方式(基于注解)
author: Loux

cover: /img/post/36.jpg

tags:

  - spring
categories:
  - Spring系列
date: 2020-03-12 10:09:00

---

我们常常利用Spring的IOC容器来管理我们系统中的对象依赖。今天就来盘点一下在Spring中的Bean被添加到IOC容器中有哪些方法。

# @Configuration注解

类上加了@Configuration注解，相当于以前使用xml配置文件方式中的`<beans>`标签，该类被用作于配置Spring容器。

 ```java
@Configuration
public class MainConfig {
    
    @Lazy
    @Scope("singleton")
    @Bean(value = "teacher")
    public Person person(){
        return new Person("lisi",20);
    }

}
 ```

例如上面这种使用方式。

## @Bean注解

在上面的代码中，可以看到我们在person()方法上加了`@Bean`注解，它的作用是向Spring IOC容器注入一个bean，该bean的id默认为方法的名称，当然我们也可以通过value属性指定bean的名称。

同时，该bean也只会被Spring创建一次，也就是说默认是单例的。可以通过`@Scope`注解更改bean的作用域。

Bean的作用域有四种：

* prototype ：多实例
* singleton：单实例
* request：同一个请求创建一个实例
* session：同一个session创建一个实例

bean会在Spring容器启动时就被创建，如果想bean在需要用时才被创建，可以通过`@Lazy`注解实，也就是单例模式中的懒加载。

## @ComponentScan

除了使用@Bean注解外，我们还可以使用包扫描的方式。`@ComponentScan`注解会从需要扫描的路径中找到那些被标识为需要装配的类，并将它们自动装配到Spring容器中。

默认的扫描路劲为@ComponentScan注解标志的类所在的包及其子包下所有的类。我们可以通过它的value属性来自定义扫描路径。

那么在包扫描的过程中哪些类会被标识为需要装配的类呢？有下面四种：

* @Controller
* @Service
* @Repository
* @Component

相信在实际的开发中我们用的也不少。在Springboot项目的启动类上，标注了@SpringbootApplication注解，而查看该注解的源码我们可以发现，@ComponentScan注解就在其中。这也就是为什么在Springboot项目中，能自动识别我们加上的@Controller、@Service等注解，而不需要进行配置的原因了。

![@SpringbootApplication注解](/images/image-20200315110948801.png)

我们可以看到，在ComponentScan注解中还有一些属性，例如`excludeFilters`属性，它的作用是排除掉一些不需要扫描进Spring容器中的bean，它的值为@Filter数组。

当然还有`includeFilters`属性，它的作用是按照规则规定只需要包含哪些bean，使用方式与excludeFilters一样，需要设置@ComponentScan注解中的useDefaultFilter为false

```java
@Configuration
@ComponentScan(value = "controller",
        excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION,classes = Controller.class)})
public class MainConfig {

    @Bean(value = "teacher")
    public Person person(){
        return new Person("lisi",20);
    }

}
```

比如上述例子中，加了@Controller注解的类，就不会被自动装配到Spring容器。正是因为我们在excludeFilters中排除了该注解。其中我们可以根据注解来排除，也可以利用其它规则来进行排除

* FilterType.ANNOTATION    按照注解
* FilterType.ASSIGNABLE_TYPE   按照给定的类型
* FilterType.ASPECTJ   按照aspectj表达式
* FilterType.REGEX  按照正则表达式
* FilterType.CUSTOM   按照自定义规则

最重要的就是自定义规则了，而要实现自定义规则就必须实现`TypeFilter`接口，不信咱们看源码，里面说的明明白白。

![](/images/image-20200315140318116.png)

下面我们就来实现以下该接口

```java
public class MyTypeFilter implements TypeFilter {

    /**
     *
     * @param metadataReader   读取到的当前正在扫描类的信息
     * @param metadataReaderFactory   可以获取到其它任何类的信息
     * @return
     * @throws IOException
     */
    public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
        //获取当前类注解信息
        AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
        //获取当前正在扫描类的类信息
        ClassMetadata classMetadata = metadataReader.getClassMetadata();
        //获取当前类资源信息
        Resource resource = metadataReader.getResource();

        String className = classMetadata.getClassName();
        if (className.equals("bean.BookService")){
            return true;
        }

        return false;
    }
}
```

这里我就定义了一个规则，当该类的类全名为"blean.bookService"时，将它排除掉，不装配进IOC容器。要使用自定义规则也很简单，和上面的使用方式是一样的。

`@ComponentScan(value = "controller",
        excludeFilters = {@ComponentScan.Filter(type = FilterType.CUSTOM,classes = MyTypeFilter.class)})`

## @import注解

我们可以发现，包扫描的方式适合于自己编写的java类，而@Bean注解的方式适合于想要将第三方包的bean加入到容器中。

但是我们使用@Bean注解就要写一个方法，实在是麻烦，特别是有的时候方法中只有return new XXX();这种语句。Spring为我们提供了一种更加简便的方法，那就是@Import注解。

`@Import可以快速的向容器中导入一个bean`

```java
@Configuration
@Import({Color.class})
public class MainConfig2 {

}
```

上面这个例子，会向Spring容器中导入一个Color类型的bean，这个bean的id为Color类的类全名。

在@Import注解中，我们还能使用**ImportSelector**来一次性导入多个bean。首先我们需要实现ImportSelector接口。

```java
 /**
 * 自定义返回需要导入到Spring容器中的组件
 */
public class MyImportSelector implements ImportSelector {

    /**
     *
     * @param annotationMetadata 当前标注@Import注解类的所有注解信息
     * @return   需要导入容器的类的全类名
     */
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        return new String[]{"bean.Car","bean.Dog"};
    }
}
```

需要使用时，就这样使用：

`@Import({Color.class,MyImportSelector.class})`

这样，就能导入Car、Dog类型的bean到Spring容器中了。

在@Import中，**ImportBeanDefinitionRegistrar**也是十分常用的，可以实现该接口来向Spring容器中注入一个bean。

```java
public class MyImportBeanDefinitionRegister implements ImportBeanDefinitionRegistrar {

    /**
     *
     * @param annotationMetadata @Import注解所在类所有注解信息
     * @param beanDefinitionRegistry  bean定义的注册类,可通过它向容器中注入一个组件
     */
    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
        if (beanDefinitionRegistry.containsBeanDefinition("bean.Color")){
            beanDefinitionRegistry.registerBeanDefinition("effective java",new RootBeanDefinition(Book.class));
        }
    }
}
```

我们可以看到，我们定义的注入方法如下：如果Spring容器中注册了bean.Color类型的bean，那么就注入一个id为"effective java"、类型为Book的bean。

同样的，需要使用时，在@Import中加入它即可。

`@Import({Color.class,MyImportSelector.class,MyImportBeanDefinitionRegister.class})`

## FactoryBean

除了上面的几种注解的方式外，我们还可以实现FactoryBean接口来向Spring容器中注入bean实例。

```java
/**
 * 创建Spring中工厂bean
 */
public class ColorFactoryBean implements FactoryBean<Color> {

    /**
     * 创建一个Color对象添加到Spring容器中
     */
    public Color getObject() throws Exception {
        System.out.println("创建bean。。。。。。");
        return new Color();
    }
    
    /**指定bean的类型*/
    public Class<?> getObjectType() {
        return Color.class;
    }

    /**是否为单例*/
    public boolean isSingleton() {
        return true;
    }
}
```

在Spring的配置类中，进行配置

```java
@Configuration
public class MainConfig2 {
    @Bean
    public ColorFactoryBean getColor(){
        return new ColorFactoryBean();
    }
}
```

容器启动后，容器中就会存在Color类型的bean，id为getColor

如果我们需要获取工厂bean本身，需要在id前面加一个&标识，例如&getColor。

 



