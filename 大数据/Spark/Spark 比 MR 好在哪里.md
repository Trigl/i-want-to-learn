首先，并不是简单的 Spark 的计算都在内存中进行，有多个原因：

1. MR 仅支持 Map 和 Reduce 两种操作，写一个 join 都很复杂，而 Spark 提供了丰富的 API 和 各种高效的算子
2. Spark 的 DAG 模型比 MR 优秀，非 shuffle 的中间结果不需要落盘，而每个 MR 任务都要落入 HDFS 中
3. Spark task 通过线程池 fork 一个线程启动，MR 却是启动了新的进程，启动开销大
4. Spark 数据可以缓存在内存中，非常适合机器学习算法中的迭代计算，而 MR 迭代的适合每次都要读取 HDFS
5. MR 在 Map 和 Reduce 的时候都要进行排序，Spark 可以避免不必要的排序操作
