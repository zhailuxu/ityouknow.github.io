---
layout: post
title: 一个简单流式迭代DB数据的后端支持组件
category: java
excerpt: 流式迭代DB数据组件，你造？
tags: [java]
--- 
# 一、前言
日常需求中，经常会有从DB检索符合条件的数据的需求，如果数据量比较大，不可能使用默认方式一次请求把全部数据加载到内存，因为这可能导致OOM。本文就介绍一个组件，可以让我们使用起来类型流式迭代方式，批次迭代部分数据到内存。
# 二、工具介绍
本组件github地址为：https://github.com/PhantomThief/cursor-iterator

要使用该工具首先需要引入maven配置：

```Java
        <dependency>
            <groupId>com.github.phantomthief</groupId>
            <artifactId>cursor-iterator</artifactId>
            <version>1.0.12</version>
        </dependency>
```

然后可以下面方式使用：
```Java
   // 0.A fake DAO for test
    public static List<PersonDo> getUsersAscById(Long startId, int limit, NamedParameterJdbcTemplate jdbcTemplate) {
        if (startId == null) {
            startId = 0L;
        }

        String sql = "select * from table where oid<= " + startId + " order by oid desc limit " + limit;
        System.out.println(sql);
        List<PersonDo> list = jdbcTemplate.query(sql, new BeanPropertyRowMapper<PersonDo>(PersonDo.class));
        return list;
    }

    public static void main(String[] arg) throws SQLException {
        //1.创建数据源
        HikariDataSource dataSource = getDataSource();
        NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(dataSource);

        //2.游标访问
        CursorIterator<Long, PersonDo> users = CursorIterator.newBuilder()
                .start(Long.MAX_VALUE) //2.1游标起始值
                .cursorExtractor(PersonDo::getOid)//2.2从DO中获取表主键id的方法
                .bufferSize(1024)//2.3每次从DB拉取多少条记录
                .build((cursor, limit) -> getUsersAscById(cursor, limit, jdbcTemplate));//2.4拉取数据函数

        //3.转换流为反应式流
        Flux.fromStream(users.stream())
                .buffer(256)//3.1缓冲
                .subscribe(personDos -> {//3.2订阅
                    System.out.println(ObjectMapperUtils.toJSON(personDos));
                }, throwable -> {//3.3异常处理
                    System.out.println(ObjectMapperUtils.toJSON(throwable));
                });

        //4,关闭数据源
        dataSource.close();
    }
```

- 如上代码1我们创建了一个数据源和jdbc模板，这个大家可以各自创建。
- 如上代码2使用CursorIterator创建了一个游标迭代器，其中2.1说明游标起始值为long型最大值，代码2.2设置数据库Do对象的主键key获取方法，代码2.3说明每次迭代从DB里面获取多少数据，代码2.4则设置每次从db迭代数据的DAO函数。

- 代码3则我们把游标迭代器转换为flux流对象，以便我们可以对db数据进行缓冲处理，这里我们聚合256条数据为一个元素，代码3.2我们订阅转换后的流，并打印。

如上代码，我们可以保证每次最多有1024条记录到内存中，然后把这1024条记录聚合为每个含有256个记录的流，流中元素是个list。

如上代码其实我们起到了反压的效果，当我们的subscribe订阅函数处理速度快时候，就会加快从db迭代数据出来的速度，当处理速度慢时候，就会反向控制从db迭代出数据的速度。


# 三、总结
本文我们介绍了一种简单流式迭代DB数据的后端支持组件，使用该组件我们可以很方便的替代我们自己手动写代码循环迭代数据的场景，另外我们使用Flux将jdk8流转换为了反应式流，从而可以让我们享受反应式编程带来的丰富的操作符。关于CursorIterator的内部原理，下期会进行讲解。
