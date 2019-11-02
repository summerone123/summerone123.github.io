---
layout:     post
title:   Spark with IDEA
subtitle:   大数据处理
date:       2019-10-18
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - Java
---

# Spark with IDEA
IDEA 全称 IntelliJ IDEA，是java语言开发的集成环境，IntelliJ在业界被公认为最好的Java开发工具之一, IDEA是JetBrains公司的产品,现在有逐步取代老牌Java开发工具Eclipse的趋势.那本人也是从Eclipse 转到IDEA.那刚转换过来时,确实很不适应,不过好在坚持使用了几天后,确实感觉IntelliJ IDEA比Eclipse更加智能.
Maven项目对象模型(POM)，是一个项目管理工具可以通过一小段描述信息来管理项目的构建，报告和文档的软件。那我们想要在IDEA中使用Maven得进行一些配置,那接下来
我们具体看一下是如何配置使用的?
## Maven
因为搭建这个Spark with IDEA，以前从没摸过 maven，这次借此来学了一下基础知识
具体的介绍见我另一篇 blog
[http://summerone.xyz/2019/10/18/IDEA-with-Maven/](http://summerone.xyz/2019/10/18/IDEA-with-Maven/)
maven是一款优秀的服务构建工具，基于约定优于配置原则，提供标准的服务构建流程。maven的优点不仅限于服务构建，使用maven能够做到高效的依赖管理，并且提供有中央仓库可以完成绝大多数依赖的下载使用。


![-w680](/img/blog_img/15724225884585.jpg)

maven自身提供有丰富的插件，可以在不使用额外插件的条件下完成服务的编译、测试、打包、部署等服务构建流程，即maven对服务的构建过程是通过多个插件完成的，且maven已经自定义了插件的行为。可以理解为每一个插件都是对接口的实现，可以自定义插件，以完成自定义功能，例如完成对不同编程语言的服务构建过程。不过相对于gradle的自定义插件行为，maven的实现过程略微复杂。

## Spark with IDEA搭建环境
- 首先安装Scala插件，File->Settings->Plugins，搜索出Scla插件，点击Install安装；（速度比较慢，可以试试其他的源，不用官网的）
![-w965](/img/blog_img/15724978721026.jpg)

- File->New Project->maven，新建一个Maven项目，填写GroupId和ArtifactId
![-w963](/img/blog_img/15724979023965.jpg)
![-w960](/img/blog_img/15724979415442.jpg)

> 这里给大家普及一下这两个的作用：
> groupid和artifactId被统称为“坐标”是为了保证项目唯一性而提出的，如果你要把你项目弄到maven本地仓库去，你想要找到你的项目就必须根据这两个id去查找。 
　　groupId一般分为多个段，这里我只说两段，第一段为域，第二段为公司名称。域又分为org、com、cn等等许多，其中org为非营利组织，com为商业组织。举个apache公司的tomcat项目例子：这个项目的groupId是org.apache，它的域是org（因为tomcat是非营利项目），公司名称是apache，artigactId是tomcat。 
　　artifact你把它理解成“生成的东西”就差不多了。这个词强调的是这是你软件生产过程中某一步的产生物，不像程序本身，或者是配置文件这些，是你手写出来的。
　　






- 为我们的maven工程，导入sdk环境

>  特别注意这里的scala 环境就是你的 spark 环境的 scala 版本，不要下错了

![-w1118](/img/blog_img/15724983946809.jpg)

- 这个导入完毕之后，基本环境就搭建完毕了，接下来，修改pom文件，增添我们的spark-core。

```shell
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>test</groupId>
    <artifactId>test</artifactId>
    <version>1.0-SNAPSHOT</version>
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <spark.version>2.4.4</spark.version>
        <scala.version>2.11.12</scala.version>
        <hadoop.version>2.8.5</hadoop.version>
    </properties>

    <dependencies>
        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-core_2.11 -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_${scala.version}</artifactId>
            <version>${spark.version}</version>
<!--            <scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-hive_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.7.3</version>
        </dependency>

        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-mllib_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.39</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>
    </dependencies>

    <repositories>
        <repository>
            <id>central</id>
            <name>Maven Repository Switchboard</name>
            <layout>default</layout>
            <url>http://repo2.maven.org/maven2</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>

    <build>
        <sourceDirectory>src/main/scala</sourceDirectory>
        <testSourceDirectory>src/test/scala</testSourceDirectory>

        <plugins>
            <plugin>
                <!-- MAVEN 编译使用的JDK版本 -->
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.5</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

> 其中试了几次，spark 源码的包也没法引入，最终直接把 spark 源码里的 jars 包直接加到了 library 里，简单粗暴🤟。
![-w1118](/img/blog_img/15724995365553.jpg)


## 测试

```scala
import scala.math.random
import org.apache.spark._

object SparkPi {
  def main(args: Array[String]) {
    //val conf = new SparkConf().setAppName("Spark Pi").setMaster("spark://192.168.189.136:7077").setJars(List("D:\\scala\\sparkjar\\sparktest.jar"))
    //val spark = new SparkContext("spark://master:7070", "Spark Pi", "F:\\soft\\spark\\spark-1.1.0-bin-hadoop2.4", List("out\\artifacts\\sparkTest_jar\\sparkTest.jar"))
    val conf = new SparkConf().setAppName("Spark Pi").setMaster("local")//主要是这句
    val spark = new SparkContext(conf)
    val slices = if (args.length > 0) args(0).toInt else 2
    val n = 100000 * slices
    val count = spark.parallelize(1 to n, slices).map { i =>
      val x = random * 2 - 1
      val y = random * 2 - 1
      if (x * x + y * y < 1) 1 else 0
    }.reduce(_ + _)
    println("Pi is roughly " + 4.0 * count / n)
    spark.stop()
  }
}

```

- File->Project Structure->Artifacts，新建一个Jar->From modules with dependencies...，选择Main Class：

![-w1118](/img/blog_img/15724993445203.jpg)

- Build->Build Artifacts...，生成jar，稍后放到集群上运行也是可以
 ![-w1552](/img/blog_img/15724994497622.jpg)

![-w1440](/img/blog_img/15724233597915.jpg)

- 然后再运行，成功！
![-w1552](/img/blog_img/15724235313794.jpg)
![-w594](/img/blog_img/15724236824857.jpg)

```scala
import org.apache.spark.{SparkConf, SparkContext}object TopN {  def main(args: Array[String]): Unit = {    val conf = new SparkConf().setAppName("TopN").setMaster("local")    val sc = new SparkContext(conf)    sc.setLogLevel("ERROR")    val lines = sc.textFile("hdfs://localhost:9000/user/hadoop/spark/mycode/rdd/examples",2)    var num = 0;    val result = lines.filter(line => (line.trim().length > 0) && (line.split(",").length == 4))      .map(_.split(",")(2))      .map(x => (x.toInt,""))      .sortByKey(false)      .map(x => x._1).take(5)      .foreach(x => {        num = num + 1        println(num + "\t" + x)      })  }}
```
![-w1552](/img/blog_img/15724235760541.jpg)



![-w1552](/img/blog_img/15724234435017.jpg)

![-w1440](/img/blog_img/15724233597915.jpg)

## 坑
- 注意集群的 scala 的版本和你的本机的 scala 的版本应该是一样的
![-w777](/img/blog_img/15726061209304.jpg)

- java.lang.NoClassDefFoundError: org/apache/spark/streaming/dstream/DStream 

```shell
<dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming_2.11</artifactId>
        <version>2.3.1</version>
        <!--<scope>provided</scope>-->
    </dependency>
```
经过查阅资料，这个错误的产生原因是，Maven库中SparkSteaming的Scope默认是Provided的，因此在运行的时候实际上并不会包含SparkSteaming相关的Jar包。因此，把Scope删去，或者改为Compile即可（默认即为Compile）。

