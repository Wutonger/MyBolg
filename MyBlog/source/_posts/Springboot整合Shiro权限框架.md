title: Springboot整合Shiro权限框架
author: Loux
tags:
  - shiro
  - Springboot系列
categories:
  - 后端知识
cover: /img/post/5.jpg
date: 2019-11-27 16:05:00
---

网站的用户模块，免不了需要做权限管理，而Shiro框架则为我们提供了轻量级的用户认证、身份认证、加密等功能，为我们带来了极大的方便。

# 简介

 Apache Shiro 是 Java 的一个安全框架。目前，使用 Apache Shiro 的人越来越多，因为它相当简单，对比 Spring Security，可能没有 Spring Security 做的功能强大，但是在实际工作时可能并不需要那么复杂的东西，所以使用小而简单的 Shiro 就足够了。对于它俩到底哪个好，这个不必纠结，能更简单的解决项目问题就好了。 ——来自W3Cschool

但是需要注意的是，Shiro不会去维护用户、维护权限；这些需要我们自己去设计，然后通过相应的接口注入给Shiro。

# Shiro工作原理

在使用Shiro前，知道它的工作方式也很重要。那么在程序中Shiro是怎样完成它的工作的呢？

![Shiro工作原理](/images/image-20191127163103431.png)

可以看到，与代码直接交互的是Subject,也就是说Shiro对外的核心Api就是Subject;

*  **Subject**：主体，代表了当前 “用户”，这个用户不一定是一个具体的人，与当前应用交互的任何东西都是 Subject，如网络爬虫，机器人等；即一个抽象概念；所有 Subject 都绑定到 SecurityManager，与 Subject 的所有交互都会委托给 SecurityManager；可以把 Subject 认为是一个门面；SecurityManager 才是实际的执行者；  
*  **SecurityManager**：安全管理器；即所有与安全有关的操作都会与 SecurityManager 交互；且它管理着所有 Subject；可以看出它是 Shiro 的核心，它负责与后边介绍的其他组件进行交互，如果学习过 SpringMVC，你可以把它看成 DispatcherServlet 前端控制器； 
*  **Realm**：域，Shiro 从从 Realm 获取安全数据（如用户、角色、权限），就是说 SecurityManager 要验证用户身份，那么它需要从 Realm 获取相应的用户进行比较以确定用户身份是否合法；也需要从 Realm 得到用户相应的角色 / 权限进行验证用户是否能进行操作；可以把 Realm 看成 DataSource，即安全数据源。 

好了，简单的介绍之后，让我开始Springboot与Shiro的整合吧。

# Springboot整合Shiro

### 第一步—创建一个Springboot项目，maven的依赖如下：

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!--添加shrio相关依赖-->
        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-spring</artifactId>
            <version>1.4.0</version>
        </dependency>
        <!-- 添加lombock依赖-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        <!--添加Spring data Jpa依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>
```

pom文件的其它部分省略。

### 第二步—Springboot配置文件application.yml内容如下：

```yml
server:
  port: 8088
spring:
  application:
    name: shiro
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/shirocontrol?serverTimezone=UTC&useUnicode=true&characterEncoding=utf8
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    database: mysql
    showSql: true
    hibernate:
      ddlAuto: update
    database-platform: org.hibernate.dialect.MySQL5InnoDBDialect  #创建表采用InnoDB数据库引擎
```

### 第三步—用户、角色、权限实体类设计

经典的权限管理中，用户应该与角色关联，角色应该与权限关联，同时它们之间的关系应当为多对多关系，根据数据库设计三大范式，此时需要用户-角色中间表、角色-权限中间表。关于Spring Data Jpa相关的知识这里就不介绍了。

<b>SysUser类代码如下：</b>

```java
@Data
@Entity
@Table(name = "sys_user")
public class SysUser extends BaseEntity{

    private static final long serialVersionUID = -6407094407329594165L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  //主键自增策略
    private Long id;

    //用户名
    private String username;

    //密码
    private String password;

    //盐值字符串，用于md5加密
    private String salt;

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(name="sys_user_role",joinColumns = {@JoinColumn(name="uid")} ,inverseJoinColumns = {@JoinColumn(name="rid")})
    private List<SysRole> roles;

    public String getUniqueSalt() {
        return username + salt;
    }
}
```

<b>SysRole类代码如下：</b>

```java
@Data
@Entity
@Table(name = "sys_role")
public class SysRole extends BaseEntity{

    private static final long serialVersionUID = 1049246951726382288L;
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    //角色名称
    private String role;

    //角色与权限多对多关系
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(name = "sys_role_permission", joinColumns = { @JoinColumn(name = "rid") }, inverseJoinColumns = {
            @JoinColumn(name = "pid") })
    private List<SysPermission> permissions;

    //角色与用户多对多关系
    @ManyToMany
    @JoinTable(name = "sys_user_role", joinColumns = { @JoinColumn(name = "rid") }, inverseJoinColumns = {
            @JoinColumn(name = "uid") })
    private List<SysUser> sysUsers;
}
```

<b>SysPermission类代码如下：</b>

```java
@Data
@Entity
@Table(name = "sys_permission")
public class SysPermission extends BaseEntity{

    private static final long serialVersionUID = 5550136188011671184L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @ManyToMany
    @JoinTable(name = "sys_role_permission", joinColumns = { @JoinColumn(name = "pid") }, inverseJoinColumns = {
            @JoinColumn(name = "rid") })
    private List<SysRole> roles;

    //权限名称
    private String name;
}
```

这样配置过后，会自动生成两张中间表。

### 第四步—Shiro权限相关代码

首先，用户注册时，密码不应当明文存放到数据库，而是应该加密后存储到数据库。我们需要利用Shiro的加密功能对密码进行加密。加密工具类的代码如下：

```java
@Component
public class PasswordUtil {

    private RandomNumberGenerator randomNumberGenerator = new SecureRandomNumberGenerator();

    // 采用MD5信息摘要算法进行加密
    public static final String ALGORITHM_NAME = "md5";

     // 散列次数为3次
    public static final int HASH_ITERATIONS = 3;

    //对用户密码进行加密
    public void encryptPassword(SysUser user){
        //随机生成字符串作为salt;
        user.setSalt(randomNumberGenerator.nextBytes().toHex());
        //加密
        user.setPassword(new SimpleHash(ALGORITHM_NAME, user.getPassword(),
                ByteSource.Util.bytes(user.getUniqueSalt()), HASH_ITERATIONS).toHex());
    }

}
```

另外为了帮助Shiro能够正确为当前登陆用户做认证和赋权，我们需要实现自定义的Realm。具体来说就是实现doGetAuthenticationInfo和doGetAuthorizationInfo，这两个方法前者负责登陆认证后者负责提供一个权限信息。其代码如下：

```java
public class OwnAuthorizingRealm extends AuthorizingRealm {

    @Autowired
    private SysUserService sysUserService;

    /**
     * 用户权限校验
     * @param collection
     * @return
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection collection) {
        //用来记录用户具有哪些权限
        SimpleAuthorizationInfo authorizationInfo = new SimpleAuthorizationInfo();
        String username = (String) collection.getPrimaryPrincipal();
        //根具用户名查询到user
        SysUser user =sysUserService.findUserByName(username);
        user.getRoles().forEach( role -> {
            authorizationInfo.addRole(role.getRole());
            role.getPermissions().forEach( permission -> authorizationInfo.addStringPermission(permission.getName()));
        });
        return authorizationInfo;
    }

    /**
     * 登录认证校验
     * @param token
     * @return
     * @throws AuthenticationException
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token)
            throws AuthenticationException {
        String username = (String) token.getPrincipal();
        SysUser user = sysUserService.findUserByName(username);
        if (user == null){
            return null;
        }
        SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(user.getUsername(), user.getPassword(),
                ByteSource.Util.bytes(user.getUniqueSalt()), getName());
        return authenticationInfo;
    }
}
```

我们可以看到，`doGetAuthenticationInfo()`方法返回的authenticationInfo对象，并没有对散列算法与散列次数进行设置，那么Shiro是怎么做的呢？由于`AuthorizingRealm`类是一个抽象类，它提供了setCredentialsMatcher()方法来设置散列算法等信息，我们将在ShiroConfig中对其进行相关设置。

下面最重要的一部分，就是Shiro相关的配置代码

```java
@Configuration
public class ShiroConfig {
    @Bean
    public ShiroFilterFactoryBean shirFilter(SecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager);

        Map<String, String> filterChainDefinitionMap = new HashMap<String, String>();
        shiroFilterFactoryBean.setLoginUrl("/login");
        shiroFilterFactoryBean.setUnauthorizedUrl("/unauthc");
        shiroFilterFactoryBean.setSuccessUrl("/home/index");

        filterChainDefinitionMap.put("/*", "anon");
        filterChainDefinitionMap.put("/authc/index", "authc");
        filterChainDefinitionMap.put("/authc/admin", "roles[admin]");
        filterChainDefinitionMap.put("/authc/addAndUpdateUser", "perms[Create,Update]");
        filterChainDefinitionMap.put("/authc/removeUser", "perms[Delete]");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        return shiroFilterFactoryBean;
    }

    @Bean
    public HashedCredentialsMatcher hashedCredentialsMatcher() {
        HashedCredentialsMatcher hashedCredentialsMatcher = new HashedCredentialsMatcher();
        hashedCredentialsMatcher.setHashAlgorithmName(PasswordUtil.ALGORITHM_NAME); // 散列算法
        hashedCredentialsMatcher.setHashIterations(PasswordUtil.HASH_ITERATIONS); // 散列次数
        return hashedCredentialsMatcher;
    }

    @Bean
    public OwnAuthorizingRealm shiroRealm() {
        OwnAuthorizingRealm shiroRealm = new OwnAuthorizingRealm();
        shiroRealm.setCredentialsMatcher(hashedCredentialsMatcher());
        return shiroRealm;
    }

    @Bean
    public SecurityManager securityManager() {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        securityManager.setRealm(shiroRealm());
        return securityManager;
    }

    @Bean
    public PasswordUtil passwordHelper() {
        return new PasswordUtil();
    }
}
```

`ShiroFilter`方法中，为我们提供了很多filter来控制访问权限，内部也预定了多个过滤器，我们可以直接使用字符串配置这些过滤器。

常用的过滤器如下：

* authc：所有已登陆用户可访问

* roles：有指定角色的用户可访问，通过[ ]指定具体角色，这里的角色名称与数据库中配置一致

* perms：有指定权限的用户可访问，通过[ ]指定具体权限，这里的权限名称与数据库中配置一致

## 第五步—编写Controller类进行测试

首先我们预先将一些数据导入到数据库中

```sql
INSERT INTO `sys_permission` VALUES (1, 'Retrieve');
INSERT INTO `sys_permission` VALUES (2, 'Create');
INSERT INTO `sys_permission` VALUES (3, 'Update');
INSERT INTO `sys_permission` VALUES (4, 'Delete');
INSERT INTO `sys_role` VALUES (1, 'guest');
INSERT INTO `sys_role` VALUES (2, 'user');
INSERT INTO `sys_role` VALUES (3, 'admin');
INSERT INTO `sys_role_permission` VALUES (1, 1);
INSERT INTO `sys_role_permission` VALUES (1, 2);
INSERT INTO `sys_role_permission` VALUES (2, 2);
INSERT INTO `sys_role_permission` VALUES (3, 2);
INSERT INTO `sys_role_permission` VALUES (1, 3);
INSERT INTO `sys_role_permission` VALUES (2, 3);
INSERT INTO `sys_role_permission` VALUES (3, 3);
INSERT INTO `sys_role_permission` VALUES (3, 4);
```

MainController的代码如下：

```java
@RestController
@RequestMapping
public class MainController {

    @Autowired
    private SysUserService userService;
    @Autowired
    private PasswordUtil passwordHelper;

    @GetMapping({"login",""})
    public Object login() {
        return "Here is Login page";
    }

    @GetMapping("unauthc")
    public Object unauthc() {
        return "Here is Unauthc page";
    }

    @GetMapping("doLogin")
    public Object doLogin(@RequestParam String username, @RequestParam String password) {
        UsernamePasswordToken token = new UsernamePasswordToken(username, password);
        Subject subject = SecurityUtils.getSubject();
        try {
            subject.login(token);
        } catch (IncorrectCredentialsException ice) {
            return "用户名或密码错误，请重新输入";
        } catch (UnknownAccountException uae) {
            return "该用户不存在";
        }

        SysUser user = userService.findUserByName(username);
        subject.getSession().setAttribute("user", user);
        return "SUCCESS";
    }

    @GetMapping("register")
    public Object register(@RequestParam String username, @RequestParam String password) {
        SysUser user = new SysUser();
        user.setUsername(username);
        user.setPassword(password);
        passwordHelper.encryptPassword(user);
        try {
            userService.save(user);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "SUCCESS";
    }
}
```

AuthController的代码如下：

```java
@RestController
@RequestMapping("/authc")
public class AuthController {

    @GetMapping("/index")
    public Object index() {
        Subject subject = SecurityUtils.getSubject();
        SysUser user = (SysUser) subject.getSession().getAttribute("user");
        return user.toString();
    }

    @GetMapping("/admin")
    public Object admin() {
        return "Welcome Admin";
    }

    @GetMapping("/removeUser")
    public Object removable() {
        return "删除用户成功";
    }

    @GetMapping("/addAndUpdateUser")
    public Object renewable() {
        return "新增用户成功";
    }
}
```

其中省略SysUserService与SysUserDao的代码。

 这里为了简单，就不用页面进行验证了，直接使用Rest的方式。可以自行注册账号进行验证。