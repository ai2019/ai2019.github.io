---
layout: post
title:  "设计模式(1)-单例模式"
date:   2019-06-01 14:44:00
categories: 设计模式
tags: 设计模式 单例模式 饿汉式 懒汉式 双检锁式 静态内部类式
excerpt: 4种方式实现单例模式
mathjax: true
---

* content
{:toc}

### 1. 定义

保证一个类仅有一个实例，并提供一个访问它的全局访问点。

### 2. 使用场景

需要控制一个类的实例只能有一个时

### 3. 实现方式

#### 实现思路

如果类的构造方法是public的，那么就可以再任何地方创建类的实例。因此，将类的构造方法设置为private，外部就不能创建类的实例了。
外部不能创建类的实例，类的内部必须提供一个获取唯一的实例的方法。

#### (1)饿汉式

```java
public class Singleton {

    /**
     * 静态变量在类加载时，分配内存并初始化，只被创建一次
     */
    private static Singleton instance = new Singleton();

    /**
     * 私有化构造方法
     */
    private Singleton() {
    }

    /**
     * 获取实例的方法
     */
    public static Singleton getInstance() {
        return instance;
    }
}
```

#### (2)懒汉式

```java
public class Singleton {

    /**
     * 并未实例化
     */
    private static Singleton instance = null;

    private Singleton() {
    }

    /**
     * 获取实例的方法
     * 由于里面的代码是：检查-操作的方式，因此不是线程安全的
     * 加synchronized保证线程安全
     */
    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

#### (3)双检锁式

```java
public class Singleton {

    /**
     * volatile修饰，保证变量的内存可见性，也就是保证主存中的数据是最新的
     */
    private volatile static Singleton instance = null;

    private Singleton() {
    }

    public static Singleton getInstance() {
        // 第一次检查实例是否存在
        if (instance == null) {
            // 同步块，保证创建实例是线程安全的
            synchronized (Singleton.class) {
                // 再次检查实例是否存在，如果不存在才创建
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

#### (4)类级内部类式

该方式非常巧妙，利用类级内部类类（static修饰的内部类）和多线程缺省同步的知识，
可见能够提出该实现方式的同学是基础非常扎实，并勤于思考，具体实现：

```java
public class Singleton {

    private Singleton() {
    }

    /**
     * 静态内部类，只有被调用时才会装载，从而实现延迟加载
     */
    private static class SingletonHolder {
        
        /**
         * 静态属性，JVM保证线程安全
         */
        private static Singleton instance = new Singleton();
    }

    public static Singleton getInstance() {
        return SingletonHolder.instance;
    }
}
```

### 4. 模式本质

控制实例数目。

### 5. 优缺点

#### 优点

只有一个实例，保证数据只有一份，节约重复创建的时间和空间。

#### 缺点

- 饿汉式，在类加载时就被实例化，增加应用启动的时间，且如果启动后不被使用，浪费空间；
- 懒汉式，每次获取实例时，都是同步方法，性能不好；
- 双检锁式，比懒汉式性能好一些，但仍有同步代码块
