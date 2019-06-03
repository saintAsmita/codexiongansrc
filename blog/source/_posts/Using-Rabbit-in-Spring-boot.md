---
title: Using Rabbit in Spring boot
date: 2019-05-23 17:28:37
tags: Rabbit Springboot
---

本文介绍如何在Spring Boot中监听MQ，分别监听单个MQ与多个MQ。主要是以代码的形式展示，默认读者对Spring boot有最基本的认识。
<!--more-->

## 监听单个MQ

此处的配置使用的是Spring boot中默认的配置。
需要写四个地方：pom引入mq依赖，yml配置mq地址等，configuration-配置java bean,  Listener - 监听MQ与处理业务逻辑。

### pom
有可能你需要加上版本号
```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
```

### yml
```yml
spring:
  rabbitmq:
    addresses: 1111.1111.111.111:1234
    username: XXXXX
    password: XXXX
    virtualHost: YYYY
    listener.simple:  
      concurrency: 20  
      max-concurrency: 30  
      prefetch: 5
```

### configuration

```java
@Configuration
public class RabbitConfiguration {
    @Bean
    public Jackson2JsonMessageConverter
    jackson2JsonMessageConverter() {
        return new Jackson2JsonMessageConverter() {
            @Override
            public Object fromMessage(Message message) throws MessageConversionException {
                //强制重写contentType 
                message.getMessageProperties().setContentType(CONTENT_TYPE_JSON);
                return super.fromMessage(message);
            }
        };
    }

    @Bean
    public PropsMqListener propsMqListener() {
        return new PropsMqListener();
    }
}
```

### listener
真正处理业务逻辑的地方

```java
@RabbitListener(bindings = {
    @QueueBinding(value = @Queue(name = "queueName"),
        exchange = @Exchange(name = "exhcangeName", type = TOPIC), key =
        "keyName")
})
public void handleRevenueConsumeEvent(PropInfo info) {
    // do something
}
```


## 监听多个MQ

由于是多个MQ，此处用自己写的yml配置，自己设置java bean。
需要写四个地方：pom引入mq依赖，yml配置mq地址等，configuration-配置java bean,  Listener - 监听MQ与处理业务逻辑。

### pom
有可能你需要加上版本号
```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
```

### yml
```yml
app:  
  rabbitmq:   
    mq-1: # rename for yourself  
      addresses: 111.111.111.111:1234    
      username: XXX     
      password: XXX    
      virtual-host: XXX    
    mq-2: # rename for yourself
      addresses: 111.111.111.111:1234     
      username: YYY     
      password: YYY      
      virtualHost: YYY
```

### configuration
```java
@Slf4j
@Configuration
public class RabbitConfiguration {
    @Bean
    public Jackson2JsonMessageConverter jackson2JsonMessageConverter() {
        return new Jackson2JsonMessageConverter() {
            @Override
            public Object fromMessage(Message message)
            throws MessageConversionException {
                //强制重写contentType
                message.getMessageProperties()
                    .setContentType(CONTENT_TYPE_JSON);

                return super.fromMessage(message);
            }
        };
    }

    @Bean
    public RabbitmqListeners rabbitmqListener() {
        return new RabbitmqListeners();
    }

    @Configuration // class rename for yourself
    public static class MqFirstRabbitMqConfiguration {
        @Bean(name = "illegalReportMqConnectionFactory")
        @ConfigurationProperties("app.rabbitmq.xh-illegal-report")
        @Primary
        public CachingConnectionFactory connectionFactory() {
            return new CachingConnectionFactory();
        }
		// Bean rename for yourself
        @Bean(name = FIRST_MQ_REPORT_MQ_CONTAINER_FACTORY)
        public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            SimpleRabbitListenerContainerFactoryConfigurer configurer) {
            SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
            configurer.configure(factory, connectionFactory());

            return factory;
        }
		// Bean rename for yourself
        @Bean(name = FRIST_MQ_REPORT_MQ_AMQP_ADMIN)
        public AmqpAdmin amqpAdmin() {
            return new RabbitAdmin(connectionFactory());
        }
        // Bean rename for yourself
        @Bean(name = FIRST_MQL_REPORT_RABBIT_TEMPLATE)
        public RabbitTemplate rabbitTemplate(MessageConverter messageConverter) {
            RabbitTemplate template = new RabbitTemplate(connectionFactory());
            template.setMessageConverter(messageConverter);

            return template;
        }
    }

    @Configuration // class rename for yourself
    public static class MqSecondRabbitMqConfiguration {
        @Bean(name = "giftPropMqConnectionFactory")
        @ConfigurationProperties("app.rabbitmq.xh-gift-prop")
        public CachingConnectionFactory connectionFactory() {
            return new CachingConnectionFactory();
        }
		// Bean rename for yourself
        @Bean(name = SECOND_MQ_CONTAINER_FACTORY)
        public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            SimpleRabbitListenerContainerFactoryConfigurer configurer) {
            SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
            configurer.configure(factory, connectionFactory());

            return factory;
        }
		// Bean rename for yourself
        @Bean(name = SECOND_MQ_AMQP_ADMIN)
        public AmqpAdmin amqpAdmin() {
            return new RabbitAdmin(connectionFactory());
        }
		// Bean rename for yourself
        @Bean(name = SECOND_RABBIT_TEMPLATE)
        public RabbitTemplate rabbitTemplate(MessageConverter messageConverter) {
            RabbitTemplate template = new RabbitTemplate(connectionFactory());
            template.setMessageConverter(messageConverter);

            return template;
        }
    }
}
```

### listerner
```java
// FIRST_MQ_REPORT_MQ_CONTAINER_FACTORY - configuration中对应Factory的bean name
@RabbitListener(containerFactory = FIRST_MQ_REPORT_MQ_CONTAINER_FACTORY, bindings = {
    @QueueBinding(value = @Queue(name = "XXX"), exchange = @Exchange(name = "XXX", type =
        TOPIC), key = "XXX")
})
public void handlePropGift(GiftMqInfo giftMqInfo) {
    // do something
}

```