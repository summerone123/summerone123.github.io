---
layout:     post
title:      MapReduce 中block 和 split 的区别
subtitle:   大数据处理
date:       2018-01-04
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - 大数据处理
---
# MapReduce 中block 和 split 的区别

### MapReduce的流程
> ![-w672](/img/blog_img/15681138718403.jpg)

### 分块
在HDFS系统中，为了便于文件的管理和备份，引入分块概念（block）。这里的 块 是HDFS存储系统当中的最小单位，HDFS默认定义一个块的大小为64MB。当有文件上传到HDFS上时，若文件大小大于设置的块大小，则该文件会被切分存储为多个块，多个块可以存放在不同的DataNode上，整个过程中 HDFS系统会保证一个块存储在一个datanode上 。但值得注意的是 如果某文件大小没有到达64MB，该文件并不会占据整个块空间 。

HDFS中的NameNode会记录在上述文件分块中文件的各个块都存放在哪个dataNode上，这些信息一般也称为 元信息(MetaInfo) 。元信息的存储位置由dfs.name.dir指定。
### 分片
当一个作业提交到Hadoop运行的时候，其中的核心步骤是MapReduce，在这个过程中传输的数据可能会很多，Hadoop会将MapReduce的输入数据划分为等长的小数据块，称为输入分片或者分片。

可以通过上述计算了解到，hadoop计算的分片大小不小于blockSize，并且不小于mapred.min.split.size。默认情况下，以HDFS的一个块的大小（默认为64M）为一个分片，即分片大小等于分块大小。当某个分块分成均等的若干分片时，会有最后一个分片大小小于定义的分片大小，则该分片独立成为一个分片。

### 区别
分片是数据处理的输入逻辑划分，分块是数据存储的逻辑划分， 一个是处理 一个是存储。

### 默认两者大小是相同的
hadoop在存储有输入数据（HDFS中的数据）的节点上运行map任务，可以获得高性能，这就是所谓的数据本地化。所以最佳分片的大小应该与HDFS上的块大小一样，因为如果分片跨越2个数据块，对于任何一个HDFS节点（基本不肯能同时存储这2个数据块），分片中的另外一块数据就需要通过网络传输到map任务节点，与使用本地数据运行map任务相比，效率则更低！
