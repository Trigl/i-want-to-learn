[Kafka: a Distributed Messaging System for Log Processing](http://notes.stephenholiday.com/Kafka.pdf)

传统的消息系统提供了消息送达的强保证。
IBM Websphere MQ 在存储数据的时候提供了原子性的事务操作，JMS 在消息被消费以后会进行确认。

疑问1:生产者发布的消息达到了一定的量或者超过了一定的时间才会刷写到磁盘里面，而消息只有被刷写以后才会向消费者开放，那么这样的话消息从生产者到消费者会不会有一定的延迟？
