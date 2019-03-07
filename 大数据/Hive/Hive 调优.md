> Map 阶段并行度太大，产生大量小文件，初始化和创建 map 的开销很大。

减少map数可以通过合并小文件来实现，这点是对文件源。  

`set hive.merge.mapfiles = true;` -> 设置在Map-only的任务结束时合并小文件，map阶段Hive自动对小文件合并。  

`set hive.merge.mapredfiles = true;` -> 默认false， true时在MapReduce的任务结束时合并小文件  

`set hive.merge.per.task = 256*1000*1000;` -> 合并文件的大小

`set mapred.max.split.size = 256000000;` -> 每个Map最大分割大小（hadoop）

`set hive.input.format =org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;` -> 执行Map前进行小文件合并

在开启了org.apache.hadoop.hive.ql.io.CombineHiveInputFormat后,一个datanode节点上多个小文件会进行合并，合并文件数大小由 mapred.max.split.size 限制的大小决定。

> Map 阶段并行度太小，Job 执行时间过长。

Hive中设置map数：参数 `mapred.map.tasks`

来一个案例学习一下，案例中 hive 配置如下：

```
hive.merge.mapredfiles=true （默认是false，可以在hive-site.xml里配置）
hive.merge.mapfiles=true
hive.merge.per.task=256000000
mapred.map.tasks=2（默认值）
```

因为合并小文件默认为true，而dfs.block.size与hive.merge.per.task的搭配使得合并后的绝大部分文件都在256MB左右。

Case1：
现在我们假设有3个300MB大小的文件，整个JOB**会有**6个map．其牛3个map分别处理256M的数据，还有3个map分别处理44M的数据。

那么木桶效应就来了，整个Job的map阶段的执行时间,不是看最短的1个map的执行时间，而是看最长的1个map的执行时间。虽然有3个map分别只处理44MB的数据，可以很快跑完，但它们还是要等待另外3个处理256MB的map。显然，处理256MB的3个map拖了整个JOB的后腿。

Case2：
如果我们把mapred.map.tasks设置成6，再来看一下变化：
goalsize=min(900M/6,256M)=150M
整个JOB同样会分配6个Map来处理，每个map处理150MB，非常均匀，谁都不会拖后腿，最合理地分配了资源，执行时间大约为case1的59%（150/256）。

> Reduce数过大或过小。

reduce个数的决定
默认下，Hive分配reduce数基于以下参数：
参数1：hive.exec.reducers.bytes.per.reducer(默认是1G)
参数2：hive.exec.reducers.max(最大reduce数，默认为999)

计算reduce数的公式：
N=min（参数2，总输入数据量/参数1）,
即默认一个reduce处理1G数据量

什么情况下只有一个reduce？
很多时候你会发现任务中不管数据量多大，不管你有没有设置调reduce个数的参数，任务中一直都只有一个reduce任务（会产生数据倾斜）。

原因：
1、数据量小于hive.exec.reducers.bytes.per.reducer参数值（有时，通常情况下设置reduce个数会起作用）
2、没有group by的汇总
3、用了order by

解决：
设置reduce数
参数mapred.reduce.tasks 默认是1
set mapred.reduce.tasks=10
set的作用域是session级

设置reduce数有时对我们优化非常有帮助。
当某个job的结果被后边job**多次引用**时，设置该参数，以便增大访问的map数。Reuduce数决定中间结果或落地文件数，文件大小和Block大小无关。

> 数据倾斜的参数调节

hive.map.aggr=true
Map 端部分聚合，相当于Combiner

hive.groupby.skewindata=true
有数据倾斜的时候进行负载均衡，当选项设定为true，生成的查询计划会有两个 MR Job。第一个 MR Job 中，Map 的输出结果集合会随机分布到 Reduce 中，每个 Reduce 做部分聚合操作，并输出结果，这样处理的结果是相同的 Group By Key 有可能被分发到不同的 Reduce 中，从而达到负载均衡的目的；第二个 MR Job 再根据预处理的数据结果按照 Group By Key 分布到 Reduce 中（这个过程可以保证相同的 Group By Key 被分布到同一个 Reduce 中），最后完成最终的聚合操作。

> 空值产生的数据倾斜

场景：如日志中，常会有信息丢失的问题，比如日志中的user_id，如果取其中的user_id和用户表中的user_id 关联，会碰到数据倾斜的问题。

解决方法 ：赋与空值新的key值，把空值的key变成一个字符串加上随机数，就能把倾斜的数据分到不同的reduce上，解决数据倾斜问题。

```
select * from log a left outer join users b on case when a.user_id is null then concat('hive',rand() )
else a.user_id end = b.user_id;
```
