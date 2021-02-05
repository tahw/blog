title: flink介绍
author: jianghe
tags:
  - flink
categories:
  - 大数据
date: 2021-01-10 19:26:00
---
![](/images/pasted-1.png)
### flink是什么？
> Apache Flink is a framework and distributed processing engine for stateful computations over unbounded and bounded data streams.
> apache flink是一个<font color='red'>**框架**</font>和<font color='red'>**分布式处理引擎**</font>，用于对<font color='red'>**无界**</font>与<font color='red'>**有界**</font>数据流进行<font color='red'>**状态**</font>计算

### 为什么选择flink?
- 流数据更真实的反应我们的生活方式
	- 源源不断
- 传统的数据架构是基于有限数据集
	- 非实时
	- spark streaming
- 实时不是一堆数据到达某个时间点处理（批处理），类似于实时聊天
- 目标
	- 低延迟
		- ms级别
	- 高吞吐
		- 分布式
	- 结果的准确性和良好的容错性
		- 分布式保持准确
		- 数据传输，合并数据正确性

### 哪些业务需要使用?
- 数据报表
- 用户交互密集
- 实时聊天
- ...

### 应用场景对比
1. 事件驱动应用
	- 传统架构
		- 读写远程事务型数据库
		- 优点：实时性好
		- 缺点：能够处理的数据有限，<font color='red'>**高并发做不好**</font>
	- flink
    	- 无须查询远程数据库
        - 本地数据访问使得它具有<font color='red'>**更高的吞吐**</font>和<font color='red'>**更低的延迟**</font>
		- savepoint例子

![](/images/pasted-2.png)

2. 数据分析应用
	- 批处理
		- 利用批查询，或将事件记录下来并基于此有限数据集构建应用来完成，为了得到最新数据的分析结果，必须先将它们加入分析数据集并重新执行查询或运行应用，随后将结果写入存储系统或生成报告。
		- 能够处理海量数据
		- <font color='red'>**实时性不好**</font>
	- 流处理
		- 流式查询或应用会接入实时事件流，并随着事件消费持续产生和更新结果。这些结果数据可能会写入外部数据库系统或以内部状态的形式维护。仪表展示应用可以相应地从外部数据库读取数据或直接查询应用的内部状态。
		- 流式分析省掉了周期性的数据导入和查询过程，因此从事件中获取指标的<font color='red'>**延迟更低**</font>
	- 批处理和流处理
		- flink既可做批处理和流处理
		- 处理必须处理定期导入和输入有界性导致的人工数据边界，而流式查询则无须考虑该问题

![](/images/pasted-3.png)

3. 数据管道应用
	- 周期性 ETL 作业
		- 提取-转换-加载（ETL）是一种在存储系统之间进行数据转换和迁移的常用方法。ETL 作业通常会周期性地触发，将数据从事务型数据库拷贝到分析型数据库或数据仓库
		- <font color='red'>**实时性不好**</font>
    - 持续数据管道
        - 可以明显<font color='red'>**降低将数据移动到目的端的延迟**</font>。此外，由于它能够持续消费和发送数据，因此用途更广，支持用例更多
![](/images/pasted-4.png)

### flink 架构图
1. state
	- 数据直接存在本地内存
	- 集群
	- <font color='red'>**低延迟**</font>
	- <font color='red'>**高吞吐**</font>
2. checkpoint
	- 持久化
	- 故障容错
3. Exactly-once
	- 数据正确
![flink](/images/pasted-0.png)

### flink cdc(change data capture) etl例子
https://github.com/ververica/flink-cdc-connectors/wiki/%E4%B8%AD%E6%96%87%E6%95%99%E7%A8%8B