title: RabbitMQ订阅模式—路由与主题
author: Loux
cover: /img/post/10.jpg
tags:
  - RabbitMQ
  - 消息队列
categories:
  - 后端知识
img: /medias/featureimages/5.jpg
date: 2019-07-13 22:24:00
---
在前面的工作队列中，每一条消息只能被一个消费者消费。如果我们想要一条消息被多个消费者消费就要使用到RabbitMQ的订阅模式。即先将消息发送到<b>exchange</b>(交换机)，队列绑定交换机后，交换机会将消息转发到队列之中，消费者再从队列中获取消息进行消费。订阅模式官方示例图如下：

![订阅模式](/images/pasted-4.png)

在RabbitMQ中交换机有三种类型
* fanout exchange：不处理路由键，即消息会被转发到绑定该交换机的所有队列

![fanout exchange](/images/pasted-5.png)
* direct exchange：消息中的routing key（路由键）如果和 Binding 中的 binding key 一致， 交换器就将消息发到对应的队列中。路由键与队列名完全匹配，如果一个队列绑定到交换机要求路由键为“goods”，则只转发 routing key 标记为“goods”的消息，不会转发“orders”，也不会转发“goods.insert”等等。它是完全匹配、单播的模式。

![direct exchange](/images/pasted-6.png)
* topic exchange：topic 交换器通过模式匹配分配消息的路由键属性，将路由键和某个模式进行匹配，此时队列需要绑定到一个模式上。它将路由键和绑定键的字符串切分成单词，这些单词之间用点隔开。同时它也会识别两个通配符“*”和“#”。其中#表示匹配多个或0个单词，*表示匹配一个单词。

![topic exchange](/images/pasted-7.png)
#### 订阅模式
订阅模式采用的交换机类型为fanout exchange,订阅模式java代码示例为：
<b>生产者：</b>
```java
public class Send {
    private static final String EXCHANGE_NAME= "test_exchange_fanout";
    public static void main(String[] args) throws IOException, TimeoutException {
        //创建连接对象
        Connection connection = ConnectionUtil.getConnection();
        //创建通道
        Channel channel = connection.createChannel();
        //声明交换机与交换机类型
        channel.exchangeDeclare(EXCHANGE_NAME,"fanout");
        //发送消息
        String msg = "hello pb";
        channel.basicPublish(EXCHANGE_NAME,"",null,msg.getBytes());
        channel.close();
        connection.close();
    }
}
```
<b>消费者：</b>
```java
public class Customer {
    private static final String QUEUE_NAME= "test_queue_fanout_email";
    private static final String EXCHANGE_NAME= "test_exchange_fanout";
    public static void main(String[] args) throws IOException, TimeoutException {
        //创建连接对象
        Connection connection = ConnectionUtil.getConnection();
        //创建通道
        final Channel channel = connection.createChannel();
        //声明队列
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        //将队列绑定到交换机
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"");
        channel.basicQos(1);
        //4.定义消费者监听
        Consumer defaultConsumer = new DefaultConsumer(channel){
            //消息到达触发handleDelivery方法
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msg = new String(body,"utf-8");
                System.out.println("fair receive[1]"+msg);
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("receiv[1] done");
                    channel.basicAck(envelope.getDeliveryTag(),false);
                }
            }
        };
        //自动应答改为false
        boolean autoAck = false;
        channel.basicConsume(QUEUE_NAME,autoAck,defaultConsumer);
    }
}
```
我们再仿造Customer类写一个消费者（复制Customer类一份，更改QUEUE_NAME的值），然后运行两个消费者与生产者的代码会发现，两个队列均能收到交换机转发的新消息并被消费者消费。  
#### 路由模式
路由模式采用的交换机类型为direct exchange，路由模式Java示例代码为：
<b>生产者：</b>
```java
public class RoutingSend {
    private static final String EXCHANGE_NAME ="test_exchange_direct";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel()；
        channel.exchangeDeclare(EXCHANGE_NAME,"direct");
        String msg = "hello rabbitmq routing";
        String routingKey = "error";
        //发布消息时指定routing key
        channel.basicPublish(EXCHANGE_NAME,routingKey,null,msg.getBytes());
        channel.close();
        connection.close();
    }
}
```
<b>消费者：</b>  
消费者模式代码与订阅模式消费者代码相似，只有一句代码需要更改
```java
//将队列绑定到交换机并指定bind key
channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "error");
```
再创建一个消费者，将其bing key设置为warning。运行所有代码后我们发现只有bind key为error的队列接受到了消息。这是因为在生产者中我们设定的routing key就时error,刚好routing key的值与bind key的值匹配了。
#### 主题模式
主题面膜是采用的交换机类型为topic exchange,主题模式Java示例代码为：
<b>生产者：</b>
```java
public class TopicSend {
    private static final String EXCHANGE_NAME = "test_exchange_topic";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE_NAME,"topic");
        String msg = "hello topic";
        //发布消息时指定routing key
        channel.basicPublish(EXCHANGE_NAME,"goods.update",null,msg.getBytes());
        channel.close();
        connection.close();
    }
}
```
<b>消费者1：</b>  
消费者模式代码与路由模式消费者代码相似，只是bind key可以改用通配符
```java
//将队列绑定到交换机并指定bind key
channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "goods.#");
```
<b>消费者2：</b>  
```java
//将队列绑定到交换机并指定bind key
channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "goods.add");
```
运行代码后我们发现只有消费者1能接受到消息，那是因为消费者1中的bind key为goods.#可以匹配开头为goods.的所有单词，故能与值为goods.update的routing key进行匹配，消息能转发到消费者1所在的队列中。