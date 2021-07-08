title: BlockingQueue
author: jianghe
tags:
  - 并发
categories:
  - 并发
date: 2021-07-06 23:34:00
---

# 概要
&nbsp;&nbsp;&nbsp;&nbsp;BlockingQueue，是java.util.concurrent包提供的用于解决并发生产者-消费者问题最有用的类，他的特性是在任意时刻只有一个线程可以进行take或者put操作，并且BlockingQueue提供了超时return null的机制，在许多生产场景里都可以看到这个工具的使用身影。

# 队列特性
1. 线程安全
2. FIFO
3. 重复利用

# 队列类型
* 有界队列（bounded queue）- 定义了最大的容量
* 无界队列（unbounded queue）- 几乎可以无限延长


# 队列适用场景
1. 线程池
2. RocketMq
3. Netty
4. 生产者-消费者模型

<!-- more -->

# 常见队列
1. ArrayBlockingQueue
   * 由数组支持的有界队列
2. LinkedBlockingQueue
   * 由链表支持的可选有界队列
3. PriorityBlockingQueue
   * 优先级支持无界队列
4. DelayQueue
   * 优先级支持的、基于时间的调度队列
5. Deque 双端队列

![BlockingQueue依赖图](/images/pasted-57.png)

# 应用

## 生产者消费者
```java
/*队列*/
private static ArrayBlockingQueue<Integer> queue = new ArrayBlockingQueue<Integer>(10);

private static Integer PRODUCER_COUNT = 2;

private static Integer CONSUMER_COUNT = 2;

public static void main(String[] args) throws InterruptedException {
    producer();
    consumer();
}

private static Integer producerContext(){
    double random = Math.random();
    return (int)(random * 1000) % 1000;
}
/* 生产者 */
private static void producer() throws InterruptedException {
    for (int i = 0; i < PRODUCER_COUNT; i++) {
        new Thread(() -> {
            Integer context;
            while (true){
                context = producerContext();
                try {
                    queue.put(context);
                    System.out.println(Thread.currentThread().getName()+" put "+context);
                    if(context == 555){
                        break;
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}
/* 消费者 */
private static void consumer() throws InterruptedException {
    for (int i = 0; i < CONSUMER_COUNT; i++) {
        new Thread(() -> {
            Integer context;
            while (true){
                try {
                    context = queue.take();
                    System.out.println(Thread.currentThread().getName()+" take "+context);
                    if(context == 555){
                        break;
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        }).start();
    }
}
```
> 在这里我们可以看到，往queue队列存放元素是put方法，从queue队列里拿取元素是take方法
> 其实输出的结果可以观察下，输出的结果都是一批都是生产的线程，一批都是消费的线程，线程安全的

## BlockingQueue Api

### 存放元素
| 方法 | 描述 |
| :-----| :---- |
|boolean add(E e)|插入成功返回true，否则抛出IllegalStateException|
|boolean offer(E e)|插入成功返回true，否则返回false|
|void put(E e)|将元素插入到队列，如果队列满了，就阻塞，一直要插入成功|
|boolean offer(E e, long timeout, TimeUnit unit)|尝试将队列插入，如果队列满了，就阻塞有空间再插入|
### 拿取元素
| 方法 | 描述 |
| :-----| :---- |
|E take()|获取队列的头部节点，然后把队列的头部节点删除，如果队列为空，则阻塞等待|
|E poll(long timeout, TimeUnit unit)|timeout时间内获取队列的头部节点，如果超时的话，返回null|


# 源码解析
这里通过`ArrayBlockingQueue`来介绍队列的源码

## 初始化
初始化ArrayBlockingQueue，构建三个元素ReentrantLock、notEmpty Condition、notFull Condition，items这个是存放数组的元素
> 这里借助ReentrantLock来实现的
```java
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    # 数组长度
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```

## put
1. 先获取lock
2. 判断数组元素是否满了，如果是notFull等待，否则入队(入队后notEmpty唤醒)
3. 释放lock
```java
final ReentrantLock lock = this.lock;
lock.lockInterruptibly();
try {
    while (count == items.length)
        notFull.await();
    enqueue(e);
} finally {
    lock.unlock();
}
```
```java
/**
 * Inserts element at current put position, advances, and signals.
 * Call only when holding lock.
 */
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    notEmpty.signal();
}
```
![queue put](/images/pasted-60.png)

## take
1. 先获取lock
2. 判断数组元素是否空了，如果是notEmpty等待，否则出队(出队后notFull唤醒)
3. 释放lock
```java
final ReentrantLock lock = this.lock;
lock.lockInterruptibly();
try {
    while (count == 0)
        notEmpty.await();
    return dequeue();
} finally {
    lock.unlock();
}
```
```java
/**
 * Extracts element at current take position, advances, and signals.
 * Call only when holding lock.
 */
private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();
    return x;
}
```
![queue take](/images/pasted-61.png)

## await()
等待（AbstractQueuedSynchronizer）
```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    # 元素队列已经满了
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
![AQS等待](/images/pasted-58.png)

## signal()
唤醒（AbstractQueuedSynchronizer）
```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```
```java
/**
 * Removes and transfers nodes until hit non-cancelled one or
 * null. Split out from signal in part to encourage compilers
 * to inline the case of no waiters.
 * @param first (non-null) the first node on condition queue
 */
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
                (first = firstWaiter) != null);
}
```
```java
/**
 * Transfers a node from a condition queue onto sync queue.
 * Returns true if successful.
 * @param node the node
 * @return true if successfully transferred (else the node was
 * cancelled before signal)
 */
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     * 将节点状态-2修改为0
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     * 入CLH队列
     */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```
![AQS唤醒](/images/pasted-59.png)

## 流程图
![queue整体流程图](/images/pasted-62.png)








