---
layout:     post
title:      WordCountPartition,FlowcountPartition 源码解析
subtitle:   大数据处理
date:       2019-01-04
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - 大数据处理
---

# WordCountPartition,FlowcountPartition 源码解析

-------

## Partition 编程
1. Partitioner是partitioner的基类，如果需要定制partitioner也需要继承该类。

2. HashPartitioner是mapreduce的默认partitioner。计算方法是which reducer=(key.hashCode() &Integer.MAX_VALUE) % numReduceTasks，得到当前的目的reducer。

-------

## WordCountPartiton
**此次 WordCount 的代码相对于第一次基本 WordCount多了一个根据单词首字母对应ASCII码奇偶性分区，其他的大体思路类似**
### 源码

```java
/**
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package bigdata.hadoop.mr.partition;

import java.io.IOException;
import java.net.URI;
import java.util.StringTokenizer;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Partitioner;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.GenericOptionsParser;

//根据单词首字母对应ASCII码奇偶性分区。
public class WordCountPartition {

	public static class TokenizerMapper extends
			Mapper<Object, Text, Text, IntWritable> {

		private final static IntWritable one = new IntWritable(1);
		private Text word = new Text();

		public void map(Object key, Text value, Context context)
				throws IOException, InterruptedException {
			StringTokenizer itr = new StringTokenizer(value.toString());
			while (itr.hasMoreTokens()) {
				word.set(itr.nextToken());
				context.write(word, one);
			}
		}
	}

	public static class IntSumReducer extends
			Reducer<Text, IntWritable, Text, IntWritable> {
		private IntWritable result = new IntWritable();

		public void reduce(Text key, Iterable<IntWritable> values,
				Context context) throws IOException, InterruptedException {
			int sum = 0;
			for (IntWritable val : values) {
				int num = val.get();
				System.out.println("Reducer:" + "   " +"value:" + key.toString() + "  " + "value:"
						+ num);

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
	
	public static class ASCIIOddEven extends Partitioner<Text, IntWritable>{

	       @Override
	       public int getPartition(Text key, IntWritable value, int numPartitions) {
	       //注意 ，如果在定义的类中使用HashPartition进行分区，要重写hashcode方法
	              // 1 获取单词key
	              String firWord = key.toString().substring(0, 1);
	       //直接用下面这一句也可以，自动类型转换
	          //char  firWord = key.toString().charAt(0);

	        //转换成ASCII码
	              char[] charArray = firWord.toCharArray();
	              int result = charArray[0];

	              // 2 根据奇数偶数分区
	              if (result % 2 == 0) {
	                     return 0;
	              }else {
	                     return 1;
	              }
	       }
	}

	public static void main(String[] args) throws Exception {
		//System.setProperty("hadoop.home.dir", "d:\\hadoop-2.7.3");
		Configuration conf = new Configuration();
		String[] otherArgs = new GenericOptionsParser(conf, args)
				.getRemainingArgs();
		if (otherArgs.length < 2) {
			System.err.println("Usage: wordcount <in> [<in>...] <out>");
			System.exit(2);
		}
		Job job = Job.getInstance(conf, "word count");
		job.setJarByClass(WordCountPartition.class);
		
		job.setMapperClass(TokenizerMapper.class);
		
		//job.setCombinerClass(IntSumCombiner.class);
		
		job.setReducerClass(IntSumReducer.class);
		
		job.setPartitionerClass(ASCIIOddEven.class);
		
		job.setNumReduceTasks(2);//会生成2个结果文件。
		
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(IntWritable.class);
		
		for (int i = 0; i < otherArgs.length - 1; ++i) {
			FileInputFormat.addInputPath(job, new Path(otherArgs[i]));
		}
		
		
		Path outputPath= new Path(otherArgs[otherArgs.length - 1]);
		FileSystem fileSystem = outputPath.getFileSystem(conf);
		if(fileSystem.exists(outputPath))
			fileSystem.delete(outputPath, true);
		FileOutputFormat.setOutputPath(job, outputPath);
		
		
		System.exit(job.waitForCompletion(true) ? 0 : 1);
	}
}

```

### 源码解析
- **Map() 阶段：从38行 - 54行**

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

- **Reduce() 阶段：从56行 - 74行**


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

- **Conbiner() 阶段：从76行 - 94行**
```java
public static class IntSumCombiner
       extends Reducer<Text,IntWritable,Text,IntWritable> {}
```

此时Combiner的操作和 Reduce 的代码是相同的，对Map Task产生的结果在本地节点上进行合并、统计等，以减少后续整个集群间的Shuffle过程所需要传输的数据量。

-------

- **ASCIIOddEven() 阶段：从96行 - 117行**
<p style="color:red">**此处是与第一次的最根本的区别，即根据单词首字母对应ASCII码奇偶性分区。使用HashPartition进行分区**</p> 
  
```java
public static class ASCIIOddEven extends Partitioner<Text, IntWritable>{

	       @Override
	       public int getPartition(Text key, IntWritable value, int numPartitions) {
	       //注意 ，如果在定义的类中使用HashPartition进行分区，要重写hashcode方法
	              // 1 获取单词key
	              String firWord = key.toString().substring(0, 1);
	       //直接用下面这一句也可以，自动类型转换
	          //char  firWord = key.toString().charAt(0);

	        //转换成ASCII码
	              char[] charArray = firWord.toCharArray();
	              int result = charArray[0];

	              // 2 根据奇数偶数分区
	              if (result % 2 == 0) {
	                     return 0;
	              }else {
	                     return 1;
	              }
	       }
	}

```
- **main() 阶段：从119行到最后**

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

## FlowCountPartition

### Flowcount 问题介绍

给一个数据文件，文件包含手机用户的各种上网信息，求每个手机用户的总上行流量，总下行流量和总流量；并且结果按总流量倒序排序。

### 源码

```java
package bigdata.hadoop.mr.partition;

import java.io.IOException;

import org.apache.commons.lang.StringUtils;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Partitioner;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;
//club.drguo.mapreduce.flowcount.FlowCount


import java.io.DataInput;
import java.io.DataOutput;
import java.util.HashMap;

import org.apache.hadoop.io.WritableComparable;

//自定义分区。按手机归属地分别计算各手机上传、下载和总流量。
public class FlowCountPartition {

	public static class FlowBean implements WritableComparable<FlowBean> {
		// 手机号
		private String phoneNum;
		// 上传流量
		private long up_flow;
		// 下载流量
		private long down_flow;
		// 总流量
		private long sum_flow;

		public void set(String phoneNum, long up_flow, long down_flow) {
			this.phoneNum = phoneNum;
			this.up_flow = up_flow;
			this.down_flow = down_flow;
			this.sum_flow = up_flow + down_flow;
		}

		/**
		 * 序列化，将数据字段以字节流写出去
		 */
		@Override
		public void write(DataOutput out) throws IOException {
			out.writeUTF(phoneNum);
			out.writeLong(up_flow);
			out.writeLong(down_flow);
			out.writeLong(sum_flow);
		}

		/**
		 * 反序列化，从字节流中读出各个数据字段 读写顺序，数据类型应一致
		 */
		@Override
		public void readFields(DataInput in) throws IOException {
			phoneNum = in.readUTF();
			up_flow = in.readLong();
			down_flow = in.readLong();
			sum_flow = in.readLong();
		}

		public String getPhoneNum() {
			return phoneNum;
		}

		public void setPhoneNum(String phoneNum) {
			this.phoneNum = phoneNum;
		}

		public long getUp_flow() {
			return up_flow;
		}

		public void setUp_flow(long up_flow) {
			this.up_flow = up_flow;
		}

		public long getDown_flow() {
			return down_flow;
		}

		public void setDown_flow(long down_flow) {
			this.down_flow = down_flow;
		}

		public long getSum_flow() {
			return sum_flow;
		}

		public void setSum_flow(long sum_flow) {
			this.sum_flow = sum_flow;
		}

		// 不写结果会出问题
		@Override
		public String toString() {
			return up_flow + "\t" + down_flow + "\t" + sum_flow;
		}

		// 比较排序（总流量）
		@Override
		public int compareTo(FlowBean o) {
			return this.sum_flow > o.getSum_flow() ? -1 : 1;
		}

	}

	public static class AreaPartitioner extends
			Partitioner<Text,FlowBean > {

		// 手机号，地区代码
		private static HashMap<String, Integer> areaMap = new HashMap<>();
		// 静态代码块，将数据先加载到内存中
		static {
			areaMap.put("134", 0);
			areaMap.put("135", 1);
			areaMap.put("137", 2);
			areaMap.put("138", 3);
		}

		@Override
		public int getPartition(Text key, FlowBean value, int numPartitions) {
			Integer provinceCode = areaMap.get(key.toString().substring(0, 3));

			return provinceCode == null ? 4 : provinceCode;
		}

	}

	public static class FlowCountPartitionMapper extends
			Mapper<LongWritable, Text, Text, FlowBean> {
		// 减少内存占用（如果放下面，GC机制会使对象越积越多）
		private FlowBean flowBean = new FlowBean();

		@Override
		protected void map(LongWritable key, Text value,
				Mapper<LongWritable, Text, Text, FlowBean>.Context context)
				throws IOException, InterruptedException {
			try {
				// 拿到一行数据
				String line = value.toString();
				// 切分字段
				String[] strings = StringUtils.split(line, "\t");
				// 拿到我们需要的若干个字段
				String phoneNum = strings[0];
				long up_flow = Long.parseLong(strings[1]);
				long down_flow = Long.parseLong(strings[2]);
				// 将数据封装到一个flowbean中
				flowBean.set(phoneNum, up_flow, down_flow);
				// 以手机号为key，将流量数据输出去
				context.write(new Text(phoneNum), flowBean);
			} catch (Exception e) {
				System.out.println("-----------------mapper出现问题"+e);
			}
		}
	}

	public static class FlowCountPartitionReducer extends
			Reducer<Text, FlowBean, Text, FlowBean> {
		// 减少内存占用（如果放下面，GC机制会使对象越积越多）
		private FlowBean flowBean = new FlowBean();

		@Override
		protected void reduce(Text key, Iterable<FlowBean> values,
				Reducer<Text, FlowBean, Text, FlowBean>.Context context)
				throws IOException, InterruptedException {
			long up_flow_sum = 0;
			long down_flow_sum = 0;
			for (FlowBean bean : values) {
				up_flow_sum += bean.getUp_flow();
				down_flow_sum += bean.getDown_flow();
			}
			flowBean.set(key.toString(), up_flow_sum, down_flow_sum);
			context.write(key, flowBean);
		}
	}

	public static void main(String[] args) throws IOException,
			ClassNotFoundException, InterruptedException {
		Configuration configuration = new Configuration();

		Job job = Job.getInstance(configuration, "flowjob");
		job.setJarByClass(FlowCountPartition.class);

		FileInputFormat.setInputPaths(job, new Path(args[0]));
		job.setInputFormatClass(TextInputFormat.class);

		job.setMapperClass(FlowCountPartitionMapper.class);
		// 因为map的输出和最终输入一致，实际上下面两句可以不写。
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(FlowBean.class);

		job.setPartitionerClass(AreaPartitioner.class);
		/**
		 * 设置reduce task的数量，要跟AreaPartitioner返回的partitioner个数匹配 若reduce
		 * task多，会产生多余的几个空文件 若reduce task少，就会发生异常，因为有一些key没有对应reduce task接收
		 * 但reduce task数量为1时，不会产生异常，因为所有key都会给这一个reduce task reduce task和map
		 * task指的是reducer和mapper在集群中运行的实例
		 */
		job.setNumReduceTasks(5);

		job.setReducerClass(FlowCountPartitionReducer.class);
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(FlowBean.class);

		job.setOutputFormatClass(TextOutputFormat.class);
		
		
		Path outputPath= new Path(args[args.length - 1]);
		FileSystem fileSystem = outputPath.getFileSystem(configuration);
		if(fileSystem.exists(outputPath))
			fileSystem.delete(outputPath, true);
		FileOutputFormat.setOutputPath(job, new Path(args[1]));

		boolean b = job.waitForCompletion(true);
		System.out.println(b ? "完成" : "未完成");
	}
}

```
### 源码解析
- **FlowBean.java通用的阶段：从31行 - 114行**

```java
public static class FlowBean implements WritableComparable<FlowBean> {
		// 手机号
		private String phoneNum;
		// 上传流量
		private long up_flow;
		// 下载流量
		private long down_flow;
		// 总流量
		private long sum_flow;

		public void set(String phoneNum, long up_flow, long down_flow) {
			this.phoneNum = phoneNum;
			this.up_flow = up_flow;
			this.down_flow = down_flow;
			this.sum_flow = up_flow + down_flow;
		}

		/**
		 * 序列化，将数据字段以字节流写出去
		 */
		@Override
		public void write(DataOutput out) throws IOException {
			out.writeUTF(phoneNum);
			out.writeLong(up_flow);
			out.writeLong(down_flow);
			out.writeLong(sum_flow);
		}

		/**
		 * 反序列化，从字节流中读出各个数据字段 读写顺序，数据类型应一致
		 */
		@Override
		public void readFields(DataInput in) throws IOException {
			phoneNum = in.readUTF();
			up_flow = in.readLong();
			down_flow = in.readLong();
			sum_flow = in.readLong();
		}

		public String getPhoneNum() {
			return phoneNum;
		}

		public void setPhoneNum(String phoneNum) {
			this.phoneNum = phoneNum;
		}

		public long getUp_flow() {
			return up_flow;
		}

		public void setUp_flow(long up_flow) {
			this.up_flow = up_flow;
		}

		public long getDown_flow() {
			return down_flow;
		}

		public void setDown_flow(long down_flow) {
			this.down_flow = down_flow;
		}

		public long getSum_flow() {
			return sum_flow;
		}

		public void setSum_flow(long sum_flow) {
			this.sum_flow = sum_flow;
		}

		// 不写结果会出问题
		@Override
		public String toString() {
			return up_flow + "\t" + down_flow + "\t" + sum_flow;
		}

		// 比较排序（总流量）
		@Override
		public int compareTo(FlowBean o) {
			return this.sum_flow > o.getSum_flow() ? -1 : 1;
		}

	}
``` 
- **分区AreaPartitioner 阶段：从116行 - 136行**

 按手机归属地分别计算各手机上传、下载和总流量。
 根据手机号地区代码进行分区，分 4 个区

```java
public static class AreaPartitioner extends
			Partitioner<Text,FlowBean > {

		// 手机号，地区代码
		private static HashMap<String, Integer> areaMap = new HashMap<>();
		// 静态代码块，将数据先加载到内存中
		static {
			areaMap.put("134", 0);
			areaMap.put("135", 1);
			areaMap.put("137", 2);
			areaMap.put("138", 3);
		}

		@Override
		public int getPartition(Text key, FlowBean value, int numPartitions) {
			Integer provinceCode = areaMap.get(key.toString().substring(0, 3));

			return provinceCode == null ? 4 : provinceCode;
		}

	}
```

- **FlowCountPartitionMapper 阶段：从138行 - 164行**

```java
public static class FlowCountPartitionMapper extends
			Mapper<LongWritable, Text, Text, FlowBean> {
		// 减少内存占用（如果放下面，GC机制会使对象越积越多）
		private FlowBean flowBean = new FlowBean();

		@Override
		protected void map(LongWritable key, Text value,
				Mapper<LongWritable, Text, Text, FlowBean>.Context context)
				throws IOException, InterruptedException {
			try {
				// 拿到一行数据
				String line = value.toString();
				// 切分字段
				String[] strings = StringUtils.split(line, "\t");
				// 拿到我们需要的若干个字段
				String phoneNum = strings[0];
				long up_flow = Long.parseLong(strings[1]);
				long down_flow = Long.parseLong(strings[2]);
				// 将数据封装到一个flowbean中
				flowBean.set(phoneNum, up_flow, down_flow);
				// 以手机号为key，将流量数据输出去
				context.write(new Text(phoneNum), flowBean);
			} catch (Exception e) {
				System.out.println("-----------------mapper出现问题"+e);
			}
		}
	}
```
- **FlowCountPartitionReducer 阶段：从166行 - 185行**


```java
public static class FlowCountPartitionReducer extends
			Reducer<Text, FlowBean, Text, FlowBean> {
		// 减少内存占用（如果放下面，GC机制会使对象越积越多）
		private FlowBean flowBean = new FlowBean();

		@Override
		protected void reduce(Text key, Iterable<FlowBean> values,
				Reducer<Text, FlowBean, Text, FlowBean>.Context context)
				throws IOException, InterruptedException {
			long up_flow_sum = 0;
			long down_flow_sum = 0;
			for (FlowBean bean : values) {
				up_flow_sum += bean.getUp_flow();
				down_flow_sum += bean.getDown_flow();
			}
			flowBean.set(key.toString(), up_flow_sum, down_flow_sum);
			context.write(key, flowBean);
		}
	}
```

- **main 阶段：从186行 - 226行**
 * 设置reduce task的数量，要跟AreaPartitioner返回的partitioner个数匹配 若reduce
 * task多，会产生多余的几个空文件 若reduce task少，就会发生异常，因为有一些key没有对应reduce task接收
 * 但reduce task数量为1时，不会产生异常，因为所有key都会给这一个reduce task reduce task和map
 * task指的是reducer和mapper在集群中运行的实例
   
```java
public static void main(String[] args) throws IOException,
			ClassNotFoundException, InterruptedException {
		Configuration configuration = new Configuration();

		Job job = Job.getInstance(configuration, "flowjob");
		job.setJarByClass(FlowCountPartition.class);

		FileInputFormat.setInputPaths(job, new Path(args[0]));
		job.setInputFormatClass(TextInputFormat.class);

		job.setMapperClass(FlowCountPartitionMapper.class);
		// 因为map的输出和最终输入一致，实际上下面两句可以不写。
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(FlowBean.class);

		job.setPartitionerClass(AreaPartitioner.class);
		/**
		 * 设置reduce task的数量，要跟AreaPartitioner返回的partitioner个数匹配 若reduce
		 * task多，会产生多余的几个空文件 若reduce task少，就会发生异常，因为有一些key没有对应reduce task接收
		 * 但reduce task数量为1时，不会产生异常，因为所有key都会给这一个reduce task reduce task和map
		 * task指的是reducer和mapper在集群中运行的实例
		 */
		job.setNumReduceTasks(5);

		job.setReducerClass(FlowCountPartitionReducer.class);
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(FlowBean.class);

		job.setOutputFormatClass(TextOutputFormat.class);
		
		
		Path outputPath= new Path(args[args.length - 1]);
		FileSystem fileSystem = outputPath.getFileSystem(configuration);
		if(fileSystem.exists(outputPath))
			fileSystem.delete(outputPath, true);
		FileOutputFormat.setOutputPath(job, new Path(args[1]));

		boolean b = job.waitForCompletion(true);
		System.out.println(b ? "完成" : "未完成");
	}
}

```