title: 并发之AQS条件BlockingQueue
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
这里通过`ArrayBlockingQueue`来介绍队列的源码，这里还有一点要补充下，这里采用的是AQS的另外一个特性来实现的-条件等待队列

## 初始化
```java
/**
 * 初始化构造函数，会构造4个内容
 * 1. 数组-存放元素
 * 2. reentrantLock-锁
 * 3. notEmpty Condition-非空的条件等待对象
 * 4. notFull Condition-非满的条件等待对象
 */
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```
ReentrantLock我们已经很熟悉了，那这里看下Condition结构，这个也是AbstractQueuedSynchronizer的内部类，<font color='red'><b>其实这个比较重要的是firstWaiter、lastWaiter属性，注意这里还是使用的是Node，但是这里条件等待队列里面是采用Node的nextWaiter来连接条件等待队列</b></font>
```java
public class ConditionObject implements Condition, java.io.Serializable {
   private static final long serialVersionUID = 1173984872572414699L;
   /** First node of condition queue. */
   private transient Node firstWaiter;
   /** Last node of condition queue. */
   private transient Node lastWaiter;
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
    // 如果当前线程被中断，然后会抛出到用户方法内，被捕获可以去做线程中断的业务
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 判断是否在CLH队列里，如果是的话，就不会走到while里面，如果不是的话，那就是在Condition队列里面，就走到while里面，就park住了，（这里是不是有疑问，什么时候加在CLH队列，什么时候唤醒，接下来看看）
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 这块是不是很熟悉，判断当前线程是不是head后的第一个线程，如果是，尝试获取锁，获取不到锁那就park住，这里要park住两次
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
加入到条件队列里面，注意节点的状态是Node.CONDITION
```java
/**
* Adds a new waiter to wait queue.
* @return its new wait node
*/
private Node addConditionWaiter() {
   Node t = lastWaiter;
   // If lastWaiter is cancelled, clean out.
   if (t != null && t.waitStatus != Node.CONDITION) {
       unlinkCancelledWaiters();
       t = lastWaiter;
   }
   Node node = new Node(Thread.currentThread(), Node.CONDITION);
   if (t == null)
       firstWaiter = node;
   else
       t.nextWaiter = node;
   lastWaiter = node;
   return node;
}
```
进入到这里就是等待，进入到条件等待队列里面，然后释放锁，这里的释放锁就是把state设置成0，然后唤醒CLH队列里的第一个节点
```java
/**
 * Invokes release with current state value; returns saved state.
 * Cancels node and throws exception on failure.
 * @param node the condition node for this wait
 * @return previous sync state
 */
final int fullyRelease(Node node) {
  boolean failed = true;
  try {
      int savedState = getState();
      if (release(savedState)) {
          failed = false;
          return savedState;
      } else {
          throw new IllegalMonitorStateException();
      }
  } finally {
      if (failed)
          node.waitStatus = Node.CANCELLED;
  }
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

修改节点状态从Node.CONDITION为0，这个时候就可以准备入CLH队列了，然后把当前节点所在的CLH队列前置节点设置成-1
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

## 状态扭转图
![queue状态扭转图](/images/pasted-63.png)

## 整体流程图
![queue整体流程图](/images/pasted-62.png)








