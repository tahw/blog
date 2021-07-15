---
title: 并发CountDownLatch&Semaphore详解
date: 2021-07-07 12:42:20
tags:
 - 并发
catgories:
 - 并发
---

# Semaphore
&nbsp;&nbsp;&nbsp;&nbsp;信号量，它的作用是控制访问特性资源的线程数目

## 介绍
![semaphore](/images/pasted-64.png)

## 场景
* 停车场
* 并发超时降级（Hystrix）

<!-- more -->
## api使用
简单api使用
```java
public class SemaphoreTest {
    
    public static void main(String[] args) {
        // 定义信号量
        Semaphore semaphore = new Semaphore(2);
        for (int i = 0; i < 10; i++) {
            new Thread(()->{
                try {
                    // 获取信号量
                    semaphore.acquire();
                    Thread.sleep(2000);
                    System.out.println(Thread.currentThread().getName());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    // 释放信号量
                    semaphore.release();
                }
            }).start();
        }
    }
}
```
通过`java.util.concurrent.Semaphore.tryAcquire(long, java.util.concurrent.TimeUnit)`来实现降级方法
```java
public class SemaphoreToHystrixDemo {

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(1);
        Semaphore semaphore = new Semaphore(20); // 限流
        for (int i = 0; i < 100; i++) {
            new Thread(()->{
                try {
                    countDownLatch.await();
                    if (semaphore.tryAcquire(300, TimeUnit.MILLISECONDS)) { // 尝试获取锁，超过300ms就走降级方法
                        Thread.sleep(200);
                        System.out.println(Thread.currentThread().getName());
                    }else{
                        fallback();
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    semaphore.release();
                }
            }).start();
        }
        Thread.sleep(5000);
        countDownLatch.countDown();
    }

    /**
     * 降级方法
     */
    private static void fallback() {
        System.out.println(Thread.currentThread().getName()+"操作太快，请重新尝试！");
    }
}
```
## 原理解析
Semaphore源码很值得一读，这里也是AQS最后一部分，这里是利用AQS共享模式下操作节点，也利用了传播特性来使用

### Semaphore-构造函数
这里同样也是定义内部类Sync，来继承AQS
```java
public class Semaphore implements java.io.Serializable {
    private static final long serialVersionUID = -3222578661600680210L;
    /** All mechanics via AbstractQueuedSynchronizer subclass */
    private final Sync sync;

    /**
     * Synchronization implementation for semaphore.  Uses AQS state
     * to represent permits. Subclassed into fair and nonfair
     * versions.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        /*
         * 构造函数，就是设置state的状态
         * @param permits
         */
        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }

        /**
         * 非公平获取共享锁
         * @param acquires
         * @return
         */
        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                        compareAndSetState(available, remaining))
                    return remaining;
            }
        }

        /**
        * 释放锁
        */
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }

    /**
     * 非公平锁
     * NonFair version
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }

    /**
     * 公平锁
     * Fair version
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                        compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }

    /**
     * Creates a {@code Semaphore} with the given number of
     * permits and nonfair fairness setting.
     *
     * @param permits the initial number of permits available.
     *        This value may be negative, in which case releases
     *        must occur before any acquires will be granted.
     */
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

    /**
     * Creates a {@code Semaphore} with the given number of
     * permits and the given fairness setting.
     *
     * @param permits the initial number of permits available.
     *        This value may be negative, in which case releases
     *        must occur before any acquires will be granted.
     * @param fair {@code true} if this semaphore will guarantee
     *        first-in first-out granting of permits under contention,
     *        else {@code false}
     */
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }

    /**
     * Acquires a permit from this semaphore, blocking until one is
     * available, or the thread is {@linkplain Thread#interrupt interrupted}.
     *
     * <p>Acquires a permit, if one is available and returns immediately,
     * reducing the number of available permits by one.
     *
     * <p>If no permit is available then the current thread becomes
     * disabled for thread scheduling purposes and lies dormant until
     * one of two things happens:
     * <ul>
     * <li>Some other thread invokes the {@link #release} method for this
     * semaphore and the current thread is next to be assigned a permit; or
     * <li>Some other thread {@linkplain Thread#interrupt interrupts}
     * the current thread.
     * </ul>
     *
     * <p>If the current thread:
     * <ul>
     * <li>has its interrupted status set on entry to this method; or
     * <li>is {@linkplain Thread#interrupt interrupted} while waiting
     * for a permit,
     * </ul>
     * then {@link InterruptedException} is thrown and the current thread's
     * interrupted status is cleared.
     *
     * @throws InterruptedException if the current thread is interrupted
     */
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    /**
     * Acquires a permit from this semaphore, blocking until one is
     * available.
     *
     * <p>Acquires a permit, if one is available and returns immediately,
     * reducing the number of available permits by one.
     *
     * <p>If no permit is available then the current thread becomes
     * disabled for thread scheduling purposes and lies dormant until
     * some other thread invokes the {@link #release} method for this
     * semaphore and the current thread is next to be assigned a permit.
     *
     * <p>If the current thread is {@linkplain Thread#interrupt interrupted}
     * while waiting for a permit then it will continue to wait, but the
     * time at which the thread is assigned a permit may change compared to
     * the time it would have received the permit had no interruption
     * occurred.  When the thread does return from this method its interrupt
     * status will be set.
     */
    public void acquireUninterruptibly() {
        sync.acquireShared(1);
    }

    /**
     * Acquires a permit from this semaphore, only if one is available at the
     * time of invocation.
     *
     * <p>Acquires a permit, if one is available and returns immediately,
     * with the value {@code true},
     * reducing the number of available permits by one.
     *
     * <p>If no permit is available then this method will return
     * immediately with the value {@code false}.
     *
     * <p>Even when this semaphore has been set to use a
     * fair ordering policy, a call to {@code tryAcquire()} <em>will</em>
     * immediately acquire a permit if one is available, whether or not
     * other threads are currently waiting.
     * This &quot;barging&quot; behavior can be useful in certain
     * circumstances, even though it breaks fairness. If you want to honor
     * the fairness setting, then use
     * {@link #tryAcquire(long, TimeUnit) tryAcquire(0, TimeUnit.SECONDS) }
     * which is almost equivalent (it also detects interruption).
     *
     * @return {@code true} if a permit was acquired and {@code false}
     *         otherwise
     */
    public boolean tryAcquire() {
        return sync.nonfairTryAcquireShared(1) >= 0;
    }

    /**
     * Acquires a permit from this semaphore, if one becomes available
     * within the given waiting time and the current thread has not
     * been {@linkplain Thread#interrupt interrupted}.
     *
     * <p>Acquires a permit, if one is available and returns immediately,
     * with the value {@code true},
     * reducing the number of available permits by one.
     *
     * <p>If no permit is available then the current thread becomes
     * disabled for thread scheduling purposes and lies dormant until
     * one of three things happens:
     * <ul>
     * <li>Some other thread invokes the {@link #release} method for this
     * semaphore and the current thread is next to be assigned a permit; or
     * <li>Some other thread {@linkplain Thread#interrupt interrupts}
     * the current thread; or
     * <li>The specified waiting time elapses.
     * </ul>
     *
     * <p>If a permit is acquired then the value {@code true} is returned.
     *
     * <p>If the current thread:
     * <ul>
     * <li>has its interrupted status set on entry to this method; or
     * <li>is {@linkplain Thread#interrupt interrupted} while waiting
     * to acquire a permit,
     * </ul>
     * then {@link InterruptedException} is thrown and the current thread's
     * interrupted status is cleared.
     *
     * <p>If the specified waiting time elapses then the value {@code false}
     * is returned.  If the time is less than or equal to zero, the method
     * will not wait at all.
     *
     * @param timeout the maximum time to wait for a permit
     * @param unit the time unit of the {@code timeout} argument
     * @return {@code true} if a permit was acquired and {@code false}
     *         if the waiting time elapsed before a permit was acquired
     * @throws InterruptedException if the current thread is interrupted
     */
    public boolean tryAcquire(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    /**
     * Releases a permit, returning it to the semaphore.
     *
     * <p>Releases a permit, increasing the number of available permits by
     * one.  If any threads are trying to acquire a permit, then one is
     * selected and given the permit that was just released.  That thread
     * is (re)enabled for thread scheduling purposes.
     *
     * <p>There is no requirement that a thread that releases a permit must
     * have acquired that permit by calling {@link #acquire}.
     * Correct usage of a semaphore is established by programming convention
     * in the application.
     */
    public void release() {
        sync.releaseShared(1);
    }

    /**
     * Acquires the given number of permits from this semaphore,
     * blocking until all are available,
     * or the thread is {@linkplain Thread#interrupt interrupted}.
     *
     * <p>Acquires the given number of permits, if they are available,
     * and returns immediately, reducing the number of available permits
     * by the given amount.
     *
     * <p>If insufficient permits are available then the current thread becomes
     * disabled for thread scheduling purposes and lies dormant until
     * one of two things happens:
     * <ul>
     * <li>Some other thread invokes one of the {@link #release() release}
     * methods for this semaphore, the current thread is next to be assigned
     * permits and the number of available permits satisfies this request; or
     * <li>Some other thread {@linkplain Thread#interrupt interrupts}
     * the current thread.
     * </ul>
     *
     * <p>If the current thread:
     * <ul>
     * <li>has its interrupted status set on entry to this method; or
     * <li>is {@linkplain Thread#interrupt interrupted} while waiting
     * for a permit,
     * </ul>
     * then {@link InterruptedException} is thrown and the current thread's
     * interrupted status is cleared.
     * Any permits that were to be assigned to this thread are instead
     * assigned to other threads trying to acquire permits, as if
     * permits had been made available by a call to {@link #release()}.
     *
     * @param permits the number of permits to acquire
     * @throws InterruptedException if the current thread is interrupted
     * @throws IllegalArgumentException if {@code permits} is negative
     */
    public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }

    /**
     * Acquires the given number of permits from this semaphore,
     * blocking until all are available.
     *
     * <p>Acquires the given number of permits, if they are available,
     * and returns immediately, reducing the number of available permits
     * by the given amount.
     *
     * <p>If insufficient permits are available then the current thread becomes
     * disabled for thread scheduling purposes and lies dormant until
     * some other thread invokes one of the {@link #release() release}
     * methods for this semaphore, the current thread is next to be assigned
     * permits and the number of available permits satisfies this request.
     *
     * <p>If the current thread is {@linkplain Thread#interrupt interrupted}
     * while waiting for permits then it will continue to wait and its
     * position in the queue is not affected.  When the thread does return
     * from this method its interrupt status will be set.
     *
     * @param permits the number of permits to acquire
     * @throws IllegalArgumentException if {@code permits} is negative
     */
    public void acquireUninterruptibly(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireShared(permits);
    }

    /**
     * Acquires the given number of permits from this semaphore, only
     * if all are available at the time of invocation.
     *
     * <p>Acquires the given number of permits, if they are available, and
     * returns immediately, with the value {@code true},
     * reducing the number of available permits by the given amount.
     *
     * <p>If insufficient permits are available then this method will return
     * immediately with the value {@code false} and the number of available
     * permits is unchanged.
     *
     * <p>Even when this semaphore has been set to use a fair ordering
     * policy, a call to {@code tryAcquire} <em>will</em>
     * immediately acquire a permit if one is available, whether or
     * not other threads are currently waiting.  This
     * &quot;barging&quot; behavior can be useful in certain
     * circumstances, even though it breaks fairness. If you want to
     * honor the fairness setting, then use {@link #tryAcquire(int,
     * long, TimeUnit) tryAcquire(permits, 0, TimeUnit.SECONDS) }
     * which is almost equivalent (it also detects interruption).
     *
     * @param permits the number of permits to acquire
     * @return {@code true} if the permits were acquired and
     *         {@code false} otherwise
     * @throws IllegalArgumentException if {@code permits} is negative
     */
    public boolean tryAcquire(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        return sync.nonfairTryAcquireShared(permits) >= 0;
    }
    
    public boolean tryAcquire(int permits, long timeout, TimeUnit unit)
            throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        return sync.tryAcquireSharedNanos(permits, unit.toNanos(timeout));
    }
    
    public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
    }

    /**
     * Returns the current number of permits available in this semaphore.
     *
     * <p>This method is typically used for debugging and testing purposes.
     *
     * @return the number of permits available in this semaphore
     */
    public int availablePermits() {
        return sync.getPermits();
    }

    /**
     * Acquires and returns all permits that are immediately available.
     *
     * @return the number of permits acquired
     */
    public int drainPermits() {
        return sync.drainPermits();
    }

    /**
     * Shrinks the number of available permits by the indicated
     * reduction. This method can be useful in subclasses that use
     * semaphores to track resources that become unavailable. This
     * method differs from {@code acquire} in that it does not block
     * waiting for permits to become available.
     *
     * @param reduction the number of permits to remove
     * @throws IllegalArgumentException if {@code reduction} is negative
     */
    protected void reducePermits(int reduction) {
        if (reduction < 0) throw new IllegalArgumentException();
        sync.reducePermits(reduction);
    }

    /**
     * Returns {@code true} if this semaphore has fairness set true.
     *
     * @return {@code true} if this semaphore has fairness set true
     */
    public boolean isFair() {
        return sync instanceof FairSync;
    }

    /**
     * Queries whether any threads are waiting to acquire. Note that
     * because cancellations may occur at any time, a {@code true}
     * return does not guarantee that any other thread will ever
     * acquire.  This method is designed primarily for use in
     * monitoring of the system state.
     *
     * @return {@code true} if there may be other threads waiting to
     *         acquire the lock
     */
    public final boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }

    /**
     * Returns an estimate of the number of threads waiting to acquire.
     * The value is only an estimate because the number of threads may
     * change dynamically while this method traverses internal data
     * structures.  This method is designed for use in monitoring of the
     * system state, not for synchronization control.
     *
     * @return the estimated number of threads waiting for this lock
     */
    public final int getQueueLength() {
        return sync.getQueueLength();
    }

    /**
     * Returns a collection containing threads that may be waiting to acquire.
     * Because the actual set of threads may change dynamically while
     * constructing this result, the returned collection is only a best-effort
     * estimate.  The elements of the returned collection are in no particular
     * order.  This method is designed to facilitate construction of
     * subclasses that provide more extensive monitoring facilities.
     *
     * @return the collection of threads
     */
    protected Collection<Thread> getQueuedThreads() {
        return sync.getQueuedThreads();
    }

    /**
     * Returns a string identifying this semaphore, as well as its state.
     * The state, in brackets, includes the String {@code "Permits ="}
     * followed by the number of permits.
     *
     * @return a string identifying this semaphore, as well as its state
     */
    public String toString() {
        return super.toString() + "[Permits = " + sync.getPermits() + "]";
    }
}
```

### 获取信号量
```java
// Semaphore
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```
```java
// AbstractQueuedSynchronizer
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    // 中断抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

/**
* 获取锁失败，等待
    * Acquires in shared interruptible mode.
    * @param arg the acquire argument
    */
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 以共享模式加入到CLH等待队列
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                // 这里是如果是head后的第一个节点，然后可以尝试获取锁，这里和独占模式下获取锁的逻辑差不多
                // 注意这里多了一个setHeadAndPropagate，表示的是设置当前节点为head，然后传播
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            // 这个就是修改CLH队列的节点状态，前置节点状态修改为-1，然后park住当前线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
/**
* Sets head of queue, and checks if successor may be waiting
* in shared mode, if so propagating if either propagate > 0 or
* PROPAGATE status was set.
*
* @param node the node
* @param propagate the return value from a tryAcquireShared
*/
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    // 设置当前节点为head节点
    setHead(node);
    /*
        * Try to signal next queued node if:
        *   Propagation was indicated by caller,
        *     or was recorded (as h.waitStatus either before
        *     or after setHead) by a previous operation
        *     (note: this uses sign-check of waitStatus because
        *      PROPAGATE status may transition to SIGNAL.)
        * and
        *   The next node is waiting in shared mode,
        *     or we don't know, because it appears null
        *
        * The conservatism in both of these checks may cause
        * unnecessary wake-ups, but only when there are multiple
        * racing acquires/releases, so most need signals now or soon
        * anyway.
        * 传播特性就是调用doReleaseShared
        * 条件是
        * 1. 还有信号量
        * 2. 上一届的头节点为null
        * 3. 上一届的头节点状态小于0，或者头结点为空或者状态小于0
        */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```
### 释放信号量
```java
// Semaphore
public void release() {
    sync.releaseShared(1);
}
```
```java
// AbstractQueuedSynchronizer
public final boolean releaseShared(int arg) {
    /**
    * 释放锁，一定会返回true
    */
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

/**
    * 1. 判断当前CLH队列是否有两个节点，head和node节点
    * 2. 如果head节点是-1状态，则表示node节点肯定是入队等待了，现在将head设置为0，然后唤醒node节点
    * 3. 如果head节点不是-1，有可能是node节点以及排队，但是还没有等待，然后更新h状态从0设置成-3，就没有其他
    *
    * Release action for shared mode -- signals successor and ensures
    * propagation. (Note: For exclusive mode, release just amounts
    * to calling unparkSuccessor of head if it needs signal.)
    */
private void doReleaseShared() {
    /*
        * Ensure that a release propagates, even if there are other
        * in-progress acquires/releases.  This proceeds in the usual
        * way of trying to unparkSuccessor of head if it needs
        * signal. But if it does not, status is set to PROPAGATE to
        * ensure that upon release, propagation continues.
        * Additionally, we must loop in case a new node is added
        * while we are doing this. Also, unlike other uses of
        * unparkSuccessor, we need to know if CAS to reset status
        * fails, if so rechecking.
        */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                        !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```
![semaphore流程解析](/images/pasted-67.png)

### 引申：AQS中的传播有什么用？传播是干什么的？
<font color='red'><b>AQS传播的价值是在共享模式下，同在CLH等待的线程可以唤醒后续线程，避免线程在等待不会被唤醒的问题</b></font>
![semaphore无传播](/images/pasted-68.png)


# CountDownLatch
&nbsp;&nbsp;&nbsp;&nbsp;CountDownLatch这个类能够使一个线程等待其他工作线程完成后再执行。每当一个线程完成自己的任务后，计数器就减一，当计数器到达0时，它表示所有的线程都已经完成了任务，然后执行主线程

## 介绍
![CountDownLatch](/images/pasted-65.png)

## 场景
* 模拟高并发
* zookeeper分布式锁

## api使用
```java
public class CountDownLatchTest {

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(10);
        for (int i = 0; i < 10; i++) {
            new Thread(()->{
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName());
                countDownLatch.countDown();
            }).start();
        }
        countDownLatch.await();
        System.out.println("all thread exec finish");
    }
}
```
模拟高并发
```java
public class CountDownLatchToHighConcurrent {

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(1);
        for (int i = 0; i < 20; i++) {
            new Thread(()->{
                try {
                    countDownLatch.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName());
            }).start();
        }
        Thread.sleep(5000);
        countDownLatch.countDown();
        System.out.println("high concurrent start");
    }
}
```
## 原理解析
```java
// 构造函数
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```
`countDown()`方法，是不是很熟悉
```java
// AQS
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```
```java
// CountDownLatch
// 如果信号量为0，则返回false，获取信号量后还存在信号量，则返回false，否则返回true
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```
`await()`方法
```java
// AQS 共享模式下通用使用方法
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```
```java
// CountDownLatch
// 信号量如果存在，则需要等待park
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```
这个逻辑比较清晰了
* 第一个例子是如下逻辑
    1. 先countDown()，一直返回false，子线程可以一直在执行
    2. 主线程await()，由于信号量不能空，则入队等待
    3. 等到子线程最后一个线程执行完，信号量为0，则会调用doReleaseShared()方法，此时会主动唤醒等待的主线程
* 第二个例子是如下逻辑
    1. 显示调用await()，由于有信号量，所有的线程都在入队等待
    2. 当主线程调用countDown()，此时是由于信号量减一为0，则此时会调用doReleaseShared()方法，此时会主动调用排队的线程，则会传播调用后面所有线程


# CyclicBarrier
&nbsp;&nbsp;&nbsp;&nbsp;栅栏，让一组线程到达一个屏障时被阻塞，知道最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程都会继续运行

## 介绍
![cyclicBarrier](/images/pasted-66.png)

## 场景
* 多线程计算结果，合并计算结果

## api使用
&nbsp;&nbsp;&nbsp;&nbsp;注意，这里是创建了11个，主线程也是一个，而且这里可以重复利用
```java
public class CyclicBarrierTest {

    public static void main(String[] args) throws BrokenBarrierException, InterruptedException {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(11);
        for (int i = 0; i < 10; i++) {
            new Thread(()->{
                try {
                    cyclicBarrier.await();
                    System.out.println(Thread.currentThread().getName() + "start");
                    Double randomNum = Math.random() * 100;
                    Thread.sleep(randomNum.intValue() * 10);
                    System.out.println(Thread.currentThread().getName());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
        cyclicBarrier.await();
        System.out.println(Thread.currentThread().getName()+"start");



        for (int i = 0; i < 10; i++) {
            new Thread(()->{
                try {
                    cyclicBarrier.await();
                    System.out.println(Thread.currentThread().getName() + "start");
                    Double randomNum = Math.random() * 100;
                    Thread.sleep(randomNum.intValue() * 10);
                    System.out.println(Thread.currentThread().getName());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
        cyclicBarrier.await();
        System.out.println(Thread.currentThread().getName()+"start");

    }
}
```
其中最主要的就是这个api cyclicBarrier.await();
## 原理解析
看了下底层，使用的CLH和Condition，使用的是条件等待队列，可以重复利用，生成的nextGerention，唤醒，有机会可以再看看