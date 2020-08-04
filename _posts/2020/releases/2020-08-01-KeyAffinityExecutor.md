---
layout: post
title: 亲缘性线程池，这是什么鬼？
category: java
excerpt: 亲缘性线程池在需要保证顺序消费，并且需要高吞吐量的情况下很用用，必须普通情况下顺序消费的保证是靠单线程来做的（比如rocketmq的顺序消息，消费端消费时）。
tags: [java]
---

# 一、前言
JDK中的线程池主要解决两个问题：
- 一方面当执行大量异步任务时候线程池能够提供较好的性能，在不使用线程池的时，每当需要执行异步任务时候是直接 new一线程运行，而线程的创建和销毁是需要开销的。而使用线程池时候，线程池里面的线程是可复用的，不会每次执行异步任务时候都重新创建和销毁线程。
- 另一方面线程池提供了一种资源限制和管理的手段，比如可以限制线程的个数，动态新增线程等，每个 ThreadPoolExecutor 也保留了一些基本的统计数据，比如当前线程池完成的任务数目等。

JDK中的线程池固然好，但是其不具有亲缘性，也就是当我们顺序向其中投递多个任务后，不能保证具有相同属性的任务顺序执行，本文我们就来看一个可以实现亲缘性的线程池。

# 二、测试案例
首先我们在做个测试，看看JDK中线程池是否具有亲缘性，我们创建一个Person类，其中id作为唯一标识，data为需要处理的数据，如下代码，我们创建一些Person对象，放到list，然后把任务顺序投递到JDK线程池：

```Java
    //0.普通线程池
    static ExecutorService executorService = Executors.newFixedThreadPool(8);
    public static void executeByOldPool(List<Person> personList) {
        personList.stream().forEach(p -> executorService.execute(() -> {
            System.out.println(JSON.toJSONString(p));
        }));
    }
 public static void main(String[] args) {

        //1.创建列表
        List<Person> personList = new ArrayList<>();
        personList.add(Person.builder().id(1).data("1s").build());
        personList.add(Person.builder().id(2).data("2s").build());
        personList.add(Person.builder().id(1).data("11s").build());
        personList.add(Person.builder().id(3).data("3s").build());
        personList.add(Person.builder().id(1).data("111s").build());
        personList.add(Person.builder().id(2).data("22s").build());
        personList.add(Person.builder().id(3).data("33s").build());
        personList.add(Person.builder().id(1).data("1111s").build());

        //2.使用普通线程池执行
        executeByOldPool(personList);
}
```

执行上面代码，如果线程池是亲缘的，则比如对应id=1的Person的输出应该是和投递到线程池时一致，也就是下面顺序：
```Java
{"data":"1s","id":1}
{"data":"11s","id":1}
{"data":"111s","id":1}
{"data":"1111s","id":1}
```

但是当我们执行上面代码，一个可能的输出为：
```Java
{"data":"3s","id":3}
{"data":"2s","id":2}
{"data":"33s","id":3}
{"data":"22s","id":2}
{"data":"1111s","id":1}
{"data":"1s","id":1}
{"data":"11s","id":1}
{"data":"111s","id":1}
```
可知其并没实现亲缘性，比如id=1的person的data并没有按照投递线程池顺序输出。

究其原因是因为JDK中线程池是不保证先投递到线程池的任务先执行完毕。

# 三、亲缘性线程池实现
如果想实现亲缘线程池，则这里有大佬w.vela的一个开源实现 [https://github.com/PhantomThief/simple-pool](https://github.com/PhantomThief/simple-pool)

首先我们需要引入其依赖：
```Java
		<dependency>
			<groupId>com.github.phantomthief</groupId>
			<artifactId>simple-pool</artifactId>
			<version>0.1.17</version>
		</dependency>
```

然后上面executeByOldPool方法修改为下面：
```Java
    static KeyAffinityExecutor executor = KeyAffinityExecutor.newSerializingExecutor(8,200, "MY-POOL");
    public static void executeByAffinitydPool(List<Person> personList) {
        personList.stream().forEach(p -> executor.executeEx(p.getId(), () -> {
            System.out.println(JSON.toJSONString(p));
        }));
    }
```
如上代码投递任务到线程池时，我们使用person的id作为key，这可以保证相同的id顺序投递到线程池的任务可以顺序执行，修改后，运行，一个可能的输出为：

```Java
{"data":"3s","id":3}
{"data":"1s","id":1}
{"data":"2s","id":2}
{"data":"33s","id":3}
{"data":"11s","id":1}
{"data":"22s","id":2}
{"data":"111s","id":1}
{"data":"1111s","id":1}
```

如上输出可知对应相同id的Person，其输出与投递到线程池顺序一致。

那么亲缘性线程池如何实现保证顺序内，大家可以看下其代码，其实很简单，就是把相同key的任务按照投递线程池的顺序，放到同一个内存队列(这里我们设置为200大小)，每个内存队列有一个线程来消费。那么消费线程有几个那？其实是按照创建线程池时newSerializingExecutor的第一个参数来决定。

# 四、总结
亲缘性线程池在需要保证顺序消费，并且需要高吞吐量的情况下很用用，必须普通情况下顺序消费的保证是靠单线程来做的（比如rocketmq的顺序消息，消费端消费时）。


