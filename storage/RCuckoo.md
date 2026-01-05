Header:

论文出处：USENIX ATC '25 1。

本文针对完全存算分离架构（Fully Disaggregated Memory）中，RDMA 网络延迟显著高于本地内存访问（$1 \mu s$ vs $50 ns$）2以及现有 RDMA 原子操作在主存上吞吐受限且争用性能极差（低至 3 MOPS）3 的物理限制，提出了一种基于锁机制的 Cuckoo Hashing 键值存储设计。

# 问题

- 存算分离下的架构两难
    
    现有高性能 KVS 系统通常采用双边（Two-sided）RDMA 操作，依赖内存端的 CPU 来管理锁和执行临界区代码，这违背了完全存算分离中“被动内存池”（Passive Memory Server）的初衷，减少了资源池化的收益 4。
    
    而采用单边（One-sided）RDMA 的无锁（Lock-free）系统，为了支持无序访问，通常采用“哈希索引 + 指针”的间接寻址方式（Extent-based），导致读取需要多次 Round Trip（RTT），且在写入密集型负载下性能会显著下降 5555。
    
- **底层硬件开销分析**
    
    - **RDMA 原子操作瓶颈**：在标准 ConnectX-5 网卡上，针对主存（Main Memory）执行 RDMA 原子操作（如 CAS）存在严重的硬件吞吐上限（约 50 MOPS）。更致命的是，当多个客户端争用同一地址时，性能会崩塌至 3 MOPS 6。
        
    - **网络传输延迟放大**：由于缺乏服务器端 CPU 的计算能力，复杂的哈希冲突解决（如 Cuckoo Hashing 的踢出路径）需要在客户端和内存端之间进行多次网络交互。传统的 Cuckoo Hashing 要求两个哈希位置独立随机，这意味着无法利用空间局部性，每次探测都需要独立的 RDMA Read，导致高昂的 RTT 开销 7777。
        
    - **无锁设计的代价**：为了避免锁，现有乐观并发控制方案只能原子更新 8 字节的指针，无法内联存储 Value。这使得即便是读取小对象也必须经历“读取索引 -> 获取指针 -> 读取数据”的多次 RTT 过程 8888。
        

> 我觉得这里对 RDMA 原子操作瓶颈的洞察非常关键。很多人误以为 RDMA 只是快，却忽略了网卡原子单元（Atomic Unit）在处理高并发争用时的排队效应。RCuckoo 正是抓住了“主存原子操作慢”这个痛点，才引出了后续将锁移至 NIC 内存的神来之笔。

# 解决方案 (The Solutions)

## 针对性硬件优化：基于 NIC 内存的锁表设计

为了解决 RDMA 原子操作在主存上的性能瓶颈，RCuckoo 将锁表（Lock Table）直接存储在网卡的片上内存（NIC On-chip Memory）中。

- **机制拆解**：
    
    - **物理位置迁移**：利用 ConnectX 系列网卡提供的约 256 KB 片上内存。测试表明，在片上内存执行原子操作不仅避免了 PCIe Round Trip，还将争用场景下的吞吐量提升了 3 倍 9999。
        
    - **Masked CAS (MCAS)**：利用 RDMA 的 Masked CAS 操作，允许客户端一次性获取多达 64 把锁（如果锁位图连续），极大地减少了锁定阶段的网络交互次数 10101010。
        
    - **虚拟锁（Virtual Locks）**：由于 NIC 内存极小（256 KB），无法为海量 Key 提供一一对应的物理锁。RCuckoo 引入了虚拟锁层，将逻辑锁 $l$ 映射到物理锁位置 $p = l \pmod P$。虽然这引入了额外的伪共享（False Sharing），但使得系统能够支持任意大小的哈希表 11。
        

<img src="placeholder_architecture" alt="Figure 2: RCuckoo's datastructures showing index in Main Memory and Lock Table in NIC Memory" width=100% />

> 这种设计体现了极其精妙的软硬协同（Co-design）思维。通常我们认为片上内存是给缓存或元数据用的，作者却将其转化为高频原子操作的“避风港”。虽然引入了 False Sharing，但权衡之下，消除了 PCIe 延迟和内存控制器的原子操作瓶颈，这是典型的“用空间局限换取延迟红利”。

## 算法核心：增强局部性的相关哈希 (Locality-Enhanced Dependent Hashing)

为了减少网络往返次数（RTT），RCuckoo 修改了传统 Cuckoo Hashing 中两个哈希函数必须独立的假设，提出了一种相关哈希算法。

- **机制拆解**：
    
    - 计算公式：Key 的主位置 $L_1$ 随机选择，但备选位置 $L_2$ 是基于 $L_1$ 偏移计算得出的：
        
        $$L_{2}(K)=L_{1}(K) + (h_{2}(K) \pmod {\lfloor f^{f+\mathcal{Z}(h_{3}(K))}\rfloor}) \pmod T$$
        
        其中 $f$ 是控制距离的局部性参数，$\mathcal{Z}(x)$ 是末尾零的个数。
        
    - **物理意义**：该公式概率性地限制了 $L_1$ 和 $L_2$ 之间的物理距离。这意味着客户端可以用单次 RDMA Read 操作（Covering Read）同时读取两个可能的位置（及其邻域），或者在写入时用单次 MCAS 锁住两个位置 12121212。
        
    - **批处理与内联**：由于局部性的存在，绝大多数 Key 的两个位置都在同一个 RDMA 消息的覆盖范围内。这使得 RCuckoo 可以将 Value 直接内联（Inline）在哈希表中，读取操作通常只需 1 个 RTT，而写入/删除通常仅需 2 个 RTT 13131313。
        

<img src="placeholder_hashing" alt="Figure 4: Distance between hash locations and fill factor" width=100% />

> 这是一个反直觉的设计。哈希表的教科书式设计原则是“尽可能随机以减少冲突”，但在这里，作者为了适应 RDMA 的网络特性，故意引入了“相关性”。这种 Trade-off 非常漂亮：牺牲了一点点最大填充率（Fill Factor 仍可达 90%+ 14），换取了极具决定性的网络 I/O 减少。

## 全分离架构下的故障恢复 (Fully Disaggregated Fault Tolerance)

由于没有服务器端 CPU 来处理客户端崩溃留下的烂摊子（如持有的锁），RCuckoo 设计了一套完全基于客户端的租赁（Lease）恢复机制。

- **机制拆解**：
    
    - **故障检测**：客户端通过超时机制检测锁是否被长期持有。如果发现某行数据的 CRC 在多次尝试间发生变化，则视为误报（False Positive）并重置计时器 15151515。
        
    - **修复租约 (Repair Leases)**：恢复者必须先通过 RDMA CAS 获取一个全局的“修复租约”，获得对特定区域的独占修复权，防止多个客户端并发修复导致数据损坏 16。
        
    - **确定性状态机**：插入操作被分解为确定的状态流转（例如：从“有重复项且 CRC 错误”到“有重复项但 CRC 正确”再到“无重复项”）。恢复者只需读取受影响的行，推断当前处于哪个状态，并将其推向完成态 17171717。
        

> 在完全存算分离且无 Server CPU 的场景下做 Crash Recovery 是极难的。RCuckoo 的做法类似于构建了一个分布式的、基于共享内存的日志状态机。它强迫所有客户端遵守同一套状态变迁规则，从而实现了“无主”环境下的最终一致性。

# 核心亮点 (Key Insights & Trade-offs)

- 锁机制的回归与革新
    
    在 RDMA KVS 领域，"Lock-free" 通常被视为政治正确。RCuckoo 反其道而行之，证明了如果锁足够细粒度且实现得当（利用 NIC 内存），锁带来的收益（支持内联数据、支持 Cuckoo Hashing 的高填充率）远大于其开销。它打破了“锁 = 慢”的刻板印象 18181818。
    
- 空间局部性作为可调参数
    
    通过引入参数 $f$，RCuckoo 将哈希表的“随机性”变成了一个可调参数。这允许系统在“填充率”和“访问延迟（RTT 数量）”之间找到最佳平衡点。实验表明，当 $f=2.3$ 时，既能保持 95% 的填充率，又能保证 68% 的 Key 两个位置在 5 行以内，极好地契合了 RDMA 的批处理特性 19。
    
- 客户端协同计算
    
    RCuckoo 将复杂的 Cuckoo 踢出路径（Cuckoo Path）计算全部下放到客户端。客户端利用本地缓存（Local Cache）进行推测性搜索（BFS），计算出完整的踢出路径后，再一次性通过 RDMA 锁定并执行。这种“本地计算 + 一次性远程提交”的模式，最小化了对远程内存的多次往返依赖 20202020。