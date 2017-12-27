---
title: java 多线程任务取消与关闭
tags: ["java", "并发"]
categories: ["java", "并发编程"]
icon: fa-handshake-o
---

# 取消与关闭
java并未提供任何机制来安全的终止线程。但它提供了中断，这是一种协作机制，能够使一个线程终止另外一个线程的当前工作。

## 任务取消
在java中没有一种安全的抢占式方法来停止线程，因此也就没有一种安全的抢占式方法来停止任务。只有一些协作的机制，使请求取消的任务和代码都遵循一种协商好的协议。

### 设置某个“已请求取消”标志，任务定期检查该标志。

具体实现上，一般通过一个volatile的标记

	public class Task implement Runnable {

		private volatile boolean cancelled;

		public void run() {
			while(!cancelled) {

			}
		}

		public void cancel() {
			cancelled = true;
		}

	}

这种方案有一个缺陷，如果任务调用了一个阻塞方法，比如BlockingQueue.put, 那么，任务可能永远不会去检查取消标志，也就永远不会结束。

	public class BrokenPrimeProducer extends Thread {
	    private final BlockingQueue<BigInteger> queue;
	    private volatile boolean cancelled = false;

	    BrokenPrimeProducer(BlockingQueue<BigInteger> queue) {
	        this.queue = queue;
	    }

	    public void run() {
	        try {
	            BigInteger p = BigInteger.ONE;
	            while (!cancelled) {
	                queue.put(p = p.nextProbablePrime());
	            }
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	    }

	    public void cancel() {
	        cancelled = true;
	    }
	}


### 中断
java的一些特殊的阻塞库的方法支持中断。线程中断是一种协作机制，可用来终止线程。

每个线程都有一个boolean的中断状态。中断线程时，该状态将会被设置为true，Thread类中，

	public class Thread {
		// 中断目标线程
		public void interrupt() {}

		// 清除当前线程的中断状态
		public static boolean interrupted() {}

		// 返回目标线程的中断状态
		public boolean isInterrupted() {}
	}

阻塞库的方法，如Thread.sleep和Object.wait等都会检查线程何时中断，并且在发生中断时返回。响应中断执行的操作包括：清除中断状态，抛出InterruptedException。JVM不保证阻塞方法检测到中断的速度，但通常响应速度还是非常快的。

注意：调用interrupt并不意味着立即停止目标线程正在进行的工作，而只是传递了请求中断的消息。

前面例子中的问题如果用中断来解决如下，使用中断而不是boolean标志来取消即可。

	public class BrokenPrimeProducer extends Thread {
	    private final BlockingQueue<BigInteger> queue;

	    BrokenPrimeProducer(BlockingQueue<BigInteger> queue) {
	        this.queue = queue;
	    }

	    public void run() {
	        try {
	            BigInteger p = BigInteger.ONE;
	            while (!Thread.currentThread().isInterrupted()) {
	                queue.put(p = p.nextProbablePrime());
	            }
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	    }

	    public void cancel() {
	        interrupt();
	    }
	}

**一般而言，中断是实现取消的最合理的方式。**

- 处理时机

显然，作为一种协作机制，不会强求被中断线程一定要在某个点进行处理。实际上，被中断线程只需在合适的时候处理即可，如果没有合适的时间点，甚至可以不处理，这时候在任务处理层面，就跟没有调用中断方法一样。“合适的时候”与线程正在处理的业务逻辑紧密相关，例如，每次迭代的时候，进入一个可能阻塞且无法中断的方法之前等，但多半不会出现在某个临界区更新另一个对象状态的时候，因为这可能会导致对象处于不一致状态。

- 处理方式

一般说来，当可能阻塞的方法声明中有抛出InterruptedException则暗示该方法是可中断的，如BlockingQueue#put、BlockingQueue#take、Object#wait、Thread#sleep等，如果程序捕获到这些可中断的阻塞方法抛出的InterruptedException或检测到中断后，这些中断信息该如何处理？一般有以下两个通用原则：

1. 如果遇到的是可中断的阻塞方法抛出InterruptedException，可以继续向方法调用栈的上层抛出该异常，如果是检测到中断，则可清除中断状态并抛出InterruptedException，使当前方法也成为一个可中断的方法。
2. 若有时候不太方便在方法上抛出InterruptedException，比如要实现的某个接口中的方法签名上没有throws InterruptedException，这时就可以捕获可中断方法的InterruptedException并通过Thread.currentThread.interrupt()来重新设置中断状态。如果是检测并清除了中断状态，亦是如此。

一般的代码中，尤其是作为一个基础类库时，绝不应当吞掉中断，即捕获到InterruptedException后在catch里什么也不做，清除中断状态后又不重设中断状态也不抛出InterruptedException等。因为吞掉中断状态会导致方法调用栈的上层得不到这些信息。

当然，凡事总有例外的时候，当你完全清楚自己的方法会被谁调用，而调用者也不会因为中断被吞掉了而遇到麻烦，就可以这么做。

总得来说，就是要让方法调用栈的上层获知中断的发生。假设你写了一个类库，类库里有个方法amethod，在amethod中检测并清除了中断状态，而没有抛出InterruptedException，作为amethod的用户来说，他并不知道里面的细节，如果用户在调用amethod后也要使用中断来做些事情，那么在调用amethod之后他将永远也检测不到中断了，因为中断信息已经被amethod清除掉了。如果作为用户，遇到这样有问题的类库，又不能修改代码，那该怎么处理？只好在自己的类里设置一个自己的中断状态，在调用interrupt方法的时候，同时设置该状态，这实在是无路可走时才使用的方法。

## 参考资料
java并发编程实战
[详细分析java中断机制](http://www.infoq.com/cn/articles/java-interrupt-mechanism)
