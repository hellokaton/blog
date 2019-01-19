---
layout: post
title: wait 和 sleep 的时候到底在干嘛？
tags: ["java", "wait", "sleep"]
---

学习 Java 多线程基础的时候都遇到过 `wait` 和 `sleep` 这两个词，翻译成中文都是等待，从使用角度上我们知道前者用于线程间通信后者用户线程睡眠。本文带你复习和深入理解当我们使用这两个方法的时候 CPU 在干嘛，JVM 是如何做到的？

# 复习概念

重新打开 JDK 的 API 来看看之前是如何解释它们的，下来看看 `wait` 的定义。

```java
/**
 * 在其他线程调用此对象的 notify() 方法或 notifyAll() 方法前，导致当前线程等待。
 * 换句话说，此方法的行为就好像它仅执行 wait(0) 调用一样。
 * 
 * 当前线程必须拥有此对象管程（监视器）。该线程发布对此管程的所有权并等待，直到其他线程通过调用 notify 方法，
 * 或 notifyAll 方法通知在此对象的管程上等待的线程醒来。
 * 然后该线程将等到重新获得对管程的所有权后才能继续执行。
 * 
 * 对于一个参数的版本，实现中断和虚假唤醒是可能的，而且此方法应始终在循环中使用：
 * 
 * synchronized (obj) {
 *     while (<condition does not hold>)
 *        obj.wait();
 *     ... // 执行适合条件的操作
 * }
 * 
 * 该方法只应由作为此对象管程的所有者的线程来调用。
 * 
 * @throws IllegalMonitorStateException - 如果当前线程不是此对象管程的所有者。 
 * @throws InterruptedException - 如果在当前线程等待通知之前或者正在等待通知时，
 * 任何线程中断了当前线程。在抛出此异常时，当前线程的 中断状态 被清除。 
 */
public final void wait() throws InterruptedException

/**
 * @param timeout - 最多等待多少毫秒
 */
public final native void wait(long timeout) throws InterruptedException;
```

这个方法是 `Object` 类所拥有的，意味着 Java 中的任何对象都可以使用该方法，`wait` 会调用 `wait(0)` 也就是下面的本地方法。

`sleep` 的定义

```java
/**
 * 在指定的毫秒数内让当前正在执行的线程休眠（暂停执行），
 * 此操作受到系统计时器和调度程序精度和准确性的影响。该线程不丢失任何管程的所属权。 
 * 
 * @param millis - 以毫秒为单位的休眠时间。
 * @throws InterruptedException - 如果任何线程中断了当前线程。当抛出该异常时，当前线程的中断状态被清除。
 */
public static native void sleep(long millis) throws InterruptedException;
```

这个方法是 `Thread` 类的一个静态方法，在任何地方都可以使用，操作的是当前线程。对于前面的两种定义，先阅读两遍，然后我们来看看例子。

**线程状态图**

![thread_status]({{ "/public/images/2019/01/thread_status.png" | prepend: site.cdnurl }} "java thread status" )

# 通过例子解释

先来看看 `sleep` 方法，它比较简单。

```java
public class HiSleep implements Runnable {

    @Override
    public void run() {
        synchronized (this){
            System.out.println("我是子线程");
        }
    }

    public void hello() {
        synchronized (this) {
            System.out.println("Hello");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                System.out.println("主线程被中断了");
            }
            System.out.println("Hello End");
        }
    }

    public static void main(String[] args) {
        HiSleep hiSleep = new HiSleep();
        Thread t = new Thread(hiSleep);
        t.start();
        hiSleep.hello();
    }
}
```

这里 `HiSleep` 这个实例不论是作为线程还是执行 `hello` 方法持有的都是一把锁，我们让主线程先调用 `hello` 方法，然后启动一个线程。输出的结果是：

```shell
Hello
Hello End
我是子线程
```

## 这说明什么问题？

![sleep]({{ "/public/images/2019/01/sleep.png" | prepend: site.cdnurl }} "sleep" ){:width="500px"}

我们再回头来看看上面注释中写到 _该线程不丢失任何管程的所属权。_ 什么意思呢，就是说在执行 `sleep` 方法的时候，如果当前线程持有一把锁那么不会被释放。对于 `hello` 这个方法而言，调用者是主线程，持有的锁是 `hiSleep` 实例，正是因为 `synchronized` 内的代码没有执行完，锁没有被释放，所以上面的输出会等待 3 秒后输出 “Hello End”、“我是子线程”。

这个时候如果你通过 `jstack [pid]` 这个工具去看线程此时的状态（可以把休眠时间调大一点），就会发现是这样的：

```java
"main" #1 prio=5 os_prio=31 tid=0x00007fddb0002000 nid=0x2703 waiting on condition [0x000070000f99f000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at javabasic.HiSleep.hello(HiSleep.java:27)
	- locked <0x000000076acf4670> (a javabasic.HiSleep)
	at javabasic.HiSleep.main(HiSleep.java:39)
```

这里我们看的是主线程，它的状态是 `TIMED_WAITING`，意味着是一个有限时间的等待，等待结束后又进入运行态。

如果注释了 `hello` 方法中的同步片段就会发现输出 “Hello” 后马上输出 “我是子线程”，接下来等待小于 3 秒的时长后输出 “Hello End”，程序退出。这种情况是因为主线程没有持有任何锁，`t` 线程会立即获得自己这把锁，故会出现这样的结果。

## 如何改变线程休眠时间

在前面的代码中你可能看到了 `System.out.println("主线程被中断了");` 的输出语句，但是没有执行过，我们来看 `sleep` 方法抛出的异常解释：

> 如果任何线程中断了当前线程。当抛出该异常时，当前线程的中断状态被清除。

通过这个描述，我们可以尝试一下 “中断” 这个黑科技，需要修改一下代码：

```java
private Thread thread;

public HiSleep(Thread thread) {
    this.thread = thread;
}

@Override
public void run() {
    synchronized (this) {
        System.out.println("我是子线程");
        thread.interrupt();
    }
}

public void hello() {
    System.out.println("Hello");
    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        System.out.println("主线程被中断了");
    }
    System.out.println("Hello End");
}
```

创建 `HiSleep` 实例的时候把当前线程传进去，重新执行会发现输出：

```shell
Hello
我是子线程
主线程被中断了
Hello End
```

这里的中断操作是告诉主线程你收到了一个中断的状态，在 `catch` 块内你可以选择性的处理这个中断。

---

再来看一个 `wait` 的例子

```java
public class HiWait implements Runnable {

    @Override
    public void run() {
        synchronized (this){
            try {
                System.out.println("准备 wait");
                this.wait();
                System.out.println("wait 结束");
            } catch (InterruptedException e) {
                System.out.println("t 线程收到中断信号");
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        HiWait hiWait = new HiWait();
        Thread t = new Thread(hiWait);
        t.start();
    }

}
```

根据 JDK 里的描述就可以知道 `wait` 要使用，先要有一把锁，如果你没这么做的话就会抛出 `IllegalMonitorStateException`，这个可以自行尝试。前面我们通过 `sleep` 的分析可以得出结论，`sleep` 方法外部代码块如果持有锁的情况，该锁是不会释放的。

**执行 wait 后会怎样？**

上面的代码是有问题的，运行后会输出 “准备 wait” 然后一直阻塞。通过 `jstack [pid]` 这个命令看看程序里的栈情况：

```java
"Thread-0" #13 prio=5 os_prio=31 tid=0x00007fd327801800 nid=0xa703 in Object.wait() [0x0000700008652000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x000000076af894a8> (a javabasic.HiWait)
	at java.lang.Object.wait(Object.java:502)
	at javabasic.HiWait.run(HiWait.java:14)
	- locked <0x000000076af894a8> (a javabasic.HiWait)
	at java.lang.Thread.run(Thread.java:745)
```

`Thread-0` 就是我们启动的线程，此时可以看到状态是 `WAITING`，在对象管程上阻塞。也就是说 `wait` 会持有当前 `synchronized` 的锁不释放，根据前面 JDK 的描述也说了，**直到在其他线程调用此对象的 notify() 方法或 notifyAll()**。如果没有其他线程调用唤醒方法实际情况是怎样的呢？

![thread monitor]({{ "/public/images/2019/01/thread_monitor.png" | prepend: site.cdnurl }} "thread monitor" ){:width="580"}

每个线程都有两个 `ObjectMonitor` 对象列表，分别为 free 和 used 列表，如果当前 free 列表为空，线程将向全局global list请求分配ObjectMonitor。

、、此处 `wait` 我们做个假设，如果 `wait` 方法不会释放锁，就会导致一个问题，执行到 `this.wait();` 后开始阻塞等待

# 底层实现

# 参考

- [thread state java](https://stackoverflow.com/questions/11265289/thread-state-java)
- [Thread API](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html)
- [JVM源码分析之Object.wait/notify实现](https://www.jianshu.com/p/f4454164c017)