---
layout:     post
title:     Spark rdd 的例子合集
subtitle:   大数据处理
date:       2019-11-9
author:     Yi Xia
header-img: img/post-bg-map.jpg
catalog: true
tags:
    - Java
---

# Spark rdd 的例子合集
## 1.spark-shell交互式编程
我们的数据集为data1.txt，该数据集包含了某大学计算机系的成绩，数据格式如下所示：

```
Tom,DataBase,80
Tom,Algorithm,50
Tom,DataStructure,60
Jim,DataBase,90
Jim,Algorithm,60
Jim,DataStructure,80
……
```

请根据给定的实验数据，在spark-shell中通过编程来计算以下内容：

```
（1）该系总共有多少学生；
（2）该系共开设来多少门课程；
（3）Tom同学的总成绩平均分是多少；
（4）求每名同学的选修的课程门数；
（5）该系DataBase课程共有多少人选修；
（6）各门课程的平均分是多少；
（7）使用累加器计算共有多少人选了DataBase这门课。
```


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

## 2.Top N

求 TopN
### 输入

```
filetext1：
1,1768,50,155 2,1218, 600,211 3,2239,788,242 4,3101,28,599 5,4899,290,129 6,3110,54,12017,4436,259,877 8,2369,7890,27filetext2：
100,4287,226,233 101,6562,489,124 102,1124,33,17 103,3267,159,179 104,4569,57,125105,1438,37,116
```
### 输出

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

## 3.实现数据去重
对于两个输入文件A和B，编写Spark独立应用程序，对两个文件进行合并，并剔除其中重复的内容，得到一个新文件C。
### 输入

```
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
```

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

## 4.求平均值问题
每个输入文件表示班级学生某个学科的成绩，每行内容由两个字段组成，第一个是学生名字，第二个是学生的成绩；编写Spark独立应用程序求出所有学生的平均成绩，并输出到一个新文件中。
### 输入

```
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
```

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

## 获取平均得分超过4.0的电影列表
在推荐领域有一个著名的开放测试集，下载链接是：http://grouplens.org/datasets/movielens/
该测试集包含三个文件，分别是ratings.dat、sers.dat、movies.dat，具体介绍可阅读：README.txt。请编程实现：通过连接ratings.dat和movies.dat两个文件得到平均得分超过4.0的电影列表，采用的数据集是：ml-1m
### 输入
ratings.dat:(id,电影id，评分，时间戳)用：：符号分割
```
1::1193::5::978300760
1::661::3::978302109
1::914::3::978301968
1::3408::4::978300275
1::2355::5::978824291
1::1197::3::978302268
1::1287::5::978302039
1::2804::5::978300719
2::2628::3::978300051
2::1103::3::978298905
2::2916::3::978299809
2::3468::5::978298542
3::2167::5::978297600
3::1580::3::978297663
3::3619::2::978298201
3::260::5::978297512
3::2858::4::978297039
4::1214::4::978294260
4::1036::4::978294282
4::260::5::978294199
4::2028::5::978294230
```
movies.dat:（电影id，电影名，电影类型）用：：符号分割

```
1::Toy Story (1995)::Animation|Children's|Comedy
2::Jumanji (1995)::Adventure|Children's|Fantasy
3::Grumpier Old Men (1995)::Comedy|Romance
4::Waiting to Exhale (1995)::Comedy|Drama
```
### 输出
![-w1440](/img/blog_img/15732946267071.jpg)

部分 4 分以上的电影
```
(Bonnie and Clyde (1967),4.096209912536443)
(American Movie (1999),4.013559322033898)
(Harmonists, The (1997),4.142857142857143)
(Bells, The (1926),4.5)
(Toy Story (1995),4.146846413095811)
(Ayn Rand: A Sense of Life (1997),4.125)
(Nights of Cabiria (Le Notti di Cabiria) (1957),4.207207207207207)
(Maya Lin: A Strong Clear Vision (1994),4.101694915254237)
(My Life as a Dog (Mitt liv som hund) (1985),4.1454545454545455)
(West Side Story (1961),4.057818659658344)
(Three Days of the Condor (1975),4.040752351097178)
(Crumb (1994),4.063136456211812)
(Raging Bull (1980),4.1875923190546525)
(Three Colors: Red (1994),4.227544910179641)
(Manon of the Spring (Manon des sources) (1986),4.259090909090909)
(Who's Afraid of Virginia Woolf? (1966),4.074074074074074)
(Wallace & Gromit: The Best of Aardman Animation (1996),4.426940639269406)
(Body Heat (1981),4.031746031746032)
(Shall We Dance? (1937),4.1657142857142855)
(Red Sorghum (Hong Gao Liang) (1987),4.015384615384615)
(Sunset Blvd. (a.k.a. Sunset Boulevard) (1950),4.491489361702127)
(Farewell My Concubine (1993),4.082677165354331)
(Callej�n de los milagros, El (1995),4.5)
(Return with Honor (1998),4.4)
(Eighth Day, The (Le Huiti�me jour ) (1996),4.25)
(Ponette (1996),4.068493150684931)
(Double Indemnity (1944),4.415607985480944)
(To Kill a Mockingbird (1962),4.425646551724138)
(M*A*S*H (1970),4.124658780709736)
(Untouchables, The (1987),4.007985803016859)
(General, The (1927),4.368932038834951)
(Godfather, The (1972),4.524966261808367)
(Star Wars: Episode V - The Empire Strikes Back (1980),4.292976588628763)
(Matewan (1987),4.141242937853107)
(Persuasion (1995),4.055865921787709)
(How Green Was My Valley (1941),4.02803738317757)
(Stalag 17 (1953),4.228426395939087)
(For All Mankind (1989),4.444444444444445)
(Gandhi (1982),4.141579731743666)
(This Is Spinal Tap (1984),4.179785330948121)
(Ulysses (Ulisse) (1954),5.0)
(Once Upon a Time in the West (1969),4.023121387283237)
(Producers, The (1968),4.144376899696049)

```
### 源码及解析

```scala
import scala.math.random
import org.apache.spark._
import org.apache.spark.{SparkConf, SparkContext}
object movielens {

  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("movie").setMaster("local")
    val sc = new SparkContext(conf)
    val movieRate = sc.textFile("file:///Users/summerone/Desktop/spark/ml-1m/ratings.dat")

    //求每个电影id的影评平均数，平均数大于四
    val movieScore = movieRate.filter(line=>line.split("::").length==4).map(line=>{
      val st = line.split("::")
      (st(1).toInt,st(2).toInt) //截取电影id和评分
    }).groupByKey().map(line=>{   //计算每个电影的平均分
      var num = 0
      var total = 0.0
      for(i <- line._2){
        total = total + i
        num = num + 1
      }
      val avg = total/num
      (line._1,avg)
    }).sortByKey().filter(line=>line._2>4)  //过滤出平均分大于4.0的电影

    //获取电影id和电影名
    val movieinfo = sc.textFile("file:///Users/summerone/Desktop/spark/ml-1m/movies.dat")
    val moviessss = movieinfo.filter(line=>line.split("::").length==3).map(line=>{
      val ss = line.split("::")
      (ss(0).toInt,ss(1))
    })
    //使用join连接两个数据集的信息
    val reslut = moviessss.join(movieScore).map(line=>{
      (line._2._1,line._2._2)
    })

    reslut.collect().foreach(println)
    sc.stop()
  }


}

```


