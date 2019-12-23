---
layout:     post
title:     Hadoop 的三种调度器实现及调参
subtitle:   大数据处理
date:       2019-09-20
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - 大数据处理
---

# Hadoop 的三种调度器FIFO、Capacity Scheduler、Fair Scheduler 实现及调参

> 关于原理部分我上一篇 blog 已经介绍过了，这次通过实验来实现</br>
> 不懂三种基本调度方法的基础原理的可以点击这里看我的 上一篇介绍[点击这里](http://summerone.xyz/2019/09/17/Hadoop-%E7%9A%84%E4%B8%89%E7%A7%8D%E8%B0%83%E5%BA%A6%E5%99%A8/)

## 默认使用的是容量调度器
![-w1440](/img/blog_img/15689782483147.jpg)
## 我们首先改为FIFO 调度
#### 修改 yarn-site.xml ，修改对应的配置项

```java
<property>
<name>yarn.resourcemanager.scheduler.class</name>
<value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fifo.FifoScheduler</value>
</property>
```

**开两个终端进行测试**
![-w766](/img/blog_img/15689797813108.jpg)
**打开对应的 8088 端口观察** 此时我们可以看到对应的FIFO 调度
![-w1440](/img/blog_img/15689799071188.jpg)
**提交两个任务，观察FIFO 对于多个任务来时的调度情况**

```java
yarn jar hadoop-mapreduce-examples.jar pi 100 50
yarn jar hadoop-mapreduce-examples.jar pi 50 50
```
注:pi后的第一个参数是map的个数 第二个是每个map的执行次数
![-w1440](/img/blog_img/15689800437312.jpg)
此时我们观察，首先提交的那个任务未完成，后一个任务只能等待，符合 FIFO 的调度方式
## 容量调度
首先讲几点容量调度的特性，方便后期调参：
● 层次化的队列设计，这种层次化的队列设计保证了子队列可以使用父队列设置的全部资源。这样通过层次化的管理，更容易合理分配和限制资源的使用。
● 容量保证，队列上都会设置一个资源的占比，这样可以保证每个队列都不会占用整个集群的资源。
● 安全，每个队列又严格的访问控制。用户只能向自己的队列里面提交任务，而且不能修改或者访问其他队列的任务。
● 弹性分配，空闲的资源可以被分配给任何队列。当多个队列出现争用的时候，则会按照比例进行平衡。
● 多租户租用，通过队列的容量限制，多个用户就可以共享同一个集群，同时保证每个队列分配到自己的容量，提高利用率。
● 操作性，yarn支持动态修改调整容量、权限等的分配，可以在运行时直接修改。还提供给管理员界面，来显示当前的队列状况。管理员可以在运行时，添加一个队列；但是不能删除一个队列。管理员还可以在运行时暂停某个队列，这样可以保证当前的队列在执行过程中，集群不会接收其他的任务。如果一个队列被设置成了stopped，那么就不能向他或者子队列上提交任务了。
● 基于资源的调度，协调不同资源需求的应用程序，比如内存、CPU、磁盘等等。
### 实验
修改 yarn-site.xml ，修改调度方式为容量调度

```java
<property>
<name>yarn.resourcemanager.scheduler.class</name>
<value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
```
修改为容量调度器并添加dev、prod队列

```java
vim /etc/hadoop/conf/capacity-scheduler.xml
```

```java
<configuration>
      <property>
      <name>yarn.scheduler.capacity.root.default.capacity</name>
      <value>10</value>
    </property>

    <property>
      <name>yarn.scheduler.capacity.root.default.maximum-capacity</name>
      <value>100</value>
    </property>

    <property>
      <name>yarn.scheduler.capacity.root.default.user-limit-factor</name>
      <value>1</value>
    </property>
    <property>
      <name>yarn.scheduler.capacity.root.dev.capacity</name>
      <value>20</value>
    </property>

    <property>
      <name>yarn.scheduler.capacity.root.dev.maximum-capacity</name>
      <value>20</value>
    </property>

    <property>
      <name>yarn.scheduler.capacity.root.dev.user-limit-factor</name>
      <value>1</value>
    </property>

    <property>
      <name>yarn.scheduler.capacity.root.prod.capacity</name>
      <value>70</value>
    </property>

    <property>
      <name>yarn.scheduler.capacity.root.prod.maximum-capacity</name>
      <value>70</value>
    </property>

    <property>
      <name>yarn.scheduler.capacity.root.prod.user-limit-factor</name>
      <value>1</value>
    </property>

    <property>
      <name>yarn.scheduler.capacity.root.queues</name>
      <value>default,dev,prod</value>
    </property>
</configuration>
```
通过上网上查容量调度所有配置的参数的信息：

```java
yarn.scheduler.capacity.root.default.maximum-capacity,队列最大可使用的资源率。
capacity：队列的资源容量（百分比）。 当系统非常繁忙时，应保证每个队列的容量得到满足，而如果每个队列应用程序较少，可将剩余资源共享给其他队列。注意，所有队列的容量之和应小于100。
maximum-capacity：队列的资源使用上限（百分比）。由于存在资源共享，因此一个队列使用的资源量可能超过其容量，而最多使用资源量可通过该参数限制。
user-limit-factor：单个用户最多可以使用的资源因子，默认情况为1，表示单个用户最多可以使用队列的容量不管集群有空闲，如果该值设为5，表示这个用户最多可以使用5capacity的容量。实际上单个用户的使用资源为 min(user-limit-factorcapacity，maximum-capacity)。这里需要注意的是，如果队列中有多个用户的任务，那么每个用户的使用量将稀释。
minimum-user-limit-percent：每个用户最低资源保障（百分比）。任何时刻，一个队列中每个用户可使用的资源量均有一定的限制。当一个队列中同时运行多个用户的应用程序时中，每个用户的使用资源量在一个最小值和最大值之间浮动，其中，最小值取决于正在运行的应用程序数目，而最大值则由minimum-user-limit-percent决定。比如，假设minimum-user-limit-percent为25。当两个用户向该队列提交应用程序时，每个用户可使用资源量不能超过50%，如果三个用户提交应用程序，则每个用户可使用资源量不能超多33%，如果四个或者更多用户提交应用程序，则每个用户可用资源量不能超过25%。
maximum-applications ：集群或者队列中同时处于等待和运行状态的应用程序数目上限，这是一个强限制，一旦集群中应用程序数目超过该上限，后续提交的应用程序将被拒绝，默认值为10000。所有队列的数目上限可通过参数yarn.scheduler.capacity.maximum-applications设置（可看做默认值），而单个队列可通过参数yarn.scheduler.capacity..maximum-applications设置适合自己的值。
maximum-am-resource-percent：集群中用于运行应用程序ApplicationMaster的资源比例上限，该参数通常用于限制处于活动状态的应用程序数目。该参数类型为浮点型，默认是0.1，表示10%。所有队列的ApplicationMaster资源比例上限可通过参数yarn.scheduler.capacity. maximum-am-resource-percent设置（可看做默认值），而单个队列可通过参数yarn.scheduler.capacity.. maximum-am-resource-percent设置适合自己的值。
state ：队列状态可以为STOPPED或者RUNNING，如果一个队列处于STOPPED状态，用户不可以将应用程序提交到该队列或者它的子队列中，类似的，如果ROOT队列处于STOPPED状态，用户不可以向集群中提交应用程序，但正在运行的应用程序仍可以正常运行结束，以便队列可以优雅地退出。
acl_submit_applications：限定哪些Linux用户/用户组可向给定队列中提交应用程序。需要注意的是，该属性具有继承性，即如果一个用户可以向某个队列中提交应用程序，则它可以向它的所有子队列中提交应用程序。配置该属性时，用户之间或用户组之间用“，”分割，用户和用户组之间用空格分割，比如“user1, user2 group1,group2”。
acl_administer_queue：为队列指定一个管理员，该管理员可控制该队列的所有应用程序，比如杀死任意一个应用程序等。同样，该属性具有继承性，如果一个用户可以向某个队列中提交应用程序，则它可以向它的所有子队列中提交应用程序。
```
####分别向dev、prod两个队列提交作业
![-w1440](/img/blog_img/15689825332641.jpg)
![-w1440](/img/blog_img/15689826112145.jpg)
即使任务所需要的资源占满了dev队列的资源也不会使用prod的资源，因为配置了dev队列的最大使用限制。
### 重新修改配置，并重启 yarn ，并观察

```java
<property>
      <name>yarn.scheduler.capacity.root.dev.maximum-capacity</name>
      <value>100</value>
</property>
```
![-w1440](/img/blog_img/15690653980006.jpg)
![-w1440](/img/blog_img/15690655005436.jpg)
我们观察此时Dev 队列使用的资源已经超过了 100%，此时达到了 114%，体现了容量调度的特性，实现了资源的共享
注：maximum-capacity：队列的资源使用上限（百分比）。由于存在资源共享，因此一个队列使用的资源量可能超过其容量，而最多使用资源量可通过该参数限制。
