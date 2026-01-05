Header:

来源于 EuroSys '25。本文针对存算分离（Disaggregated Memory, DM）架构下，传统 B+Tree 索引面临的 I/O 放大严重导致网络带宽饱和 以及 远程锁竞争导致扩展性差 的物理瓶颈，提出了一种高可扩展的树索引设计方案。

# 问题

- **I/O 放大与 Round-Trip 的两难困境**：
    
    - 在 DM 架构中，计算节点（CN）必须通过单边 RDMA 动词访问存储节点（MN）。为了减少昂贵的网络 Round-Trip（RTT），传统设计倾向于使用大扇出（Large Fanout）的树节点（例如 1KB）来降低树高 1。
        
    - 然而，这种设计导致了严重的 **I/O 放大**：CN 必须取回整个节点才能在本地搜索，这迅速耗尽了 RNIC 的网络带宽 2。
        
    - 简单地减小节点大小虽然能缓解带宽压力，但会增加树高（$O(\log_F N)$），导致更多的网络 RTT，从而增加延迟 3。
        
- **Write-Write 并发控制的底层开销**：
    
    - 基于锁的并发控制（通常使用 `RDMA_CAS`）在冲突激烈时扩展性极差 4。
        
    - 获取远程互斥锁需要多次重试（Retries），每次重试都是一次跨网络交互，且过多的重试请求会堆积在 RNIC 中消耗 IOPS 5。
        
    - 粗粒度的节点锁（Node-level lock）限制了同一节点的并发操作，在倾斜负载（Skewed Workloads）下性能崩塌 6。
        
- **Read-Write 乐观同步的低效性**：
    
    - 为了实现无锁读取，现有方案（如 Sherman）可能基于错误的假设（即假设 `RDMA_READ` 保证地址递增顺序），导致数据不一致 7。
        
    - **Checksum**：计算昂贵，且限制了读取粒度必须为完整对象 8。
        
    - **Versioning**：由于 PCIe 层可能对读取进行重排，读取一个对象需要 3 次 `RDMA_READ`（读版本 -> 读数据 -> 验证版本），无法流水线化 9999。
        
    - **FaRM 缓存行版本**：写入时需要更新所有涉及的缓存行版本，导致 **写放大（Write Amplification）**，需要通过 `RDMA_WRITE` 写入整个对象 10。
        

# 解决方案 (The Solutions)

## 细粒度访问模式设计 (Fine-grained Access Patterns)

DEFT 的核心在于打破“读取整个节点”的定式，通过重构节点布局来实现细粒度访问，从而在不增加树高的前提下降低 I/O 放大。

- **分段内部节点 (Segmented Internal Node)**：
    
    - DEFT 将内部节点逻辑上划分为多个未排序的 **子节点 (Sub-nodes)**，但物理上仍保持部分有序 11。
        
    - CN 通过计算而非额外的指针来定位子节点范围，从而避免了树体积增大。搜索时，只需通过一次 `RDMA_READ` 获取目标子节点（粒度远小于整个节点）12121212。
        
    - 由于子节点内部未排序，插入操作只需原子写入（Extended `RDMA_CAS`），避免了传统有序节点在插入时的移位操作（Shift operations）13131313。
        
- **哈希叶子节点 (Hash-based Leaf Node)**：
    
    - 叶子节点采用哈希结构，包含多个 Bucket Pair（主桶 + 溢出桶）14141414。
        
    - 搜索时，仅需读取目标 Key 对应的 Bucket Pair，极大减少了读取量 15。
        
    - 利用哈希的低冲突特性，支持在同一个叶子节点内进行并发的 Upsert（更新/插入）操作 16。
        

<img src="https://i.imgur.com/PlaceholderForSegmentedNode.png" alt="image" width=100% />

(此处应展示论文 Figure 5 Segmented Internal Node 的结构图)

> Researcher's Comment:
> 
> 这种设计非常巧妙地在“逻辑结构”和“物理存储”之间做了一个解耦。它利用计算换取了带宽（Trading computation for bandwidth）。本质上，它是在一个大的物理页面（Page）内构建了一个微型的、扁平的索引结构。这种设计规避了 Radix Tree 因节点过小导致的树高爆炸问题，同时也解决了 B+Tree 大节点的带宽浪费问题。这是一个典型的针对 RDMA 带宽瓶颈的系统级优化。

## 可扩展的并发协调 (Scalable Concurrency Coordination)

为了解决 `RDMA_CAS` 的争用问题，DEFT 设计了一套基于单边操作的共享-互斥（SX）锁机制。

- **单边 Shared-Exclusive (SX) Lock**：
    
    - 基于 Lamport 的面包店算法（Bakery Algorithm）改进，锁结构包含 Shared 和 Exclusive 的 Ticket 与 Counter 17。
        
    - 使用 `RDMA_FAA` (Fetch-and-Add) 来获取 Ticket，这是一种无等待（Wait-free）的网络操作，避免了 CAS 循环重试带来的通信开销 18181818。
        
    - **并发 Upsert**：对于叶子节点的更新和插入，线程只需获取 **共享锁 (Shared Lock)** 即可进行，只有结构修改操作（SMO，如分裂）才需要互斥锁 19191919。
        
- **RDMA 操作合并**：
    
    - 利用 RDMA RC 连接的顺序性，DEFT 将“释放锁”和“写入数据”合并在一次 RTT 中 20。
        
    - 乐观地将“获取锁”和“读取数据”合并，假设共享锁获取大概率成功 21。
        

> Researcher's Comment:
> 
> 这里最精彩的点在于将大部分写操作（Upsert）降级为“共享锁”模式。在传统数据库中，写操作通常需要排他锁。DEFT 利用了哈希叶子节点内 Slot 的原子性（通过 CAS 保证 Entry 级别的原子性），使得节点级别的锁只需要保护结构完整性（SMO）。这种**锁粒度的层级分离（Lock Granularity Hierarchy）**是其在高并发写场景下扩展性优于 Sherman 的关键。

## On-Path Decoupled Versioning (OPDV)

为了实现零额外 RTT 的无锁读取，DEFT 提出了一种利用树搜索路径特性的版本控制方案。

- **版本解耦**：
    
    - DEFT 将一致性验证所需的版本号拆分：**提交版本 ($V_C$)** 存储在父节点指向当前节点的指针中，**数据版本 ($V_D$)** 存储在当前节点的数据行中 22222222。
        
    - **流程**：在遍历树的过程中（Pointer Chasing），CN 在读取父节点时就已经获得了子节点的 $V_C$。随后读取子节点数据时，只需比对子节点内部的 $V_D$ 与之前获得的 $V_C$ 23。
        
- **消除额外读取**：
    
    - 相比于传统 Versioning 需要 "Read Version -> Read Data -> Read Version" 的 3 次 RTT，OPDV 完美地将版本检查融合进了树的遍历路径中，不需要额外的 `RDMA_READ` 来获取基准版本 24。
        
    - 由于 `RDMA_WRITE` 遵循 PCIe 的地址递增顺序，写入者先写 $V_D$ 再写数据，或者 $V_C$ 的更新在最后，从而保证了读者能通过不匹配的版本号检测到不一致 25。
        

<img src="https://i.imgur.com/PlaceholderForOPDV.png" alt="image" width=100% />

(此处应展示论文 Figure 7 Node layout with OPDV 的结构图)

> Researcher's Comment:
> 
> OPDV 是本文在算法层面最优雅的创新。它敏锐地洞察到树结构索引的访问特点——即“必须经过父节点才能到达子节点”。通过将 metadata（版本号）沿路径（On-path）分布，它消除了专门为了由一致性检查而产生的网络往返。这在微秒级延迟的 RDMA 环境中收益巨大。

# 核心亮点 (Key Insights & Trade-offs)

- **计算与传输的权衡 (Computation vs. Transmission Trade-off)**：
    
    - DEFT 的设计哲学是：在计算资源受限的 MN 端不做处理，在 CN 端增加计算复杂度（如计算子节点偏移、哈希计算），以换取宝贵的 RDMA 带宽资源。考虑到 PCIe 带宽和 RNIC 吞吐通常是瓶颈，这种权衡在现代硬件上是极具性价比的 26。
        
- **原子性与锁的协同**：
    
    - DEFT 并没有完全抛弃锁，而是通过利用硬件原子操作（Extended `RDMA_CAS` 支持 32 字节原子写）来处理 Entry 级别的数据竞争，仅在 SMO 这种重操作上使用软件锁 27。这种**软硬协同（Hardware-Software Co-design）** 避免了全软件锁的低效。
        
- **极致的带宽利用**：
    
    - 通过将读取粒度从 1KB（整个节点）降低到 ~240B（子节点或 Bucket Pair），DEFT 在高负载下的吞吐量不再受限于 RNIC 的物理带宽上限，这是它能比 Sherman 高出 9.5 倍性能的根本物理原因 28282828。