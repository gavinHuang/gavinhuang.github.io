---
layout: post
title: "hive on spark"
date: 2017-04-11 13:28:07 +0800
comments: true
categories: 
---
如果要问我使用Hive的最大的感受，那第一个出现的答案一定是：慢。
不仅我这么说，Hive自己觉得挺过意不去的，所以他会建议你更换一个引擎：

```
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. 
Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
```
本文只有一个目的：告诉你Hive切换成Spark引擎很简单，远远比官方文档描述得简单。

<!-- more -->

##背景
随着产品销量的提升，我负责的数据仓库系统数据量越来越大; 2016年针对一些SQL语句进行了优化，成果也不错； 但是今年准备对用户留存信息建立模型，进行分析或者预测。这个过程涉及很多数据处理，包括导入/导出/关联等等。这个时候Hive的速度慢的特点就特别明显，常常是执行一个查询之后要等半个小时才能出结果，有时候等到结果的时候都忘了开始的时候是执行什么查询了。

俗话说，磨刀不误砍柴功，所以我觉得有必要在继续其他工作前先解决Hive的问题。

方法很简单，按照Hive的警告信息建议的那样，换个引擎就好了。Tez我不熟，而Spark则相对熟悉一些，之前已经用spark+yarn做数据提取（ETL）的工作，所以Spark成了唯一的选择。

教程官网上也有：
https://cwiki.apache.org/confluence/display/Hive/Hive+on+Spark%3A+Getting+Started

但是我个人觉得这不是一个很好的官方教程，很少有逻辑关系的演进，只有一个个的信息点，而且把必须的步骤和调优的步骤混为一谈，让初学者看得云里雾里。所以我觉得很有必要整理一个简洁的说明，不一定能解决所有问题，但是能对有切换Hive引擎的朋友传达一个清晰的步骤。

##步骤:只有一步

是的，你没看错，Hive on Spark就是这么简单，只要一步就完成了：
```
set hive.execution.engine=spark;
```
**为什么这么简单？**

这是由于Hive（我猜看到这篇文章的你，不会是基于Hive1.1之前的版本了）已经做了集成的工作。比如这个类：

`org.apache.hadoop.hive.ql.exec.spark.SparkTask`

就已经是在Hive的lib里的了。

##问题
现实里当然不会这么简单，不然也不会有本文了。当你执行完上述命令之后，想执行任意的HQL查询，会碰到各种各样的错误。而在我看来这些问题的核心问题就是：

> **版本问题**
> **版本问题**
> **版本问题**

P.S: 我这里的假设是你的环境里已经有了Spark系统。

所以Hive on Spark的核心问题就是找到一个匹配的Spark依赖包，这点官网文档没有特别提及。

问题分为两个子问题：

>**1. 找到你的Hive能匹配的Spark版本**
>**2. 编译一个不带Hive库的Spark包（防止跟Hive运行时的类冲突）**
下面具体介绍这两个步骤。

##真实步骤：有两部

###1、找到合适的Spark版本
一定要特别注意第一个问题，官方文档里轻描淡写的一句话，却是问题的核心：

```
Install/build a compatible version
```
一定要找到跟Hive匹配的Spark版本，这个可以从Hive源代码的pom.xml文件里找到，以我的Hive 2.0.0来说，pom.xml里指定的Spark版本是1.5.0。

但是加入你已经搭建的Spark系统版本不合适，也不用急着替换掉，这里指的Spark版本，只是作为Hive的依赖库，而不是实际运行的Spark版本。所以可能会出现这样的情况：Hive里以来的Spark是1.5，然后提交任务到Spark1.6.1的集群里，照样能正常工作。

###2、编译一个不带Hive的Spark
这个步骤官方也说了：
```
./make-distribution.sh --name "hadoop2-without-hive" --tgz "-Pyarn,hadoop-provided,hadoop-2.4,parquet-provided"
```

但是没有说明的是，这里的hadoop2.4只是pom.xml里的一个任务名字，如果你的hadoop是2.4，可以直接用这个命令，但是如果是2.6+（比如像我的2.7）的话，这要将hadoop-2.4换成hadoop2.6。

另外，也特别注意另外几个版本问题：

> **1. Java 版本**
> **2. Scala版本**
> **3. Maven版本**

这个pom.xml文件里指定了Java版本为1.7，我的系统里版本是1.8，所以要修改pom.xml，修改java版本为1.8。另外我的系统里，make-distribution.sh不能自动识别JAVA_HOME环境变量，所以要在脚本里指定JAVA_HOME的至。
第二，由于默认Scala版本为2.10，而我的scala为2.11，所以需要调用一个脚本修改下依赖的scala版本：
```
./dev/change-scala-version.sh 2.11
```
第三个问题，maven的版本问题。编译脚本会自动下载Maven和Scala，但是如果系统自带的Maven在PATH目录里的话，则还是会用系统的版本。我的系统版本为3.0.5，编译中一直报告莫名其妙的错误。将Maven升级到3.3.9之后则问题自动消失。

##最后一步：使用Spark依赖包
一旦Spark包编译完成，这可以直接拷贝或者建立软链接到Hive的lib目录里。
这样Hive启动后，使用本文开始的命令：
```
set hive.execution.engine=spark;
```
之后的查询就会使用spark来执行命令了。

##速度的提升
Spark之所以能提升Hive查询的速度，本质有两个原因：
>1. 缓存在内存中
>2. 省略中间shuffle的过程

这也提示了只有在有多个查询子过程的HQL的时候和/或内存中的数据能被第二次使用到的时候，查询速度才会提高。
比如这样一个简单的句子：
```
select count(*) from table_1;
```
在第一次查询时，使用Spark引擎和使用MR引擎并没有太多的速度差别。但是spark的好处在于，第二次查询的时候，MR引擎话费的时间跟第一次一样，而Spark引擎的查询速度则有数量级的优化。

另外，在有关联操作、Group操作、Union操作时，Spark引擎能立刻提升执行速度。速度会比MR引擎快接近10倍左右，当然具体提升幅度也依赖其他因素。

##跟其他系统的集成
一种比较常见的用法是利用hive跟Hbase/Elasticsearch等系统集成，这样可以通过HQL访问其他系统的数据。
这里以跟Hbase集成为例，需要在HiverServer和Hive Cli的机器中的hive-site.xml中配置
```
<property>
    <name>hive.aux.jars.path</name>
    <value>file:///path/to/hbase-jars, separated by comma.these file can be found at HBASE_HOME/lib/hbase*</value>
    <description>The location of the plugin jars that contain implementations of user defined functions and serdes.</description>
  </property>
```


