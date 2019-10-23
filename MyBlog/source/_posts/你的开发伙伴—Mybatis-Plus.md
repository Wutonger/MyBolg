title: 你的开发伙伴—Mybatis-plus（一）
author: Loux
cover: /img/post/3.jpg
tags:

  - mybatis
categories:
  - 后端知识
date: 2019-03-01 19:28:00
---
目前最主流的持久层框架有两种，一种是Mybatis,一种是hibernate.而最近这些年来，为了加快开发效率，越来越多的公司选择使用mybatis作为持久层框架（上手更简单）。但是当我们使用mybatis时，无论是多简单的CRUD方法，都需要自己写出来（mybatis本身不提供对象化操作的CRUD方法）。使用过Spring Data JPA的人可能会羡慕它提供的这种功能，用来开发小型的系统特别简单快速。
#### Mybatis-plus简介
Mybatis-Plus（下面简称MP）只是一个mybatis的增强工具，它只做增强不做改变，什么意思呢？也就是说当引入MP后你原有的项目不会产生任何影响，它仅仅只是对你的开发提供帮助，同时由于在容器刚生成时MP就一起被初始化了，所以在使用中它和mybatis相比的效率损耗非常小。它与mybatis的关系就如同魂斗罗的1p和2p一样，起到互相帮助的作用。
![mybatis-plus与mybatis关系](/images/mybatisplus1.png)
****
#### Mybatis-plus特性
- 无侵入：只做增强不做改变，引入它不会对现有工程产生影响，如丝般顺滑
- 损耗小：启动即会自动注入基本 CURD，性能基本无损耗，直接面向对象操作
- 强大的 CRUD 操作：内置通用 Mapper、通用 Service，仅仅通过少量配置即可实现单表大部分 CRUD 操作，更有强大的条件构造器，满足各类使用需求
- 支持 Lambda 形式调用：通过 Lambda 表达式，方便的编写各类查询条件，无需再担心字段写错
- 支持主键自动生成：支持多达 4 种主键策略（内含分布式唯一 ID 生成器 - Sequence），可自由配置，完美解决主键问题
- 支持 ActiveRecord 模式：支持 ActiveRecord 形式调用，实体类只需继承 Model 类即可进行强大的 CRUD 操作
支持自定义全局通用操作：支持全局通用方法注入（ Write once, use anywhere ）
- 内置代码生成器：采用代码或者 Maven 插件可快速生成 Mapper 、 Model 、 Service 、 Controller 层代码，支持模板引擎，更有超多自定义配置等您来使用
- 内置分页插件：基于 MyBatis 物理分页，开发者无需关心具体操作，配置好插件之后，写分页等同于普通 List 查询
- 分页插件支持多种数据库：支持 MySQL、MariaDB、Oracle、DB2、H2、HSQL、SQLite、Postgre、SQLServer2005、SQLServer 等多种数据库
- 内置性能分析插件：可输出 Sql 语句以及其执行时间，建议开发测试时启用该功能，能快速揪出慢查询
- 内置全局拦截插件：提供全表 delete 、 update 操作智能分析阻断，也可自定义拦截规则，预防误操作
****
#### 快速入门
1.新建一个idea中新建一个Springboot项目，除了自动生成的pom文件中的依赖外，再引入以下的依赖  
``` bash
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.1.2</version>
        </dependency>
```
application.yml中的内容如下
``` bash
server:
     port: 8888
spring:
     datasource:
     driver-class-name: com.mysql.cj.jdbc.Driver
     url: jdbc:mysql://localhost:3306/owntest?serverTimezone=UTC&useUnicode=true&characterEncoding=utf8&useSSL=false
    username: root
    password: root
```
2.数据库新建数据表user,建表语句及表的内容如下  

``` bash
DROP TABLE IF EXISTS user;
CREATE TABLE user
(
	id BIGINT(20) NOT NULL COMMENT '主键ID',
	name VARCHAR(30) NULL DEFAULT NULL COMMENT '姓名',
	age INT(11) NULL DEFAULT NULL COMMENT '年龄',
	email VARCHAR(50) NULL DEFAULT NULL COMMENT '邮箱',
	PRIMARY KEY (id)
);
--加入内容--
DELETE FROM user;
INSERT INTO user (id, name, age, email) VALUES
(1, 'Jone', 18, 'test1@baomidou.com'),
(2, 'Jack', 20, 'test2@baomidou.com'),
(3, 'Tom', 28, 'test3@baomidou.com'),
(4, 'Sandy', 21, 'test4@baomidou.com'),
(5, 'Billie', 24, 'test5@baomidou.com');
```
3.新建User类对象和UserDao接口  

``` bash
package cn.lxfun.mybatisplust.entity;
import lombok.Data;
//加入lombok,自动生成getter/setter和toString方法
@Data
public class User {
    private long id;
    private String name;
    private Integer age;
    private String email;
}
```
``` bash
//继承了mybatis-plus中的BaseMapper类，自动实现了CRUD方法
public interface UserDao extends BaseMapper<User> {
}
```
4.Springboot启动类中加入@MapperScan注解，扫描DAO接口
``` bash
@SpringBootApplication
@MapperScan("cn.lxfun.mybatisplust.dao")
public class MybatisplustApplication {
    public static void main(String[] args) {
        SpringApplication.run(MybatisplustApplication.class, args);
    }
}
```
5.编写测试类进行测试
``` bash
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserDaoTest {
    @Autowired
    private UserDao userDao;
    @Test
    public void testSelect(){
        System.out.println(("----- selectAll method test ------"));
        List<User> userList = userDao.selectList(null);
        Assert.assertEquals(5, userList.size());
        //遍历输出list中的内容
        userList.forEach((item) -> {
            System.out.println(item);
        });
    }
}
```
console中输出结果如下:  

![控制台输入结果](/images/mybatisplus2.png)
****
#### 案例小结
可以看到，使用了MP后对一些简单的CRUD操作我们不必再在xml文件中编写SQL语句了，对于开发效率来说提高了不少。当然MP的核心内容可不止这一点，后续还会对MP的其它特性陆续进行讲解。