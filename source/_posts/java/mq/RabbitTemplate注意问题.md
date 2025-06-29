---
title: RabbitTemplate注意问题
toc: true
description: 明明已经给 RabbitTemplate 配置了 MessageConverter，但是 @RabbitListener 仍然需要手动指定消息转换器，否则反序列化失败。
tags:
  - mq
  - 问题
  - java
categories:
  - mq
date: 2025-06-17 12:32:04
---
你明明已经给 RabbitTemplate 配置了 MessageConverter，但是 @RabbitListener 仍然需要手动指定消息转换器，否则反序列化失败。

一句话解释原因：
👉 @RabbitListener 默认使用的是 SimpleRabbitListenerContainerFactory 的配置，而不是你配置的 RabbitTemplate。

详细解释：
你自定义的这个 RabbitTemplate 的确使用了你的 MessageConverter：
```java
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory, MessageConverter messageConverter) {
	RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
	rabbitTemplate.setMessageConverter(messageConverter);
	return rabbitTemplate;
}
```
这个 RabbitTemplate 只影响你主动调用 .convertAndSend() 之类的行为。但 @RabbitListener 用的是容器工厂 RabbitListenerContainerFactory，默认是 Spring Boot 自动帮你创建的，不会自动把你在 RabbitTemplate 中配置的 converter 套进来。

正确做法：自定义 Listener 容器工厂
你可以显式地配置 SimpleRabbitListenerContainerFactory，并设置你自己的消息转换器，比如 JSON：
```java
@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
        ConnectionFactory connectionFactory,
        MessageConverter messageConverter) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setMessageConverter(messageConverter); // 💡关键就在这
    factory.setConcurrentConsumers(10); // 可选：设置并发消费者数量
    factory.setMaxConcurrentConsumers(60); // 可选
    return factory;
}
```
然后让你的 @RabbitListener 使用这个 factory：
```java
@RabbitListener(queues = RabbitMQConfig.QUEUE_NAME, containerFactory = "rabbitListenerContainerFactory")
public void handleMessage(YourObject obj) {
    // 处理反序列化后的对象
}
```
其他建议：
如果你用了 Jackson2JsonMessageConverter，记得注入：
```java
@Bean
public MessageConverter messageConverter() {
    return new Jackson2JsonMessageConverter();
}
```
小结：
配置项	作用
RabbitTemplate	你主动发消息时的消息转换器
RabbitListenerContainerFactory	消费端监听器使用的转换器
所以要让 @RabbitListener 自动反序列化成你的 POJO，不是改 RabbitTemplate，而是要配置 容器工厂里的 converter！

