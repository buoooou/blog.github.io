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

### spark运行架构

**spark 运行流程：**

Spark架构采用了分布式计算中的Master-Slave模型。Master是对应集群中的含有Master进程的节点，Slave是集群中含有Worker进程的节点。

- Master作为整个集群的控制器，负责整个集群的正常运行；
- 
- Worker相当于计算节点，接收主节点命令与进行状态汇报；
- 
- Executor负责任务的执行；
- 
- Client作为用户的客户端负责提交应用；
- 
- Driver负责控制一个应用的执行。


Spark集群部署后，需要在主节点和从节点分别启动Master进程和Worker进程，对整个集群进行控制。在一个Spark应用的执行过程中，Driver和Worker是两个重要角色。Driver 程序是应用逻辑执行的起点，负责作业的调度，即Task任务的分发，而多个Worker用来管理计算节点和创建Executor并行处理任务。在执行阶段，Driver会将Task和Task所依赖的file和jar序列化后传递给对应的Worker机器，同时Executor对相应数据分区的任务进行处理。

1. Excecutor /Task 每个程序自有，不同程序互相隔离，task多线程并行
1. 集群对Spark透明，Spark只要能获取相关节点和进程
1. Driver 与Executor保持通信，协作处理

**三种集群模式：**

1.Standalone 独立集群
2.Mesos, apache mesos
3.Yarn, hadoop yarn

**基本概念：**

1. Application =>Spark的应用程序，包含一个Driver program和若干Executor
1. SparkContext => Spark应用程序的入口，负责调度各个运算资源，协调各个Worker Node上的Executor
1. Driver Program => 运行Application的main()函数并且创建SparkContext
1. Executor => 是为Application运行在Worker node上的一个进程，该进程负责运行Task，并且负责将数据存在内存或者磁盘上。每个Application都会申请各自的Executor来处理任务
1. Cluster Manager =>在集群上获取资源的外部服务 (例如：Standalone、Mesos、Yarn)
1. Worker Node => 集群中任何可以运行Application代码的节点，运行一个或多个Executor进程
1. Task => 运行在Executor上的工作单元
1. Job => SparkContext提交的具体Action操作，常和Action对应
1. Stage => 每个Job会被拆分很多组task，每组任务被称为Stage，也称TaskSet
1. RDD => 是Resilient distributed datasets的简称，中文为弹性分布式数据集;是Spark最核心的模块和类
1. DAGScheduler => 根据Job构建基于Stage的DAG，并提交Stage给TaskScheduler
1. TaskScheduler => 将Taskset提交给Worker node集群运行并返回结果 
1. Transformations => 是Spark API的一种类型，Transformation返回值还是一个RDD，所有的Transformation采用的都是懒策略，如果只是将Transformation提交是不会执行计算的
1. Action => 是Spark API的一种类型，Action返回值不是一个RDD，而是一个scala集合；计算只有在Action被提交的时候计算才被触发。

**Spark核心概念之Jobs / Stage**

Job => 包含多个task的并行计算，一个action触发一个job

stage => 一个job会被拆为多组task，每组任务称为一个stage，以shuffle进行划分

**Spark 资源调优**

Executor的内存主要分为三块：

第一块是让task执行我们自己编写的代码时使用，默认是占Executor总内存的20%；

第二块是让task通过shuffle过程拉取了上一个stage的task的输出后，进行聚合等操作时使用，默认也是占Executor总内存的20%；

第三块是让RDD持久化时使用，默认占Executor总内存的60%。

每个task以及每个executor占用的内存需要分析一下。每个task处理一个partiiton的数据，分片太少，会造成内存不够。


