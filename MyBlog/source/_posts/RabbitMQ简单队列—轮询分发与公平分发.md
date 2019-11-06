title: RabbitMQ工作队列—轮询分发与公平分发
author: Loux
cover: /img/post/9.jpg
tags:
  - RabbitMQ
  - 消息队列
categories:
  - 后端知识
date: 2019-07-06 09:47:00
---
前面说了RabbitMQ的简单队列，但是在实际应用中我们的业务逻辑往往不是只有一个消费者来消费消息，有时候会有多个消费者。而RabbitMQ也提供了这样的队列—工作队列，用来解决简单队列的不足。在工作队列之中，存在多个消费者，但是它和简单队列一样一条消息只能被一个消费者消费。它的模型如下

![工作队列模型](/images/pasted-1.png)
工作队列消息的消费又分为两种模式  
* 轮询分发（Round-robin）：消息的分发是轮询的，即多个消费者依次从队列中获取消息并消费，每次获取一条，轮询分发不考虑消费者消费消息的速度。举个例子，队列中有100条消息，C1，C2两个消费者，无论它两的消费速度如何，最后的结果总是你一个我一个的每个人拿到25条消息。  

* 公平分发（Fair dispatcher）：消息的分发是公平的，即按照消费者消费的能力来分配消息，能者多劳。举个例子，队列中有90条消息，C1,C2两个消费者，其中C1消费一条消息的速度是1s，C2消费一条消息的速度是2s,那么最后的结果是C1消费了60条消息，C2消费了30条消息。
****

#### 轮询分发
轮询分发采用的是自动应答，即消费者消费完一条消息后不会手动的对队列做出应答，所以队列并不知道谁消费的快，谁消费的慢。故而无论多少个消费者，它的消息分配总是一个一条轮着来。轮询分发的核心Java代码如下
```java
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        //3定义消费者监听
        Consumer defaultConsumer = new DefaultConsumer(channel){
            //消息到达触发handleDelivery方法
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msg = new String(body,"utf-8");
                System.out.println("receive[1]"+msg);
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("receive[1] done");
                }
            }
        };
        //设置自动应答
        boolean autoAck = true;
        channel.basicConsume(QUEUE_NAME,autoAck,defaultConsumer);
```
其中关键的地方就是autoAck变量的值为true
****
#### 公平分发
公平分发采用的是手动应答，即每个消费者在没有手动发送确认到队列之前，队列不会向该消费者发送消息。故而消费的快的消费者会比其它的消费者更快的进行应答，因此它能消费更多的消息。  
1. 在Channel声明队列之后,Custumer创建之前加入这样一条语句：channel.basicQos(x),其中x代表每次向消费者发送消息不超过x条。  
2. 消费完成之后进行应答，加入的语句为：channel.basicAck(deliveryTag,multiple),其中deliveryTag表示唯一标识ID,当一个消费者向 RabbitMQ 注册后，会建立起一个 Channel ，RabbitMQ 会用 basic.deliver 方法向消费者推送消息，这个方法携带了一个 delivery tag， 它代表了 RabbitMQ 向该 Channel 投递的这条消息的唯一标识 ID，是一个long类型的变量。multiple：为了减少网络流量，手动确认可以被批处理，当该参数为 true 时，则可以一次性确认 delivery_tag 小于等于传入值的所有消息
3. 最后在channel对customer进行监听时，设置手动应答，即basicConsume()方法中的autoAck变量的值为false  
公平分发模式消费者核心Java代码如下  

```java
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        channel.basicQos(1);

        //定义消费者监听
        Consumer defaultConsumer = new DefaultConsumer(channel){
            //消息到达触发handleDelivery方法
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msg = new String(body,"utf-8");
                System.out.println("fair receive[2]"+msg);
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("receive[2] done");
                    channel.basicAck(envelope.getDeliveryTag(),false);
                }
            }
        };
        //自动应答改为false
        boolean autoAck = false;
        channel.basicConsume(QUEUE_NAME,autoAck,defaultConsumer);
```