排序算子和常见的reduce算子算法有何区别？

- 常见的一些聚合、reduce算子，不需要排序。
- 将相同的 hashcode 分配到同一个 partition，哪怕是不同的 executor。
- 在做最后的合并的时候，只需要合并不同的 executor 里相同的 partition 就可以了。
- 对每个 partition 进行排序，考虑内存因数，解决相同的 Partition 多文件合并的问题，使用外排序进行相同的 key 合并。
