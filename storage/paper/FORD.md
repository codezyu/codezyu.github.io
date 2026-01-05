Header:

**论文出处**: FAST '22 1111**一句话总结**: 本文针对存算分离架构下的持久内存（Disaggregated Persistent Memory, DPM）场景，解决了其面临的三大物理瓶颈：DPM 池端缺乏 CPU 算力无法处理复杂逻辑、One-sided RDMA 操作带来的高额网络往返（RTT）开销，以及真实 PM 介质写入带宽受限且缺乏远程持久化保证的问题 222222。

**# 问题**

- 原有分布式事务方案在 DPM 上的失效
    
    传统方案（如 FaRM, DrTM+H）通常采用乐观并发控制（OCC）和主备复制（Primary-Backup Replication）。在 DPM 架构下，内存池中的 CPU 极弱或不参与计算，导致无法执行 RPC 风格的逻辑（如日志重放、数据查找）。现有方案若强行使用 One-sided RDMA 模拟，会导致极高的延迟 3333。
    
    具体表现为：一个读写事务在 Execution、Locking、Validation 阶段需要消耗 3 个 RTT 才能开始更新副本；而在提交阶段，为了保证高可用，通常需要 2 个 RTT（先写备份 Log，再更新主节点） 4。
    
- PM 物理带宽瓶颈与竞争
    
    论文指出，真实的 PM DIMM（如 Optane）写入带宽显著低于现代 RNIC 网卡。例如，6 条交织的 256GB PM DIMM 写带宽仅为 12.9 GB/s，而双端口 ConnectX-5 RNIC 可达 25 GB/s 5。
    
    传统方案只允许从 Primary 副本读取数据（因为 Backup 中的数据通常以 Log 形式存在，未 Apply），这导致 Primary 节点的 PM 既要处理所有的读请求，又要处理写请求，极易饱和并阻塞写操作，导致吞吐量下降 6666。
    
- 底层持久化语意缺失 (Remote Persistency)
    
    现有的 RDMA Write 操作仅保证数据到达远程 RNIC 的易失性缓存（Volatile Cache），并不保证数据落入持久化介质 7777。若此时发生掉电或崩溃，客户端以为已持久化的数据将会丢失。全量使用 RDMA FLUSH（或模拟操作）会导致 RTT 数量激增 8。
    

**# 解决方案 (The Solutions)**

## Hitchhiked Locking (搭便车锁机制)

为了消除读写集（Read-Write Set）在 Locking 和 Validation 阶段额外的 RTT 开销，FORD 提出了一种将“锁”依附在“读”操作上的机制。

- 机制拆解：
    
    利用 RDMA 网卡的 Doorbell Batching 特性，FORD 将一个 RDMA CAS（用于锁定）和一个 RDMA READ（用于读取数据）打包成一个请求发出 9。
    
    在执行阶段（Execution Phase），协调者（Coordinator）直接对读写集的数据尝试加锁并读取。如果 CAS 成功（返回值与预期一致），则视为锁定成功且数据有效；否则事务 Abort。这样将 Read、Lock 和部分 Validation 合并到了 1 个 RTT 中 10101010。
    
    对于只读数据（Read-only Set），FORD 依然采用 OCC 的方式，仅读取不加锁，但在 Commit 前需进行一次版本校验 11。
    

<img src="ford_hitchhiked_locking.png" alt="image" width=100% />

> Researcher's Comment:
> 
> 这是一个非常典型的“Speculative Execution”思想在网络层面的应用。传统两阶段锁（2PL）是悲观的，OCC 是乐观的，而 Hitchhiked Locking 则是“激进的”。它赌的是在低竞争环境下，可以直接在读取数据的同时完成占位（Locking）。这种设计巧妙地利用了 RDMA PCIe 事务的有序性（CAS 先于 READ 执行），用 RNIC 的原子性保证了逻辑的原子性，从而避开了 DPM 端无 CPU 的尴尬。

## Coalescent Commit & Parallel Undo Logging (合并提交与并行 Undo Log)

为了解决传统主备复制需要 2 个 RTT（先 Log 后 Update）的问题，FORD 设计了并行 Undo Log 和合并提交协议。

- **机制拆解**：
    
    1. **Parallel Undo Logging**: 不同于传统的 Redo Log（需要 Commit 时发送），FORD 在执行阶段读取到旧数据后，立刻生成 Undo Log 并通过 `RDMA WRITE` 并行写入到远程副本中 12。这完全掩盖了写日志的网络延迟。
        
    2. **Coalescent Commit**: 在提交阶段，FORD 不再区分 Primary 和 Backup 的写入顺序，而是向所有副本（Primary + Backups）同时发起更新请求（1 个 RTT） 13。
        
    3. **Visibility Control**: 为了防止并发读取到未完全提交的数据，FORD 引入了一个 `VLock`（包含 1-bit 不可见标记）。在更新数据时，利用 Batching 同时写入数据并将 `VLock` 设为不可见。只有当所有副本都 ACK 后，才会在后台异步将数据置为可见（Release 阶段） 14141414。
        

> Researcher's Comment:
> 
> 这个设计最精彩的地方在于它抛弃了传统的日志重放（Log Replay）模型。在存算分离架构中，因为存储端没有 CPU 去 Apply Redo Log，所以备份节点如果只存 Log，根本无法被读取。FORD 强行让备份节点也进行 In-Place Update（原地更新），并通过 Undo Log 来保证原子性（失败了回滚）。这不仅减少了 RTT，更重要的是它让“读备份”成为了可能（见下文），彻底解耦了算力和存储。

## Backup-enabled Read & Selective Remote Flush (备份读与选择性刷写)

针对 PM 带宽瓶颈和持久化开销，FORD 做出了针对性的优化。

- **机制拆解**：
    
    1. **Backup-enabled Read**: 由于 Coalescent Commit 保证了 Backup 副本也是原地更新的最新数据，FORD 允许协调者直接从 Backup 节点读取只读数据（Read-only data） 15151515。这直接分摊了 Primary 节点的 PM 读取压力，释放了宝贵的写带宽。
        
    2. **Selective Remote Flush**: 为了保证远程持久性（Remote Persistency），必须将数据从 RNIC 缓存刷入 PM。FORD 观察到，只要 $(f+1)$ 个副本中的 $f$ 个备份副本持久化了，数据就是安全的（Primary 挂了可以用 Backup 恢复） 16。
        
    3. **Implementation**: 因此，FORD 仅在针对 Backup 的最后一次写入后触发 Flush 操作（当前实现通过 `RDMA READ-after-WRITE` 模拟 Flush，未来兼容 `RDMA FLUSH` 原语），而 Primary 节点不需要同步 Flush，从而大幅减少了 RTT 17171717。
        

> Researcher's Comment:
> 
> 这里体现了极其精妙的 Trade-off：FORD 牺牲了 Primary 节点崩溃后的“快速原地恢复能力”（因为 Primary 可能未持久化），换取了正常运行时的极高性能。只要保证 $f$ 个备份落盘，数据就不丢。这种“主节点在持久化上偷懒”的策略，非常适合 PM 这种写带宽比网络带宽还小的特殊硬件场景。

**# 核心亮点 (Key Insights & Trade-offs)**

1. **对 RDMA 原子性的极致压榨**：FORD 的设计哲学是**完全旁路远端 CPU**。它通过组合 `CAS`、`READ`、`WRITE` 和 Doorbell Batching，在网卡层面构建了复杂的事务原语（如 Lock-and-Read, Write-and-Flush）。
    
2. **带宽感知的系统设计**：现有系统大多假设内存带宽无限（DRAM 场景），而 FORD 敏锐地捕捉到了 PM 写带宽小于 RNIC 带宽这一倒挂现象，通过 Backup Read 均衡负载，这是针对特定硬件特性的架构级优化。
    
3. **流水线掩盖延迟**：论文还提出了基于协程（Coroutine）的交错执行模型，在单个线程内并发处理多个事务，有效掩盖了 One-sided RDMA 操作固有的 RTT 延迟 18。
    
4. **持久化与性能的博弈**：通过 Selective Flush，FORD 证明了在分布式一致性协议中，并非所有副本都需要同等级别的持久化时效性，利用副本冗余来换取主节点的写性能是一个高收益的决策。