---
layout:     post
title:     Spark rdd 的例子合集
subtitle:   大数据处理
date:       2019-10-18
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - Java
---
# Spark rdd 的例子合集
## 1.spark-shell交互式编程
我们的数据集为data1.txt，该数据集包含了某大学计算机系的成绩，数据格式如下所示：
Tom,DataBase,80
Tom,Algorithm,50
Tom,DataStructure,60
Jim,DataBase,90
Jim,Algorithm,60
Jim,DataStructure,80
……
请根据给定的实验数据，在spark-shell中通过编程来计算以下内容：
（1）该系总共有多少学生；
（2）该系共开设来多少门课程；
（3）Tom同学的总成绩平均分是多少；
（4）求每名同学的选修的课程门数；
（5）该系DataBase课程共有多少人选修；
（6）各门课程的平均分是多少；
（7）使用累加器计算共有多少人选了DataBase这门课。

### 分析
- 主要是熟悉一下这种函数式编程，以前接触 python 的 lambda，这一次对于这种函数式编程有了更多的理解
- 交互式的 spark-shell 手动实现了一下，但是截图太多了，索性直接在配置好的 idea 环境下又跑了一遍

### 代码
```scala
import scala.math.random
import org.apache.spark._

object student {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("Student").setMaster("local")
    val sc = new SparkContext(conf)
    sc.setLogLevel("ERROR")
    val data = sc.textFile("file:///Users/summerone/Desktop/spark/Data01.txt")
    var num = 0;
//    val wordcount=data.flatMap(_.split(' ')).map((_,1)).groupByKey
    println("学生一共选了课程数："+data.count())
    val student = data.map(row=>row.split(",")(0))
    val distinct_stu = student.distinct() //去重操作
    println("学生一共的数目："+distinct_stu.count) //取得总数


    val course = data.map(row=>row.split(",")(1))//根据，切分的每行数据的第二列进行map
    val distinct_cor = course.distinct()//去重
    println("该系共开设的课程："+distinct_cor.count)//取总数

    println("Tom所选的课程：")
    val tom = data.filter(row=>row.split(",")(0)=="Tom")
    tom.foreach(println)
    tom.map(row=>(row.split(",")(0),row.split(",")(2).toInt))
      .mapValues(x=>(x,1))//mapValues是对值的操作,不操作key使数据变成(Tom,（26,1）)
      .reduceByKey((x,y) => (x._1+y._1,x._2 + y._2))//接着需要按key进行reduce，让key合并当将Tom进行reduce后 这里的(x,y) 表示的是(26,1)(12,1)
      .mapValues(x => (x._1 / x._2))//接着要对value进行操作，用mapValues()就行啦
      .foreach(x => {
      println(x)
    })

    //每名同学选修的课程门数
    val course_each = data.map(row=>(row.split(",")(0),row.split(",")(1)))
    course_each.mapValues(x => (x,1))//数据变为(Tom,(DataBase,1)),(Tom,(Algorithm,1)),(Tom,(OperatingSystem,1)),(Tom,(Python,1)),(Tom,(Software,1))
      .reduceByKey((x,y) => (" ",x._2 + y._2))//数据变为(Tom,( ,5))
      .mapValues(x =>x._2)//数据变为(Tom, 5)
      .foreach(println)

    //计算选DataBase的人数
    val Data_num = data.filter(row=>row.split(",")(1)=="DataBase")
    println(Data_num.count())


    //计算各门平均分
    val average = data.map(row=>(row.split(",")(1),row.split(",")(2).toInt))
    average.mapValues(x=>(x,1)).reduceByKey((x,y) => (x._1+y._1,x._2 + y._2)).mapValues(x => (x._1 / x._2)).foreach(println)


    //使用累加器计算共有多少人选了 DataBase 这门课。
    val pare = data.filter(row=>row.split(",")(1)=="DataBase").map(row=>(row.split(",")(1),1))
    val accum = sc.longAccumulator("Yi_Xia Accumulator")//累加器函数Accumulator
    pare.values.foreach(x => accum.add(x))
    println(accum.value)
  }
}

```
### 程序及其结果
![-w1440](/img/blog_img/15726189633645.jpg)

## 2.实现数据去重
对于两个输入文件A和B，编写Spark独立应用程序，对两个文件进行合并，并剔除其中重复的内容，得到一个新文件C。
### 输入
输入文件A的样例如下：
20170101    x
20170102    y
20170103    x
20170104    y
20170105    z
20170106    z
输入文件B的样例如下：
20170101    y
20170102    y
20170103    x
20170104    z
20170105    y
### 输出
![-w1440](/img/blog_img/15732790075604.jpg)

### 源码及解析

```scala
import org.apache.spark.{SparkConf, SparkContext}
object merge {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setMaster("local").setAppName("reduce")
    val sc = new SparkContext(conf)
    sc.setLogLevel("ERROR")
    //获取数据
    val two = sc.textFile("file:///Users/summerone/Desktop/spark/merge")
    two.filter(_.trim().length>0) //需要有空格。
      .map(line=>(line.trim,""))//全部值当key，(key value,"")
      .groupByKey()//groupByKey,过滤重复的key value ，发送到总机器上汇总
      .sortByKey() //按key value的自然顺序排序
      .keys.collect().foreach(println) //所有的keys变成数组再输出
    

  }
}

```

## 3.求平均值问题
每个输入文件表示班级学生某个学科的成绩，每行内容由两个字段组成，第一个是学生名字，第二个是学生的成绩；编写Spark独立应用程序求出所有学生的平均成绩，并输出到一个新文件中。
### 输入
Algorithm成绩：
小明 92
小红 87
小新 82
小丽 90
Database成绩：
小明 95
小红 81
小新 89
小丽 85
Python成绩：
小明 82
小红 83
小新 94
小丽 91
### 输出
![-w1440](/img/blog_img/15732794596327.jpg)
### 源码解解析

```scala
import org.apache.spark.{SparkConf, SparkContext}

object pingjunzhi {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setMaster("local").setAppName("reduce")
    val sc = new SparkContext(conf)
    sc.setLogLevel("ERROR")

    val fourth = sc.textFile("file:///Users/summerone/Desktop/spark/pingjunzhi")

    val res = fourth.filter(_.trim().length>0).map(line=>(line.split(" ")(0).trim(),line.split(" ")(1).trim().toInt)).groupByKey().map(x => {
      var num = 0.0
      var sum = 0
      for(i <- x._2){
        sum = sum + i
        num = num +1
      }
      val avg = sum/num
      val format = f"$avg%1.2f".toDouble
      (x._1,format)
    }).collect.foreach(x => println(x._1+"\t"+x._2))
  }
}
```

## 4.Top N

求 TopN
### 输入
filetext1：
1,1768,50,155 2,1218, 600,211 3,2239,788,242 4,3101,28,599 5,4899,290,129 6,3110,54,12017,4436,259,877 8,2369,7890,27filetext2：
100,4287,226,233 101,6562,489,124 102,1124,33,17 103,3267,159,179 104,4569,57,125105,1438,37,116### 输出

![-w1552](/img/blog_img/15724234435017.jpg)将 打的jar 包在集群上运行
![-w1440](/img/blog_img/15724233597915.jpg)```scala
import org.apache.spark.{SparkConf, SparkContext}
object TopN {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("TopN").setMaster("local")
    val sc = new SparkContext(conf)
    sc.setLogLevel("ERROR")
    val lines = sc.textFile("hdfs://10.113.10.11:9000/spark/TopN",2)
    var num = 0;
    val result = lines.filter(line => (line.trim().length > 0) && (line.split(",").length == 4))
      .map(_.split(",")(2))
      .map(x => (x.toInt,""))
      .sortByKey(false)
      .map(x => x._1).take(5)
      .foreach(x => {
        num = num + 1
        println(num + "\t" + x)
      })
  }
}

```
