checkpoint在spark中主要有两块应用：一块是在spark core中对RDD做checkpoint，可以切断做checkpoint RDD的依赖关系，将RDD数据保存到可靠存储（如HDFS）以便数据恢复；另外一块是应用在spark streaming中，使用checkpoint用来保存DStreamGraph以及相关配置信息，以便在Driver崩溃重启的时候能够接着之前进度继续进行处理（如之前waiting batch的job会在重启后继续处理）。

本文主要将详细分析checkpoint在以上两种场景的读写过程。

# spark core中checkpoint分析
## checkpoint的使用方法
使用checkpoint对RDD做快照大体如下：

```scala
sc.setCheckpointDir(checkpointDir.toString)
val rdd = sc.makeRDD(1 to 20, numSlices = 1)
rdd.checkpoint()
```

首先，设置checkpoint的目录（一般是hdfs目录），这个目录用来将RDD相关的数据（包括每个partition实际数据，以及partitioner（如果有的话））。然后在RDD上调用checkpoint的方法即可。

## checkpoint写流程
可以看到checkpoint使用非常简单，设置checkpoint目录，然后调用RDD的checkpoint方法。针对checkpoint的写入流程，主要有以下四个问题：

- RDD中的数据是什么时候写入的？是在rdd调用checkpoint方法时候吗？
- 在做checkpoint的时候，具体写入了哪些数据到HDFS了？
- 在对RDD做完checkpoint以后，对做RDD的本身又做了哪些收尾工作？
- 实际过程中，使用RDD做checkpoint的时候需要注意什么问题？

弄清楚了以上四个问题，我想对checkpoint的写过程也就基本清楚了。接下来通过对源码的分析来一一回答上面提出的问题。

需要使用checkpoint都需要通过sparkcontext的setCheckpointDir方法设置一个目录以存checkpoint的各种信息数据，下面我们来看看该方法：

```scala
def setCheckpointDir(directory: String) {
    if (!isLocal && Utils.nonLocalPaths(directory).isEmpty) {
      logWarning("Spark is not running in local mode, therefore the checkpoint directory " +
        s"must not be on the local filesystem. Directory '$directory' " +
        "appears to be on the local filesystem.")
    }
    checkpointDir = Option(directory).map { dir =>
      val path = new Path(dir, UUID.randomUUID().toString)
      val fs = path.getFileSystem(hadoopConfiguration)
      fs.mkdirs(path)
      fs.getFileStatus(path).getPath.toString
    }
  }
```

在非local模式下，directory必须是HDFS的目录；在该目录下创建一个以UUID生成的一个唯一的目录名的目录。

设置了 checkpoint 目录以后，我们在代码中会通过 `rdd.checkpoint()` 来 checkpoint 这个 rdd，我们来看一下 checkpoint() 方法的实现：

```scala
def checkpoint(): Unit = RDDCheckpointData.synchronized {
    if (context.checkpointDir.isEmpty) {
      throw new SparkException("Checkpoint directory has not been set in the SparkContext")
    } else if (checkpointData.isEmpty) {
      checkpointData = Some(new ReliableRDDCheckpointData(this))
    }
  }
```

先判断是否设置了checkpointDir，再判断checkpointData.isEmpty是否成立，checkpointData的定义是这样的：

```scala
private[spark] var checkpointData: Option[RDDCheckpointData[T]] = None
```

RDDCheckpointData和RDD一一对应，保存着和checkpoint相关的信息。这里通过new ReliableRDDCheckpointData(this)实例化了checkpointData ，ReliableRDDCheckpointData是其子类，这里相当于是checkpoint的一个标记，并没有真正执行checkpoint。

那么到底是什么时候是真正执行 checkpoint 呢？事实上 checkpoint 也是懒实现的，所以它是在一个 RDD 执行了算子操作以后才会执行，而 RDD 执行算子操作会触发 sparkcontext对runJob的调用：

```scala
def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      resultHandler: (Int, U) => Unit): Unit = {
    if (stopped.get()) {
      throw new IllegalStateException("SparkContext has been shutdown")
    }
    val callSite = getCallSite
    val cleanedFunc = clean(func)
    logInfo("Starting job: " + callSite.shortForm)
    if (conf.getBoolean("spark.logLineage", false)) {
      logInfo("RDD's recursive dependencies:\n" + rdd.toDebugString)
    }
    dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
    progressBar.foreach(_.finishAll())
    rdd.doCheckpoint()
  }
```

我们可以看到在执行完job后会执行 rdd.doCheckpoint()，这里就是对前面标记了的RDD的checkpoint，我们继续看这个方法：

```scala
private[spark] def doCheckpoint(): Unit = {
    RDDOperationScope.withScope(sc, "checkpoint", allowNesting = false, ignoreParent = true) {
      if (!doCheckpointCalled) {
        doCheckpointCalled = true
        if (checkpointData.isDefined) {
          if (checkpointAllMarkedAncestors) {
              dependencies.foreach(_.rdd.doCheckpoint())
          }
          checkpointData.get.checkpoint()
        } else {
      dependencies.foreach(_.rdd.doCheckpoint())
        }
      }
    }
  }
```

先判断是否已经被处理过checkpoint，没有才执行，并将doCheckpointCalled 设为true，因为前面已经初始化过了checkpointData，所以checkpointData.isDefined也满足，若想要把checkpointData定义过的RDD的parents也进行checkpoint的话，那么我们需要先对parents checkpoint。因为，如果RDD把自己checkpoint了，那么它就将lineage中它的parents给切除了。继续跟进checkpointData.get.checkpoint()

```scala
final def checkpoint(): Unit = {
    // Guard against multiple threads checkpointing the same RDD by
    // atomically flipping the state of this RDDCheckpointData
    RDDCheckpointData.synchronized {
      if (cpState == Initialized) {
        cpState = CheckpointingInProgress
      } else {
        return
      }
    }

    val newRDD = doCheckpoint()

    // Update our state and truncate the RDD lineage
    RDDCheckpointData.synchronized {
      cpRDD = Some(newRDD)
      cpState = Checkpointed
      rdd.markCheckpointed()
    }
  }
```

这里就是进行 checkpoint 宏观上的流程：

1. 判断 checkpoint 状态是否是 Initialized，是的话就把其状态改成进行中 CheckpointingInProgress
2. 通过 doCheckpoint() 正式执行 checkpoint
3. 最后做一些收尾工作，将 checkpoint 状态改为 Checkpointed，同时对这个 rdd 标记已经 checkpointed，markCheckpointed() 的内部实现其实就是清除RDD的所有依赖。

接下来我再看一下上面 doCheckpoint() 到底是怎么做的 checkpoint：

```scala
protected override def doCheckpoint(): CheckpointRDD[T] = {
    val newRDD = ReliableCheckpointRDD.writeRDDToCheckpointDirectory(rdd, cpDir)
    if (rdd.conf.getBoolean("spark.cleaner.referenceTracking.cleanCheckpoints", false)) {
      rdd.context.cleaner.foreach { cleaner =>
        cleaner.registerRDDCheckpointDataForCleanup(newRDD, rdd.id)
      }
    }
    logInfo(s"Done checkpointing RDD ${rdd.id} to $cpDir, new parent is RDD ${newRDD.id}")
    newRDD
  }
```

ReliableCheckpointRDD.writeRDDToCheckpointDirectory(rdd, cpDir)，将一个RDD写入到多个checkpoint文件，并返回一个ReliableCheckpointRDD来代表这个RDD，继续看 writeRDDToCheckpointDirectory 的实现：

```scala
def writeRDDToCheckpointDirectory[T: ClassTag](
      originalRDD: RDD[T],
      checkpointDir: String,
      blockSize: Int = -1): ReliableCheckpointRDD[T] = {
    val sc = originalRDD.sparkContext
    // Create the output path for the checkpoint
    val checkpointDirPath = new Path(checkpointDir)
    val fs = checkpointDirPath.getFileSystem(sc.hadoopConfiguration)
    if (!fs.mkdirs(checkpointDirPath)) {
      throw new SparkException(s"Failed to create checkpoint path $checkpointDirPath")
    }
    // Save to file, and reload it as an RDD
    val broadcastedConf = sc.broadcast(
      new SerializableConfiguration(sc.hadoopConfiguration))
    // TODO: This is expensive because it computes the RDD again unnecessarily (SPARK-8582)
    sc.runJob(originalRDD,
      writePartitionToCheckpointFile[T](checkpointDirPath.toString, broadcastedConf) _)
    if (originalRDD.partitioner.nonEmpty) {
      writePartitionerToCheckpointDir(sc, originalRDD.partitioner.get, checkpointDirPath)
    }
    val newRDD = new ReliableCheckpointRDD[T](
      sc, checkpointDirPath.toString, originalRDD.partitioner)
    if (newRDD.partitions.length != originalRDD.partitions.length) {
      throw new SparkException(
        s"Checkpoint RDD $newRDD(${newRDD.partitions.length}) has different " +
          s"number of partitions from original RDD $originalRDD(${originalRDD.partitions.length})")
    }
    newRDD
  }
```

可以看到首先通过 writePartitionToCheckpointFile 把 partition 写入到了 checkpoint 目录文件中，然后判断是否存在 partitioner，如果存在的话还会把 partitioner 写入 checkpoint 目录。  
但是注意这里在 checkpoint 的时候是调用了 `sc.runJob` 实现的，我们指定这个操作只有在调用 action 算子才会有，所以事实上这个时候又对 rdd 重新计算了一次，加上 rdd 本身算子操作，其实 rdd 被重复计算了两次，相当耗费资源，所以在做 checkpoint 的时候我们最好将 RDD 缓存下来避免重复计算。

上面就是 spark core 对 rdd checkpoint 的整个流程，现在让我们尝试着回答上面提出的 4 个问题。

Q1：RDD中的数据是什么时候写入的？是在rdd调用checkpoint方法时候吗？  
A1：首先看一下RDD中checkpoint方法，可以看到在该方法中是只是新建了一个ReliableRDDCheckpintData的对象，并没有做实际的写入工作。实际触发写入的时机是在runJob生成改RDD后，调用RDD的doCheckpoint方法来做的。

Q2：在做checkpoint的时候，具体写入了哪些数据到HDFS了？  
A2：在经历调用RDD.doCheckpoint → RDDCheckpintData.checkpoint → ReliableRDDCheckpintData.doCheckpoint → ReliableRDDCheckpintData.writeRDDToCheckpointDirectory后，在writeRDDToCheckpointDirectory方法中可以看到：将作为一个单独的任务（RunJob）将RDD中每个parition的数据依次写入到checkpoint目录（writePartitionToCheckpointFile），此外如果该RDD中的partitioner如果不为空，则也会将该对象序列化后存储到checkpoint目录。所以，在做checkpoint的时候，写入的hdfs中的数据主要包括：RDD中每个parition的实际数据，以及可能的partitioner对象

Q3：在对RDD做完checkpoint以后，对做RDD的本身又做了哪些收尾工作？  
A3：在写完checkpoint数据到hdfs以后，将会调用rdd的markCheckpoined方法，主要斩断该rdd的对上游的依赖，以及将paritions置空等操作。

Q4：实际过程中，使用RDD做checkpoint的时候需要注意什么问题？  
A4：通过上面的分析可知，在RDD计算完毕后，会再次通过RunJob将每个partition数据保存到HDFS。这样RDD将会计算两次，所以为了避免此类情况，最好将RDD进行cache。即rdd的推荐使用方法如下：

```scala
sc.setCheckpointDir(checkpointDir.toString)
val rdd = sc.makeRDD(1 to 20, numSlices = 1)
rdd.cache()
rdd.checkpoint()
```

## checkpoint 读流程
读 checkpoint 应该发生在应用出现问题的时候，当我们去读取一个 RDD 的时候，一定会读取到它的 partition，而读取 partition 内部是通过 rdd.iterator() 实现的：

```scala
final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
    if (storageLevel != StorageLevel.NONE) {
      getOrCompute(split, context)
    } else {
      computeOrReadCheckpoint(split, context)
    }
  }
```

首先会读 cache，没有 cache 就调用 computeOrReadCheckpoint 方法：

```scala
private[spark] def computeOrReadCheckpoint(split: Partition, context: TaskContext): Iterator[T] =
  {
    if (isCheckpointedAndMaterialized) {
      firstParent[T].iterator(split, context)
    } else {
      compute(split, context)
    }
  }
```

isCheckpointedAndMaterialized就是在checkpoint成功时的一个状态标记：cpState = Checkpointed。当该RDD被成功checkpoint了，直接使用parent rdd 的 iterator() 也就是 CheckpointRDD.iterator()，否则直接调用该RDD的compute方法。

# spark streaming中checkpoint分析

[spark-streaming的checkpoint机制源码分析](https://www.cnblogs.com/dongxiao-yang/p/7994357.html)
[Spark Streaming如何使用checkpoint容错](https://qindongliang.iteye.com/blog/2350846)
[spark checkpoint详解](https://www.cnblogs.com/superhedantou/p/9004820.html)
