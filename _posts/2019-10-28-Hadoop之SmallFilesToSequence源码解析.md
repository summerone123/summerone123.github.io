---
layout:     post
title:  Hadoop之SmallFilesToSequenceFileConverter 源码详细解析
subtitle:   大数据处理
date:       2019-10-18
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - 大数据处理
---

# Hadoop之SmallFilesToSequenceFileConverter 源码详细解析
## Hadoop文件合并
众所周知，Hadoop对处理单个大文件比处理多个小文件更有效率，另外单个文件也非常占用HDFS的存储空间。所以往往要将其合并起来。

## 源码及解析

```java
package bigdata.hadoop.mr.inputformat;

import java.io.IOException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IOUtils;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.InputSplit;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.JobContext;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.RecordReader;
import org.apache.hadoop.mapreduce.TaskAttemptContext;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.FileSplit;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.output.SequenceFileOutputFormat;
import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;

/*
 * 要计算的目标文件夹中有大量的小文本文件，会造成分配任务和资源的开销比实际的计算开销还大，这就产生了效率损耗。
 * 需要把一堆小文件合并成一个大文件
 */
public class SmallFilesToSequenceFileConverter {

	// 将整个文件作为一条记录处理
	public static class WholeFileInputFormat extends
			FileInputFormat<NullWritable, Text> {

		// 表示文件不可分
		@Override
		protected boolean isSplitable(JobContext context, Path filename) {
			return false;
		}

		@Override
		public RecordReader<NullWritable, Text> createRecordReader(
				InputSplit split, TaskAttemptContext context)
				throws IOException, InterruptedException {
			WholeRecordReader reader = new WholeRecordReader();
			reader.initialize(split, context);
			return reader;
		}

	}

	// 实现RecordReader,为自定义的InputFormat服务
	/*
	 * nextKeyValue负责生成要传递给map方法的key和value。
	 * getCurrentKey、getCurrentValue是实际获取key和value的。
	 * 所以RecordReader的核心机制就是：通过nextKeyValue生成key value，
	 * 然后通过getCurrentKey和getCurrentValue来返回上面构造好的key value。
	 * 这里的nextKeyValue负责把整个文件内容作为value。
	 */
	public static class WholeRecordReader extends
			RecordReader<NullWritable, Text> {

		private FileSplit fileSplit;
		private Configuration conf;
		private Text value = new Text();
		private boolean processed = false;// 表示记录是否被处理过

		@Override
		public NullWritable getCurrentKey() throws IOException,
				InterruptedException {
			return NullWritable.get();
		}

		@Override
		public Text getCurrentValue() throws IOException, InterruptedException {
			return value;
		}

		@Override
		public float getProgress() throws IOException, InterruptedException {
			return processed ? 1.0f : 0.0f;
		}

		@Override
		public void initialize(InputSplit split, TaskAttemptContext context)
				throws IOException, InterruptedException {
			this.fileSplit = (FileSplit) split;
			this.conf = context.getConfiguration();
		}

		@Override
		public boolean nextKeyValue() throws IOException, InterruptedException {
			if (!processed) {
				byte[] contents = new byte[(int) fileSplit.getLength()];
				Path file = fileSplit.getPath();
				FileSystem fs = file.getFileSystem(conf);
				FSDataInputStream in = null;
				try {
					in = fs.open(file);
					IOUtils.readFully(in, contents, 0, contents.length);
					value.set(contents, 0, contents.length);
				} finally {
					IOUtils.closeStream(in);
				}
				processed = true;
				return true;
			}
			return false;
		}

		@Override
		public void close() throws IOException {
			// TODO Auto-generated method stub

		}

	}

	private static class SequenceFileMapper extends
			Mapper<NullWritable, Text, Text, Text> {
		private Text filenameKey;

		// setup在task之前调用，用来初始化filenamekey

		@Override
		protected void setup(Context context) throws IOException,
				InterruptedException {
			InputSplit split = context.getInputSplit();
			Path path = ((FileSplit) split).getPath();
			filenameKey = new Text(path.toString());
		}

		@Override
		protected void map(NullWritable key, Text value, Context context)
				throws IOException, InterruptedException {
			context.write(filenameKey, value);
		}
	}

	public static void main(String[] args) throws Exception {
		Configuration conf = new Configuration();
		Job job = Job.getInstance(conf, "SmallFilesToSequenceFileConverter");

		job.setJarByClass(SmallFilesToSequenceFileConverter.class);
		
		
		
		FileInputFormat.addInputPath(job, new Path(args[0]));
		job.setInputFormatClass(WholeFileInputFormat.class);
		
		
		job.setMapperClass(SequenceFileMapper.class);

		// 设置最终的输出
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(Text.class);//如果不是文本文件，而是字节文件，此处和其他相关部分需要修改为BytesWritable
		job.setOutputFormatClass(TextOutputFormat.class);//MR默认就是采用此种输出文件格式。如果是字节文件，此处要变为SequenceFileOutputFormat
		

		Path outputPath = new Path(args[1]);
		FileSystem fileSystem = outputPath.getFileSystem(conf);
		if (fileSystem.exists(outputPath))
			fileSystem.delete(outputPath, true);
		FileOutputFormat.setOutputPath(job, outputPath);

		System.exit(job.waitForCompletion(true) ? 0 : 1);
	}
}

```

## 运行结果
- 合并后的文件（用的是上一篇SecondSortMapReduce的两个数据，如图，将文件里内容进行合并，并存在hdfs 中）
![-w951](/img/blog_img/15722578167336.jpg)
![-w1552](/img/blog_img/15722579781659.jpg)


