title: synchronized
author: jianghe
tags:
  - 并发
categories:
  - 并发
date: 2021-05-19 09:02:00
---
# 如何解决并发问题
&nbsp;&nbsp;&nbsp;&nbsp;实际上，所有的并发模式在解决并发安全问题时，采用的方案都是序列化访问临界资源。即在同一时刻，只能有一个线程访问临界资源，也称作同步互质访问。
Java中提供了两种方式来实现同步互质访问：<b><font color='red'>synchronized和lock</font></b>。
同步器的本质就是加锁，加锁的目的：序列化访问临界资源。
> 当多个线程执行一个方法时，该方法的局部变量并不是临界资源，因为这些局部变量是在每个线程的私有栈中，因此不具备共享性，不会导致线程安全问题

```java
private static int total = 0;

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(1);

        for (int i = 0; i < 10; i++) {
            new Thread(()->{
                try {
                    countDownLatch.await();
                    for (int j = 0; j < 1000; j++) {
                        total++;
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
        Thread.sleep(200);
        countDownLatch.countDown();
        Thread.sleep(2000);
        System.out.println(total);
    }
```
这样的方式先采用`countDownLatch.await()`，然后再执行`countDownLatch.countDown()`，可以最大限度实现并发，这里的`CountDownLatch`为1，结果肯定是小于10000，解决方式可以采用`synchronized`和`lock`

<!-- more -->

# 锁发展历史
1. `jdk<1.6`是jdk自带的，这是第一代加锁方式，但是这种锁非常的重，一旦加锁，都是采用os实现的加锁方式，这里就会发生上下文切换，空间切换，非常缓慢。
2. doug li看`synchronized`太慢了，自己开发了一个并发包，juc，性能非常好，纯java写的，里面还实现了jdk没有的特性，虽然底层还是调用了一个java内置的函数。
3. oracle收购sun公司后，优化了synchronized，锁的膨胀升级，锁粗化，锁消除，现在的性能和aqs差不多。

![锁的升级历史](/images/pasted-43.png)


# synchronized原理详解
&nbsp;&nbsp;&nbsp;&nbsp;这里主要还是聊`jdk>=1/6`后的`synchronized`。
&nbsp;&nbsp;&nbsp;&nbsp;`synchronized`内置锁是一种对象锁（锁的是对象而非引用），作用粒度是对象，可以用来实现对临界资源的访问，是可重入的。
```java
public synchronized void a() 锁的是这个实例对象
public static synchronized void a() 锁的是这个类对象
public void a(){ 锁的是括号里的这个对象
	synchronized(Object.class)
} 
```
## synchronized底层原理
&nbsp;&nbsp;&nbsp;&nbsp;synchronized是基于<b><font color='red'>JVM内置锁实现，通过内部对象Monitor（监视器锁）实现。基于进入与退出Monitor对象实现方法与代码块同步，监视器锁的实现依赖操作系统的Mutex Lock（互斥锁）实现，它是一个重量级锁性能低。（在>=jdk1.6，实现重量级锁是monitor锁，轻量级锁、偏向锁不是使用monitor锁）</font></b>当然JVM内置锁在jdk>=1.6后做了升级优化，锁粗化（Lock Coarsening）、锁消除（Lock Elimination）、轻量级锁（LightWeight Locking）、偏向锁（Biased Locking）、适应性自旋锁（adaptive Spinning）等技术减少锁的开销。
> jdk>=1.6，偏向锁、轻量级锁的实现是通过对象头的mark word锁标识和lock record来实现。那为什么一个线程的锁编译还是有monitorenter和monitorexit呢？其实是编译器加的https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-3.html
The compiler ensures that at any method invocation completion, a monitorexit instruction will have been executed for each monitorenter instruction executed since the method invocation.


## monitor监视器锁
&nbsp;&nbsp;&nbsp;&nbsp;任何一个对象都有一个`monitor`与之关联，并且一个`monitor`被持有后，它将处于锁定状态。
synchronized在jvm里的实现都是基于进入和退出monitor对象来实现方法的同步与代码块同步。虽然具体实现细节不一样，但是都可以成对的`monitorenter`和`monitorexit`指令来实现。
* monitorenter
	* 当monitorenter为0，如果线程进入monitor，则进入数加1，该线程持有了monitor
    * 如果这个线程持有了monitor，尝试再进入monitor（例如方法调用方法），则monitor的进入数加1
    * 那其他线程已经占有的monitor，则其他线程会阻塞状态，直至monitor的进入数为0，再重新尝试获取monitor的所有权
* monitorexit
	* 执行monitorexit的线程必须是objectref所对应的monitor的所有者，指令执行时，monitor的进入数减1，如果减1为0，那线程退出monitor，不再是这个monitor的持有者了。

下面这个例子，其中有一个`monitorenter`和两个`monitorexit`，两个中的一个是异常情况下的退出
```java
	private Object object = new Object();
    public void sync(){
        synchronized (object){
            System.out.println(111);
        }
    }

#对应的字节码
public void sync();
    Code:
       0: aload_0
       1: getfield      #3                  // Field object:Ljava/lang/Object;
       4: dup
       5: astore_1
       6: monitorenter
       7: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
      10: bipush        111
      12: invokevirtual #5                  // Method java/io/PrintStream.println:(I)V
      15: aload_1
      16: monitorexit
      17: goto          25
      20: astore_2
      21: aload_1
      22: monitorexit
      23: aload_2
      24: athrow
      25: return
```

下面这个例子是修饰在方法上，看字节码，里面的flags增加了`ACC_SYNCHRONIZED`，虽然没有加上monitor对象，但是jvm也能根据这个标识来实现方法的同步，当调用指令将会检查方法的`ACC_SYNCHRONIZED`访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完了再释放monitor。
```
 public synchronized void syncO(){
        System.out.println(111);
    }

# 对应字节码
public synchronized void syncO();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: bipush        111
         5: invokevirtual #5                  // Method java/io/PrintStream.println:(I)V
         8: return
      LineNumberTable:
        line 14: 0
        line 15: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Ljuc/juc04/SynchronizedDemo;            
```
<b><font color='red'>两种同步方法本质上没有区别，只是方法的同步是一种隐式的方法来实现，无需通过字节码来完成。两个指令的执行是jvm通过调用操作系统的互斥源于mutex来实现，被阻塞的线程会被挂起，等待重新调度，会导致”用户态和内核态“两个态切换。</font></b>

> Synchronized的语义底层是通过一个monitor的对象来完成，其实wait/notify等方法也依赖于monitor对象，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则 会抛出java.lang.IllegalMonitorStateException的异常的原因。

## 什么是monitor
&nbsp;&nbsp;&nbsp;&nbsp;`monitor`可以把它理解为一个同步工具，也可以描述为一个同步机制，它通常被描述一个对象。<b><font color='red'>所有的java对象都是monitor，每一个java对象都有成为monitor的潜质。因为在java设计中，每一个java对象都带一把看不见的锁，它叫做内部锁或者monitor锁。也就是synchronized的对象锁。MarkWord锁标志位为10，其中指针指向monitor对象的起始地址。</font></b>

在jdk中Monitor由ObjectMonitor.hpp文件实现的（c++实现）。
```c++
 ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL; // 线程拥有者
    _WaitSet      = NULL; // 等待wait状态的线程，会被加入到这里
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ;// 处于等待锁block状态的线程，会被加入到这里
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
    _previous_owner_tid = 0;
  }

```
`ObjectMonitor`中有两个队列，`_WaitSet`和`_EntryList`，用来保存ObjectWaiter对象列表（每个等待锁的线程都会被封装成ObjectWaiter对象），`_owner`指向持有`ObjectMonitor`对象的线程，多个线程访问一个同步代码时：
1. 首先会进入`_EntryList`，当线程获取到对象的monitor后，进入`_owner`区域并把monitor的`_owner`为当前线程，同时`_count`计数器会加1
2. 若线程调用wait()方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入waitSet集合中等待被唤醒
3. 若当前线程执行完毕，也将释放monitor并复位count的值，以便其他线程进入获取monitor。

> monitor对象存在于每个java对象的对象头mark word中（存储的指针的指向），synchronized锁便是通过这种方式获取锁的，也是为什么java中任意对象可以作为锁的原因，同时notify、notifyAll、wait等方法会使用monitor对象，所以必须在同步代码块中使用。

## 对象的内存布局

&nbsp;&nbsp;&nbsp;&nbsp;当成功对分配的内存空间进行零值初始化后，jvm就会对对象进行实例化。在hotSpot中，对象实例化操作无非就是初始化<b><font color='red'>对象头</font></b>和<b><font color='red'>实例数据</font></b>，而且存储对象实例信息的内存布局也主要由这两个部分组成。
* 对象头
	* 主要存储Mark Word和元数据指针等数据，其中Mark Word主要用于存储对象运行时的数据信息，比如HashCode、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。元数据指针则是用于指向方法区中目标类的类型信息，也就是说可以通过元数据指针可以准确定位到当前对象的具体目标类型
* 实例数据
	* 用于存储定义在当前对象中各种类型的字段信息（包括派生于超类的字段信息）
* 对齐填充
	* 由于虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐

![对象内存布局](/images/pasted-44.png)


### Mark Word
&nbsp;&nbsp;&nbsp;&nbsp;现在我们虚拟机基本是64位的，而64位的对象头有点浪费空间,JVM默认会开启指针压缩，所以基本上也是按32位的形式记录对象头的，手动设置`‐XX:+UseCompressedOops`

![32位Mark Word](/images/pasted-46.png)

![64位Mark Word](/images/pasted-45.png)

mark word，有方法可以去看吗？
```xml
# 1. 引入下面pom
<dependency>
  <groupId>org.openjdk.jol</groupId>
  <artifactId>jol-core</artifactId>
  <version>0.15</version>
</dependency>

# 2.打印对象头，object是锁对象
System.out.println(ClassLayout.parseInstance(object).toPrintable());
```

### 锁的膨胀升级过程
&nbsp;&nbsp;&nbsp;&nbsp;锁的状态总共有四种，无锁状态、偏向锁、轻量级锁、重量级锁。锁可以从偏向锁升级到轻量级锁，再升级到重量级锁，锁的升级是单向的，只能从低到高，不会出现锁的降级。
![锁的膨胀升级](/images/pasted-47.png)

#### 偏向锁
&nbsp;&nbsp;&nbsp;&nbsp;偏向锁是在jdk1.6引入的新锁，它是一种针对加锁的优化手段，经过研究发现，在大多情况下，锁不仅不存在多线程竞争，而且总是有同一线程多次获得，因此为了减少同一线程获取锁（会涉及一些cas操作，耗时）的代价而引入偏向锁。偏向锁的核心思想是，如果一个线程获得了锁，那么锁就进入偏向模式，此时Mark Word的结构也变为偏向锁的结构，当这个线程再次请求锁时，无需再做任何同步的操作，从而节省锁的申请过程，提供了性能。所以，对于没有锁竞争的场合，偏向锁有很好的效果，毕竟极有可能连续多次是同一线程申请相同的锁。但是对于锁竞争比较激烈的场合，偏向锁就失效了，因为这个场合极有可能申请锁的线程是不相同的，因此使用偏向锁就得不偿失，<b><font color='red'>需要注意的是，偏向锁失败后，并不会直接升级重量级锁，而是先升级轻量级锁。</font></b>

```
默认开启偏向锁 
开启偏向锁：-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0 
关闭偏向锁：-XX:-UseBiasedLocking
```
下面是偏向锁的例子
```java
		Thread.sleep(5000);
        Object o = new Object();
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
        synchronized (o){
            System.out.println(ClassLayout.parseInstance(o).toPrintable());
        }
```
打印Mark Word
```
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007fd8f000a805 (biased: 0x0000001ff63c002a; epoch: 0; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```
其中，java虚拟机开启偏向锁是懒加载的方式，默认需要等待4s的时间，因为java虚拟机在启动的时候，里面有多个线程会去加载，会存在竞争，如果打开偏向锁，多个线程竞争还需要锁升级，这里要演示偏向锁就需要等待4s多，其中`biasable`表示无锁状态，但是可偏向，表示能偏向线程，mark word 为`101`，但是mark word 前面字节都是0，但是还没有偏向。`biased`表示偏向锁。

#### 轻量级锁
&nbsp;&nbsp;&nbsp;&nbsp;偏向锁失败后，虚拟机并不会立马升级为重量级锁，它还会尝试使用一种轻量级锁的优化手段，此时Mark Word的结构也变为轻量级锁的结构。轻量级锁能够提升程序性能的依据是“对绝大部分的锁，在整个同步周期内都不存在竞争”，注意这是经验数据。轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间访问同一个锁的场景，就会导致轻量级锁升级为重量级锁。

下面是轻量级锁的例子
```java
		Thread.sleep(5000);
        final Object o = new Object();
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
        new Thread(new Runnable() {
            public void run() {
                synchronized (o){
                    System.out.println(ClassLayout.parseInstance(o).toPrintable());
                }
            }
        }).start();
        // 等待的时间如果长的话，接下来mark word不一定是轻量级锁，如果等待3s有可能是偏向锁，看资源已经释放
        Thread.sleep(30); 
        new Thread(new Runnable() {
            public void run() {
                synchronized (o){
                    System.out.println(ClassLayout.parseInstance(o).toPrintable());
                }
            }
        }).start();
```
打印Mark Word
```
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007f81460aa805 (biased: 0x0000001fe05182aa; epoch: 0; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000700003528960 (thin lock: 0x0000700003528960)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

```

还有一个例子，是调用对象的hashCode会导致偏向锁升级为轻量级锁
```
Thread.sleep(5000);
        Object o = new Object();
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
        synchronized (o){
            System.out.println(ClassLayout.parseInstance(o).toPrintable());
        }
        System.out.println(o.hashCode()); // 会将偏向锁升级为轻量级锁
        System.out.println(ClassLayout.parseInstance(o).toPrintable()); //可重偏向
        synchronized (o){
            System.out.println(ClassLayout.parseInstance(o).toPrintable()); // 轻量级锁
        }
```
打印的Mark Word
```
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007ffeda809005 (biased: 0x0000001fffb6a024; epoch: 0; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

607635164
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000002437c6dc01 (hash: 0x2437c6dc; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000070000c4d2988 (thin lock: 0x000070000c4d2988)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

```
那打印的`System.out.println(o.hashCode());`不是锁降为无锁状态，而是此时作为没有synchronized修饰对象，获取的此时对象的头。
那这里为什么偏向锁调用hashCode会升级偏向锁呢？由于偏向锁的时候Mark Word里是没有存储hashCode，而轻量级锁里面是有hashCode的，hashCode是锁对象里的Mark Word的指针指向栈中lock record中的mark word的hashCode，详细见锁的膨胀升级。那这是直接调用底层生成的hashcode，还是使用的lock record的hashcode，这块是不确定的。

#### 自旋锁
&nbsp;&nbsp;&nbsp;&nbsp;轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的手段。这是基于在大多情况下，线程持有锁的时间都不太长，如果直接挂起操作系统层面的线程可能会得不偿失，毕竟操作系统实现线程之间的切换时需要从用户态切换到核心态，这个状态切换是需要时间的，成本比较高。因此自旋锁会假设在不久将来，当前的线程可以获取锁，因此虚拟机会让当前想要获取锁的线程做几个空循环（这也是成为自旋的原因），一般不会太久，可能是50个循环或100循环，在经过若干次循环后，如果得到锁，就顺利进入临界区。如果还不能获取锁，那就会将线程在操作系统层面挂起，这就是自旋锁的优化。否则就升级为重量级锁

#### 锁消除
&nbsp;&nbsp;&nbsp;&nbsp;锁消除是虚拟机的另外一种锁的优化。这种优化更彻底，java虚拟机在Jit编译时（可以简单理解为当某段代码即将第一次被执行时进行编译，又称及时编译），通过对上下文的扫描，去除不可能存在共享资源竞争的锁，通过这种方式消除没有必要的锁，可以节省毫无意义的请求锁时间。所消除的依据是逃逸分析的数据支持。
> 如StringBuffer的append是一个同步方法，但是在add方法中的StringBuffer属于一个局部变量，并且不会被其他线程所使用，因此StringBuffer不可能存在共享资源竞争的情景，JVM会自动将其锁消除。
```java
public void sync(){
	Object obj = new Object();
    synchronized(obj){
    	// xxx
    }
}
```
这里的synchronized可以去掉，锁的对象是方法内部，不会存在竞争
> 锁消除，前提是java必须运行在server模式，同时必须开启逃逸分析
:-XX:+DoEscapeAnalysis 开启逃逸分析
-XX:+EliminateLocks 表示开启锁消除。

#### 锁粗化
&nbsp;&nbsp;&nbsp;&nbsp;按理来说，同步块的作用范围应该尽可能小，仅在共享数据的实际作用域中才进行同步，这样做的目的是为了使需要同步的操作数量尽可能缩小，缩短阻塞时间，如果存在锁竞争，那么等待锁的线程也能尽快拿到锁。但是加锁解锁也需要消耗资源，如果存在一系列的连续加锁解锁操作，可能会导致不必要的性能损耗。锁粗化就是将多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁，避免频繁的加锁解锁操作。
```java
public void sync(){
    synchronized(Object.class){
    	// aaa
    }
    synchronized(Object.class){
    	// bbb
    }
    synchronized(Object.class){
    	// ccc
    }
}
```
可以优化为
```java
public void sync(){
    synchronized(Object.class){
    	// aaa
        // bbb
        // ccc
    }
}
```




