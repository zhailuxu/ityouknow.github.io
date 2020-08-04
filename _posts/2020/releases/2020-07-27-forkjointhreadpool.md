---
layout: post
title: 关于Java中流式编程与ForkJoinPool的一点事
category: java
excerpt: 如何修改Java并行流的线程池。
tags: [java]
---

# 一、前言
最近在看项目代码时候，发现有一段奇怪的代码，细看完全多余，然后这其中却隐藏着一个不为人知的关于 ForkJoinPool 的秘密...

# 二、流式编程基础
 如下代码我们首先创建了一个list,然后从list上获取流对象，并使用foreach进行遍历：
```Java

public static void main(String[] args) throws IOException {
		// 1.创建list
		ArrayList<String> arrayList = new ArrayList<>();
		for (int i = 0; i < 100; ++i) {
			arrayList.add(i + "");
		}
		arrayList.stream().forEach(e -> System.out.println(Thread.currentThread().getName() + " " + e));
}
```


运行上面代码，输出为：
```Java
main 0
main 1
main 2
main 3
main 4
...
```


上面打印元素使用的main线程顺序进行的，大家都知道我们可以把流转换为并行流，代码如下：
```java
	public static void main(String[] args) throws IOException {

		// 1.创建list
		ArrayList<String> arrayList = new ArrayList<>();
		for (int i = 0; i < 100; ++i) {
			arrayList.add(i + "");
		}

		arrayList.parallelStream().forEach(e -> System.out.println(Thread.currentThread().getName() + " " + e));
}
```

运行上面代码输出如下：
```Java
ForkJoinPool.commonPool-worker-6 94
main 73
main 74
ForkJoinPool.commonPool-worker-6 95
ForkJoinPool.commonPool-worker-1 39
ForkJoinPool.commonPool-worker-3 69
ForkJoinPool.commonPool-worker-3 70  
...
```

上面代码则是使用ForkJoinPool的common线程池与main线程并行输出的，另外我们知道我们无法对流式的并行处理的线程池线程数量进行定制，其内部使用的是整个JVM内唯一的common线程池。

# 二、猜执行结果
上面我们介绍了流式编程的并行流，下面请看下面代码输出时候，打印的线程名称是什么：

```Java
 //代码示例1
	private static final ForkJoinPool pool = new ForkJoinPool(3);

	public static void main(String[] args) throws IOException {

		// 1.创建list
		ArrayList<String> arrayList = new ArrayList<>();
		for (int i = 0; i < 100; ++i) {
			arrayList.add(i + "");
		}

		pool.submit(() -> arrayList.parallelStream().forEach(e -> {
			System.out.println(Thread.currentThread().getName() + " " + e);
		})).join();
		System.out.println("Main is over");

}
```

阅读上面代码，我们可以看到main线程向forkjoin线程池里面添加了一个任务，然后阻塞等待任务的完成，然后打印输出Main is over。

那么这与不提交任务到线程池，而是直接执行，如下代码，看起来没啥区别：

```Java
 //代码示例2
	public static void main(String[] args) throws IOException {

		// 1.创建list
		ArrayList<String> arrayList = new ArrayList<>();
		for (int i = 0; i < 100; ++i) {
			arrayList.add(i + "");
		}

		arrayList.parallelStream().forEach(e -> System.out.println(Thread.currentThread().getName() + " " + e));
		System.out.println("Main is over");
}
```
如上代码，我们也是在main线程等待打印任务执行完毕后，在输出Main is over。

其实不然，如上代码示例1中，我们创建了一个名称为pool的线程池，然后向其中提交了一个任务。

运行上面代码后pool中会创建一个ForkJoinWorkerThread类型的线程，来执行我们提交的任务，也就是执行

```Java
arrayList.stream().forEach(e -> System.out.println(Thread.currentThread().getName() + " " + e));
```

运行上面代码，按理说是ForkJoinPool中的common线程池线程并行，执行打印输出。但是运行后你会发现打印任务的线程却是我们自己创建的pool中的线程，也就是我们使用自己创建的pool替代了并行流默认的ForkJoinPool中的common线程池。

究其原因是当我们调用并行流的forEach方法时候，会调用到ForkJoinTask的fork方法进行子任务切分：

```Java
    public final ForkJoinTask<V> fork() {
        Thread t;
        if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
            ((ForkJoinWorkerThread)t).workQueue.push(this);
        else
            ForkJoinPool.common.externalPush(this);
        return this;
    }
```
由于调用forEach的是我们自己创建的pool里面的线程（其是ForkJoinWorkerThread类型的），所以会把切分的任务添加到我们调用线程所在的队列里面，而不是添加到了common线程池里面。


# 三、总结
通过本文介绍的trick方法我们可以切换并行流执行的线程池，从而可以有效避免使用全局线程池造成相互影响。

