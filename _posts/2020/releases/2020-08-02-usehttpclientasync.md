---
layout: post
title: HttpClient的异步调用，你造？
category: java
no-post-nav: false
excerpt: 本文来深度探讨如何正确使用httpclient的异步调用。
tags: [java]
---

# 一、前言
HttpClient提供了两种I/O模型：经典的java阻塞I/O模型和基于Java NIO的异步非阻塞事件驱动I/O模型。

Java中的阻塞I/O是一种高效、便捷的I/O模型，非常适合并发连接数量相对适中的高性能应用程序。只要并发连接的数量在1000个以下并且连接大多忙于传输数据，阻塞I/O模型就可以提供最佳的数据吞吐量性能。
然而，对于连接大部分时间保持空闲的应用程序，上下文切换的开销可能会变得很大，这时非阻塞I/O模型可能会提供更好的替代方案。

异步I/O模型可能更适合于比较看重资源高效利用、系统可伸缩性、以及可以同时支持更多HTTP连接的场景。

<!--more-->

# 二、HttpClient中的Future
在HttpClient官网Tutorial的高级话题中，我们可以发现其提供了用于异步执行的FutureRequestExecutionService服务类。

使用FutureRequestExecutionService，允许我们发起http调用后，调用函数马上返回（调用线程不会阻塞等到相应结果返回）一个Future对象，然后调用线程可以在需要响应结果的地方调用Future对象的get方法来阻塞等待结果。

使用FutureRequestExecutionService的优点是，我们可以使用多个线程并发调度请求、设置任务超时，或者在不再需要响应时取消它们。

FutureRequestExecutionService其实是用一个HttpRequestFutureTask包装请求，该HttpRequestFutureTask扩展了JDK中的FutureTask。
这个类允许我们取消任务、跟踪各种执行指标，如请求持续时间等。

下面我们看一个例子：
```Java
	// 1.创建线程池
	private static ExecutorService executorService = Executors.newFixedThreadPool(5);

	// 2.创建http回调函数
	private static final class OkidokiHandler implements ResponseHandler<String> {
		public String handleResponse(final HttpResponse response) throws ClientProtocolException, IOException {
			// 2.1处理响应结果
			return EntityUtils.toString(response.getEntity());
		}
	}

	public static void main(String[] args) throws IOException, InterruptedException, ExecutionException {
		// 3.创建httpclient对象
		CloseableHttpClient httpclient = HttpClients.createDefault();

		// 4.创建FutureRequestExecutionService实例
		FutureRequestExecutionService futureRequestExecutionService = new FutureRequestExecutionService(httpclient,
				executorService);

		// 5.发起调用
		try {
			// 5.1请求参数
			HttpGet httpget1 = new HttpGet("http://127.0.0.1:8080/test1");
			HttpGet httpget2 = new HttpGet("http://127.0.0.1:8080/test2");
			// 5.2发起请求，不阻塞，马上返回
			HttpRequestFutureTask<String> task1 = futureRequestExecutionService.execute(httpget1,
					HttpClientContext.create(), new OkidokiHandler());

			HttpRequestFutureTask<String> task2 = futureRequestExecutionService.execute(httpget2,
					HttpClientContext.create(), new OkidokiHandler());

			// 5.3 do somthing

			// 5.4阻塞获取结果
			String str1 = task1.get();
			String str2 = task2.get();
			System.out.println("response:" + str1 + " " + str2);
		} finally {
			httpclient.close();
		}
	}
```

如上代码1创建了一个线程池用来作为http异步执行的后台线程，代码2创建了一个http响应结果处理器，用来异步处理http的响应结果。

代码3创建了一个HttpClient对象，代码4创建一个FutureRequestExecutionService，参数1为创建的httpclient对象，参数2为创建的线程池。

代码5则创建2个Get请求参数，然后执行代码5.2发起两个http请求,该调用会马上返回自己对于的HttpRequestFutureTask对象，调用线程也会马上返回，
然后调用线程就可以在5.3做其他的事情，最后在需要获取http响应结果的地方，比如代码5.4调用两个future的get()方法来获取结果。

如上基于Future方式，我们可以并发的发起两个http请求，而之前阻塞方式，则是顺序执行的。

但是基于上面Future方式，我们调用线程还是会在代码5.4阻塞等待响应结果，这并不是我们想要的，我们想要的是事件通知，http确实也提供了注册CallBack的方式：

首先我们需要实现FutureCallback接口，用来接收通知：
```Java
	private static final class MyCallback implements FutureCallback<String> {

		public void failed(final Exception ex) {
			System.out.println(ex.getLocalizedMessage());
		}

		public void completed(final String result) {
			System.out.println(result);
		}

		public void cancelled() {
			System.out.println("cancelled");
		}
	}
```

然后我们只需要修改代码5.2，使用三个参数的execute方法发起调用:
```java
		// 5.发起调用
		try {
			// 5.1请求参数
			HttpGet httpget1 = new HttpGet("http://127.0.0.1:8080/test1");
			HttpGet httpget2 = new HttpGet("http://127.0.0.1:8080/test2");
			// 5.2发起请求，不阻塞，马上返回
			HttpRequestFutureTask<String> task1 = futureRequestExecutionService.execute(httpget1,
					HttpClientContext.create(), new OkidokiHandler(), new MyCallback());

			HttpRequestFutureTask<String> task2 = futureRequestExecutionService.execute(httpget2,
					HttpClientContext.create(), new OkidokiHandler(), new MyCallback());
              //main线程休眠10s,避免请求结束前，关闭了链接
			Thread.sleep(10000);
...
```
如上代码，使用CallBack后，调用线程就得到了彻底解放，就不必再阻塞获取结果了，当http返回结果后，会自动调用我们注册的CallBack。

# 三、HttpAsyncClient-真正的异步
上面HttpClient提供的CallBack的方式，虽然解放了调用线程，但是并不是真正意义上的异步调用，因为其异步调用的支持是基于我们创建的executorService线程。
即：虽然发起http调用后，调用线程马上返回了，但是其内部还是使用executorService中的一个线程阻塞等待响应结果。

HttpAsyncClient则使用Java NIO的异步非阻塞事件驱动I/O模型，实现了真正意义的异步调用，使用HttpAsyncClient我们需要引入其专门的包：
```Java
<dependency>
  <groupId>org.apache.httpcomponents</groupId>
  <artifactId>httpasyncclient</artifactId>
  <version>4.1.4</version>
</dependency>
```

然后改造后代码如下：
```Java
	// 1.创建CallBack
	private static final class MyCallback implements FutureCallback<HttpResponse> {

		public void failed(final Exception ex) {
			System.out.println(ex.getLocalizedMessage());
		}

		public void completed(final HttpResponse response) {
			try {
				System.out.println(EntityUtils.toString(response.getEntity()));
			} catch (ParseException e) {
				e.printStackTrace();
			} catch (IOException e) {
				e.printStackTrace();
			}
		}

		public void cancelled() {
			System.out.println("cancelled");
		}
	}

	public static void main(String[] args) throws IOException, InterruptedException, ExecutionException {
		// 2.创建异步httpclient对象
		CloseableHttpAsyncClient httpclient = HttpAsyncClients.custom().build();

		// 3.发起调用
		try {

			// 3.0启动
			httpclient.start();
			// 3.1请求参数
			HttpGet httpget1 = new HttpGet("http://127.0.0.1:8080/test1");
			HttpGet httpget2 = new HttpGet("http://127.0.0.1:8080/test2");
			// 3.2发起请求，不阻塞，马上返回
			httpclient.execute(httpget1, new MyCallback());
			httpclient.execute(httpget2, new MyCallback());

			// 3.3休眠10s,避免请求执行完成前，关闭了链接
			Thread.sleep(10000);
		} finally {
			httpclient.close();
		}
	}
```

如上代码1，创建异步回调实现，用于处理Http响应结果。代码2创建了异步HttpClient,代码3.0启动client,代码3.2发起请求。

基于Java NIO的异步，当发起请求后，调用方不会使用任何线程同步等待http服务端的响应结果（少量的NIO线程不算哦，因为其个数固定，并且不随并发请求数量变化），
而是会使用少量内存来记录请求信息，以便服务端响应结果回来后，可以找到对应的回调函数进行执行。




# 四、总结
本文概要讲解了Http的异步调用，关于更多Java中异步调用与异步执行的知识，可以参考[《Java异步编程实战》](https://mp.weixin.qq.com/s/yF9cX4t5lUUPNm2-vKzp2Q
)

