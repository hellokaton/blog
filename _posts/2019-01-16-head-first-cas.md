---
layout: post
title: 深入浅出 CAS
tags: ["java", "cas"]
---

后端开发中大家肯定遇到过实现一个线程安全的计数器这种需求，根据经验你应该知道我们要在多线程中实现 **共享变量** 的原子性和可见性问题，于是锁成为一个不可避免的话题，今天我们讨论的是与之对应的无锁 CAS。本文会从怎么来的、是什么、怎么用、原理分析、遇到的问题等不同的角度带你真正搞懂 CAS。

# 为什么要无锁

我们一想到在多线程下保证安全的方式头一个要拎出来的肯定是锁，不管从硬件、操作系统层面都或多或少在使用锁。锁有什么缺点吗？当然有了，不然 JDK 里为什么出现那么多各式各样的锁，就是因为每一种锁都有其优劣势。

![lock]({{ "/public/images/2019/01/lock.png" | prepend: site.cdnurl }} "锁" ){:width="400px"}

使用锁就需要获得锁、释放锁，CPU 需要通过上下文切换和调度管理来进行这个操作，对于一个 **独占锁** 而言一个线程在持有锁后没执行结束其他的哥们就必须在外面等着，等到前面的哥们执行完毕 CPU 大哥就会把锁拿出来其他的线程来抢了（非公平）。锁的这种概念基于一种悲观机制，它总是认为数据会被修改，所以你在操作一部分代码块之前先加一把锁，操作完毕后再释放，这样就安全了。其实在 JDK1.5 使用 `synchronized` 就可以做到。

![cas]({{ "/public/images/2019/01/cas.png" | prepend: site.cdnurl }} "CAS" ){:width="420px"}

但是像上面的操作在多线程下会让 CPU 不断的切换，非常消耗资源，我们知道可以使用具体的某一类锁来避免部分问题。那除了锁的方式还有其他的吗？当然，有人就提出了无锁算法，比较有名的就是我们今天要说的 CAS（compare and swap），和锁不同的是它是一种乐观的机制，它认为别人去拿数据的时候不会修改，但是在修改数据的时候去判断一下数据此时的状态，这样的话 CPU 不会切换，在读多的情况下性能将得到大幅提升。当前我们使用的大部分 CPU 都有 CAS 指令了，从硬件层面支持无锁，这样开发的时候去调用就可以了。

不论是锁还是无锁都有其优劣势，后面我们也会通过例子说明 CAS 的问题。

# 什么是 CAS

前面提了无锁的 CAS，那到底 CAS 是个啥呢？我已经迫不及待了，我们来看看维基百科的解释

> 比较并交换(compare and swap, CAS)，是原子操作的一种，可用于在多线程编程中实现不被打断的数据交换操作，从而避免多线程同时改写某一数据时由于执行顺序不确定性以及中断的不可预知性产生的数据不一致问题。 该操作通过将内存中的值与指定数据进行比较，当数值一样时将内存中的数据替换为新的值。

CAS 给我们提供了一种思路，通过 **比较** 和 **替换** 来完成原子性，来看一段代码：

```c
int cas(long *addr, long old, long new) {
    /* 原子执行 */
    if(*addr != old)
        return 0;
    *addr = new;
    return 1;
}
```

这是一段 c 语言代码，可以看到有 3 个参数，分别是：

- `*addr`: 进行比较的值
- `old`: 内存当前值
- `new`: 准备修改的新值，写入到内存

只要我们当前传入的进行比较的值和内存里的值相等，就将新值修改成功，否则返回 0 告诉比较失败了。学过数据库的同学都知道悲观锁和乐观锁，乐观锁总是认为数据不会被修改。基于这种假设 CAS 的操作也认为内存里的值和当前值是相等的，所以操作总是能成功，我们可以不需要加锁就实现多线程下的原子性操作。

在多线程情况下使用 CAS 同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被阻塞挂起，而是告诉它这次修改失败了，你可以重新尝试，于是可以写这样的代码。

```c
while (!cas(&addr, old, newValue)) {

}
// success
printf("new value = %ld", addr);
```

不过这样的代码相信你可能看出其中的蹊跷了，这个我们后面来分析，下面来看看 Java 里是怎么用 CAS 的。

# Java 里的 CAS

还是前面的问题，如果让你用 Java 的 API 来实现你可能会想到两种方式，一种是加锁（可能是 synchronized 或者其他种类的锁），另一种是使用 `atomic` 类，如 `AtomicInteger`，这一系列类是在 JDK1.5 的时候出现的，在我们常用的 `java.util.concurrent.atomic` 包下，我们来看个例子：

```java
ExecutorService executorService = Executors.newCachedThreadPool();
AtomicInteger   atomicInteger   = new AtomicInteger(0);

for (int i = 0; i < 5000; i++) {
    executorService.execute(atomicInteger::incrementAndGet);
}

System.out.println(atomicInteger.get());
executorService.shutdown();
```

这个例子开启了 5000 个线程去进行累加操作，不管你执行多少次答案都是 5000。这么神奇的操作是如何实现的呢？就是依靠 CAS 这种技术来完成的，我们揭开 `AtomicInteger` 的老底看看它的代码：

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;

    /**
     * Creates a new AtomicInteger with the given initial value.
     *
     * @param initialValue the initial value
     */
    public AtomicInteger(int initialValue) {
        value = initialValue;
    }

    /**
     * Gets the current value.
     *
     * @return the current value
     */
    public final int get() {
        return value;
    }

    /**
     * Atomically increments by one the current value.
     *
     * @return the updated value
     */
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }

}
```

这里我只帖出了我们前面例子相关的代码，其他都是类似的，可以看到 `incrementAndGet` 调用了 `unsafe.getAndAddInt` 方法。`Unsafe` 
这个类是 JDK 提供的一个比较底层的类，它不让我们程序员直接使用，主要是怕操作不当把机器玩坏了。。。（其实可以通过反射的方式获取到这个类的实例）你会在 JDK 源码的很多地方看到这家伙，我们先说说它有什么能力：

- 内存管理：包括分配内存、释放内存
- 操作类、对象、变量：通过获取对象和变量偏移量直接修改数据
- 挂起与恢复：将线程阻塞或者恢复阻塞状态
- CAS：调用 CPU 的 CAS 指令进行比较和交换
- 内存屏障：定义内存屏障，避免指令重排序

这里只是大致提一下常用的操作，具体细节可以在文末的参考链接中查看。下面我们继续看 `unsafe` 的 `getAndAddInt` 在做什么。

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}

public native int getIntVolatile(Object var1, long var2);
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

其实很简单，先通过 `getIntVolatile` 获取到内存的当前值，然后进行比较，展开 `compareAndSwapInt` 方法的几个参数：

- `var1`: 当前要操作的对象（其实就是 `AtomicInteger` 实例）
- `var2`: 当前要操作的变量偏移量（可以理解为 CAS 中的内存当前值）
- `var4`: 期望内存中的值
- `var5`: 要修改的新值

所以 `this.compareAndSwapInt(var1, var2, var5, var5 + var4)` 的意思就是，比较一下 `var2` 和内存当前值 `var5` 是否相等，如果相等那我就将内存值 `var5` 修改为 `var5 + var4`（`var4` 就是 1，也可以是其他数）。

---

这里我们还需要解释一下 **偏移量** 是个啥？你在前面的代码中可能看到这么一段：

```java
// setup to use Unsafe.compareAndSwapInt for updates
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}

private volatile int value;
```

可以看出在静态代码块执行的时候将 `AtomicInteger` 类的 `value` 这个字段的偏移量获取出来，拿这个 long 数据干嘛呢？在 `Unsafe` 类里很多地方都需要传入 `obj` 和偏移量，结合我们说 `Unsafe` 的诸多能力，其实就是直接通过更底层的方式将对象字段在内存的数据修改掉。

使用上面的方式就可以很好的解决多线程下的原子性和可见性问题。由于代码里使用了 `do while` 这种循环结构，所以 CPU 不会被挂起，比较失败后重试，就不存在上下文切换了，实现了无锁并发编程。

# CAS 存在的问题

## 自旋的劣势

你留意上面的代码会发现一个问题，`while` 循环如果在最坏情况下总是失败怎么办？会导致 CPU 在不断处理。像这种 `while(!compareAndSwapInt)` 的操作我们称之为自旋，CAS 是乐观的，认为大家来并不都是修改数据的，现实可能出现非常多的线程过来都要修改这个数据，此时随着并发量的增加会导致 CAS 操作长时间不成功，CPU 也会有很大的开销。所以我们要清楚，如果是读多写少的情况也就满足乐观，性能是非常好的。

## ABA 问题

提到 CAS 不得不说 ABA 问题，它是说假如内存的值原来是 A，被一个线程修改为了 B，此时又有一个线程把它修改为了 A，那么 CAS 肯定是操作成功的。真的这样做的话代码可能就有 bug 了，对于修改数据为 B 的那个线程它应该读取到 B 而不是 A，如果你做过数据库相关的乐观锁机制可能会想到我们在比较的时候使用一个版本号 `version` 来进行判断就可以搞定。在 JDK 里提供了一个 `AtomicStampedReference` 类来解决这个问题，来看一个例子：

```java
int stamp = 10001;

AtomicStampedReference<Integer> stampedReference = new AtomicStampedReference<>(0, stamp);

stampedReference.compareAndSet(0, 10, stamp, stamp + 1);

System.out.println("value: " + stampedReference.getReference());
System.out.println("stamp: " + stampedReference.getStamp());
```

它的构造函数是 2 个参数，多传入了一个初始 _时间戳_，用这个戳来给数据加了一个版本，这样的话多个线程来修改如果提供的戳不同。在修改数据的时候除了提供一个新的值之外还要提供一个新的戳，这样在多线程情况下只要数据被修改了那么戳一定会发生改变，另一个线程拿到的是旧的戳所以会修改失败。

# 尝试应用

既然 CAS 提供了这么好的 API，我们不妨用它来实现一个简易版的独占锁。思路是当某个线程进入 `lock` 方法就比较锁对象的内存值是否是 false，如果是则代表这把锁它可以获取，获取后将内存之修改为 true，获取不到就自旋。在 `unlock` 的时候将内存值再修改为 false 即可，代码如下：

```java
public class SpinLock {

    private AtomicBoolean mutex = new AtomicBoolean(false);

    public void lock() {
        while (!mutex.compareAndSet(false, true)) {
            // System.out.println(Thread.currentThread().getName()+ " wait lock release");
        }
    }

    public void unlock() {
        while (!mutex.compareAndSet(true, false)) {
            // System.out.println(Thread.currentThread().getName()+ " wait lock release");
        }
    }
    
}
```

这里使用了 `AtomicBoolean` 这个类，当然用 `AtomicInteger` 也是可以的，因为我们只保存一个状态 `boolean` 占用比较小就用它了。这个锁的实现比较简单，缺点非常明显，由于 `while` 循环导致的自旋会让其他线程都在占用 CPU，但是也可以使用，关于锁的优化版本实现我会在后续的文章中进行改进和说明，正因为这些问题我们也会在后续研究 `AQS` 这把利器的优点。

# CAS 源码

看了上面的这些代码和解释相信你对 CAS 已经理解了，下面我们要说的原理是前面的 `native` 方法中的 C++ 代码写了什么，在 openjdk 的 `/hotspot/src/share/vm/prims` 目录中有一个 [Unsafe.cpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/tip/src/share/vm/prims/unsafe.cpp#l1185) 文件中有这样一段代码：

> 注意：这里以 hotspot 实现为例

```cpp
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  // 通过偏移量获取对象变量地址
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  // 执行一个原子操作
  // 如果结果和现在不同，就直接返回，因为有其他人修改了；否则会一直尝试去修改。直到成功。
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END
```

# 参考资料

- [cas wiki](https://zh.wikipedia.org/wiki/%E6%AF%94%E8%BE%83%E5%B9%B6%E4%BA%A4%E6%8D%A2)
- [说一说Java的Unsafe类](https://www.cnblogs.com/pkufork/p/java_unsafe.html)
- [Java Magic. Part 4: sun.misc.Unsafe](http://mishadoff.com/blog/java-magic-part-4-sun-dot-misc-dot-unsafe/)