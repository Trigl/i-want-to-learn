Apache Flink是一个面向分布式数据流处理和批量数据处理的开源计算平台，它能够基于同一个Flink运行时引擎（Flink Runtime），提供支持流处理和批处理两种类型应用的功能。

Flink在实现流处理和批处理时，与传统的一些方案完全不同，它从另一个视角看待流处理和批处理，将二者统一起来：Flink是完全支持流处理，也就是说作为流处理看待时输入数据流是无界的；批处理被作为一种特殊的流处理，只是它的输入数据流被定义为有界的。

## 基本特性

- 支持高吞吐、低延迟、高性能的流处理
- 支持高度灵活的窗口（Window）操作，支持基于time、count、session，以及data-driven的窗口操作
- 支持具有Backpressure功能的持续流模型
- 支持基于轻量级分布式快照（Snapshot）实现的容错
- 支持有状态计算的Exactly-once语义
- Flink在JVM内部实现了自己的内存管理
- 一个运行时同时支持Batch on Streaming处理和Streaming处理
- 支持迭代计算
- 支持程序自动优化：避免特定情况下Shuffle、排序等昂贵操作，中间结果有必要进行缓存

## 组件栈
Flink是一个分层架构的系统，每一层所包含的组件都提供了特定的抽象，用来服务于上层组件。Flink分层的组件栈如下图所示：

![](/resource/flink-component-stack.png)

下面，我们自下而上，分别针对每一层进行解释说明：
### Deployment层
该层主要涉及了Flink的部署模式，Flink支持多种部署模式：本地、集群（Standalone/YARN）、云（GCE/EC2）。Standalone部署模式与Spark类似，这里，我们看一下Flink on YARN的部署模式，如下图所示：

![](/resource/flink-on-yarn.png)

了解YARN的话，对上图的原理非常熟悉，实际Flink也实现了满足在YARN集群上运行的各个组件：Flink YARN Client负责与YARN RM通信协商资源请求，Flink JobManager和Flink TaskManager分别申请到Container去运行各自的进程。通过上图可以看到，YARN AM与Flink JobManager在同一个Container中，这样AM可以知道Flink JobManager的地址，从而AM可以申请Container去启动Flink TaskManager。待Flink成功运行在YARN集群上，Flink YARN Client就可以提交Flink Job到Flink JobManager，并进行后续的映射、调度和计算处理。

### Runtime层
Runtime层提供了支持Flink计算的全部核心实现，比如：支持分布式Stream处理、JobGraph到ExecutionGraph的映射、调度等等，为上层API层提供基础服务。

### API层
API层主要实现了面向无界Stream的流处理和面向Batch的批处理API，其中面向流处理对应DataStream API，面向批处理对应DataSet API。

### Libraries层
该层也可以称为Flink应用框架层，根据API层的划分，在API层之上构建的满足特定应用的实现计算框架，也分别对应于面向流处理和面向批处理两类。面向流处理支持：CEP（复杂事件处理）、基于SQL-like的操作（基于Table的关系操作）；面向批处理支持：FlinkML（机器学习库）、Gelly（图处理）。

## 基本概念
### Stream & Transformation & Operator
用户实现的Flink程序是由Stream和Transformation这两个基本构建块组成，其中Stream是一个中间结果数据，而Transformation是一个操作，它对一个或多个输入Stream进行计算处理，输出一个或多个结果Stream。当一个Flink程序被执行的时候，它会被映射为Streaming Dataflow。一个Streaming Dataflow是由一组Stream和Transformation Operator组成，它类似于一个DAG图，在启动的时候从一个或多个Source Operator开始，结束于一个或多个Sink Operator。

下面是一个由Flink程序映射为Streaming Dataflow的示意图，如下所示：

![](/resource/flink-streaming-dataflow-example.png)

上图中，FlinkKafkaConsumer是一个Source Operator，map、keyBy、timeWindow、apply是Transformation Operator，RollingSink是一个Sink Operator。

### Parallel Dataflow
在Flink中，程序天生是并行和分布式的：一个Stream可以被分成多个Stream分区（Stream Partitions），一个Operator可以被分成多个Operator Subtask，每一个Operator Subtask是在不同的线程中独立执行的。一个Operator的并行度，等于Operator Subtask的个数，一个Stream的并行度总是等于生成它的Operator的并行度。

有关Parallel Dataflow的实例，如下图所示：

![](/resource/flink-parallel-dataflow.png)

上图Streaming Dataflow的并行视图中，展现了在两个Operator之间的Stream的两种模式：

- One-to-one模式：比如从Source[1]到map()[1]，它保持了Source的分区特性（Partitioning）和分区内元素处理的有序性，也就是说map()[1]的Subtask看到数据流中记录的顺序，与Source[1]中看到的记录顺序是一致的。
- Redistribution模式：这种模式改变了输入数据流的分区，比如从map()[1]、map()[2]到keyBy()/window()/apply()[1]、keyBy()/window()/apply()[2]，上游的Subtask向下游的多个不同的Subtask发送数据，改变了数据流的分区，这与实际应用所选择的Operator有关系。另外，Source Operator对应2个Subtask，所以并行度为2，而Sink Operator的Subtask只有1个，故而并行度为1。

### Task & Operator Chain
在Flink分布式执行环境中，会将多个Operator Subtask串起来组成一个Operator Chain，实际上就是一个执行链，每个执行链会在TaskManager上一个独立的线程中执行，如下图所示：

![](/resource/flink-tasks-chains.png)

上图中上半部分表示的是一个Operator Chain，多个Operator通过Stream连接，而每个Operator在运行时对应一个Task；图中下半部分是上半部分的一个并行版本，也就是对每一个Task都并行化为多个Subtask。感觉Flink的一个 operator chain 类似于 Spark 的 stage，下面由多个 task 组成。

### Time & Window
Flink支持基于时间窗口操作，也支持基于数据的窗口操作，如下图所示：

![](/resource/flink-window.png)

上图中，基于时间的窗口操作，在每个相同的时间间隔对Stream中的记录进行处理，通常各个时间间隔内的窗口操作处理的记录数不固定；而基于数据驱动的窗口操作，可以在Stream中选择固定数量的记录作为一个窗口，对该窗口中的记录进行处理。

有关窗口操作的不同类型，可以分为如下几种：倾斜窗口（Tumbling Windows，记录没有重叠）、滑动窗口（Slide Windows，记录有重叠）、会话窗口（Session Windows），具体可以查阅相关资料。

在处理Stream中的记录时，记录中通常会包含各种典型的时间字段，Flink支持多种时间的处理，如下图所示：

![](/resource/flink-event-ingestion-processing-time.png)

上图描述了在基于Flink的流处理系统中，各种不同的时间所处的位置和含义，其中，Event Time表示事件创建时间，Ingestion Time表示事件进入到Flink Dataflow的时间 ，Processing Time表示某个Operator对事件进行处理事的本地系统时间（是在TaskManager节点上）。这里，谈一下基于Event Time进行处理的问题，通常根据Event Time会给整个Streaming应用带来一定的延迟性，因为在一个基于事件的处理系统中，进入系统的事件可能会基于Event Time而发生乱序现象，比如事件来源于外部的多个系统，为了增强事件处理吞吐量会将输入的多个Stream进行自然分区，每个Stream分区内部有序，但是要保证全局有序必须同时兼顾多个Stream分区的处理，设置一定的时间窗口进行暂存数据，当多个Stream分区基于Event Time排列对齐后才能进行延迟处理。所以，设置的暂存数据记录的时间窗口越长，处理性能越差，甚至严重影响Stream处理的实时性。

有关基于时间的Streaming处理，可以参考官方文档，在Flink中借鉴了Google使用的WaterMark实现方式，可以查阅相关资料。

## 基本架构
Flink系统的架构与Spark类似，是一个基于Master-Slave风格的架构，如下图所示：

![](/resource/flink-system-architecture.png)

Flink集群启动时，会启动一个JobManager进程、至少一个TaskManager进程。在Local模式下，会在同一个JVM内部启动一个JobManager进程和TaskManager进程。当Flink程序提交后，会创建一个Client来进行预处理，并转换为一个并行数据流，这是对应着一个Flink Job，从而可以被JobManager和TaskManager执行。在实现上，Flink基于Actor实现了JobManager和TaskManager，所以JobManager与TaskManager之间的信息交换，都是通过事件的方式来进行处理。

如上图所示，Flink系统主要包含如下3个主要的进程：

### Client
当用户提交一个Flink程序时，会首先创建一个Client，该Client首先会对用户提交的Flink程序进行预处理，并提交到Flink集群中处理，所以Client需要从用户提交的Flink程序配置中获取JobManager的地址，并建立到JobManager的连接，将Flink Job提交给JobManager。Client会将用户提交的Flink程序组装一个JobGraph， 并且是以JobGraph的形式提交的。一个JobGraph是一个Flink Dataflow，它由多个JobVertex组成的DAG。其中，一个JobGraph包含了一个Flink程序的如下信息：JobID、Job名称、配置信息、一组JobVertex等。

### JobManager
JobManager是Flink系统的协调者，它负责接收Flink Job，调度组成Job的多个Task的执行。同时，JobManager还负责收集Job的状态信息，并管理Flink集群中从节点TaskManager。JobManager所负责的各项管理功能，它接收到并处理的事件主要包括：

- RegisterTaskManager  
在Flink集群启动的时候，TaskManager会向JobManager注册，如果注册成功，则JobManager会向TaskManager回复消息AcknowledgeRegistration。
- SubmitJob  
Flink程序内部通过Client向JobManager提交Flink Job，其中在消息SubmitJob中以JobGraph形式描述了Job的基本信息。
- CancelJob  
请求取消一个Flink Job的执行，CancelJob消息中包含了Job的ID，如果成功则返回消息CancellationSuccess，失败则返回消息CancellationFailure。
- UpdateTaskExecutionState  
TaskManager会向JobManager请求更新ExecutionGraph中的ExecutionVertex的状态信息，更新成功则返回true。
- RequestNextInputSplit  
运行在TaskManager上面的Task，请求获取下一个要处理的输入Split，成功则返回NextInputSplit。
- JobStatusChanged  
ExecutionGraph向JobManager发送该消息，用来表示Flink Job的状态发生的变化，例如：RUNNING、CANCELING、FINISHED等。

### TaskManager
TaskManager也是一个Actor，它是实际负责执行计算的Worker，在其上执行Flink Job的一组Task。每个TaskManager负责管理其所在节点上的资源信息，如内存、磁盘、网络，在启动的时候将资源的状态向JobManager汇报。TaskManager端可以分成两个阶段：

- 注册阶段  
TaskManager会向JobManager注册，发送RegisterTaskManager消息，等待JobManager返回AcknowledgeRegistration，然后TaskManager就可以进行初始化过程。
- 可操作阶段  
该阶段TaskManager可以接收并处理与Task有关的消息，如SubmitTask、CancelTask、FailTask。如果TaskManager无法连接到JobManager，这是TaskManager就失去了与JobManager的联系，会自动进入“注册阶段”，只有完成注册才能继续处理Task相关的消息。

## 内部原理
### 容错机制
### 调度机制
### 迭代机制
### Backpressure监控

## Refer
[Apache Flink：特性、概念、组件栈、架构及原理分析](http://shiyanjun.cn/archives/1508.html)
