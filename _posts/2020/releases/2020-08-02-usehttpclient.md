---
layout: post
title: HttpClient连接池管理，你用对了？
category: java
no-post-nav: true
excerpt: 本文来深度探讨如何正确使用httpclient。
tags: [java]
---

# 一、前言
为何要用http连接池那？因为使用它我们可以得到以下好处：

因为使用它可以有效降低延迟和系统开销。如果不采用连接池，每当我们发起http请求时，都需要重新发起Tcp三次握手建立链接，请求结束时还需要四次挥手释放链接。而链接的建立和释放是有时间和系统开销的。另外每次发起请求时，需要分配一个端口号，请求完毕后在进行回收。

使用链接池则可以复用已经建立好的链接，一定程度的避免了建立和释放链接的时间开销。


# 二、连接池使用
```Java
public static void init() {
	//1.创建连接池管理器
	PoolingHttpClientConnectionManager connectionManager = new PoolingHttpClientConnectionManager(60000,//1.1
			TimeUnit.MILLISECONDS);
	connectionManager.setMaxTotal(1000);//1.2
	connectionManager.setDefaultMaxPerRoute(50);//1.3
	
	//2.创建httpclient对象
	httpClient = HttpClients.custom()
			.setConnectionManager(connectionManager)//2.1
			.disableAutomaticRetries()//2.2
			.build();
}
```


如上代码1，我们创建了一个连接池管理器，ClientConnectionPoolManager会维护每个路由维护和最大连接数限制。默认情况下，此实现将为每个给定路由创建不超过2个并发连接，并且总共不超过20个连接。对于许多现实应用程序，这些限制可能证明过于严格。但是，我们可以自由来调整连接限制。

另外构造函数中可以设置持久链接的存活时间TTL（timeToLive），其定义了持久连接的最大使用时间，超过其TTL值的链接不会再被复用。

如上代码1.1我们设置TTL为60s（tomcat服务器默认支持保持60s的链接，超过60s，会关闭客户端的链接）。代码1.2我们设置连接器最多同时支持1000个链接，代码1.3设置每个路由最多支持50个链接。注意这里路由是指IP+PORT或者指域名。如果使用域名来访问则每个域名有自己的链接池，如果使用IP+PORT访问，则每个IP+PORT有自己的链接池。

如上代码2我们基于连接池管理器创建了一个httpClient对象，下面我们就可以使用它发起http请求了。


```Java
CloseableHttpResponse response = null;
try {
	
	//3.创建请求对象
	final HttpGet httpGet = new HttpGet("http://127.0.0.1:8080/test");
	
	RequestConfig.Builder builder = RequestConfig.custom();
	builder.setSocketTimeout(3000)//3.1设置客户端等待服务端返回数据的超时时间
	       .setConnectTimeout(1000)//3.2设置客户端发起TCP连接请求的超时时间
	       .setConnectionRequestTimeout(3000);//3.3设置客户端从连接池获取链接的超时时间
	httpGet.setConfig(builder.build());

	//4.发起请求
	response = httpClient.execute(httpGet);

	//5.成功则解析结果
	if (200 == response.getStatusLine().getStatusCode()) {
		String strResult = EntityUtils.toString(response.getEntity());
		System.out.println(strResult);
	}

} catch (ClientProtocolException e) {
	e.printStackTrace();
} catch (IOException e) {
	e.printStackTrace();
} finally {
	//5.回收链接到连接池
	if (null != response) {
		try {
			EntityUtils.consume(response.getEntity());
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}
```

如上代码3创建了一个HttpGet对象，并创建RequestConfig对象来设置请求参数。

代码3.3设置客户端从连接池获取链接的超时时间，如果在该时间内没能从连接池获取到连接，则抛出ConnectionPoolTimeoutException异常。

代码3.2设置客户端发起TCP连接请求的超时时间，也就是创建TCP连接时候等待的时间，如果该时间内还没完成TCP三次握手，则抛出ConnectTimeoutException异常。

代码3.1设置客户端等待服务端返回数据的超时时间，也就是请求发出去后，如果等待该时间服务端还没返回结果，则抛出SocketTimeoutException异常。

代码4则发起http请求，代码5发现请求OK，则使用自带工具类EntityUtils.toString解析返回值（内部读取流结束后，会自动返还链接到连接池）

代码5则当请求结束后做一个兜底链接归还（如果返回状态值不是200，则需要在这里处理链接归还），注意这里不能调用response.close();因为其不是归还链接到连接池，而是关闭链接。


# 三、总结
本文简单介绍了如何使用链接池，使用连接池时需要注意合理设置最大链接数和每个路由（比如域名）对应的链接数，另外特别需要注意设置setConnectionRequestTimeout参数，其决定了从连接池拿链接的超时时间。

另外需要注意使用链接池时，请求结果回来后，要记得归还链接，如果链接得不到归还，则首先会把连接池打满，然后新来的请求从连接池拿不到链接会抛出ConnectionPoolTimeoutException异常。

另外归还链接不要调用response对象的close（）方法，因为其是关闭链接，而不是把链接返回到链接池。需要调用EntityUtils中方法或者自己从response.getEntity().getContent()获取流，读取完毕（读取完毕后会自动归还链接）或者不读取时主动调用流的close()来显示归还链接到连接池。


知识扩展：
自http1.1后，http默认支持keep-alive。对于Tomcat服务器默认保持客户端的链接60s,我们httpclient这边也可以设置链接存活时间，最终链接的存活时间是取两者中最小的。

对于过期链接的处理，当Tomcat主动关闭链接时，httpclient 4.4之前是每次在复用链接前进行检查链接是否可用，http4.4后，是自上次使用连接以来所经过的时间超过已设置的超时时（默认超时设置为2000ms），才检查连接。如果发现链接不可用，则从链接池剔除，在创建新的链接。

当客户端设置的TTL到期时（此时Tomcat容器没有主动关闭链接时），在每次发起请求时，会检查链接是否已经过期，如果过期（虽然链接本身是可以用的），则也主动关闭链接，然后从链接池剔除，在创建新的链接。


另外我们可以实现自己的ConnectionKeepAliveStrategy来给不同的域名设置不同的链接存活策略。

