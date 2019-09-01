---
title: 【RabbitMQ】如何进行消息可靠投递【下篇】
date: 2019-09-01 16:16:56
tags:
- RabbitMQ
categorys:
- 编程
---

## 说明

上一篇文章里，我们了解了如何保证消息被可靠投递到RabbitMQ的交换机中，但还有一些不完美的地方，试想一下，如果向RabbitMQ服务器发送一条消息，服务器确实也接收到了这条消息，于是给你返回了ACK确认消息，但服务器拿到这条消息一看，找不到路由它的队列，于是就把它丢进了垃圾桶，emmm，我猜应该属于可回收垃圾。

![1](https://i.loli.net/2019/08/22/nmPFS3RkIMweEXc.png)

## 如何让消息可靠投递到队列

如果你对上面的描述还不是很清楚，那我再用代码来说明一次。

在仅开启了生产者确认机制的情况下，交换机接收到消息后，会直接给消息生产者发送确认消息，如果发现该消息不可路由，那么消息会被直接丢弃，此时，生产者是不知道消息被丢弃这个事件的。

我们将上一篇中的交换机类型改为DirectExchange，这样就只有当消息的 RoutingKey 和队列绑定时设置的 Bindingkey （这里即“key”）一致时，才会真正将该消息进行路由。

```java
public static final String BUSINESS_EXCHANGE_NAME = "rabbitmq.tx.demo.simple.business.exchange";
public static final String BUSINESS_QUEUEA_NAME = "rabbitmq.tx.demo.simple.business.queue";

// 声明业务 Exchange
@Bean("businessExchange")
public DirectExchange businessExchange(){
    return new DirectExchange(BUSINESS_EXCHANGE_NAME);
}

// 声明业务队列
@Bean("businessQueue")
public Queue businessQueue(){
    return QueueBuilder.durable(BUSINESS_QUEUEA_NAME).build();
}

// 声明业务队列绑定关系
@Bean
public Binding businessBinding(@Qualifier("businessQueue") Queue queue,
                               @Qualifier("businessExchange") DirectExchange exchange){
    return BindingBuilder.bind(queue).to(exchange).with("key");
}
```

对消息生产者也稍作修改：

```java
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

    correlationData = new CorrelationData(UUID.randomUUID().toString());

    log.info("消息id:{}, msg:{}", correlationData.getId(), msg);

    rabbitTemplate.convertAndSend(exchange, "key2", msg, correlationData);
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
```

然后我们调用该方法，发送两条消息测试一下：

```java
消息id:ba6bf502-9381-4220-8dc9-313d6a289a4e, msg:1
消息id:f0040a41-dc02-4e45-b8af-e3cfa8a118b2, msg:1
消息确认成功, id:ba6bf502-9381-4220-8dc9-313d6a289a4e
消息确认成功, id:f0040a41-dc02-4e45-b8af-e3cfa8a118b2
收到业务消息：1
```

可以看到，发送了两条消息，第一条消息的 RoutingKey 为 “key”，第二条消息的 RoutingKey 为 “key2”，两条消息都成功被交换机接收，也收到了交换机的确认回调，但消费者只收到了一条消息，因为第二条消息的 RoutingKey 与队列的 BindingKey 不一致，也没有其它队列能接收这个消息，所有第二条消息被直接丢弃了。

那么，如何让消息被路由到队列后再返回ACK呢？或者无法被路由的消息帮我想办法处理一下？最起码通知我一声，我好自己处理啊。

别慌别慌，RabbitMQ里有两个机制刚好可以解决我们上面的疑问：

1、mandatory 参数
2、备份交换机

## mandatory 参数

设置 mandatory 参数可以在当消息传递过程中不可达目的地时将消息返回给生产者。

当把 mandotory 参数设置为 true 时，如果交换机无法将消息进行路由时，会将该消息返回给生产者，而如果该参数设置为false，如果发现消息无法进行路由，则直接丢弃。

![2.png](https://i.loli.net/2019/09/01/YZ8acG1HxKQ9fLN.png)

那么如何设置这个参数呢？在发送消息的时候，只需要在初始化方法添加一行代码即可：

```java
rabbitTemplate.setMandatory(true);
```

开启之后我们再重新运行前面的代码：

```java
消息id:19729f33-15c4-4c1b-8d48-044c301e2a8e, msg:1
消息id:4aea5c57-3e71-4a7b-8a00-1595d2b568eb, msg:1
消息确认成功, id:19729f33-15c4-4c1b-8d48-044c301e2a8e
Returned message but no callback available
消息确认成功, id:4aea5c57-3e71-4a7b-8a00-1595d2b568eb
收到业务消息：1
```

我们看到中间多了一行提示 `Returned message but no callback available` 这是什么意思呢？

我们上面提到，设置 mandatory 参数后，如果消息无法被路由，则会返回给生产者，是通过回调的方式进行的，所以，生产者需要设置相应的回调函数才能接受该消息。

为了进行回调，我们需要实现一个接口 `RabbitTemplate.ReturnCallback`。

```java
@Slf4j
@Component
public class BusinessMsgProducer implements RabbitTemplate.ConfirmCallback, RabbitTemplate.ReturnCallback{

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @PostConstruct
    private void init() {
        rabbitTemplate.setConfirmCallback(this);
        rabbitTemplate.setMandatory(true);
        rabbitTemplate.setReturnCallback(this);
    }

    public void sendCustomMsg(String exchange, String msg) {
        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());

        log.info("消息id:{}, msg:{}", correlationData.getId(), msg);

        rabbitTemplate.convertAndSend(exchange, "key", msg, correlationData);

        correlationData = new CorrelationData(UUID.randomUUID().toString());

        log.info("消息id:{}, msg:{}", correlationData.getId(), msg);

        rabbitTemplate.convertAndSend(exchange, "key2", msg, correlationData);
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

    @Override
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
        log.info("消息被服务器退回。msg:{}, replyCode:{}. replyText:{}, exchange:{}, routingKey :{}",
                new String(message.getBody()), replyCode, replyText, exchange, routingKey);
    }
}
```

然后我们再来重新运行一次：

```java
消息id:2e5c336a-883a-474e-b40e-b6e3499088ef, msg:1
消息id:85c771cb-c88f-47dd-adea-f0da57138423, msg:1
消息确认成功, id:2e5c336a-883a-474e-b40e-b6e3499088ef
消息无法被路由，被服务器退回。msg:1, replyCode:312. replyText:NO_ROUTE, exchange:rabbitmq.tx.demo.simple.business.exchange, routingKey :key2
消息确认成功, id:85c771cb-c88f-47dd-adea-f0da57138423
收到业务消息：1
```

可以看到，我们接收到了被退回的消息，并带上了消息被退回的原因：`NO_ROUTE`。但是要注意的是， mandatory 参数仅仅是在当消息无法被路由的时候，让生产者可以感知到这一点，只要开启了生产者确认机制，无论是否设置了 mandatory 参数，都会在交换机接收到消息时进行消息确认回调，而且通常消息的返回回调会在消息的确认回调之前。

## 备份交换机

有了 mandatory 参数，我们获得了对无法投递消息的感知能力，有机会在生产者的消息无法被投递时发现并处理。但有时候，我们并不知道该如何处理这些无法路由的消息，最多打个日志，然后触发报警，再来手动处理。而通过日志来处理这些无法路由的消息是很不优雅的做法，特别是当生产者所在的服务有多台机器的时候，手动复制日志会更加麻烦而且容易出错。

而且设置 mandatory 参数会增加生产者的复杂性，需要添加处理这些被退回的消息的逻辑。如果既不想丢失消息，又不想增加生产者的复杂性，该怎么做呢？

前面在设置死信队列的文章中，我们提到，可以为队列设置死信交换机来存储那些处理失败的消息，可是这些不可路由消息根本没有机会进入到队列，因此无法使用死信队列来保存消息。

不要慌，在 RabbitMQ 中，有一种备份交换机的机制存在，可以很好的应对这个问题。

什么是备份交换机呢？备份交换机可以理解为 RabbitMQ 中交换机的“备胎”，当我们为某一个交换机声明一个对应的备份交换机时，就是为它创建一个备胎，当交换机接收到一条不可路由消息时，将会将这条消息转发到备份交换机中，由备份交换机来进行转发和处理，通常备份交换机的类型为 Fanout ，这样就能把所有消息都投递到与其绑定的队列中，然后我们在备份交换机下绑定一个队列，这样所有那些原交换机无法被路由的消息，就会都进入这个队列了。当然，我们还可以建立一个报警队列，用独立的消费者来进行监测和报警。

听的不太明白？没关系，看个图就知道是怎么回事了。

![3.png](https://i.loli.net/2019/09/01/VsGpZJbk2u7wSv4.png)

（emmm，调整了一下配色，感觉还是很丑- - 。急需一个UI来拯救我。）

接下来，我们就来设置一下备份交换机：

```java
@Configuration
public class RabbitMQConfig {

    public static final String BUSINESS_EXCHANGE_NAME = "rabbitmq.backup.test.exchange";
    public static final String BUSINESS_QUEUE_NAME = "rabbitmq.backup.test.queue";
    public static final String BUSINESS_BACKUP_EXCHANGE_NAME = "rabbitmq.backup.test.backup-exchange";
    public static final String BUSINESS_BACKUP_QUEUE_NAME = "rabbitmq.backup.test.backup-queue";
    public static final String BUSINESS_BACKUP_WARNING_QUEUE_NAME = "rabbitmq.backup.test.backup-warning-queue";

    // 声明业务 Exchange
    @Bean("businessExchange")
    public DirectExchange businessExchange(){
        ExchangeBuilder exchangeBuilder = ExchangeBuilder.directExchange(BUSINESS_EXCHANGE_NAME)
                .durable(true)
                .withArgument("alternate-exchange", BUSINESS_BACKUP_EXCHANGE_NAME);

        return (DirectExchange)exchangeBuilder.build();
    }

    // 声明备份 Exchange
    @Bean("backupExchange")
    public FanoutExchange backupExchange(){
        ExchangeBuilder exchangeBuilder = ExchangeBuilder.fanoutExchange(BUSINESS_BACKUP_EXCHANGE_NAME)
                .durable(true);
        return (FanoutExchange)exchangeBuilder.build();
    }

    // 声明业务队列
    @Bean("businessQueue")
    public Queue businessQueue(){
        return QueueBuilder.durable(BUSINESS_QUEUE_NAME).build();
    }

    // 声明业务队列绑定关系
    @Bean
    public Binding businessBinding(@Qualifier("businessQueue") Queue queue,
                                    @Qualifier("businessExchange") DirectExchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with("key");
    }

    // 声明备份队列
    @Bean("backupQueue")
    public Queue backupQueue(){
        return QueueBuilder.durable(BUSINESS_BACKUP_QUEUE_NAME).build();
    }

    // 声明报警队列
    @Bean("warningQueue")
    public Queue warningQueue(){
        return QueueBuilder.durable(BUSINESS_BACKUP_WARNING_QUEUE_NAME).build();
    }

    // 声明备份队列绑定关系
    @Bean
    public Binding backupBinding(@Qualifier("backupQueue") Queue queue,
                                   @Qualifier("backupExchange") FanoutExchange exchange){
        return BindingBuilder.bind(queue).to(exchange);
    }

    // 声明备份报警队列绑定关系
    @Bean
    public Binding backupWarningBinding(@Qualifier("warningQueue") Queue queue,
                                 @Qualifier("backupExchange") FanoutExchange exchange){
        return BindingBuilder.bind(queue).to(exchange);
    }

}
```

这里我们使用 `ExchangeBuilder` 来声明交换机，并为其设置备份交换机：

```java
 .withArgument("alternate-exchange", BUSINESS_BACKUP_EXCHANGE_NAME);
```

为业务交换机绑定了一个队列，为备份交换机绑定了两个队列，一个专门用来做报警用途。

接下来，分别为业务交换机和备份交换机创建消费者：

```java
@Slf4j
@Component
public class BusinessMsgConsumer {

    @RabbitListener(queues = BUSINESS_QUEUE_NAME)
    public void receiveMsg(Message message, Channel channel) throws IOException {
        String msg = new String(message.getBody());
        log.info("收到业务消息：{}", msg);
        channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
    }
}
```

```java
@Slf4j
@Component
public class BusinessWaringConsumer {

    @RabbitListener(queues = BUSINESS_BACKUP_WARNING_QUEUE_NAME)
    public void receiveMsg(Message message, Channel channel) throws IOException {
        String msg = new String(message.getBody());
        log.error("发现不可路由消息：{}", msg);
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    }
}
```

接下来我们分别发送一条可路由消息和不可路由消息：

```java
@Slf4j
@Component
public class BusinessMsgProducer {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void sendCustomMsg(String exchange, String msg) {


        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());


        log.info("消息id:{}, msg:{}", correlationData.getId(), msg);

        rabbitTemplate.convertAndSend(exchange, "key", msg, correlationData);

        correlationData = new CorrelationData(UUID.randomUUID().toString());

        log.info("消息id:{}, msg:{}", correlationData.getId(), msg);

        rabbitTemplate.convertAndSend(exchange, "key2", msg, correlationData);
    }
}
```

消息如下：

```java
消息id:5c3a33c9-0764-4d1f-bf6a-a00d771dccb4, msg:1
消息id:42ac8c35-1d0a-4413-a1df-c26a85435354, msg:1
收到业务消息：1
发现不可路由消息：1
```

这里仅仅使用 error 日志配合日志系统进行报警，如果是敏感数据，可以使用邮件、钉钉、短信、电话等报警方式来提高时效性。

那么问题来了，mandatory 参数与备份交换机可以一起使用吗？设置 mandatory 参数会让交换机将不可路由消息退回给生产者，而备份交换机会让交换机将不可路由消息转发给它，那么如果两者同时开启，消息究竟何去何从？？

emmm，想这么多干嘛，试试不就知道了。

修改一下生产者即可：

```java
@Slf4j
@Component
public class BusinessMsgProducer implements RabbitTemplate.ConfirmCallback, RabbitTemplate.ReturnCallback{

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @PostConstruct
    private void init() {
//        rabbitTemplate.setChannelTransacted(true);
        rabbitTemplate.setConfirmCallback(this);
        rabbitTemplate.setMandatory(true);
        rabbitTemplate.setReturnCallback(this);
    }

    public void sendCustomMsg(String exchange, String msg) {


        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());


        log.info("消息id:{}, msg:{}", correlationData.getId(), msg);

        rabbitTemplate.convertAndSend(exchange, "key", msg, correlationData);

        correlationData = new CorrelationData(UUID.randomUUID().toString());

        log.info("消息id:{}, msg:{}", correlationData.getId(), msg);

        rabbitTemplate.convertAndSend(exchange, "key2", msg, correlationData);
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

    @Override
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
        log.info("消息被服务器退回。msg:{}, replyCode:{}. replyText:{}, exchange:{}, routingKey :{}",
                new String(message.getBody()), replyCode, replyText, exchange, routingKey);
    }
}
```

再来测试一下：

```java
消息id:0a3eca1e-d937-418c-a7ce-bfb8ce25fdd4, msg:1
消息id:d8c9e010-e120-46da-a42e-1ba21026ff06, msg:1
消息确认成功, id:0a3eca1e-d937-418c-a7ce-bfb8ce25fdd4
消息确认成功, id:d8c9e010-e120-46da-a42e-1ba21026ff06
发现不可路由消息：1
收到业务消息：1
```

可以看到，两条消息都可以收到确认成功回调，但是不可路由消息不会被回退给生产者，而是直接转发给备份交换机。可见备份交换机的处理优先级更高。

## 总结

上一篇中，我们介绍了事务机制和生产者确认机制来确保消息的可靠投递，相对而言，生产者确认机制更加高效和灵活。本篇中，我们介绍了另外两种确保生产者的消息不丢失的机制，即通过 mandatory 参数和备份交换机来处理不可路由消息。

通过以上几种机制，我们总算是可以确保消息被万无一失的投递到目的地了。到此，我们的消息可靠投递也就告一段落了。消息可靠投递是我们使用MQ时无法逃避的话题，一次性搞定它，就不会再为其所困。总的来说，方法总比问题多，但如果你不知道这些方法，那么当问题来临时，也许就会不知所措了。

相信通过这几篇关于 RabbitMQ 文章的学习，对于 RabbitMQ 的理解已经突破天际，那还在等什么，赶紧把接入 RabbitMQ 的项目好好优化一下吧，相信现在你就不会再被那些不知所云的配置和代码所迷惑了。

到此为止，本篇就完美落幕了，希望能给你带来一些启发，也欢迎关注我的公众号进行留言交流。

![1565529015677.png](https://i.loli.net/2019/08/11/8MxBPNgyIlTZDRk.png)
