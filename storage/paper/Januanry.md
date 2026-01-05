Header:

论文出处：NSDI 2025。本文指出构建极低延迟（Wire-Latency）存储系统的主要瓶颈在于：现有的 RDMA 单边原语无法处理多客户端间的线性一致性协调，而涉及服务器 CPU 的传统 RPC 路径又引入了不可忽视的软件开销；同时，商用 RNIC 在处理大规模连接和特定队列结构（SRQ）时存在 DMA 效率低和 SRAM 缓存抖动的物理限制 111111111111。

# 问题

- RDMA 单边原语的语义局限性
    
    RDMA WRITE/READ 虽然可以绕过远程 CPU 实现极低延迟（即 Wire-Latency），但它们天然缺乏多客户端协调能力 2222。客户端在发出 RDMA WRITE 时必须指定目的地址，若要保证线性一致性（Linearizability），通常需要引入额外的组件（如 Sequencer）来协调写入位置，这会增加额外的 RTT，从而破坏 Wire-Latency 的定义 3333。因此，目前 Wire-Latency 仅局限于简单的单客户端数据访问或主从复制场景 4444。
    
- Nilext 请求的软件处理开销
    
    对于存储系统中常见的 Nil-externalizing (nilext) 请求（即不返回执行结果或状态的请求，如 set、put），现有 RPC 框架仍需服务器 CPU 参与接收、排队和执行，导致延迟增加 5555。虽然 nilext 请求语义上允许延迟执行，但缺乏一种机制在绕过服务器 CPU 的同时完成“提交”确认 6666。
    
- **底层硬件架构的物理瓶颈**
    
    - **SRQ 的 DMA 效率问题**：为了支持多客户端连接，服务端通常使用共享接收队列（SRQ）。但在 SRQ 中，Work Queue Elements (WQEs) 是通过链表（Linked List）而非数组连接的 7。这迫使 RNIC 在预取 WQE 时必须进行“指针追踪”（Pointer-chasing），导致更多的 DMA 操作和 RNIC 缓存未命中，进而造成网络流停顿 8888。
        
    - **RNIC SRAM 容量限制**：当服务端创建大量 RC（Reliable Connection）QP 连接时，RNIC 有限的 SRAM 无法缓存所有连接状态，导致性能随连接数增加而急剧下降 9999。
        
    - **持久化写的不可见性**：在持久化内存（PM）场景下，客户端收到 RDMA SEND 的硬件 ACK 仅代表数据到达了接收端 RNIC 或进入了 PCIe 域，并不保证数据已写入 PM 介质，存在数据丢失风险 10。
        

# 解决方案 (The Solutions)

## Ordered Queue (OQ) 抽象与硬件 ACK 复用

为了解决多客户端协调与 CPU 旁路之间的矛盾，论文提出了 Ordered Queue (OQ) 抽象。该设计的核心洞察是：远程 CPU 旁路的本质来自于 RNIC 生成硬件 ACK 的能力，这与 RDMA 原语类型无关 1111。只要使用可靠传输（RC 模式），RNIC 就会生成硬件 ACK。

- **机制拆解**：
    
    1. **NIC Rule (Rule 1)**：利用 RNIC 自身的硬件行为，即 RNIC 会按顺序从接收队列（SRQ）头部由分配 Buffer，并在分配后立即返回硬件 ACK 12121212。JUNEBERRY 将此硬件 ACK 重新定义为 nilext 请求的“提交信号” 13。
        
    2. **CPU Rule (Rule 2)**：服务器 CPU 严格按照 SRQ 中 Buffer 的顺序执行请求 14。即使 nilext 请求已经通过硬件 ACK 向客户端确认提交，CPU 也会在稍后异步执行它，从而保证系统整体的线性一致性 15。
        

<img src="figure_1_placeholder" alt="image" width=100% />

> Researcher's Comment:
> 
> 这是一个非常狡猾且聪明的“语义重载”设计。通常我们认为 ACK 只是传输层的可靠性保证（Reliability），但作者将其提升为存储层的持久化/提交保证（Commit Signal）。这种做法巧妙地剥离了“提交”和“执行”：提交由网卡硬件光速完成（Wire-Latency），执行由 CPU 异步完成。这种对现有硬件特性的压榨（Exploiting）体现了极佳的软硬协同思维。

## 针对 RNIC 架构的性能优化

为了克服 SRQ 带来的硬件效率问题，JUNEBERRY 引入了特定的软件技术来适配商用 RNIC（如 Mellanox ConnectX-5）。

- Multi-Packet (MP) SRQ：
    
    针对 SRQ 链表结构导致的 DMA 效率低问题，JUNEBERRY 利用了 RNIC 的 Multi-Packet 特性 16。允许一个较大的接收 Buffer 容纳多个请求（例如 64B 对齐），这样 RNIC 一次 DMA 预取到的 WQE 就可以服务多个后续的 SEND 请求，大幅减少了 PCI 总线上的 DMA 事务数量和 RNIC 内部的 Cache Miss 17171717。
    
- 客户端委托（Client-side Delegation）：
    
    针对 QP 数量过多的问题，采用了线程委托机制。每个客户端机器只建立 $S_t$（Server 线程数）个 QP，而不是 $C_t \times S_t$（客户端线程数 $\times$ Server 线程数）。客户端内部通过共享槽位将请求委托给拥有 QP 的“Owner”线程发送 18181818。
    

<img src="figure_4_placeholder" alt="image" width=100% />

> Researcher's Comment:
> 
> 这一部分揭示了学术界与工业界结合的痛点。理论上 SRQ 是完美的扩展性方案，但物理实现上，链表式的 WQE 居然成为了网卡 DMA 引擎的噩梦。使用 Multi-Packet 其实是一种变相的“批处理”或者说“空间换时间”，通过增加 Buffer 粒度来平摊 DMA 开销，这对于理解现代网卡的微架构瓶颈非常有价值。

## 持久化支持与崩溃一致性

在涉及持久化内存（PM）的场景（JUNEBERRY-D）中，必须保证硬件 ACK 返回时数据确实已持久化。

- PCIe Flush 机制：
    
    为了确保数据不仅到达 RNIC 还写入了 PM，客户端在发送每个 SEND 请求后，立即在该 QP 上发起一个 1 字节的 RDMA READ 19。由于 RNIC 严格按序处理同一 QP 上的指令，READ 操作触发的 PCIe 读事务会强制将之前的 PCIe 写事务（即 SEND 数据）Flush 到 PM 中 20。
    
- DDIO Bypass：
    
    服务器端禁用了 DDIO（Data Direct I/O），迫使 RNIC 直接将数据 DMA 写入内存/PM，而不是写入 CPU 的 L3 Cache。这避免了 Cache Line（64B）与 PM 内部写粒度（如 Optane 的 256B）不匹配导致的写放大问题 21。
    
- 可恢复的 SRQ 扫描：
    
    崩溃恢复时，系统扫描 PM 中的 SRQ Buffer。利用 Checksum 和请求长度字段，识别出所有有效且连续的请求，并重新执行那些可能已返回 ACK 但未被 CPU 执行的 nilext 请求 22222222。
    

> Researcher's Comment:
> 
> 这里的亮点在于利用 RDMA READ 作为一种“远程内存屏障”（Remote Fence）。在 x86 体系结构中我们习惯了 SFENCE 或 CLFLUSH，但在网络侧，利用协议的有序性来强制 PCIe 刷写是一个非常底层的技巧。不过，这也带来了代价：禁用了 DDIO 意味着后续 CPU 读取这些请求时必须从 PM 读，这比从 L3 Cache 读要慢得多（约 3 倍延迟），这是为了正确性不得不做的性能牺牲。

# 核心亮点 (Key Insights & Trade-offs)

- 重新定义“远程 CPU 旁路” (Redefining Remote CPU Bypass)
    
    传统观点认为只有 One-sided Verbs 才能旁路 CPU。本文证明了只要利用 NIC 的硬件 ACK 机制，Two-sided Verbs (SEND/RECV) 对于 nilext 请求同样可以实现“逻辑上”的 CPU 旁路提交 23232323。
    
- 线性一致性的异步实现 (Asynchronous Linearizability)
    
    通过严格定义 NIC 分配顺序（Rule 1）和 CPU 执行顺序（Rule 2），系统在物理上解耦了提交与执行，但在逻辑上保持了严格的线性顺序。即使 $R_1$ 是通过硬件 ACK 提交的，$R_2$ 是通过软件响应的，只要 $R_1$ 在 SRQ 中排在 $R_2$ 前面，CPU 就保证先执行 $R_1$ 24242424。
    
- **性能权衡 (Trade-offs)**
    
    - **读密集型负载的退化**：在 JUNEBERRY-D 中，由于禁用了 DDIO 且 PM 读延迟较高，对于读密集型负载（如 95% get），其中位数延迟比普通 RPC 方案高出约 1.2 倍 25。
        
    - **病态访问模式**：如果客户端在收到 nilext 请求的 ACK 后立即发送 non-nilext 请求，且两者落入同一 SRQ，服务器 CPU 可能来不及“隐藏”前者的执行延迟，导致后者的排队延迟增加 26262626。但论文数据表明这种情况在实际负载（如 Twitter Trace）中非常罕见 27。