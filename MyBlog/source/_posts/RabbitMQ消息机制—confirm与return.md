title: RabbitMQ消息机制—confirm与return
author: Loux
cover: /img/post/7.jpg
tags:
  - RabbitMQ
  - 消息队列
categories:
  - 后端知识
date: 2019-07-14 21:20:00
---
为了让我们知道消息是否发送成功、处理路由不匹配的消息等，RabbitMQ为我们提供了confirm与return机制
#### confirm确认消息
对于confirm确认消息机制的理解，可以分为：
* 消息的确认，是指生产者投递消息后，如果broker(RabbitMQ服务器)收到消息，则会给我们生产者一个应答
* 生产者进行接受应答，用来确定这条消息是否被正常的发送到了broker,这种方式也是实现消息的可靠性投递的核心保障  

消息确认机制流程图如下：


![confirm确认消息流程图](/images/pasted-8.png)
使用confirm确认消息需要经过以下两个步骤
1. 在channel上开启确认模式：<b>channel.confirmSelect()</b>
2. 在channel上添加Listener：<b>channel.addConfirmListener(new ConfirmListener(){....})</b>，监听投递成功和失败的返回结果，并在方法中对结果进行处理，例如记录日志、重新发送等等  

这里有一个demo对confirm确认消息机制进行代码表示，其Java代码示例为  
<b>生产者：</b>
```bash
public class Producer {
    private static final String EXCHANGE_NAME = "test_confirm_exchange";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection  connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        //指定我们的消息投递模式：消息确认模式
        channel.confirmSelect();
        String msg = "hello confirmListener rabbitmq";
        String routingKey = "confirmListener.save";
        channel.basicPublish(EXCHANGE_NAME,routingKey,null,msg.getBytes());
        //添加一个确认监听
        channel.addConfirmListener(new ConfirmListener() {
            //投递成功
            @Override
            public void handleAck(long l, boolean b) throws IOException {
                System.out.println("-------ack!");
            }
            //投递失败
            @Override
            public void handleNack(long l, boolean b) throws IOException {
                System.out.println("-------no ack!");
            }
        });
    }
}
```
<b>消费者：</b>
```bash
public class Consumer {
    private static final String EXCHANGE_NAME = "test_confirm_exchange";
    private static final String QUEUE_NAME = "test_confirm_queue";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE_NAME,"topic",true);
        channel.queueDeclare(QUEUE_NAME,true,false,false,null);
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"confirmListener.*");
        DefaultConsumer defaultConsumer = new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msg = new String(body,"utf-8");
                System.out.println(msg);
            }
        };
        channel.basicConsume(QUEUE_NAME,true,defaultConsumer);
    }
}
```
先运行消费者代码，再运行生产者代码。结果发现生产者控制台输出：
> -------ack!  //说明消息投递成功

#### return消息机制
对于return消息机制的理解，可以分为：
* return listener用于处理一些不可路由的消息
* 某些情况下，在发送消息的时候当前的exchange不存在或者指定的routing key路由不匹配时，监听这种不可达的消息就需要用到return listener

注意在channel.basicPublish()方法中的Mandatory参数：如果为true,则return监听器会接收路由不可到达的消息并进行处理；如果为false,则broker端将会自动将该消息删除
return消息机制流程图如下：

![return消息机制流程图](/images/pasted-9.png)
这里有一个demo对return消息机制进行代码表示，其Java代码示例为  
<b>生产者：</b>
```bash
public class Producer {
    private static final String EXCHANGE_NAME = "test_return_exchange";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        String routingKeyError = "abc.save";
        String msg = "hello return rabbitmq";
        channel.addReturnListener(new ReturnListener() {
            @Override
            public void handleReturn(int i, String s, String s1, String s2, AMQP.BasicProperties basicProperties, byte[] bytes) throws IOException {
                System.out.println("------handle return------");
                System.out.println("replycode:"+i);
                System.out.println("replyText:"+s);
                System.out.println("exchange_name:"+s1);
                System.out.println("routingKey:"+s2);
                System.out.println("properties:"+basicProperties);
                System.out.println("body:"+new String(bytes,"utf-8"));
            }
        });
        channel.basicPublish(EXCHANGE_NAME,routingKeyError,true,null,msg.getBytes());
    }
}
```
<b>消费者（省略部分代码）：</b>
```bash
.........
 channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"return.#");
 ........
```
从代码中我们可以看到，routing key="abc.save",而消费者中的binding key="return.#"，两者不能匹配，故消息不能被转发到队列。return listener监听到不可到达的消息后，进行处理。运行代码后控制台输出如下：
> ------handle return------
replycode:312
replyText:NO_ROUTE
exchange_name:test_return_exchange
routingKey:abc.save
properties:#contentHeader<basic>(content-type=null, content-encoding=null, headers=null, delivery-mode=null, priority=null, correlation-id=null, reply-to=null, expiration=null, message-id=null, timestamp=null, type=null, user-id=null, app-id=null, cluster-id=null)
body:hello return rabbitmq