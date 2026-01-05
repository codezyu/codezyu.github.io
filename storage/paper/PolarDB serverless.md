Header:

论文出处：SIGMOD '21。本文针对存算分离架构（Disaggregated Data Centers, DDC）下，传统单机数据库面临的资源耦合导致的弹性不足、以及引入远程内存（Remote Memory）后带来的高网络延迟（Network Latency）与缓存一致性（Cache Coherency）挑战，提出了一种基于 RDMA 网络和共享内存池的 Serverless 数据库架构。

# 问题

- **原有方案的不适应性**
    
    - **资源紧耦合与装箱问题**：在单机架构下，CPU、内存和存储资源紧密耦合，导致云服务商在分配实例时面临装箱问题（Bin-packing problems），难以同时维持各类资源的高利用率，且容易产生碎片 1。
        
    - **Serverless 弹性的局限性**：现有的 Serverless 数据库（如基于共享存储的架构）中，CPU 和内存仍然是绑定的。扩缩容或自动暂停（Auto-pause）时，必须同时释放 CPU 和内存，导致恢复时间长，因为必须从磁盘重新加载数据到内存（Cold Cache） 2222。
        
    - **故障恢复的命运共享（Fate Sharing）**：单机架构下，单一资源（如内存）故障会导致整个节点服务不可用，无法独立恢复 3。
        
- **底层开销分析**
    
    - **网络延迟惩罚**：在 DDC 架构中，数据访问可能跨越网络，访问远程内存的延迟远高于本地内存，导致 CPU 流水线停顿（Stalling）等待数据填充缓存行 4444。
        
    - **分布式索引的一致性开销**：当计算节点（RW/RO）并发访问共享的 B+Tree 索引时，传统的本地锁机制失效。维护跨节点的物理结构一致性（Structural Consistency）需要昂贵的全局锁协调 5555。
        
    - **脏页刷脏的网络风暴**：传统数据库周期性将脏页刷入存储，在存算分离架构下会产生大量写流量，阻塞关键路径上的日志写入和页面读取 6。
        
    - **复杂的缓存一致性维护**：RW 节点修改页面后，必须确保所有持有该页面副本的 RO 节点感知到数据过期，这涉及跨节点的同步通信开销 7。
        

# 解决方案 (The Solutions)

## 存算完全分离的内存架构设计

PolarDB Serverless 采用了计算（CPU）、内存（Remote Memory Pool）和存储（PolarFS）三层解耦的架构。核心在于引入了可独立扩展的远程内存池，通过 `librmem` 接口管理。

- **机制拆解**：
    
    - **分层缓存系统**：引入本地缓存（Local Cache）作为第一层，大小约为远程内存的 $1/8$，用于缓解远程访问的延迟；远程内存（Remote Memory）作为第二层，通过 RDMA 互联；底层是分布式存储 8。
        
    - **元数据管理**：在 Slab 节点的 Home Node 上维护全局元数据，包括页地址表（PAT）、页失效位图（PIB）和页引用目录（PRD）。PIB 用于标记远程内存中的页面是否因 RW 节点的本地修改而过期 999999999。
        
    - **RDMA 原语的使用**：利用单边 RDMA READ/WRITE（One-sided RDMA verbs）直接存取远程页面，减少 CPU 参与 10。
        

<img src="Figure 6: Cache Invalidation" alt="image" width=100% />

> 专家点评
> 
> 这种设计本质上是将操作系统的 Swap 机制通过 RDMA 扩展到了分布式集群。它非常巧妙地利用了 RDMA 的单边操作特性，将大部分元数据查找和数据搬运卸载到了网卡，避免了远端 CPU 的上下文切换开销。PIB（Page Invalidation Bitmap）的设计是一个经典的“空间换一致性”的权衡，通过维护一个全局位图来快速判断页面版本的有效性，避免了广播查询。

## 乐观并发控制与 RDMA 加速的 B+Tree 维护

为了解决跨节点索引访问的一致性问题，系统引入了全局页锁（Global Page Latch, PL），但为了性能，大量使用了乐观策略。

- **机制拆解**：
    
    - **全局页锁（PL）**：在 Home Node 的 PLT 表中维护。RW 节点在进行结构修改操作（SMO, 如分裂/合并）时，需获取相关页面的 X-Lock；RO 节点读取时需获取 S-Lock 11111111。
        
    - **RDMA CAS 锁获取**：为了加速锁获取，优先尝试使用 RDMA CAS（Compare-And-Swap）原子操作直接修改远端锁状态，只有在失败（如冲突）时才回退到基于消息的协商机制 12。
        
    - **乐观锁（Optimistic Locking）**：在 RW 节点维护一个全局 $SMO$ 计数器。RO 节点遍历树时，先假设没有 SMO 发生（不加全局锁）。查询结束时，检查路径上页面的 $SMO_{page}$ 版本是否大于查询开始时的全局 $SMO_{query}$。若大于，说明途中发生了结构变更，需重试或回退到悲观锁 13131313。
        

> 专家点评
> 
> 这是一个教科书级别的软硬结合优化案例。传统的分布式锁（Distributed Lock Manager）开销巨大，通常需要多次网络往返（RTT）。这里利用 RDMA 硬件提供的原子性 CAS 指令，将锁操作压缩到单次网络交互。同时，乐观锁策略利用了 OLTP 场景下“结构变更（SMO）远少于普通读写”的负载特征，极大地减少了对全局锁的争用。

## 页面物化卸载 (Page Materialization Offloading)

为了消除脏页刷盘带来的网络瓶颈，PolarDB Serverless 借鉴了 "Log is Database" 的思想，但做了进一步解耦。

- **机制拆解**：
    
    - **日志即数据**：RW 节点只将 Redo Log 写入存储层的 Log Chunk，而不直接刷写脏页 14。
        
    - **异步物化**：存储节点（Page Chunk）接收日志后，在后台负责 Replay 日志并生成新的页面版本。RW 节点在收到 Log 持久化的确认后，即可从本地缓存中驱逐脏页，无需回写到存储 15151515。
        
    - **冷热分离恢复**：在 RW 节点故障恢复时，由于大部分热数据仍驻留在远程内存中且未丢失（独立供电/节点），新 RW 节点只需处理少量日志即可快速恢复服务，无需经历漫长的缓存预热（Cache Warm-up） 16。
        

<img src="Figure 7: Page Materialization Offloading" alt="image" width=100% />

> 专家点评
> 
> 这一设计彻底改变了数据库的 I/O 模型。传统数据库的 Checkpoint 是性能抖动的元凶之一。通过将“页面重放”这一计算密集型任务下推（Pushdown）到存储节点，既节省了计算节点的 CPU，又利用了存储集群的聚合算力。这实际上是将存储层变成了一个专门负责数据重构的计算单元，体现了 Near-Data Processing 的设计哲学。

# 核心亮点 (Key Insights & Trade-offs)

- **极速故障恢复（Fast Failover）**：利用存算分离特性，RW 节点崩溃后，内存状态保留在独立的内存节点中。新 RW 节点接管时，缓存命中率几乎不下降，故障恢复速度比传统架构快 5.3 倍 17171717。
    
- **一致性与性能的权衡**：通过**缓存失效（Invalidation）**而非**更新传播（Propagation）**来维护一致性。RW 修改页面后，仅通知 RO 标记失效，RO 下次访问时再拉取最新版。这避免了写操作被广播风暴拖慢，牺牲了 RO 节点的单次读取延迟（Cache Miss）来换取写性能的扩展性 18。
    
- **硬件感知的预取（Index-Aware Prefetching, BKP）**：针对远程访问的高延迟，利用数据库执行计划（SQL Plan）的信息，提前计算 B+Tree 的 Key 并批量发射 RDMA 请求进行预取，掩盖了 PCIe 和网络链路的 Round-trip 开销 19191919。
# 怎么分配内存的

远程内存被切分为 1GB 大小的 Slabs，分布在多个 Slab Nodes 上。每个实例按需申请 Slab，解决了传统物理机的资源碎片问题
# 缓存一致性
这是一个非常敏锐的问题！你触碰到了分布式缓存一致性协议中最关键的环节：**元数据（Metadata）与数据（Data）在传输上的解耦**。

你的直觉是对的：如果 RW 仅仅更新 Home Node 上的 PIB，RO 节点确实无法凭空“通灵”感知到变化。

根据论文的 **3.1.4 Cache Coherency** 章节以及 **Figure 6** 的描述，PolarDB Serverless 实际上采用了一种 **“同步推失效（Push Invalidation），按需拉数据（Pull Data）”** 的策略。RW 并不是只更新了 Home Node，而是通过一个**同步阻塞**的流程强制更新了持有该页面副本的 RO 节点的本地 PIB。

以下是具体的底层交互流程拆解：

### 核心机制：PRD (Page Reference Directory) 指路

RW 节点发起的 `page_invalidate` 操作是一个**涉及多方的同步事务**，不仅仅是修改 Home Node 111。

1. **RW 发起失效**：当 RW 节点修改了一个本地页面后，它调用 `page_invalidate` 2。
    
2. **Home Node 查表 (PRD)**：
    
    - 首先，更新 Home Node 自己的全局 PIB，标记该页面已脏 3333。
        
    - **关键一步**：Home Node 查找 **PRD (Page Reference Directory)**。PRD 记录了当前有哪些 RO 节点 `page_register` 了这个页面（即哪些 RO 的本地缓存里有这个页面的副本）4444。
        
3. **定向广播失效 (Targeted Invalidation)**：
    
    - 根据 PRD 查到的名单，系统会**直接修改这些特定 RO 节点上的本地 PIB**，将其对应位设置为 1（失效）5555。
        
    - 注意：论文并未明确说明是 RW 拿到列表后去写，还是 Home Node 代为去写，但根据 RDMA 优化的上下文，通常是通过 RDMA WRITE 直接修改 RO 节点注册的 PIB 内存区域。
        
4. **同步阻塞 (Barrier)**：
    
    - `page_invalidate` 是一个**同步阻塞操作**。只有当 **Home Node 的 PIB** 和 **所有相关 RO 节点的本地 PIB** 都被成功置位后，该函数才会返回成功 6。
        
    - 如果有 RO 节点超时未响应（例如宕机），Cluster Manager 会介入将其踢出集群，以确保一致性不被卡死 7。
        

### 总结 (The "Aha!" Moment)

所以，RO 节点并不是“猜”出来的，而是被“通知”的（通过被修改了本地 PIB 内存）。

- **数据流 (Data Path)**：是 **Lazy** 的。RW 改了数据，RO 不知道新数据是什么，只知道“我手里的旧了”。
    
- **控制流 (Control Path)**：是 **Eager** 且 **Synchronous** 的。RW 在修改完成前，必须确保所有持有旧副本的 RO 都已经被打上了“失效标记”。
    

> Researcher's Comment:
> 
> 这种设计极其聪明地利用了 RDMA 的特性。
> 
> 如果是传统 TCP/IP，给所有 RO 发消息通知失效的开销极大（CPU 中断、上下文切换）。
> 
> 但在这里，更新 RO 的 PIB 可能仅仅是一次 One-sided RDMA WRITE 操作，直接修改 RO 的内存，RO 的 CPU 甚至不需要参与（Zero-CPU overhead）。
> 
> RO 只有在下次真的需要读取该页面时，检查自己的 PIB 发现是 1，才会触发 Cache Miss 逻辑去拉取新数据。这是典型的“写时主动通知元数据，读时被动拉取数据”的优化策略。