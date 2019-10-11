---
title: 【RabbitMQ】如何进行消息可靠投递【上篇】
date: 2019-08-19 22:11:33
tags:
- RabbitMQ
categorys:
- 编程
---

## 说明

前几天，突然发生线上报警，钉钉连发了好几条消息，一看是RabbitMQ相关的消息，心头一紧，难道翻车了？

![u=1091165172,1855706818&fm=26&gp=0.jpg](https://i.loli.net/2019/08/19/o3ZELv7GkRImstJ.jpg)

```java
[橙色报警] 应用[xxx]在[08-15 16:36:04]发生[错误日志异常]，alertId=[xxx]。由[org.springframework.amqp.rabbit.listener.BlockingQueueConsumer:start:620]触发。
应用xxx 可能原因如下
服务名为：
 异常为：org.springframework.amqp.rabbit.listener.BlockingQueueConsumer:start:620
 产生原因如下:
1.org.springframework.amqp.rabbit.listener.QueuesNotAvailableException: Cannot prepare queue for listener. Either the queue doesn't exist or the broker will not allow us to use it.
||Consumer received fatal=false exception on startup:
...
应用xxx 可能原因如下
服务名为：
 异常为：org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer$AsyncMessageProcessingConsumer:run:1160
 产生原因如下:
1.Stopping container from aborted consumer||Stopping container from aborted consumer:
```

定睛一看，看样子像是消费者莫名其妙断开了连接，正逢公司搬家之际，难道是机房又双叒叕。。。。断电了？于是赶紧联系了运维，咨询RabbitMQ是否发生了调整。几分钟后，得到了运维的回复，由于一些不可描述的原因，RabbitMQ进行了重启，emmmm，虽然重启只持续了10分钟，但是导致该集群下所有消费者都挂了，需要将项目重启后才能正常进行消费。

项目重启后，一切似乎又正常运转起来，但好景不长，没过多久，工单就找上了门来，经过排查，发现是生产者在RabbitMQ重启期间消息投递失败，导致消息丢失，需要手动处理和恢复。

于是，我开始思考，如何才能进行RabbitMQ的消息可靠投递呢？特别是在这样比较极端的情况，RabbitMQ集群不可用的时候，无法投递的消息该如何处理呢？

## 可靠投递

先来说明一个概念，什么是可靠投递呢？在RabbitMQ中，一个消息从生产者发送到RabbitMQ服务器，需要经历这么几个步骤：

1. 生产者准备好需要投递的消息。
2. 生产者与RabbitMQ服务器建立连接。
3. 生产者发送消息。
4. RabbitMQ服务器接收到消息，并将其路由到指定队列。
5. RabbitMQ服务器发起回调，告知生产者消息发送成功。

所谓可靠投递，就是确保消息能够百分百从生产者发送到服务器。

![{6582FAF9-A46E-4239-810B-E1D6883ED070}.png.jpg](https://i.loli.net/2019/08/18/yHcfoILDmTh7qbd.jpg)

为了避免争议，补充说明一下，如果没有设置Mandatory参数，是不需要先路由消息才发起回调的，服务器收到消息后就会进行回调确认。

2、3、5步都是通过TCP连接进行交互，有网络调用的地方就会有事故，网络波动随时都有可能发生，不管是内部机房停电，还是外部光缆被切，网络事故无法预测，虽然这些都是小概率事件，但对于订单等敏感数据处理来说，这些情况下导致消息丢失都是不可接受的。

![20170716034945131.jpg](https://i.loli.net/2019/08/19/Fwp87UqONIKCZsJ.jpg)

## RabbitMQ中的消息可靠投递

默认情况下，发送消息的操作是不会返回任何信息给生产者的，也就是说，默认情况下生产者是不知道消息有没有正确地到达服务器。

那么如何解决这个问题呢？

对此，RabbitMQ中有一些相关的解决方案：

1. 使用事务机制来让生产者感知消息被成功投递到服务器。
2. 通过生产者确认机制实现。

在RabbitMQ中，所有确保消息可靠投递的机制都会对性能产生一定影响，如使用不当，可能会对吞吐量造成重大影响，只有通过执行性能基准测试，才能在确定性能与可靠投递之间的平衡。

在使用可靠投递前，需要先思考以下问题：

1. 消息发布时，保证消息进入队列的重要性有多高？
2. 如果消息无法进行路由，是否应该将该消息返回给发布者？
3. 如果消息无法被路由，是否应该将其发送到其他地方稍后再重新进行路由？
4. 如果RabbitMQ服务器崩溃了，是否可以接受消息丢失？
5. RabbitMQ在处理新消息时是否应该确认它已经为发布者执行了所有请求的路由和持久化？
6. 消息发布者是否可以批量投递消息？
7. 在可靠投递上是否有可以接受的平衡性？是否可以接受一部分的不可靠性来提升性能？

只考虑平衡性不考虑性能是不行的，至于这个平衡的度具体如何把握，就要具体情况具体分析了，比如像订单数据这样敏感的信息，对可靠性的要求自然要比一般的业务消息对可靠性的要求高的多，因为订单数据是跟钱直接相关的，可能会导致直接的经济损失。

所以不仅应该知道有哪些保证消息可靠性的解决方案，还应该知道每种方案对性能的影响程度，以此来进行方案的选择。

### RabbitMQ的事务机制

RabbitMQ是支持AMQP事务机制的，在生产者确认机制之前，事务是确保消息被成功投递的唯一方法。

在SpringBoot项目中，使用RabbitMQ事务其实很简单，只需要声明一个事务管理的Bean，并将RabbitTemplate的事务设置为true即可。

配置文件如下：

```yml
spring:
  rabbitmq:
    host: localhost
    password: guest
    username: guest
    listener:
      type: simple
      simple:
        default-requeue-rejected: false
        acknowledge-mode: manual
```

先来配置一下交换机和队列，以及事务管理器。

```java
@Configuration
public class RabbitMQConfig {

    public static final String BUSINESS_EXCHANGE_NAME = "rabbitmq.tx.demo.simple.business.exchange";
    public static final String BUSINESS_QUEUEA_NAME = "rabbitmq.tx.demo.simple.business.queue";

    // 声明业务Exchange
    @Bean("businessExchange")
    public FanoutExchange businessExchange(){
        return new FanoutExchange(BUSINESS_EXCHANGE_NAME);
    }

    // 声明业务队列
    @Bean("businessQueue")
    public Queue businessQueue(){
        return QueueBuilder.durable(BUSINESS_QUEUEA_NAME).build();
    }

    // 声明业务队列绑定关系
    @Bean
    public Binding businessBinding(@Qualifier("businessQueue") Queue queue,
                                    @Qualifier("businessExchange") FanoutExchange exchange){
        return BindingBuilder.bind(queue).to(exchange);
    }


    /**
     * 配置启用rabbitmq事务
     * @param connectionFactory
     * @return
     */
    @Bean
    public RabbitTransactionManager rabbitTransactionManager(CachingConnectionFactory connectionFactory) {
        return new RabbitTransactionManager(connectionFactory);
    }
}
```

然后创建一个消费者，来监听消息，用以判断消息是否成功发送。

```java
@Slf4j
@Component
public class BusinessMsgConsumer {


    @RabbitListener(queues = BUSINESS_QUEUEA_NAME)
    public void receiveMsg(Message message, Channel channel) throws IOException {
        String msg = new String(message.getBody());
        log.info("收到业务消息：{}", msg);
        channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
    }
}
```

然后是消息生产者：

```java
@Slf4j
@Component
public class BusinessMsgProducer{

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @PostConstruct
    private void init() {
        rabbitTemplate.setChannelTransacted(true);
    }

    @Transactional
    public void sendMsg(String msg) {
        rabbitTemplate.convertAndSend(BUSINESS_EXCHANGE_NAME, "key", msg);
        if (msg != null && msg.contains("exception"))
            throw new RuntimeException("surprise!");
        log.info("消息已发送 {}" ,msg);
    }
}
```

这里有两个注意的地方：

1. 在初始化方法里，通过使用` rabbitTemplate.setChannelTransacted(true);` 来开启事务。
2. 在发送消息的方法上加上 `@Transactional` 注解，这样在该方法中发生异常时，消息将不会发送。

在controller中加一个接口来生产消息：

```java
@RestController
public class BusinessController {

    @Autowired
    private BusinessMsgProducer producer;

    @RequestMapping("send")
    public void sendMsg(String msg){
        producer.sendMsg(msg);
    }
}
```

来验证一下：

```shell
msg:1
消息已发送 1
收到业务消息：1
msg:2
消息已发送 2
收到业务消息：2
msg:3
消息已发送 3
收到业务消息：3
msg:exception

Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.RuntimeException: surprise!] with root cause

java.lang.RuntimeException: surprise!
	at com.mfrank.rabbitmqdemo.producer.BusinessMsgProducer.sendMsg(BusinessMsgProducer.java:30)
    ...
```

当 `msg` 的值为 `exception` 时， 在调用`rabbitTemplate.convertAndSend` 方法之后，程序抛出了异常，消息并没有发送出去，而是被当前事务回滚了。

当然，你可以将事务管理器注释掉，或者将初始化方法的开启事务注释掉，这样事务就不会生效，即使在调用了发送消息方法之后，程序发生了异常，消息也会被正常发送和消费。

RabbitMQ中的事务使用起来虽然简单，但是对性能的影响是不可忽视的，因为每次事务的提交都是阻塞式的等待服务器处理返回结果，而默认模式下，客户端是不需要等待的，直接发送就完事了，除此之外，事务消息需要比普通消息多4次与服务器的交互，这就意味着会占用更多的处理时间，所以如果对消息处理速度有较高要求时，尽量不要采用事务机制。

### RabbitMQ的生产者确认机制

RabbitMQ中的生产者确认功能是AMQP规范的增强功能，当生产者发布给所有队列的已路由消息被消费者应用程序直接消费时，或者消息被放入队列并根据需要进行持久化时，一个Basic.Ack请求会被发送到生产者，如果消息无法路由，代理服务器将发送一个Basic.Nack RPC请求用于表示失败。然后由生产者决定该如何处理该消息。

也就是说，通过生产者确认机制，生产者可以在消息被服务器成功接收时得到反馈，并有机会处理未被成功接收的消息。

在Springboot中开启RabbitMQ的生产者确认模式也很简单，只多了一行配置：

```yml
spring:
  rabbitmq:
    host: localhost
    password: guest
    username: guest
    listener:
      type: simple
      simple:
        default-requeue-rejected: false
        acknowledge-mode: manual
    publisher-confirms: true
```

` publisher-confirms: true` 即表示开启生产者确认模式。

然后将消息生产者的代表进行部分修改：

```java
@Slf4j
@Component
public class BusinessMsgProducer implements RabbitTemplate.ConfirmCallback{

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @PostConstruct
    private void init() {
//        rabbitTemplate.setChannelTransacted(true);
        rabbitTemplate.setConfirmCallback(this);
    }

    public void sendCustomMsg(String exchange, String msg) {
        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());

        log.info("消息id:{}, msg:{}", correlationData.getId(), msg);

        rabbitTemplate.convertAndSend(exchange, "key", msg, correlationData);
    }
    
    @Override
    public void confirm(CorrelationData correlationData, boolean b, String s) {
        String id = correlationData != null ? correlationData.getId() : "";
        if (b) {
            log.info("消息确认成功, id:{}", id);
        } else {
            log.error("消息未成功投递, id:{}, cause:{}", id, s);
        }
    }
}
```

让生产者继承自`RabbitTemplate.ConfirmCallback` 类，然后实现其`confirm` 方法，即可用其接收服务器回调。

需要注意的是，在发送消息时，代码也进行了调整：

```java
CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());

rabbitTemplate.convertAndSend(exchange, "key", msg, correlationData);
```

这里我们为消息设置了消息ID，以便在回调时通过该ID来判断是对哪个消息的回调，因为在回调函数中，我们是无法直接获取到消息内容的，所以需要将消息先暂存起来，根据消息的重要程度，可以考虑使用本地缓存，或者存入Redis中，或者Mysql中，然后在回调时更新其状态或者从缓存中移除，最后使用定时任务对一段时间内未发送的消息进行重新投递。

以下是我盗来的图，原谅我偷懒不想画了[手动狗头]：

![5b65729e0001439305000294.jpg](https://i.loli.net/2019/08/19/CO5bs2xwh4WHFpM.png)

另外，还需要注意的是，如果将消息发布到不存在的交换机上，那么发布用的信道将会被RabbitMQ关闭。

此外，生产者确认机制跟事务是不能一起工作的，是事务的轻量级替代方案。因为事务和发布者确认模式都是需要先跟服务器协商，对信道启用的一种模式，不能对同一个信道同时使用两种模式。

在生产者确认模式中，消息的确认可以是异步和批量的，所以相比使用事务，性能会更好。

使用事务机制和生产者确认机制都能确保消息被正确的发送至RabbitMQ，这里的“正确发送至RabbitMQ”说的是消息成功被交换机接收，但如果找不到能接收该消息的队列，这条消息也会丢失。至于如何处理那些无法被投递到队列的消息，将会在下篇进行说明。

## 结题

所以当公司机房“断电”时，如何处理那些需要发送的消息呢？相信看完上文之后，你的心中已经有了答案。

一般来说，这种“断电”不会持续较长时间，一般几分钟到半小时之间，很快能够恢复，所以如果是重要消息，可以保存到数据库中，如果是非重要消息，可以使用redis进行保存，当然，还要根据消息的数量级来进行判断。

如果消息量比较大，可以考虑将消息发送到另一个集群的死信队列中，事实上，所在公司就有两个RabbitMQ集群，所以当一个集群不可用时，可以往另一个集群发消息，emmm，如果两个机房都停电了的话，当我没说。

![111.png.jpg](https://i.loli.net/2019/08/19/c8ZGAL7OEPQTeWM.jpg)
