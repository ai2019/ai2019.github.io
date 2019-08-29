---
layout: post
title:  "获取ApplicationContext及Bean的工具类"
date:   2019-07-10 14:44:00
categories: spring-boot
tags: Spring ApplicationContext Bean
excerpt: 从Spring容器中获取ApplicationContext及Bean的工具类
mathjax: true
---

* content
{:toc}

实现ApplicationContextAware可以获取所在容器的ApplicationContext，因此工具类的实现如下：
```java
@Component
@Slf4j
public class SpringUtil implements ApplicationContextAware {

    private static ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        context = applicationContext;
    }

    /**
     * 获取上下文
     *
     * @return
     */
    private static ApplicationContext getContext() {
        return context;
    }

    /**
     * 获取bean
     *
     * @param name
     * @return
     */
    public static Object getBean(String name) {
        return getContext().getBean(name);
    }

    /**
     * 获取bean
     *
     * @param name
     * @param requiredType
     * @param <T>
     * @return
     * @throws BeansException
     */
    public static <T> T getBean(String name, Class<T> requiredType) throws BeansException {
        return getContext().getBean(name, requiredType);
    }

    /**
     * 获取有给定注解的bean
     *
     * @param annotationType
     * @return
     */
    public static Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType) {
        return getContext().getBeansWithAnnotation(annotationType);
    }

    /**
     * 获取顺序bean的列表
     *
     * @param annotationType
     * @param clazz
     * @param <T>
     * @return
     */
    public static <T> List<T> getBeanListWithAnnotationAndOrder(Class<? extends Annotation> annotationType, Class<T> clazz) {
        Map<String, Object> beanMap = getBeansWithAnnotation(annotationType);
        List<Object> beanList = Lists.newArrayList(beanMap.values());
        List<T> beans = (List<T>) beanList;
        beans.sort((v1, v2) -> {
            Order order1 = AopUtils.getTargetClass(v1).getAnnotation(Order.class);
            Order order2 = AopUtils.getTargetClass(v2).getAnnotation(Order.class);
            Integer o1 = order1 == null ? Integer.MAX_VALUE : order1.value();
            Integer o2 = order2 == null ? Integer.MAX_VALUE : order2.value();
            return o1.compareTo(o2);
        });
        return beans;
    }
}
```