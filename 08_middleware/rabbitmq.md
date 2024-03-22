## rabbitmq
rabbitmq 是一个基于 erlang 开发的可靠的、成熟的消息队列。

* 易用 ：支持 AMQP 1.0 和 MQTT 5 等协议。多种编程语言的客户端可用。
* 灵活 ：灵活的消息发布和消费模式。 Routing, filtering, streaming, federation
* 可靠 ：完善的消息传输确认机制和消息备份机制保证消息的安全

## 使用场景

* 解耦服务依赖
* RPC 。向一个队列发送请求，订阅另一个队列获得返回结果。
* 流式处理
* IoT 物联网


## 简单使用
1. producer 发送消息到一个队列。 consumer 从这个队列接受消息

## Work Queues
一个 producer ，一个队列，  多个 consumer 。
* 默认使用Round-robin 算法分配，轮询分配。 每次分配重试3次，3次之后切换  consumer
* 消息确认 。 默认 autoAck=true； 手动
* 消息持久化
```
boolean durable = true;
channel.queueDeclare("task_queue", durable, false, false, null);
```
* 公平调度: 设置 prefetchCount = 1 ， 每次只发送一个消息到一个 consumer，在 consumer 处理完成之前， 不会向 consumer 发送新的消息， 不会造成性能不均衡。
```
int prefetchCount = 1;
channel.basicQos(prefetchCount);
```

## Publish/Subscribe
* 发送消息到多个 consumer， 广播
* 定义一个 exchange， 多个queue， 队列都绑定到这个 exchange。exchange 的模式是 fanout。
```
channel.exchangeDeclare("logs", "fanout");
channel.queueBind(queueName, EXCHANGE_NAME, "");
```

## Routing
* 根据 routingKey  判断消息发送到哪个对列， 一个队列可以接受一个或者多个 routingKey。
* 定义一个 exchange， 多个queue。 exchange 的模式是 direct。队列绑定exchange，时指定 routeKey 。可指定多次。
```
channel.exchangeDeclare(EXCHANGE_NAME, "direct");

channel.queueBind(queueName, EXCHANGE_NAME, routeKey);

```

## Topics
* 比 Routing 模式更灵活。 根据表达式匹配来判断消息发送到哪个对列。
* 定义一个 exchange， 多个queue。 exchange 的模式是 topic。队列绑定exchange，时指定 routeKey 。可指定多次。

```
channel.exchangeDeclare(EXCHANGE_NAME, "topic");
channel.queueBind(queueName, EXCHANGE_NAME, bindingKey);

bindingKey 示例：kern.* ； *.critical； # 
* (star) can substitute for exactly one word.
# (hash) can substitute for zero or more words.
```



## RPC 
1. 客户端发送RPC请求设置两个属性：replyTo 和 correlationId。 replyTo 是一个专门用来发送RPC请求的队列。correlationId 是每次请求的唯一ID
2. 发送请求到 一个 队列 rpc_queue
3. 服务端监听 rpc_queue， 处理请求，然后发送消息到 replyTo 指定的队列， 并且设置 correlationId 。
4. 客户端监听 replyTo 指定的队列 ，收到消息， 如果 correlationId 匹配，则返回消息到调用方。


## Publisher confirms 消息确认
* 在 channel 上开启。开启后， 消息发送成功的确认是异步返回给 producer 
```
Channel channel = connection.createChannel();
channel.confirmSelect();
```


## 
