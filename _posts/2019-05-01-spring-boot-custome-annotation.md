---
layout: post
title:  "spring boot三步实现自定义注解"
date:   2019-05-21 14:00:00
categories: spring-boot
tags: spring-boot,aop,annotation
excerpt: AOP三步实现自定义注解
mathjax: true
---

* content
{:toc}

### 1. 引入依赖
通过AOP的方式实现自定义注解，因此需要引入AOP的依赖：

```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### 2. 定义注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface CustomAnnotation {

    /**
     * 定义需要的属性
     */
    String name() default "";
}
```


### 3. 注解处理器

```java
@Aspect
@Component
@Slf4j
public class CustomAnnotationSupport {

    @Pointcut(value = "@annotation(CustomAnnotation)")
    private void pointcut() {
    }

    @Before(value = "pointcut()&&@annotation(customAnnotation)")
    private void before(JoinPoint joinPoint, CustomAnnotation customAnnotation) {
        log.error("param:{}",joinPoint.getArgs());
        log.error("annotation name:{}", customAnnotation.name());
    }
}
```

本处使用了前置通知@Before，AOP的通知共有5种，根据实际需要选择使用。



End