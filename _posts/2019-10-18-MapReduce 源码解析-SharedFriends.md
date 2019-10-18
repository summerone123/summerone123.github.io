---
layout:     post
title:     Hadoop之 SharedFriends 源码详细解析
subtitle:   大数据处理
date:       2018-01-04
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - 大数据处理
---


# Hadoop之MapReduce SharedFriends 源码详细解析
## SharedFriends基本概述
求出哪些人两两之间有共同好友，及他俩的共同好友都是谁 

```
数据（网上通用）
A:B,C,D,F,E,O
B:A,C,E,K
C:F,A,D,I
D:A,E,F,L
E:B,C,D,M,L
F:A,B,C,D,E,O,M
G:A,C,D,E,F
H:A,C,D,E,O
I:A,O
J:B,O
K:A,C,D
L:D,E,F
M:E,F,G
O:A,H,I,J
```
## 整体思路
第一步：
可以把朋友作为key，人作为value
**形成**：友–>（人，人，人）这样的中间结果 ，即*SharedFriendsStepOne.java*

第二步：
把（人，人，人）进行排序，避免重复，然后进行两两匹配形成：(人-人）–>友。
这样的键值对，进行mr统计，最后结果就是两两的共同好友了 。即*SharedFriendsStepTwo.java*
## 源码及其解析
- *SharedFriendsStepOne.java*

```java

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;
import java.net.URI;
import java.net.URISyntaxException;

/**
 * Created by tianjun on 2017/3/20.
 * 
 * 
 */

//每一行代表一个用户和他的好友列表，用户和好友列表用：分割，好友间用，分割
//需要求出这些用户两两之间那些有共同好友，及他俩的共同好友都有谁

public class SharedFriendsStepOne {

	static class SharedFriendsStepOneMapper extends
			Mapper<LongWritable, Text, Text, Text> {
		Text k = new Text();
		Text v = new Text();

		@Override
		//输入：                             a:b,c,d，即a的好友是b,c,d,反向输出(b,a),(c,a),(d,a)
		protected void map(LongWritable key, Text value, Context context)
				throws IOException, InterruptedException {
			String line = value.toString();//每一行转为一个字符串
			String[] person_friends = line.split(":");//分割 key 与value
			String person = person_friends[0];
			String[] friends = person_friends[1].split(",");
			for (String friend : friends) {
				k.set(friend);
				v.set(person);
				// <好友，用户>
				context.write(k, v);//将key 和 value 互换位置，统计这个好友都和那些人认识
			}
		}
	}

	static class SharedFriendsStepOneReduce extends
			Reducer<Text, Text, Text, Text> {
		@Override
		protected void reduce(Text friend, Iterable<Text> persons,
				Context context) throws IOException, InterruptedException {
			StringBuffer sb = new StringBuffer();
			for (Text person : persons) {
				if (sb.length() != 0) {
					sb.append(",");
				}
				sb.append(person);
			}
			//<张三，李四，王二，刘五> 说明张三是李王刘的共同好友
			context.write(friend, new Text(sb.toString()));//Reduce进行汇总，这些共同好友都有那些人是认识的
		}
	}

	public static void main(String[] args) throws IOException,
			ClassNotFoundException, InterruptedException, URISyntaxException {

		Configuration conf = new Configuration();


		Job job = Job.getInstance(conf);

		job.setJarByClass(SharedFriendsStepOne.class);

		job.setMapperClass(SharedFriendsStepOneMapper.class);
		job.setReducerClass(SharedFriendsStepOneReduce.class);

		// 设置我们的业务逻辑Mapper类的输出key和value的数据类型
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(Text.class);

		// 设置我们的业务逻辑Reducer类的输出key和value的数据类型
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(Text.class);

		// 如果不设置InputFormat，默认就是使用TextInputFormat.class
		// wcjob.setInputFormatClass(CombineFileInputFormat.class);
		// CombineFileInputFormat.setMaxInputSplitSize(wcjob,4194304);
		// CombineFileInputFormat.setMinInputSplitSize(wcjob,2097152);

		
		Path path = new Path(args[1]);
		FileSystem fileSystem = path.getFileSystem(conf);
		if (fileSystem.exists(path)) {
			fileSystem.delete(path, true);
		}
		
		// 指定要处理的数据所在的位置
		FileInputFormat.addInputPath(job, new Path(args[0]));
		// 指定处理完成之后的结果所保存的位置
		FileOutputFormat.setOutputPath(job, new Path(args[1]));

		boolean res = job.waitForCompletion(true);
		System.exit(res ? 0 : 1);
	}

}
```


- **SharedFriendsStepOne.java 计算结果**
```java
A       I,K,C,B,G,F,H,O,D
B       A,F,J,E
C       A,E,B,H,F,G,K
D       G,C,K,A,L,F,E,H
E       G,M,L,H,A,F,B,D
F       L,M,D,C,G,A
G       M
H       O
I       O,C
J       O
K       B
L       D,E
M       E,F
O       A,H,I,J,F
```

- 代码主要注释已经标出
- Map将key 和 value 互换位置，统计这个好友都和那些人认识
- Reduce进行汇总，这些共同好友都有那些人是认识的
- 为了防止b–>c和c–>b这样同一对朋友的重复，所以，下面基于这个结果处理的时候，需要进行排序，这样就能达到没有重复朋友对的出现。


- **SharedFriendsStepTwo.java**

```java

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;
import java.net.URI;
import java.net.URISyntaxException;
import java.util.Arrays;

/**
 * Created by tianjun on 2017/3/20.
 */
public class SharedFriendsStepTwo {

    static class SharedFriendsStepTwoMapper extends Mapper<LongWritable,Text,Text,Text> {
        Text k = new Text();
        Text v = new Text();
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            String line = value.toString();
            String[] friend_persons = line.split("\t"); //划分为两个部分，前面一部分是后面一部分的共同好友。
            String friend = friend_persons[0]; //如a
            String[] persons = friend_persons[1].split(",");//如b，c，d
            //说明a是b，c，d的共同好友
            
            //排序，防止重复的出现
            Arrays.sort(persons);
            for(int i = 0 ; i<persons.length-2;i++){  //将b，c，d两两组合，a是bc的共同好友，也是bd的共同好友，也是cd的共同好友
                for(int j=i+1;j<persons.length-1;j++){
                    //<用户m-用户n,共同好友> ,这样相同的“人-人”对好友发到一起了
                    context.write(new Text(persons[i]+"-"+persons[j]),new Text(friend));
                }
            }
        }
    }

    static class SharedFriendsStepTwoReduce extends Reducer<Text,Text,Text,Text>{
        @Override
        protected void reduce(Text person_person, Iterable<Text> friends, Context context) throws IOException, InterruptedException {
            StringBuffer sb = new StringBuffer();
            for(Text friend : friends){
                sb.append(friend).append(" ");
            }
            context.write(person_person,new Text(sb.toString()));// Reduced <用户m-用户n,共同好友> ,这样相同的“人-人”对好友发到一起后，对于相同两者的所有共同好友进行 Reduced 汇总·
        }
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException, URISyntaxException {
//        String os = System.getProperty("os.name").toLowerCase();
//        if (os.contains("windows")) {
//            System.setProperty("HADOOP_USER_NAME", "root");
//        }

        Configuration conf = new Configuration();

//        conf.set("mapreduce.framework.name","yarn");
//        conf.set("yarn.resourcemanager.hostname","mini01");
//        conf.set("fs.defaultFS","hdfs://mini01:9000/");

//            默认就是local模式
//        conf.set("mapreduce.framework.name","local");
//        conf.set("mapreduce.jobtracker.address","local");
//        conf.set("fs.defaultFS","file:///");


        Job wcjob = Job.getInstance(conf);

       // wcjob.setJar("F:/myWorkPlace/java/dubbo/demo/dubbo-demo/mr-demo1/target/mr.demo-1.0-SNAPSHOT.jar");

        //如果从本地拷贝，是不行的，这时需要使用setJar
        wcjob.setJarByClass(SharedFriendsStepTwoReduce.class);

        wcjob.setMapperClass(SharedFriendsStepTwoMapper.class);
        wcjob.setReducerClass(SharedFriendsStepTwoReduce.class);

        //设置我们的业务逻辑Mapper类的输出key和value的数据类型
        wcjob.setMapOutputKeyClass(Text.class);
        wcjob.setMapOutputValueClass(Text.class);


        //设置我们的业务逻辑Reducer类的输出key和value的数据类型
        wcjob.setOutputKeyClass(Text.class);
        wcjob.setOutputValueClass(Text.class);


        //如果不设置InputFormat，默认就是使用TextInputFormat.class
//        wcjob.setInputFormatClass(CombineFileInputFormat.class);
//        CombineFileInputFormat.setMaxInputSplitSize(wcjob,4194304);
//        CombineFileInputFormat.setMinInputSplitSize(wcjob,2097152);
        
		Path path = new Path(args[1]);
		FileSystem fileSystem = path.getFileSystem(conf);
		if (fileSystem.exists(path)) {
			fileSystem.delete(path, true);
		}

        //指定要处理的数据所在的位置
        FileInputFormat.setInputPaths(wcjob, new Path(args[0]));
        //指定处理完成之后的结果所保存的位置
        FileOutputFormat.setOutputPath(wcjob, new Path(args[1]));

        boolean res = wcjob.waitForCompletion(true);
        System.exit(res ? 0 : 1);
    }

}
```

- 代码主要注释已经标出
- Map为了防止b–>c和c–>b这样同一对朋友的重复，结果处理的时候，需要进行排序，这样就能达到没有重复朋友对的出现。   然后再进行 map 两两匹配，发现每一个用户是那两个人的共同好友
- Reduced 通过对 map 形式<用户m-用户n,共同好友>进行 reduce 汇总 ,这样相同的“人-人”对好友发到一起后，对于相同两者的**所有**共同好友进行汇总
- 最后计算得出的两俩好友如下：

```shell
[root@bd2016-11 ~]# hdfs dfs -cat /wc/friends/steptwo/*
A-B     C E 
A-C     F D 
A-D     E F 
A-E     B C D 
A-F     C D B E O 
A-G     D E F C 
A-H     E O C D 
A-I     O 
A-K     D 
A-L     F E 
B-C     A 
B-D     E A 
B-E     C 
B-F     E A C 
B-G     C E A 
B-H     E C A 
B-I     A 
B-K     A 
B-L     E 
C-D     F A 
C-E     D 
C-F     D A 
C-G     F A D 
C-H     A D 
C-I     A 
C-K     D A 
C-L     F 
D-F     E A 
D-G     A E F 
D-H     A E 
D-I     A 
D-K     A 
D-L     F E 
E-F     C D B 
E-G     D C 
E-H     D C 
E-K     D 
F-G     C E D A 
F-H     C A D E O 
F-I     A O 
F-K     D A 
F-L     E 
G-H     D E C A 
G-I     A 
G-K     A D 
G-L     F E 
H-I     A O 
H-K     A D 
H-L     E 
I-K     A 

```

