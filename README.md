# BusTubDB
Recording the learning of CMU15445 with code in private

---
### Lab 1 Buffer Pool

##### #1 Extendible Hash Table (22.11.30 - 22.12.1)
实现一个可扩展哈希表，支持 <k, v> 对的插入、删除、查询以及桶的拆分，扩容。\
没有实现桶的收缩，桶的合并。\
整个数据结构类似页表，通过一个中间层（目录）统一索引某个 k 的哈希值对应的 v 所在的桶。\
即 k ---> hash(k) ---> directory[hash(k)] ---> bucket_k ---> v\
维护一个全局深度，该深度用于在目录中进行索引。\
每个桶维护一个桶深度，若该深度 < 全局深度，则会有多个目录 entry 指向同一个桶。\
假设全局深度 = n，某个桶深度 = k，若 k < n，则会有 2^(n - k) 个目录 entry 指向同一个桶。\
大致逻辑如下：
```c++
given <k, v>
hash_code <- hash(k)
mash <- global_depth
target_bucket <- directory[hash_code & mask]
if target_bucket is full:
  if target_bucket.local_depth == global_depth:
    split(target_bucket)
  redistribute pointers which point to the target bucket
```

其中第一个 if 需要用循环，因为极有可能桶分裂后，该桶还是满的，桶中元素新的掩码最高位都是相等的，并且等于新插入的元素。\
这里的分裂需要将目录扩容为原来的两倍，因为二进制位每多一位就会使得目录数量乘2.\
新多出来的一半目录只需要复制原来的一半，因为桶分裂只针对该元素插入的桶，其他的桶深度没有变化。\
因此只需要重新分配目录中指向该桶的指针。

##### #2 LRU-K Replacement Policy (22.12.4)
优化版本的朴素 LRK 算法。\
其实普通的 LRU 可以看作 LRU-1。\
这个 LRU-K 便是访问多少次之后才淘汰。\
它的核心思想是：旧的数据，并且经常被访问的数据，不能被轻易移除。\
维护两个队列：历史队列 + 缓存队列\
一旦某个新的元素被访问，首先加入历史队列的队头。\
队头都是一些新鲜的元素。\
优先淘汰`历史队列`，这里都是使用不到 k 次的，即按照 LRU-1 的规则淘汰。\
并且这里的元素即便被再次访问，只要不到 k 次，都不会动他一根毫毛（位置），而仅仅是记录的访问次数++。\
一旦历史队列中的某个元素访问到达 k 次，就从中删除并移步`缓存队列`。\
缓存队列中的数据被再次访问后也需要更新次数，同时还要更新位置，即挪到队头，因为他们被多次访问，不能轻易删除。\
相关 paper 和博客：\
<a href="https://www.cs.cmu.edu/~natassa/courses/15-721/papers/p297-o_neil.pdf">The LRU-K Page Replacement Algorithm
For Database Disk Buffering</a> \
<a href="http://it.cha138.com/tech/show-252225.html">深入理解关系型数据库</a>

##### #3 Buffer Pool Manager Instance
