title: Springboot整合Redis实现消息队列
author: Loux
cover: /img/post/29.jpg
tags:
  - Springboot系列
  - Redis
categories:
  - 后端知识
date: 2020-01-03 15:16:00
---

在去年我更新了一系列的文章来介绍RabbitMQ这种消息队列的使用方法。

![](/images/image-20200103152955692.png)

但是有的时候我们需要实现一些简单的队列其实不必引入额外的MQ。

我相信在开发中都使用过Redis作缓存，其实它也是可以做消息队列的。