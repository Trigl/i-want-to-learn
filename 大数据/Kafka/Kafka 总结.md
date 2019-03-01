[Kafka史上最详细原理总结](http://bigdatastudy.net/show.aspx?id=495&cid=9)

[Kafka 数据可靠性深度解读](https://www.infoq.cn/article/depth-interpretation-of-kafka-data-reliability)

leader 维护自己的 HW 可以理解，那么 follower 如何维护自己的 HW 呢？  
ISR 中副本的 HW 完全一致。
[Kafka水位(high watermark)与leader epoch的讨论](https://www.cnblogs.com/huxi2b/p/7453543.html)

ISR 如何保证数据不丢失？如果 leader 刚接收到一条数据，还没来得及被 follower 复制，这个时候 leader 宕机，是不是就意味着数据丢失？
确认机制 -1/0/1

ISR 如何保证吞吐率，听起来每一条消息都需要复制到所以 follower 中，这样的复制方式很影响吞吐率吧？

实际上，leader 选举的算法非常多，比如 Zookeeper 的Zab、Raft以及Viewstamped Replication。而 Kafka 所使用的 leader 选举算法更像是微软的PacificA算法。

leader 选举时，如果副本失败了以后，为什么不把这个副本从 ISR 中移出去？
因为和 leader 的偏移时间不大，认为它们是同步的

producer 的 retries 在哪设置？
