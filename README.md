# BusTubDB
Recording the learning of CMU15445 with code in private.\
**Do not post your project on a public Github repository.**\
Andy 老师和助教们幸苦做课程实验代码，还给我们 NON-CMU 的学生开放在线评测，不希望我们把代码放出来，不能辜负了他们的一片好心。\
开一个 private 自己提交，开一个 public 记录学习过程。\
况且 CMU 的学生那么强，我个人代码还有许多需要学习、优化的地方。（CMU 的学生应该看不懂中文hhh）\
加油！

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

##### #3 Buffer Pool Manager Instance (22.12.5 - 22.12.6)
这一块就根据缓冲池的工作逻辑模拟就好了，需要实现前面写的两个组件：HashTable 和 LRUKReplacer。\
有一个坑点就是，UnpinPgImp 会传入一个参数 is_dirty，不可以直接把这个 bool 赋值给页面的脏位，\
因为一个页面可能并发地被多个线程 Unpin，因此对于每个线程来说该页是否 dirty 是由他们自己决定的。\
而一个页相较于磁盘是否 dirty，也就是 page->is_dirty 位，是客观的。\
因此，只要有人 Unpin 的时候传入 is_dirty = true，那么在刷盘之前，这个页始终都是脏的。\
因此要用逻辑或来给脏位赋值。

自己写的子模块不用加锁，不然死锁了。
目前锁的粒度还有有点大的，一个函数一把大锁。

NewPgImpl 和 FetchPgImpl 都需要使用到一个核心功能，那就是 `寻找页框`。\
这里的代码可以复用，因此写成了一个独立的函数。\
逻辑也很清晰：
```c++
if free_list.empty == false:  // 首先考虑从空闲链表拿一个空的来用
  frame_id = free_list.front
  free_list.pop_front
  return frame_id
 else if replacer.size != 0:  // 再考虑淘汰旧页面来换
  evict(&frame_id)
  if frame_id is valid:
    return frame_id
  else:
    return -1
```

> 马上就要期末考试了，还没复习\
唉...

### lab2 B+ Tree Index

#### checkpoint 1

**Insert，Split** (22.12.20、21)\
数据的插入，主要流程是从获取根结点开始，一路查找到叶子结点，找到该数据应该存放的叶子结点。\
插入该数据，并判断该页面是否溢出，即数量有没有超过 $n - 1$ 个 kv 对。\
如果超出，则需要分裂该页面，将一个页面保留 $\lceil (n - 1) / 2 \rceil$ 个 kv 对，并且将剩下的 $n - \lceil (n - 1) / 2 \rceil$ 个 kv 对搬到一个新的页面中。
一旦叶子结点分裂，需要将指向新的页面的指针插入到父结点当中，同时检查父结点的指针数量不得超过 $ n $ 个，如果超出，则也需要分裂，并且递归执行该过程向上传递分裂。

**Delete，Coalesce，Redistribute** （22.12.25 - 22.12.27）\
删除首先和插入一样，通过查找函数找到要删除的元素所在的叶子结点。\
从叶子结点中删除该元素，然后判断该页面的元素是否少于 $\lceil (n - 1) / 2 \rceil$。\
如果少于，需要做一些额外的操作来达到平衡:
1. 合并：如果该结点元素加上兄弟结点元素不超过 n - 1 个（内部节点不超过 n 个），则可以和兄弟结点合并，删除多余结点，从父结点删除指针并递归检查传递合并。
2. 重新分配：如果两者元素数量之和超过了页面的最大元素数量，那么该页面可以从兄弟结点偷一个元素过来以满足最低要求，同时兄弟结点被偷了一个后剩余元素数也不会少于最低要求。

#### checkpoint 2

**Index Iterator** (22.12.28)\
这部分需要在 B_Plus_Tree 类中完善 Begin 和 End 函数，这俩接口需要返回一个迭代器类，即 IndexIterator 类。\
Begin 接口包含有参数版本和无参数版本，其中有参数版本会从指定的 Key 开始 Iterate，无参数版本默认从第一个元素开始。\
在 IndexIterator 类中完善相应的运算符重载就好了，需要自己添加成员变量。\
首先考虑，迭代器肯定要指向当前元素，那么肯定需要在某个叶子页面中索引该元素。\
因此一个 LeafPage 类型的指针是必需的，此外还需要一个 int 来指示该元素在本页面中的位置。\
由于 B+ 树中元素都在叶子结点，因此遍历所有的元素就是在遍历所有的叶子结点，因此涉及不同页面的切换，因此需要一个 BUfferPool。\
有了这三个成员变量就差不多了。

**Concurrent** (23.1.3, 23.1.11)\
书上对于索引结构的并发控制介绍了 `蟹行协议 (Crabbing Protocal)` 和 `B-link 树封锁协议 (B-link-tree locking protocol)`。\
先加把大锁混过去吧。

### lab3 Query Execution


