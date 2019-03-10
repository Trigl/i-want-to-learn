MongoDB做高性能数据库，Redis做缓存，HBase做大数据分析。Redis定位在"快"，HBase定位于"大",mongodb定位在"灵活"。

MongoDB是高性能、无模式的文档型数据库，支持二级索引，非常适合文档化格式的存储及查询。MongoDB的官方定位是通用数据库，确实和MySQL有些像，现在也很流行，但它还是有事务、join等短板，在事务、复杂查询应用下无法取代关系型数据库。

Redis是内存型Key/Value系统，读写性能非常好，支持操作原子性，很适合用来做高速缓存。Redis的魅力还在于它不像HBase只支持简单的字符串，他还支持集合set，有序集合zset和哈希hash。但是处理的数据量要小于HBase与MongoDB

HBase存储容量大，一个表可以容纳上亿行、上百万列，可应对超大数据量要求扩展简单的需求。Hadoop的无缝集成，让HBase的数据可靠性和海量数据分析性能（MapReduce）值得期待。
