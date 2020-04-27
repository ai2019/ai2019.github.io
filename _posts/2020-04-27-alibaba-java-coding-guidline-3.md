---
layout: post
title:  "2020阿里巴巴Java开发手册（泰山版）学习-集合处理"
date:   2020-04-27 10:00:00
categories: Java
tags: Java 阿里巴巴 开发手册 集合
excerpt: 阅读2020阿里巴巴Java开发手册（泰山版），集合处理查漏补缺。
mathjax: true
---

* content
{:toc}

### 1. 官网地址

2020阿里巴巴Java开发手册（泰山版）下载地址：https://developer.aliyun.com/topic/java2020

《集合处理》学习地址：https://developer.aliyun.com/article/755086

### 2. 查漏补缺

##### (1) 强制】关于hashCode和equals的处理，遵循如下规则：

- 只要重写equals，就必须重写hashCode；
- 由于Set存储的是不重复的对象，依据hashCode和equals进行判断，所以Set存储的对象必须重写这两个方法；
- 如果自定义对象作为Map的键，那么必须重写hashCode和equals。

**说明**

> String正因为重写了hashCode和equals方法，所以我们可以非常愉快地使用String对象作为key来使用

##### (2)使用Map的方法keySet()/values()/entrySet()返回集合对象时，不可以对其进行添加元素操作，否则会抛出UnsupportedOperationException异常。

在使用迭代器时，不能对集合进行修改。

##### (3)【强制】判断所有集合内部的元素是否为空，使用isEmpty()方法，而不是size()==0的方式。

官网给的正例：

```java
Map<String, Object> map = new HashMap<>();
if(map.isEmpty()) {
    System.out.println("no element in this map.");
}
```

但是，如果集合是其他人给我们的，可能存在NPE，所以，本人更建议使用 apache的`commons-collections4`，maven依赖：

```java
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-collections4</artifactId>
    <version>4.4</version>
</dependency>
```

```java
Map<String, Object> map = new HashMap<>();
if (MapUtils.isEmpty(map)) {
    System.out.println("map is empty");
}

List<String> list = new ArrayList<>();
if(CollectionUtils.isEmpty(list)){
    System.out.println("list is empty");
}
```

##### （4）【强制】Collections类返回的对象，如：emptyList()/singletonList()等都是immutable list，不可对其进行添加或者删除元素的操作。

对于其他人返回的数据一定要谨慎处理。

##### （5）ArrayList的subList结果不可强转成ArrayList，否则会抛出ClassCastException异常：java.util.RandomAccessSubList cannot be cast to java.util.ArrayList ;

`subList`返回的是 `ArrayList` 的内部类 `SubList`，并不是 `ArrayList` ，而是 `ArrayList` 的一个视图，对于`SubList`子列表的所有操作最终会反映到原列表上。

小声说：我面试社招同学时，经常会问subList的底层实现，大部分同学都回答不正确。

##### （6）【强制】在subList场景中，高度注意对父集合元素的增加或删除，均会导致子列表的遍历、增加、删除产生ConcurrentModificationException 异常。

知其然，知其所以然。

##### （7）【强制】使用集合转数组的方法，必须使用集合的toArray(T[] array)，传入的是类型完全一致、长度为0的空数组。

正例

```java
List<String> list = new ArrayList<>(2);
list.add("guan");
list.add("bao");
array = list.toArray(new String[0]);
```

**说明**

使用toArray带参方法，数组空间大小的length：

- 等于0，动态创建与size相同的数组，性能最好。
- 大于0但小于size，重新创建大小等于size的数组，增加GC负担。
- 等于size，在高并发情况下，数组创建完成之后，size正在变大的情况下，负面影响与2相同。
- 大于size，空间浪费，且在size处插入null值，存在NPE隐患。


##### （8）【强制】在使用Collection接口任何实现类的addAll()方法时，都要对输入的集合参数进行NPE判断。

在ArrayList#addAll方法的第一行代码即Object[] a = c.toArray(); 其中c为输入集合参数，如果为null，则直接抛出异常。

##### （9）【强制】使用工具类Arrays.asList()把数组转换成集合时，不能使用其修改集合相关的方法，它的add/remove/clear方法会抛出UnsupportedOperationException异常。

这一点之前没有关注过，后续要关注。

asList的返回对象是一个Arrays内部类，并没有实现集合的修改方法。Arrays.asList体现的是适配器模式，只是转换接口，后台的数据仍是数组。

```java
String[] str = new String[] { "a", "b" };
List list = Arrays.asList(str);
```
第一种情况：list.add("c"); 运行时异常。

第二种情况：str[0]= "changed"; 那么list.get(0)也会随之修改，反之亦然。


##### （10）【强制】泛型通配符<? extends T>来接收返回的数据，此写法的泛型集合不能使用add方法，而<? super T>不能使用get方法，两者在接口调用赋值的场景中易出错。

扩展说一下PECS (Producer Extends Consumer Super)原则：

第一、频繁往外读取内容的，适合用<? extends T>。

第二、经常往里插入的，适合用<? super T>。


### 3. 学习感想

作为一名Java程序员，在CRUD的过程中，经常使用的数据结构就是集合，因此在处理集合时，要注意下面几点：

- 知其然，知其所以然；

- 防御式编程；

- 使用开源工具类（在业界广泛使用和验证），避免潜在的坑以及性能问题。