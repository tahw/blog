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


# 三大特性

## 可见性

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

可见性解决方案
```java
	# 第一种方案
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
    
    # 第二种方案
	private static volatile boolean initFlag = false;
    
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
    
    # 第三种方案
	private static volatile boolean initFlag = false;
    
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
    
    # 第四种方案
	private static volatile boolean initFlag = false;

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
	private static volatile boolean initFlag = false;
    
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


## 原子性


## 有序性