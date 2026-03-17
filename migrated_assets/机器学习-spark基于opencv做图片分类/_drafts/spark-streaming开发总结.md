---
title: spark streaming开发总结
tags:
---
spark streaming的流式处理是一种基于micro-batch的想法
分为有状态流式处理和无状态流式处理
在有状态流式处理下 
满足以下任意一个条件需要开启checkpoingting
- 使用了 stateful 转换 - 如果 application 中使用了updateStateByKey或reduceByKeyAndWindow等 stateful 操作，必须提供 checkpoint目录来允许定时的RDD checkpoint
- 希望能从意外中恢复 driver

如果 streaming app 没有 stateful 操作，也允许 driver 挂掉后再次重启的进度丢失，就没有启用 checkpoint的必要了。
需要注意的是，随着 streaming application 的持续运行，checkpoint 数据占用的存储空间会不断变大。因此，需要小心设置checkpoint 的时间间隔。设置得越小，checkpoint 次数会越多，占用空间会越大；如果设置越大，会导致恢复时丢失的数据和进度越多。一般推荐设置为 batch duration 的5~10倍。

checkpoint 形式
最终 checkpoint 的形式是将类 Checkpoint的实例序列化后写入外部存储，值得一提的是，有专门的一条线程来做将序列化后的 checkpoint 写入外部存储。类 Checkpoint 包含以下数据
![](https://upload-images.jianshu.io/upload_images/204749-f40597d29d729280.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

Checkpoint 的局限
Spark Streaming 的 checkpoint 机制看起来很美好，却有一个硬伤。上文提到最终刷到外部存储的是类 Checkpoint 对象序列化后的数据。那么在 Spark Streaming application 重新编译后，再去反序列化 checkpoint 数据就会失败。这个时候就必须新建 StreamingContext。

针对这种情况，在我们结合 Spark Streaming + kafka 的应用中，我们自行维护了消费的 offsets，这样一来及时重新编译 application，还是可以从需要的 offsets 来消费数据
这种方法应该是比较好的方法

## Spark Streaming 小结
Spark-Streaming获取kafka数据的两种方式-Receiver与Direct的方式，可以从代码中简单理解成Receiver方式是通过zookeeper来连接kafka队列，Direct方式是直接连接到kafka的节点上获取数据了。

###基于Receiver的方式
这种方式使用Receiver来获取数据。Receiver是使用Kafka的高层次Consumer API来实现的。receiver从Kafka中获取的数据都是存储在Spark Executor的内存中的，然后Spark Streaming启动的job会去处理那些数据。

然而，在默认的配置下，这种方式可能会因为底层的失败而丢失数据。如果要启用高可靠机制，让数据零丢失，就必须启用Spark Streaming的预写日志机制（Write Ahead Log，WAL）。
该机制会同步地将接收到的Kafka数据写入分布式文件系统（比如HDFS）上的预写日志中。所以，即使底层节点出现了失败，也可以使用预写日志中的数据进行恢复。 
需要注意的要点
1. Kafka中的topic的partition，与Spark中的RDD的partition是没有关系的。所以，在KafkaUtils.createStream()中，提高partition的数量，
只会增加一个Receiver中，读取partition的线程的数量。不会增加Spark处理数据的并行度。

2. 可以创建多个Kafka输入DStream，使用不同的consumer group和topic，来通过多个receiver并行接收数据。
3. 如果基于容错的文件系统，比如HDFS，启用了预写日志机制，接收到的数据都会被复制一份到预写日志中。因此，在KafkaUtils.createStream()中，设置的持久化级别是StorageLevel.MEMORY_AND_DISK_SER。

###基于Direct的方式
这种新的不基于Receiver的直接方式，是在Spark 1.3中引入的，从而能够确保更加健壮的机制。替代掉使用Receiver来接收数据后，这种方式会周期性地查询Kafka，
来获得每个topic+partition的最新的offset，从而定义每个batch的offset的范围。当处理数据的job启动时，就会使用Kafka的简单consumer api来获取Kafka指定offset范围的数据。

这种方式有如下优点：
1. 简化并行读取：如果要读取多个partition，不需要创建多个输入DStream然后对它们进行union操作。
Spark会创建跟Kafka partition一样多的RDD partition，并且会并行从Kafka中读取数据。所以在Kafka partition和RDD partition之间，有一个一对一的映射关系。

2. 高性能：如果要保证零数据丢失，在基于receiver的方式中，需要开启WAL机制。这种方式其实效率低下，
因为数据实际上被复制了两份，Kafka自己本身就有高可靠的机制，会对数据复制一份，而这里又会复制一份到WAL中。而基于direct的方式，不依赖Receiver，不需要开启WAL机制，只要Kafka中作了数据的复制，那么就可以通过Kafka的副本进行恢复。

3. 一次且仅一次的事务机制：
基于receiver的方式，是使用Kafka的高阶API来在ZooKeeper中保存消费过的offset的。这是消费Kafka数据的传统方式。
这种方式配合着WAL机制可以保证数据零丢失的高可靠性，但是却无法保证数据被处理一次且仅一次，可能会处理两次。因为Spark和ZooKeeper之间可能是不同步的。

4. 降低资源。 
Direct不需要Receivers，其申请的Executors全部参与到计算任务中；而Receiver-based则需要专门的Receivers来读取Kafka数据且不参与计算。因此相同的资源申请，Direct 能够支持更大的业务。

5. 降低内存。 
Receiver-based的Receiver与其他Exectuor是异步的，并持续不断接收数据，对于小业务量的场景还好，如果遇到大业务量时，需要提高Receiver的内存，但是参与计算的Executor并无需那么多的内存。
而Direct 因为没有Receiver，而是在计算时读取数据，然后直接计算，所以对内存的要求很低。实际应用中我们可以把原先的10G降至现在的2-4G左右。

6. 鲁棒性更好。 
Receiver-based方法需要Receivers来异步持续不断的读取数据，因此遇到网络、存储负载等因素，导致实时任务出现堆积，但Receivers却还在持续读取数据，此种情况很容易导致计算崩溃。
Direct 则没有这种顾虑，其Driver在触发batch 计算任务时，才会读取数据并计算。队列出现堆积并不会引起程序的失败。
基于direct的方式，使用kafka的简单api，Spark Streaming自己就负责追踪消费的offset，并保存在checkpoint中。Spark自己一定是同步的，因此可以保证数据是消费一次且仅消费一次。



### spark反压机制详解
对于基于Receiver 形式，我们可以通过配置 spark.streaming.receiver.maxRate 参数来限制每个 receiver 每秒最大可以接收的记录的数据；对于 Direct Approach 的数据接收，我们可以通过配置 spark.streaming.kafka.maxRatePerPartition 参数来限制每次作业中每个 Kafka 分区最多读取的记录条数。

这种限速的弊端很明显，比如假如我们后端处理能力超过了这个最大的限制，会导致资源浪费。需要对每个spark Streaming任务进行压测预估。成本比较高。由此，从1.5开始引入了back pressure，这种机制呢实际上是基于自动控制理论的pid这个概念。我们就简单讲一下其中思路：为了实现自动调节数据的传输速率，在原有的架构上新增了一个名为 RateController 的组件，这个组件继承自 StreamingListener，其监听所有作业的 onBatchCompleted 事件，并且基于 processingDelay 、schedulingDelay 、当前 Batch 处理的记录条数以及处理完成事件来估算出一个速率；这个速率主要用于更新流每秒能够处理的最大记录的条数。这样就可以实现处理能力好的话就会有一个较大的最大值，处理能力下降了就会生成一个较小的最大值。来保证Spark Streaming流畅运行。

spark.streaming.kafka.maxRatePerPartition to set the maximum number of messages per partition per batch.
同时需要打开 sparkConf.set("spark.streaming.backpressure.enabled",”true”)

1. 设置一个batch RDD中的一个分区的最大消息数, 这个数值可以是最优估计值的1.5到2倍
2. 开启背压 会自动调节读取的message数
所以需要我们对于一个最优的估计值要有认识(基于现有逻辑以及kafka峰值)
spark streamning的处理能力受到两方面因素影响
1. kafka的生产消息能力
2. 在一个batch time之间的处理数据的能力
所以中间需要一个背压机制(本质是一个PID控制器)来调节 确保RDD在一个batch time内全部执行完 不产生积压(通过少读取kafka消息来实现)






 默认情况下，Spark Streaming通过Receiver以生产者生产数据的速率接收数据，计算过程中会出现batch processing time > batch interval的情况，其中batch processing time 为实际计算一个批次花费时间， batch interval为Streaming应用设置的批处理间隔。这意味着Spark Streaming的数据接收速率高于Spark从队列中移除数据的速率，也就是数据处理能力低，在设置间隔内不能完全处理当前接收速率接收的数据。如果这种情况持续过长的时间，会造成数据在内存中堆积，导致Receiver所在Executor内存溢出等问题（如果设置StorageLevel包含disk, 则内存存放不下的数据会溢写至disk, 加大延迟）。Spark 1.5以前版本，用户如果要限制Receiver的数据接收速率，可以通过设置静态配制参数“spark.streaming.receiver.maxRate”的值来实现，此举虽然可以通过限制接收速率，来适配当前的处理能力，防止内存溢出，但也会引入其它问题。比如：producer数据生产高于maxRate，当前集群处理能力也高于maxRate，这就会造成资源利用率下降等问题。为了更好的协调数据接收速率与资源处理能力，Spark Streaming 从v1.5开始引入反压机制（back-pressure）,通过动态控制数据接收速率来适配集群数据处理能力。

为了Spark Streaming应用能在生产中稳定、有效的执行，每批次数据处理时间（批处理时间）必须非常接近批次调度的时间间隔（批调度间隔），并且要一直低于批调度间隔。如果批处理时间一直高于批调度间隔，调度延迟就会一直增长并且不会恢复。最终，Spark Streaming应用会变得不再稳定。另一方面，如果批处理时间长时间远小于批调度间隔，就会浪费集群资源。
        当Spark Streaming与Kafka使用Direct API集群时，我们可以很方便的去控制最大数据摄入量--通过一个被称作spark.streaming.kafka.maxRatePerPartition的参数。根据文档描述，他的含义是：Direct API读取每一个Kafka partition数据的最大速率（每秒读取的消息量）。

        配置项spark.streaming.kafka.maxRatePerPartition，对防止流式应用在下边两种情况下出现流量过载时尤其重要：
1.Kafka Topic中有大量未处理的消息，并且我们设置是Kafka auto.offset.reset参数值为smallest，他可以防止第一个批次出现数据流量过载情况。
2.当Kafka 生产者突然飙升流量的时候，他可以防止批次处理出现数据流量过载情况。

        但是，配置Kafka每个partition每批次最大的摄入量是个静态值，也算是个缺点。随着时间的变化，在生产环境运行了一段时间的Spark Streaming应用，每批次每个Kafka partition摄入数据最大量的最优值也是变化的。有时候，是因为消息的大小会变，导致数据处理时间变化。有时候，是因为流计算所使用的多租户集群会变得非常繁忙，比如在白天时候，一些其他的数据应用（例如Impala/Hive/MR作业）竞争共享的系统资源时（CPU/内存/网络/磁盘IO）。

        背压机制可以解决该问题。背压机制是呼声比较高的功能，他允许根据前一批次数据的处理情况，动态、自动的调整后续数据的摄入量，这样的反馈回路使得我们可以应对流式应用流量波动的问题。

        Spark Streaming的背压机制是在Spark1.5版本引进的，我们可以添加如下代码启用改功能：

sparkConf.set("spark.streaming.backpressure.enabled",”true”)
        那应用启动后的第一个批次流量怎么控制呢？因为他没有前面批次的数据处理时间，所以没有参考的数据去评估这一批次最优的摄入量。在Spark官方文档中有个被称作spark.streaming.backpressure.initialRate的配置，看起来是控制开启背压机制时初始化的摄入量。其实不然，该参数只对receiver模式起作用，并不适用于direct模式。推荐的方法是使用spark.streaming.kafka.maxRatePerPartition控制背压机制起作用前的第一批次数据的最大摄入量。我通常建议设置spark.streaming.kafka.maxRatePerPartition的值为最优估计值的1.5到2倍，让背压机制的算法去调整后续的值。请注意，spark.streaming.kafka.maxRatePerPartition的值会一直控制最大的摄入量，所以背压机制的算法值不会超过他。
        另一个需要注意的是，在第一个批次处理完成前，紧接着的批次都将使用spark.streaming.kafka.maxRatePerPartition的值作为摄入量。通过Spark UI可以看到，批次间隔为5s，当批次调度延迟31秒时候，前7个批次的摄入量是20条记录。直到第八个批次，背压机制起作用时，摄入量变为5条记录。






