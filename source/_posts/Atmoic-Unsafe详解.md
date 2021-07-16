---
title: Atmoic&Unsafe详解
date: 2021-07-15 17:07:15
tags:
    - 并发
categories:
    - 并发
---
# Atmoic类

## 原子操作
&nbsp;&nbsp;&nbsp;&nbsp;原子本意是不能被进一步分隔的最小粒子，而原子操作为不可被中断的一个或者一个系列操作。

## 处理器实现原子操作
1. 缓存行加锁
2. 总线加锁

## jvm实现原子操作
1. 锁
2. cas
    * cas是通过CMPXCHG指令来实现的，自旋cas，直至成功

<!--more-->

## cas应用-原子类
&nbsp;&nbsp;&nbsp;&nbsp;原子类也是juc里面的，也是doug li写的，原子类目录是`java.util.concurrent.atomic.*`。
1. 原子更新基本类型类
    1. AtomicInteger
        * 原子更新整型
    2. AtomicLong
        * 原子更新长整型
    3. AtomicBoolean
        * 原子更新布尔类型
2. 原子更新数组类型类
    1. AtomicIntegerArray
        * 原子更新整型数组元素
    2. AtomicLongArray
        * 原子更新长整型数组元素
    3. AtomicReferenceArray
        * 原子类型引用类型数组元素
3. 原子更新引用类型类（原子更新多个字段）
    1. AtomicReference
        * 原子更新引用类型
    2. AtomicReferenceArray
        * 原子更新引用类型数组
    3. AtomicMarkableReference
        * 原子更新带有标记位的引用类型
    4. AtomicStampedReference
        * 原子更新带有版本的引用类型（解决cas aba问题）
4. 属性原子修改器(Updater)
    1. AtomicIntegerFieldUpdater
        * 原子更新整型类型字段
    2. AtomicLongFieldUpdater
        * 原子更新长整型类型字段
    3. AtomicReferenceFieldUpdater
        * 原子更新引用类型的字段

> 属性原子修改器都是抽象类，每次使用都是必须使用newUpdater创建一个更新器。<font color='red'><b>那这个时候还需要注意的是，字段还必须设置成public volatile</b></font>

## api介绍
1. AtomicInteger使用
```java
public class AtomicIntegerTest {

    private static AtomicInteger atomicInteger = new AtomicInteger();

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(1);
        for (int i = 0; i < 10; i++) {
            new Thread(()->{
                try {
                    countDownLatch.await();
                    System.out.println(Thread.currentThread().getName());
                    atomicInteger.incrementAndGet();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
        Thread.sleep(3000);
        countDownLatch.countDown();
        Thread.sleep(3000);
        System.out.println(atomicInteger.get());
    }
}
```
2. AtomicIntegerArray使用
```java
/**
 * 这里需要注意的是，数组Integer原子操作不会操作原有的数组
 * AtomicIntegerArray这个是数组拷贝
 */
public class AtomicIntegerArrayTest {

    public static void main(String[] args) {
        int[] arr = new int[]{1,2};
        AtomicIntegerArray atomicIntegerArray = new AtomicIntegerArray(arr);
        atomicIntegerArray.getAndSet(0,10);
        System.out.println(atomicIntegerArray.get(0));
        System.out.println(arr[0]); // 数组数据没有改
    }
}
```
3. AtomicIntegerFieldUpdater使用
```java
/**
 * 注意：
 * private volatile Integer code; 这样的申明是有问题的
 * 1. java.lang.IllegalAccessException
 * 2. Must be integer type
 */
public class AtomicIntegerFieldUpdaterTest {

    public static void main(String[] args) {
        AtomicIntegerFieldUpdater atomicIntegerFieldUpdater = AtomicIntegerFieldUpdater.newUpdater(Student.class,"code");
        Student student = new Student(1,"jianghe");
        atomicIntegerFieldUpdater.getAndIncrement(student);
        System.out.println(atomicIntegerFieldUpdater.get(student));
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    private static class Student {
        public volatile int code;
        private String name;
    }
}
```
4. AtomicReferenceFieldUpdater
```java
/**
* 注意，这个ReferenceField是需要安装类型的
*
*/
public class AtomicReferenceFieldUpdaterTest {

    public static void main(String[] args) {

        AtomicReferenceFieldUpdater atomicReferenceFieldUpdater = AtomicReferenceFieldUpdater.newUpdater(Student.class,Integer.class,"code");
        Student student = new Student(1,"jianghe");
        atomicReferenceFieldUpdater.compareAndSet(student,1,2);
        System.out.println(student.getCode());
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    private static class Student {
        public volatile Integer code;
        private String name;
    }
}
```
5. AtomicStampedReference使用
```java
/**
 * 解决ABA问题
 */
public class AtomicStampedReferenceTest {

    public static void main(String[] args) {
        Integer a = 1;
        AtomicStampedReference atomicStampedReference = new AtomicStampedReference(a,1);
        atomicStampedReference.compareAndSet(1,2,1,2);
        System.out.println(atomicStampedReference.getReference());
        System.out.println(a); // 原值是不会变的
    }
}
```

## 原理介绍
这里通过`AtomicInteger`的使用来介绍原理，是通过`Unsafe`来实现的，想操作`Unsafe`类，这个也是固定的写法
```java
public class StudentIntegerFieldUpdater {
    // 这里必须是申明为volatile，类属性可以访问
    private volatile long code;
    private String name;

    public StudentIntegerFieldUpdater(Integer code, String name) {
        this.code = code;
        this.name = name;
    }

    private static Unsafe unsafe = UnsafeInstance.getUnsafe();

    private static final long codeOffset;

    static{
        try {
            codeOffset = unsafe.objectFieldOffset
                    (StudentIntegerFieldUpdater.class.getDeclaredField("code"));
        } catch (Exception ex) { throw new Error(ex); }

    }

    public void updateCode(Integer oldV,Integer newV){
        boolean b = unsafe.compareAndSwapInt(this, codeOffset, oldV,newV);
        System.out.println(b);
    }

    public static void main(String[] args) {
        StudentIntegerFieldUpdater studentIntegerFieldUpdater = new StudentIntegerFieldUpdater(10,"jianghe");
        System.out.println("偏移量："+codeOffset);
        System.out.println(ClassLayout.parseInstance(studentIntegerFieldUpdater).toPrintable());
        studentIntegerFieldUpdater.updateCode(10,11);
        System.out.println(studentIntegerFieldUpdater.code);
    }
}
```
输出结果
```java
偏移量：16
# WARNING: Unable to attach Serviceability Agent. You can try again with escalated privileges. Two options: a) use -Djol.tryWithSudo=true to try with sudo; b) echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
juc.juc08.StudentIntegerFieldUpdater object internals:
OFF  SZ               TYPE DESCRIPTION                       VALUE
  0   8                    (object header: mark)             0x0000000000000001 (non-biasable; age: 0)
  8   4                    (object header: class)            0xf800c105
 12   4   java.lang.String StudentIntegerFieldUpdater.name   (object)
 16   8               long StudentIntegerFieldUpdater.code   10
Instance size: 24 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

true
11
```
这里可以看到输出结果，偏移量是16，<font color='red'><b>那么偏移量到底是什么？通过对象头可以看到，offset为16表示的是`StudentIntegerFieldUpdater.code`=>10，其实就是通过偏移量得到对象的内存值</b></font>，然后再去操作


# Unsafe
&nbsp;&nbsp;&nbsp;&nbsp;那Unsafe如此重要，那Unsafe是什么呢？Unsafe是sun.misc下面的一个类，主要提供一些低级别、不安全的操作，如直接访问系统内存资源、自主管理内存资源，这些方法在提升Java运行效率、增强Java语言底层资源操作能力方面起了很大作用。但由于Unsafe类使java像c一样操作内存空间，这样肯定是有问题，会导致程序出错的几率变大，一定要对Unsafe使用一定非常注意

## Unsafe能力
1. 内存操作
2. cas
3. 内存屏障
4. 获取释放锁
5. 线程阻塞
![Unsafe](/images/pasted-70.jpg)

### Unsafe-内存管理
Unsafe操作的native方法，是<font color='red'><b>堆外内存</b></font>的分配、扩展、拷贝、释放、设置特定值
```java
// 分配内存
public native long allocateMemory(long var1);
// 扩展内存
public native long reallocateMemory(long var1, long var3);
// 设置内存值
public native void setMemory(Object var1, long var2, long var4, byte var6);
// 内存拷贝
public native void copyMemory(Object var1, long var2, Object var4, long var5, long var7);
// 释放内存
public native void freeMemory(long var1);
// 获取地址的byte值
public native byte getByte(long var1);
```

***
<font color='red'><b>堆外内存</b></font>
1. 堆外内存是不受jvm管理，垃圾回收也不是自动回收的
2. 使用堆外内存的原因
    1. 对垃圾回收停顿的改善。由于堆外内存是直接受操作系统管理而不是jvm，所以当我们使用堆外内存的时候，即可保持较小的的堆内内存的规模，减少GC停顿的影响
    2. 提升程序I/O操作的性能。通常在I/O通信过程中，会存在堆内内存到堆外内存的数据拷贝，对于需要频繁进行内存间数据拷贝且生命周期较短的暂存数据，都建议存在堆外内存

DirectByteBuffer是堆外内存的典型例子，通常在通信过程中当做缓冲池，如Netty等NIO网络使用
![DirectByteBuffer](/images/pasted-69.jpg)

### Unsafe-cas
原子类操作
```java
public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);

public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);

public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);

```

### Unsafe-线程调度
线程阻塞、唤醒、锁机制
```java
public native void unpark(Object var1);

public native void park(boolean var1, long var2);

@Deprecated
public native void monitorEnter(Object var1);

@Deprecated
public native void monitorExit(Object var1);

@Deprecated
public native boolean tryMonitorEnter(Object var1);

```

### Unsafe-内存屏障
```java
// 禁止load操作重排序
public native void loadFence();
// 禁止store操作重排序
public native void storeFence();
// 禁止load、store操作重排序
public native void fullFence();
```
哪里有使用呢？`StampedLock`，其中validate方法，校验数据是否一致，通过unsafe.loadFence()加入load屏障
```java
/**
* 
* Returns true if the lock has not been exclusively acquired
* since issuance of the given stamp. Always returns false if the
* stamp is zero. Always returns true if the stamp represents a
* currently held lock. Invoking this method with a value not
* obtained from {@link #tryOptimisticRead} or a locking method
* for this lock has no defined effect or result.
*
* @param stamp a stamp
* @return {@code true} if the lock has not been exclusively acquired
* since issuance of the given stamp; else false
*/
public boolean validate(long stamp) {
    U.loadFence();
    return (stamp & SBITS) == (state & SBITS);
}
```

## 注意事项
如果直接Unsafe.getUnsafe()来获取Unsafe是有问题的，因为如果不是系统类加载器来获取的话会报错的
```java
@CallerSensitive
public static Unsafe getUnsafe() {
    Class var0 = Reflection.getCallerClass();
    if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
        throw new SecurityException("Unsafe");
    } else {
        return theUnsafe;
    }
}
```
那如果想获取Unsafe的话，可以通过反射来获取
```java
public static Unsafe getUnsafe() {
    try {
        Field field = Unsafe.class.getDeclaredField("theUnsafe");
        field.setAccessible(true);
        return (Unsafe)field.get(null);
    } catch (Exception e) {
    }
    return null;
}
```