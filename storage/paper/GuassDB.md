GaussDB: A Cloud-Native Multi-Primary Database with Compute-Memory-Storage Disaggregation
Header:

论文出处：PVLDB, Vol. 17, No. 12, 2024 1。

本文针对云原生数据库在多主（Multi-Primary）写入场景下，面临的写入吞吐量受限、跨节点冲突检测开销大以及故障恢复慢这三大物理限制与性能瓶颈进行了系统性优化。

# 问题

- **原有多主架构的局限性**
    
    - **Shared-Nothing 架构（如 Spanner, CockroachDB）**：虽然支持多写，但强依赖两阶段提交（2PC）来处理跨分片事务，导致高延迟和低性能 2222。此外，物理分片依赖分片键（Sharding Keys）的选择，面对复杂负载（如 ERP/CRM）时难以找到最优键，且难以适应动态变化的工作负载 3。
        
    - **Shared-Storage 架构（如 Oracle RAC, Db2 PureScale）**：虽然逻辑上共享数据，但传统方案未实现计算-内存-存储的彻底解耦（Disaggregation），导致扩展性（Scale-out/Scale-in）受限 4。
        
    - **现有云原生多主方案（Aurora, Taurus, PolarDB）**：Aurora 依赖乐观并发控制，导致高冲突率和低性能 5；Taurus 采用悲观并发控制维护缓存一致性，带来了高昂的控制开销 6；PolarDB 使用有状态的内存层，导致内存层故障恢复效率低下 7。
        
- **底层开销分析**
    
    - **日志重放的乱序与合并开销**：在多主架构中，不同计算节点对同一页面的更新分布在不同的 Redo Log 流中 888。恢复时，必须按全局顺序重放这些日志。若无特殊机制，需扫描所有节点的日志并排序，且需从存储层频繁读取旧页面，导致巨大的 I/O 和计算开销 9。
        
    - **全局锁与缓存一致性风暴**：为了维护页面所有权，节点间需要频繁通信。如果每次访问都查询全局锁或目录，会产生惊人的网络延迟（Network Round-trip）和锁竞争 10101010。
        
    - **时钟漂移问题**：分布式环境下节点时钟不一致，导致日志序列号（LSN）无法直接用于全局排序，影响恢复的一致性 11111111。
        

# 解决方案 (The Solutions)

## 三层解耦与页面所有权目录 (Compute-Memory-Storage & POD)

GaussDB 提出了一种计算、内存、存储三层解耦的架构 12。核心在于引入了一个无状态（Stateless）的内存层，用于承载页面所有权目录（Page Owner Directory, POD）、全局锁和全局缓冲 13131313。

- **机制拆解**：
    
    - **页面逻辑分区与所有权（Ownership）**：数据库页面被逻辑划分给不同的计算节点。每个计算节点维护一个 Local Buffer 和 Local Lock 14。如果事务涉及的页面由当前节点拥有，则直接本地处理，无需网络交互 15。
        
    - **所有权转移协议**：若节点不拥有页面，需查询内存层中的 POD。POD 记录了页面当前的 Owner Node 16。节点通过 POD 申请页面所有权（对于写操作）或读取权限（对于读操作）17171717。
        
    - **分布式 POD 实现**：为了避免单点瓶颈，POD 本身通过一致性哈希分布在多个内存节点上 18。
        
    - **RDMA 加速**：层间通信支持 RDMA，且对于频繁变动的页面所有权，可利用单边 RDMA 直接更新内存节点状态 19191919。
        

> Researcher's Comment
> 
> 这种设计本质上是把硬件层面的“目录式缓存一致性协议（Directory-based Cache Coherence）”提升到了软件层面实现。通过引入 Memory Layer 作为目录（Directory）的持有者，GaussDB 将昂贵的全局锁竞争转化为了所有权的申请与转移。最精妙的地方在于内存层被设计为“无状态” 20，这意味着 POD 可以随时通过扫描计算节点的状态重建 21，从而避免了内存层成为可靠性的短板。

## 双层恢复机制与 Past-Image 优化 (Two-Tier Recovery)

为了解决多主日志分散导致的恢复难题，GaussDB 设计了内存+存储的双层检查点（Checkpoint）机制，并引入了 Past-Image 技术。

- **机制拆解**：
    
    - **双层检查点**：计算节点定期将脏页刷新到内存层（Memory Checkpoint）和存储层（Storage Checkpoint）22222222。内存检查点更快，恢复时优先使用 23。
        
    - **Past-Image (PI) 技术**：当页面所有权从节点 $N_1$ 转移到 $N_2$ 时，GaussDB 不会立即丢弃 $N_1$ 上的旧版本，而是将其标记为 Past Image 并保留 24。
        
    - **Lamport Clock LSN**：使用 Lamport 时钟同步 LSN，确保跨节点的日志具有偏序关系，解决时钟漂移问题 25252525。
        
    - **并行按需恢复**：恢复时，如果目标页面在某个存活节点的 PI 中存在，或者在内存层有较新的 Checkpoint，则无需重放其他节点的旧日志，直接基于 PI 或 Memory Checkpoint 进行 Redo 26。这极大地减少了需要扫描和重放的日志量。
        

<img src="[https://via.placeholder.com/800x400?text=Architecture+Diagram+Placeholder](https://www.google.com/search?q=https://via.placeholder.com/800x400%3Ftext%3DArchitecture%2BDiagram%2BPlaceholder)" alt="GaussDB Architecture and Recovery Flow" width=100% />

> Researcher's Comment
> 
> Past-Image 的设计是典型的“空间换时间”策略。在传统的 ARIES 恢复算法中，最头疼的就是 Page 不在内存时需要去磁盘捞，或者多主场景下需要去捞别人的 Log。GaussDB 通过强行在内存（或内存层）暂留旧版本（PI），构建了一个逻辑上的“版本链”，使得恢复过程可以“短路”掉大量的日志回放。这对于降低 RTO（Recovery Time Objective）至关重要。

## 智能路由与数据亲和性 (Smart Routing)

为了减少节点间的页面所有权转移（Page Transfer），GaussDB 试图将计算推向数据所在的节点。

- **机制拆解**：
    
    - **轻量级路由模型**：在 JDBC/ODBC 层部署轻量级推理模型（基于 MLP），预测查询可能访问的页面组 27272727。
        
    - **亲和性调度**：计算节点定期更新其拥有的页面向量（Owned Vector）给驱动层。驱动层根据查询的预测访问向量与节点拥有向量的相似度（如余弦相似度），将查询路由到拥有最多相关页面的节点 28。
        

# 核心亮点 (Key Insights & Trade-offs)

- Log-is-Data 的反向权衡：
    
    与 Aurora 等推崇 "Log-is-Data"（即只写日志，由存储层异步生成页面）不同，GaussDB 明确指出在他们的部署环境（华为云内部）中，网络并不是瓶颈 29292929。因此，他们选择将脏页刷入存储层，而不是仅传输日志。这避免了在存储层引入额外的计算资源来回放日志 30，体现了架构设计需严格适配基础设施特性的原则。
    
- 无状态内存层的弹性设计：
    
    Memory Layer 虽然承载了核心的元数据（POD, Global Locks），但它是Stateless的 31。这意味着内存层的扩缩容不需要迁移大量数据，只需重新通过一致性哈希分配 Bucket，必要时可从计算节点状态重建。这种设计使得 GaussDB 能够实现秒级的计算和内存弹性伸缩 32323232。
    
- Redo Log 的独立性与 Undo Segment 的共享性：
    
    为了性能，Redo Log 是每个计算节点独享的（避免写入竞争）33；而 Undo Segment 是全局共享的（为了支持 MVCC 和全局一致性读取）34343434。这种分离设计在保证写入高性能的同时，通过全局 Undo 实现了快照隔离。
    

> Researcher's Comment
> 
> 这篇论文展示了一个非常扎实的工程系统。它没有盲目追求“完全无共享”或“完全共享”，而是在中间地带通过 Disaggregation 寻找平衡。特别是 Past-Image 配合内存检查点的恢复机制，非常精准地解决了多主数据库最痛的“故障恢复慢”的问题。这种针对特定痛点（RTO）的架构级优化，比单纯的算法优化更具工业价值。