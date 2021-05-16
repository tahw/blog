title: MESI
author: jianghe
date: 2021-05-15 20:17:59
tags:
---
CPU架构中传统的MESI协议中有两个行为的执行成本比较大。一个是将某个Cache Line标记为Invalid状态，另一个是当某Cache Line当前状态为Invalid时写入新的数据。所以CPU通过Store Buffer和Invalidate Queue组件来降低这类操作的延时。

![MESI](/images/pasted-37.png)

当一个核心在Invalid状态进行写入时，首先会给其它CPU核发送Invalid消息，然后把当前写入的数据写入到Store Buffer中。然后异步在某个时刻真正的写入到Cache Line中。当前CPU核如果要读Cache Line中的数据，需要先扫描Store Buffer之后再读取Cache Line（Store-Buffer Forwarding）。但是此时其它CPU核是看不到当前核的Store Buffer中的数据的，要等到Store Buffer中的数据被刷到了Cache Line之后才会触发失效操作。

而当一个CPU核收到Invalid消息时，会把消息写入自身的Invalidate Queue中，随后异步将其设为Invalid状态。和Store Buffer不同的是，当前CPU核心使用Cache时并不扫描Invalidate Queue部分，所以可能会有极短时间的脏读问题。这里的Store Buffer和Invalidate Queue的说法是针对一般的SMP架构来说的，不涉及具体架构。