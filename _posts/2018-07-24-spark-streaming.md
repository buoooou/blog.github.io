---
layout: post
published: false
title: spark streaming
---
## spark

### Spark 特点

运行速度快 => Spark拥有DAG执行引擎，支持在内存中对数据进行迭代计算。官方提供的数据表明，如果数据由磁盘读取，速度是Hadoop MapReduce的10倍以上，如果数据从内存中读取，速度可以高达100多倍。

适用场景广泛 => 大数据分析统计，实时数据处理，图计算及机器学习

易用性 => 编写简单，支持80种以上的高级算子，支持多种语言，数据源丰富，可部署在多种集群中

容错性高。Spark引进了弹性分布式数据集RDD (Resilient Distributed Dataset) 的抽象，它是分布在一组节点中的只读对象集合，这些集合是弹性的，如果数据集一部分丢失，则可以根据“血统”（即充许基于数据衍生过程）对它们进行重建。另外在RDD计算时可以通过CheckPoint来实现容错，而CheckPoint有两种方式：CheckPoint Data，和Logging The Updates，用户可以控制采用哪种方式来实现容错。

### Spark的适用场景

目前大数据处理场景有以下几个类型：

复杂的批量处理（Batch Data Processing），偏重点在于处理海量数据的能力，至于处理速度可忍受，通常的时间可能是在数十分钟到数小时；

基于历史数据的交互式查询（Interactive Query），通常的时间在数十秒到数十分钟之间

基于实时数据流的数据处理（Streaming Data Processing），通常在数百毫秒到数秒之间