---
layout: post
title: Java中异常处理小细节
category: java
excerpt: 虽然Error类型的错误是不可恢复错误，但是有时候我们还是需要显示的捕获并打印日志，以便问题排查.
tags: [java]
---

# 一、前言
Java中异常分为两种：一种是基于Error的，一种是基于Exception的。其两者都是继承自Throwable；其中Error错误一般都是不可恢复的错误，比如系统崩溃、虚拟机错误，内存空间不足、类定义找不到、方法调用栈溢出等；而Exception错误则是我们经常使用来做业务异常拦截的；对于Error类型错误一般由于是不可恢复错误，所以没必要catch掉，但是凡事都有例外...

# 二、来龙去脉

如下代码，service（）方法用来模拟业务服务，代码比较简单，一般下我们是首先创建一个返回对象，然后在try块中执行业务，然后设置结果；执行异常后在catch使用Exception类型捕获异常，然后设置result返回值为false。最后返回结果，这个代码看起来很正常：

```Java
public class Test {

	// 业务服务器提供的服务
	public static Result service() {

		// 1.创建返回对象
		Result result = new Result();
		result.setSucess(true);

		// 2.执行业务
		try {
			// 2.1执行业务，设置返回值 .....
		    ...
			result.setData("ok");

		} catch (Exception e) {
			// 2.2比如业务异常则设置为false，并且返回异常信息
			result.setSucess(false);
			result.setCode("biz-error1");
		}
		// 3.返回结果
		return result;

	}

	public static void main(String[] args) throws InterruptedException, ClassNotFoundException {

		Executor executor = Executors.newFixedThreadPool(5);
		executor.execute(() -> {
			System.out.println(JSON.toJSONString(service()));
		});

		executor.shutdown();

	}
}
```

当service()服务在线程池里面执行时候，并且service()方法里面抛出NoClassDefFoundError异常后，我们会看不到System.out.println(JSON.toJSONString(service()));输出结果，也不知道异常NoClassDefFoundError跑到哪里去了，这非常不利于排查问题。

由于NoClassDefFoundError是继承自Error，而Error继承自Throwable，所以我们有必要在加一个catch块来捕获Throwable异常：

```Java
public static Result service() {

		// 1.创建返回对象
		Result result = new Result();
		result.setSucess(true);

		// 2.执行业务
		try {
			// 2.1执行业务，设置返回值 .....
			if (true) {
				throw new NoClassDefFoundError("calss def can not found");
			}

			result.setData("ok");

		} catch (Exception e) {
			// 2.2比如业务异常则设置为false，并且返回异常信息
			result.setSucess(false);
			result.setCode("biz-error1");
		} catch (Throwable e) {
			// 2.3比如不可恢复的异常，比如NoClassDefFoundError，则设置为false，并且返回异常信息
			result.setSucess(false);
			result.setCode("sys-error1");
			System.out.println(e.getLocalizedMessage());

		}

		// 3.返回结果
		return result;

	}
```


当然要想实现简单捕获线程中抛出的异常也可以实现UncaughtExceptionHandler来捕获。

# 三、总结
虽然Error类型的错误是不可恢复错误，但是有时候我们还是需要显示的捕获并打印日志，以便问题排查；另外比如NoClassDefFoundError类型错误，可以只是应用中部分服务不可用，但是其他模块服务是可用的，所以这时候我们还是要理会的。
