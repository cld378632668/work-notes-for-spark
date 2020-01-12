# 笔者背景

在某大厂工作时部门架构使用了自研的X大数据计算框架，布置给我一个对其进行调优的工作，该计算框架和 Spark 很像，我进行调优后， 批量自动化执行脚本中的 23 个任务处理时间缩短 2%-67%，超过一半的任务运行时间缩短 35%以上。

# 前言

面试中如果考察计算框架的调优思路该如何回答？调优思路主要从内存、CPU和网络带宽等角度来考虑。本文以 Spark 为例，来看看计算框架进行调优的思路。

## Spark 的架构

在研究每个系统前，我们应该先看看它的架构。Spark是内存计算框架，适用于多次迭代的计算模型，诸如各种机器学习算法。Spark里面的一个核心的概念就是RDD，弹性分布式数据集。Spark支持内存计算模型，用户可以指定存储的策略，当内存不够的时候，可以放置到磁盘上。并且Spark提供了一组RDD的接口，Transformations和Action。Transformations是把一个RDD转换成为另一个RDD以便形成Lineage血统链，这样当数据发生错误的时候可以快速的依靠这种继承关系恢复数据。Action操作是启动一个Job并开始真正的进行一些计算并把返回的结果可以给Driver或者是缓存在worker里面。


### 程序示例

创建一个 Spark 独立应用程序的三个主要步骤：

1、创建SparkConf对象，进行配置
```
val conf = new SparkConf().setAppName("CountingSheep").setMaster("local[2]").set("spark.executor.memory","1g")
```
2、用SparkConf创建SparkContext，告诉Spark如何访问集群
> val sc = new SparkContext(conf)

SparkContext是spark程序所有功能的唯一入口，无论是采用Scala,java,python,R等都必须有一个SprakContext. 
 - SparkContext核心作用：初始化spark应用程序运行所需要的核心组件，包括DAGScheduler,TaskScheduler,SchedulerBackend 
 - 同时还会负责Spark程序往Master注册程序等； 
 - SparkContext是整个应用程序中最为至关重要的一个对象； 

3、 通过SparkContext来创建RDD和计算	
>s c.textFile(filePath).flatMap(_.split(" ")).filter(_ => _ != " ").map(key => (key, 1)).reduceByKey(_+_).collect.foreach(println)

##  内存角度调优：

### 通用角度
1、合理设计数据结构，详见另一篇文章《开发时如何合理设计数据结构减少不必要的内存开销？》
-
2、JVM 调优，详见另一篇文章《如何合理节省 JVM 的内存使用》

### Spark 角度

1、避免创建重复的 RDD

## 网络带宽调优：

1、使用更好的序列化工具，如配置Spark 支持的 Kyro 序列化来替代默认序列化进行序列化。具体配置如下：
> conf.set( "spark.serializer", "org.apache.spark.serializer.KryoSerializer" )

此单项优化将超过一半的用例的运行时间缩短了 30%以上。


## CPU调优： 
1、配置合理的任务并行度。

2、配合合理的系统 CPU 资源使用率。

# 项目过程回顾

拿到任务后我分几个步骤了来完成这个项目。

第一步，我先看了产品说明文档和内部博客上同事们相关的经验贴，对如何配置、编译、运行和使用有了大致了解。

第二步，阅读了逐流大数据框架（如 Spark）官方的调优指南，列出了一个基本的 checkList。

第三步，将主要的代码浏览一遍，对发现的如下问题进行整改
- 发现序列化使用了 Spark 默认的`ObjectOutputStream`框架，果断决定将其换成`Kyro`，然后测试效果，此单项优化将超过一半的用例的运行时间缩短了 30%以上。
- 垃圾回收的开销和对象个数成正比，减少对象的个数（比如用`Int`数组取代`LinkedList`）能大大减少垃圾回收的开销。具体而言，设计数据结构的时候，优先使用对象数组和原生类型，减少对复杂集合类型（如：HashMap）的使用。fastutil库提供了一些很方便的原声类型集合，同时兼容Java标准库。

第四部，check JVM的配置。
- set the JVM flag -XX:+UseCompressedOops into the configs，for example spark-env.sh of Spark.
- 