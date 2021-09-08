---
title: ThreadPoolExecutor详解
date: 2021-08-18 14:54:22
tags:
    - 并发
categories:
    - 并发
---
# ThreadPoolExecutor

## 线程
&nbsp;&nbsp;&nbsp;&nbsp;这里介绍下线程，线程是cpu调度的资源，线程是非常珍贵的资源。
![线程状态](/images/pasted-94.png)

## 介绍
&nbsp;&nbsp;&nbsp;&nbsp;线程池，juc下的线程池。百度百科，ThreadPoolExecutor减少了每个任务调用的开销，它们通常可以在执行大量异步任务时提供增强的性能，并且还可以提供绑定和管理资源（包括执行任务集时使用的线程）的方法。
&nbsp;&nbsp;&nbsp;&nbsp;如果并发请求比较多，但每个线程执行的时间都很短，这样就会频繁的创建和销毁线程，如此一来会大大的降低系统的效率。可能会出现每个服务器在为每个请求创建新的线程和销毁线程上花费的时间和消耗的系统资源要比实际处理的时间还要短。
&nbsp;&nbsp;&nbsp;&nbsp;线程池就是为线程生命周期的开销和资源不足问题提供解决方案的。通过对多个任务重用线程，线程创建的开销被分摊到了多个任务上。

## 场景
1. 单个任务处理时间比较短
2. 需要处理的任务数量很大

## 使用
```java
public class ThreadPoolExecutorTest {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        System.out.println(System.getProperty("jianghe"));
        ThreadPoolExecutor executor = new ThreadPoolExecutor(1,10,5, TimeUnit.SECONDS,new ArrayBlockingQueue<>(20), new ThreadPoolExecutor.CallerRunsPolicy());
        for (int i = 0; i < 100; i++) {
            executor.execute(() -> { // 无返回值
                System.out.println(Thread.currentThread().getName());
            });
            Future<String> submit = executor.submit(() -> { // 有返回值
                Random random = new Random();
                return random.toString();
            });
            System.out.println(submit.get());
            if(i == 10){
                executor.shutdownNow(); // 线程池立即关闭
            }
        }
    }
}
```

## 结构
![ThreadPoolExecutor](/images/pasted-93.png)
> 上面图比较重要，可以多看下，也可以看完下面再回来看下。

<!-- more -->

## 参数
```java
/**
     * The main pool control state, ctl, is an atomic integer packing
     * two conceptual fields
     *   workerCount, indicating the effective number of threads
     *   runState,    indicating whether running, shutting down etc
     *
     * In order to pack them into one int, we limit workerCount to
     * (2^29)-1 (about 500 million) threads rather than (2^31)-1 (2
     * billion) otherwise representable. If this is ever an issue in
     * the future, the variable can be changed to be an AtomicLong,
     * and the shift/mask constants below adjusted. But until the need
     * arises, this code is a bit faster and simpler using an int.
     *
     * The workerCount is the number of workers that have been
     * permitted to start and not permitted to stop.  The value may be
     * transiently different from the actual number of live threads,
     * for example when a ThreadFactory fails to create a thread when
     * asked, and when exiting threads are still performing
     * bookkeeping before terminating. The user-visible pool size is
     * reported as the current size of the workers set.
     *
     * The runState provides the main lifecycle control, taking on values:
     *
     *   RUNNING:  Accept new tasks and process queued tasks
     *   SHUTDOWN: Don't accept new tasks, but process queued tasks
     *   STOP:     Don't accept new tasks, don't process queued tasks,
     *             and interrupt in-progress tasks
     *   TIDYING:  All tasks have terminated, workerCount is zero,
     *             the thread transitioning to state TIDYING
     *             will run the terminated() hook method
     *   TERMINATED: terminated() has completed
     *
     * The numerical order among these values matters, to allow
     * ordered comparisons. The runState monotonically increases over
     * time, but need not hit each state. The transitions are:
     *
     * RUNNING -> SHUTDOWN
     *    On invocation of shutdown(), perhaps implicitly in finalize()
     * (RUNNING or SHUTDOWN) -> STOP
     *    On invocation of shutdownNow()
     * SHUTDOWN -> TIDYING
     *    When both queue and pool are empty
     * STOP -> TIDYING
     *    When pool is empty
     * TIDYING -> TERMINATED
     *    When the terminated() hook method has completed
     *
     * Threads waiting in awaitTermination() will return when the
     * state reaches TERMINATED.
     *
     * Detecting the transition from SHUTDOWN to TIDYING is less
     * straightforward than you'd like because the queue may become
     * empty after non-empty and vice versa during SHUTDOWN state, but
     * we can only terminate if, after seeing that it is empty, we see
     * that workerCount is 0 (which sometimes entails a recheck -- see
     * below).
     */
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0)); // 线程池ctl，包含线程池状态和线程工作数量
    private static final int COUNT_BITS = Integer.SIZE - 3; // Integer.SIZE = 32，COUNT_BITS = 29
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1; // 二级制表示 29个1，大约是5亿

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; } // 线程池状态
    private static int workerCountOf(int c)  { return c & CAPACITY; } // 有效的线程个数
    private static int ctlOf(int rs, int wc) { return rs | wc; }
    /**
     * The queue used for holding tasks and handing off to worker
     * threads.  We do not require that workQueue.poll() returning
     * null necessarily means that workQueue.isEmpty(), so rely
     * solely on isEmpty to see if the queue is empty (which we must
     * do for example when deciding whether to transition from
     * SHUTDOWN to TIDYING).  This accommodates special-purpose
     * queues such as DelayQueues for which poll() is allowed to
     * return null even if it may later return non-null when delays
     * expire.
     */
    private final BlockingQueue<Runnable> workQueue; // 阻塞队列

    /**
     * Lock held on access to workers set and related bookkeeping.
     * While we could use a concurrent set of some sort, it turns out
     * to be generally preferable to use a lock. Among the reasons is
     * that this serializes interruptIdleWorkers, which avoids
     * unnecessary interrupt storms, especially during shutdown.
     * Otherwise exiting threads would concurrently interrupt those
     * that have not yet interrupted. It also simplifies some of the
     * associated statistics bookkeeping of largestPoolSize etc. We
     * also hold mainLock on shutdown and shutdownNow, for the sake of
     * ensuring workers set is stable while separately checking
     * permission to interrupt and actually interrupting.
     */
    private final ReentrantLock mainLock = new ReentrantLock(); // 可重入锁

    /**
     * Set containing all worker threads in pool. Accessed only when
     * holding mainLock.
     */
    private final HashSet<Worker> workers = new HashSet<Worker>(); // 线程池工作线程

    /**
     * Wait condition to support awaitTermination
     */
    private final Condition termination = mainLock.newCondition(); // AQS条件锁，支持awaitTermination

    /**
     * Tracks largest attained pool size. Accessed only under
     * mainLock.
     */
    private int largestPoolSize; // 线程池工作的时候最大的线程数量，有可能大于maximumPoolSize

    /**
     * Counter for completed tasks. Updated only on termination of
     * worker threads. Accessed only under mainLock.
     */
    private long completedTaskCount; // 完成的数量

    /*
     * All user control parameters are declared as volatiles so that
     * ongoing actions are based on freshest values, but without need
     * for locking, since no internal invariants depend on them
     * changing synchronously with respect to other actions.
     */

    /**
     * Factory for new threads. All threads are created using this
     * factory (via method addWorker).  All callers must be prepared
     * for addWorker to fail, which may reflect a system or user's
     * policy limiting the number of threads.  Even though it is not
     * treated as an error, failure to create threads may result in
     * new tasks being rejected or existing ones remaining stuck in
     * the queue.
     *
     * We go further and preserve pool invariants even in the face of
     * errors such as OutOfMemoryError, that might be thrown while
     * trying to create threads.  Such errors are rather common due to
     * the need to allocate a native stack in Thread.start, and users
     * will want to perform clean pool shutdown to clean up.  There
     * will likely be enough memory available for the cleanup code to
     * complete without encountering yet another OutOfMemoryError.
     */
    private volatile ThreadFactory threadFactory; // 线程工厂，创建线程的工厂

    /**
     * Handler called when saturated or shutdown in execute.
     */
    private volatile RejectedExecutionHandler handler; // 拒绝策略

    /**
     * Timeout in nanoseconds for idle threads waiting for work.
     * Threads use this timeout when there are more than corePoolSize
     * present or if allowCoreThreadTimeOut. Otherwise they wait
     * forever for new work.
     */
    private volatile long keepAliveTime; // 存活时间，空间线程等待工作的最长时间

    /**
     * If false (default), core threads stay alive even when idle.
     * If true, core threads use keepAliveTime to time out waiting
     * for work.
     */
    private volatile boolean allowCoreThreadTimeOut; // 是否允许核心线程数超时，默认是false，当空闲的时候核心线程还是存在，true就是超时工作

    /**
     * Core pool size is the minimum number of workers to keep alive
     * (and not allow to time out etc) unless allowCoreThreadTimeOut
     * is set, in which case the minimum is zero.
     */
    private volatile int corePoolSize; // 核心线程数

    /**
     * Maximum pool size. Note that the actual maximum is internally
     * bounded by CAPACITY.
     */
    private volatile int maximumPoolSize; // 最大线程数
```
线程池状态位分解
![ThreadPoolExecutor参数](/images/pasted-95.png)

线程状态扭转
![ThreadPoolExecutor状态扭转](/images/pasted-96.png)
> <font color='red'><b>根据上图我们发现，只有RUNNING状态的表示的是负数，其他都是正数，状态大于0，线程池是不可使用了，如果状态等于0，如果队列还有任务，可以执行，否则都不能执行了</b></font>

## 概念
ThreadPoolExecutor大量使用的位运算，这里稍微介绍下位运算相关的概念

### 补码
负数的补码则是将其对应正数按位取反再加1
`-1` 就是 `11111111111111111111111111111111`

### 移位操作
1. `<<`:左移运算符，num << 1,相当于num乘以2
2. `>>`:右移运算符，num >> 1,相当于num除以2
3. `>>>`:无符号右移，忽略符号位，空位都以0补齐
4. `~`:非，就是将所属的位0变成1，1变成0，包括符号位，其中~-1是0

## 源码解析
### 构造函数
```java
/**
    * Creates a new {@code ThreadPoolExecutor} with the given initial
    * parameters.
    *
    * @param corePoolSize the number of threads to keep in the pool, even
    *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
    * @param maximumPoolSize the maximum number of threads to allow in the
    *        pool
    * @param keepAliveTime when the number of threads is greater than
    *        the core, this is the maximum time that excess idle threads
    *        will wait for new tasks before terminating.
    * @param unit the time unit for the {@code keepAliveTime} argument
    * @param workQueue the queue to use for holding tasks before they are
    *        executed.  This queue will hold only the {@code Runnable}
    *        tasks submitted by the {@code execute} method.
    * @param threadFactory the factory to use when the executor
    *        creates a new thread
    * @param handler the handler to use when execution is blocked
    *        because the thread bounds and queue capacities are reached
    * @throws IllegalArgumentException if one of the following holds:<br>
    *         {@code corePoolSize < 0}<br>
    *         {@code keepAliveTime < 0}<br>
    *         {@code maximumPoolSize <= 0}<br>
    *         {@code maximumPoolSize < corePoolSize}
    * @throws NullPointerException if {@code workQueue}
    *         or {@code threadFactory} or {@code handler} is null
    */
public ThreadPoolExecutor(int corePoolSize, // 核心线程数
                            int maximumPoolSize, // 最大线程数
                            long keepAliveTime, // 等待工作的空闲线程的超时（纳秒）
                            TimeUnit unit,  // 单位
                            BlockingQueue<Runnable> workQueue, // 阻塞队列
                            ThreadFactory threadFactory,    // 线程工厂
                            RejectedExecutionHandler handler) { // 拒绝异常处理器
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
在很多面试过程中，线程池的参数是需要考察的，这些指标都是很重要
1. corePoolSize 核心线程数
2. maximumPoolSize 最大线程数
3. keepAliveTime 空闲线程等待工作的超时时间
4. unit 单位
5. workQueue 阻塞队列
6. threadFactory 线程工厂
7. handler 拒绝策略处理器

### execute
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
        * Proceed in 3 steps:
        *
        * 1. If fewer than corePoolSize threads are running, try to
        * start a new thread with the given command as its first
        * task.  The call to addWorker atomically checks runState and
        * workerCount, and so prevents false alarms that would add
        * threads when it shouldn't, by returning false.
        *
        * 2. If a task can be successfully queued, then we still need
        * to double-check whether we should have added a thread
        * (because existing ones died since last checking) or that
        * the pool shut down since entry into this method. So we
        * recheck state and if necessary roll back the enqueuing if
        * stopped, or start a new thread if there are none.
        *
        * 3. If we cannot queue task, then we try to add a new
        * thread.  If it fails, we know we are shut down or saturated
        * and so reject the task.
        */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) { // 线程工作数量小于核心线程数，就使用核心线程数量
        if (addWorker(command, true)) // 将当前线程以核心线程的方式执行
            return;
        c = ctl.get(); // 加入核心线程失败，重新获取ctl
    }
    if (isRunning(c) && workQueue.offer(command)) { // 线程是运行状态，将线程加入到队列中
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command)) // 如果线程是非运行状态，则从队列移除这个线程
            reject(command); // 拒绝策略
        else if (workerCountOf(recheck) == 0) // 线程是运行状态，线程池工作线程数量为0
            addWorker(null, false); // 线程加入一个null的线程，但是不会去执行
    }
    else if (!addWorker(command, false)) // 线程不是运行状态 或者 队列满了，就会让非核心线程来执行
        reject(command); // 如果非核心线程来执行失败后，就会走到拒绝策略
}
```
![线程池工作流程图](/images/pasted-97.png)

![线程池工作顺序图](/images/pasted-98.png)


### addWorker
```java
/*
    * Methods for creating, running and cleaning up after workers
    */

/**
    * Checks if a new worker can be added with respect to current
    * pool state and the given bound (either core or maximum). If so,
    * the worker count is adjusted accordingly, and, if possible, a
    * new worker is created and started, running firstTask as its
    * first task. This method returns false if the pool is stopped or
    * eligible to shut down. It also returns false if the thread
    * factory fails to create a thread when asked.  If the thread
    * creation fails, either due to the thread factory returning
    * null, or due to an exception (typically OutOfMemoryError in
    * Thread.start()), we roll back cleanly.
    *
    * @param firstTask the task the new thread should run first (or
    * null if none). Workers are created with an initial first task
    * (in method execute()) to bypass queuing when there are fewer
    * than corePoolSize threads (in which case we always start one),
    * or when the queue is full (in which case we must bypass queue).
    * Initially idle threads are usually created via
    * prestartCoreThread or to replace other dying workers.
    *
    * @param core if true use corePoolSize as bound, else
    * maximumPoolSize. (A boolean indicator is used here rather than a
    * value to ensure reads of fresh values after checking other pool
    * state).
    * @return true if successful
    */
private boolean addWorker(Runnable firstTask, boolean core) { // core true表示是使用核心线程数，false表示是最大线程数
    retry:
    for (;;) { // 自旋判断，校验
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
                firstTask == null &&
                ! workQueue.isEmpty())) // 这个是表示什么?rs表示的是状态，这个表示如果状态是stop以上，线程池已经是停止的状态，那就要停止，如果rs是SHUTDOWN状态，但是如果workQueue是空的 并且 firstTask也是空的，那就是没有任务要执行，也是需要停止的
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize)) // 如果当前工作线程数量已经超过核心线程数或者最大线程数，那就不会往下执行
                return false;
            if (compareAndIncrementWorkerCount(c)) // 直到工作数量加1成功，就退出循环
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs) // 判断线程池状态是否被其他线程修改
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask); // 创建Worker，将当前线程放入到Worker
        final Thread t = w.thread; // 这里又出现了一个Thread，是和firstTask不同的，下面有详细介绍Worker
        if (t != null) { // 注意，这里会判断t，但是是从Worker里面出来的Thread
            final ReentrantLock mainLock = this.mainLock; 
            mainLock.lock(); // 获取ReentrantLock重入锁
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get()); // 获取线程池状态

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {  // 再次判断线程池状态，这里为什么？有可能在获取锁的时候线程池状态修改了
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w); // 加入到workers里面？这里有什么作用？这里先说明下吧，获取线程池工作线程；中断线程；总的线程数
                    int s = workers.size();
                    if (s > largestPoolSize) // largestPoolSize记录最大的线程数
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start(); // 这里是启动，注意，这是在释放锁后再去启动的
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```


### class Worker
&nbsp;&nbsp;&nbsp;&nbsp;这个`Worker`是`ThreadPoolExecutor`类的内部类，是非常重要的，是线程池中实际工作的类，这个方法继承了`AbstractQueuedSynchronizer`，里面属性有Thread类，还要Runnable，这个Runnable里就是用户定义的线程，completedTasks表示的是完成数量，<font color='red'><b>注意这个state是-1</b></font>
&nbsp;&nbsp;&nbsp;&nbsp;线程池中的每一个线程被封装成一个Worker对象，Worker类还实现了Runnable的接口，本身也是一个线程，所有Worker对象启动的时候会调用Worker的start的方法，最后会调用run方法，那run方法会调用用户自己定义的线程的run方法，这里只用一个线程的资源，是Worker类的
```java
/**
    * Class Worker mainly maintains interrupt control state for
    * threads running tasks, along with other minor bookkeeping.
    * This class opportunistically extends AbstractQueuedSynchronizer
    * to simplify acquiring and releasing a lock surrounding each
    * task execution.  This protects against interrupts that are
    * intended to wake up a worker thread waiting for a task from
    * instead interrupting a task being run.  We implement a simple
    * non-reentrant mutual exclusion lock rather than use
    * ReentrantLock because we do not want worker tasks to be able to
    * reacquire the lock when they invoke pool control methods like
    * setCorePoolSize.  Additionally, to suppress interrupts until
    * the thread actually starts running tasks, we initialize lock
    * state to a negative value, and clear it upon start (in
    * runWorker).
    */
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    /**
        * This class will never be serialized, but we provide a
        * serialVersionUID to suppress a javac warning.
        */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
        * Creates with given first task and thread from ThreadFactory.
        * @param firstTask the first task (null if none)
        */
    Worker(Runnable firstTask) {
        // 默认创建的时候state=-1，这个不能获取锁，这个有什么作用？
        setState(-1); // inhibit interrupts until runWorker 
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }

    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.

    protected boolean isHeldExclusively() { // 是否是独占，如果等于1就是独占，如果等于0就是非独占
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) { // 这里是修改state状态从0到1，修改成功后，将当前线程修改成当前线程，其他线程来获取是获取不到的
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) { // 直接释放锁，修改状态为0，直接返回true
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) { // 如果当前线程已经被中断了，就不做处理，如果没有被中断，就中断线程
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```
> <font color='red'><b>引申：这里有一个疑问，为什么Worker不直接使用ReentrantLock，而是自己继承AQS来实现锁的逻辑？具体为什么？</b></font>
这里解释一下：其实最主要的是`tryAcquire`方法导致的，`tryAcquire`这个方法是不重入的，而`ReentrantLock`是可重入的，那为什么Worker不能使用可重入的锁？
1. lock方法一旦获取到线程，就表示当前线程正在执行任务
2. 如果正在执行任务，是不允许中断任务的
3. 如果该线程不是独占任务，就是空闲状态，说明它不是在处理任务，这个时候可以中断线程
4. 线程池中会调用`interruptIdleWorkers`，获取到这个，需要获取到线程锁，才能去中断空闲状态
5. 还有一个原因是，我们不希望任务在调用像`setCorePoolSize`这样线程池方法时重新获取锁，如果使用`ReentrantLock`，会导致线程会中断。

### runWorker
这个`runWorker`并不是在`Worker`里面，而是在`ThreadPoolExecutor`里面的方法。
> <font color='red'><b>这里要引申出来一个问题，如果一个线程池里面添加中一个线程执行抛出了异常，那后面添加的线程还能正常执行吗？答案是能正常执行，具体逻辑就是在 `task.run();`处理的</b></font>

```java
/**
    * Main worker run loop.  Repeatedly gets tasks from queue and
    * executes them, while coping with a number of issues:
    *
    * 1. We may start out with an initial task, in which case we
    * don't need to get the first one. Otherwise, as long as pool is
    * running, we get tasks from getTask. If it returns null then the
    * worker exits due to changed pool state or configuration
    * parameters.  Other exits result from exception throws in
    * external code, in which case completedAbruptly holds, which
    * usually leads processWorkerExit to replace this thread.
    *
    * 2. Before running any task, the lock is acquired to prevent
    * other pool interrupts while the task is executing, and then we
    * ensure that unless pool is stopping, this thread does not have
    * its interrupt set.
    *
    * 3. Each task run is preceded by a call to beforeExecute, which
    * might throw an exception, in which case we cause thread to die
    * (breaking loop with completedAbruptly true) without processing
    * the task.
    *
    * 4. Assuming beforeExecute completes normally, we run the task,
    * gathering any of its thrown exceptions to send to afterExecute.
    * We separately handle RuntimeException, Error (both of which the
    * specs guarantee that we trap) and arbitrary Throwables.
    * Because we cannot rethrow Throwables within Runnable.run, we
    * wrap them within Errors on the way out (to the thread's
    * UncaughtExceptionHandler).  Any thrown exception also
    * conservatively causes thread to die.
    *
    * 5. After task.run completes, we call afterExecute, which may
    * also throw an exception, which will also cause thread to
    * die. According to JLS Sec 14.20, this exception is the one that
    * will be in effect even if task.run throws.
    *
    * The net effect of the exception mechanics is that afterExecute
    * and the thread's UncaughtExceptionHandler have as accurate
    * information as we can provide about any problems encountered by
    * user code.
    *
    * @param w the worker
    */
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    /*
    * 这里先unlock，有两个意思
    * 1. 其他线程如果正在执行，将state设置成0，aqs锁释放，这个时候就可以执行当前线程
    * 2. 第二个意思就是unlock后，可以中断线程，等到下面w.lock()获取到锁后，就不能中断线程了，具体看下shutdownNow方法？这里解释下，中断线程需要先获取到Worker锁，才能操作中断
    */
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) { // 如果task任务为null，是不会执行任务的，那就只能去取队列的任务
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                    (Thread.interrupted() &&
                    runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted()) 
                /**
                * 先解释一个疑问，wt是否和Thread.interrupted()中的线程是否是一个？有可能不是一个，可能情况下w.lock()在获取锁的时候一直没有获取成功，然后一直阻塞，到后续unlock释放锁后能获取到锁后才唤醒阻塞的线程
                * wt是当时保留的线程状态
                *
                * 如果线程池的状态>=STOP 或者 当前线程是中断状态并且线程池的状态>=STOP  并且 wt线程没有中断的话 就将wt给中断
                * 否则如果线程池是正常状态，或是当前线程没有中断过 就不会走wt.interrupt()
                */
                wt.interrupt(); // 将wt中断
            try {
                beforeExecute(wt, task); // 空的，还没有实现
                Throwable thrown = null;
                try {
                    task.run(); // 执行用户提交的线程任务方法
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown); // 空的，还没有实现
                }
            } finally {
                task = null;
                w.completedTasks++; // 任务数量+1
                w.unlock(); // 释放锁
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly); // 如果task.run()抛出了异常，catch捕获后，会走到这里，但是completedAbruptly是true
    }
}
```
> 引申：<font color='red'><b>`Thread.interrupted()`和`Thread.isInterrupted()`和`Thread.interrupt()`有什么区别？</b></font>
解释下：
1. `Thread.interrupted()`源码是`currentThread().isInterrupted(true)`，这个返回的是true或false，表示是否被中断，但是传参是true，就会清除掉中断的信号，那后面来的判断，是没有中断的
2. `Thread.isInterrupted()`源码是`isInterrupted(false)`，这个也表示是否被中断，但是传参是false，不会清楚掉中断的信号
3. `Thread.interrupt()`就是中断线程

### getTask
从队列里面取任务
```java
/**
    * Performs blocking or timed wait for a task, depending on
    * current configuration settings, or returns null if this worker
    * must exit because of any of:
    * 1. There are more than maximumPoolSize workers (due to
    *    a call to setMaximumPoolSize).
    * 2. The pool is stopped.
    * 3. The pool is shutdown and the queue is empty.
    * 4. This worker timed out waiting for a task, and timed-out
    *    workers are subject to termination (that is,
    *    {@code allowCoreThreadTimeOut || workerCount > corePoolSize})
    *    both before and after the timed wait, and if the queue is
    *    non-empty, this worker is not the last thread in the pool.
    *
    * @return task, or null if the worker must exit, in which case
    *         workerCount is decremented
    */
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) { // 线程池的状态如果是 Running状态 或者 ( SHUTDOWN状态 & 队列不为null )就不会走里面
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c); // 工作线程

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize; // allowCoreThreadTimeOut允许核心线程是否超时，默认是false

        /**
        * wc > maximumPoolSize 为true，有可能发生在setMaximumPoolSize方法时
        * (timed && timedOut) 表示超时控制，如果为true，表示当前操作需要进行超时控制，并且上次从阻塞队列中获取了任务超时了
        * 如果wc = 1，那就是表示线程池只有自己一个任务
        * workQueue或者线程池是空的
        */
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) { // 并发情况下考虑下timedOut参数，有可能为true
            if (compareAndDecrementWorkerCount(c)) // 就会将WorkCount减一，如果减一失败，重试
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take(); // 核心线程不会有超时的限制，如果allowCoreThreadTimeOut为false
            if (r != null)
                return r;
            timedOut = true; // 超时
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
```java
/**
    * Decrements the workerCount field of ctl. This is called only on
    * abrupt termination of a thread (see processWorkerExit). Other
    * decrements are performed within getTask.
    */
private void decrementWorkerCount() { // 一直减成功
    do {} while (! compareAndDecrementWorkerCount(ctl.get()));
}
```
![任务处理流程](/images/pasted-99.png)

### processWorkerExit
```java
/**
    * Performs cleanup and bookkeeping for a dying worker. Called
    * only from worker threads. Unless completedAbruptly is set,
    * assumes that workerCount has already been adjusted to account
    * for exit.  This method removes thread from worker set, and
    * possibly terminates the pool or replaces the worker if either
    * it exited due to user task exception or if fewer than
    * corePoolSize workers are running or queue is non-empty but
    * there are no workers.
    *
    * @param w the worker
    * @param completedAbruptly if the worker died due to user exception
    */
private void processWorkerExit(Worker w, boolean completedAbruptly) { // 工作线程退出
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount(); // 减少工作线程数量

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w); // 移除当前工作线程
    } finally {
        mainLock.unlock();
    }

    tryTerminate(); // 尝试Terminate

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) { // RUNNING || SHUTDOWN
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min) // 如果allowCoreThreadTimeOut为false，则工作线程至少必须是核心线程数，如果是true，队列不为空的情况下，至少至少保留一个，否则不用保留
                return; // replacement not needed
        }
        addWorker(null, false); // 异常情况下直接走这个，不会影响其他线程，加入当前null线程到addWorker
    }
}
```

### tryTerminate
尝试终止，这里有点疑问，怎么样算正常停止?
```java
/**
    * Transitions to TERMINATED state if either (SHUTDOWN and pool
    * and queue empty) or (STOP and pool empty).  If otherwise
    * eligible to terminate but workerCount is nonzero, interrupts an
    * idle worker to ensure that shutdown signals propagate. This
    * method must be called following any action that might make
    * termination possible -- reducing worker count or removing tasks
    * from the queue during shutdown. The method is non-private to
    * allow access from ScheduledThreadPoolExecutor.
    */
final void tryTerminate() { // 尝试终止
    for (;;) {
        int c = ctl.get();
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty())) // 运行状态 或者 已经是停止状态或准备停止状态 或者 SHUTDOWN并且队列不为空
            return;
        if (workerCountOf(c) != 0) { // Eligible to terminate
            interruptIdleWorkers(ONLY_ONE); // TODO 这里是什么意思？什么时候会减workCount的值
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) { // 修改为TIDYING
                try {
                    terminated();
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0)); // 修改为TERMINATED状态
                    termination.signalAll(); // 唤醒等待终止
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```
看完上面代码，一个正常状态的线程池是如何减少WorkCount，不然的话一直会执行这个代码，中断空闲线程，那怎么执行后面的停止状态代码
```java
if (workerCountOf(c) != 0) { // Eligible to terminate
            interruptIdleWorkers(ONLY_ONE); // TODO 这里是什么意思？什么时候会减workCount的值
            return;
}
```
答案在这里，在`getTask`代码中，当一个线程池的队列任务已经没有了，那空闲线程也是获取不到线程的，下面的代码是为true的，workCount就是减少到核心线程数
```java
// 这里是getTask的任务的逻辑
for(;;){
if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }
}
```
还有一个问题，什么时候减少核心线程数？<font color='red'><b>这里如果你注意的话，线程池如果没有调用shutdown的话，是不会停止的，一直在运行？不是，是一直在阻塞中，在哪里阻塞了，其实是核心线程数在阻塞，而且是在getTask的take的时候阻塞，现在队列数为0了，取不到线程，那什么时候唤醒，那就是把任务加到队列中就会唤醒，那核心线程数就是不会减少</b></font>
```java
// workQueue.take(); 核心线程就会执行这个，这个就会阻塞
try {
    Runnable r = timed ?
        workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
        workQueue.take();
    if (r != null)
        return r;
    timedOut = true;
} catch (InterruptedException retry) {
    timedOut = false;
}


// workQueue.offer 这里就是唤醒
if (isRunning(c) && workQueue.offer(command)) {
    int recheck = ctl.get();
    if (! isRunning(recheck) && remove(command))
        reject(command);
    else if (workerCountOf(recheck) == 0)
        addWorker(null, false);
}
```


### interruptIdleWorkers
```java
/**
    * Interrupts threads that might be waiting for tasks (as
    * indicated by not being locked) so they can check for
    * termination or configuration changes. Ignores
    * SecurityExceptions (in which case some threads may remain
    * uninterrupted).
    *
    * @param onlyOne If true, interrupt at most one worker. This is
    * called only from tryTerminate when termination is otherwise
    * enabled but there are still other workers.  In this case, at
    * most one waiting worker is interrupted to propagate shutdown
    * signals in case all threads are currently waiting.
    * Interrupting any arbitrary thread ensures that newly arriving
    * workers since shutdown began will also eventually exit.
    * To guarantee eventual termination, it suffices to always
    * interrupt only one idle worker, but shutdown() interrupts all
    * idle workers so that redundant workers exit promptly, not
    * waiting for a straggler task to finish.
    */
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}

```

```java
public boolean awaitTermination(long timeout, TimeUnit unit) // 等待终止，api接口，判断是否是终止的
        throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (;;) {
                if (runStateAtLeast(ctl.get(), TERMINATED))
                    return true;
                if (nanos <= 0)
                    return false;
                nanos = termination.awaitNanos(nanos); // 这里是有等待
            }
        } finally {
            mainLock.unlock();
        }
    }
```


### shutdownNow
```java
/**
     * Attempts to stop all actively executing tasks, halts the
     * processing of waiting tasks, and returns a list of the tasks
     * that were awaiting execution. These tasks are drained (removed)
     * from the task queue upon return from this method.
     *
     * <p>This method does not wait for actively executing tasks to
     * terminate.  Use {@link #awaitTermination awaitTermination} to
     * do that.
     *
     * <p>There are no guarantees beyond best-effort attempts to stop
     * processing actively executing tasks.  This implementation
     * cancels tasks via {@link Thread#interrupt}, so any task that
     * fails to respond to interrupts may never terminate.
     *
     * @throws SecurityException {@inheritDoc}
     */
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP); // 修改状态
            interruptWorkers();
            tasks = drainQueue(); // 只有shutdownNow的时候，废弃队列里的任务
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
```
```java
/**
    * Interrupts all threads, even if active. Ignores SecurityExceptions
    * (in which case some threads may remain uninterrupted).
    */
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock(); // 可重入的
    try {
        for (Worker w : workers)
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}
```
```java

/**
    * Drains the task queue into a new list, normally using
    * drainTo. But if the queue is a DelayQueue or any other kind of
    * queue for which poll or drainTo may fail to remove some
    * elements, it deletes them one by one.
    */
private List<Runnable> drainQueue() { // 废弃队列，只有shutdownNow()的时候才会废弃队列里面
    BlockingQueue<Runnable> q = workQueue;
    ArrayList<Runnable> taskList = new ArrayList<Runnable>();
    q.drainTo(taskList);
    if (!q.isEmpty()) {
        for (Runnable r : q.toArray(new Runnable[0])) {
            if (q.remove(r))
                taskList.add(r);
        }
    }
    return taskList;
}
```