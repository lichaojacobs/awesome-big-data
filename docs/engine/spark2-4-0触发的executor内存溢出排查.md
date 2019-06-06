---
title: 【spark-tips】spark2.4.0触发的executor内存溢出排查
date: 2019-01-12 12:01:01
tags:
	- spark
---

### 版本升级背景

spark 2.4.0 最近刚发版，新增了很多令人振奋的特性。由于本司目前使用的是spark 2.3.0版本，本没打算这么快升级到2.4.0。无奈最近排查出的两个大bug迫使我们只能对spark进行升级。排查的两个bug如下：

- spark2.3.0 bug导致driver跑一段时间内存溢出，经过dump下来的堆转储文件发现占绝大内存的对象是spark自身的ElementTrackingStore。这是统计任务运行时资源占用情况的类，在每一个批次处理完之后都没有释放，导致driver内存溢出 
    ![](http://imgs.wanhb.cn/memory.png)

    - 详情参考文章：[导致driver OOM的一个SparkPlanGraphWrapper源码的bug](https://www.cnblogs.com/bethunebtj/p/9103547.html)
- spark streaming 2.3.0 + kafka 0.8.2.1 + zk管理offset 每次重启，会导致offsetrange的左区间莫名向右移动若干offset size，导致每个批次通过offsetrange从kafka消费的数据普遍会丢失部分数据，具体问题还在通过源码定位中

第一个bug在spark 2.4.0中得到解决（[参考issue](https://issues.apache.org/jira/browse/SPARK-23670)），于是对spark进行了升级。所幸spark升级对spark on yarn这种运行方式来说非常解耦，只需要定义spark.jars依赖就行，yarn nodemanager会对依赖包进行下载。

### 遇到的问题

#### 问题描述
spark 版本升级之后，当天对在线任务观察，运行平稳，上一节提到的bug也修复了；但是第二天离线任务的运行却出现了问题：部分离线任务在做聚合运算的时候出现executor 集体内存溢出，任务执行失败

#### 问题排查

- 查看日志发现是内存溢出导致executor触发钩子异常退出

    ![](http://imgs.wanhb.cn/spark_1.png)

- 进一步发现在某个计算步骤，需要读入上一个步骤写入hdfs的数据，每个task处理的数据量比较大，且都放到了内存中（*导火线*）
    ![](http://imgs.wanhb.cn/spark_2.png)

- 接着因为要做各种聚合运算（reduceby, groupgy, join…）execution 的内存也不断增大，濒临内存的限制边缘 8G * 0.6(spark.memory.fraction) =4.8G，很容易就会来不及spill到磁盘，导致内存溢出
    ![](http://imgs.wanhb.cn/spark_2.png)

- 于是基本可以得到问题原因：从读入hdfs的源头去排查，为什么导致一个task处理的数据量过大；发现hdfs中上一步save到hdfs中的每一个part都是将近500M大小的parquet+snappy 压缩文件，而这种格式无法切分，导致一个map task只能对这400多M的文件照单全收，而由于我应用申请的配置是 *8 cores 8 mem / executor* 导致8个task同时读入大文件到executor jvm中，最终jvm报内存溢出异常
    ![](http://imgs.wanhb.cn/spark_4.png)

#### 解决方案
- 限制executor 并行度，将cores 调小，内存适当调大
- 由于上一步写hdfs的操作并行度太小（只有40），重新调整并行度，让输出的每个part文件减小