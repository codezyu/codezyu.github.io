Header:

论文出处：OSDI '24 1111。

一句话总结：在存算分离（Memory Disaggregation）架构下，利用RDMA网络连接计算池与内存池，但这引入了高昂的网络往返（RTT）延迟瓶颈，且内存池计算能力微弱无法处理复杂的事务逻辑，导致传统多版本并发控制（MVCC）无法直接应用 222222222。

---

# 问题

- **单版本方案（Single-Versioning）的局限性**：
    
    - 现有的存算分离事务系统（如 FORD 3）采用单版本设计。这导致**读操作必须等待写操作完成**（读写阻塞），严重限制了并发度 4。
        
    - 为了保证原子性，必须向所有副本写入**Undo Logs**。这些日志不仅消耗了宝贵的网络带宽，协调者还必须等待日志写入的 ACK 才能提交更新，增加了延迟 5555。
        
- **传统多版本方案（MVCC）的水土不服**：
    
    - **不兼容的事务协议（Incompatible Protocol）**：传统的 MVCC 依赖服务器端强大的 CPU 进行锁管理、验证或时间戳计算。但在存算分离架构中，内存池的 CPU 非常弱（仅用于内存分配或连接管理），无法频繁轮询和执行这些任务 6666。
        
    - **低效的版本结构（Pointer Chasing 导致 RTT 爆炸）**：
        
        - 传统方案使用链表（Linked Chains）来组织版本（如 Old-to-New 或 New-to-Old 链）。
            
        - **底层开销分析**：在单机内存中，CPU 进行“链表游走（Chain Walking）”很快。但在存算分离架构下，数据存储在远端。计算池每读取链表中的一个节点，就需要消耗一次 **RDMA RTT**。
            
        - 数据表明，当链表遍历步数从 1 增加到 20 时，RDMA 读取延迟增加了 **24.8倍** 7。
            
    - **垃圾回收（GC）困难**：计算池需要频繁追踪最老的活跃事务来回收版本，这会消耗大量的 RTT 用于同步状态，浪费计算资源 8。
        

> Researcher's Comment:
> 
> 这种不适应性本质上是 "Memory Access Paradigm" 的冲突。传统 MVCC 假设的是 Random Access Latency 极低（本地 DRAM ~100ns），因此可以使用指针随意跳跃。而在存算分离中，虽然 RDMA 很快（~2-3µs），但相比本地内存依然是数量级的差异。任何依赖 $O(N)$ 次远程内存访问的数据结构（如链表）在这里都是性能杀手。

---

# 解决方案 (The Solutions)

## 1. 连续版本元组 (Consecutive Version Tuple, CVT)

Motor 彻底抛弃了链表结构，提出了一种新的内存布局 **CVT**。

- **机制拆解**：
    
    - **扁平化存储**：Motor 将一条记录的多个版本（$VNum$）**连续**地存储在一段连续的物理地址空间中，而不是通过指针链接 9。
        
    - **单次 RTT 获取**：计算池可以通过**一次 RDMA READ** 操作读取整个 CVT（包含多个版本的元数据），从而在本地进行版本选择，消除了远程“链表游走”的开销 10。
        
    - **分离的值区域 (Separate Value Region)**：为了避免 CVT 过大导致 RDMA payload 增加（影响延迟），Motor 将具体的 Data Value 与 Version Metadata 分离存储。CVT 中仅包含元数据（Key, Lock, Pointers），实际的数据存储在独立的 Value Region 11。
        
    - **增量存储 (Attribute Bar)**：利用 OLTP 负载通常只修改少量属性的特点，Motor 在 Delta Area 中仅存储**被修改的属性（Modified Attributes）**，而不是全量数据，极大降低了内存开销 12121212。
        

<img src="[https://via.placeholder.com/800x400?text=Structure+of+CVT+and+Value+Region+(Fig+3](https://www.google.com/search?q=https://via.placeholder.com/800x400%3Ftext%3DStructure%2Bof%2BCVT%2Band%2BValue%2BRegion%2B\(Fig%2B3\))" alt="image" width=100% />

> Researcher's Comment:
> 
> 这个设计非常聪明地利用了 "Spatial Locality" (空间局部性) 来换取 "Network Round Trips"。虽然一次读取整个 CVT 可能会读到一些无用的旧版本数据（浪费了一点带宽），但相比于多次 RTT 带来的延迟惩罚，这个 Trade-off 是绝对值得的。特别是在 RDMA 环境下，Payload 大小对延迟的影响远小于 RTT 次数的影响。

## 2. 锚点辅助读取 (Anchor-Assisted Read)

由于数据（Value）和版本（Version）是分离存储的，无法通过单次原子 RDMA 读取同时获取两者，这可能导致读到不一致的数据（例如读到了版本 $V_1$ 的元数据，但取回了并发 GC 写入的 $V_9$ 的值）13131313。

- **机制拆解**：
    
    - Motor 引入了 4 个 1字节的锚点（Anchors）：
        
        - **VcellSA / VcellEA**：位于 CVT 中版本单元的头部和尾部 14。
            
        - **VpkgSA / VpkgEA**：位于 Value Package 数据包的头部和尾部 15。
            
    - **写入流程**：协调者在写入时，会增加所有 4 个锚点的值，使其相等，并利用 RNIC 保证写入顺序（禁用 DDIO 以防止缓存乱序刷回）16161616。
        
    - **读取流程**：读取者检查 **$VcellSA = VcellEA = VpkgSA = VpkgEA$**。只有当这四个值完全相等时，才认为读取到的数据是一致的 17171717。
        
    - **无锁读取**：这种机制允许读操作**完全绕过锁**，且无需像 Silo 那样进行两次版本读取验证，即可检测并发冲突或 GC 干扰 18。
        

<img src="[https://via.placeholder.com/800x400?text=Anchor-Assisted+Read+Mechanism+(Fig+5](https://www.google.com/search?q=https://via.placeholder.com/800x400%3Ftext%3DAnchor-Assisted%2BRead%2BMechanism%2B\(Fig%2B5\))" alt="image" width=100% />

> Researcher's Comment:
> 
> 这是一个典型的 "End-to-End Argument" 的应用。在缺乏硬件事务内存（HTM）或远端锁支持的情况下，通过在应用层协议（锚点检查）来保证跨越多个内存区域的数据一致性。它利用了 RDMA WRITE 的顺序性（Ordering）假设，巧妙地构建了一个软件层面的“原子快照”。

## 3. 协调者主动式 GC 与 RDMA 事务协议

Motor 设计了一套完全基于 **One-sided RDMA** 的 MVCC 协议，无需内存池 CPU 参与。

- **机制拆解**：
    
    - **抢占式 GC (Coordinator-Active GC)**：不同于传统方案追踪“最老活跃事务”，Motor 允许协调者在写入新版本且没有空槽位时，**直接覆盖** CVT 中最老的版本（Victim Version）19191919。虽然这可能导致正在读取该旧版本的长事务 Abort，但避免了昂贵的全局追踪开销。
        
    - **灵活的隔离级别**：
        
        - **Serializability (SR)**: 通过在 $T_{start}$ 读取并在 $T_{commit}$ 阶段**重新验证**读集（Validation）来实现 20202020。
            
        - **Snapshot Isolation (SI)**: 禁用读集验证，允许事务读取 $T_{start}$ 时的快照，进一步提高吞吐 21。
            
    - **无日志提交**：由于保留了多版本，旧版本天然充当了“Undo Logs”。如果事务 Abort，只需丢弃新写入的数据即可，无需回滚日志，节省了网络带宽 22。
        

---

# 核心亮点 (Key Insights & Trade-offs)

- 一致性与性能的精妙平衡：
    
    Motor 通过 Anchor-Assisted Read 解决了存算分离下“数据-元数据分离”导致的一致性难题。它不需要对远端内存加读锁，也不需要多次 RTT 握手，仅通过简单的整数比较就实现了快照隔离。这种设计将复杂性完全卸载到了计算侧（Coordinator），完美契合 Disaggregated Memory 中“胖计算、瘦内存”的硬件特征。
    
- **空间换时间的极致权衡**：
    
    - **CVT 的版本数量 ($VNum$) 选择**：这是一个关键的 Trade-off。$VNum$ 太小会导致高频的 GC 和事务 Abort；$VNum$ 太大会导致 CVT 膨胀，增加 RDMA 读取延迟。实验表明，对于像 TATP 这样的低竞争负载，2 个版本足矣；而对于 TPCC，4 个版本是最佳平衡点 23232323。
        
    - **Attribute Bar**：为了缓解多版本带来的内存压力，Motor 放弃了存储全量版本数据，而是仅存储 Delta。这使得即使保留多个版本，其内存开销也仅比单版本系统（FORD）高出 **17.3% - 37.7%**，远低于预期的倍数增长 24。
        
- 性能提升：
    
    相比于最先进的单版本系统 FORD，Motor 在 TPCC 负载下吞吐量提升高达 98.1%，延迟降低 55.8% 25。这是多版本并发控制（读写不阻塞）与 RDMA 友好型数据结构（CVT 减少 RTT）共同作用的结果。