Header:

NSDI '20 1。本文主要针对现有共享日志系统（Shared Log）在追求全序（Total Order）时面临的动态重配置导致的不可用性以及单点定序器带来的吞吐量瓶颈问题 2222。

# 问题

- **“先定序后持久化”的架构缺陷**：现有的主流方案（如 Corfu, LogDevice）采用“Order-first”策略，即客户端在写入前必须先从定序器（Sequencer）获取全局唯一的序列号（Log Sequence Number），然后再写入存储层 3333。这种设计使得元数据（序列号映射）先于数据存在，导致在扩缩容（添加或移除分片）时，必须在全系统范围内更新映射关系，造成服务暂时不可用（System-wide outage） 4444。
    
- **定序器的物理瓶颈**：由于所有写入操作都必须经过定序器，定序器成为了系统的单一吞吐量瓶颈。要提升吞吐量通常需要依赖定制硬件（如 FPGA 或可编程交换机），且难以随着存储节点的增加而线性扩展 5555。
    
- **数据局部性与可靠性的权衡（vCorfu 问题）**：为了解决 Corfu 轮询写入导致读取性能差的问题，vCorfu 引入了物化流（Materialized Streams）来聚合数据。但这带来了可靠性降低的代价：如果日志副本和流副本同时发生故障（在高并发大规模场景下概率增加），就会导致数据丢失 6。
    
- **一致性协议的底层开销**：传统方案在处理故障恢复时，往往依赖复杂的分布式提交协议来填补日志中的“空洞”（Holes），这不仅增加了延迟，还限制了系统的写入吞吐量 7。
    

# 解决方案 (The Solutions)

## 1. “先持久化后定序”的反转架构 (Persistence-First Architecture)

Scalog 的核心创新在于彻底颠覆了传统的写入流程。客户端直接将记录写入选定的存储分片（Shard），记录在存储节点内部仅保持 FIFO 顺序并立即在分片内复制，**此时记录尚未分配全局序列号** 8888。

- **机制拆解**：
    
    1. **Append**：客户端向分片内的存储服务器发送 `APPEND` 请求 9。
        
    2. **Replicate**：接收请求的主节点（Primary）通过 FIFO 通道将数据复制到分片内的备份节点（Backups） 10101010。
        
    3. **Ordering**：只有当记录在分片内安全复制（Durable）后，才会参与全局定序。
        

> Researcher's Comment:
> 
> 这种“Persistence-first”的设计本质上解耦了**持久化（Durability）和定序（Ordering）**的依赖关系。传统方案中最头疼的“写了一半挂了导致序列号空洞”的问题在这里直接消失了，因为只有“活着”的数据才有资格被排序。这种设计思路非常像存储系统中的 Log-structured Merge Tree (LSM) 在宏观架构上的投射——先把数据落盘，再通过后台进程（这里是 Ordering Layer）来整理秩序。

## 2. 基于“全局割集”的确定性定序 (Global Cut & Deterministic Ordering)

为了在没有中心化定序器分发序列号的情况下实现全序，Scalog 引入了一个独立的**Ordering Layer**。

<img src="[https://via.placeholder.com/800x400?text=Scalog+Architecture:+Storage+Nodes+reporting+to+Ordering+Layer](https://www.google.com/search?q=https://via.placeholder.com/800x400%3Ftext%3DScalog%2BArchitecture:%2BStorage%2BNodes%2Breporting%2Bto%2BOrdering%2BLayer)" alt="Scalog 的定序层架构：存储节点上报 Local Cuts，聚合层生成 Global Cut" width=100% />

- **机制拆解**：
    
    - **Local Cut 上报**：每个存储节点定期向 Ordering Layer 上报其本地日志的“Local Cut”（一个整数向量，表示该节点已持久化的日志位置） 11111111。
        
    - **Global Cut 计算**：Ordering Layer（由 Paxos 保证容错）收集所有分片的 Local Cut，计算出“Global Cut”。Global Cut 的计算逻辑是对各节点向量取**元素级最小值（element-wise minimum）**，以此确定哪些记录已在分片内完全复制 12121212。
        
    - **确定性排序**：Ordering Layer 将 Global Cut 广播回存储节点。存储节点利用确定的规则（如按分片 ID 字典序）将 Global Cut 覆盖的新记录插入到全局全序流中 13131313。
        

> Researcher's Comment:
> 
> 这里的精妙之处在于将细粒度的“每条记录定序”转化为了粗粒度的“批量元数据定序”。Ordering Layer 处理的流量与实际数据大小无关，只与分片数量和上报频率有关 14。这极大地推高了吞吐量上限，只要 Paxos 和聚合层能扛得住元数据流，系统吞吐量就是 $m \times n$（分片数 $\times$ 单机吞吐）。

## 3. 无缝重配置 (Seamless Reconfiguration)

得益于 Persistence-first 架构，Scalog 实现了在重配置期间的零停机。

- **机制拆解**：
    
    - **添加分片**：新分片只需向 Ordering Layer 注册。一旦注册成功，Ordering Layer 在下一次广播 Global Cut 时包含新分片即可。客户端可以立即向新分片写入，无需全网同步新的配置映射 15。
        
    - **故障处理**：如果某个存储节点失效，Scalog 可以选择将整个分片“Finalize”（定格），并添加新分片。由于定序是后置的，失效分片中未定序的数据由客户端重发到新分片，而已定序的数据依然可读 16161616。
        

> Researcher's Comment:
> 
> 相比于 Corfu 每次变更配置都要“Stop-the-world”来达成一致，Scalog 将配置变更处理为 Ordering Layer 中的一个普通原子事件 17。这种将控制流（Control Plane）和数据流（Data Plane）完全异步化的处理方式，是实现高可用性的关键。

# 核心亮点 (Key Insights & Trade-offs)

- **几乎无限的吞吐扩展性**：Scalog 的吞吐量本质上受到 $Aggregation\ Tree$ 处理元数据能力的限制，而非单一硬件定序器。实验显示在仿真环境下可达到 5200 万写入/秒 18181818。
    
- **一致性与可用性的新平衡**：Scalog 在提供线性一致性（Linearizability）的同时 19，通过移除“预分配序列号”这一强依赖，消除了重配置期间的可用性缺口。
    
- **Trade-off 分析**：
    
    - **延迟（Latency）**：虽然吞吐量极高，但 Scalog 的写入延迟包含 `Client -> Storage -> Ordering -> Storage -> Client` 的多跳往返以及定序层的定期轮询间隔（Interleaving Interval） 20。这使得它不适合超低延迟（如高频交易）场景 21。
        
    - **资源开销**：为了掩盖故障带来的性能抖动，Scalog 推荐使用 $2f+1$ 个节点来容忍 $f$ 个故障（Mask 模式），这比传统的 $f+1$ 模式消耗更多存储资源 22。