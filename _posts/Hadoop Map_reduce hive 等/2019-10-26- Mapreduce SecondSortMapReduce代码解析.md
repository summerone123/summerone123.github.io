---
layout:     post
title:     Hadoop之 SecondSortMapReduce 源码详细解析
subtitle:   大数据处理
date:       2019-10-18
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - 大数据处理
---

# Hadoop之 SecondSortMapReduce 源码详细解析

## 数据
### 输入数据
```java

sort1	1
sort2	3
sort2	88
sort2	54
sort1	2
sort6	22
sort6	888
sort6	58
```
### 目标输出

## 具体解决思路
### Map端处理
- 目标就是要对第一列相同的记录，并且对合并后的数字进行排序。我们都知道MapReduce框架不管是默认排序或者是自定义排序都只是对key值进行排序，现在的情况是这些数据不是key值，怎么办？其实我们可以将原始数据的key值和其对应的数据组合成一个新的key值，然后新的key值对应的value还是原始数据中的value。那么我们就可以将原始数据的map输出变成类似下面的数据结构：

```java
{[sort1,1],1}
{[sort2,3],3}
{[sort2,88],88}
{[sort2,54],54}
{[sort1,2],2}
{[sort6,22],22}
{[sort6,888],888}
{[sort6,58],58}
```
- 那么我们只需要对[]里面的心key值进行排序就OK了，然后我们需要自定义一个分区处理器，因为我的目标不是想将新key相同的记录传到一个reduce中，而是想将新key中第一个字段相同的记录放到同一个reduce中进行分组合并，所以我们需要根据新key值的第一个字段来自定义一个分区处理器。通过分区操作后，得到的数据流如下：

```java
Partition1：{[sort1,1],1}、{[sort1,2],2}
 
Partition2：{[sort2,3],3}、{[sort2,88],88}、{[sort2,54],54}
 
Partition3：{[sort6,22],22}、{[sort6,888],888}、{[sort6,58],58}
```
- 分区操作完成之后，我调用自己的自定义排序器对新的key值进行排序。

```java
{[sort1,1],1}
{[sort1,2],2}
{[sort2,3],3}
{[sort2,54],54}
{[sort2,88],88}
{[sort6,22],22}
{[sort6,58],58}
{[sort6,888],888}
```

### Reduce端处理
- 经过Shuffle处理之后，数据传输到Reducer端了。在Reducer端按照组合键的第一个字段进行分组，并且每处理完一次分组之后就会调用一次reduce函数来对这个分组进行处理和输出。最终各个分组的数据结果变成类似下面的数据结构：

```java
sort1	1,2
sort2	3,54,88
sort6	22,58,888
```
## 源码及解析
```java
package bigdata.hadoop.mr.sortsecondary;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;
import java.net.URI;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.WritableComparable;
import org.apache.hadoop.io.WritableComparator;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Partitioner;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.KeyValueLineRecordReader;
import org.apache.hadoop.mapreduce.lib.input.KeyValueTextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;
import org.apache.hadoop.mapreduce.lib.partition.HashPartitioner;

//二次排序，结果写入一个文件。
public class SecondSortMapReduce {
	
	public static class CombinationKey implements WritableComparable<CombinationKey>{

		private Text firstKey;
		private IntWritable secondKey;
		
		//无参构造函数
		public CombinationKey() {
			this.firstKey = new Text();
			this.secondKey = new IntWritable();
		}
		
		//有参构造函数
		public CombinationKey(Text firstKey, IntWritable secondKey) {
			this.firstKey = firstKey;
			this.secondKey = secondKey;
		}

		public Text getFirstKey() {
			return firstKey;
		}

		public void setFirstKey(Text firstKey) {
			this.firstKey = firstKey;
		}

		public IntWritable getSecondKey() {
			return secondKey;
		}

		public void setSecondKey(IntWritable secondKey) {
			this.secondKey = secondKey;
		}

		public void write(DataOutput out) throws IOException {
			this.firstKey.write(out);
			this.secondKey.write(out);
		}

		public void readFields(DataInput in) throws IOException {
			this.firstKey.readFields(in);
			this.secondKey.readFields(in);
		}

		/**
		 * 自定义比较策略
		 * 注意：该比较策略用于MapReduce的第一次默认排序
		 * 也就是发生在Map端的sort阶段
		 * 发生地点为环形缓冲区(可以通过io.sort.mb进行大小调整)
		 */
		//经过测试，上述说法错误。setSortComparatorClass优先级总是比compareTo高。
		public int compareTo(CombinationKey combinationKey) {
			System.out.println("------------------------CombineKey flag-------------------");
			return this.firstKey.compareTo(combinationKey.getFirstKey());
		}

		@Override
		public int hashCode() {
			final int prime = 31;
			int result = 1;
			result = prime * result + ((firstKey == null) ? 0 : firstKey.hashCode());
			return result;
		}

		@Override
		public boolean equals(Object obj) {
			if (this == obj)
				return true;
			if (obj == null)
				return false;
			if (getClass() != obj.getClass())
				return false;
			CombinationKey other = (CombinationKey) obj;
			if (firstKey == null) {
				if (other.firstKey != null)
					return false;
			} else if (!firstKey.equals(other.firstKey))
				return false;
			return true;
		}

		
	}
	
	
	public static class DefinedPartition extends Partitioner<CombinationKey, IntWritable>{

		/**
		 * 数据输入来源：map输出 我们这里根据组合键的第一个值作为分区依据
		 * 如果不自定义分区的话，MapReduce会根据默认的Hash分区方法
		 * 将整个组合键相等的分到一个分区中，这样的话显然不是我们要的效果
		 * @param key map输出键值
		 * @param value map输出value值
		 * @param numPartitions 分区总数，即reduce task个数
		 */
		public int getPartition(CombinationKey key, IntWritable value, int numPartitions) {
			System.out.println("---------------------进入自定义分区---------------------");
			System.out.println("---------------------结束自定义分区---------------------");
			int hash = key.getFirstKey().hashCode();
			
			//key为Text，hash后可能为负数，所以此处按位与一下。
			int v = hash & Integer.MAX_VALUE;
			int result = v % numPartitions;
			
			System.out.println("分区"+key.getFirstKey()+"   "+result);
			return result;
		}

	}
	
	public static class DefinedComparator extends WritableComparator{

		protected DefinedComparator() {
			super(CombinationKey.class,true);
		}

		/**
		 * 第一列按升序排列，第二列也按升序排列
		 */
		public int compare(WritableComparable a, WritableComparable b) {
			System.out.println("------------------进入二次排序-------------------");
			CombinationKey c1 = (CombinationKey) a;
			CombinationKey c2 = (CombinationKey) b;
			int minus = c1.getFirstKey().compareTo(c2.getFirstKey());
			
			if (minus != 0){
				System.out.println("------------------结束二次排序-------------------");
				return minus;
			} else {
				System.out.println("------------------结束二次排序-------------------");
				return c1.getSecondKey().get() -c2.getSecondKey().get();
			}
		}
	}
	
	public static class DefinedGroupSort extends WritableComparator{


		protected DefinedGroupSort() {
			super(CombinationKey.class,true);
		}
		public int compare(WritableComparable a, WritableComparable b) {
			System.out.println("---------------------进入自定义GroupSort---------------------");
			CombinationKey combinationKey1 = (CombinationKey) a;
			CombinationKey combinationKey2 = (CombinationKey) b;
			System.out.println("---------------------GroupSort结果：" + combinationKey1.getFirstKey().compareTo(combinationKey2.getFirstKey()));
			System.out.println("---------------------结束自定义分组---------------------");
			//自定义按原始数据中第一个key分组
			return combinationKey1.getFirstKey().compareTo(combinationKey2.getFirstKey());
		}


	}

	public static void main(String[] args) {
		
		String INPUT_PATH = args[0];
		String OUT_PATH = args[1];
		
		try {
			// 创建配置信息
			Configuration conf = new Configuration();
			conf.set(KeyValueLineRecordReader.KEY_VALUE_SEPERATOR, "\t");

			// 创建文件系统
			FileSystem fileSystem = FileSystem.get(new URI(OUT_PATH), conf);
			// 如果输出目录存在，我们就删除
			if (fileSystem.exists(new Path(OUT_PATH))) {
				fileSystem.delete(new Path(OUT_PATH), true);
			}

			// 创建任务
			Job job = Job.getInstance(conf, "secondary sort");

			//1.1	设置输入目录和设置输入数据格式化的类
			FileInputFormat.setInputPaths(job, INPUT_PATH);
			job.setInputFormatClass(KeyValueTextInputFormat.class);

			//1.2	设置自定义Mapper类和设置map函数输出数据的key和value的类型
			job.setMapperClass(SecondSortMapper.class);
			job.setMapOutputKeyClass(CombinationKey.class);
			job.setMapOutputValueClass(IntWritable.class);

			//1.3	设置分区和reduce数量(reduce的数量，和分区的数量对应，因为分区为一个，所以reduce的数量也是一个)
			job.setPartitionerClass(DefinedPartition.class);
			//设置reducer为1个，则上面的自定义分区运行时不会被调用。
			job.setNumReduceTasks(3);
			
			//设置自定义分组策略
			job.setGroupingComparatorClass(DefinedGroupSort.class);
			//设置自定义比较策略(因为我的CombineKey重写了compareTo方法，所以这个可以省略)
			job.setSortComparatorClass(DefinedComparator.class);
			
			//1.4	排序
			//1.5	归约
			//2.1	Shuffle把数据从Map端拷贝到Reduce端。
			//2.2	指定Reducer类和输出key和value的类型
			job.setReducerClass(SecondSortReducer.class);
			job.setOutputKeyClass(Text.class);
			job.setOutputValueClass(Text.class);

			//2.3	指定输出的路径和设置输出的格式化类
			FileOutputFormat.setOutputPath(job, new Path(OUT_PATH));
			job.setOutputFormatClass(TextOutputFormat.class);


			// 提交作业 退出
			System.exit(job.waitForCompletion(true) ? 0 : 1);
		
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
	
public static class SecondSortMapper extends Mapper<Text, Text, CombinationKey, IntWritable>{
	/**
	 * 这里要特殊说明一下，为什么要将这些变量写在map函数外边
	 * 对于分布式的程序，我们一定要注意到内存的使用情况，对于MapReduce框架
	 * 每一行的原始记录的处理都要调用一次map()函数，假设，这个map()函数要处理1一亿
	 * 条输入记录，如果将这些变量都定义在map函数里面则会导致这4个变量的对象句柄
	 * 非常的多(极端情况下将产生4*1亿个句柄，当然java也是有自动的GC机制的，一定不会达到这么多)
	 * 导致栈内存被浪费掉，我们将其写在map函数外面，顶多就只有4个对象句柄
	 */
	private CombinationKey combinationKey = new CombinationKey();
	Text sortName = new Text();
	IntWritable score = new IntWritable();
	String[] splits = null;
	protected void map(Text key, Text value, Mapper<Text, Text, CombinationKey, IntWritable>.Context context) throws IOException, InterruptedException {
		System.out.println("---------------------进入map()函数---------------------");
		//过滤非法记录(这里用计数器比较好)
		if (key == null || value == null || key.toString().equals("")){
			return;
		}
		//构造相关属性
		sortName.set(key.toString());
		score.set(Integer.parseInt(value.toString()));
		//设置联合key
		combinationKey.setFirstKey(sortName);
		combinationKey.setSecondKey(score);
		
		//通过context把map处理后的结果输出
		context.write(combinationKey, score);
		System.out.println("---------------------结束map()函数---------------------");
	}
	
}


public static class SecondSortReducer extends Reducer<CombinationKey, IntWritable, Text, Text>{
	
	StringBuffer sb = new StringBuffer();
	Text score = new Text();
	/**
	 * 这里要注意一下reduce的调用时机和次数：
	 * reduce每次处理一个分组的时候会调用一次reduce函数。
	 * 所谓的分组就是将相同的key对应的value放在一个集合中
	 * 例如：<sort1,1> <sort1,2>
	 * 分组后的结果就是
	 * <sort1,{1,2}>这个分组会调用一次reduce函数
	 */
	protected void reduce(CombinationKey key, Iterable<IntWritable> values, Reducer<CombinationKey, IntWritable, Text, Text>.Context context)
			throws IOException, InterruptedException {
		
		
		//先清除上一个组的数据
		sb.delete(0, sb.length());
		
		for (IntWritable val : values){
			sb.append(val.get() + ",");
		}
		
		//取出最后一个逗号
		if (sb.length() > 0){
			sb.deleteCharAt(sb.length() - 1);
		}
		
		//设置写出去的value
		score.set(sb.toString());
		
		//将联合Key的第一个元素作为新的key，将score作为value写出去
		context.write(key.getFirstKey(), score);
		
		System.out.println("---------------------进入reduce()函数---------------------");
		System.out.println("---------------------{[" + key.getFirstKey()+"," + key.getSecondKey() + "],[" +score +"]}");
		System.out.println("---------------------结束reduce()函数---------------------");
	}
}
}
```

## 运行结果
![-w1440](/img/blog_img/15722397376145.jpg)
![-w940](/img/blog_img/15722397495171.jpg)
![-w939](/img/blog_img/15722397629108.jpg)

## 总结
看了Reduce端的日志，第一个信息我应该很容易能够很容易看出来，就是分组和reduce()函数处理都是在Shuffle完成之后才进行的。另外一点我们也非常容易看出，就是每次处理完一个分组数据就会去调用一次的reduce()函数对这个分组进行处理和输出。此外，说明一些分组函数的返回值问题，当返回0时才会被分到同一个组中。另外一点我们也可以看出来，一个分组中每合并n个值就会有n-1分组函数返回0值，也就是说进行了n-1次比较。