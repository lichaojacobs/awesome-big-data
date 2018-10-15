---
title: impala集群搭建
date: 2018-08-12 20:17:45
tags:
---

## 前言

- 说起Impala，很多人都不会陌生。它区别于MapReduce 中间结果溢写，跨节点数据获取的低效，采用MPP 查询引擎，各查询节点并发执行查询语句，并将生成的查询结果汇总输出。
- 近期开始真正的使用impala，之前只是小玩过已经集成好的环境，并没有真正的从0到1的去构建Impala集群。基于我司所有的大数据组件都是采用容器的方式部署以便统一管理，我们需要先构建Impala镜像

## impala 相关介绍

- impala是由cloudera公司主导开发的一款大数据实时查询的分析工具，区别于Hive底层传统的MapReduce批处理方式，采用MPP查询引擎架构，相比于Hive 能带来查询性能30-90倍的提升
- 特点
	- 查询速度快: 底层MPP查询引擎。基于内存计算，中间结果不写入磁盘，由coordinator汇总数据结果
	- 灵活性高：可以兼容存储在HDFS上的原生数据，也可以兼容优化处理过的压缩数据，与Hive共用metastore兼容从Hive上导入的所有sql语句
	- 可伸缩性：可以很好的和一些BI工具去配合使用，如Microstrategy、Tableau、Qlikview等。

- 架构

![impala-architecture](http://ol7zjjc80.bkt.clouddn.com/impala-architecture.png)

- 集群角色

	- impalad(query planner,coordinator, exec engine)：

		- 分布在datanode节点上，接受客户端的查询请求
		- 接收查询请求的impalad 会作为本次查询的coordinator，生成查询计划树，并分发给具有相应数据的impalad执行
		- 汇总各个impalad上执行的查询结果，返回给客户端
	- statestore
		- 跟踪集群中impalad的健康状态及信息位置，并把健康状况同步到所有的impalad进程节点
	- catalog
		- 将元数据的变化通知给集群的各个节点，减少refresh和invalidate metadata语句使用

## 镜像构建篇

### hive镜像构建

- 因为Impala依赖hive metastore，所以在构建impala镜像之前，先要构建hive镜像
- 构建过程
	- impala 不支持 hive 2.x以上的系列，于是选择1.1.0版本，在cloudera官网下载编译好的tarball
	- 原本的hive lib 中缺少连接metastore的mysql jdbc驱动，自己下载jdbc connector jar放入lib目录下即可
	- 由于我hive 用的cdh的版本，而hadoop用的是apache的版本，导致真正运行hive的时候会找不到mapreduce指定的类，为此lib目录下需加入hadoop-core-2.6.0-mr1-cdh5.9.1.jar
	- 使用自带的schematool 创建元数据表
	
	```
	./schematool -initSchema -dbType mysql
	
	```
	
### impala镜像构建

- impala 我使用的是2.12.0的版本，这个版本cloudera官方没有提供tarball文件，只提供的rpm包。考虑到自身编译impala 成本比较大，于是采用rpm 安装的方法
- 详细的安装流程参考链接 [Impala安装配置–RPM方式](http://lxw1234.com/archives/2017/06/862.htm) 我这边只记录一下安装过程中遇到的坑
- 坑一：跟hive一样，由于我使用的hadoop是apache开源版本，有些class对应的包中没有
	- 软链hadoop-core-2.6.0-mr1-cdh5.9.1.jar到impala/lib中
	- 需要对服务的启动文件catalogd, impalad, stastored改动

		```
		for JAR_FILE in ${IMPALA_HOME}/lib/*.jar; do
   			export CLASSPATH="${JAR_FILE}:${CLASSPATH}"
		done
		#hadoop share lib
		for HADOOP_JAR_FILE in $HADOOP_HOME/share/hadoop/tools/lib/*.jar; do
   			export CLASSPATH="${HADOOP_JAR_FILE}:${CLASSPATH}"
		done
	
		```
	- 坑二：软链jdbc driver到impala/lib 并修改catalogd，其他启动文件相应都要修改

	- 坑三：启动catalogd,impala-server时报错

		```
		E0804 16:42:09.008862   543 MetaStoreUtils.java:1274] Got exception: java.io.IOException No FileSystem for scheme: hdfs
Java exception follows:
		java.io.IOException: No FileSystem for scheme: hdfs
		```
		
		core-site.xml中加入配置
		
		```
		<property>
     		<name>fs.file.impl</name>
     		<value>org.apache.hadoop.fs.LocalFileSystem</value>
     		<description>The FileSystem for file: uris.</description>
   		</property>
   		<property>
     		<name>fs.hdfs.impl</name>
     		<value>org.apache.hadoop.hdfs.DistributedFileSystem</value>
     		<description>The FileSystem for hdfs: uris.</description>
   		</property>
		
		```
		
	- 坑四 (class org.apache.hdfs.DistributedFileSystem not found)：

		```
		在hadoop2.8.3的版本中org.apache.hadoop.hdfs.DistributedFileSystem can be found in hadoop-hdfs-client jar. 
		只需要将包引入impala/lib 目录下即可

		
		```
		
	- 坑五：HBase client各种依赖包没有

		```
		ln -s $HBASE_HOME/lib/hbase-shaded-miscellaneous-2.1.0.jar hbase-shaded-miscellaneous.jar
		ln -s $HBASE_HOME/lib/hbase-shaded-protobuf-2.1.0.jar hbase-shaded-protobuf.jar
		ln -s $HBASE_HOME/lib/commons-lang3-3.6.jar commons-lang3.jar
以
		
		```
		
	- 坑六: 启动impala-server提示sasl plugin not found

		```
		yum install cyrus-sasl* 
		restart impala service

		
		```
		
## impala 集群搭建

### 集群规划

- 20 个 impalad服务 与datanode混部，以最大化利用分布式本地查询的优势
- catalog, statestore 与namenode节点混部以减少namenode网络IO带来的影响
- 全部服务部署docker化，脚本化，提供200与503脚本以便统一批量操作服务启动终止
- 200启动脚本是对impala提供的原生的启动命令的封装，包括监听运行进程的存活，便于对集群机器批量操作

- 200脚本封装代码

```
#!/usr/bin/env bash
EXEC_FILE=/impala/impala-server
PROGRESS=/usr/lib/impala/bin/impalad

function usage() {
    echo -e "\n A tool used for starting impala services
Usage: 200.sh {statestore|catalog|server}
"
}

check_alive() {
    PID=`ps -ef | grep $IMPALA_USER_NAME | grep "$PROGRESS" | awk '{print $2}'`
    [ -n "$PID" ] && return 0 || return 1
}

start_service() {
    if [ ! -f $EXEC_FILE ];then
        echo "file not exists"
        exit 1
    fi
    check_alive
    if [ $? -ne 0 ];then
        $EXEC_FILE restart
        sleep 10
        check_alive
        if [ $? -ne 0 ];then
            echo "service start error"
            exit 1
        else
            echo "service start success"
            exit 0
        fi
    else
        echo "service alreay started"
        exit 0
    fi
}

function main() {
    # $SERVICE_POOL是对应的服务池，通过docker run -e 参数传入
    [[ -f /data0/hcp/conf/pools/${SERVICE_POOL}/init-env.sh ]] && source /data0/hcp/conf/pools/${HCP_POOL}/init-env.sh
    [[ -f /data0/hcp/sbin/impala/init-impala.sh ]] && source /data0/hcp/sbin/impala/init-impala.sh
    case "$1" in
    	statestore)
            echo "starting impala statestore"
    	    EXEC_FILE=/data0/hcp/sbin/impala/impala-state-store
            PROGRESS=/usr/lib/impala/sbin/statestored
            start_service
    	    ;;
        server)
            echo "starting impala server"
            EXEC_FILE=/data0/hcp/sbin/impala/impala-server
            PROGRESS=/usr/lib/impala/sbin/impalad
            start_service
    	    ;;
        catalog)
            echo "starting impala catalog"
            EXEC_FILE=/data0/hcp/sbin/impala/impala-catalog
            PROGRESS=/usr/lib/impala/sbin/catalogd
            start_service
    	    ;;            
        *)
            usage
            exit 1
    esac
}
main "$@"

```

- 503脚本设计思路类似






