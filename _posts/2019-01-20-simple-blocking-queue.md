---
layout: post
title: 阻塞队列的简单实现
tags: ["queue", "java"]
---

Java 并发常用的组件中有一种队列叫阻塞队列（BlockingQueue），当队列为空时，获取元素的线程会阻塞等待直到队列有数据；当队列满时，想要存储元素的线程会阻塞等待直到队列有空间。我们经常会用这种数据结构可以实现生产者、消费者模型。

本文会通过两种方式来实现简单的有界阻塞队列，在最后分别测试不同实现的性能差异。

# Monitor 和 Condition

看过 Java 并发相关书籍的同学应该都见过 Monitor 这个词，有人称为监视器也有人叫它管程，不过都是一个意思：一个同步工具，相当于操作系统中的互斥量（mutex），即值为 1 的信号量。

`synchronized` 关键词的背后就是靠 monitor 实现的，monitor 的重要特点是，**同一个时刻，只有一个线程能进入 monitor 中定义的临界区**，这使得 monitor 能够达到互斥的效果。但仅仅有互斥的作用是不够的，无法进入 monitor 临界区的线程应该被阻塞，在必要的时候可以被唤醒，所以 Java 提供了 `wait` 和 `notify`、`notifyAll` 的 API 给我们使用。

- `wait()`: 让当前线程进入等待队列，同时会释放锁，直到被唤醒。
- `notify()`: 从条件队列中随机唤醒一个线程，让它去参与锁竞争
- `notifyAll()`: 唤醒条件队列中的所有线程，让它们去参与锁竞争

---

实现同步的一种方式是使用 `synchronized` 关键字，还可以使用 `Lock` 接口下的实现来完成，比如 `ReentrantLock`，它是一把重入锁（synchronized 也是），基于 AQS 并发框架实现。我们可以使用它来进行加锁和释放锁，如果遇到有条件需要阻塞可以使用 `Condition` API。

- `Lock#newCondition()`: 创建一个新的条件
- `Condition#await()`: 让当前线程等待
- `Condition#signal()`: 唤醒一个等待线程

条件和锁总是息息相关，在没有 Lock 接口的时候你会发现 monitor 机制有一个严重的问题：一把锁只能对应一个条件（也就是只可以做一次 wait），那么在唤醒的时候就可能出现唤醒丢失。举个例子，在两个方法上有不同的条件会导致阻塞，它们持有一把锁，唤醒时候如果用 `notify` 只会从条件队列选择一个，使用 `notifyAll` 会带来大量的 CPU 上下文切换和锁竞争，伪代码如下：

```java
synchronized void foo() {
    while(CONDITION1){
        wait();
    }
    notifyAll();
}
synchronized void bar() {
    while(CONDITION2){
        wait();
    }
    notifyAll();
}
```

# 具体实现

## 实现思路

我们通过定义一个 `Queue` 接口来实现两种队列，该队列是有界队列，使用数组的方式实现，如果你有兴趣也可以使用链表或栈来实现这个队列。提供 `put` 方法添加元素（满了则阻塞），`take` 方法弹出元素（没有元素则阻塞）。

## 定义接口

```java
public interface Queue<E> {

    // 添加新元素，当队列满则阻塞
    void put(E e) throws InterruptedException;

    // 弹出队头元素，当队列空则阻塞
    E take() throws InterruptedException;

    // 队列元素个数
    int size();

    // 队列是否为空
    boolean isEmpty();

}
```

## 基于 synchronized 的实现

核心思路：

- 添加元素时队列满则阻塞
- 弹出元素时队列空则阻塞
- 添加元素后唤醒消费者
- 弹出元素后唤醒生产者

```java
public class BlockingQueueWithSync<E> implements Queue<E> {

    private E[] array;
    private int head;  // 队头指针
    private int tail;  // 队尾指针

    private volatile int size; // 队列元素个数

    public BlockingQueueWithSync(int capacity) {
        array = (E[]) new Object[capacity];
    }

    @Override
    public synchronized void put(E e) throws InterruptedException {
        // 当队列满的时候阻塞
        while (size == array.length) {
            this.wait();
        }

        array[tail] = e;
        // 队列装满后索引归零
        if (++tail == array.length) {
            tail = 0;
        }
        ++size;
        // 通知其他消费端有数据了
        this.notifyAll();
    }

    @Override
    public synchronized E take() throws InterruptedException {
        // 当队列空的时候阻塞
        while (isEmpty()) {
            this.wait();
        }

        E element = array[head];
        // 消费完后从0开始
        if (++head == array.length) {
            head = 0;
        }
        --size;
        // 通知其他生产者可以生产了
        this.notifyAll();
        return element;
    }

    @Override
    public synchronized boolean isEmpty() {
        return size == 0;
    }

    @Override
    public synchronized int size() {
        return size;
    }

}
```

## 基于 ReentrantLock 的实现

```java
public class BlockingQueueWithLock<E> implements Queue<E> {

    private E[] array;
    private int head;
    private int tail;

    private volatile int size;

    private Lock      lock     = new ReentrantLock();
    private Condition notFull  = lock.newCondition();
    private Condition notEmpty = lock.newCondition();

    public BlockingQueueWithLock(int capacity) {
        array = (E[]) new Object[capacity];
    }

    @Override
    public void put(E e) throws InterruptedException {
        lock.lockInterruptibly();
        try {
            // 队列满，阻塞
            while (size == array.length) {
                notFull.await();
            }
            array[tail] = e;
            if (++tail == array.length) {
                tail = 0;
            }
            ++size;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    @Override
    public E take() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            // 队列空，阻塞
            while (isEmpty()) {
                notEmpty.await();
            }
            E element = array[head];
            if (++head == array.length) {
                head = 0;
            }
            --size;
            // 通知isFull条件队列有元素出去
            notFull.signal();
            return element;
        } finally {
            lock.unlock();
        }
    }

    @Override
    public boolean isEmpty() {
        lock.lock();
        try {
            return size == 0;
        } finally {
            lock.unlock();
        }
    }

    @Override
    public int size() {
        lock.lock();
        try {
            return size;
        } finally {
            lock.unlock();
        }
    }

}
```

# 对比性能

```java
public class Benchmark {

    @Test
    public void testWithMonitor() {
        Queue<Integer> queue = new BlockingQueueWithSync<>(5);
        execute(queue);
    }

    @Test
    public void testWithCondition() {
        Queue<Integer> queue = new BlockingQueueWithLock<>(5);
        execute(queue);
    }

    private void execute(Queue<Integer> queue) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int i = 1; i <= 1000; i++) {
            final int finalNum = i;
            executorService.execute(() -> {
                try {
                    queue.put(finalNum);
                    Integer take = queue.take();
                    System.out.println("item: " + take);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        executorService.shutdown();
    }

}
```

这个测试程序让 2 个队列的可存储的元素数都为 5，开启 1000 个线程进行 `put` 和 `take` 操作，运行后查看总耗时。

![性能对比]({{ "/public/images/2019/01/blockingqueue_benchmark.png" | prepend: site.cdnurl }} "性能对比"){:width="600px"}

可以看出，使用 `synchronized` 的方式性能较差。
