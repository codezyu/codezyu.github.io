Header

论文出处：USENIX FAST '25 1111。

本文主要研究了现有基于 RDMA 的分布式锁在高争用场景下面临的性能崩溃问题，指出主要物理瓶颈在于客户端的无限重试耗尽了锁服务器 RNIC 的入站 IOPS 资源 2222。

# 问题

- **原有方案的不适应性**
    
    - **N-to-1 通信模式导致的拥塞**：现有的 RDMA 锁（如 CAS 锁）在获取锁失败时，客户端必须重试。在高争用情况下，所有客户端的流量都涌向锁服务器，形成了 N-to-1 的通信风暴。这种过度的重试流量会导致极高的尾延迟，并显著降低系统的整体 Goodput（有效吞吐量） 3。
        
    - **Backoff 策略的局限性**：虽然传统的指数退避（Backoff）策略被广泛使用，但在 RDMA 环境下，它难以微调，且本质上只是缓解而非解决重试问题。实验表明，即便使用了最先进的 Backoff 方案，服务器接收到的 CAS 请求中仍有 87.1% 是无效的重试流量 4444。
        
- **底层开销分析**
    
    - **RNIC Inbound IOPS 耗尽**：每一次失败的锁获取尝试（Retry）都会消耗一个完整的 RDMA Roundtrip。这些无效请求大量占用了服务器网卡（RNIC）的入站 IOPS 处理能力，直接导致性能下降 5555。
        
    - **传统 MCS 锁在分布式环境的缺陷**：虽然 MCS 锁通过本地自旋（Local Spinning）解决了总线争用问题，但在分布式 RDMA 环境中实现 MCS 面临巨大挑战：
        
        - **CPU 开销**：传统的客户端间通信往往依赖 RPC，这会导致 CPU 阻塞和上下文切换开销 6666。
            
        - **连接不可扩展**：如果在所有客户端之间建立可靠连接（RC QP），会产生 $O(N^2)$ 的连接爆炸问题，不仅占用网卡缓存，建立连接的延迟也高达毫秒级 7777。
            
        - **元数据膨胀**：RDMA Write 操作需要 96-bit 的元数据（64-bit 地址 + 32-bit RKey），将其存储在锁条目（Lock Entry）中以维护队列链表是不切实际的 8。
            

> Researcher's Comment
> 
> 这里的问题分析非常透彻。现有 RDMA 锁设计的最大痛点在于“无谓的硬件资源浪费”。在单机多核环境下，Cache Coherence 协议处理争用已经很昂贵；到了分布式环境，网络往返（RTT）和 PCIe 事务的开销更是让自旋重试（Spin-Retry）变得不可接受。ShiftLock 的核心洞察在于：必须将“向服务器争抢”转变为“客户端间的有序交接”，这在逻辑上回归了 MCS 的初衷，但必须跨越 RDMA 动词（Verbs）语义贫乏和连接管理的硬件鸿沟。

# 解决方案 (The Solutions)

## 基于动态连接（DC）的非阻塞客户端切换机制

ShiftLock 实现了一种分布式 MCS 队列，允许持有锁的客户端直接将锁“移交”（Handover）给下一个等待者，从而消除重试流量。

- **机制拆解**：
    
    - **通信原语选择**：摒弃了 RPC，转而使用双边 RDMA（SEND/RECV）。因为 SEND/RECV 在数据路径上是单边的（无需接收端 CPU 实时介入），且不需要接收端提供内存地址和 RKey，减少了元数据传输 9。
        
    - **动态连接（DC Transport）**：为了解决全连接扩展性问题，ShiftLock 利用了现代 RNIC（如 Mellanox ConnectX 系列）支持的动态连接（Dynamic Connection, DC）。每个客户端只需维护一个 DC Initiator (DCI) 和一个 DC Target (DCT)，DCI 可以动态地向任意 DCT 发送数据，无需显式建立连接 10。
        
    - **路由信息压缩**：为了将路由信息塞入有限的锁条目空间，ShiftLock 引入了 `NodeId`（16-bit）替代庞大的 GID/LID。客户端缓存 `NodeId` 到物理地址的映射，结合 24-bit 的 `DctQpNum`，将路由指针压缩至 40-bit，得以放入 8 字节的原子操作范围内 111111。
        

<img src="placeholder_image_architecture" alt="ShiftLock的客户端间通信架构：使用 DCI/DCT 替代全连接 RC QP" width=100% />

> Researcher's Comment
> 
> 这一步设计非常精妙地利用了硬件特性。利用 DC QP 解决了分布式算法中常见的“拓扑连接爆炸”问题，同时用 SEND/RECV 替代 WRITE 巧妙规避了元数据过大的问题。这是一种典型的“硬件感知算法设计”（Hardware-aware Algorithm Design），如果不是对 Infiniband/RoCE 规范极其熟悉，很难做出这样的取舍。

## 基于扩展原子操作的读写锁语义

ShiftLock 在支持 MCS 队列的同时，实现了无饥饿的读写锁（Reader-Writer Lock），这得益于对 RDMA 扩展原子操作的深度利用。

- **机制拆解**：
    
    - **数据结构布局**：锁条目被设计为 128-bit（16字节），包含 `Epoch` (1b), `RdrCnt` (23b), `TailPtr` (NodeId 16b + DctQpNum 24b), 和 `RelCnt` (64b) 12。
        
    - **扩展原子动词（Extended Atomics）**：标准 RDMA 原子操作不支持对部分位域的修改。ShiftLock 利用 **ExTCAS** (Extended Compare-And-Swap) 和 **ExTFAA** (Extended Fetch-And-Add)，通过掩码（Mask）仅对 `RdrCnt` 或 `TailPtr` 进行操作，而不破坏其他字段。这使得读锁的获取（增加 `RdrCnt`）和写锁的入队（修改 `TailPtr`）可以安全并发进行 13131313。
        
    - **防饥饿机制**：引入 `Epoch` 字段划分锁的生命周期。当写者（Writer）连续持有锁超过阈值（例如 $N=16$）时，会主动翻转 `Epoch` 并通知后续节点，强制将锁的所有权移交给读者（Readers），从而避免读者饥饿 14141414。
        

> Researcher's Comment
> 
> 这是一个非常硬核的底层操作。在 128-bit 的空间内通过位域（Bit-field）操作实现复杂的读写互斥逻辑，完全绕过了服务器 CPU。特别值得注意的是使用 ExTFAA 来处理 RdrCnt，这意味着多个读者可以真正的并发获取锁，而不需要像传统 MCS 那样在链表中串行化，这极大地提升了读密集型负载的性能。

## 容错与恢复机制

针对客户端崩溃导致锁死锁的问题，ShiftLock 采用了一种服务器辅助的恢复策略。

- **机制拆解**：
    
    - **租约（Leasing）与 Era 版本号**：客户端持有锁有最大时间限制 $T_{lease}$。服务器维护一个 64-bit 的 `Era` 计数器 15151515。
        
    - **避免 ABA 问题**：ShiftLock 拒绝使用纯单边的 CAS 恢复（即客户端直接重置锁），因为这会导致严重的 ABA 问题（僵尸客户端复活后破坏状态）。相反，恢复过程通过 RPC 请求服务器介入。服务器增加 `Era` 版本号，并使用 ExTCAS 重置锁状态，同时将 `RelCnt` 增加 huge value ($2^{63}$)，使所有等待中的客户端立刻感知到并重试 161616161616161616。
        

> Researcher's Comment
> 
> 在追求极致性能的 RDMA 系统中引入 RPC 看起来是“倒退”，但在这里是明智的 Trade-off。故障是罕见路径（Rare Path），而正确性是底线。纯 One-sided 恢复虽然快，但在异步网络模型下极难保证线性一致性。将复杂性甩给 CPU（服务器 RPC）来处理罕见的故障，保证了正常路径（Common Path）的绝对纯净和高性能。

# 核心亮点 (Key Insights & Trade-offs)

- **空间换时间的路由优化**：通过在客户端缓存 `NodeId` 映射表，将复杂的 RDMA 寻址信息压缩到 40-bit，使得在单次原子操作中更新队列尾指针成为可能。这是实现单次 RTT 入队的关键 17。
    
- **写者优先（Write-Preferring）的设计哲学**：ShiftLock 默认让写者在队列中连续交接，直到达到阈值。在读写混合负载中，这种策略能更快地清空写者队列，因为写操作通常是系统瓶颈。实验表明这种策略比读优先（如 RMA-RW）具有更好的 Goodput 18181818。
    
- **硬件协同设计**：ShiftLock 的性能增益高度依赖于特定硬件特性（Mellanox 的 DC Transport 和 Extended Atomics）。这是一个典型的 Trade-off：牺牲了通用性（只能在支持这些特性的 NIC 上运行），换取了极致的性能（Goodput 提升高达 3.62倍） 1919。
    
- **确定性的延迟边界**：通过 Handover 机制，ShiftLock 将锁获取的尾延迟降低了 76.6%。在高并发下，CAS 锁的延迟是无界的（受运气影响），而 MCS 类型的锁提供了确定性的 FIFO 顺序，这对 SLA 敏感的存储系统至关重要 20202020。