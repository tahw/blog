title: 并发之Volatile详解MESI
author: jianghe
tags:
  - 并发
categories:
  - 并发
date: 2021-05-15 20:17:00
---
# jvm整个执行过程
![jvm->cpu执行过程](/images/pasted-39.png)

<!-- more -->

# volatile可见性
```java
/**
 * 打印汇编
 * 在jdk库里加上/Library/Java/JavaVirtualMachines/jdk1.8.0_201.jdk/Contents/Home/jre/lib/hsdis-amd64.dylib
 * vm options：-server -Xcomp -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand=compileonly,*JavaVolatile.fresh
 */
public class JavaVolatile {

    private volatile static boolean initFlag = false;

    private static int j = 0;

    private static void fresh(){
        initFlag = true;
    }

    public static void main(String[] args) {
        Thread thread = new Thread(new Runnable() {
            public void run() {
                while (!initFlag) {
                    j++;
                }
                System.out.println("param=" + initFlag);
            }
        });

        Thread thread1 = new Thread(new Runnable() {
            public void run() {
                fresh();
            }
        });

        thread.start();

        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        thread1.start();
    }
}
```
其中输出结果如下，发现被volatile修饰的initFlag中有`lock#`打头。
```
  0x000000010657165b: movabs $0x7956a1a38,%rsi  ;   {oop(a 'java/lang/Class' = 'juc/juc03/JavaVolatile')}
  0x0000000106571665: mov    $0x1,%edi
  0x000000010657166a: mov    %dil,0x6c(%rsi)
  0x000000010657166e: lock addl $0x0,(%rsp)     ;*putstatic initFlag
                                                ; - juc.juc03.JavaVolatile::fresh@1 (line 14)
```

# 总线锁和缓存一致性协议
`lock#`是汇编指令。用lock会触发硬件缓存锁定机制。总线锁和缓存一致性协议。
> 在IA-32架构中有描述以下文字：在修改内存操作时，使用 LOCK 前缀去调用<font color='red'><b>加锁的读-修改-写操作(调用虽然是多步，但是整体的操作是原子操作)</b></font>。这种机制用于多处理器系统中处理器之间进行可靠的通讯，具体描述如下：在 Pentium 和早期的 IA-32 处理器中，LOCK 前缀会使处理器执行当前指令时产生一个 LOCK#信号，这总是引起显式总线锁定出现

## 总线锁
&nbsp;&nbsp;&nbsp;&nbsp;早期技术比较落后，cpu多核操作主内存，会发生线程安全问题，然后会加一个总线锁来解决线程安全。如果一个cpu要操作数据，首先先获取总线锁。然后再去操作数据。但是这样的效率非常低，一个cpu拿到总线锁，另外的cpu就只能等待。发挥不了多核的能力。
> 这里要引申下，现在还是用到总线锁，是在mesi操作不了的时候才采用总线锁，那么什么是mesi操作不了，<font color='red'><b>由于cpu数据是放在cache line里的，如果锁不住cache line。就只能是用总线锁。（什么情况下锁不住cache line，使用多个cache line，多个cache line不是原子操作）</b></font>

## 缓存一致性协议（MESI）
现在的处理器都是多核处理器，并且每个核都带有多个缓存（指令缓存和数据缓存，见下图）。为什么需要缓存呢，这是因为CPU访问内存的速度比较慢，所以在CPU和内存之间加了个缓存以提高访问速度。既然每个核都有缓存，那么假设两个核或者多个核同时访问同一个变量时这些缓存是如何进行同步的呢（缓存细分为一个个缓存行），这就有了MESI协议。MESI是指4个状态的首字母。每个cache line有4个状态。用两个2个bit表示。
![cpu](/images/并发3-2.png)

| 状态 | 描述 |监听任务 |
| :-----| :---- | :---- |
| M(修改) | <b><font color='red'>当前核的cache line有效，数据被修改了，和内存中的数据不一致，数据只存在本cache中</font></b> | 缓存行必须时刻监听其他核试图读该缓存行数据，，这种操作必须在当前核的cache line写回主存并将状态变成S（共享）状态，要不然就是等待 |
| E(独享) | <b><font color='red'>当前核的Cache line有效，数据和内存中的数据一致，数据只存在于本Cache中。</font></b> | 缓存行必须监听其它核读主存中该缓存行的操作，一旦有这种操作，该缓存行需要变成S（共享）状态。 |
| S(共享) | <b><font color='red'>当前核的Cache line有效，数据和内存中的数据一致，数据存在于很多Cache中。</font></b> | 缓存行必须监听其它核操作，当其他核要修改当前缓存行，当前核需要将缓存行变成无效（Invalid）。|
| I(无效) | <b><font color='red'>该Cache line无效。</font></b> | 无 |

> 对于M和E状态而言总是精准的。他们在和该缓存行的真正状态是一致的，而S状态有可能是不一致的。如果一个核A将处于S状态的缓存行作废了，而另一个核B实际上可能已经独享了该缓存行，但是核B缓存却不会将该缓存行升迁为E状态，这是因为核A缓存不会广播他们作废掉缓存行的通知，同样由于缓存并没有保存该缓存行的copy的数量，因此（即使有这种通知）也没有办法确定核B是否已经独享了该缓存行。
从上面的意义看来E状态是一种投机性的优化：如果一个CPU想修改一个处于S状态的缓存行，总线事务需要将所有该缓存行的copy变成invalid状态，而修改E状态的缓存不需要使用总线事务。

### MESI状态切换

![](/images/pasted-41.png)
理解该图的前置说明：
1. 触发事件
* 本地读取(local read)-本地内核读取本地cache数据
* 本地写入(local write)-本地内核写入本地cache数据
* 远端读取(remote read)-其它内核读取其它Cache中的值
* 远端写入(remote write)-其他内核写入其他cache数据

> 内核不会跨核读取

MESI状态扭转：
<table>
	<tr>
	    <th>当前核缓存状态</th>
	    <th>事件</th>
	    <th>行为</th>
      	<th>当前核下一个缓存状态</th>
	</tr >
	<tr >
	    <td rowspan="4">I</td>
	    <td>local read</td>
	    <td>1. 如果其它核没有这份数据，本核Cache从内存中取数据，Cache line状态变成E；<br>
          2. 如果其它核Cache有这份数据，且状态为M，则需要等其他核将数据更新到内存，本核Cache再从内存中取数据，2个Cache 的Cache line状态都变成S；<br>
		  3. 如果其它核Cache有这份数据，且状态为S或者E，本核Cache从内存中取数据，这些Cache 的Cache line状态都变成S
</td>
        <td>E/S</td>
	</tr>
	<tr>
	    <td>local write</td>
	    <td>1. 从内存中取数据，如果其他核cache没有数据，然后在当前核Cache中修改，状态变成M；<br>
          2. 如果其它核Cache有这份数据，且状态为M，则要先将数据更新到内存；其他核状态会变成S，然后当前核再去写入<br>
          3. 如果其它核Cache有这份数据，且状态为E或S，则其它核Cache的Cache line状态变成I</td>
      	    <td>M</td>
	</tr>
	<tr>
	    <td>remote read</td>
	    <td>既然是Invalid，别的核的操作与它无关</td>
      	    <td>I</td>
	</tr>
	<tr>
	    <td>remote write</td>
	    <td>既然是Invalid，别的核的操作与它无关</td>
      	    <td>I</td>
	</tr>
  <tr >
	    <td rowspan="4">E</td>
	    <td>local read</td>
	    <td>从Cache中取数据，状态不变</td>
        <td>E</td>
	</tr>
	<tr>
	    <td>local write</td>
	    <td>修改Cache中的数据，状态变成M</td>
      	<td>M</td>
	</tr>
	<tr>
	    <td>remote read</td>
	    <td>数据和其它核共用，状态变成了S</td>
      	<td>S</td>
	</tr>
	<tr>
	    <td>remote write</td>
	    <td>数据被修改，本Cache line不能再使用，状态变成I</td>
      	<td>I</td>
	</tr>
  <tr >
	    <td rowspan="4">S</td>
	    <td>local read</td>
	    <td>从Cache中取数据，状态不变</td>
        <td>S</td>
	</tr>
	<tr>
	    <td>local write</td>
	    <td>修改Cache中的数据，状态变成M，其它核共享的Cache line状态变成I</td>
      	<td>M</td>
	</tr>
	<tr>
	    <td>remote read</td>
	    <td>状态不变</td>
      	    <td>S</td>
	</tr>
	<tr>
	    <td>remote write</td>
	    <td>数据被修改，本Cache line不能再使用，状态变成I</td>
      	<td>I</td>
	</tr>
  <tr >
	    <td rowspan="4">M</td>
	    <td>local read</td>
	    <td>从Cache中取数据，状态不变</td>
        <td>E</td>
	</tr>
	<tr>
	    <td>local write</td>
	    <td>修改Cache中的数据，状态不变</td>
      	<td>M</td>
	</tr>
	<tr>
	    <td>remote read</td>
	    <td>等待当前核数据被写到内存后，然后其它核能使用到最新的数据，当前核状态变成S</td>
      	<td>S</td>
	</tr>
	<tr>
	    <td>remote write</td>
	    <td>等待当前核数据被写到内存中，使其它核能使用到最新的数据，由于其它核会修改这行数据，当前核状态变成I</td>
      	<td>I</td>
	</tr>
</table>


#### Store Buffer和Invalidate Queue
CPU架构中传统的MESI协议中有两个行为的执行成本比较大。一个是将某个Cache Line标记为Invalid状态，另一个是当某Cache Line当前状态为Invalid时写入新的数据。所以CPU通过Store Buffer和Invalidate Queue组件来降低这类操作的延时。

![MESI](/images/pasted-37.png)

> 当一个核心在Invalid状态进行写入时，首先会给其它CPU核发送Invalid消息，然后把当前写入的数据写入到Store Buffer中。然后异步在某个时刻真正的写入到Cache Line中。当前CPU核如果要读Cache Line中的数据，需要先扫描Store Buffer之后再读取Cache Line（Store-Buffer Forwarding）。但是此时其它CPU核是看不到当前核的Store Buffer中的数据的，要等到Store Buffer中的数据被刷到了Cache Line之后才会触发失效操作。
而当一个CPU核收到Invalid消息时，会把消息写入自身的Invalidate Queue中，随后异步将其设为Invalid状态。和Store Buffer不同的是，当前CPU核心使用Cache时并不扫描Invalidate Queue部分，所以可能会有极短时间的脏读问题。这里的Store Buffer和Invalidate Queue的说法是针对一般的SMP架构来说的，不涉及具体架构。