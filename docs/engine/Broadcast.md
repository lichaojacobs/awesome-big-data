---
title: 【Spark源码分析】Broadcast
date: 2019-06-02 13:31:16
tags:
    - 学习
    - 大数据
    - Spark
---

## Broadcast 原理

### 满足broadcast join的条件源码分析

- 来看SparkStrategies.scala文件
- Broadcast 策略入口 broadcastSideBySizes
  ![](http://imgs.wanhb.cn/spark-broadcast1.png)

  - 可以发现broadcast 左表或者是右表是根据两个策略来控制的：canBuildLeft/canBuildRight， canBroadcast；

  - canBroadcast控制的是数据大小是否符合参数设定
    ![](http://imgs.wanhb.cn/spark-broadcast2.png)

  - canBuildLeft/canBuildRight是判断被广播的表是否作为left或right join基表的情况；如果作为基表的话是不能被broadcast的；当然Inner join不用管是不是基表
    ![](http://imgs.wanhb.cn/spark-broadcast3.png)

    ![](http://imgs.wanhb.cn/spark-broadcast4.png)

- 基表不能被广播的原因
	- 	left/right join 之所以基表不能broadcast是因为这样做会破坏left join语义，产生重复的数据(比如广播了n份基表，因为最后都要保留基表的数据，不管有没有匹配上，所以会导致归并的时候有重复的情况)

	- 翻阅其他博客对broadcast的解释，也能发现基表不能被广播的事实 [Spark SQL中Join常用的几种实现](https://www.iteblog.com/archives/2086.html) 
	
	  ![](http://imgs.wanhb.cn/spark-broadcast5.png)