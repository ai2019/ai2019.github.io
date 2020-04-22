---
layout: post
title:  "2020阿里巴巴Java开发手册（泰山版）学习-SQL规约"
date:   2020-04-22 11:00:00
categories: MySQL
tags: MySQL 阿里巴巴 开发手册 SQL规约
excerpt: 阅读2020阿里巴巴Java开发手册（泰山版），SQL规约查漏补缺。
mathjax: true
---

* content
{:toc}

### 1. 官网地址

2020阿里巴巴Java开发手册（泰山版）下载地址：https://developer.aliyun.com/topic/java2020

《SQL规约》学习地址：https://developer.aliyun.com/article/755082

### 2. 查漏补缺

##### (1) 【强制】不要使用count(列名)或count(常量)来替代count()，count()就是SQL92定义的标准统计行数的语法，跟数据库无关，跟NULL和非NULL无关。

在MySQL中，并不支持直接使用count()，例如

```sql
select count(1) from t;
```

MySQL的官方文档中指出还是要有参数的：https://dev.mysql.com/doc/refman/8.0/en/group-by-functions.html#function_count

**注意**

> count(*)会统计值为NULL的行
> count(列名)不会统计此列为NULL值的行

##### (2)【强制】count(distinct col) 计算该列除NULL之外的不重复数量

> count(distinct col) 计算该列除NULL之外的不重复数量；
> count(distinct col1, col2) 如果其中一列全为NULL，那么即使另一列有不同的值，也返回为0。

##### (3)【强制】当某一列的值全是NULL时，count(col)的返回结果为0，但sum(col)的返回结果为NULL，因此使用sum()时需注意NPE问题。

避免sum()时NPE的方式：

```sql
SELECT IFNULL(SUM(column), 0) FROM table;
```

##### （4）【强制】使用ISNULL()来判断是否为NULL值。

进行列值判空时，使用ISNULL()，不要使用其他的方式。

##### （5）【强制】对于数据库中表记录的查询和变更，只要涉及多个表，都需要加表名（或别名）进行限定。

多表join后作为条件进行查询记录、更新记录、删除记录时，如果出现没有限定表名（或别名）的列名在多个表中均有存在，那么会抛出异常。

强制使用表名限定，避免直接进行数据库变更，比如增加同名字段，造成的潜在风险。

##### （6）【强制】在代码中写分页查询逻辑时，若count为0应直接返回，避免执行后面的分页语句。

目前，看到的代码中，多数的同事都会遵循这个原则，但是还是偶尔发现有同事不注意这些细小的问题。

##### （7）【强制】不得使用外键与级联，一切外键概念必须在应用层解决。

这个问题主要针对刚毕业的同学，不要再使用外键。外键与级联更新适用于单机低并发，不适合分布式、高并发集群；级联更新是强阻塞，存在数据库更新风暴的风险；外键影响数据库的插入速度。

##### （8）【强制】DB数据订正（特别是删除或修改记录操作）时，要先select，避免出现误删除，确认无误才能提交执行。

根据多年的工作经验，直接在DB进行数据订正时，有两种规范的方式：

**数据备份**

- 创建跟要订正的表一样结构的备份表，命名方式bk_table_name；
- 将要进行数据订正的数据，全量备份到备份表；
- 执行数据订正；
- 观察业务影响，一旦出问题，直接用备份数据回滚。

**回滚SQL**

- 写数据订正的SQL的同时，写对应的回滚SQL。