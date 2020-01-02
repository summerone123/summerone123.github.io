---
layout:     post
title:     Spark Mllib
subtitle:   大数据处理
date:       2019-12-23
author:     Yi Xia
header-img: img/post-bg-map.jpg
catalog: true
tags:
    - 大数据处理
---

# spark Mllib 实践
> 基于分布式架构的开源机器学习

## spark Mllib 简介
MLlib（machine learning library）是Spark提供的可扩展的机器学习库。MLlib中已经包含了一些通用的学习算法和工具，如：分类、回归、聚类、协同过滤、降维以及底层的优化原语等算法和工具。

![-w511](/img/blog_img/15772576153324.jpg)
从架构图可以看出MLlib主要包含三个部分：

* 底层基础：包括Spark的运行库、矩阵库和向量库；

* 算法库：包含广义线性模型、推荐系统、聚类、决策树和评估的算法；

* 实用程序：包括测试数据的生成、外部数据的读入等功能。

MLlib提供的API主要分为以下两类：

* spark.mllib包中提供的主要API。

* spark.ml包中提供的构建机器学习工作流的高层次的API。

*<div align=center>![-w294](/img/blog_img/15772341906992.jpg)
</div>




## spark Mllib 实践
### 实践 1 ：实现 Mllib 的聚类算法（Kmeans 算法）
```scala
import org.apache.log4j.{Level, Logger}
import org.apache.spark.mllib.clustering.KMeans
import org.apache.spark.mllib.linalg.Vectors
import org.apache.spark.{SparkConf, SparkContext}

object test_Mllib {
  def main(args: Array[String]): Unit = {



      //处理数据，转换为向量
      val conf = new SparkConf().setAppName("kmeans").setMaster("local")
      val sc = new SparkContext(conf)
      val data = sc.textFile("file:///Users/summerone/Desktop/spark/Mllib/kmeans_data.txt")
      println(data)
      val parsedData = data.map(line => {
        Vectors.dense(line.split(" ").map(_.toDouble))
      })

      //k=3：设置分为3个簇；maxIterations=10：设置最大迭代次数；然后传入train方法训练得到模型
      val k = 3
      val maxIterations = 10
      val model = KMeans.train(parsedData, k, maxIterations)

      //输出最终3个类簇的质心
      println("Cluster centers:")
      for (c <- model.clusterCenters) {
        println(c.toString)
      }

      //使用模型测试单点数据
      println(" ")
      val v1 = Vectors.dense("0.222 0.444".split(" ").map(_.toDouble))
      val v2 = Vectors.dense("0.333 0.222".split(" ").map(_.toDouble))
      val v3 = Vectors.dense("0.666 0.111".split(" ").map(_.toDouble))
      val v4 = Vectors.dense("0.111 0.666".split(" ").map(_.toDouble))
      println(s"v1=(${v1.apply(0)},${v1.apply(1)}) is belong to cluster:" + model.predict(v1))
      println(s"v2=(${v2.apply(0)},${v2.apply(1)}) is belong to cluster:" + model.predict(v2))
      println(s"v3=(${v3.apply(0)},${v3.apply(1)}) is belong to cluster:" + model.predict(v3))
      println(s"v4=(${v4.apply(0)},${v4.apply(1)}) is belong to cluster:" + model.predict(v4))
    }

}

```
结果：
![-w1440](/img/blog_img/15772455731633.jpg)


### 实现 MlLib 的分类算法（SVM 支持向量机，随机森林分类算法）

```scala
package Mllib

import org.apache.spark.SparkContext
import org.apache.spark.SparkConf
import org.apache.spark.mllib.util._
import org.apache.spark.mllib.linalg.Vector
import org.apache.spark.mllib.stat.{ MultivariateStatisticalSummary, Statistics }
import org.apache.spark.mllib.regression._
import org.apache.spark.mllib.classification.{ SVMModel, SVMWithSGD }
import org.apache.spark.mllib.evaluation.BinaryClassificationMetrics
import org.apache.spark.mllib.feature.ChiSqSelector
import org.apache.spark.mllib.tree.RandomForest
import org.apache.spark.mllib.tree.model.RandomForestModel
import scala.collection.mutable.ListBuffer

object test_Mllib_svm {
  def main(args: Array[String]): Unit = {


    val conf = new SparkConf().setAppName("mllib_example").setMaster("local")
    val sc = new SparkContext(conf)

    //Read the dataset
    val rawdata = MLUtils.loadLibSVMFile(sc, "/Users/summerone/IdeaProjects/test/src/main/scala/Mllib/epsilon_normalized_train")
    //    val rawdata = MLUtils.loadLibSVMFile(sc, "hdfs://hadoop-master:8020/user/spark/datasets/epsilon/epsilon_normalized")

    val data = rawdata.map { lp =>
      val newclass = if (lp.label == 1.0) 0 else 1
      new LabeledPoint(newclass, lp.features)
    }

    //Count the number of sample for each class
    val classInfo = data.map(lp => (lp.label, 1L)).reduceByKey(_ + _).collectAsMap()

    val observations = data.map(_.features)
    // Compute column summary statistics. Param -> number of the column
    val summary: MultivariateStatisticalSummary = Statistics.colStats(observations)
    var outputString = new ListBuffer[String]
    outputString += "\n\n@Mean (0) --> " + summary.mean(0) + "\n" // a dense vector containing the mean value for each column

    outputString += "@NumNonZeros (0) --> " + summary.numNonzeros(0) + "\n" // number of nonzeros in each column

    // Load the test data
    val rawtest = MLUtils.loadLibSVMFile(sc, "/Users/summerone/IdeaProjects/test/src/main/scala/Mllib/epsilon_normalized_test")
    //val rawtest = MLUtils.loadLibSVMFile(sc, "hdfs://hadoop-master:8020/user/spark/datasets/epsilon/epsilon_normalized.t")
    val test = rawtest.map { lp =>
      val newclass = if (lp.label == 1.0) 0 else 1
      new LabeledPoint(newclass, lp.features)
    }

    // Cache data (only training)
    val training = data.cache()

    /** SUPPORT VECTOR MACHINE - SVM **/

    // Run training algorithm to build the model
    val numIterations = 10
    var modelSVM = SVMWithSGD.train(training, numIterations)
    // Clear the default threshold.
    modelSVM.clearThreshold()
    // Compute raw scores on the test set.
    val trainScores = training.map { point =>
      val score = modelSVM.predict(point.features)
      (score, point.label)
    }
    // Get evaluation metrics (do separately).
    val metrics = new BinaryClassificationMetrics(trainScores)
    val measuresByThreshold = metrics.fMeasureByThreshold.collect()
    val maxThreshold = measuresByThreshold.maxBy { _._2 }
    println("Max (Threshold, Precision):" + maxThreshold)
    modelSVM.setThreshold(maxThreshold._1)
    // Compute raw scores on the test set.
    var testScores = test.map(p => (modelSVM.predict(p.features), p.label))
    outputString += "@Acc SVM --> " + 1.0 * testScores.filter(x => x._1 == x._2).count() / test.count() + "\n" // number of nonzeros in each column

    /** RANDOM FOREST - RF **/

    // Train a RandomForest model.
    // Empty categoricalFeaturesInfo indicates all features are continuous.
    val numClasses = 2
    val categoricalFeaturesInfo = Map[Int, Int]()
    val numTrees = 50 // Use more in practice.
    val featureSubsetStrategy = "auto" // Let the algorithm choose.
    val impurity = "gini"
    val maxDepth = 4
    val maxBins = 32
    var modelRF = RandomForest.trainClassifier(training, numClasses, categoricalFeaturesInfo, numTrees, featureSubsetStrategy, impurity, maxDepth, maxBins)
    modelRF.numTrees
    modelRF.totalNumNodes
    modelRF.trees
    // Evaluate model on test instances and compute test error
    testScores = test.map(p => (modelRF.predict(p.features), p.label))
    outputString += "@Acc RF --> " + 1.0 * testScores.filter(x => x._1 == x._2).count() / test.count() + "\n"

    println(outputString)

    val predictionsTxt = sc.parallelize(outputString, 1)
    predictionsTxt.saveAsTextFile("/Users/summerone/IdeaProjects/test/src/main/scala/Mllib/output.txt")

  }
}

```

![-w1440](/img/blog_img/15779521396964.jpg)

分别用训练集进行训练模型，并在测试集上进行预测，比较在此测试集上其准确率的比较。
如图为分类的准确率结果，在这个测试集上，SVM 分类的准确率为 0.61，而 RF 随机森林的准确率是 0.695。

##总结
数据挖掘的主要任务也就是分类，聚类以及回归算法，我在阅读 spark Mllib 的官方教程的基础上，并进行实践。认识到 spark 的 Mllib 对于这些机器学习算法的实现方式还是比较完善的，并且调用方式较为简单。并且 spark Mllib 最大的优势是可以基于分布式架构的开源机器学习，在对大规模数据进行训练或测试时，一定会得到更加好的效果。