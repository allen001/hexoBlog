---
title: RabbitMQ实战
date: 2020-05-19 16:01:48
toc: true
comments: true
tags:
- springboot
- rabbitmq
---

## 01：为什么要使用RabbitMQ

RabbitMQ是一个开源的消息队列服务器 ，用来通过普通协议在完全不同的应用之间共享数据，Rabbitmq是使用Erlang语言（**数据传递**）来编写的，并且Rabbitmq是**基于AMQP协议**(协议是一种规范和约束)的。开源做到跨平台机制。

消息队列中间件是分布式系统中重要的组件，主要解决应用 **解耦，异步消息，流量削锋等问题，实现高性能，高可用，可伸缩和最终一致性架构**。

目前使用较多的消息队列有 、ActiveMQ，RabbitMQ，ZeroMQ，Kafka，MetaMQ，RocketMQ。

### 在什么情况下使用rabbitmq

* 写多读少的情况下使用 读多写少用缓存（Redis）,写多读少用队列（Rabbitmq ,Activemq）
* 解耦，系统A在代码中直接调用系统B和系统C的代码，如果将来D系统接入，系统A还需要修改代码，过于麻烦
* 异步，将消息写入消息队列，非必要的业务逻辑以异步的方式运行，加快响应速度
* 削峰，并发量大的时候，所有的请求直接怼到数据库，造成数据库连接异常

### rabbitmq优点

* 开源，性能优秀，稳定性好
* 提供可靠的消息投递模式（confirm）,返回模式（return）
* 与SpringAMQP 完美的整合，API丰富
* 集群模式丰富(mirror rabbitmq)，表达式配置，HA模式，镜像队列模型。
* 保证数据不丢失的前提做到高可用性，可用性。

<!-- more -->

## 02：什么是AMQP高级消息队列协议

AMQP全称：Advanced Message Queuing Protocol(高级消息队列协议)。

是具有现代特性的二进制协议，是一个提供统一消息服务的应用层标准高级消息队列协议，是应用协议的一个开发标准，为面向消息的中间件设计。

![image-20200519145141339](https://i.loli.net/2020/05/19/dNzsTkrADQxBCm9.png)

核心概念：

**Server**：又称Broker，接收客户端的链接，实现AMQP实体服务。安装rabbitmq-server

**Connection**：连接，应用程序与Broker的网络连接TCP/IP三次握手和四次挥手

**Channel**：网络信道，几乎所有的操作都在Channel中进行，Channel是进行消息读写的通道，客户端可以建立对各Channel，每个Channel代表一个会话任务。

**Message**：消息，服务与应用程序之间传送的数据，由Properties和body组成，Properties可是对消息进行修饰，比如消息的优先级，延迟等高级特性，Body则就是消息体的内容。

**Virtual Host**：虚拟地址，用于进行逻辑隔离，最上层的消息路由，一个虚拟主机理由可以有若干个Exchange和Queue，同一个虚拟主机里面不能有相同名字的Exchange或Queue

**Exchange**：交换机，接受消息，根据路由键发送消息到绑定的队列。direct activemq kafka

**Bindings**：Exchange和Queue之间的虚拟连接，binding中可以保护多个routing key。

**Routing key**：是一个路由规则，虚拟机可以用它来确定如何路由一个特定消息。

**Queue**：队列，也成为Message Queue,消息队列，保存消息并将它们转发给消费者。

**RabbitMQ整体架构**：

![image-20200519145915721](https://i.loli.net/2020/05/19/iUIpfh72YtLDjzJ.png)

**消息流转图**：

![image-20200519150440788](https://i.loli.net/2020/05/19/zReSYu6qs1v9TaW.png)

## 03：使用Docker安装RabbitMQ

**帮助文档**： https://www.rabbitmq.com/getstarted.html

* 获取rabbitmq镜像

```bash
docker pull rabbitmq:management
```

* 创建并运行容器

```bash
docker run -di --name myrabbit 
	-e RABBITMQ_DEFAULT_USER=admin 
	-e RABBITMQ_DEFAULT_PASS=admin 
	-p 15672:15672 
	-p 5672:5672
	rabbitmq:management
```

* 其他服务命名

```bash
docker start myrabbit
docker stop myrabbit
docker restart myrabbit
docker logs -f myrabbit
```

![image-20200519183217569](https://i.loli.net/2020/05/19/sbid5qKf2EWGNx7.png)


* 使用 http://127.0.0.1:15672 访问rabbitmq控制台

## 04：使用RabbitMQ场景

#### 1. 同步异步的问题（串行）

**串行方式**：将订单信息写入数据库成功后，发送注册邮件，再发送注册短信。以上三个任务全部完成后，返回给客户端

![image-20200519154451554](https://i.loli.net/2020/05/19/UuaPxNIOgDR1Aqt.png)

```java
public void makeOrder() {
        // 1:保存订单
        orderService.saveOrder();
        // 2:发送短信服务
        messageService.sendSMS("order");
        // 3:发送email服务
        emailService.sendEmail("order");
        // 4:发送短信服务
        appService.sendApp("order");
        // 5:发送微信服务
        weixinService.sendApp("order");
    }
```

**并行方式**：将订单信息写入数据库成功后，发送注册邮件的同时，发送注册短信。以上三个任务完成后，返回给客户端。与串行的差别是，并行的方式可以提高处理的时间

![image-20200519154757383](https://i.loli.net/2020/05/19/QpL8CVDv47iXudw.png)

```java
public void makeOrder() {
        // 1:保存订单
        orderService.saveOrder();

        theadpool.submit(new Callable<Object> {
            public Object call () {
                // 2:发送短信服务
                messageService.sendSMS("order");
            }
        })
        
        theadpool.submit(new Callable<Object> {
            public Object call () {
                // 3:发送email服务
                emailService.sendEmail("order");
            }
        })
        
        theadpool.submit(new Callable<Object> {
            public Object call () {
                // 4:发送短信服务
                appService.sendApp("order");
            }
        })
    }
```

![image-20200519155220016](https://i.loli.net/2020/05/19/rJWaDCbeTyugXfU.png)

按照以上约定，用户的响应时间相当于是订单信息写入数据库的时间，也就是50毫秒。注册邮件，发送短信写入消息队列后，直接返回，因此写入消息队列的速度很快，基本可以忽略，因此用户的响应时间可能是50毫
秒。因此架构改变后，系统的吞吐量提高到每秒20 QPS。比串行提高了3倍，比并行提高了两倍。

```java
public void makeOrder(){
 // 1:保存订单
 orderService.saveOrder();
 // 2:发送消息
 rabbitTemplate.convertSend("ex","2","消息内容");
}
```

#### 2. 高内聚低耦合

![image-20200519155342875](https://i.loli.net/2020/05/19/hoNQ5XOJuZjaGne.png)

#### 3. 流量的削峰（自己限流）

![image-20200519155433977](https://i.loli.net/2020/05/19/7SpHXNRj9F32Dul.png)

![image-20200519155443824](https://i.loli.net/2020/05/19/OCbPd98Er5s3jFy.png)

## 05：SpringBoot-整合RabbitMQ发布订阅机制-Fanout
#### 概述

![image-20200519155607920](https://i.loli.net/2020/05/19/XOhAovxzZrJpnej.png)

#### 生产者工程：springboot-rabbitmq-fanout-producers

1：引入依赖

   ```xml
   <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-amqp</artifactId>
   </dependency>
   <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-web</artifactId>
   </dependency>
   <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-devtools</artifactId>
     <scope>runtime</scope>
   </dependency>
   <dependency>
     <groupId>org.projectlombok</groupId>
     <artifactId>lombok</artifactId>
     <optional>true</optional>
   </dependency>
   <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-test</artifactId>
     <scope>test</scope>
   </dependency>
   ```

2：定义订单的消息生产者`Sender.java`

```java
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import java.util.Date;

@Component
public class Sender {
  @Autowired
  private AmqpTemplate amqpTemplate;
  @Value("${mq.config.exchange}")
  private String exchangeName;
  
  public void sendMessage() throws InterruptedException {
    amqpTemplate.convertAndSend(exchangeName,"","订单保存成功，去发送短信和微信.....");
  }
}
```

3：定义配置文件和配置交换机

```properties
server.port=8094
spring.application.name=springboot-rabbitmq-topic-provider

# rabbitmq
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin

# 设置交换机
mq.config.exchange=order.fanout
```

#### 消费者工程：springboot-rabbitmq-fanout-consumbers

1：定义消费者配置文件

```properties
server.port=8093
spring.application.name=springboot-rabbitmq-topic-consumer

# rabbitmq
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin

# 设置交换机
mq.config.exchange=order.fanout

# 定义消费者队列
# 消息队列
mq.config.queue.sms=order.sms
# 微信队列
mq.config.queue.weixin=order.weixin
```

2：定义消息消费类`SMSReceiver` `WeixinReceiver`

```java
import org.springframework.amqp.core.ExchangeTypes;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.stereotype.Component;

@Component
@RabbitListener(bindings =@QueueBinding(
 value = @Queue(value =
"${mq.config.queue.sms}",autoDelete = "true"),
 exchange = @Exchange(value =
"${mq.config.exchange}",type = ExchangeTypes.FANOUT)
))
public class SMSReceiver {
  @RabbitHandler
  public void process(String msg) {
    System.out.println("SMS----------->接受到的的消息是：" + msg);
 }
}
```

```java
import org.springframework.amqp.core.ExchangeTypes;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.stereotype.Component;

@Component
@RabbitListener(bindings =@QueueBinding(
 value = @Queue(value =
"${mq.config.queue.weixin}",autoDelete = "true"),
 exchange = @Exchange(value =
"${mq.config.exchange}",type = ExchangeTypes.FANOUT)
))
public class WeiXinReceiver {
  @RabbitHandler
  public void process(String msg) {
    System.out.println("WEIXIN----------->接受到的的消息是：" + msg);
 }
}
```

#### 测试

1：启动消费者工程

2：在`springboot-rabbitmq-fanout-producers`工程定义测试类

```java
import com.itbooking.springbootrabbitmqfadoutproducers.mq.Sender;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootRabbitmqFadoutProducersApplicationTests {
  
  @Autowired
  private Sender sender;
  
  @Test
  public void contextLoads() throws InterruptedException {
    sender.sendMessage();
  }
}
```

3：结果

```properties
WEIXIN----------->接受到的的消息是：订单保存成功，去发送短信和微信.....
SMS----------->接受到的的消息是：订单保存成功，去发送短信和微信.....
```

## 06: SpringBoot-整合RabbitMQ发布订阅机制-Direct
#### 概述

![image-20200519161611982](https://i.loli.net/2020/05/19/89d6pbiv5aELcAV.png)

#### 配置依赖`pom.xml`

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-devtools</artifactId>
  <scope>runtime</scope>
</dependency>
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <optional>true</optional>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
```

#### 生产者工程：springboot-rabbitmq-direct-producer

```properties
server.port=8089
spring.application.name=springboot-rabbitmq-server

# rabbitmq
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin

# 设置交换机
mq.config.exchange=log.direct

# 设置路由key
mq.config.routekey=log.routing.key
```

```java
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import java.util.Date;

@Component
public class Sender {

	@Autowired
  	private AmqpTemplate amqpTemplate;
  	@Value("${mq.config.exchange}")
  	private String exchangeName;
  	@Value("${mq.config.routekey}")
  	private String routekey;
  	
  	public void sendMessage() throws InterruptedException {
	    while (true) {
    	  Thread.sleep(2000);
      	  String msg = "Hello rabbitmq " + new Date();
		  amqpTemplate.convertAndSend(exchangeName,routekey,msg)
		}
    }
}
```

#### 消费者工程：springboot-rabbitmq-direct-consumer

```properties
server.port=8188
spring.application.name=springboot-rabbitmq-server

# rabbitmq
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin

# 设置交换机
mq.config.exchange=log.direct

# 设置错误队列
mq.config.queue.error=log.error.queue
# 设置info队列
mq.config.queue.info=log.info.queue
# 设置路由键
mq.config.queue.routing.key=log.routing.key
```

定义订阅的INFO和ERROR两个类

```java
import org.springframework.amqp.core.ExchangeTypes;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.stereotype.Component;

@Component
@RabbitListener(bindings =@QueueBinding(
 value = @Queue(value =
"${mq.config.queue.info}",autoDelete = "true"),
 exchange = @Exchange(value =
"${mq.config.exchange}",type = ExchangeTypes.DIRECT),
  key="${mq.config.queue.routing.key}"
))
public class InfoReceiver {
  @RabbitHandler
  public void process(String msg) {
    System.out.println("info----------->接受到的的消息是：" + msg);
 }
}
```

```java
import org.springframework.amqp.core.ExchangeTypes;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.stereotype.Component;

@Component
@RabbitListener(bindings =@QueueBinding(
 value = @Queue(value ="${mq.config.queue.error}",autoDelete = "true"),
 exchange = @Exchange(value ="${mq.config.exchange}",type = ExchangeTypes.DIRECT),
 key="${mq.config.queue.routing.key}"
))
public class ErrorReceiver {
  @RabbitHandler
  public void process(String msg) {
    System.out.println("error----------->接受到的的消息是：" + msg);
 }
}
```

#### 测试

```java
import
com.itbooking.springbootrabbitmqdirectproducer.mq.Sender;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootRabbitmqDirectProducerApplicationTests {
  @Autowired
  private Sender sender;
  @Test
  public void contextLoads() throws InterruptedException {
    sender.sendMessage();
  }
}
```

打印内容

![image-20200519162602764](https://i.loli.net/2020/05/19/d3hiy8HVXu6mo42.png)

## 07：SpringBoot-整合RabbitMQ主题匹配Topic

#### 概述

![image-20200519162808370](https://i.loli.net/2020/05/19/BkonOQ1FK3ECh7l.png)

#### 配置依赖`pom.xml`

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-devtools</artifactId>
  <scope>runtime</scope>
</dependency>
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <optional>true</optional>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
```

#### 生产者工程：springboot-rabbitmq-topic-producer

1：配置`application.properties`

```properties
server.port=8091
spring.application.name=springboot-rabbitmq-topic-producer

# rabbitmq
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin

# 设置交换机
mq.config.exchange=log.topic
```

2：定义三个消息发送者分别是: `GoodsSender` , `OrderSender` , `UserSender` 

```java
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class GoodsSender {

    @Autowired
    private AmqpTemplate amqpTemplate;
    @Value("${mq.config.exchange}")
    private String exchangeName;

    public void sendMessage() throws InterruptedException {
        amqpTemplate.convertAndSend(exchangeName, "product.log.info", "product.log.info................");
        amqpTemplate.convertAndSend(exchangeName, "product.log.warn", "product.log.warn................");
        amqpTemplate.convertAndSend(exchangeName, "product.log.debug", "product.log.debug..............");
        amqpTemplate.convertAndSend(exchangeName, "product.log.error", "product.log.error..............");
    }
}
```

```java
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class OrderSender {

    @Autowired
    private AmqpTemplate amqpTemplate;
    @Value("${mq.config.exchange}")
    private String exchangeName;

    public void sendMessage() throws InterruptedException {
        amqpTemplate.convertAndSend(exchangeName, "order.log.info", "order.log.info................");
        amqpTemplate.convertAndSend(exchangeName, "order.log.warn", "order.log.warn................");
        amqpTemplate.convertAndSend(exchangeName, "order.log.debug", "order.log.debug..............");
        amqpTemplate.convertAndSend(exchangeName, "order.log.error", "order.log.error..............");
    }
}
```

```java
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class UserSender {

    @Autowired
    private AmqpTemplate amqpTemplate;
    @Value("${mq.config.exchange}")
    private String exchangeName;

    public void sendMessage() throws InterruptedException {
        amqpTemplate.convertAndSend(exchangeName, "user.log.info", "user.log.info................");
        amqpTemplate.convertAndSend(exchangeName, "user.log.warn", "user.log.warn................");
        amqpTemplate.convertAndSend(exchangeName, "user.log.debug", "user.log.debug..............");
        amqpTemplate.convertAndSend(exchangeName, "user.log.error", "user.log.error..............");
    }
}
```

#### 消费者工程：springboot-rabbitmq-topic-consumer

1：配置`application.properties`

```properties
server.port=8192
spring.application.name=springboot-rabbitmq-topic-consumer

# rabbitmq
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin

# 设置交换机
mq.config.exchange=log.topic

# 设置log队列
mq.config.queue.info=log.info
# 设置error队列
mq.config.queue.error=log.error
# 设置msg队列
mq.config.queue.logs=log.msg
```

2：定义消费者 `ErrorReceiver` ,` InfoReceiver` ,` LogsReceiver`

```java
import org.springframework.amqp.core.ExchangeTypes;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.stereotype.Component;

@Component
@RabbitListener(bindings = @QueueBinding(
        value = @Queue(value = "${mq.config.queue.error}", autoDelete = "true"),
        exchange = @Exchange(value = "${mq.config.exchange}",
                type = ExchangeTypes.TOPIC),
        key = "*.log.error"
))
public class ErrorReceiver {

    @RabbitHandler
    public void process(String msg) {
        System.out.println("error----------->接受到的的消息是：" + msg);
    }
}
```

```java
import org.springframework.amqp.core.ExchangeTypes;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.stereotype.Component;

@Component
@RabbitListener(bindings = @QueueBinding(
        value = @Queue(value = "${mq.config.queue.info}", autoDelete = "true"),
        exchange = @Exchange(value = "${mq.config.exchange}",
                type = ExchangeTypes.TOPIC),
        key = "*.log.info"
))
public class InfoReceiver {

    @RabbitHandler
    public void process(String msg) {
        System.out.println("info----------->接受到的的消息是：" + msg);
    }
}
```

```java
import org.springframework.amqp.core.ExchangeTypes;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.stereotype.Component;

@Component
@RabbitListener(bindings = @QueueBinding(
        value = @Queue(value = "${mq.config.queue.log}", autoDelete = "true"),
        exchange = @Exchange(value = "${mq.config.exchange}",
                type = ExchangeTypes.TOPIC),
        key = "*.log.msg"
))
public class LogsReceiver {

    @RabbitHandler
    public void process(String msg) {
        System.out.println("Logs----------->接受到的的消息是：" + msg);
    }
}
```

#### 测试

```java
import com.itbooking.springbootrabbitmqtopicproducer.mq.GoodsSender;
import com.itbooking.springbootrabbitmqtopicproducer.mq.OrderSender;
import com.itbooking.springbootrabbitmqtopicproducer.mq.UserSender;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootRabbitmqTopicProducerApplicationTests {

    @Autowired
    private GoodsSender goodsSender;
    @Autowired
    private OrderSender orderSender;
    @Autowired
    private UserSender userSender;

    @Test
    public void contextLoads() throws InterruptedException {
        goodsSender.sendMessage();
        orderSender.sendMessage();
        userSender.sendMessage();
    }
}
```

测试结果

```properties
info----------->接受到的的消息是：product.log.info................
info----------->接受到的的消息是：order.log.info................
info----------->接受到的的消息是：user.log.info................
Logs----------->接受到的的消息是：product.log.info................
Logs----------->接受到的的消息是：product.log.warn................
Logs----------->接受到的的消息是：product.log.debug..............
Logs----------->接受到的的消息是：product.log.error..............
Logs----------->接受到的的消息是：order.log.info................
Logs----------->接受到的的消息是：order.log.warn................
Logs----------->接受到的的消息是：order.log.debug..............
Logs----------->接受到的的消息是：order.log.error..............
Logs----------->接受到的的消息是：user.log.info................
Logs----------->接受到的的消息是：user.log.warn................
Logs----------->接受到的的消息是：user.log.debug..............
Logs----------->接受到的的消息是：user.log.error..............
error----------->接受到的的消息是：product.log.error..............
error----------->接受到的的消息是：order.log.error..............
error----------->接受到的的消息是：user.log.error..............
```

## 08：SpringBoot-RabbitMQ消息处理-持久化

#### 配置

修改autoDelete的状态即可，false是持久化，true是非持久化

![image-20200519165552900](https://i.loli.net/2020/05/19/PBzZbW1pMArqHlY.png)

#### 概述

消息丢了怎么办？一般我们进行持久化即可，操作如下：

```java
import org.springframework.amqp.core.ExchangeTypes;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.stereotype.Component;

// autoDelete 是否持久化操作。false是持久化，true不持久化，
@Component
@RabbitListener(bindings =@QueueBinding(
 value = @Queue(value ="${mq.config.queue.error}",autoDelete = "false"),
 exchange = @Exchange(value ="${mq.config.exchange}",type = ExchangeTypes.DIRECT),
 key = "${mq.config.queue.error.routing.key}"
))
public class ErrorReceiver {
  @RabbitHandler
  public void process(String msg) {
    System.out.println("error----------->接受到的的消息是：" + msg);
 }
}
```

把上面的 `autoDelete `改成false，即持久化操作。这样可以防止因为网络故障问题导致消费消息不连续的问题。

注意测试的时候后一定是要：先在rabbitmq管控台先删除，在重新生成在测试，结论是

* 这个时候把 `autoDelete` 改成true以后，一定关闭连接，队列就直接丢失。

* 如果 `autoDelete `是false,就是持久化的队列，这个时候即使你重启了rabbitmq服务，消息依然存在。

![image-20200519165812505](https://i.loli.net/2020/05/19/8hCt9f3dNJ7zijo.png)

#### 消费出现异常

因为rabbitmq当消费者出现异常的时候，rabbitmq的默认情况是一种：自动确认过程，而报异常以后这自动确认的过程，就没办法确认，没办法确认，它就会把这个消息重新放入到消息队列继续去发送，这样就造成死循环。

* 开启消息重试即可，如果你单机版，会一直重试自己服务器，如果集群版：切换别的服务器重试。如果重试的次数已经达到限制次数，还是都失败，这个消息直接扔掉。
* **推荐做法：try/catch + 死信 + 重定向+重试 + 人工干预短信预警**

**消费出现服务宕机：用持久队列**

## 09：SpringBoot-RabbitMQ如何解决消息出现异常死循环问题
#### 概述

什么是消息确认的ACK

如果在处理消息的过程中，消费者在消费消息的时候服务器、网络、出现故障挂掉了，那可能这条正在处理的消息就没有完成，**数据就会丢失**。为了确保消息不会丢失，RabbitMQ支持消息确认-ACK。

#### pom.xml

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
	<groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
	<optional>true</optional>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-test</artifactId>
	<scope>test</scope>
</dependency>
```

#### 生产者

配置rabbitmq

```properties
server.port=8489
spring.application.name=springboot-rabbitmq-server

# rabbitmq
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin

# 设置交换机
mq.config.exchange=log.direct

# 设置路由key
mq.config.routekey=log.routing.key
```

```java
@Component
public class Sender {
  @Autowired
  private AmqpTemplate amqpTemplate;
  @Value("${mq.config.exchange}")
  private String exchangeName;
  @Value("${mq.config.routekey}")
  private String routerkey;
    
  public void sendMessage() throwsInterruptedException {
    String msg = "Hello rabbitmq " + new Date();
    amqpTemplate.convertAndSend(exchangeName,routerkey, msg);
  }
}
```

#### 消费者

配置rabbitmq

```properties
server.port=8288
spring.application.name=springboot-rabbitmq-server

# rabbitmq
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin
# 设置交换机
mq.config.exchange=log.direct

# 设置错误队列
mq.config.queue.error=log.error.queue
# 设置info队列
mq.config.queue.info=log.info.queue
# 设置路由键
mq.config.queue.routing.key=log.routing.key
```

```java
import org.springframework.amqp.core.ExchangeTypes;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.stereotype.Component;

@Component
@RabbitListener(bindings =@QueueBinding(
 value = @Queue(value ="${mq.config.queue.error}",autoDelete = "true"),
 exchange = @Exchange(value ="${mq.config.exchange}",type = ExchangeTypes.DIRECT),
 key="${mq.config.queue.routing.key}"
))
public class ErrorReceiver {
    
  @RabbitHandler
  public void process(String msg) {
    System.out.println("error----------->接受到的的消息是：" + msg);
    throw  new  RuntimeException();//模拟消息超时
  }
}
```

核心代码：

```java
System.out.println("error----------->接受到的的消息是：" + msg);
throw  new  RuntimeException();
```

这个时候总又一条消息无法消费，rabbitmq会把这条消息重新放入到队列
中继续消费，**造成死循环**。

![image-20200519170915424](https://i.loli.net/2020/05/19/tENaHZI5vmLzjBg.png)

#### 分析

1、ACK的消息确认机制

ACK的确认机制是消费者端从RabbitMQ收到消息并处理完成后，反馈给RabbitMQ。

**RabbitMQ收到反馈后才将此消息从队列中删除。**

* 如果一个消费者在处理消息时，挂掉了（网络不稳定，服务器异常，网站故障等），那么就不会有ACK反馈，RabbitMQ会认为这个消息没有正常消费，**会将此消息重新放入队列中。就会引发死循环**。

* 如果集群的情况下，RabbitMQ会立即将这个消息推送给这个在线的其他消费者，这种机制保证了在消费服务器故障时，不会丢失任何消息和任务。

* **消息永远不会从RabbitMQ服务器中删除**；只有当消费者正确的发送ACK确认反馈，RabbitMQ确认收到后，消息才会从RabbitMQ服务器的数据中删除。

* 消息的ACK确认机制默认是打开的。**自动的ACK**

2、Ack机制的开发注意事项

如果忘记了ACK，那么后果是很严重的。当Consumber退出时，Message会一直从新发送，然后RabbitMQ会占用越来越多的内存，由于RabbitMQ会长时间运行，因此这个 “内存泄露”是致命的。

#### 测试

```java
import com.itbooking.springbootrabbitmqackdirectproducer.mq.Sender;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootRabbitmqAckDirectProducerApplicationTests {
  @Autowired
  private Sender sender;
  @Test
  public void contextLoads() throws InterruptedException {
    sender.sendMessage();
  }
}
```

* 先启动消费者
* 然后在运行测试用例
* 出现死循环。

#### 解决方案

1：在消费者工程中在代码中加入try/catach机制，防止消息死循环。

2：在消费者工程中使用消息的重试机制。

```properties
# 解决消息的ACK死循环问题
spring.rabbitmq.listener.simple.retry.enabled=true
spring.rabbitmq.listener.simple.retry.max-attempts=5
```

3：**因为：默认情况下是：自动ACK，所以重试以后达到限定的次数就回丢掉消息，造成不安全**。

## 10：消息如何保障100%的投递成功(重点)-手动ACK和Confirm确认
消息落库，对消息状态进行打标。做法就是把消息发送MQ一份，同时存储到数据库一份。然后进行消息装填的控制，发送成功1，发送失败0。必须结合应答来完成。对于哪些如果没有发送成功的消息，可以采用定时器进行轮询发送。

![image-20200519171741305](https://i.loli.net/2020/05/19/juBmnXIUWdyJ7FQ.png)

> 存在问题：可能出现分布式定时任务出现消息重发的情况。因为在消息保存进去的时候状态是0，其实这个时候ACK是可以回执，但是定时器恰好在那个时间点运行到了。就出现了消息重发的问题，解决方案：幂等性。

#### 生产者

1：配置依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

2：配置application.properties

```properties
#服务器
server.port=7878

#配置rabbitmq
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin
spring.rabbitmq.virtual-host=/

# 定义交换机
mq.config.exchange=log.ack.direct

# 设置路由key
mq.config.routekey=red.ack.routing.key

# 开启消息发送确认机制，重要！！！
#spring.rabbitmq.publisher-confirm-type=correlated
spring.rabbitmq.publisher-confirms=true
```

3：生产者

```java
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;

import javax.annotation.PostConstruct;

public class Sender {
    // 初始化一个rabbitmq的实例对象
    @Autowired
    private RabbitTemplate rabbitTemplate;
    @Value("${mq.config.exchange}")
    private String exchangeName;
    @Value("${mq.config.routekey}")
    private String routekey;

    @PostConstruct
    public void setup() {
        // 消息发送完毕后，则回调此方法，ack代表发送是否成功
        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                // 如果ack为true,代表MQ已经准备收到消息。
                if (!ack) {
                    return;
                }
                try {
                    System.out.println("接受确认的数据是：" + correlationData.getId());
                    System.out.println("本地消息保存成功 !!!");
                } catch (Exception ex) {
                    System.out.println("警告：修改本地消息表的状态时出现异常 !" + ex.getMessage());
                }
            }

        });
    }

    public void sendMessage(int i) throws InterruptedException {
        //下单
        String msg = "{id:10001,price:100}";
        rabbitTemplate.convertAndSend(exchangeName, routekey, msg, new CorrelationData("10001"));
        System.out.println(msg);
    }
}
```

#### 消费者

1：配置手动确认application.properties

```properties
#服务器
server.port=7879

#配置rabbitmq
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin
spring.rabbitmq.virtual-host=/

# 定义交换机
mq.config.exchange=log.ack.direct

# 定义队列三个消费者队列
# 红包队列
mq.config.queue.red=order.ack.red.log
# 设置路由key
mq.config.red.routekey=red.ack.routing.key

# 解决消息的ACK死循环问题--自动确认
spring.rabbitmq.listener.simple.retry.enabled=true
# 最多重试几次，但是在实际的生产中如果不重要的可以这样做，
spring.rabbitmq.listener.simple.retry.max-attempts=5
# 重要，开启手动ack，控制消息在MQ中的删除，重发。
spring.rabbitmq.listener.simple.acknowledge-mode=MANUAL
```

2：定义消费者并且手动确认

```java
import com.rabbitmq.client.Channel;
import org.springframework.amqp.core.ExchangeTypes;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Service;

@Service
// 用RabbitListener来完成队列和交换机的定义和绑定关系以及设置交换机的类型
// autoDelete= true代表非持久化 false 持久化
@RabbitListener(bindings = @QueueBinding(
    value=@Queue(value="${mq.config.queue.red}",autoDelete="false"),
    exchange =@Exchange(value = "${mq.config.exchange}",type = ExchangeTypes.DIRECT),
    key = "${mq.config.red.routekey}"
))
public class RedxinReceiver {
    
  @Autowired
  RabbitTemplate rabbitTemplate;
    
  @RabbitHandler
  public void getMessage(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag){
    try {
      System.out.println("red------接收到订单的消息了,内容是：" + message);
      //参数1：deliveryTag（唯一标识 ID）：当一个消费者向 RabbitMQ 注册后，会建立起一个 Channel ，RabbitMQ 会用basic.deliver 方法向消费者推送消息，这个方法携带了一个delivery tag， 它代表了 RabbitMQ 向该 Channel 投递的这条消息的唯一标识 ID，是一个单调递增的正整数，delivery tag 的范围仅限于 Channel
      //参数2：multiple：为了减少网络流量，手动确认可以被批处理，当该参数为 true 时，则可以一次性确认 delivery_tag 小于等于传入值的所有消息
      channel.basicAck(tag,false);
   }catch (Exception ex) {
      ex.printStackTrace();
   }
 }
}
```

#### 测试

```java
import com.icoding.springbootrabbitfanoutproducer.mq.Sender;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class SpringbootRabbitDirectProducerAckApplicationTests {
  @Autowired
  private Sender sender;
  @Test
  void contextLoads() throws  Exception{
    sender.sendMessage(1);
  }
}
```

运行结果：消费者消费消息

```properties
red------接收到订单的消息了,内容是：{id:10001,price:100} 
```

生产者回执消息如下：

```properties
{id:10001,price:100}
接受确认的数据是：10001
本地消息保存成功!!!
```

## 11：消费者出现异常 - 采用重新投递和取消确认的方式--解决方式1
接着上面的代码。如果这个时候消费者确认失败了，怎么处理？

* 一般采用重新投递 + 取消确认的方式
* 或者打入死信队列或者记录到redis数据库中
* 或者预警发送短信预警 + redis数据库

```java
import com.icoding.springbootrabbitfanoutconsumber.config.RabbitDeadLetterConfig;
import com.rabbitmq.client.Channel;
import lombok.extern.log4j.Log4j2;
import org.springframework.amqp.core.ExchangeTypes;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Service;

@Service
// 用RabbitListener来完成队列和交换机的定义和绑定关系以及设置交换机的类型
// autoDelete= true代表非持久化  false 持久化
@RabbitListener(bindings = @QueueBinding(
        value = @Queue(value="${mq.config.queue.red}",autoDelete ="false"),
        exchange =@Exchange(value = "${mq.config.exchange}",type = ExchangeTypes.DIRECT),
        key = "${mq.config.red.routekey}"
))
@Log4j2
public class RedxinReceiver {

    @Autowired
    RabbitTemplate rabbitTemplate;

    @RabbitHandler
    public void getMessage(String msg, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag, Message message){
        try{
            System.out.println("red------接收到订单的消息了,内容是：" + message);
            //接收消息出错了
            throw new RuntimeException();
        }catch(Exception ex){
            /**
             * 这里对消息重入队列做设置，例如将消息序列化缓存至 Redis, 并记录重入队列次数
             * 如果该消息重入队列次数达到一定次数，比如3次，将不再重入队列，直接拒绝
             * 这时候需要对消息做补偿机制处理
             * channel.basicNack与channel.basicReject要结合越来使用
             **/
            // 如果你的消息没用在确认之前出现了错误，redelivered是false，代表消息没用确认成功，表明是第一次消费，可以直接打入到死信队列中去
            // 如果重启了消费者以后，redelivered就回变成了false说明消息已经收到但是一直没用确认，属于重复消费了。
            Boolean redelivered = message.getMessageProperties().getRedelivered();
            System.out.println("=============>" + redelivered);
            try {
                if (redelivered) {
                    /**
                     * 1. 对于重复处理的队列消息做补偿机制处理
                     * 2. 从队列中移除该消息，防止队列阻塞
                     */
                    // 消息已重复处理失败, 扔掉消息
                    channel.basicReject(message.getMessageProperties().getDeliveryTag(), false); // 拒绝消息
                    log.error("消息[{}]重新处理失败，扔掉消息", msg);
                }

                // redelivered != true,表明该消息是第一次消费
                if (!redelivered) {
                    // 消息重新放回队列
                    channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, true);
                    log.error("消息[{}]处理失败，重新放回队列", msg);
                }
            } catch (Exception exc) {
                exc.printStackTrace();
            }
        }
    }
}
```

## 12：消费者出现异常 - 采用重死信队列+预警的方式--解决方式2
#### 什么是死信队列

死信队列实际上就是，当我们的业务队列处理失败(比如抛异常并且达到了retry的上限)，就会将消息重新投递到另一个Exchange(Dead LetterExchanges)，该Exchange再根据routingKey重定向到另一个队列，在这个队列重新处理该消息。

来自一个队列的消息可以被当做‘死信’，即被重新发布到另外一个“exchange”去，这样的情况有：

* 消息被拒绝 (basic.reject or basic.nack) 且带 requeue=false不重新入队参数或达到的retry重新入队的上限次数
* 消息的TTL(Time To Live)-存活时间已经过期
* 队列长度限制被超越（队列满，queue的"x-max-length"参数）

#### 生产者

1：导入依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-devtools</artifactId>
  <scope>runtime</scope>
</dependency>
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <optional>true</optional>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
```

2：配置

```properties
#服务器
server.port=7878

#配置rabbitmq
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin
spring.rabbitmq.virtual-host=/

# 定义交换机
mq.config.exchange=log.dead.direct

# 设置路由key
mq.config.routekey=red.dead.routing.key
```

3：消息发送

```java
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class Sender {
  // 初始化一个rabbitmq的实例对象
  @Autowired
  private AmqpTemplate amqpTemplate;
  @Autowired
  private RabbitTemplate rabbitTemplate;
  @Value("${mq.config.exchange}")
  private String exchangeName;
  @Value("${mq.config.routekey}")
  private String routekey;

    public void sendMessage(int i) throws InterruptedException {
    //下单
    String msg = i + " direct--订单保存成功，去发送短信和微信.....";
    amqpTemplate.convertAndSend(exchangeName,routekey, msg);
    System.out.println(msg);
 }
}
```

#### 消费者

1：配置

```properties
#服务器
server.port=7879

#配置rabbitmq
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin
spring.rabbitmq.virtual-host=/

# 定义交换机
mq.config.exchange=log.dead.direct
# 红包队列
mq.config.queue.red=order.dead.red.log

# 设置路由key
mq.config.red.routekey=red.dead.routing.key

# 解决消息的ACK死循环问题--自动确认
#spring.rabbitmq.listener.simple.retry.enabled=true
# 最多重试几次，但是在实际的生产中如果不重要的可以这样做，
#spring.rabbitmq.listener.simple.retry.max-attempts=3
# 消息多次消费的间隔1秒
#spring.rabbitmq.listener.simple.retry.initial-interval=1000

# 设置为false，会丢弃消息或者重新发布到死信队列
spring.rabbitmq.listener.simple.default-requeue-rejected=false
```

2：死信配置，重定向队列已经绑定关系

```java
import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.boot.SpringBootConfiguration;

import java.util.HashMap;
import java.util.Map;

/**
 * 死信队列的配置
 */
@SpringBootConfiguration
public class RabbitDeadLetterConfig {

    // 定义死信队列的交换机的名字
    public static final String DEAD_LETTER_EXCHANGE = "TDL_EXCHANGE";
    // 死信队列的路由key
    public static final String DEAD_LETTER_TEST_ROUTING_KEY = "TDL_KEY";
    // 死信队列
    public static final String DEAD_LETTER_QUEUE = "TDL_QUEUE";

    // 重定向队列路由key
    public static final String DEAD_LETTER_REDIRECT_ROUTING_KEY = "TKEY_R";
    // 重定向队列
    public static final String REDIRECT_QUEUE = "TREDIRECT_QUEUE";

    /**
     * 死信队列跟交换机类型没有关系 不一定为directExchange  不影响该类型交换机的特性.
     */
    @Bean("deadLetterExchange")
    public Exchange deadLetterExchange() {
        return ExchangeBuilder.directExchange(DEAD_LETTER_EXCHANGE).durable(true).build();
    }

    @Bean("deadLetterQueue")
    public Queue deadLetterQueue() {
        Map<String, Object> args = new HashMap<>(2);
        //x-dead-letter-exchange 声明  死信队列Exchange
        args.put("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE);
        //x-dead-letter-routing-key    声明 死信队列抛出异常重定向队列的routingKey(TKEY_R)
        args.put("x-dead-letter-routing-key", DEAD_LETTER_REDIRECT_ROUTING_KEY);
        return QueueBuilder.durable(DEAD_LETTER_QUEUE).withArguments(args).build();
    }

    @Bean("redirectQueue")
    public Queue redirectQueue() {
        return QueueBuilder.durable(REDIRECT_QUEUE).build();
    }

    /**
     * 死信队列绑定到死信交换器上.
     *
     * @return the binding
     */
    @Bean
    public Binding deadLetterBinding() {
        return new Binding(DEAD_LETTER_QUEUE, Binding.DestinationType.QUEUE, DEAD_LETTER_EXCHANGE, DEAD_LETTER_TEST_ROUTING_KEY, null);

    }
    
    /**
     * 将重定向队列通过routingKey(TKEY_R)绑定到死信队列的Exchange上
     *
     * @return the binding
     */
    @Bean
    public Binding redirectBinding() {
        return new Binding(REDIRECT_QUEUE, Binding.DestinationType.QUEUE, DEAD_LETTER_EXCHANGE, DEAD_LETTER_REDIRECT_ROUTING_KEY, null);
    }
}
```

3：消息消费类

```java
import com.icoding.springbootrabbitfanoutconsumber.config.RabbitDeadLetterConfig;
import com.rabbitmq.client.Channel;
import lombok.extern.log4j.Log4j2;
import org.springframework.amqp.core.ExchangeTypes;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Service;

@Service
// 用RabbitListener来完成队列和交换机的定义和绑定关系以及设置交换机的类型
// autoDelete= true代表非持久化  false 持久化
@RabbitListener(bindings = @QueueBinding(
        value = @Queue(value="${mq.config.queue.red}",autoDelete ="false"),
        exchange =@Exchange(value = "${mq.config.exchange}",type = ExchangeTypes.DIRECT),
        key = "${mq.config.red.routekey}"
))
@Log4j2
public class RedxinReceiver {

    @Autowired
    RabbitTemplate rabbitTemplate;

    @RabbitHandler
    public void getMessage(String msg, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag, Message message){
        try{
            System.out.println("red------接收到订单的消息了,内容是：" + message);
            //接收消息出错了
            throw new RuntimeException();
        }catch(Exception ex){
            /**
             * 这里对消息重入队列做设置，例如将消息序列化缓存至 Redis, 并记录重入队列次数
             * 如果该消息重入队列次数达到一定次数，比如3次，将不再重入队列，直接拒绝
             * 这时候需要对消息做补偿机制处理
             * channel.basicNack与channel.basicReject要结合越来使用
             **/
            // 如果你的消息没用在确认之前出现了错误，redelivered是false，代表消息没用确认成功，表明是第一次消费，可以直接打入到死信队列中去
            // 如果重启了消费者以后，redelivered就回变成了false说明消息已经收到但是一直没用确认，属于重复消费了。
            Boolean redelivered = message.getMessageProperties().getRedelivered();
            System.out.println("=============>" + redelivered);
            try {
                if (redelivered) {
                    /**
                     * 1. 对于重复处理的队列消息做补偿机制处理
                     * 2. 从队列中移除该消息，防止队列阻塞
                     */
                    // 消息已重复处理失败, 扔掉消息
                    channel.basicReject(message.getMessageProperties().getDeliveryTag(), false); // 拒绝消息
                    //log.error("消息[{}]重新处理失败，扔掉消息", msg);
                    //在这里预警，或者发短信
                    rabbitTemplate.convertAndSend(
                            RabbitDeadLetterConfig.DEAD_LETTER_EXCHANGE,
                            RabbitDeadLetterConfig.DEAD_LETTER_TEST_ROUTING_KEY,
                            message);
                }

                // redelivered != true,表明该消息是第一次消费
                if (!redelivered) {
                    // 消息重新放回队列
                    channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, true);
                    log.error("消息[{}]处理失败，重新放回队列", msg);
                }
            } catch (Exception exc) {
                exc.printStackTrace();
            }
        }
    }
}
```

4：死信队列消费

```java
import com.icoding.springbootrabbitfanoutproducer.mq.Sender;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class SpringbootRabbitDirectProducerDeadApplicationTests {
  @Autowired
  private Sender sender;
  @Test
  void contextLoads() throws  Exception{
    sender.sendMessage(1);
 }
}
```

5：重定向队列继续消费

死信队列

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RabbitListener(queues = RabbitDeadLetterConfig.DEAD_LETTER_QUEUE)
public class DeadLetterConsumer {
  /*@RabbitListener(bindings = @QueueBinding(
      value = @Queue(value = RabbitDeadLetterConfig.DEAD_LETTER_QUEUE, durable =
"true"),
      exchange = @Exchange(value = RabbitDeadLetterConfig.DEAD_LETTER_EXCHANGE, type =
ExchangeTypes.DIRECT),
      key = RabbitDeadLetterConfig.DEAD_LETTER_TEST_ROUTING_KEY)
  )*/
  @RabbitHandler
  public void testDeadLetterQueueAndThrowsException(String message){
    log.warn("DeadLetterConsumer ", message);
    //System.out.println(1/0);
 }
}
```

重定向队列

```java
import com.icoding.springbootrabbitfanoutconsumber.config.RabbitDeadLetterConfig;
import lombok.extern.log4j.Log4j2;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@RabbitListener(queues = RabbitDeadLetterConfig.REDIRECT_QUEUE)
@Component
@Log4j2
public class RedirectQueueConsumer {
  /**
  * 重定向队列和死信队列形参一致Integer number
  */
  @RabbitHandler
  public void fromDeadLetter(String message){
    log.warn("RedirectQueueConsumer : {}", message);
    // 对应的操作
    System.out.println(1/0);
 }
}
```

#### 测试

```java
import com.icoding.springbootrabbitfanoutproducer.mq.Sender;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class SpringbootRabbitDirectProducerDeadApplicationTests {
    @Autowired
    private Sender sender;
  	@Test
  	void contextLoads() throws  Exception{
    	sender.sendMessage(1);
 	}
}
```

## 13：RabbitMQ通信为什么是信道(Channel)? 为什么不是TCP直接通信
#### 概述

关于rabbitmq的链接和信道测试。如果消息发送完毕，connection和channel就会关闭。

1：修改Sender.java类

```java
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import java.util.Date;

@Component
public class Sender {
    @Autowired
  	private AmqpTemplate amqpTemplate;
  	public void sendMessage() throws InterruptedException{
   		while (true){
     		Thread.sleep(2000);
		    String  msg = "Hello rabbitmq " + new Date();
		    amqpTemplate.convertAndSend("item-service-queue",msg);
   		}
 	}
}
```

2：点击测试运行

```java
import com.itbooking.springbootrabbitmqdemo.mq.Sender;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootRabbitmqDemoApplicationTests {
  @Autowired
  private Sender sender;
  @Test
  public void contextLoads() throws InterruptedException {
    sender.sendMessage();
  }
}
```

3：运行测试结果

![image-20200519175638191](https://i.loli.net/2020/05/19/nZvFHIsA9TlQ8Sb.png)

![image-20200519175647431](https://i.loli.net/2020/05/19/DLZ2Q1HTl5fuI3e.png)

## 附加

1：Springboot && RabbitMQ配置

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@Slf4j
public class RabbitmqConfig {

    /*注入rabbitmq连接池*/
    @Autowired
    private CachingConnectionFactory connectionFactory;

    @Bean
    public DirectExchange game_direct_exchange(){
        return new DirectExchange("game_direct_exchange");
    }
    @Bean
    public TopicExchange game_topic_exchange(){
        return new TopicExchange("game_topic_exchange");
    }
    @Bean
    public FanoutExchange game_fanout_exchange(){
        return new FanoutExchange("game_fanout_exchange");
    }

    @Bean
    public Queue game_direct_queue(){
        return new Queue("game_direct_queue");
    }

    @Bean
    public Queue game_topic_queue(){
        return new Queue("game_topic_queue");
    }
    @Bean
    public Queue game_topic_queue2(){
        return new Queue("game_topic_queue2");
    }

    @Bean
    public Queue game_fanout_queue(){
        return new Queue("game_fanout_queue");
    }

    @Bean
    public Binding game_direct_binding(Queue game_direct_queue,DirectExchange game_direct_exchange){
        return BindingBuilder.bind(game_direct_queue).to(game_direct_exchange).with("game_direct_key");
    }

    @Bean
    //如果参数名称不适用定义队列或交换器名称时，需要使用注解标明
    public Binding game_topic_binding(@Qualifier("game_topic_queue") Queue queue,@Qualifier("game_topic_exchange") TopicExchange topicExchange){
        return BindingBuilder.bind(queue).to(topicExchange).with("game.topic.*");//匹配game.topic.xx所有的路由键
//        return BindingBuilder.bind(queue).to(topicExchange).with("game.topic.#");//匹配game.topic.xx或game.topic.xx.xx等多个后缀所有的路由键
    }

    @Bean
    //如果参数名称不适用定义队列或交换器名称时，需要使用注解标明
    public Binding game_topic_binding2(@Qualifier("game_topic_queue2") Queue queue,@Qualifier("game_topic_exchange") TopicExchange topicExchange){
        return BindingBuilder.bind(queue).to(topicExchange).with("game.topic.news");//匹配game.topic.xx所有的路由键
//        return BindingBuilder.bind(queue).to(topicExchange).with("game.topic.#");//匹配game.topic.xx或game.topic.xx.xx等多个后缀所有的路由键
    }

    @Bean
    public Binding game_fanout_binding(Queue game_fanout_queue,FanoutExchange game_fanout_exchange){
        return BindingBuilder.bind(game_fanout_queue).to(game_fanout_exchange);
    }

    /*使用spring提供的rabbitmqTemplate,在默认的情况下，是不需要进行任何配置就可以进行使用的
    * 只有在以下情况下才需要进行配置
    * 1.使用confirm-callBack确认信息从生产者到达了交换器
    * 2.使用return-callBack确认信息从交换器到达了队列*/
    @Bean
    public RabbitTemplate rabbitTemplate(){
        //若使用confirm-callback或return-callback，必须要配置publisherConfirms或publisherReturns为true
        //每个rabbitTemplate只能有一个confirm-callback和return-callback，如果这里配置了，那么写生产者的时候不能再写confirm-callback和return-callback
        //使用return-callback时必须设置mandatory为true，或者在配置中设置mandatory-expression的值为true
        connectionFactory.setPublisherConfirms(true);
        connectionFactory.setPublisherReturns(true);
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMandatory(true);
        //        /**
//         * 如果消息没有到exchange,则confirm回调,ack=false
//         * 如果消息到达exchange,则confirm回调,ack=true
//         * exchange到queue成功,则不回调return
//         * exchange到queue失败,则回调return(需设置mandatory=true,否则不回回调,消息就丢了)
//         */
        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                if(ack){
                    log.info("消息推送到交换器成功:correlationData({}),ack({}),cause({})",correlationData,ack,cause);
                }else{
                    log.info("消息推送到交换器失败:correlationData({}),ack({}),cause({})",correlationData,ack,cause);
                }
            }
        });

        rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback() {
            @Override
            public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
                log.info("消息推送到队列丢失:exchange({}),route({}),replyCode({}),replyText({}),message:{}",exchange,routingKey,replyCode,replyText,message);
            }
        });

        return rabbitTemplate;
    }
}
```

## 面试题：Rabbitmq 为什么需要信道，为什么不是TCP直接通信

> 1. TCP的创建和销毁，开销大，创建要三次握手，销毁要4次分手。
> 2. 如果不用信道，那应用程序就会TCP连接到Rabbit服务器，高峰时每秒成千上万连接就会造成资源的巨大浪费，而且**操作系统每秒处理tcp连接数也是有限制的**，必定造成性能瓶颈。
> 3. 信道的原理是一条线程一条信道，多条线程多条信道同用一条TCP连接，一条TCP连接可以容纳无限的信道，即使每秒成千上万的请求也不会成为性能瓶颈。

## 资料

[RabbitMQ测试源码](http://allen001.github.io/data/rabbitmq-combat/RabbitMQ测试源码.zip)

[死信队列](http://allen001.github.io/data/rabbitmq-combat/死信队列.zip)