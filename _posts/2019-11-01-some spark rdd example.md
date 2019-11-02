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
![-w1315](/img/blog_img/15726195431952.jpg)
