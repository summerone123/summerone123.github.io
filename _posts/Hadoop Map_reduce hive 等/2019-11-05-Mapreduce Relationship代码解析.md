---
layout:     post
title:     Mapreduce Relationship代码解析
subtitle:   大数据处理
date:       2019-09-05
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - 大数据处理
---

# Mapreduce Relationship祖孙关系（单表关联）
## 概述
### 输入文件内容如下:

```
child    parent
Steven   Lucy
Steven   Jack
Jone     Lucy
Jone     Jack
Lucy     Mary
Lucy     Frank
Jack     Alice
Jack     Jesse
David    Alice
David    Jesse
Philip   David
Philip   Alma
Mark     David
Mark     Alma
```
### 要求
根据父辈和子辈挖掘爷孙关系，比如：

```
Steven   Jack
Jack     Alice
Jack     Jesse
```

根据这三条记录，可以得出Jack是Steven的长辈，而Alice和Jesse是Jack的长辈，很显然Steven是Alice和Jesse的孙子。挖掘出的结果如下：

```
grandson    grandparent
Steven      Jesse
Steven      Alice
```

## 分析

解决这个问题要用到一个小技巧，就是单表关联。具体实现步骤如下，Map阶段每一行的key-value输入，同时也把value-key输入。以其中的两行为例：

```
Steven   Jack
Jack     Alice
```
key-value和value-key都输入，变成4行：

```
Steven   Jack
Jack     Alice
Jack     Steven  
Alice    Jack

```
shuffle以后，Jack作为key值，起到承上启下的桥梁作用，Jack对应的values包含Alice、Steven，这时候Alice和Steven肯定是爷孙关系。为了标记哪些是孙子辈，哪些是爷爷辈，可以在Map阶段加上前缀，比如小辈加上前缀”-“，长辈加上前缀”+”。加上前缀以后，在Reduce阶段就可以根据前缀进行分类。

## 源码及其及其解析

```java
package com.javacore.hadoop;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;
import java.util.ArrayList;
import java.util.StringTokenizer;


/**
 * Created by bee on 3/29/17.
 */
public class RelationShip {

    public static class RsMapper extends Mapper<Object, Text, Text, Text> {

        private static int linenum = 0;

        public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
            String line = value.toString();
            if (linenum == 0) {
                ++linenum;
            } else {
                StringTokenizer tokenizer = new StringTokenizer(line, "\n");
                while (tokenizer.hasMoreElements()) {
                    StringTokenizer lineTokenizer = new StringTokenizer(tokenizer.nextToken());
                    String son = lineTokenizer.nextToken();
                    String parent = lineTokenizer.nextToken();
                    context.write(new Text(parent), new Text(
                            "-" + son));
                    context.write(new Text(son), new Text
                            ("+" + parent));
                }
            }

        }
    }

    public static class RsReducer extends Reducer<Text, Text, Text, Text> {
        private static int linenum = 0;

        public void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {

            if (linenum == 0) {
                context.write(new Text("grandson"), new Text("grandparent"));
                ++linenum;
            }
            ArrayList<Text> grandChild = new ArrayList<Text>();
            ArrayList<Text> grandParent = new ArrayList<Text>();

            for (Text val : values) {
                String s = val.toString();

                if (s.startsWith("-")) {
                    grandChild.add(new Text(s.substring(1)));
                } else {
                    grandParent.add(new Text(s.substring(1)));
                }
            }

            for (Text text1 : grandChild) {
                for (Text text2 : grandParent) {
                    context.write(text1, text2);
                }
            }


        }


    }

    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {

        
        Configuration cong = new Configuration();

        String[] otherArgs = new String[]{"C:/Users/student/Desktop/input/input6/input.txt",
                "C:/Users/student/Desktop/input/input6/output.txt"};
        if (otherArgs.length != 2) {
            System.out.println("参数错误");
            System.exit(2);
        }

        Job job = Job.getInstance();
        job.setJarByClass(RelationShip.class);
        job.setMapperClass(RsMapper.class);
        job.setReducerClass(RsReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(Text.class);

        FileInputFormat.addInputPath(job, new Path(otherArgs[0]));
        FileOutputFormat.setOutputPath(job, new Path(otherArgs[1]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);

    }
}
```
## 输出
![-w770](/img/blog_img/15729229965508.jpg)
![-w1069](/img/blog_img/15729230089179.jpg)

`