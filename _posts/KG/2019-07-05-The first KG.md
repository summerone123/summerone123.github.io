---

layout:     post
title:      The first KG (Knowledgable Graph)
subtitle:   心得记录
date:       2019-12-26
author:     Yi Xia
header-img: img/post-bg-map.jpg
catalog: true
tags:
    - 知识图谱
---
#  The first KG (Knowledgable Graph)
> 基于图数据库实践股票数据的知识图谱（对于理论部分，可见我的另一篇 blog [http://summerone.xyz/2019/12/23/cognization-on-KG/](http://summerone.xyz/2019/12/23/cognization-on-KG/)）

## 知识的爬取

### 任务1：从⽹页中抽取董事会的信息
在给定的html文件中，需要对每一个股票/公司抽取董事会成员的信息，这部分信息包括董事会成员“姓名”、“职务”、“性别”、“年龄”共四个字段。首先，姓名和职务的字段来自于：

![-w768](/img/blog_img/15622850873404.jpg)


在这里总共有12位董事成员的信息，都需要抽取出来。另外，性别和年龄字段也可以从下附图里抽取出来：

![-w783](/img/blog_img/15622851205385.jpg)


最后，生成一个 `executive_prep.csv`文件，格式如下：

| 高管姓名 | 性别 | 年龄 | 股票代码 |    职位     |
| :------: | :--: | :--: | :------: | :---------: |
|  朴明志  |  男  |  51  |  600007  | 董事⻓/董事 |
|   高燕   |  女  |  60  |  600007  |  执⾏董事   |
|  刘永政  |  男  |  50  |  600008  | 董事⻓/董事 |
|   ···    | ···  | ···  |   ···    |     ···     |

注：建议表头最好用相应的英文表示，不然编码可能出现相关问题。
> 一共爬取的是 24776 个页面,
> 操作方式 : 直接运行 extract.py ,生成 executive_prep.csv

### 任务2：获取股票行业和概念的信息

对于这部分信息，我们可以利⽤工具`Tushare`来获取，官网为http://tushare.org/ ，使用pip命令进行安装即可。下载完之后，在python里即可调用股票行业和概念信息。参考链接：http://tushare.org/classifying.html#id2

通过以下的代码即可获得股票行业信息，并把返回的信息直接存储在`stock_industry_prep.csv`文件里。

```python
import tushare as ts
df = ts.get_industry_classified()
# TODO 保存到"stock_industry_prep.csv"
```

类似的，可以通过以下代码即可获得股票概念信息，并把它们存储在`stock_concept_prep.csv`文件里。

```python
df = ts.get_concept_classified()
# TODO 保存到“stock_concept_prep.csv”
```

> 使用方法:运行 stock.py,生成两个csv 文件
## 知识图谱的设计
![-w1446](/img/blog_img/15622859423594.jpg)

设计一个这样的图谱：

- 创建“人”实体，这个人拥有姓名、性别、年龄

- 创建“公司”实体，除了股票代码，还有股票名称

- 创建“概念”实体，每个概念都有概念名

- 创建“行业”实体，每个行业都有⾏业名

- 给“公司”实体添加“ST”的标记，这个由LABEL来实现

- 创建“人”和“公司”的关系，这个关系有董事长、执行董事等等
- 创建“公司”和“概念”的关系

- 创建“公司”和“行业”的关系


注：实体名字和关系名字需要易懂，对于上述的要求，并不一定存在唯一的设计，只要能够覆盖上面这些要求即可。“ST”标记是⽤用来刻画⼀个股票严重亏损的状态，这个可以从给定的股票名字前缀来判断，背景知识可参考百科[ST股票](https://baike.baidu.com/item/ST%E8%82%A1%E7%A5%A8/632784?fromtitle=ST%E8%82%A1&fromid=2430646)，“ST”股票对应列表为['\*ST', 'ST', 'S*ST', 'SST']。 

### 任务4：创建可以导⼊Neo4j的csv文件

在前两个任务里，我们已经分别生成了 `executive_prep.csv`, `stock_industry_prep.csv`, `stock_concept_prep.csv`，但这些文件不能直接导入到Neo4j数据库。所以需要做⼀些处理，并生成能够直接导入Neo4j的csv格式。
我们需要生成这⼏个文件：`executive.csv`,  `stock.csv`, `concept.csv`, `industry.csv`, `executive_stock.csv`, 
`stock_industry.csv`, `stock_concept.csv`。对于格式的要求，请参考：https://neo4j.com/docs/operations-manual/current/tutorial/import-tool/
![-w359](/img/blog_img/15622861195563.jpg)
> 导入数据时遇到了许多坑,其中格式占了很大的比例,导入图数据库的方法有很多,又通过内部的命令调入的,也有通过外部命令行方式调入的

**常见的如下**![-w984](/img/blog_img/15623903609412.jpg)
> **遇到的问题** 
> 1. 生成的 csv 路径问题,不要放在 import 下,直接放在 bin 下,生成后在删除
> 2. 在插入多个文件时,有重复的键值,图数据库是不允许的,此时要加入参数来避免
```
--ignore-duplicate-nodes=true
```
> 3. 要注意,--后直接加入的是功能选项,不然会给你报空格的错
> 

```
./neo4j-admin import --nodes executive.csv --nodes stock.csv --nodes concept.csv --nodes industry.csv --relationships executive_stock.csv --relationships stock_industry.csv --relationships stock_concept.csv --ignore-duplicate-nodes=true
```

## 知识图谱的展示
**企业与公司之间的关系**
![上海的 5G所有公司及相关个人](/img/blog_img/%E4%B8%8A%E6%B5%B7%E7%9A%84%205G%E6%89%80%E6%9C%89%E5%85%AC%E5%8F%B8%E5%8F%8A%E7%9B%B8%E5%85%B3%E4%B8%AA%E4%BA%BA.jpg)

**上海的 5G所有公司及相关个人：**
![企业与公司之间的关系](/img/blog_img/%E4%BC%81%E4%B8%9A%E4%B8%8E%E5%85%AC%E5%8F%B8%E4%B9%8B%E9%97%B4%E7%9A%84%E5%85%B3%E7%B3%BB.jpg)

## 知识图谱的运用（基于 Neo4j 的query查询语句）
### 图谱展示
- 如何将 Tom hanks 介绍给 Tom keuis

```sql
MATCH (tom:Person {name:"Tom Hanks"})-[:ACTED_IN]->(m)<-[:ACTED_IN]-(coActors),
      (coActors)-[:ACTED_IN]->(m2)<-[:ACTED_IN]-(cruise:Person {name:"Tom Cruise"})
RETURN tom, m, coActors, m2, cruise
```
![演员推荐](/img/blog_img/%E6%BC%94%E5%91%98%E6%8E%A8%E8%8D%90.jpg)

- 4 度关系

```sql
MATCH (bacon:Person {name:"Kevin Bacon"})-[*1..4]-(hollywood)
RETURN DISTINCT hollywood

```
![电影拍摄中所有的四度关系](/img/blog_img/%E7%94%B5%E5%BD%B1%E6%8B%8D%E6%91%84%E4%B8%AD%E6%89%80%E6%9C%89%E7%9A%84%E5%9B%9B%E5%BA%A6%E5%85%B3%E7%B3%BB.jpg)
- 最短路径

```sql
MATCH p=shortestPath(
  (bacon:Person {name:"Kevin Bacon"})-[*]-(meg:Person {name:"Meg Ryan"})
)
RETURN p
```
![最短路径](/img/blog_img/%E6%9C%80%E7%9F%AD%E8%B7%AF%E5%BE%84.jpg)

### 相关问题查询
基于构建好的知识图谱，通过编写Cypher语句回答如下问题
(1)  有多少个公司目前是属于 “ST”类型的？
> match (n:ST) return count(distinct(n))
> 104

(2) “600519”公司的所有独立董事人员中，有多少人同时也担任别的公司的独立董事职位？
> MATCH (m:Company{code:'600519'})<-[:employ_of{jobs:'独立董事'}]-(n:Person)-[:employ_of{jobs:'独立董事'}]->(q:Company)
RETURN count(distinct(n))
> 3

(3) 有多少公司既属于环保行业，又有外资背景？
> MATCH (:Concept{name:'外资背景'})<-[:concept_of]-(m:Company)-[:industry_of]-(:Industry{name:'环保行业'})
RETURN count(distinct(m))
> 0

(4) 对于有锂电池概念的所有公司，独立董事中女性人员比例是多少？
> MATCH (m:Concept{name:'锂电池'})<-[:concept_of]-(n:Company)<-[:employ_of{jobs:'独立董事'}]-(p:Person{gender:'女'})
MATCH (m:Concept{name:'锂电池'})<-[:concept_of]-(n:Company)<-[:employ_of{jobs:'独立董事'}]-(p2:Person)
RETURN count(distinct(p))*1.0/count(distinct(p2))
> 0.3541666666666667











