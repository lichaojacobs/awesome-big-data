---
title: Kylin二次开发——测试环境搭建
date: 2017-08-07 17:55:39
tags:
	- kylin
	- Java
---

### 调研背景

虽然公司目前在生产环境上正式用上了kylin，但是由于其本身年龄不长，社区并不完善，难难免会暴露出各种各样的源码级别的问题(包括上一篇介绍的kylin的同步机制的问题)。这时候使用者想等着官方推出新的release未免太过于被动。于是，我们想着对kylin进行二次开发以满足我们对定制化需求。事实上，目前我们使用的所有开源框架在一定程度上都进行了多多少少的二次开发：

- superset接入kylin，完成了自编译集成进了docker，并修改了 flask-appbuilder的源码逻辑，兼容ldap与原本的账号系统。
- airflow 正在考虑从输出信息中判断task是否执行成功，而不是单纯的靠进程是否异常退出判断（主要考虑到支持kylin的任务调度）

其实在kylin的官网对于开发环境搭建大致的步骤都做了介绍下面描述整个过程

### 搭建过程

- 想应用经过自己二次开发的kylin当然必须得全覆盖的跑一遍所有的单元测试。这就意味着必须得有Hadoop+hive+hbase等一整套测试环境。这里使用kylin官方推荐的Hortonworks Sandbox。为了方便，直接使用Sandbox on docker。
	- [按照官网的教程（docker占用的内存至少8G以上，否则运行不了）](https://hortonworks.com/tutorial/sandbox-deployment-and-install-guide/section/3/)
	- 在执行start_sandbox-hdp.sh的时候需要往映射的端口里面加入hive metastore的thrift端口9083，否则本地跑单元测试的时候连不上metastore
	- ssh上运行的container，修改admin密码:
	
	```
	ssh -p 2222 root@localhost
	或者http://127.0.0.1:4200/ 进入shell浏览器界面
	//执行
   ambari-admin-password-reset

	```

- 用修改的admin密码登录[http://localhost:8080](http://localhost:8080)，确保dashboard中hive+mapreduce+hdfs+hbase正常启动

- 修改kylin.properties的几个值:
	
	```
	//KYLIN_HOME/examples/test_case_data/sandbox/kylin.properties
	
	kylin.job.use-remote-cli=true
	kylin.job.remote-cli-hostname=sandbox
	kylin.job.remote-cli-username=root
	kylin.job.remote-cli-password=xxxx
	//这个默认是22端口，由于我本地不生效，就直接设置为2222端口
	kylin.job.remote-cli-port=2222
	
	```
	
- 如果单个单元进行测试，不想每次从头开始，方便集中debug某个moudle的错误，可以注释掉pom.xml中的check-style插件
	
	```
	 <!-- <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-checkstyle-plugin</artifactId>
                    <version>2.17</version>
                    <dependencies>
                        <dependency>
                            <groupId>com.puppycrawl.tools</groupId>
                            <artifactId>checkstyle</artifactId>
                            <version>6.19</version>
                        </dependency>
                    </dependencies>
                    <executions>
                        <execution>
                            <id>check-style</id>
                            <phase>validate</phase>
                            <configuration>
                                <configLocation>dev-support/checkstyle.xml</configLocation>
                                <suppressionsLocation>dev-support/checkstyle-suppressions.xml</suppressionsLocation>
                                <includeTestSourceDirectory>true</includeTestSourceDirectory>
                                <consoleOutput>true</consoleOutput>
                                <failsOnError>true</failsOnError>
                            </configuration>
                            <goals>
                                <goal>check</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin> -->
                
	```
	
- 执行测试命令

	```
	//base
	mvn test -fae -Dhdp.version=${HDP_VERSION:-"2.6.1.0-129"} -P sandbox -X
	//全覆盖
	mvn verify -Dhdp.version=${HDP_VERSION:-"2.6.1.0-129"} -fae 2>&1 | tee mvnverify.log
	
	```

### 踩坑记录

- 该暴露的端口得暴露出来，否则本地测试的时候对指定的端口无法进行tcp通信
	
	```
	9083:9083 //hive metastore
	8050:8050 // kylin 获取job output信息端口
	50010:50010 // dfs.datanode.address，这里踩坑很久,不配置的话会有：createBlockOutputStream when copying data into HDFS错误

	```
- 启动start_sandbox-hdp脚本的时候，可能会出现postgresql服务启动不了，这很可能是因为其申请的共享内存超过了系统的，只需要进入到容器里面做相关操作
	
	```
	ssh -p root@localhost
	sudo sysctl -w kernel.shmmax=17179869184 //假设你有16G内存，按实际扩充
	sudo service postgresql start
	//postgresql启动之后在容器外重新执行
	./start_sandbox-hdp.sh 就可以了

	```

