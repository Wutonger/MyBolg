title: RabbitMQ简单队列—消息的生产与消费
author: Loux
cover: /img/post/8.jpg
tags:
  - RabbitMQ
  - 消息队列
categories:
  - 后端知识
date: 2019-07-05 21:25:00
---
RabbitMQ简单队列十分的容易理解，分为消息的生产者与消费者，并且要注意的是一个生产者对应一个消费者。模型如下图所示：

![简单队列模型](/images/pasted-0.png)
生产者将“hello”这个消息发送到队列中，消费者从队列中接受该消息。  
#### 引入依赖
对RabbitMQ的安装与配置就不予赘述了，在Java代码中要使用RabbitMQ需要引入以下的依赖
```bash
    <dependency>
      <groupId>com.rabbitmq</groupId>
      <artifactId>amqp-client</artifactId>
      <version>5.6.0</version>
    </dependency>
```
#### 建立连接对象
这里的连接对象类似于操作数据库时JDBC中的Connection对象，需要指明RabbitMQ所在服务器的的IP+Port,以及VirtualHost（可以通俗的理解为数据库库名）,最重要的是用户名与密码。建立一个简单的RabbitMQ连接的单例类，代码如下
```bash
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import java.io.IOException;
import java.util.concurrent.TimeoutException;
public class ConnectionUtil {
    //创建连接的单例对象
    private static Connection connection=null;

    /**
     * 对外部提供唯一获取连接的方法
     * @return
     */
    public static Connection getConnection() throws IOException,TimeoutException {
        //创建一个连接工厂
        ConnectionFactory connectionFactory = new ConnectionFactory();
        //设置服务器地址
        connectionFactory.setHost("127.0.0.1");
        //amqp协议
        connectionFactory.setPort(5672);
        //设置连接的VirtualHost
        connectionFactory.setVirtualHost("/louxhost");
        //设置连接的用户名与密码
        connectionFactory.setUsername("loux");
        connectionFactory.setPassword("123456");
        if(connection!=null){
            return connection;
        }
        //工厂创建连接时需要抛出连接超时异常
        return connectionFactory.newConnection();
    }
}
```
#### 创建消息的生产者
要向队列中发送消息，分为下面几步  
1. 获取连接至RabbitMQ服务器的Connection对象
2. 从Connection中获取一个Channel
3. 从Channel中建立一个队列的声明
4. 消息的定义与发送
5. 关闭Channel与Connection
建立生产者的Java代码如下  

```bash 
import com.lx.util.ConnectionUtil;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
public class SimpleQueue {
    private static final String  QUEUE_NAME = "simpleQueue";
    public static void main(String[] args) throws Exception{
        //获取一个连接
        Connection connection = ConnectionUtil.getConnection();
        //从连接中获取一个通道
        Channel channel = connection.createChannel();
        //从管道中创建队列声明
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        //定义消息
        String msg = "hello rabbitmq!";
        //发送消息
        channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
        //关闭管道和连接
        channel.close();
        connection.close();
    }
}
```
#### 创建消息的消费者
创建消息的消费者前面两个步骤与创建生产者相同。当创建完Channel对象后，需要创建一个DefaultConsumer对象，并重写它的handleDelivery()方法，handleDelivery()方法中编写消费消息的内容。最后Channel需要监听Consumer,即从哪个队列中消费消息。建立消费者的Java代码如下

```bash
import com.lx.util.ConnectionUtil;
import com.rabbitmq.client.*;

import java.io.IOException;

/**
 * 消费消息
 */
public class RecvMessage {
    public static void main(String[] args) throws Exception {
        //获取一个连接
        Connection connection = ConnectionUtil.getConnection();
        //从连接中获取一个通道
        Channel channel = connection.createChannel();

        DefaultConsumer defaultConsumer = new DefaultConsumer(channel){
            //重写方法，事件模型，一旦消息进入队列就会触发
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msgString = new String(body,"utf-8");
                System.out.println("recived:"+msgString);
            }
        };
        //监听队列
        channel.basicConsume("simpleQueue",true,defaultConsumer);
     }
    }
```
到此为止，一个RabbitMQ的简单队列消息的生产与消费就完成了。运行生产者main()方法一次就会向队列中发送一条“hello rabbitmq!”消息，多次运行会发送多条消息到队列中。而运行消费者main()方法后会消费队列中所有的消息，并进行监听，一旦队列中进入了新的消息，马上进行消费。console内容如下
![结果](/images/console.png)
