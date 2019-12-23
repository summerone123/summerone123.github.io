---
layout:     post
title:     Spark 完全分布式搭建
subtitle:   大数据处理
date:       2019-10-16
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - 大数据处理
---

# Spark 完全分布式搭建
## 实验目的
- 掌握在Linux环境下安装Spark、配置环境变量、配置Spark，启动和关闭Spark集群等；
- 掌握根据集群部署模式的不同，在集群上运行Spark应用程序的方法；

-------

## 实验平台
- 操作体统 ：centos
- spark 版本：2.4.4
- Hadoop 版本：2.8.4
- Scala 版本 2.12.2

-------

## 实验内容
### 安装 Scala
- 直接打开下面的地址
[http://www.scala-lang.org/download/2.12.2.html](http://www.scala-lang.org/download/2.12.2.html)
- 解压放到相应的文件夹下</br>

```shell
    cd    /opt/scala
    tar   -xvf    scala-2.12.2
```
- 配置环境变量
编辑.bashrc文件

```shell
添加
    export    SCALA_HOME=/opt/scala/scala-2.12.2
    ${SCALA_HOME}/bin     
配置后 
    source .bashrc    
```
- 验证是否成功

```shell
    scala     -version
```

### 安装 Spark
- 直接打开下面的地址
[http://spark.apache.org/downloads.html](http://spark.apache.org/downloads.html)
- 解压放到相应的文件夹下</br>

```shell
    cd    /opt/spark
    tar   -zxvf   spark-2.1.1-bin-hadoop2.7.tgz
```
- 配置环境变量
编辑.bashrc文件

```shell
添加
    export  SPARK_HOME=/opt/spark/spark-2.1.1-bin-hadoop2.7
    ${SPARK_HOME}/bin    
配置后 
    source .bashrc    
```

- 新建conf文件夹中spark-env.h文件
    
```shell
    cd    /opt/spark/spark-2.1.1-bin-hadoop2.7/conf
    cp    spark-env.sh.template   spark-env.sh
```
编辑spark-env.h文件，在里面加入配置**(具体路径以自己的为准)**：

```shell
    export SCALA_HOME=/opt/scala/scala-2.12.2
    export JAVA_HOME=/opt/java/jdk1.8.0_121
    export HADOOP_HOME=/opt/hadoop/hadoop-2.8.0
    export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
    export SPARK_HOME=/opt/spark/spark-2.1.1-bin-hadoop2.7
    export SPARK_MASTER_IP=hserver1
    export SPARK_EXECUTOR_MEMORY=1G
```

 - 新建conf文件夹中slaves文件
 
```shell
    cd   /opt/spark/spark-2.1.1-bin-hadoop2.7/conf
    cp    slaves.template   slaves
```
编辑slaves文件，里面的内容为：

```
    bd2016-41
    bd2016-71
    bd2016-101
    bd2016-131
    bd2016-161
```

### 启动和测试Spark集群

```shell
    cd   /opt/spark/spark-2.1.1-bin-hadoop2.7/sbin
    ./start-all.sh
```
![-w640](/img/blog_img/15712256598760.jpg)

### 测试Spark集群
访问 8080 端口
![-w1336](/img/blog_img/15712257858367.jpg)

### 使用Spark集群(采用独立集群管理器)
运行Spark提供的计算圆周率的示例程序(采用独立集群管理器)

```shell
./bin/spark-submit  --class  org.apache.spark.examples.SparkPi  --master local   examples/jars/spark-examples_2.11-2.4.4.jar 

```
![-w863](/img/blog_img/15712263447077.jpg)


### 使用Spark集群(采用Yarn集群管理器)
经过辛苦调试后正常使用 yarn 作为资源管理器运行程序的
![-w768](/img/blog_img/15712304432654.jpg)
![-w1437](/img/blog_img/15712305059543.jpg)
## 实验遇到的问题
-  ssh可以免密连接，但是执行`./start-all.sh`时，所有的子节点报`permission deny` ，此时因为传文件时出现问题等原因，使权限变了
![-w640](/img/blog_img/15712256598760.jpg)

解决方法：递归本文件所有文件给权限

```shell
    chmod -R 777 文件名  
    -R 是递归下面所有的文件
```
- 一直在执行某一个 application，其他的因此而堵塞
![-w818](/img/blog_img/15712271935314.jpg)
解决方法:直接 kill 掉此 application
![-w854](/img/blog_img/15712273364245.jpg)


-  改用Yarn 资源管理器时出现虚拟内存总量超过这个计算所得的数值等一系列问题
  好多的问题这里就不一一列举
- 解决方法：
- 1.关闭 hadoop 和spark 服务
  
```shell
[spark@master hadoop]$ cd /opt/hadoop-2.9.0
[spark@master hadoop]$ sbin/stop-all.sh
[spark@master hadoop]$ cd /opt/spark-2.2.1-bin-hadoop2.7
[spark@master hadoop]$ sbin/stop-all.sh
```
- 2.在master上修改yarn-site.xml 
在yarn-site.xml增加配置

```shell
<property>
    <name>yarn.nodemanager.pmem-check-enabled</name>
    <value>false</value>
</property>
<property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
    <value>false</value>
    <description>Whether virtual memory limits will be enforced for containers</description>
</property>
<property>
    <name>yarn.nodemanager.vmem-pmem-ratio</name>
    <value>4</value>
    <description>Ratio between virtual memory to physical memory when setting memory limits for containers</description>
</property>
```

- 3.将master上将l修改后的yarn-site.xm文件覆盖到各个slaves节点
这一点也是我们最长忽视的一个点(重点）

```shell
[spark@master hadoop]$ scp -r /opt/hadoop-2.9.0/etc/hadoop/yarn-site.xml spark@slave1:/opt/hadoop-2.9.0/etc/hadoop/
yarn-site.xml                                                                                                                                                             100% 2285   577.6KB/s   00:00    
[spark@master hadoop]$ scp -r /opt/hadoop-2.9.0/etc/hadoop/yarn-site.xml spark@slave2:/opt/hadoop-2.9.0/etc/hadoop/
yarn-site.xml                                                                                                                                                             100% 2285   795.3KB/s   00:00    
[spark@master hadoop]$ scp -r /opt/hadoop-2.9.0/etc/hadoop/yarn-site.xml spark@slave3:/opt/hadoop-2.9.0/etc/hadoop/
yarn-site.xml                                                                                                                                                             100% 2285     1.5MB/s   00:00    
```
- 4.重新启动hadoop,spark服务

```shell
[spark@master hadoop]$ cd /opt/hadoop-2.9.0
[spark@master hadoop]$ sbin/start-all.sh
[spark@master hadoop]$ cd /opt/spark-2.2.1-bin-hadoop2.7
[spark@master spark-2.2.1-bin-hadoop2.7]$ sbin/start-all.sh
[spark@master spark-2.2.1-bin-hadoop2.7]$ jps
5938 ResourceManager
6227 Master
5780 SecondaryNameNode
6297 Jps
5579 NameNode
```
- 5.重新以 yarn 为资源管理器正常运行
![-w768](/img/blog_img/15712304432654.jpg)
![-w1437](/img/blog_img/15712305059543.jpg)