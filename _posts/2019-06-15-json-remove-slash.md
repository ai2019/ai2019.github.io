---
layout: post
title:  "去除JSON字符串中的转义斜杠"
date:   2019-06-15 14:00:00
categories: Java
tags: java json
excerpt: 去除JSON字符串中因为转义而被自动添加的斜杠
mathjax: true
---

* content
{:toc}

有的时候，我们得到的JSON字符串中会出现用于转义的斜杠，比如：
```json
{"first firstBlog":"{\"title\":\"firstBlog title\",\"author\":\"firstBlog author\"}"}
```

如果想要去除用于转义双引号的斜杠，第一想法是通过`replace()`去除。

其实，apache-common-text已经提供处理的方法:
```java
StringEscapeUtils.unescapeJson()
```
查看源码，可以去除：
```java
final Map<CharSequence, CharSequence> unescapeJavaMap = new HashMap<>();
unescapeJavaMap.put("\\\\", "\\");
unescapeJavaMap.put("\\\"", "\"");
unescapeJavaMap.put("\\'", "'");
unescapeJavaMap.put("\\", "");
```

最后，附上maven依赖：
```java
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-text</artifactId>
    <version>1.7</version>
</dependency>
```