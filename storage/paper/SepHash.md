PVLDB 2024 | **SepHash: A Write-Optimized Hash Index On Disaggregated Memory via Separate Segment Structure**

本文针对存算分离（Disaggregated Memory）架构，指出了内存节点计算能力近乎为零（Near-zero computation power）以及RDMA网络传输在高并发下的带宽与延迟瓶颈 111111111。

# 问题

- **原有方案的不适应性**
    
    - **Entry-based Resize 带来的带宽浪费**：现有的哈希索引（如 RACE）在进行扩容（Resize）时，采用基于条目（Entry）的迁移策略。由于内存节点无法进行计算，数据迁移必须拉回客户端处理。这种细粒度的数据搬运导致了巨大的网络开销，成为性能瓶颈 2222。同时，对于变长 KV 数据，迁移每个条目通常需要回读原始 KV 数据以重新计算哈希，导致严重的读放大 3333。
        
    - **并发控制引入的高额 RTT**：为了保证数据一致性（避免插入丢失或重复键），现有方案通常采用基于锁（Lock-based）或基于重读（Reread-based）的策略。锁机制需要额外的 `Lock` 和 `Unlock` 网络往返（RTT）；重读机制（如 RACE）在 `CAS` 写入后需要再次读取 Bucket 进行检查。这些操作都显著增加了写入延迟 4444。
        
    - **分层结构牺牲了读性能**：为了优化写入，一些方案（如 CLevel, Plush）采用了类似 LSM-tree 的分层结构。这导致读操作必须逐层搜索，引发读放大。此外，这些方案最初是为持久化内存（NVM）设计的，倾向于发起大量小粒度的读写，无法充分利用 RDMA 在大块数据传输上的高带宽优势 555555555。
        
- **底层开销分析**
    
    - **RDMA CAS 的局限性**：在并发写入时，为了争抢空闲槽位，客户端必须频繁使用原子指令 `RDMA CAS`。由于 `CAS` 操作本身具有较低的吞吐上限（相比于 `WRITE/READ`），且失败重试会进一步浪费网络资源 6666。
        
    - **网络往返（RTT）累积**：在存算分离架构中，每一次元数据的更新（如更新指纹、过滤器、链表指针）若采用同步等待确认的方式，都会引入毫秒级的网络延迟。传统的同步 IO 模型导致 CPU 在等待 ACK 时处于空转状态 7。
        

# 解决方案 (The Solutions)

## 存写分离的双层段结构 (Separate Segment Structure)

SepHash 提出了一种双层索引结构，由负责接收写入的小型无序段（CurSegment）和负责存储的大型有序段（MainSegment）组成 8。数据流完全是批处理导向的，消除了单条目移动的开销。

- **机制拆解**
    
    - **Shared CurSegment**：CurSegment 中的所有条目共享哈希后缀，且采用追加写（Append-only）模式。这扩大了哈希冲突的容忍空间，避免了因桶（Bucket）溢出导致的频繁分裂 9999。
        
    - **Resize-optimized Entry（核心设计）**：为了解决扩容时的读放大问题，SepHash 在每个 Entry 中预存储了部分哈希值（4 bits depth info）。在段分裂（Segment Split）时，客户端可以直接根据这些预存的位信息决定条目的去向，而无需通过 RDMA 读取远程的原始 KV 数据 10101010。
        
    - **Batch Merge/Split**：当 CurSegment 满时，触发 Merge 操作，将其批量合并到 MainSegment；当 MainSegment 满时，触发 Split。旧数据的清理通过翻转元数据中的 `sign` 位瞬间完成，无需逐个置空 11111111。
        

<img src="" alt="image" width=100% />

> Researcher's Comment
> 
> 这种 Separate Segment 的设计本质上是一种“写缓冲 + 批量落盘”的思想在 RDMA 环境下的变体。其最精妙之处在于 Entry 结构的设计：它牺牲了 Entry 中微小的空间（4 bits）来存储 depth info，换取了在 Resize 过程中原本必须发生的昂贵的远程 KV 读取（Read Amplification）。这是典型的用空间换时间（更是换带宽）的 Trade-off，非常适合存算分离这种“计算昂贵、IO更昂贵”的场景。

## 低 RTT 并发控制 (RTT-reduced Concurrency Control)

SepHash 设计了一套无需额外 RTT 的并发控制策略，结合了追加写、滑动窗口和协程技术 12。

- **机制拆解**
    
    - **Append Write & Sliding Window**：所有客户端使用 `RDMA CAS` 顺序争抢 CurSegment 中的空闲位置。客户端在本地维护一个 `offset` 滑动窗口，记录上一次探测到的第一个空槽位，从而避免每次都扫描整个 Segment 13131313。
        
    - **Zero-wait Write（协程优化）**：插入操作包含三个步骤：CAS 抢占 Entry、更新指纹 `fp2`、更新过滤器 `filter`。SepHash 发现后两步是幂等的且不需要返回值。因此，它利用协程（Coroutine）机制，将这两个 `RDMA WRITE` 请求异步发出（Post）后立即返回，不等待 Completion Queue 的确认，从而隐藏了网络延迟 14141414。
        

<img src="" alt="image" width=100% />

> Researcher's Comment
> 
> 这里的 Zero-wait write 是对 RDMA 异步特性的极致利用。传统模型中，为了保证元数据一致性，我们往往会等待每一次 Write 完成。但 SepHash 敏锐地指出，只要 CAS 成功占据了槽位（这一步是强一致的），后续的 filter 和 fp2 更新即便稍有延迟也不影响正确性（最多导致稍微增加一点误判率或读开销，但在 Write-optimized 场景下完全可接受）。利用协程在用户态调度 IO，彻底榨干了 CPU 周期。

## 高效缓存与过滤器 (Efficient Cache and Filter)

为了弥补分层结构带来的读性能损失，SepHash 在客户端和内存端引入了特定的加速结构。

- **机制拆解**
    
    - **Bitmap Filter**：在 CurSegment 的元数据中维护一个位图（Bitmap）。查询时，客户端并发读取 MainSegment 和这个 Bitmap。只有当 Bitmap 对应位为 1 时，才去读取 CurSegment 的内容。这只需要一次 RDMA Read 15151515。
        
    - **Record Point Based FPTable**：针对 MainSegment，客户端缓存了一个 FPTable 来索引指纹分布。为了节省客户端宝贵的内存，SepHash 没有存储每个指纹的计数，而是利用线性回归模型（$pre_i = avg \times i + \delta$）来拟合指纹分布，仅在预测误差超过阈值时才插入“记录点”（Record Point）。这种压缩方案相比朴素实现节省了高达 67% 的缓存空间 161616161616161616。
        

> Researcher's Comment
> 
> 在客户端缓存索引元数据是常规操作，但 SepHash 的 Record Point 设计非常有意思。它利用了哈希函数分布均匀的特性，用线性函数拟合位置，把元数据索引变成了一个“数学预测问题”而非单纯的“查表问题”。这种对于数据分布特性的利用，使得它在极小的客户端内存占用下实现了高效的寻址，这对于未来内存受限的 SmartNIC 或计算侧客户端极具参考价值。

# 核心亮点 (Key Insights & Trade-offs)

- **原子性与性能的解耦**：SepHash 将强一致性需求集中在 `CAS` 抢占 Entry 这一步，而将辅助元数据（如 Filter）的更新弱化为异步操作。这种设计打破了传统“所有步骤必须同步完成”的思维定式。
    
- **计算卸载到数据布局**：由于无法在内存端执行代码，SepHash 通过预计算（Local Depth Info）和特定布局（Shared Segment），将原本需要在扩容时进行的“重哈希计算”转化为了简单的“位检查”，从而规避了存算分离架构下的“计算墙”问题。
    
- **带宽友好的 IO 粒度**：不同于 Leveling Hash 的小粒度读写，SepHash 的 Segment Merge/Split 机制确保了数据在网络上主要以较大的块（Batch）进行传输，这契合了 RDMA 在大包传输下更高带宽利用率的硬件特性 17。