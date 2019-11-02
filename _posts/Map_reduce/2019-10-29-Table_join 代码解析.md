---
layout:     post
title:     Hadoop之Table_join源码详细解析
subtitle:   大数据处理
date:       2019-10-30
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - 大数据处理
---

# Table_ join 源码解析
## Rjoin
### 数据
- 输入
 - 输入 1```
1001,20150710,P0001,20
1002,20150711,P0001,30
1002,20150712,P0002,300
1003,20150713,P0003,250
```</br>
 - 输入 2```
P0001,xiaomi5,1000,2000
P0002,oppoR15,1000,3100
```

- 输出
![-w1440](/img/blog_img/15723500935090.jpg)

### 源码及其解析

```java
package bigdata.hadoop.mr.tablejoin;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import org.apache.commons.beanutils.BeanUtils;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.Writable;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.FileSplit;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

/*
 * 订单格式：订单号 订单日期 商品号 商品数量      （每个订单只能购买一种商品）
 * 商品信息：商品号 商品名称  商品分类  商品单价
 * 
 * 要求输出：订单号 订单日期  商品号 商品名称 商品分类 商品单价
 */


public class RJoin {
	
	public static class InfoBean implements Writable{

	    private int order_id = 0;
	    private String dateString = "";
	    private String p_id = "";
	    private int amount = 0;
	    private String pname = "";
	    private String category_id = "";
	    private float price = 0;
	    private int flag = -1; // =0是订单，=1是产品

	    public InfoBean(){}

	    public InfoBean(int order_id, String dateString, String p_id, int amount) {
	        super();
	        this.order_id = order_id;
	        this.dateString = dateString;
	        this.p_id = p_id;
	        this.amount = amount;
	        this.flag = 0;
	    }

	    public InfoBean(String p_id, String pname, String category_id, float price) {
	        super();
	        this.p_id = p_id;
	        this.pname = pname;
	        this.category_id = category_id;
	        this.price = price;
	        this.flag = 1;
	    }


	    public int getFlag() {
	        return flag;
	    }

	    public void setFlag(int flag) {
	        this.flag = flag;
	    }

	    public int getOrder_id() {
	        return order_id;
	    }

	    public void setOrder_id(int order_id) {
	        this.order_id = order_id;
	    }

	    public String getDateString() {
	        return dateString;
	    }

	    public void setDateString(String dateString) {
	        this.dateString = dateString;
	    }

	    public String getP_id() {
	        return p_id;
	    }

	    public void setP_id(String p_id) {
	        this.p_id = p_id;
	    }

	    public int getAmount() {
	        return amount;
	    }

	    public void setAmount(int amount) {
	        this.amount = amount;
	    }

	    public String getPname() {
	        return pname;
	    }

	    public void setPname(String pname) {
	        this.pname = pname;
	    }

	    public String getCategory_id() {
	        return category_id;
	    }

	    public void setCategory_id(String category_id) {
	        this.category_id = category_id;
	    }

	    public float getPrice() {
	        return price;
	    }

	    public void setPrice(float price) {
	        this.price = price;
	    }

	    @Override
	    public void write(DataOutput out) throws IOException {
	        out.writeInt(order_id);
	        out.writeUTF(dateString);
	        out.writeUTF(p_id);
	        out.writeInt(amount);
	        out.writeUTF(pname);
	        out.writeUTF(category_id);
	        out.writeFloat(price);
	        out.writeInt(flag);
	    }

	    @Override
	    public void readFields(DataInput in) throws IOException {
	        order_id = in.readInt();
	        dateString = in.readUTF();
	        p_id = in.readUTF();
	        amount = in.readInt();
	        pname = in.readUTF();
	        category_id = in.readUTF();
	        price = in.readFloat();
	        flag = in.readInt();
	    }

	    @Override
	    public String toString() {
	        return "order_id=" + order_id + ", dateString=" + dateString
	                + ", p_id=" + p_id + ", amount=" + amount + ", pname=" + pname
	                + ", category_id=" + category_id + ", price=" + price
	                + ", flag=" + flag;
	    }


	}
	
	
	
	
    public static class RJoinMapper extends Mapper<LongWritable, Text, Text, InfoBean>{

        @Override
        protected void map(LongWritable key, Text value, Context context)
                throws IOException, InterruptedException {
            String line = value.toString();
            String[] fields = line.split(",");
            FileSplit inputSplit = (FileSplit) context.getInputSplit();
            String name = inputSplit.getPath().getName();

            InfoBean bean = null;
            // 通过文件名判断是哪种数据
            if(name.startsWith("order")){
                bean = new InfoBean(Integer.parseInt(fields[0]), fields[1],
                        fields[2], Integer.parseInt(fields[3]));
                context.write(new Text(fields[2]), bean);
            }else{
                bean = new InfoBean(fields[0], fields[1], 
                        fields[2], Float.parseFloat(fields[3]));
                context.write(new Text(fields[0]), bean);
            }

        }

    }

    static class RJoinReducer extends Reducer<Text, InfoBean, InfoBean, NullWritable>{

        @Override
        protected void reduce(Text pid, Iterable<InfoBean> beans,Context context)
                throws IOException, InterruptedException {
            InfoBean productBean = new InfoBean();
            List<InfoBean> orderBeans = new ArrayList<InfoBean>();

            try {
                for(InfoBean bean : beans){
                    if(bean.getFlag() == 0){
                        InfoBean orderBean = new InfoBean();
                        BeanUtils.copyProperties(orderBean, bean);      
                        orderBeans.add(orderBean);
                    }else{
                        BeanUtils.copyProperties(productBean, bean);
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }

            for(InfoBean orderBean : orderBeans){
                // 把orderBean的数据拷贝到productBean里面
                productBean.setOrder_id(orderBean.getOrder_id());
                productBean.setDateString(orderBean.getDateString());
                productBean.setP_id(orderBean.getP_id());
                productBean.setAmount(orderBean.getAmount());
                context.write(productBean, NullWritable.get());
            }
        }

    }

    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);
        job.setJarByClass(RJoin.class);
        job.setMapperClass(RJoinMapper.class);
        job.setReducerClass(RJoinReducer.class);
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(InfoBean.class);
        job.setOutputKeyClass(InfoBean.class);
        job.setOutputValueClass(NullWritable.class);

        FileInputFormat.addInputPath(job, new Path(args[0]));
        
		Path outputPath= new Path(args[args.length - 1]);
		FileSystem fileSystem = outputPath.getFileSystem(conf);
		if(fileSystem.exists(outputPath))
			fileSystem.delete(outputPath, true);
        FileOutputFormat.setOutputPath(job,outputPath);
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```

## Two TablesJoin1
### 数据

- 输入
 - 输入 1```
4052016005,2016.08.12
4052016006,2016.02.30
4052016003,2016.10.11
4042016001,2018.11.11
```</br>
 - 输入 2```
4052016005,liuchen
4052016006,liuyinxiang
4052016003,heyuchen
4042016001,cengjian
```
- 输出
![-w1440](/img/blog_img/15723506301025.jpg)

### 源码及其解析

```java
package bigdata.hadoop.mr.tablejoin;

import java.io.IOException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.*;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;

//使用join1to1生成数据,处理数据时会出现（id,name,date)和(id,date,name)两种输出，需修正。

public class TwoTablesJoin1
{
	
	public static class MapClass extends 
        Mapper<LongWritable, Text, Text, Text>
    {
        private Text key = new Text();
        private Text value = new Text();
        private String[] keyValue = null;
        
        @Override
        protected void map(LongWritable key, Text value, Context context)
            throws IOException, InterruptedException
        {
            keyValue = value.toString().split(",", 2);
            this.key.set(keyValue[0]);
            this.value.set(keyValue[1]);
            context.write(this.key, this.value);
        }
        
    }
    
    public static class ReduceClass extends Reducer<Text, Text, Text, Text>
    {
        private Text value = new Text();
        
        @Override
        //传入对应一个key的所有的value。
        protected void reduce(Text key, Iterable<Text> values, Context context)
                throws IOException, InterruptedException
        {
            StringBuilder valueStr = new StringBuilder();
            

            for(Text val : values)
            {
                valueStr.append(val);
                valueStr.append(",");
            }
            
            //去掉最后一个逗号
            this.value.set(valueStr.deleteCharAt(valueStr.length()-1).toString());
            context.write(key, this.value);
        }
        
    }
    
    public static void main(String[] args) throws Exception
    {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);
        
        job.setJarByClass(TwoTablesJoin1.class);
        job.setMapperClass(MapClass.class);
        job.setReducerClass(ReduceClass.class);
        
        
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(Text.class);
        
        job.setInputFormatClass(TextInputFormat.class);
        job.setOutputFormatClass(TextOutputFormat.class);
        
        FileInputFormat.addInputPath(job, new Path(args[0]));
        
        Path outputPath = new Path(args[1]);
        FileSystem fileSystem = outputPath.getFileSystem(conf);
		if(fileSystem.exists(outputPath))
			fileSystem.delete(outputPath, true);
        FileOutputFormat.setOutputPath(job,outputPath);

        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}

```

## Two TablesJoin2
> 运用 set 对Two TablesJoin1 ，输出为固定的(id,name,date)，不会出现 1 中的（id,name,date)和(id,date,name)两种输出

### 数据

- 输入
 - 输入 1```
4052016005,2016.08.12
4052016006,2016.02.30
4052016003,2016.10.11
4042016001,2018.11.11
```</br>
 - 输入 2```
4052016005,liuchen
4052016006,liuyinxiang
4052016003,heyuchen
4042016001,cengjian
```
- 输出

![-w1552](/img/blog_img/15723600879214.jpg)


### 源码及其解析

```java
package bigdata.hadoop.mr.tablejoin;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.*;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.FileSplit;
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;
 
//使用join1to1生成数据,通过自定义类型，输出为固定的(id,name,date)，是对1的修正。
public class TwoTablesJoin2
{
	public static class CombineValues implements WritableComparable<CombineValues>{
	    private Text joinKey;
	    private Text flag;
	    private Text secondPart;
	    
	    public void setJoinKey(Text joinKey) {
	        this.joinKey = joinKey;
	    }
	    public void setFlag(Text flag) {
	        this.flag = flag;
	    }
	    public void setSecondPart(Text secondPart) {
	        this.secondPart = secondPart;
	    }
	    public Text getFlag() {
	        return flag;
	    }
	    public Text getSecondPart() {
	        return secondPart;
	    }
	    public Text getJoinKey() {
	        return joinKey;
	    }
	    public CombineValues() {
	        this.joinKey =  new Text();
	        this.flag = new Text();
	        this.secondPart = new Text();
	    }
	                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          
	  
	    public void write(DataOutput out) throws IOException {
	        this.joinKey.write(out);
	        this.flag.write(out);
	        this.secondPart.write(out);
	    }
	  
	    public void readFields(DataInput in) throws IOException {
	        this.joinKey.readFields(in);
	        this.flag.readFields(in);
	        this.secondPart.readFields(in);
	    }
	 
	    public int compareTo(CombineValues o) {
	        return this.joinKey.compareTo(o.getJoinKey());
	    }
	    
	    @Override
	    public String toString() {
	        // TODO Auto-generated method stub
	        return "[flag="+this.flag.toString()+",joinKey="+this.joinKey.toString()+",secondPart="+this.secondPart.toString()+"]";
	    }

	}
	
    public static class Map extends 
        Mapper<LongWritable, Text, Text, CombineValues>
    {
    	private CombineValues combineValues = new CombineValues();
        private Text flag = new Text();
        private Text key = new Text();
        private String[] keyValue = null;
        
        @Override
        //形成(userid,(userid,flag,username))和(userid,(userid,flag,date))的输出。
        //username和date为secondpart
        protected void map(LongWritable key, Text value, Context context)
            throws IOException, InterruptedException
        {
        	String pathName = ((FileSplit) context.getInputSplit()).getPath().toString();
        	if(pathName.endsWith("input1.txt"))
        		flag.set("0");
        	else
        		flag.set("1");
        	
        	combineValues.setFlag(flag);
            keyValue = value.toString().split(",", 2);
            
            combineValues.setJoinKey(new Text(keyValue[0]));
            combineValues.setSecondPart(new Text(keyValue[1]));

            this.key.set(keyValue[0]);
            context.write(this.key, combineValues);
        }
        
    }
    
    public static class Reduce extends Reducer<Text, CombineValues, Text, Text>
    {
        private Text left = new Text();
        //注意和3对比类型。
        private Text right = new Text();
        
        @Override
        //传入对应一个key的所有的value
        protected void reduce(Text key, Iterable<CombineValues> values, Context context)
                throws IOException, InterruptedException
        {
            for(CombineValues val : values)
            {
            	System.out.println("val:" + val.toString());
            	Text secondPar = new Text(val.getSecondPart().toString());
            	if(val.getFlag().toString().equals("0")){
            		System.out.println("left :" + secondPar);
            		left.set(secondPar);
            	}
            	else{
            		System.out.println("right :" + secondPar);
            		//注意，此处是set
            		right.set(secondPar);
            	}
            }
            
            
            Text output = new Text(left.toString() + "," + right.toString());
            
            context.write(key, output);
        }
        
    }
    
    public static void main(String[] args) throws Exception
    {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);
        
        job.setJarByClass(TwoTablesJoin2.class);
        job.setMapperClass(Map.class);
        job.setReducerClass(Reduce.class);
        
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(CombineValues.class);
        
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(Text.class);
        
        job.setInputFormatClass(TextInputFormat.class);
        job.setOutputFormatClass(TextOutputFormat.class);
        
        FileInputFormat.addInputPath(job, new Path(args[0]));

        Path outputPath = new Path(args[1]);
        FileSystem fileSystem = outputPath.getFileSystem(conf);
		if(fileSystem.exists(outputPath))
			fileSystem.delete(outputPath, true);
        FileOutputFormat.setOutputPath(job,outputPath);

        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```

## TwoTablesJoinWithCache
> 特别注意输入的路径,由程序反推路径
> ![-w586](/img/blog_img/15724307187250.jpg)



### 数据
- 输入
 - 输入 1```
4052016005,2016.08.12
4052016006,2016.02.30
4052016003,2016.10.11
4042016001,2018.11.11
```</br>
 - 输入 2```
4052016005,liuchen
4052016006,liuyinxiang
4052016003,heyuchen
4042016001,cengjian
```
- 输出

![-w1552](/img/blog_img/15723609081541.jpg)


### 源码及其解析

```java

package bigdata.hadoop.mr.tablejoin;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.URI;

import java.util.HashMap;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.*;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;

import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;

import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;
//join in map with distributed cache.
public class TwoTablesJoinWithCache {
	public static class Map extends
			Mapper<LongWritable, Text, Text, Text> {
		private Text key = new Text();
		private Text value = new Text();
		private String[] keyValue = null;
		private HashMap<String, String> keyMap = null;

		@Override
		protected void map(LongWritable key, Text value, Context context)
				throws IOException, InterruptedException {
			keyValue = value.toString().split(",", 2);

			//从代表文件的keymap中找id对应的用户名
			String name = keyMap.get(keyValue[0]);
			
			this.key.set(keyValue[0]);
			//名字+日期构成输出的value。
			String output = name + "," + keyValue[1];
			this.value.set(output);
			context.write(this.key, this.value);
		}

		@Override//第一次执行map方法时调用
		protected void setup(Context context) throws IOException,
				InterruptedException {
			URI[] localPaths = context.getCacheFiles();
			
			keyMap = new HashMap<String, String>();
			for(URI url : localPaths){
			     FileSystem fs = FileSystem.get(URI.create("hdfs://hadoop1:9000"), context.getConfiguration());
			     FSDataInputStream in = null;
			     in = fs.open(new Path(url.getPath()));
			     BufferedReader br=new BufferedReader(new InputStreamReader(in));
			     String s1 = null;
			     while ((s1 = br.readLine()) != null)
			     {
			    	 keyValue = s1.split(",", 2);
			    	 
			    	 keyMap.put(keyValue[0], keyValue[1]);
			         System.out.println(s1);
			     }
			     br.close();
			}
		}
	}

	public static class Reduce extends Reducer<Text, Text, Text, Text> {


		@Override
		protected void reduce(Text key, Iterable<Text> values,
				Context context) throws IOException, InterruptedException {
			
			for(Text val : values)
				context.write(key, val);
			
		}

	}
	

	//MR缓存相关程序不能使用插件运行
	//必须在实际环境中运行,本程序运行需要三个参数
	//hdfs://node01:9000/joinwithcache_input 输入文件目录
	//hdfs://node01:9000/joinwithcache_output 输出目录
	//hdfs://node01:9000/joinwithcache_cache/input2.txt 需要分发到各结点的缓存文件地址

	public static void main(String[] args) throws Exception {
		Configuration conf = new Configuration();
		Job job = Job.getInstance(conf);

		job.setJarByClass(TwoTablesJoinWithCache.class);
		job.setMapperClass(Map.class);
		job.setReducerClass(Reduce.class);

		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(Text.class);

		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(Text.class);

		job.setInputFormatClass(TextInputFormat.class);
		job.setOutputFormatClass(TextOutputFormat.class);

		FileInputFormat.addInputPath(job, new Path(args[0]));

        Path outputPath = new Path(args[1]);
        FileSystem fileSystem = outputPath.getFileSystem(conf);
		if(fileSystem.exists(outputPath))
			fileSystem.delete(outputPath, true);
        FileOutputFormat.setOutputPath(job,outputPath);

		job.addCacheFile(new Path(args[2]).toUri());

		System.exit(job.waitForCompletion(true) ? 0 : 1);
	}
}




```