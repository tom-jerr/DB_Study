# Lab 1

## 任务
实现线程安全的BufferPool。多个线程将同时访问缓冲池的内部数据结构，并且必须确保关键部分受到latch的保护

- LRU-K replacer
- Disk scheduler
- Buffer pool manager

## LRU-K replacer
### 原理
LRU-K中的K代表最近使用的次数，因此LRU可以认为是LRU-1。LRU-K的主要目的是为了解决LRU算法“缓存污染”的问题，其核心思想是将“最近使用过1次”的判断标准扩展为“最近使用过K次”。

相比LRU，LRU-K需要多维护一个队列，用于记录所有缓存数据被访问的历史。只有当数据的访问次数达到K次的时候，才将数据放入缓存。当需要淘汰数据时，LRU-K会淘汰第K次访问时间距当前时间最大的数据。

>　　(1). 数据第一次被访问，加入到访问历史列表；  
(2). 如果数据在访问历史列表里后没有达到K次访问，则按照一定规则（FIFO，LRU）淘汰；  
(3). 当访问历史队列中的数据访问次数达到K次后，将数据索引从历史队列删除，将数据移到缓存队列中，并缓存此数据，缓存队列重新按照时间排序；  
(4). 缓存数据队列中被再次访问后，重新排序；  
(5). 需要淘汰数据时，淘汰缓存队列中排在末尾的数据，即：淘汰“倒数第K次访问离现在最久”的数据。

![](../img/lru-k.png)
### 实现
这里每个frame对应的访问次数用LRUNode来存储，这里主要是要注意计算`backward k distance`的方法
```c++
class LRUKNode {
 public:
  LRUKNode() = default;
  explicit LRUKNode(size_t k, frame_id_t fid);
  explicit LRUKNode(size_t k, frame_id_t fid, size_t current_timestamp);

  auto Fid() -> frame_id_t { return fid_; }
  /**
   * @brief 增加该节点的访问次数，如果超过K次，驱逐最远的访问时间
   *
   * @param current_timestamp
   * @return true
   * @return false
   */
  auto AccessNode(size_t current_timestamp) -> bool;
  /**
   * @brief 计算当前时间戳与最近k次访问的时间戳的差值
   *
   * @param current_timestamp
   * @return size_t
   */
  auto BackwardKDistance(size_t current_timestamp) -> size_t;
  /**
   * @brief Get the Last Access Time object
   *
   * @return size_t
   */
  auto GetLastAccessTime() -> size_t { return history_.back(); }

  auto IsEvictable() -> bool { return is_evictable_; }

  void SetEvictable(bool set_evictable) { is_evictable_ = set_evictable; }

 private:
  /** History of last seen K timestamps of this page. Least recent timestamp stored in front. */
  // Remove maybe_unused if you start using them. Feel free to change the member variables as you want.

  std::list<size_t> history_;         // history timestamp
  size_t k_{0};                       // k times
  frame_id_t fid_{INVALID_FRAME_ID};  // frame id
  bool is_evictable_{false};          // whether or not can be evicted
};
```
- LRUKReplacer 的 Evict() 函数要注意分为两种情况
  1. 存在访问未达到 K 次的 frame ，这时需要驱逐当前最早被访问的 frame
  2. 如果所有的frame访问都达到了 K 次，这时按照 backward k distance 排序，驱逐最小的 frame

## Disk Scheduler
- DiskScheduler 内部实际上是一个请求队列，不断接收请求；后台线程作为成员变量不断在另一个线程上处理请求(通过`std::promise`和`std::future`实现)

- 当前实现是单线程，可以考虑使用多个线程并行处理 disk request

## Buffer Pool Manager
BufferPoolManager负责使用 从磁盘获取数据库页面DiskScheduler并将它们存储在内存中。BufferPoolManager还可以在明确指示或需要逐出页面以腾出空间放置新页面时安排将脏页面写入磁盘。

### :skull: 锁策略
- 遵循pin->lock->unlock->unpin的顺序
  1. 如果是lock->pin->unlock->unpin
    > A进程lock 并且pin page1，B进程在等待latch，当A进程释放锁后，B进程获取锁但pin page1前，该page可能被其他线程驱逐，我们会读到错误的数据
  2. 如果是pin->lock->unpin->unlock
    > 如果A,B两个进程均希望访问page1，当A lock后，B pin 前，A可能将page1刷写到磁盘，导致A,B结果不一致
- bpm_latch 保护 pin_count_-- 和 replacer_ 相关方法
  > 如果不保护可能存在两个 ReadPageGuard 同时进行 pin_count--，可能驱逐 page 的时机有误

- 获取page读写锁前释放bpm_latch

### :loudspeaker: 获取PageGuard
这里存在三种情况
1. 当前frame已经在page_table中，更新变量，这里不需要额外的IO操作
2. 空闲内存充足，可以从空闲内存中获取frame，然后从对应的page中读取page内容到frame
3. 没有空闲内存，通过lru-k replacer驱逐出一个frame，如果frame已经被写入内容，需要将该frame回写到磁盘，在进行page对应内容的读取

### :skull: PageGuard 的 ctor & dtor
- PageGuard是一个RAII的类，我们需要在ctor中获取frame的读写锁并在dtor中进行释放
- dtor中对pin_count和replacer的操作需要加bpm_latch
### frame 重置
- 我们仅仅应该只在DeletePage中对frame进行Reset操作
  > flushPage可能pin_count不为0，此时进行重置，PageGuard中的Drop()会让pin_count为负数

