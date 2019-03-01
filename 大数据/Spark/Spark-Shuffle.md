![](/resource/spark-shuffle.png)

[Spark性能优化指南——高级篇](https://tech.meituan.com/2016/05/12/spark-tuning-pro.html)

[彻底搞懂spark的shuffle过程（shuffle write）](https://zhuanlan.zhihu.com/p/55954840)

疑问：Spark 是全部在内存中运行吗？  

答案 1:  
在目前的 Spark 实现中，shuffle block 一定是落地到磁盘的，无法像普通 RDD 那样 cache 到本地内存或 Tachyon 中。想将 shuffle block cache 到内存中，应该主要是为了提速，但事实上并没有什么必要。  
首先，内存 cache 发挥作用的前提是被 cache 的数据会被反复使用。使用越频繁，相对来说收益越高。而 shuffle block 只有在下游 task 失败，进行容错恢复时才有重用机会，频次很低。值得注意的是，在不 cache 的情况下，针对同一个含 shuffle 的 RDD 执行多个 action，并不会重用  shuffle 结果。Shuffle block 是按 job ID + task ID + attempt ID 标识的，每个 action 都对应于一个独立的 job，因此无法重用。这里或许是 Spark 的一个可改进点。  
其次，从数据量上说，如果执行的是需要 shuffle 大数据量的 Spark job，内存容量不够，无论如何都需要落盘；如果执行的是小数据量的 Spark job，虽然 shuffle block 会落盘，但仍然还在 OS cache 内，而 shuffle block 一般都是在生成之后短时间内即被下游 task 取走，所以大部分情况下仍然还是内存访问。  
最后，将 shuffle block cache 在内存中的确有一个潜在好处，就是有机会直接在内存中保存原始的 Java 对象而避免序列化开销。但这个问题在新近的 Spark 版本中也有了比较好的解决方案。Tungsten project 中引入的 UnsafeRow 格式统一了内存和磁盘表示，已经最小化了序列化成本。

答案 2:  
[Spark会把数据都载入到内存么？](https://www.jianshu.com/p/b70fe63a77a8)
