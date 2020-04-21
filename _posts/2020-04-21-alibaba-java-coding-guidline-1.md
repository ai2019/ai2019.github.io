---
layout: post
title:  "2020阿里巴巴Java开发手册（泰山版）学习-日期规约"
date:   2020-04-21 14:00:00
categories: Java
tags: Java 阿里巴巴 开发手册 日期规约
excerpt: 阅读2020阿里巴巴Java开发手册（泰山版），查漏补缺。
mathjax: true
---

* content
{:toc}

### 1. 官网地址

2020阿里巴巴Java开发手册（泰山版）下载地址：https://developer.aliyun.com/topic/java2020

《日期时间规约》学习地址：https://developer.aliyun.com/article/754900

### 2. 查漏补缺

##### (1) 【强制】不要在程序中写死一年为365天，避免在闰年时出现日期转换错误或程序逻辑错误。

*获取今年的天数*
```java
    /**
     * 获取今天的天数
     *
     * @return
     */
    public static int daysOfThisYear() {
        return LocalDate.now().lengthOfYear();
    }
```

*获取指定某年的天数*
```java
    /**
     * 获取指定某年的天数
     *
     * @param year
     * @return
     */
    public static int daysOfYear(int year) {
        return LocalDate.of(year, 1, 1).lengthOfYear();
    }
```

##### (2)【推荐】避免闰年二月问题。闰年的二月份有29天，一周年后的那一天不可能是2月29日。

*获取指定某年某月的天数*
```java
    /**
    /**
     * 获取指定某年某月的天数
     *
     * @param year
     * @param month
     * @return
     */
    public static int daysOfMonth(int year, int month) {
        return LocalDate.of(year, month, 1).lengthOfMonth();
    }
```

##### (3)【推荐】使用枚举值来指代月份。如果使用数字，注意Date，Calendar等日期相关类的月份month取值在0-11之间。

推荐使用Calendar中的枚举，例如：
- 1月：Calendar.JANUARY
- 星期一：Calendar.MONDAY

个人认为：

- 如果是跨国的合作、国际型的项目，非常有必要统一使用Calendar的枚举值；
- 如果是国内的项目，在使用月份时，直接使用阿拉伯数字更符合国人的思维；在使用星期时，建议使用枚举值，便于在代码中的数据运算。

##### （4）其他一些日期工具

```java
    /**
     * 获取当前小时(24小时制)
     *
     * @return
     */
    public static int hourOfDay() {
        return Calendar.getInstance().get(Calendar.HOUR_OF_DAY);
    }

    /**
     * 获取当前星期几
     * 注意：周日是1，周一是2，周二是3，以此类推，周六是7
     *
     * @return
     */
    public static int dayOfWeek() {
        return Calendar.getInstance().get(Calendar.DAY_OF_WEEK);
    }
```
