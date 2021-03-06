title: HTTP协议常用方法及响应状态码
author: Loux
cover: /img/post/12.jpg
tags:
  - 网络协议
  - http/https
categories:
  - 网络协议
date: 2018-12-24 10:41:00
---
#### 简介
在面试和笔试中，我们总会遇见关于HTTP协议相关的内容，因此为了掌握知识，同时也为了以后的面试，这里对其进行一个简单的记录。
根据HTTP标准，HTTP请求可以使用多种请求方法。  
HTTP1.0定义了三种请求方法： GET, POST 和 HEAD方法。  
HTTP1.1新增了五种请求方法：OPTIONS, PUT, DELETE, TRACE 和 CONNECT 方法。
****

#### 常用方法
| num | 方法名     | 方法描述                                       
|---|---------|---------------------------------------------------------------
| 1 | GET     | 请求指定的页面信息，并返回实体主体 
| 2 | HEAD    | 类似于GET请求，只不过返回的响应中没有具体的内容，用于获取报头|
| 3 | POST    | 向指定资源提交数据进行处理请求（例如提交表单或者上传文件）。数据被包含在请求体中。POST请求可能会导致新的资源的建立和/或已有资源的修改 |
| 4 | PUT     | 从客户端向服务器传递数据用来取代指定文档的内容                                                                                        |
| 5 | DELETE  | 请求服务器删除指定的资源                                                                                                              |
| 6 | CONNECT | 要求在与代理服务器通信时建立隧道，实现用隧道协议进行TCP通信。主要使用SSL和TSL协议把通信内容加密后经网络隧道传输                       |
| 7 | OPTIONS | 用来查询对于指定url的资源提供支持的方法                                                                                               |
| 8 | TRACE   | 让Web服务器端将之前的请求通信回显给客户端。客户端可以用TRACE方法查询发送出去的请求时怎样被加工修改的。不常用，还容易引发XST攻击       |
****

#### GET和POST方法的区别


对于我们程序员来说，平时开发过程中使用的最多的两种请求就是GET和POST请求。对于增删改查四种操作类型来讲，大体可以对应如下：

GET 查询 POST 增加 PUT 修改 DELETE 删除

**GET**：使用GET方法时，参数通常会通过?附加到url地址后面一起发送到服务器，多个参数用&符号分割。GET请求特点如下：

-   GET请求能够被缓存

-   GET请求具有长度限制

-   GET请求安全性不高，主要用于获取数据

-   GET请求产生两个TCP包，即将header和data一起发送出去

**POST**：使用POST方法时，参数存储于请求的request
body中，所以在url上是不可见的。GET请求的特点如下：

-   POST请求不能被浏览器主动缓存，除非手动设置

-   POST请求没有长度限制

-   POST请求更加的安全

-   POST请求产生两个TCP数据包，先发送header,服务器响应100
    continue,浏览器再发送data,服务器响应200 success

但是实际上，GET和POST均为HTTP协议中两种发送请求的方法，而HTTP又是基于TCP/IP的，也就是说
GET和POST都是TCP链接，它们两个实际上并没有本质上的区别。HTTP只是一种规则，而TCP才是GET和POST的基本。GET和POST本质上就是TCP链接，并无差别。但是由于HTTP的规定和浏览器/服务器的限制，导致他们在应用过程中体现出一些不同。
****

#### HTTP状态码

当浏览者访问一个网页时，浏览者的浏览器会向网页所在服务器发出请求。当浏览器接收并显示网页前，此网页所在的服务器会返回一个包含HTTP状态码的信息头（server
header）用以响应浏览器的请求。HTTP状态码的英文为HTTP Status
Code。状态码分类如下：

| 状态码 | 状态码描述   |
|-----|------------------------------------------------|
| 1xx | 信息，服务器收到请求，需要请求者继续执行操作   |
| 2xx | 成功，操作被成功接收并处理                     |
| 3xx | 重定向，需要进一步的操作以完成请求             |
| 4xx | 客户端错误，请求包含语法错误或无法完成请求     |
| 5xx | 服务器错误，服务器在处理请求的过程中发生了错误 |

下面是常见的HTTP状态码

-   100 - Continue 初始的请求已经接受，客户应当继续发送请求的其余部分。

-   200 - OK  一切正常，对GET和POST请求的应答文档跟在后面。

-   304 - Not Modified 
    客户端有缓冲的文档并发出了一个条件性的请求。服务器告诉浏览器原来的请求还可以继续使用。

-   400 - Bad Request 请求出现语法错误，例如请求参数和需要接受的参数不对应。

-   403 - Forbidden
    资源不可用，服务器理解浏览器的请求，但拒绝它。通常是用于服务器上文件目录设置有权限导致的。

-   500 - Internal Server Error 服务器内部错误，不能完成浏览器的请求。

更多的HTTP状态码可以在 <https://www.runoob.com/http/http-status-codes.html>
进行查看。