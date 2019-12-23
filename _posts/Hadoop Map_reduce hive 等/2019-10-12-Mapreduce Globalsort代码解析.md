---
layout:     post
title:     Mapreduce Globalsort代码解析
subtitle:   大数据处理
date:       2018-01-04
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - 大数据处理
---

# Mapreduce Globalsort代码解析
排序是MapReduce的核心技术，有三种方法实现Hadoop(MapReduce)全局排序，下面分别介绍
## 1、使用一个Reduce进行排序

* MapReduce默认只是保证同一个分区内的Key是有序的，但是不保证全局有序。
    如果我们将所有的数据全部发送到一个Reduce，就可以实现结果全局有序。实现如下：
    
```java
package bigdata.hadoop.mr.sortglobal;

import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;

import java.io.IOException;

//使用一个reducer实现全排序

public class GlobalSortV1 extends Configured implements Tool {
	static class SimpleMapper extends
			Mapper<LongWritable, Text, IntWritable, IntWritable> {
		@Override
		protected void map(LongWritable key, Text value, Context context)
				throws IOException, InterruptedException {
			IntWritable intWritable = new IntWritable(Integer.parseInt(value
					.toString()));
			context.write(intWritable, intWritable);
		}
	}

	static class SimpleReducer extends
			Reducer<IntWritable, IntWritable, IntWritable, NullWritable> {
		@Override
		protected void reduce(IntWritable key, Iterable<IntWritable> values,
				Context context) throws IOException, InterruptedException {
			for (IntWritable value : values)
				context.write(value, NullWritable.get());
		}
	}

	@Override
	public int run(String[] args) throws Exception {
		if (args.length != 2) {
			System.err.println("<input> <output>");
			System.exit(127);
		}

		Job job = Job.getInstance(getConf());
		job.setJobName("GlobalSort");
		job.setJarByClass(GlobalSortV1.class);
		
		FileInputFormat.addInputPath(job, new Path(args[0]));
		
		job.setMapperClass(SimpleMapper.class);
		
		
		//当mapper端的key与最终输出的key类型一致时，此方法可省略
		//job.setMapOutputKeyClass(IntWritable.class);
		
		//map端的输出value与最终输出value类型不一样，因此本方法不能省略。
		job.setMapOutputValueClass(IntWritable.class);
		
		job.setReducerClass(SimpleReducer.class);		
		job.setNumReduceTasks(1);
		
		job.setOutputKeyClass(IntWritable.class);
		job.setOutputValueClass(NullWritable.class);
		

		
		Path outputPath = new Path(args[args.length - 1]);
		FileSystem fileSystem = outputPath.getFileSystem(getConf());
		if (fileSystem.exists(outputPath))
			fileSystem.delete(outputPath, true);
		FileOutputFormat.setOutputPath(job, outputPath);
		
		FileOutputFormat.setOutputPath(job, new Path(args[1]));
		
		
		return job.waitForCompletion(true) ? 0 : 1;
	}

	public static void main(String[] args) throws Exception {
		int exitCode = ToolRunner.run(new GlobalSortV1(), args);
		System.exit(exitCode);
	}
}
```

- 实现结果及说明：
    - **map阶段输入key和value** </br>
 ![map阶段输入key和value](/img/blog_img/map%E9%98%B6%E6%AE%B5%E8%BE%93%E5%85%A5key%E5%92%8Cvalue.png)
    - **map阶段输入key和value** </br>
![reduce阶段的key和value](/img/blog_img/reduce%E9%98%B6%E6%AE%B5%E7%9A%84key%E5%92%8Cvalue.png)
    - **Globalsort_v1排序结果** </br>
![v1排序结果](/img/blog_img/v1%E6%8E%92%E5%BA%8F%E7%BB%93%E6%9E%9C.png)


    
- 实现说明：
    * 上面程序的实现很简单，我们直接使用 TextInputFormat 类来读取上面生成的随机数文件。因为文件里面的数据是正整数，所以我们在 SimpleMapper 类里面直接将value转换成int类型，然后赋值给IntWritable。等数据到 SimpleReducer 的时候，同一个Reduce里面的Key已经全部有序；因为我们设置了一个Reduce作业，这样的话，我们就实现了数据全局有序。

- 缺点：
所有的数据都发送到一个Reduce进行排序，这样不能充分利用集群的计算资源，而且在数据量很大的情况下，很有可能会出现OOM问题

# 2、自定义分区函数实现全局有序

上面实现数据全局有序有个很大的局限性：
所有的数据都发送到一个Reduce进行排序，这样不能充分利用集群的计算资源，而且在数据量很大的情况下，很有可能会出现OOM问题。我们分析一下，MapReduce默认的分区函数是HashPartitioner，其实现的原理是计算map输出key的 hashCode ，然后对Reduce个数求模，这样只要求模结果一样的Key都会发送到同一个Reduce。如果我们能够实现一个分区函数，使得

    - 所有 Key < 10000 的数据都发送到Reduce 0；
    - 所有 10000 < Key < 20000 的数据都发送到Reduce 1；
    - 其余的Key都发送到Reduce 2；
    
这就实现了Reduce 0的数据一定全部小于Reduce 1，且Reduce 1的数据全部小于Reduce 2，再加上同一个Reduce里面的数据局部有序，这样就实现了数据的全局有序。实现如下：
```java
package bigdata.hadoop.mr.sortglobal;

import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Partitioner;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;

import java.io.IOException;

//手工定义分界点使用分区实现全排序

public class GlobalSortV2 extends Configured implements Tool {
	static class SimpleMapper extends
			Mapper<LongWritable, Text, IntWritable, IntWritable> {
		@Override
		protected void map(LongWritable key, Text value, Context context)
				throws IOException, InterruptedException {

			// System.out.println(key.get());
			// Thread.sleep(5000);
			IntWritable intWritable = new IntWritable(Integer.parseInt(value
					.toString()));
			context.write(intWritable, intWritable);
		}
	}

	static class SimpleReducer extends
			Reducer<IntWritable, IntWritable, IntWritable, NullWritable> {
		@Override
		protected void reduce(IntWritable key, Iterable<IntWritable> values,
				Context context) throws IOException, InterruptedException {
			for (IntWritable value : values)
				context.write(value, NullWritable.get());
		}
	}

	public static class IteblogPartitioner extends
			Partitioner<IntWritable, IntWritable> {
		@Override
		public int getPartition(IntWritable key, IntWritable value,
				int numPartitions) {
			int keyInt = Integer.parseInt(key.toString());
			if (keyInt < 30) {
				return 0;
			} else if (keyInt < 60) {
				return 1;
			} else {
				return 2;
			}
		}
	}

	@Override
	public int run(String[] args) throws Exception {
		if (args.length != 2) {
			System.err.println("<input> <output>");
			System.exit(127);
		}

		Job job = Job.getInstance(getConf());
		job.setJobName("dw_subject");
		job.setJarByClass(GlobalSortV2.class);

		FileInputFormat.addInputPath(job, new Path(args[0]));
		
		job.setMapperClass(SimpleMapper.class);

		// 可省略，因为map端输出key与最终输出key类型一致。
		job.setMapOutputKeyClass(IntWritable.class);
		// 不可省略
		job.setMapOutputValueClass(IntWritable.class);

		job.setPartitionerClass(IteblogPartitioner.class);

		job.setReducerClass(SimpleReducer.class);
		job.setNumReduceTasks(3);

		job.setOutputKeyClass(IntWritable.class);
		job.setOutputValueClass(NullWritable.class);

		Path outputPath = new Path(args[args.length - 1]);
		FileSystem fileSystem = outputPath.getFileSystem(getConf());
		if (fileSystem.exists(outputPath))
			fileSystem.delete(outputPath, true);
		FileOutputFormat.setOutputPath(job, outputPath);

		return job.waitForCompletion(true) ? 0 : 1;
	}

	public static void main(String[] args) throws Exception {
		int exitCode = ToolRunner.run(new GlobalSortV2(), args);
		System.exit(exitCode);
	}
}
```
- 实现结果及说明：
    - 输入文件</br>
![输入文件](/img/blog_img/%E8%BE%93%E5%85%A5%E6%96%87%E4%BB%B6.png)

   - 分区规则</br>
   ![-w832](/img/blog_img/15708662097298.jpg)

    - 第一个分区局部有序</br>![v2_reduce](/img/blog_img/v2_reducer1.png)</br>
![V2_result1](/img/blog_img/V2_result1.png)                                          

    - 第二个分区局部</br>
    ![V2_reduce](/img/blog_img/V2_reducer2.png)
![V2_result2](/img/blog_img/V2_result2.png)
    - 第三个区局部有</br>![V2_reduce](/img/blog_img/V2_reducer3.png)
![V2_result3](/img/blog_img/V2_result3.png)

- 实现说明：
    * 第二版的排序实现除了自定义的 IteblogPartitioner，其余的和第一种实现一样。
    * 我们已经看到了这个程序生成了三个文件（因为我们设置了Reduce个数为3），而且每个文件都是局部有序；所有小于30的数据都在part-r-00000里面，所有大于 30，小于60的数据都在part-r-00001里面，所有大于60的数据都在part-r-00002里面。part-r-00000、part-r-00001和part-r-00002三个文件实现了全局有序。

- 缺点：
我们必须手动地找到各个Reduce的分界点，尽量使得分散到每个Reduce的数据量均衡。而且每次修改Reduce的个数时，都得手动去找一次Key的分界点！非常不灵活。

# 3、使用TotalOrderPartitioner进行全排序

程序源码：
```java
package bigdata.hadoop.mr.sortglobal;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.*;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.KeyValueTextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.partition.InputSampler;
import org.apache.hadoop.mapreduce.lib.partition.TotalOrderPartitioner;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;
 

import java.io.IOException;
 
//自动定义分界点实现全排序
public class GlobalSortV3 extends Configured implements Tool {
	
    static class SimpleMapper extends Mapper<Text, Text, Text, IntWritable> {
        @Override
        protected void map(Text key, Text value,
                           Context context) throws IOException, InterruptedException {
            IntWritable intWritable = new IntWritable(Integer.parseInt(key.toString()));
           
            //System.out.println(key.toString()+"  "+intWritable.get());
            context.write(key, intWritable);
        }
    }
    
    //因为key是Text型，所以需要编写keyComparator。
    public static class KeyComparator extends WritableComparator {
        protected KeyComparator() {
            super(Text.class, true);
        }
 
        @Override
        public int compare(WritableComparable w1, WritableComparable w2) {
            int v1 = Integer.parseInt(w1.toString());
            int v2 = Integer.parseInt(w2.toString());
 
            return v1 - v2;
        }
    }
 
    static class SimpleReducer extends Reducer<Text, IntWritable, IntWritable, NullWritable> {
        @Override
        protected void reduce(Text key, Iterable<IntWritable> values,
                              Context context) throws IOException, InterruptedException {
            for (IntWritable value : values)
                context.write(value, NullWritable.get());
        }
    }
 
    

 
    @Override
    public int run(String[] args) throws Exception {
        Configuration conf = getConf();

        
        Job job = Job.getInstance(conf, "Total Order Sorting");
        job.setJarByClass(GlobalSortV3.class);
        job.setJobName("iteblog");
        
        
         FileInputFormat.addInputPath(job, new Path(args[0]));
        /*设置如何读取待处理文件，默认是TextInputFormat，即key为偏移量，value为一行值
                          此处使用KeyValueTextInputFormat，如果行中有分隔符，那么分隔符前面的作为key，后面的作为value
 		     如果行中没有分隔符，那么整行作为key，value为空。默认分隔符为 \t
         */
        job.setInputFormatClass(KeyValueTextInputFormat.class);
        
        job.setMapperClass(SimpleMapper.class);
        
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);
        
        job.setPartitionerClass(TotalOrderPartitioner.class);
        job.setSortComparatorClass(KeyComparator.class);
        
        job.setReducerClass(SimpleReducer.class);
        job.setNumReduceTasks(3);
       
      
        /*
         *  map 输出 Key 的类型是 Text ，这是没办法的，（参见下面注释的大段代码）
         *  因为 InputSampler.writePartitionFile 函数实现的原因，
         *  必须要求 map 输入和输出 Key 的类型一致，否则会出现异常，而对map而言，没有输入类型为IntWritable的inputformat
         */
        job.setOutputKeyClass(IntWritable.class);
        job.setOutputValueClass(NullWritable.class);
        
		Path outputPath = new Path(args[args.length - 1]);
		FileSystem fileSystem = outputPath.getFileSystem(getConf());
		if (fileSystem.exists(outputPath))
			fileSystem.delete(outputPath, true);
		FileOutputFormat.setOutputPath(job, outputPath);
 
		 //抽样并保存抽样文件
		
	      
        //如果 Key 的类型是 BinaryComparable （BytesWritable 和 Text ），
        //并且 mapreduce.totalorderpartitioner.naturalorder 属性的指是 true或null
        //则会构建trie 树,否则是按照setSortComparatorClass的设置进行key比较后分区。
        conf.set("mapreduce.totalorderpartitioner.naturalorder", "false");//折腾了半天就少这句话,否则按照字典序进行分区比较。
        TotalOrderPartitioner.setPartitionFile(job.getConfiguration(), new Path(args[1]));//获取设置的分区数目，以便确定分隔点数目
        //从100个文件分片（如果文件分片总数>=100个）中以1%的概率选择1000个样本，从而获取分隔点。
        InputSampler.Sampler<Text, Text> sampler = new InputSampler.RandomSampler<Text,Text>(0.01, 1000, 100);
        InputSampler.writePartitionFile(job, sampler);
 
        return job.waitForCompletion(true) ? 0 : 1;
    }
 
    public static void main(String[] args) throws Exception {
        int exitCode = ToolRunner.run(new GlobalSortV3(), args);
        System.exit(exitCode);
    }
    
```

运行截图：
选出了三个最佳的分割点，分别是 26，52，66.
自动分区的第一个分区，最终判定以 26 为分割点
![V3_1](/img/blog_img/V3_1.png)

自动分区的第一个分区，最终判定以 52 为分割点
![V3_2](/img/blog_img/V3_2.png)
自动分区的第一个分区，最终判定以 66 为分割点
![V3_3](/img/blog_img/V3_3.png)


-------

程序分析：
Hadoop 内置还有个名为 `TotalOrderPartitioner` 的分区实现类，它解决全排序的问题。其主要做的事 实际上和 上面介绍的第二种分区实现类很类似，也就是根据Key的分界点将不同的Key发送到相应的分区。
问题是，上文用到的分界点是我们人为计算的；而这里用到的分界点是由程序计算的。
- 数据抽样
寻找合适的Key分割点需要我们对数据的分布有个大概的了解；如果数据量很大的话，我们不可能对所有的数据进行分析然后选出 N-1 （N代表Reduce的个数）个分割点，最适合的方式是对数据进行抽样，然后对抽样的数据进行分析并选出合适的分割点。Hadoop提供了三种抽样的方法：
    * SplitSampler：从s个split中选取前n条记录取样
    * RandomSampler：随机取样
    * IntervalSampler：从s个split里面按照一定间隔取样，通常适用于有序数据

    这三个抽样都实现了

```java
K[] getSample(InputFormat inf, Job job) throws IOException, InterruptedException;
```
方法；通过调用这个方法我们可以返回抽样到的Key数组，除了 IntervalSampler 类返回的抽样Key是有序的，其他都无序。获取到采样好的Key数组之后，需要对其进行排序，然后选择好N-1 （N代表Reduce的个数）个分割点，最后将这些Key分割点存储到指定的HDFS文件中，存储的文件格式是SequenceFile，使用如下：

```java
TotalOrderPartitioner.setPartitionFile(job.getConfiguration(), new Path(args[2]));
InputSampler.Sampler<Text, Text> sampler = new InputSampler.RandomSampler<>(0.01, 1000, 100);
InputSampler.writePartitionFile(job, sampler);
```


上面通过 `InputSampler.writePartitionFile(job, sampler);` 存储好了分割点，然后 `TotalOrderPartitioner` 类会在 `setConf` 函数中读取这个文件，并根据`Key`的类型分别创建不同的数据结构：
    - 如果 `Key` 的类型是 `BinaryComparable （BytesWritable 和 Text ）`，并且 `mapreduce.totalorderpartitioner.naturalorder` 属性的指是 true ，则会构建trie 树，便于后面的查找；
    - 其他情况会构建一个 BinarySearchNode，用二分查找

最后程序通过调用 getPartition 函数决定当前Key应该发送到哪个Reduce中：

```java
public int getPartition(K key, V value, int numPartitions) {
    return partitions.findPartition(key);
}
```


-------


