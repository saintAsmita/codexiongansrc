---
title: Git的操作展示
date: 2019-05-31 18:06:37
tags: Git
---

# Git的实际操作

本文主要是介绍在实际开发过程中，对于Git的一些操作，用来给大家分享的同时，也用以记录。对于稍微复杂点的操作，我会配上图文例子
<!--more-->

## reset

`reset`用于回退暂存区的内容，例如我做如下操作，提交了`commit1`与`commit2`，此时`git status`如下展示


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