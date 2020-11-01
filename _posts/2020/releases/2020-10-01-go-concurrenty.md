---
layout: post
title: Go并发编程之美-并发编程难在哪里
category: go
excerpt: 并发难道你了？
tags: [go]
--- 

# 一、前言
 编写正确的程序本身就不容易，编写正确的并发程序更是难中之难，那么并发编程究竟难道哪里那？本节我们就来一探究竟。
# 二、数据竞争
当两个或者多个线程（goroutine）在没有任何同步措施的情况下同时读写同一个共享资源时候，这多个线程（goroutine）就处于数据竞争状态，数据竞争会导致程序的运行结果超出写代码的人的期望。下面我们来看个例子：

```Java
package main
import (
	"fmt"
)

var a int

//goroutine1
func main() {
	
	//1,gouroutine2
	go func(){
		a = 1//1.1
	}()
	
	//2
	if 0 == a{//2.1
		fmt.Println(a)//2.2
	}
}
```

- 如上代码首先创建了一个int类型的变量，默认被初始化为0值，运行main函数会启动一个进程和这个进程中的一个运行main函数的goroutine(轻量级线程)
- 在main函数内使用go语句创建了一个新的goroutine（该goroutine运行匿名函数里面的内容）并启动运行，匿名函数内给变量赋值为1
- main函数里面代码2判断如果变量a的值为0，则打印a的值。


运行main函数后，启动的进程里面存在两个并发运行的线程，分别是开启的新goroutine(起名为goroutine2)和main函数所在的goroutine(起名为goroutine1),前者试图修改共享变量a,后者试图读取共享变量a,也就是存在两个线程在没有任何同步的情况下对同一个共享变量进行读写访问，这就出现了数据竞争，由于数据竞争存在，导致上面程序可能会有下面三种输出:

-  输出0，由于运行时调度系统的随机性，会存在goroutine1的2.2代码比goroutine2的代码1.1先执行
- 输出1，当存在goroutine1先执行代码2.1,然后goroutine2在执行代码1.1，最后goroutine1在执行代码2.2的时候
- 什么都不输出，当goroutine2执行先于goroutine1的2.1代码时候。

由于数据竞争的存在上面一段很短的代码会有三种可能的输出，究其原因是goroutine1和groutine2的运行时序是不确定的，也就是没有对他们的操作做同步，以便让这些内存操作变为可以预知的顺序执行。

这里编写程序者或许受单线程模型的影响认为代码1.1会先于代码2.1执行，当发现输出不符合预期时候，或许会在代码2.1前面让goroutine1 休眠一会确保goroutine2执行完毕1.1后在让goroutine1执行2.1，这看起来或许有效，但是这是非常低效，并且并不是所有情况下都可以解决的。

正确的做法可以使用信号量等同步措施，保证goroutine2执行完毕再让goroutine1执行代码2.1，如下面代码，我们使用sync包的WaitGroup来保证goroutine2执行完毕代码2.1后，goroutine1才可以执行步骤4.1,关于WaitGroup后面章节我们具体会讲解：

```Java
package main

import (
	"fmt"
	"sync"
)

var a int
var wg sync.WaitGroup//信号量
//goroutine1
func main() {
	//1.
	wg.Add(1);//一个信号
	
	//2. goroutine1
	go func(){
		a = 1//2.1
		wg.Done()
	}()
	
	wg.Wait()//3. 等待goroutine1运行结束
	
	//4
	if 0 == a{//4.1
		fmt.Println(a)//4.2
	}
}
```

# 三、操作的原子性
所谓原子性操作是指当执行一系列操作时候，这些操作那么全部被执行，那么全部不被执行，不存在只执行其中一部分的情况。在设计计数器时候一般都是先读取当前值，然后+1，然后更新，这个过程是读-改-写的过程，如果不能保证这个过程是原子性，那么就会出现线程安全问题。如下代码是线程不安全的，因为不能保证a++是原子性操作:

```java
package main

import (
	"fmt"
	"sync"
)

var count int32
var wg sync.WaitGroup //信号量
const THREAD_NUM = 1000

//goroutine1
func main() {
	//1.信号
	wg.Add(THREAD_NUM) 

	//2. goroutine
	for i := 0; i < THREAD_NUM; i++ {
		go func() {
			count++//2.1
			wg.Done()//2.2
		}()
	}

	wg.Wait() //3. 等待goroutine运行结束

	fmt.Println(count) //4输出计数
}
```
- 如上代码在main函数所在为goroutine内创建了THREAD_NUM个goroutine，每个新的goroutine执行代码2.1对变量count计数增加1。
- 这里创建了THREAD_NUM个信号量，用来在代码3处等待THREAD_NUM个goroutine执行完毕，然后输出最终计数，执行上面代码我们 期望输出1000，但是实际却不是。

这是因为a++操作本身不是原子性的，其等价于b := count;b=b+1;count=b;是三步操作，所以可能导致导致计数不准确，如下表：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/f32676e4f00b7621dcb8d4e23e3e2a5e.png)

假如当前count=0那么t1时刻线程A读取了count值到变量countA,然后t2时刻递增countA值为1，同时线程B读取count的值0放到内存countB值为0（因为countA还没有写入主内存），t3时刻线程A才把countA为1的值写入主内存，至此线程A一次计数完毕，同时线程B递增CountB值为1，t4时候线程B把countB值1写入内存，至此线程B一次计数完毕。明明是两次计数，最后结果是1而不是2。

上面的程序需要保证count++的原子性才是正确的，后面章节会知道使用sync/atomic包的一些原子性函数或者锁可以解决这个问题。
    
```java 
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

var count int32
var wg sync.WaitGroup //信号量
const THREAD_NUM = 1000

//goroutine1
func main() {
	//1.信号
	wg.Add(THREAD_NUM) 

	//2. goroutine
	for i := 0; i < THREAD_NUM; i++ {
		go func() {
			//count++//
			atomic.AddInt32(&count, 1)//2.1
			wg.Done()//2.2
		}()
	}

	wg.Wait() //3. 等待goroutine运行结束

	fmt.Println(count) //4输出计数
}
```
如上代码使用原子性操作可以保证每次输出都是1000


# 三、内存访问同步
上节原子性操作第一个例子有问题是因为count++操作是被分解为类似b := count;b=b+1;count =b; 的三部操作，而多个goroutine同时执行count++时候并不是顺序执行者三个步骤的，而是可能交叉访问的。所以如果能对内存变量的访问添加同步访问措施，就可以避免这个问题：

```Java
package main

import (
	"fmt"
	"sync"
)

var count int32
var wg sync.WaitGroup //信号量
var lock sync.Mutex   //互斥锁
const THREAD_NUM = 1000

//goroutine1
func main() {
	//1.信号
	wg.Add(THREAD_NUM)

	//2. goroutine
	for i := 0; i < THREAD_NUM; i++ {
		go func() {
			lock.Lock()   //2.1
			count++       //2.2
			lock.Unlock() //2.3
			wg.Done()     //2.4
		}()
	}

	wg.Wait() //3. 等待goroutine运行结束

	fmt.Println(count) //4输出计数
}
```
- 如上代码创建了一个互斥锁lock,然后goroutine内在执行count++前先获取锁，执行完毕后在释放锁。
- 当1000个goroutine同时执行到代码2.1时候只有一个线程可以获取到锁，其他的线程被阻塞，直到获取到锁的goroutine释放了锁。也就是这1000个线程的并发行使用锁转换为了串行执行，也就是对共享内存变量的访问施加了同步措施。


# 四、总结
本文我们从数据竞争、原子性操作、内存同步三个方面探索了并发编程到底难在哪里，后面章节我们会结合go的内存模型和happen-before原则在具体探索这些难点如何解决。
