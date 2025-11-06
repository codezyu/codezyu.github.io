
# 查找
## SST文件查询优化
> https://rocksdb.org/blog/2014/04/21/indexing-sst-files-for-better-lookup-performance.html

**瓶颈**：
- **L0层文件**：按刷新时间排序，键范围大多重叠，需要查找每个L0文件
- **L1层及以下**：虽然文件按键排序且键范围互斥，但仍需要通过二分查找定位候选文件，复杂度为O(log(N))
**瓶颈分析**：
log(N)的复杂度仍然很大。例如，在扇出比为10的情况下，第3层可能有1000个文件，需要10次比较才能定位到候选文件
### 分级级联
LSM Tree构建后文件位置固定，因此可以通过在compaction的时候，减少二分搜索的范围
只有 compaction 才会改变 SST
1. **预构建指针**：在压缩时，为L1层文件预构建指向L2层文件范围的指针
2. **范围推导**：通过上层查找结果推导键的可能范围，排除下层不可能包含目标键的文件
### 实现
为左右边界都定义了一个文件索引
```cpp
struct IndexUnit {
    IndexUnit() : smallest_lb(0), largest_lb(0), smallest_rb(-1), largest_rb(-1) {}
    
    // 左边界：下层文件中 > smallest/largest 的第一个文件索引
    int32_t smallest_lb;  // target > smallest 时使用
    int32_t largest_lb;   // target > largest 时使用
    
    // 右边界：下层文件中 < smallest/largest 的最后一个文件索引  
    int32_t smallest_rb;  // target < smallest 时使用
    int32_t largest_rb;   // target < largest 时使用
};
```
当用户查找某个 user_key 时，FileIndexer 根据 key 与当前文件边界的关系，提供五种不同的查找策略：
1. **user_key < smallest_key**: 使用前一个文件的 largest_lb 和当前文件的 smallest_rb
2. **user_key == smallest_key**: 使用当前文件的 smallest_lb 和 smallest_rb
3. **user_key 在文件范围内**: 使用当前文件的 smallest_lb 和 largest_rb
4. **user_key == largest_key**: 使用当前文件的 largest_lb 和 largest_rb
5. **user_key > largest_key**: 使用当前文件的 largest_rb 到最右边文件
**结果**
> 提升QPS 5%

## 前缀key查询优化
快速查找以指定前缀开头的所有 key
当定义了一个 `prefix_extractor`（例如，指定所有键的前 N 个字节为前缀）时，RocksDB 会启用一系列专门针对前缀的优化措施。
Prefix seek是RocksDB的一种模式，主要影响Iterator的行为。 在这种模式下，RocksDB的Iterator并不保证所有key是有序的，而只保证具有相同前缀的key是有序的。这样可以保证具有相同特征的key（例如具有相同前缀的key）尽量地被聚合在一起。
### 布隆过滤器：Prefix Bloom Filter
不定义 `prefix_extractor`，布隆过滤器会基于**整个 Key** 来创建。
但当 `Seek("userA-")`（一个前缀）时，则无法判断是否存在，当定义了 `prefix_extractor`，RocksDB 会转而基于**Key 的前缀**来构建布隆过滤器。
- 如果布隆过滤器显示这个 "prefix" **绝对不存在**于该 SSTable 文件中，RocksDB 会**完全跳过**对这个文件的任何 I/O 操作（既不读索引块，也不读数据块）。
### memtable优化
当你用了 `prefix_extractor`，RocksDB 可以在 MemTable 内部额外维护一个基于**前缀的哈希索引**。
- 快速定位到 MemTable 中第一个匹配该前缀的 Key
### 迭代器优化
ReadOptions.iterate_upper_bound， 可以计算出刚好大于这个前缀的下一个前缀，并将其设置为 `iterate_upper_bound`
如果在当前文件超出上界，则无需查询
# IO
## 异步IO

# 事务
它默认提供的`Put`、`Get`、`Delete`操作是原子的（Atomic），但它本身不直接提供跨多个操作的事务（Transaction）。
## 原子性
单个`Put()`或`Delete()`操作，在内部也会被转换成一个只包含一个操作的`WriteBatch`。
`WriteBatch` (写批处理)
- 当你提交这个`WriteBatch`时，RocksDB保证会 **原子地** 将它写入WAL（Write Ahead Log）并应用到MemTable。
- 批量提交到WAL
- 可以判断这个Record是不是完整的

### 实现
RocksDB中的每一条记录(KeyValue)都有一个LogSequenceNumber(后面统称lsn)
在WriteAheadLog(WAL)中`以WriteBatch为单位递增`(count(batch.records)为单位)。
- 所以进行`snapshot读`时，使用的就是此lsn。


## 隔离性
RocksDB可以创建一个`Snapshot`对象，它代表了数据库在 _某个特定时间点_ 的一致性视图。使用这个`Snapshot`进行的所有读操作（`Get`、`Iterator`），只能看到该时间点之前的数据，_看不到_ 之后提交的任何新数据（包括其他事务的未提交数据）。**这就是隔离性的来源。**
```cpp
class SnapshotImpl {
 public:
  // 当前的lsn
  SequenceNumber number_;       
 private:
  SnapshotImpl* prev_;
  SnapshotImpl* next_;
  SnapshotList* list_; 
  // unix时间戳
  int64_t unix_time_;           
  // 是否属于Transaction(用于写冲突)
  bool is_write_conflict_boundary_;
};
```

实现快照隔离时，通过 Snapshot 设置其与一个 lsn 绑定，则该 snapshot 能够访问到小于等于当前 lsn 的 k-v 数据，而大于该 lsn 的 key-value 是不可见的。
## 事务
 当使用 TransactionDB 或者 OptimisticTransactionDB 的时候，可以使用 RocksDB 的 BEGIN/COMMIT/ROLLBACK 等事务 API。RocksDB 支持活锁或者死等两种事务。