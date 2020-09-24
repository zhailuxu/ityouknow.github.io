---
layout: post
title: MySQL can't specify target table for update in FROM clause
category: java
excerpt: 子查询？？
tags: [java]
--- 

在开发过程经常遇到更新mysql表的情景，例如根据id更新用户的状态。比如我们需要更新id小于100的用户的状态，假设你的sql如下:
```sql
update Users set valid = 0
where id in (
  select id from Users where id < 100
);
```

那么当你信心满满的去执行时候，却会发现执行结果如下：
```#1093 - You can't specify target table 'Users' for update in FROM clause```

为何会出现这样情况那？其实是因为MySQL中不允许更新在内部select语句中使用的表，如下：
```sql
update myTable
   set  myTable.A = 
(
select  B from myTable
join...
)
```
许多其他数据库引擎都支持此功能，但MySQL没有，因此我们需要解决该限制。

那么在mysql中我们如何解决这个问题那?其实可以使用子查询生成临时来解决，文初的sql可以修改为下面：
```sql
update Users set valid = 0
where id in (
  select id from  (
    select id from Users where id < 100
  ) as t
);
```

如上sql该查询中的别名t很重要，它告诉MySQL从该选择查询中创建一个临时表t，并把它用作更新语句的条件。






