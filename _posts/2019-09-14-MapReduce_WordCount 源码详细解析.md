
---
layout:     post
title:     Hadoop之MapReduce WordCount 源码详细解析
subtitle:   大数据处理
date:       2018-01-04
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - 大数据处理
---


# Hadoop之MapReduce WordCount 源码详细解析
## MapReduce 基本的执行流程
众所周知，与学习编程语言时采用“hello world”程序作为入门示例程序不同。
在大数据处理领域常常使用“wordcount”程序作为入门程序。WordCount 程序是用来统计一段输入的数据中相同单词出现的频率。
其基本的执行流程如下图所示：
![-w725](/img/blog_img/15684536613029.jpg)
一个基于MapReduce的WordCount程序主要由一下几个部分组成：
**1、Split**

将程序的输入数据进行切分，每一个 split 交给一个 Map Task 执行。split的数量可以自己定义。

**2、Map**

输入为一个split中的数据，对split中的数据进行拆分，并以 < key, value> 对的格式保存数据，其中 key 的值为一个单词，value的值固定为 1。如 < I , 1>、< wish, 1> …

**3、Shuffle/Combine/sort**

这几个过程在一些简单的MapReduce程序中并不需要我们关注，因为源代码中已经给出了一些默认的Shuffle/Combine/sort处理器，这几个过程的作用分别是：

Combine：对Map Task产生的结果在本地节点上进行合并、统计等，以减少后续整个集群间的Shuffle过程所需要传输的数据量。**(代码其实和 Reduce 的是一样的）**
Shuffle / Sort：将集群中各个Map Task的处理结果在集群间进行传输，排序，数据经过这个阶段之后就作为 Reduce 端的输入。

**4、Reduce**

Reduce Task的输入数据其实已经不仅仅是简单的< key, value>对，而是经过排序之后的一系列key值相同的< key, value>对。Reduce Task对其进行统计等处理，产生最终的输出。

-------

## WordCount 实现
代码实现：
![-w1440](/img/blog_img/15684545144542.jpg)
输入：
![-w1438](/img/blog_img/15684546040805.jpg)
输出：
![-w1439](/img/blog_img/15684546563996.jpg)

Hadoop 2.8.5版本的 WordCount 源码如下：

```java
package hadoop;

import java.io.IOException;
import java.util.StringTokenizer;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.GenericOptionsParser;

public class WordCount {

// 继承Mapper类,Mapper类的四个泛型参数分别为：输入key类型，输入value类型，输出key类型，输出value类型
	public static class TokenizerMapper extends
			Mapper<Object, Text, Text, IntWritable> {

		private final static IntWritable one = new IntWritable(1); //输出的value的类型，可以理解为int
		private Text word = new Text(); //输出的key的类型，可以理解为String

		public void map(Object key, Text value, Context context)
				throws IOException, InterruptedException {
			StringTokenizer itr = new StringTokenizer(value.toString());//每行句子
			while (itr.hasMoreTokens()) {
				word.set(itr.nextToken());
				context.write(word, one);
			}
		}
	}
// Reduce类，继承了Reducer类
	public static class IntSumReducer extends
			Reducer<Text, IntWritable, Text, IntWritable> {
		private IntWritable result = new IntWritable();

                                //在这里，reduce步的输入相当于<单词,valuelist>,如<Hello,<1,1>>
		public void reduce(Text key, Iterable<IntWritable> values,
				Context context) throws IOException, InterruptedException {
			int sum = 0;
			for (IntWritable val : values) {
				int num = val.get();
				System.out.println("Reducer:" + "   " + "value:"
						+ key.toString() + "  " + "value:" + num);

				sum += num;
			}

			result.set(sum);
			context.write(key, result);
		}
	}

	public static class IntSumCombiner extends
			Reducer<Text, IntWritable, Text, IntWritable> {
		private IntWritable result = new IntWritable();

		public void reduce(Text key, Iterable<IntWritable> values,
				Context context) throws IOException, InterruptedException {
			int sum = 0;
			for (IntWritable val : values) {
				int num = val.get();
				System.out.println("Combiner:" + "   " + "value:"
						+ key.toString() + "  " + "value:" + num);

				sum += num;
			}

			result.set(sum);
			context.write(key, result);
		}
	}

	public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();    
        // 获取我们在执行这个任务时传入的参数，如输入数据所在路径、输出文件的路径的等
    String[] otherArgs = new GenericOptionsParser(conf, args).getRemainingArgs();
        //因为此任务正常运行至少要给出输入和输出文件的路径，因此如果传入的参数少于两个，程序肯定无法运行。
    if (otherArgs.length < 2) {
      System.err.println("Usage: wordcount <in> [<in>...] <out>");
      System.exit(2);
    }
    Job job = Job.getInstance(conf, "word count");  // 实例化job，传入参数，job的名字叫 word count
    job.setJarByClass(WordCount.class);  //使用反射机制，加载程序
    job.setMapperClass(TokenizerMapper.class);  //设置job的map阶段的执行类
    job.setCombinerClass(IntSumReducer.class);  //设置job的combine阶段的执行类
    job.setReducerClass(IntSumReducer.class);  //设置job的reduce阶段的执行类
    job.setOutputKeyClass(Text.class);  //设置程序的输出的key值的类型
    job.setOutputValueClass(IntWritable.class);  //设置程序的输出的value值的类型
    for (int i = 0; i < otherArgs.length - 1; ++i) {
      FileInputFormat.addInputPath(job, new Path(otherArgs[i]));
    }  //获取我们给定的参数中，输入文件所在路径
    FileOutputFormat.setOutputPath(job,
      new Path(otherArgs[otherArgs.length - 1]));  //获取我们给定的参数中，输出文件所在路径
    System.exit(job.waitForCompletion(true) ? 0 : 1);  //等待任务完成，任务完成之后退出程序
  }
}
```

-------

## WordCount 源码解析
- **Map() 阶段：从36行 - 47行**

```java
public static class TokenizerMapper 
       extends Mapper<Object, Text, Text, IntWritable>{}
```

MapReduce程序需要继承 org.apache.hadoop.mapreduce.Mapper 这个类，并在这个类的继承类中至少自定义实现 Map() 方法，其中 org.apache.hadoop.mapreduce.Mapper 要求的参数有四个（keyIn、valueIn、keyOut、valueOut），即Map（）任务的输入和输出都是< key，value >对的形式。

源代码中此处各个参数意义是：

1、Object：输入< key, value >对的 key 值，此处为文本数据的起始位置的偏移量。在大部分程序下这个参数可以直接使用 Long 类型，源码此处使用Object做了泛化。
2、Text：输入< key, value >对的 value 值，此处为一段具体的文本数据。
3、Text：输出< key, value >对的 key 值，此处为一个单词。
4、IntWritable：输出< key, value >对的 value 值，此处固定为 1 。IntWritable 是 Hadoop 对 Integer 的进一步封装，使其可以进行序列化。

```java
private final static IntWritable one = new IntWritable(1);
private Text word = new Text();

```

此处定义了两个变量：

one：类型为Hadoop定义的 IntWritable 类型，其本质就是序列化的 Integer ，one 变量的值恒为 1 。
word：因为在WordCount程序中，
<p style="color:red">**Map 端的任务是对输入数据按照单词进行切分**</p> ，每个单词为 Text 类型。

```java
 public void map(Object key, Text value, Context context
                    ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
      while (itr.hasMoreTokens()) {
        word.set(itr.nextToken());
        context.write(word, one);
      }
    }

```
这段代码为Map端的核心，定义了Map Task 所需要执行的任务的具体逻辑实现。
map() 方法的参数为 Object key, Text value, Context context，其中：

key： 输入数据在原数据中的偏移量。
value：具体的数据数据，此处为一段字符串。
context：用于暂时存储 map() 处理后的结果。

方法内部首先把输入值转化为字符串类型，并且对
<p style="color:red">**Hadoop自带的分词器 StringTokenizer 进行实例化用于存储输入数据。**</p> 
之后对输入数据从头开始进行切分，把字符串中的每个单词切分成< key, value >对的形式，如：< hello , 1>、< world, 1> …

-------

- **Reduce() 阶段：从52行 - 64行**


```java
public static class IntSumReducer 
       extends Reducer<Text,IntWritable,Text,IntWritable> {}
```

import org.apache.hadoop.mapreduce.Reducer 类的参数也是四个（keyIn、valueIn、keyOut、valueOut），即Reduce（）任务的输入和输出都是< key，value >对的形式。

源代码中此处各个参数意义是：
1、Text：输入< key, value >对的key值，此处为一个单词
2、IntWritable：输入< key, value >对的value值。
3、Text：输出< key, value >对的key值，此处为一个单词
4、IntWritable：输出< key, value >对，此处为相同单词词频累加之后的值。实际上就是一个数字。


```java
public void reduce(Text key, Iterable<IntWritable> values, 
                       Context context
                       ) throws IOException, InterruptedException {
      int sum = 0;
      for (IntWritable val : values) {
        sum += val.get();
      }
      result.set(sum);
      context.write(key, result);
    }

```

Reduce() 的三个参数为：
1、Text：输入< key, value >对的key值，也就是一个单词
2、value：这个地方值得注意，在前面说到了，在MapReduce任务中，除了我们自定义的map()和reduce()之外，在从map 刀reduce 的过程中，系统会自动进行combine、shuffle、sort等过程对map task的输出进行处理，因此reduce端的输入数据已经不仅仅是简单的< key, value >对的形式，而是一个一系列key值相同的序列化结构，如：< hello，1，1，2，2，3…>。因此，此处value的值就是单词后面出现的序列化的结构：（1，1，1，2，2，3…….）
3、context：临时存储reduce端产生的结果

因此再<p style="color:red">**reduce端的代码中，对value中的值进行累加，所得到的结果就是对应key 值的单词在文本出所出现的词频**</p> 

```java
public static class IntSumCombiner
       extends Reducer<Text,IntWritable,Text,IntWritable> {}
```

此时Combiner的操作和 Reduce 的代码是相同的，对Map Task产生的结果在本地节点上进行合并、统计等，以减少后续整个集群间的Shuffle过程所需要传输的数据量。

-------

- **main() 阶段：从68行到最后**

```java
public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();    
        // 获取我们在执行这个任务时传入的参数，如输入数据所在路径、输出文件的路径的等
    String[] otherArgs = new GenericOptionsParser(conf, args).getRemainingArgs();
        //因为此任务正常运行至少要给出输入和输出文件的路径，因此如果传入的参数少于两个，程序肯定无法运行。
    if (otherArgs.length < 2) {
      System.err.println("Usage: wordcount <in> [<in>...] <out>");
      System.exit(2);
    }
    Job job = Job.getInstance(conf, "word count");  // 实例化job，传入参数，job的名字叫 word count
    job.setJarByClass(WordCount.class);  //使用反射机制，加载程序
    job.setMapperClass(TokenizerMapper.class);  //设置job的map阶段的执行类
    job.setCombinerClass(IntSumReducer.class);  //设置job的combine阶段的执行类
    job.setReducerClass(IntSumReducer.class);  //设置job的reduce阶段的执行类
    job.setOutputKeyClass(Text.class);  //设置程序的输出的key值的类型
    job.setOutputValueClass(IntWritable.class);  //设置程序的输出的value值的类型
    for (int i = 0; i < otherArgs.length - 1; ++i) {
      FileInputFormat.addInputPath(job, new Path(otherArgs[i]));
    }  //获取我们给定的参数中，输入文件所在路径
    FileOutputFormat.setOutputPath(job,
      new Path(otherArgs[otherArgs.length - 1]));  //获取我们给定的参数中，输出文件所在路径
    System.exit(job.waitForCompletion(true) ? 0 : 1);  //等待任务完成，任务完成之后退出程序
  }
}
 
```

## 实验总结
这个大数据处理领域**“hello word”**级别的代码终于是“学”会了一点 ，🐮🍺，虽然也只是读懂了并分析了这个代码，但是自己会继续努力的，成为大数据时代的弄潮儿🥳。