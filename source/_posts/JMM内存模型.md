title: JMM内存模型
author: jianghe
tags:
  - 并发
categories:
  - 并发
date: 2021-04-22 21:01:00
---
# 什么是JMM?
&nbsp;&nbsp;&nbsp;&nbsp;Java 内存模型（Java Mermory Model）是一个抽象的概念，并不真实存在。它描述的是一种规范和准则，通过这组规范定义了程序中各个变量的访问方式。</b><font color='red'>它是描述的线程模型。</font></b>JVM运行程序的实体是线程，而每个线程创建时JVM都会为其创建一个工作内存，用户存储线程私有的数据，而Java内存模型中规定所有变量都存储在主内存，主内存是共享区域，所有线程都可以访问，但线程对变量的操作必须在工作内存中，其中首先会从主内存中变量复制到工作内存中，然后对变量进行操作后写会到主内存中。不同的工作线程无法通信，只能通过主内存来进行通信。

![java memory model](/images/pasted-29.png)

<!-- more -->

## 主内存
&nbsp;&nbsp;&nbsp;&nbsp;主要存储的是java对象，所有线程创建的实例对象都存放在主内存，不管该实例对象是成员变量还是局部变量，还包括类信息、常量、静态变量。由于是共享区域，会发生线程安全问题。

## 工作内存
&nbsp;&nbsp;&nbsp;&nbsp;主要存储当前方法的所有本地方法变量，存储的主内存的变量副本，每个线程只能访问自己的工作内存，线程间无法通信，工作内存不存在线程安全问题。


# JMM和jvm内存区域模型关系
&nbsp;&nbsp;&nbsp;&nbsp;提起jvm内存模型，那就肯定有堆、栈、本地方法栈、方法区、程序计数器这些，他和jmm有相似之处，都有共享区域和私有区域，不同的地方是，jmm是描述的一个规范，通过这种规范来控制变量在共享区域和私有区域的访问方式，jmm围绕的还是原子性、可见性、有序性。

# JMM和硬件内存架构关系

&nbsp;&nbsp;&nbsp;&nbsp;从前面可以了解到，多线程的执行还是会映射到硬件上，但是jmm和硬件内存架构并不完全一致。硬件内存架构只有寄存器、缓存内存、主内存的概念，并没有工作内存和主内存。对于硬件内存架构中数据存储在主内存、寄存器中。java内存模型和计算机硬件内存架构是一个相互交叉的关系，是一种抽象和真实物理硬件的交叉。

![jmm->硬件](/images/pasted-30.png)


# JMM存在必要性

&nbsp;&nbsp;&nbsp;&nbsp;从下面这个问题，线程B读取的值到底是1还是2，工作内存读取主内存变量实现细节，jmm定义了八种操作来完成，来解决这类问题。
![jmm问题](/images/pasted-31.png)

## 数据同步八大原子操作

1. lock（锁定）：作用于主内存的变量，把一个变量标记为一条线程独占状态
2. unlock（解锁）：作用于主内存的变量，把一个变量处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定
3. read（读取）：作用于主内存的变量，把一个变量值从主内存传输到线程的工作内存中，以便随后的load动作使用
4. load（载入）：作用于工作内存的变量，他把read操作从主内存中得到的变量值放入工作内存的变量副本中
5. use（使用）：作用于工作内存的变量，把工作内存中的一个变量值传递给执行引擎
6. assign（赋值）：作用于工作内存的变量，他把一个从执行引擎执行后的变量赋值给工作内存的变量
7. store（存储）：作用于工作内存的变量，把工作内存中的一个变量的值传递给主内存中，以便随后的write的操作
8. write（写入）：作用于工作内存的变量，它把store操作从工作内存中的一个变量的值传送到主内存的变量中

![数据同步过程](/images/pasted-32.png)

## 同步规则分析
1. 不允许一个线程无原因地把数据（没有经过任何assign操作）从工作内存同步到主内存中
2. 一个新的变量只能在主内存中诞生，不允许在工作内存中直接使用一个未被初始化变量。
3. 一个变量在同一时刻只允许一条线程对其进行lock操作，但lock操作可以被同一线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，才有可能被其他线程给获取。lock和unlock必须成对出现
4. 如果对一个变量执行lock操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量之前需要重新执行load或assign操作初始化变量的值
5. 如果一个变量事先没有被lock操作锁定，则不允许对它执行unlock操作；也不允许去 unlock一个被其他线程锁定的变量。
6. 对一个变量执行unlock操作之前，必须先把此变量同步到主内存中（执行store和write 操作）


# 三大特性问题

## 原子性问题
&nbsp;&nbsp;&nbsp;&nbsp;原子性指的是一个操作是不可中断的，即使在多线程的环境下，一个操作一旦开始就不会被其他线程影响。在并发情况下可以使用synchronized关键字来解决原子性,
后面会着重讲解<font color='red'><b>`synchronized`和`lock`</b></font>
```
# 
private static int count = 0;

    static Object o = new Object();

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 10; i++) {
            new Thread(new Runnable() {
                public void run() {
                    for (int j = 0; j <1000; j++) {
                        synchronized (o) { //一个线程只能进入一次
                            count++; //不会发生指令重排
                        }
                    }
                }
            }).start();
        }
        Thread.sleep(200);
        System.out.println(count);
    }
```

## 可见性问题

下面这个会出现可见性问题
```java
	private static boolean initFlag = false;

    public static void main(String[] args) throws InterruptedException {
        new Thread(new Runnable() {
            public void run() {
                System.out.println("thread 1 start");
                while (!initFlag){
                }
                System.out.println("thread 1 stop");
            }
        }).start();
        Thread.sleep(200); // 休眠
        new Thread(new Runnable() {
            public void run() {
                System.out.println("thread 2 start");
                initFlag = true;
                System.out.println("thread 2 stop");
            }
        }).start();
    }
```

可见性解决方案，后面会着重讲解<font color='red'><b>`volatile`和`synchronized`</b></font>

```java
	# 第一种方案（volatile可以解决可见性问题，通过MESI来控制）
	private static volatile boolean initFlag = false;

    public static void main(String[] args) throws InterruptedException {
        new Thread(new Runnable() {
            public void run() {
                System.out.println("thread 1 start");
                while (!initFlag){
                }
                System.out.println("thread 1 stop");
            }
        }).start();
        Thread.sleep(200); // 休眠
        new Thread(new Runnable() {
            public void run() {
                System.out.println("thread 2 start");
                initFlag = true;
                System.out.println("thread 2 stop");
            }
        }).start();
    }
    
    # 第二种方案（原因有可能是i和initFlag在一个cacheline上面，导致获取i的时候initFlag也读取到最新的）
	private static boolean initFlag = false;
    
    // 如果换成int类型就不可以了
    private static Integer i = 0; 

    public static void main(String[] args) throws InterruptedException {
        new Thread(new Runnable() {
            public void run() {
                System.out.println("thread 1 start");
                while (!initFlag){
                	i++;
                }
                System.out.println("thread 1 stop");
            }
        }).start();
        Thread.sleep(200); // 休眠
        new Thread(new Runnable() {
            public void run() {
                System.out.println("thread 2 start");
                initFlag = true;
                System.out.println("thread 2 stop");
            }
        }).start();
    }
    
    # 第三种方案（原因有可能是i和initFlag在一个cacheline上面，导致获取i的时候initFlag也读取到最新的）
	private static boolean initFlag = false;
    
    private static volatile int i = 0;

    public static void main(String[] args) throws InterruptedException {
        new Thread(new Runnable() {
            public void run() {
                System.out.println("thread 1 start");
                while (!initFlag){
                	i++;
                }
                System.out.println("thread 1 stop");
            }
        }).start();
        Thread.sleep(200); // 休眠
        new Thread(new Runnable() {
            public void run() {
                System.out.println("thread 2 start");
                initFlag = true;
                System.out.println("thread 2 stop");
            }
        }).start();
    }
    
    # 第四种方案（为什么加上sout就会可以，因为空循环的优先级非常高，一旦获取cpu，很难切换时间片的）
	private static boolean initFlag = false;

    public static void main(String[] args) throws InterruptedException {
        new Thread(new Runnable() {
            public void run() {
                System.out.println("thread 1 start");
                while (!initFlag){
					System.out.println(111);
                }
                System.out.println("thread 1 stop");
            }
        }).start();
        Thread.sleep(200); // 休眠
        new Thread(new Runnable() {
            public void run() {
                System.out.println("thread 2 start");
                initFlag = true;
                System.out.println("thread 2 stop");
            }
        }).start();
    }
    
    # 第五种方案（jvm 增加-Djava.compiler=NONE 这是关掉jvm jit编译，jit编译优化是：编译过一次，下次再执行的时候就不用再次编译了，类似于for循环，就非常好）
	private static boolean initFlag = false;
    
    // 注意这里是int类型
    private static int i = 0; 

    public static void main(String[] args) throws InterruptedException {
        new Thread(new Runnable() {
            public void run() {
                System.out.println("thread 1 start");
                while (!initFlag){
                	i++;
                }
                System.out.println("thread 1 stop");
            }
        }).start();
        Thread.sleep(200); // 休眠
        new Thread(new Runnable() {
            public void run() {
                System.out.println("thread 2 start");
                initFlag = true;
                System.out.println("thread 2 stop");
            }
        }).start();
    }
```

## 有序性问题

```java
private static Integer x, y = 0;
    private static Integer a, b = 0;

    public static void main(String[] args) throws InterruptedException {
        int i=0;
        for (;;){
            i++;
            x=0;y=0;
            a=0;b=0;
            Thread t1 = new Thread(new Runnable() {
                public void run() {
                    // 等待
                    sleep();
                    a = 1;
                    x = b;
                }
            });
            Thread t2 = new Thread(new Runnable() {
                public void run() {
                    // 等待
                    sleep();
                    b = 1;
                    y = a;
                }
            });
            t1.start();
            t2.start();
            t1.join();
            t2.join();
            if(x ==0 && y ==0){
                System.out.println("第"+i+"次"+",x="+x+",y="+y);
                break;
            }else{
                System.out.println("第"+i+"次"+",x="+x+",y="+y);
            }
        }
    }

    private static void sleep() {
        try {
            Thread.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```
当x和y都是0的时候就发生指令重排，可以通过<font color='red'><b>`volatile`</b></font>来解决。
![分析](/images/pasted-33.png)

### happens-before原则
&nbsp;&nbsp;&nbsp;&nbsp;每个线程都有自己的工作内存。每个线程对变量的操作都必须在工作内存中进行，而不是直接对主内存进行操作。并且每个线程不能访问其他线程的工作内存。在java内存模型中存在一些先天的“有序性”，即不需要通过任何手段就能够得到程序的有序性。这个就是happens-before原则。如果两个操作无法从happens-before原则推导出来，那么他们就不能保证有序性。
那happens-before原则原则就是如下：
1. 程序顺序原则，即在一个线程内必须保证语义串行性，也就是说按照代码顺序执行。
2. 锁规则 解锁(unlock)操作必然发生在后续的同一个锁的加锁(lock)之前，也就是说，如果对于一个锁解锁后，再加锁，那么加锁的动作必须在解锁动作之后(同一个 锁)。 
3. volatile规则 volatile变量的写，先发生于读，这保证了volatile变量的可见性，简 单的理解就是，volatile变量在每次被线程访问时，都强迫从主内存中读该变量的 值，而当该变量发生变化时，又会强迫将最新的值刷新到主内存，任何时刻，不同的 线程总是能够看到该变量的最新值。 4. 线程启动规则 线程的start()方法先于它的每一个动作，即如果线程A在执行线程B 的start方法之前修改了共享变量的值，那么当线程B执行start方法时，线程A对共享 变量的修改对线程B可见 
5. 传递性 A先于B ，B先于C 那么A必然先于C 
6. 线程终止规则 线程的所有操作先于线程的终结，Thread.join()方法的作用是等待 当前执行的线程终止。假设在线程B终止之前，修改了共享变量，线程A从线程B的 join方法成功返回后，线程B对共享变量的修改将对线程A可见。 
7. 线程中断规则 对线程 interrupt()方法的调用先行发生于被中断线程的代码检测到 中断事件的发生，可以通过Thread.interrupted()方法检测线程是否中断。 
8. 对象终结规则对象的构造函数执行，结束先于finalize()方法

### 指令重排
&nbsp;&nbsp;&nbsp;&nbsp;java语言规范规定jvm线程内部维持顺序化语义。即只要程序的最终结果与它顺序的结果相等，那么指令的执行顺序可以与代码顺序不一致，此过程叫指令的重排序。指令重排的意义在哪？jvm能够根据处理器的特性（CPU多级缓存、多核处理器等）适当对机器指令进行重排序，是机器指令能更符合CPU的执行特性，最大限度发挥机器性能。

![指令优化](/images/pasted-36.png)


### as-if-serial语义
不管怎么重排序（编译器和处理器为了提高并行度），（单线程下）程序的执行结果不能被改变。编译器，runtime 和处理器都必须遵守as-if-serial语义。

## volatile语义
&nbsp;&nbsp;&nbsp;&nbsp;volatile是java虚拟机提供的轻量级的同步机制。有两个作用：
1. 可见性，一个被volatile变量修改会被其他线程感知到
2. 有序性，通过内存屏障，禁止指令重排
3. 无法保证原子性，参照下面的代码，并发情况下，执行incr方法会线程安全问题，可以通过`synchronized`解决。其中`synchronized`本身也具有可见性，故不用`volatile`修饰

```
private static volatile Interger counter = 0

public static void incr(){
	counter++; // 不会发生指令重排，但是不是原子性
}
```

> 这里引申下，dcl（双重检测锁）在并发情况下，会有线程问题，细节不过多展示，那这里需要加入`volatile`，这里`volatile`和`synchronized`都必须在，因为不加`volatile`,这里`instance = new Instance()`会发生指令重排，会出现线程问题

```
private static Instance instance;

public static Instance getInstance(){
	if(instance == null){
    	synchronized(Instance.class){
        	if(instance == null){
            	instance = new Instance();
            }
        }
    }
}
```

### volatile禁止重排优化
&nbsp;&nbsp;&nbsp;&nbsp;要说明`volatile`禁止重排，这里就的引入一个名词<b><font color='red'>内存屏障（memory barrier）</font></b>。
#### 内存屏障
是一个cpu指令。jvm有四个内存屏障指令。

|屏蔽类型| 指令实例 | 说明 |
| :-----| :---- | :---- |
| LoadLoad | Load1;LoadLoad;Load2 | 保证Load1的读取操作在Load2之前 |
| StoreStore | Store1;StoreStore;Store2 | 在Store2及其后的写操作执行前，保证Store1的写操作已刷新到主内存 |
| LoadStore | Load1;LoadStore;Store2; | 在Store2及其后的写操作执行前，保证Load1的读操作是正确数据 |
| StoreLoad | Store1;StoreLoad;Load2 | 保证Store1的写操作已刷新内存，Load2读取数据是正确数据 |

##### 编译器屏障（Compiler Barrior）
阻止编译器重排，保证编译程序时在优化屏障之前的指令不会在优化屏障之后执行。

```
/* The "volatile" is due to gcc bugs */
#define barrier() __asm__ __volatile__("": : :"memory") 

```

##### CPU屏障（Cpu Barrior）
1. 作用：
	1. 防止指令重排序
	2. 保证数据可见性
2. 分类：
	1. lfence，是一种Load Barrior 读屏障，会将invalidate queue失效，强制读取L1 Cache中，而且lfence之后的读操作不会被调度到之前，即lfence的读操作一定在lfence完成（并未规定全局可见性）；
	2. sfence，是一种save Barrior 写屏障，会将store buffer中缓存的修改刷入L1 Cache中，使得其他cpu核可以观察到这些变化，而且之后的写操作不会被调度到之前，即sfence之前的写操作一定在sfence完成且全局可见
	3. mfence，是一种全能型屏障，具备lfence和sfence的能力，同时刷新store buffer和invalidate queue，保证了mfence前后的读写操作的顺序，同时要求mfence之后写操作结果全局可见之前，mfence之前写操作全局可见
    
> cpu屏障也包含指令<b>lock前缀</b>，lock不是一种内存屏障，但是它能完成类似内存屏障的功能。lock会对cpu总线和高速缓存加锁，可以理解为cpu指令级的一种锁。会让指令操作原子化，而且自带mfence效果。

&nbsp;&nbsp;&nbsp;&nbsp;<b>X86-64一般情况根本不会需要使用lfence与sfence这两个指令，除非操作Write-Through内存或使用 non-temporal 指令（NT指令，属于SSE指令集），比如movntdq, movnti, maskmovq，这些指令也使用Write-Through内存策略，通常使用在图形学或视频处理，Linux编程里就需要使用GNC提供的专门的函数，下面是GNU中的三种内存屏障定义方法，下面是结合编译器屏障和CPU指令屏障。
```
#define lfence() __asm__ __volatile__("lfence": : :"memory") 
#define sfence() __asm__ __volatile__("sfence": : :"memory") 
#define mfence() __asm__ __volatile__("mfence": : :"memory") 
```
&nbsp;&nbsp;&nbsp;&nbsp;代码中仍然使用lfence()与sfence()这两个内存屏障应该也是一种长远的考虑。按照Interface写代码是最保险的，万一Intel以后出一个采用弱一致模型的CPU，遗留代码出问题就不好了。目前在X86下面视为编译器屏障即可。</b>

```java
# java通过魔术类来加内存屏障
private static final Unsafe unsafe = Unsafe.getUnsafe();
unsafe.loadFence();
unsafe.storeFence();
unsafe.fullFence();
```

> 下面这里有一个典型的使用内存屏障的例子（DCL）
```
private static A a= null;

    public static A getInstance(){
        if(a == null){
            synchronized (双重检查锁.class){
                a = new A(); // 多线程可能会出现问题的地方
            }
        }
        return a;
    }

```
> 为什么在多线程的场景下a = new A();会出现问题？
其实这里不是原子操作，有分为下面三步；
> 1. 分配对象内存空间 memory = allocate();
> 2. 初始化对象	 instance(memory);
> 3. 设置instance的对象指向刚分配的内存空间instance = memory;

> <font color='red'><b>由于第二步和第三步没有依赖关系，是可以重排的。重排后再单线程结果是没有改变的，所有这种重排是可以允许的。指令重排只会保证串行语义的执行的一致性(单线程)，但并不会关心多线程间的语义一致性。所以当一个线程访问instance对象不为null时，虽然是不为空的，但是执行方法时就会报错，对象没有实例化完成。解决重排通过加上`volatile`就可以解决。</b></font>

#### volatile内存语义的实现

编译器制定的volatile重排序规则表

|  | 第二个操作：普通读写 | 第二个操作：volatile读 | <font color='red'>第二个操作：volatile写</font> |
| :-----:| :----: | :----: | :----: |
| 第一个操作：普通读写 | 可以 | 可以 | 不可以 |
| <font color='red'><b>第一个操作：volatile读</b></font> | 不可以 | 不可以 | 不可以 |
| 第一个操作：volatile写 | 可以 | 不可以 | 不可以 |
* 其中第一个操作：volatile读，都不可以重排
* 其中第二个操作：volatile写，都不可以重排
* 其中第一个操作：volatile写，第二个操作：volatile读，都不可以重排

&nbsp;&nbsp;&nbsp;&nbsp;为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎不可能。为此，JMM采取保守策略。下面是基于保守策略的JMM内存屏障插入策 略。
* 在每个volatile写操作的前面插入一个StoreStore屏障。
* 在每个volatile写操作的后面插入一个StoreLoad屏障。 
* 在每个volatile读操作的后面插入一个LoadLoad屏障
* 在每个volatile读操作的后面插入一个LoadStore屏障

这里举一个来说明
```
 int a;
    volatile int v1 = 1;
    volatile int v2 = 2;

    void readAndWrite(){
        int i = v1; // 第一个volatile读
        int j = v2; // 第二个volatile读
        a = i+j;    // 普通写
        v1 = i + 1; // 第一个volatile写
        v2 = j * 2; // 第二个volatile写
    }
```
针对readAndWrite方法，编译器在字节码会做如下的优化，见下面的解释就可以会很清楚了。
![内存屏障优化](/images/pasted-38.png)