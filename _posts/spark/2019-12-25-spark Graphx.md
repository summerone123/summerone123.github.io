---
layout:     post
title:     Spark Graphx
subtitle:   大数据处理
date:       2019-12-24
author:     Yi Xia
header-img: img/post-bg-map.jpg
catalog: true
tags:
    - 大数据处理
---

# spark Graphx
> 图处理技术包括图数据库(Neo4j、Titan、OrientDB、DEX和InfiniteGraph等)、图数据查询(对图数据库中的内容进行查询)、图数据分析(Google Pregel、Spark GraphX、GraphLab等图计算软件)和图数据可视化(OLTP风格的图数据库或者OLAP风格的图数据分析系统（或称为图计算软件）)。

## spark Graphx 简介
![-w571](/img/blog_img/15772346412240.jpg)

    
    Spark GraphX是一个分布式图处理框架，它是基于Spark平台提供对图计算和图挖掘简洁易用的而丰富的接口，极大的方便了对分布式图处理的需求。 
那么什么是图，都计算些什么？众所周知社交网络中人与人之间有很多关系链，例如Twitter、Facebook、微博和微信等，数据中出现网状结构关系都需要图计算。

    GraphX是一个新的Spark API，它用于图和分布式图(graph-parallel)的计算。GraphX通过引入弹性分布式属性图（Resilient Distributed Property Graph）： 顶点和边均有属性的有向多重图，来扩展Spark RDD。为了支持图计算，GraphX开发了一组基本的功能操作以及一个优化过的Pregel API。另外，GraphX也包含了一个快速增长的图算法和图builders的集合，用以简化图分析任务。

    从社交网络到语言建模，不断增长的数据规模以及图形数据的重要性已经推动了许多新的分布式图系统的发展。 通过限制计算类型以及引入新的技术来切分和分配图，这些系统可以高效地执行复杂的图形算法，比一般的分布式数据计算（data-parallel，如spark、MapReduce）快很多。
    
     我们如何处理数据取决于我们的目标，有时同一原始数据可能会处理成许多不同表和图的视图，并且图和表之间经常需要能够相互移动。如下图所示：
     ![-w616](/img/blog_img/15772359518969.jpg)

## spark Graphx基本常识
在此仅介绍我遇到的比较简单的常识，更加深入的知识请大家自行问度娘
###vertices、edges以及triplets
 vertices、edges以及triplets是GraphX中三个非常重要的概念。
 ![-w588](/img/blog_img/15772375151741.jpg)
 Edge类中包含源顶点id，目标顶点id以及边的属性。所以从源代码中我们可以知道，triplets既包含了边属性也包含了源顶点的id和属性、目标顶点的id和属性。
 
-  vertices
在GraphX中，vertices对应着名称为VertexRDD的RDD。这个RDD有顶点id和顶点属性两个成员变量。它的源码如下所示：

```
abstract class VertexRDD[VD](sc: SparkContext, deps: Seq[Dependency[_]]) extends RDD[(VertexId, VD)](sc, deps)
```
从源码中我们可以看到，VertexRDD继承自RDD[(VertexId, VD)]，这里VertexId表示顶点id，VD表示顶点所带的属性的类别。这从另一个角度也说明VertexRDD拥有顶点id和顶点属性。

- edges
在GraphX中，edges对应着EdgeRDD。这个RDD拥有三个成员变量，分别是源顶点id、目标顶点id以及边属性。它的源码如下所示：

```
abstract class EdgeRDD[ED](sc: SparkContext, deps: Seq[Dependency[_]]) extends RDD[Edge[ED]](sc, deps)
```
从源码中我们可以看到，EdgeRDD继承自RDD[Edge[ED]]，即类型为Edge[ED]的RDD。 

- triplets（也就是三元组）
在GraphX中，triplets对应着EdgeTriplet。它是一个三元组视图，这个视图逻辑上将顶点和边的属性保存为一个RDD[EdgeTriplet[VD, ED]]。可以通过下面的Sql表达式表示这个三元视图的含义:

```
SELECT
	src.id ,
	dst.id ,
	src.attr ,
	e.attr ,
	dst.attr
FROM
	edges AS e
LEFT JOIN vertices AS src ,
 vertices AS dst ON e.srcId = src.Id
AND e.dstId = dst.Id

```
## spark Graphx简单实践
运行图计算程序
![-w606](/img/blog_img/15772365391674.jpg)


```scala
import org.apache.log4j.{Level, Logger}
import org.apache.spark.{SparkContext, SparkConf}
import org.apache.spark.graphx._
import org.apache.spark.rdd.RDD

object test_graphx {
  def main(args: Array[String]) {
    //屏蔽日志
    Logger.getLogger("org.apache.spark").setLevel(Level.WARN)
    //设置运行环境
    val conf = new SparkConf().setAppName("GraphXTest").setMaster("local[*]")
    val sc = new SparkContext(conf)
    //设置顶点和边，注意顶点和边都是用元组定义的 Array
    //顶点的数据类型是 VD:(String,Int)
    val vertexArray = Array(
      (1L, ("Alice", 28)),
      (2L, ("Bob", 27)),
      (3L, ("Charlie", 65)),
      (4L, ("David", 42)),
      (5L, ("Ed", 55)),
      (6L, ("Fran", 50))
    )
    //边的数据类型 ED:Int
    val edgeArray = Array(
      Edge(2L, 1L, 7),
      Edge(2L, 4L, 2),
      Edge(3L, 2L, 4),
      Edge(3L, 6L, 3),
      Edge(4L, 1L, 1),
      Edge(5L, 2L, 2),
      Edge(5L, 3L, 8),
      Edge(5L, 6L, 3)
    )
    //构造 vertexRDD 和 edgeRDD
    val vertexRDD: RDD[(Long, (String, Int))] = sc.parallelize(vertexArray)
    val edgeRDD: RDD[Edge[Int]] = sc.parallelize(edgeArray)
    //构造图 Graph[VD,ED]
    val graph: Graph[(String, Int), Int] = Graph(vertexRDD, edgeRDD)
    println("***********************************************")
    println("属性演示")
    println("**********************************************************")
    println("找出图中年龄大于 30 的顶点：")
    graph.vertices.filter { case (id, (name, age)) => age > 30 }.collect.foreach {
      case (id, (name, age)) => println(s"$name is $age")
    }
    graph.triplets.foreach(t => println(s"triplet:${t.srcId},${t.srcAttr},${t.dstId},${t.dstAttr},${t.attr}"))
    //边操作：找出图中属性大于 5 的边
    println("找出图中属性大于 5 的边：")
    graph.edges.filter(e => e.attr > 5).collect.foreach(e => println(s"${e.srcId} to ${e.dstId} att ${e.attr}"))
    println
    //triplets 操作，((srcId, srcAttr), (dstId, dstAttr), attr)
    println("列出边属性>5 的 tripltes：")
    for (triplet <- graph.triplets.filter(t => t.attr > 5).collect) {
      println(s"${triplet.srcAttr._1} likes ${triplet.dstAttr._1}")
    }
    println
    //Degrees 操作
    println("找出图中最大的出度、入度、度数：")

    def max(a: (VertexId, Int), b: (VertexId, Int)): (VertexId, Int) = {
      if (a._2 > b._2) a else b
    }

    println("max of outDegrees:" + graph.outDegrees.reduce(max) + ", max of inDegrees:" + graph.inDegrees.reduce(max) + ", max of Degrees:" +
      graph.degrees.reduce(max))
    println
    println("**********************************************************")
    println("转换操作")
    println("**********************************************************")
    println("顶点的转换操作，顶点 age + 10：")
    graph.mapVertices { case (id, (name, age)) => (id, (name,
      age + 10))
    }.vertices.collect.foreach(v => println(s"${v._2._1} is ${v._2._2}"))
    println
    println("边的转换操作，边的属性*2：")
    graph.mapEdges(e => e.attr * 2).edges.collect.foreach(e => println(s"${e.srcId} to ${e.dstId} att ${e.attr}"))
    println
    println("**********************************************************")
    println("结构操作")
    println("**********************************************************")
    println("顶点年纪>30 的子图：")
    val subGraph = graph.subgraph(vpred = (id, vd) => vd._2 >= 30)
    println("子图所有顶点：")
    subGraph.vertices.collect.foreach(v => println(s"${v._2._1} is ${v._2._2}"))
    println
    println("子图所有边：")
    subGraph.edges.collect.foreach(e => println(s"${e.srcId} to ${e.dstId} att ${e.attr}"))
    println
    println("**********************************************************")
    println("连接操作")
    println("**********************************************************")
    val inDegrees: VertexRDD[Int] = graph.inDegrees
    case class User(name: String, age: Int, inDeg: Int, outDeg: Int)
    //创建一个新图，顶点 VD 的数据类型为 User，并从 graph 做类型转换
    val initialUserGraph: Graph[User, Int] = graph.mapVertices { case (id, (name, age))
    => User(name, age, 0, 0)
    }
    //initialUserGraph 与 inDegrees、outDegrees（RDD）进行连接，并修改 initialUserGraph中 inDeg 值、outDeg 值
    val userGraph = initialUserGraph.outerJoinVertices(initialUserGraph.inDegrees) {
      case (id, u, inDegOpt) => User(u.name, u.age, inDegOpt.getOrElse(0), u.outDeg)
    }.outerJoinVertices(initialUserGraph.outDegrees) {
      case (id, u, outDegOpt) => User(u.name, u.age, u.inDeg, outDegOpt.getOrElse(0))
    }
    println("连接图的属性：")
    userGraph.vertices.collect.foreach(v => println(s"${v._2.name} inDeg: ${v._2.inDeg} outDeg: ${v._2.outDeg}"))
    println
    println("出度和入读相同的人员：")
    userGraph.vertices.filter {
      case (id, u) => u.inDeg == u.outDeg
    }.collect.foreach {
      case (id, property) => println(property.name)
    }
    println
    println("**********************************************************")
    println("聚合操作")
    println("**********************************************************")
    println("找出年纪最大的follower：")
    val oldestFollower: VertexRDD[(String, Int)] = userGraph.aggregateMessages[(String,
      Int)](
      // 将源顶点的属性发送给目标顶点，map 过程
      et => et.sendToDst((et.srcAttr.name,et.srcAttr.age)),
      // 得到最大follower，reduce 过程
      (a, b) => if (a._2 > b._2) a else b
    )
    userGraph.vertices.leftJoin(oldestFollower) { (id, user, optOldestFollower) =>
      optOldestFollower match {
        case None => s"${user.name} does not have any followers."
        case Some((name, age)) => s"${name} is the oldest follower of ${user.name}."
      }
    }.collect.foreach { case (id, str) => println(str) }
    println
    println("**********************************************************")
    println("聚合操作")
    println("**********************************************************")
    println("找出距离最远的顶点，Pregel基于对象")
    val g = Pregel(graph.mapVertices((vid, vd) => (0, vid)), (0, Long.MinValue), activeDirection = EdgeDirection.Out)(
      (id: VertexId, vd: (Int, Long), a: (Int, Long)) => math.max(vd._1, a._1) match {
        case vd._1 => vd
        case a._1 => a
      },
      (et: EdgeTriplet[(Int, Long), Int]) => Iterator((et.dstId, (et.srcAttr._1 + 1+et.attr, et.srcAttr._2))) ,
      (a: (Int, Long), b: (Int, Long)) => math.max(a._1, b._1) match {
        case a._1 => a
        case b._1 => b
      }
    )
    g.vertices.foreach(m=>println(s"原顶点${m._2._2}到目标顶点${m._1},最远经过${m._2._1}步"))

    // 面向对象
    val g2 = graph.mapVertices((vid, vd) => (0, vid)).pregel((0, Long.MinValue), activeDirection = EdgeDirection.Out)(
      (id: VertexId, vd: (Int, Long), a: (Int, Long)) => math.max(vd._1, a._1) match {
        case vd._1 => vd
        case a._1 => a
      },
      (et: EdgeTriplet[(Int, Long), Int]) => Iterator((et.dstId, (et.srcAttr._1 + 1, et.srcAttr._2))),
      (a: (Int, Long), b: (Int, Long)) => math.max(a._1, b._1) match {
        case a._1 => a
        case b._1 => b
      }
    )
    //    g2.vertices.foreach(m=>println(s"原顶点${m._2._2}到目标顶点${m._1},最远经过${m._2._1}步"))
    sc.stop()
  }

}

```

图运算结果

```
***********************************************
属性演示
**********************************************************
找出图中年龄大于 30 的顶点：
David is 42
Ed is 55
Fran is 50
Charlie is 65
triplet:5,(Ed,55),3,(Charlie,65),8
triplet:5,(Ed,55),6,(Fran,50),3
triplet:2,(Bob,27),1,(Alice,28),7
triplet:2,(Bob,27),4,(David,42),2
triplet:3,(Charlie,65),2,(Bob,27),4
triplet:3,(Charlie,65),6,(Fran,50),3
triplet:4,(David,42),1,(Alice,28),1
triplet:5,(Ed,55),2,(Bob,27),2
找出图中属性大于 5 的边：
2 to 1 att 7
5 to 3 att 8

列出边属性>5 的 tripltes：
Bob likes Alice
Ed likes Charlie

找出图中最大的出度、入度、度数：
max of outDegrees:(5,3), max of inDegrees:(2,2), max of Degrees:(2,4)

**********************************************************
转换操作
**********************************************************
顶点的转换操作，顶点 age + 10：
4 is (David,52)
1 is (Alice,38)
5 is (Ed,65)
6 is (Fran,60)
2 is (Bob,37)
3 is (Charlie,75)

边的转换操作，边的属性*2：
2 to 1 att 14
2 to 4 att 4
3 to 2 att 8
3 to 6 att 6
4 to 1 att 2
5 to 2 att 4
5 to 3 att 16
5 to 6 att 6

**********************************************************
结构操作
**********************************************************
顶点年纪>30 的子图：
子图所有顶点：
David is 42
Ed is 55
Fran is 50
Charlie is 65

子图所有边：
3 to 6 att 3
5 to 3 att 8
5 to 6 att 3

**********************************************************
连接操作
**********************************************************
连接图的属性：
David inDeg: 1 outDeg: 1
Alice inDeg: 2 outDeg: 0
Ed inDeg: 0 outDeg: 3
Fran inDeg: 2 outDeg: 0
Bob inDeg: 2 outDeg: 2
Charlie inDeg: 1 outDeg: 2

出度和入读相同的人员：
David
Bob

**********************************************************
聚合操作
**********************************************************
找出年纪最大的follower：
Bob is the oldest follower of David.
David is the oldest follower of Alice.
Ed does not have any followers.
Charlie is the oldest follower of Fran.
Charlie is the oldest follower of Bob.
Ed is the oldest follower of Charlie.

**********************************************************
聚合操作
**********************************************************
找出距离最远的顶点，Pregel基于对象
原顶点5到目标顶点1,最远经过22步
原顶点5到目标顶点5,最远经过0步
原顶点5到目标顶点3,最远经过9步
原顶点5到目标顶点4,最远经过17步
原顶点5到目标顶点6,最远经过13步
原顶点5到目标顶点2,最远经过14步
```
### 构件图的过程及具体说明
构建图的过程很简单，分为三步，它们分别是构建边EdgeRDD、构建顶点VertexRDD、生成Graph对象。

- 引入相关文件

```scala
import org.apache.log4j.{Level, Logger}
import org.apache.spark.{SparkContext, SparkConf}
import org.apache.spark.graphx._
import org.apache.spark.rdd.RDD
```

- 有很多方式从一个原始文件、RDD构造一个属性图。最一般的方法是利用Graph object。下面的代码从RDD集合生成属性图。（具体的三个属性的介绍见 **spark Graphx基本常识**）

```
al conf = new SparkConf().setAppName("GraphXTest").setMaster("local[*]")
    val sc = new SparkContext(conf)
    //设置顶点和边，注意顶点和边都是用元组定义的 Array
    //顶点的数据类型是 VD:(String,Int)
    val vertexArray = Array(
      (1L, ("Alice", 28)),
      (2L, ("Bob", 27)),
      (3L, ("Charlie", 65)),
      (4L, ("David", 42)),
      (5L, ("Ed", 55)),
      (6L, ("Fran", 50))
    )
    //边的数据类型 ED:Int
    val edgeArray = Array(
      Edge(2L, 1L, 7),
      Edge(2L, 4L, 2),
      Edge(3L, 2L, 4),
      Edge(3L, 6L, 3),
      Edge(4L, 1L, 1),
      Edge(5L, 2L, 2),
      Edge(5L, 3L, 8),
      Edge(5L, 6L, 3)
    )
    //构造 vertexRDD 和 edgeRDD
    val vertexRDD: RDD[(Long, (String, Int))] = sc.parallelize(vertexArray)
    val edgeRDD: RDD[Edge[Int]] = sc.parallelize(edgeArray)
    //构造图 Graph[VD,ED]
    val graph: Graph[(String, Int), Int] = Graph(vertexRDD, edgeRDD)
```

- 利用所构建的图进行一定的统计运算操作及挖掘
在构建好图之后，运用 RDD 的操作，我所进行的图数据挖掘主要如下

```

**属性操作**
1. 找出图中属性值（年龄）大于 30 的顶点，并将其triplets进行输出（即其属性值相关联的三元组）
2. 找出图中属性大于 5 的边（找到满足一定条件的边）
3. 列出边属性>5 的 tripltes（通过边属性筛选对应符合条件的三元组）
4. 找出图中最大的出度、入度、度数

**转换操作**
1. 对所有的定点属性进行操作（顶点的转换操作，顶点 age + 10）
2. 对所有的边属性进行操作（边的转换操作，边的属性*2）

**结构操作**
1. 挖掘符合条件的子图：顶点年纪>30 的子图
2. 查询子图所有顶点
3. 查询子图所有边

**连接操作**
1. 利用outerJoinVertices将两个 RDD 进行连接
2. 寻找连接后出入度相同的子图

**聚合操作**
1. 找出每个节点中某属性最大的从节点（找出年纪最大的follower）
2. 找出距离最远的顶点，Pregel基于对象，并计算所需要的步数（一种图计算框架，Spark在其graphX组件中提供了pregel API,让我们可以用pregel的计算框架来处理spark上的图数据。可以寻找最短路径）


```


## 小结
因为在上一次大数据存储大作业中，我实践的是图数据库，并做了一些查询等操作，所以此次对图的处理也比较感兴趣。在对 Spark Graphx 进行实践的过程中，逐渐发现 spark Graphx 对于图计算的便捷操作，如上述我进行的一些挖掘子图，聚合操作和连接操作等，对于图的度的查询也非常方便，也算是基本的入门了图的计算。
