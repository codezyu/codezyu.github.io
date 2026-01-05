你不需要缓存所有数据，只需要缓存**最热的**那一部分。具体数量由服务器的数量 $n$ 决定，数量级为 $O(n \log n)$
剩下的就是负载均衡的问题了
- switch kv: memory to SSD
- net cache: switch to memory

L3 cache: to memory?


对cache 做partition 不适合skew的workload
- local count
- global count

缓存和数据量无关，而是跟node数量相关，来保证负载均衡
**Extendible Hashing:** 只要桶满了就分裂，数据会被均匀地摊到新的桶里。它天然保证了所有存储节点的**数据量**是相对均匀的
- 对“长尾碰撞”的免疫力更强：
    
    Extendible Hashing 因为有 Split 机制，它不像一致性哈希那样听天由命。如果“热点”伴随着“数据量大”（通常热点区域数据也密集），它会自动分裂扩容，从而天然地平衡了负载。在这种情况下，你需要的缓存小于 $O(n \log n)$。
    
- 对“小体积高热度”依然无解：
    
    如果你的热点是“数据量很小（不触发分裂），但访问量极大”，Extendible Hashing 退化为普通哈希。此时，你依然需要缓存那最热的 $O(n \log n)$ 个对象。


