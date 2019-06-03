---
title: Spring Boot过滤多个Profile
date: 2019-06-03 20:50:37
tags:  Springboot profile
---

有时候我们需要在开发或者测试环境中屏蔽某些profile，仅仅在生产上启动，那么怎么办呢？
<!--more-->

## Spring Boot 2.1及以上

这里说的是你生产上没有配置profile，但是测试与开发环境配置了，所以需要在不是**生产**且不是**测试**的环境中才使用，对于这个版本，仅仅配置如下代码即可：
```java
@Component
@Profile("!test & !dev")
public class SpecialBean {}
```

## Spring Boot 2.1以下
这个需要用到`@Conditional`注解，自己去实现一个`Conditional`即可，如下：
```java
public class SomeCustomCondition implements Condition {
  @Override
  public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {

    // Return true if NOT "test" AND NOT "dev"
    return !context.getEnvironment().acceptsProfiles("test") 
                  && !context.getEnvironment().acceptsProfiles("dev");
  }
}

@Bean
@Conditional(SomeCustomCondition.class)
public MyBean specialBean(){/*...*/}
```
参考[stackoverflow相关问题](https://stackoverflow.com/questions/35429168/how-to-conditionally-declare-bean-when-multiple-profiles-are-not-active)