---
layout: post
title:  "Java8新特性之StringJoiner"
date:   2019-03-23 22:00:00
categories: Java
tags: Java String StringJoiner
excerpt: JDK8字符串处理的新姿势
---

* content
{:toc}

在Java 8之前，拼接字符串的通常姿势如下：
```java
    String DELIMITER = ",";
    List<String> strList = Arrays.asList("1", "2", "3", "4", "5");
    StringBuilder sb = new StringBuilder();
    for (String str : strList) {
        if (sb.length() > 0) {
            sb.append(DELIMITER);
        }
        sb.append(str);
    }
    System.out.println(sb);
```
在Java 8之后，通过String.join()方法可以简便地实现字符串拼接：
```java
    String DELIMITER = ",";
    List<String> strList = Arrays.asList("1", "2", "3", "4", "5");
    System.out.println(String.join(DELIMITER, strList));
```
JDK时如何实现的呢，查看源码：
```java
    public static String join(CharSequence delimiter,
            Iterable<? extends CharSequence> elements) {
        Objects.requireNonNull(delimiter);
        Objects.requireNonNull(elements);
        StringJoiner joiner = new StringJoiner(delimiter);
        for (CharSequence cs: elements) {
            joiner.add(cs);
        }
        return joiner.toString();
    }
```
可以看出，代码结构类似最开始的代码，使用了StringJoiner，而不是StringBuilder，同时没有每次追加分隔符的语句。
查看源码，StringJoiner原来是Java 8为了方便操作字符串而新增的类，内部的实现仍然是通过StringBuilder实现的：
```java
    private StringBuilder value;
```
以后，可以使用StringJoiner拼接字符串了：

```java
    private static final String DELIMITER = ",";
    private static final String PREFIX = "[";
    private static final String SUFFIX = "]";
    
    StringJoiner sj = new StringJoiner(DELIMITER, PREFIX, SUFFIX);
    sj.add("10").add("20").add("30");
    System.out.println(sj);
```
可以看到，定义好分隔符，以及可选的前缀、后缀后，只需要通过add方法就可以拼接字符串，非常简便。
JDK是如何实现的呢，查看源码：
```java
    public StringJoiner add(CharSequence newElement) {
        prepareBuilder().append(newElement);
        return this;
    }
```
查看prepareBuilder():
```java
    private StringBuilder prepareBuilder() {
        if (value != null) {
            value.append(delimiter);
        } else {
            value = new StringBuilder().append(prefix);
        }
        return value;
    }
```
原来，该方法就是封装了以前拼接字符串时，添加分隔符、前缀的逻辑。
那么后缀时如何添加的呢？原来重写了toString()方法，在调用toString时才append()后缀。

Java 8拼接字符串的新姿势解锁：
```java
    public static void main(String[] args) {
        List<String> strList = Arrays.asList("1", "2", "3", "4", "5");
        System.out.println(String.join(DELIMITER, strList));

        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
        System.out.println(numbers.stream().map(Object::toString).collect(Collectors.joining(DELIMITER)));

        StringJoiner sj = new StringJoiner(DELIMITER, PREFIX, SUFFIX);
        sj.add("10").add("20").add("30");
        System.out.println(sj);
    }
```