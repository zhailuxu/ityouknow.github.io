---
layout: post
title: Java异步编程实战：基于JDK中的Future实现异步编程
category: java
excerpt: 如何JDK中的Future如何实现异步编程？
tags: [java]
--- 


# 一、前言
本节主要讲解如何使用JDK中的Future实现异步编程，这包含如何使用FutureTask实现异步编程以及其内部实现原理；

# 二、 JDK 中的Future

在Java并发包（JUC包）中Future代表着异步计算结果，Future中提供了一些列方法用来检查计算结果是否已经完成，还提供了同步等待任务执行完成的方法，以及获取计算结果的方法等。当计算结果完成时只能通过提供的get系列方法来获取结果，如果使用了不带超时时间的get方法则在计算结果完成前，调用线程会被一直阻塞。另外计算任务是可以使用cancle方法来取消的，但是一旦一个任务计算完成，则不能再被取消了。


首先我们看下Future接口的类图结构如图3-1-1：

![image.png](/assets/images/2020/future1.png)

图3-1-1Future类图

如上图3-1-1Future类总共就有5个接口方法，下面我们来一一讲解：

- V get() throws InterruptedException, ExecutionException :等待异步计算任务完成，并返回结果；如果当前任务计算还没完成则会阻塞调用线程直到任务完成；如果在等待结果的过程中有其他线程取消了该任务，则调用线程抛出CancellationException异常；如果在等待结果的过程中有其他线程中断了该线程，则调用线程抛出InterruptedException异常；如果任务计算过程中抛出了异常，则调用线程会抛出ExecutionException异常。

-  V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException:相比get()方法多了超时时间，不同在于线程调用了该方法后在任务结果没有计算出来前调用线程不会一直被阻塞，而是会在等待timeout个unit单位的时间后抛出TimeoutException异常后返回。添加超时时间这避免了调用线程死等的情况，让调用线程可以及时释放。

- boolean isDone():如果计算任务已经完成则返回true，否则返回false，需要注意的是任务完成是指任务正常完成了或者由于抛出异常而完成了或者任务被取消了。


- boolean cancel(boolean mayInterruptIfRunning) :尝试取消任务的执行;如果当前任务已经完成或者任务已经被取消了，则尝试取消任务会失败；如果任务还没被执行时候，调用了取消任务，则任务将永远不会被执行；如果任务已经开始运行了，这时候取消任务，则参数mayInterruptIfRunning将决定是否要将正在执行任务的线程中断，如果为true则标识要中断，否则标识不中断；当调用取消任务后，在调用isDone()方法，后者会返回true，随后调用isCancelled()方法也会一直返回true；该方法会返回false，如果任务不能被取消，比如任务已经完成了，任务已经被取消了。

-  boolean isCancelled():如果任务在被执行完毕前被取消了，则该方法返回true，否则返回false。

# 三 JDK中的FutureTask
## 3.1 FutureTask 概述
FutureTask代表了一个可被取消的异步计算任务，该类实现了Future接口，比如提供了启动和取消任务、查询任务是否完成、获取计算结果的接口。

FutureTask任务的结果只有当任务完成后才能获取，并且只能通过get系列方法获取，当结果还没出来时候，线程调用get系列方法会被阻塞；另外一旦任务被执行完成，任务不能被重启，除非运行时候使用了runAndReset方法；FutureTask中的任务可以是Callable类型，也可以是Runnable类型（因为FutureTask实现了Runnable接口），FutureTask类型的任务可以被提交到线程池执行。

我们修改上节的例子如下：


```
public class AsyncFutureExample {

	public static String doSomethingA() {

		try {
			Thread.sleep(2000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("--- doSomethingA---");

		return "TaskAResult";
	}

	public static String doSomethingB() {
		try {
			Thread.sleep(2000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("--- doSomethingB---");
		return "TaskBResult";

	}

	public static void main(String[] args) throws InterruptedException, ExecutionException {

		long start = System.currentTimeMillis();

		// 1.创建future任务
		FutureTask<String> futureTask = new FutureTask<String>(() -> {
			String result = null;
			try {
				result = doSomethingA();

			} catch (Exception e) {
				e.printStackTrace();
			}
			return result;
		});

		// 2.开启异步单元执行任务A
		Thread thread = new Thread(futureTask, "threadA");
		thread.start();

		// 3.执行任务B
		String taskBResult = doSomethingB();

		// 4.同步等待线程A运行结束
		String taskAResult = futureTask.get();
		
		//5.打印两个任务执行结果
		System.out.println(taskAResult + " " + taskBResult); 
		System.out.println(System.currentTimeMillis() - start);

	}
}
```

- 如上代码doSomethingA和doSomethingB方法都是有返回值的任务，main函数内代码1创建了一个异步任务futureTask，其内部执行任务doSomethingA。
- 代码2则创建了一个线程，并且以futureTask为执行任务，并且启动；代码3使用main线程执行任务doSomethingB，这时候任务doSomethingB和doSomethingA是并发运行的，等main函数运行doSomethingB完毕后，执行代码4同步等待doSomethingA任务完成，然后代码5打印两个任务的执行结果。
- 如上可知使用FutureTask可以获取到异步任务的结果。

当然我们也可以把FutureTask提交到线程池来执行，使用线程池运行方式代码如下：


```
	// 0自定义线程池
	private final static int AVALIABLE_PROCESSORS = Runtime.getRuntime().availableProcessors();
	private final static ThreadPoolExecutor POOL_EXECUTOR = new ThreadPoolExecutor(AVALIABLE_PROCESSORS,
			AVALIABLE_PROCESSORS * 2, 1, TimeUnit.MINUTES, new LinkedBlockingQueue<>(5),
			new ThreadPoolExecutor.CallerRunsPolicy());

	public static void main(String[] args) throws InterruptedException, ExecutionException {

		long start = System.currentTimeMillis();

		// 1.创建future任务
		FutureTask<String> futureTask = new FutureTask<String>(() -> {
			String result = null;
			try {
				result = doSomethingA();

			} catch (Exception e) {
				e.printStackTrace();
			}
			return result;
		});

		// 2.开启异步单元执行任务A
		POOL_EXECUTOR.execute(futureTask);

		// 3.执行任务B
		String taskBResult = doSomethingB();

		// 4.同步等待线程A运行结束
		String taskAResult = futureTask.get();
		// 5.打印两个任务执行结果
		System.out.println(taskAResult + " " + taskBResult);
		System.out.println(System.currentTimeMillis() - start);
	}
```

如上可知代码0创建了一个线程池，代码2添加异步任务到线程池，这里我们是调用了线程池的execute方法把futureTask提交到线程池的，其实下面代码与上面是等价的：

```
	public static void main(String[] args) throws InterruptedException, ExecutionException {

		long start = System.currentTimeMillis();

		// 1.开启异步单元执行任务A
		Future<String> futureTask = POOL_EXECUTOR.submit(() -> {
			String result = null;
			try {
				result = doSomethingA();

			} catch (Exception e) {
				e.printStackTrace();
			}
			return result;
		});

		// 2.执行任务B
		String taskBResult = doSomethingB();

		// 3.同步等待线程A运行结束
		String taskAResult = futureTask.get();
		// 4.打印两个任务执行结果
		System.out.println(taskAResult + " " + taskBResult);
		System.out.println(System.currentTimeMillis() - start);
	}
```

如上代码1我们调用了线程池的submit方法提交了一个任务到线程池，然后返回了一个FutureTask对象。


## 3.2 FutureTask的类图结构：
由于FutureTask在异步编程领域还是比较重要的，所以我们有必要探究下其原理，以便加深对异步的理解，首先我们来看下其类图结构如图3-2-2-1：
![image.png](/assets/images/2020/future2.png)
图3-2-2-1 FutureTask的类图


- 如上时序图3-2-2-1FutureTask实现了Future接口的所有方法，并且实现了Runnable接口，所以其是可执行任务，可以投递到线程池或者线程来执行。

- FutureTask中变量state是一个使用volatile关键字修饰（用来解决多线程下内存不可见问题，具体可以参考《Java并发编程之美》一书）的int类型，用来记录任务状态，任务状态枚举值如下：

```Java
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
```

一开始任务状态会被初始化为NEW；当通过set、 setException、 cancel函数设置任务结果时候，任务会转换为终止状态；在任务完成过程中，设置任务状态会变为COMPLETING（当结果被使用set方法设置时候），也可能会经过INTERRUPTING状态（当使用cancel(true)方法取消任务并中断任务时候）；当任务被中断后，任务状态为INTERRUPTED；当任务被取消后，任务状态为CANCELLED；当任务正常终止时候，任务状态为NORMAL；当任务执行异常后，任务会变为EXCEPTIONAL状态。

另外在任务运行过程中，任务可能的状态转换路径如下:

- NEW -> COMPLETING -> NORMAL ：正常终止流程转换
- NEW -> COMPLETING -> EXCEPTIONAL：执行过程中发生异常流程转换
- NEW -> CANCELLED：任务还没开始就被取消
- NEW -> INTERRUPTING -> INTERRUPTED：任务被中断

从上述转换可知，任务最终只有四种终态：NORMAL、EXCEPTIONAL、CANCELLED、INTERRUPTED，另外可知任务的状态值是从上到下递增的。

- 类图中callable是有返回值的可执行任务，创建FutureTask对象时候，可以通过构造函数传递该任务。

- 类图中outcome是任务运行的结果，可以通过get系列方法来获取该结果,另外outcome这里没有被修饰为volatile，是因为变量state已经被使用volatile修饰了，这里是借用volatile的内存语义来保证写入outcome时候会把值刷新到主内存，读取时候会从主内存读取，从而避免多线程下内存不可见问题（可以参考《Java并发编程之美》一书）。

- 类图中runner变量，记录了运行该任务的线程，这个是在FutureTask的run方法内使用CAS函数设置的。

- 类图中waiters变量是一个WaitNode节点，使用Treiber stack实现的无锁栈，栈顶元素就是使用waiters代表，栈用来记录所有等待任务结果的线程节点，其定义为：

```Java
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }
```

可知其是一个简单的链表，用来记录所有等待结果而被阻塞的线程。


最后我们看下其构造函数：

```Java
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       
    }
```

如上代码可知构造函数内保存了传递的callable任务到callable变量，并且设置任务状态为NEW，这里由于state为volatile修饰，所以写入state的值可以保证callable的写入也会被刷入主内存，这避免了多线程下内存不可见性。

另外还有一个构造函数：

```Java
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;     
    }
```

该函数传入一个Runnable类型的任务，由于该任务是不具有返回值的，所以这里使用Executors.callable方法进行适配，适配为Callable类型的任务.

Executors.callable(runnable, result);把Runnable类型任务转换为了callable：

```Java
    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }

    static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }
```

如上可知使用了适配器模式来做转换。



另外FutureTask中使用了UNSAFE机制来操作内存变量：

```Java
 private static final sun.misc.Unsafe UNSAFE;

    private static final long stateOffset;//state变量的偏移地址
    private static final long runnerOffset;//runner变量的偏移地址
    private static final long waitersOffset;//waiters变量的偏移地址
    static {
        try {
            //获取UNSAFE的实例
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = FutureTask.class;
            //获取变量state的偏移地址
            stateOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("state"));
            //获取变量runner的偏移地址
            runnerOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("runner"));
            //获取变量waiters变量的偏移地址
            waitersOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("waiters"));
        } catch (Exception e) {
            throw new Error(e);
        }
}
```

如上代码分别获取了FutureTask中几个变量在FutureTask对象内的内存地址偏移量，以便实现中使用UNSAFE的CAS操作来操作这些变量。


## 3.3  FutureTask的run() 方法
该方法是任务的执行体，线程是调用该方法来具体运行任务的，如果任务没有被取消，则该方法会运行任务，并且设置结果到outcome变量里面，其代码：

```Java
public void run() {
    //1.如果任务不是初始化的NEW状态，或者使用CAS设置runner为当前线程失败，则直接返回
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    //2.如果任务不为null，并且任务状态为NEW，则执行任务
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            //2.1执行任务，如果OK则设置ran标记为true
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
            //2.2执行任务出现异常，则标记false，并且设置异常
                result = null;
                ran = false;
                setException(ex);
            }
            //3.任务执行正常，则设置结果
            if (ran)
                set(result);
        }
    } finally {
        
        
        runner = null;
        int s = state;
       //4.为了保证调用cancel(true)的线程在该run方法返回前中断任务执行的线程
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}

private void handlePossibleCancellationInterrupt(int s) {
    //为了保证调用cancel在该run方法返回前中断任务执行的线程
    //这里使用Thread.yield()让run方法执行线程让出cpu执行权，以便让
    //cancel(true)的线程执行cancel(true)中的代码中断任务线程
    if (s == INTERRUPTING)
        while (state == INTERRUPTING)
            Thread.yield(); // wait out pending interrupt
}
```



- 如上代码1，如果任务不是初始化的NEW状态，或者使用CAS设置runner为当前线程失败，则直接返回；这个可以防止同一个FutureTask对象被提交给多个线程来执行，导致run方法被多个线程同时执行造成混乱。

- 代码2，如果任务不为null，并且任务状态为NEW，则执行任务，其中代码2.1调用c.call()具体执行任务，如果任务执行OK，则调用set方法把结果记录到result,并设置ran为true；否则执行任务过程中抛出异常则设置result为null，ran为false，并且调用setException设置异常信息后，任务就处于终止状态，其中setException代码如下：

```Java
protected void setException(Throwable t) {
    //2.2.1
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = t;
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
        //2.2.1.1
        finishCompletion();
    }
}
```

如上代码，使用CAS尝试设置任务状态state为COMPLETING，如果CAS成功则把异常信息设置到outcome变量，并且设置任务状态为EXCEPTIONAL终止状态，然后调用finishCompletion，其代码：

```Java
private void finishCompletion() {
    //a遍历链表节点
    for (WaitNode q; (q = waiters) != null;) {
        //a.1 CAS设置当前waiters节点为null
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            //a.1.1
            for (;;) {
                //唤醒当前q节点对应的线程
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);
                }
                //获取q的下一个节点
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; //help gc
                q = next;
            }
            break;
        }
    }
    //b。所有阻塞的线程都被唤醒后，调用done方法
    done();

    callable = null;        // callable设置为null
}
```

如上代码比较简单，就是当任务已经处于终态后，激活waiters链表中所有由于等待获取结果而被阻塞的线程，并从waiters链表中移除他们，等所有由于等待该任务结果的线程被唤醒后，调用done（）方法，done默认实现为空实现。

上面我们讲了当任务执行过程中出现异常后如何处理的，下面我们看代码3，当任务是正常执行完毕后set(result)的实现：

```Java
protected void set(V v) {
    //3.1
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}
```

如上代码3.1，使用CAS尝试设置任务状态state为COMPLETING，如果CAS成功则把任务结果设置到outcome变量，并且设置任务状态为NORMAL终止状态，然后调用finishCompletion唤醒所有因为等待结果而被阻塞的线程。

## 3.4 FutureTask的get()方法
等待异步计算任务完成，并返回结果；如果当前任务计算还没完成则会阻塞调用线程直到任务完成；如果在等待结果的过程中有其他线程取消了该任务，则调用线程会抛出CancellationException异常；如果在等待结果的过程中有线程中断了该线程，则抛出InterruptedException异常；如果任务计算过程中抛出了异常，则会抛出ExecutionException异常。

其代码如下：

```Java
public V get() throws InterruptedException, ExecutionException {
    //1.获取状态，如有需要则等待
    int s = state;
    if (s <= COMPLETING)
        //等待任务终止
        s = awaitDone(false, 0L);
    //2.返回结果
    return report(s);
}
```

- 如上代码1获取任务的状态，如果任务状态的值小于等于COMPLETING则说明任务还没有被完成，所以调用awaitDone挂起调用线程。
- 代码2如果任务已经被完成，则返回结果。下面我们看awaitDone方法实现：

```Java
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    //1.1超时时间
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    //1.2 循环，等待任务完成
    for (;;) {
        //1.2.1任务被中断，则移除等待线程节点，抛出异常
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }
        //1.2.2 任务状态>COMPLETING说明任务已经终止
        int s = state;
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;   
        }
        //1.2.3任务状态为COMPLETING
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        //1.2.4为当前线程创建节点
        else if (q == null)
            q = new WaitNode();
        //1.2.5 添加当前线程节点到链表
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                               q.next = waiters, q);
        //1.2.6 设置了超时时间
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        //1.2.7没有设置超时时间
        else
            LockSupport.park(this);
    }
}
```




- 如上代码1.1获取设置的超时时间，如果传递的timed为false说明没有设置超时时间，则deadline设置为0
- 代码1.2无限循环等待任务完成，其中代码1.2.1如果发现当前线程被中断则从等待链表中移除当前线程对应的节点（如果队列里面有该节点的话），然后抛出
InterruptedException异常；代码1.2.2如果发现当前任务状态大于COMPLETING说明任务已经进入了终态（可能是NORMAL、EXCEPTIONAL、CANCELLED、INTERRUPTED中的一种），则把执行任务的线程的引用设置为null，并且返回结果；

- 代码1.2.3如果当前任务状态为COMPLETING说明任务已经接近完成了，就差设置结果到outCome了，则这时候让当前线程放弃CPU执行，意在让任务执行线程获取到CPU然后让任务状态从COMPLETING转换到终态NORMAL，这样可以避免当前调用get系列的方法的线程被挂起，然后在被唤醒的开销；
- 代码1.2.4如果当前q为null，则创建一个与当前线程相关的节点，代码1.2.5如果当前线程对应节点还没放入到waiters管理的等待列表，则使用CAS操作放入；

- 代码1.2.6如果设置了超时时间则使用LockSupport.parkNanos(this, nanos)让当前线程挂起deadline时间，否则会调用 LockSupport.park(this);让线程一直挂起直到其他线程调用了unpark方法,并且以当前线程为参数（比如finishCompletion（）方法）。

另外带超时参数的V get(long timeout, TimeUnit unit)方法与get()方法类似，只是添加了超时时间，这里不再累述。

## 3.5 FutureTask的cancel(boolean mayInterruptIfRunning)方法
尝试取消任务的执行，如果当前任务已经完成或者任务已经被取消了，则尝试取消任务会失败；如果任务还没被执行时候，调用了取消任务，则任务将永远不会被执行；如果任务已经开始运行了，这时候取消任务，则参数mayInterruptIfRunning将决定是否要将正在执行任务的线程中断，如果为true则标识要中断，否则标识不中断；

当调用取消任务后，在调用isDone()方法，后者会返回true，随后调用isCancelled()方法也会一直返回true；该方法会返回false，如果任务不能被取消，比如任务已经完成了，任务已经被取消了。

cancel方法的代码如下：

```Java
public boolean cancel(boolean mayInterruptIfRunning) {
    //1.如果任务状态为New则使用CAS设置任务状态为INTERRUPTING或者CANCELLED
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    //2.如果设置了中断正常执行任务线程，则中断
    try {    
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        //3.移除并激活所有因为等待结果而被阻塞的线程
        finishCompletion();
    }
    return true;
}
```


- 如上代码1，如果任务状态为New则使用CAS设置任务状态为INTERRUPTING或者CANCELLED，如果mayInterruptIfRunning设置为了true则说明要中断正在执行任务的线程，则使用CAS设置任务状态为INTERRUPTING，否则设置为CANCELLED；如果CAS失败则直接返回false。
- 如果CAS成功，则说明当前任务状态已经为INTERRUPTING或者CANCELLED，如果mayInterruptIfRunning为true则中断执行任务的线程，然后设置任务状态为INTERRUPTED。

- 最后代码3移除并激活所有因为等待结果而被阻塞的线程。

另外我们可以使用isCancelled()方法判断一个任务是否被取消了，使用isDone()方法判断一个任务是否处于终态了。



总结：当我们创建一个FutureTask时候，其任务状态初始化为NEW，当我们把任务提交到线程或者线程池后，会有一个线程来执行该FutureTask任务，具体是调用其run方法来执行任务。在任务执行过程中，我们可以在其他线程调用FutureTask的get()方法来等待获取结果，如果当前任务还在执行，则调用get的线程会被阻塞然后放入FutureTask内的阻塞链表队列；多个线程可以同时调用get方法，这些线程可能都会被阻塞到了阻塞链表队列。当任务执行完毕后会把结果或者异常信息设置到outcome变量，然后会移除和唤醒FutureTask内的阻塞链表队列里面的线程节点，然后这些由于调用FutureTask的get方法而被阻塞的线程就会被激活。

## 3.6 FutureTask的局限性
FutureTask虽然提供了用来检查任务是否执行完成、等待任务执行结果、获取任务执行结果的方法，但是这些特色并不足以让我们写出简洁的并发代码。比如它并不能清楚的表达出多个FutureTask之间的关系，另外为了从Future获取结果，我们必须调用get()方法，而该方法还是会在任务执行完毕前阻塞调用线程的，这明显不是我们想要的。

我们真正要想要的是：
- 可以将两个或者多个异步计算结合在一起变成一个，这包含两个或者多个异步计算是相互独立的时候或者第二个异步计算依赖第一个异步计算结果的时候。
- 对反应式编程的支持，也就是当任务计算完成后能进行通知，并且可以以计算结果作为一个行为动作的参数进行下一步计算，而不是仅仅提供调用线程以阻塞的方式获取计算结果。
- 可以通过编程的方式手动设置（代码的方式）Future的结果；FutureTask则不可以让用户通过函数来设置其计算结果，而是其任务内部来进行设置。
- 可以等多个Future对应的计算结果都出来后做一些事情

为了克服FutureTask的局限性，以及满足我们对异步编程的需要，JDK8中提供了CompletableFuture，CompletableFuture是一个可以通过编程方式显式的设置计算结果和状态以便让任务结束的Future,本书后面章节我们会具体讲解。


# 四、总结
本节内容摘自《Java异步编程实战》中的一小节。《Java异步编程实战》一书是国内首本系统讲解Java异步编程的书籍，本书涵盖了Java中常见的异步编程场景：这包含单JVM内的异步编程、以及跨主机通过网络通讯的远程过程调用的异步调用与异步处理、Web请求的异步处理、以及常见的异步编程框架原理解析和golang语言内置的异步编程能力。
