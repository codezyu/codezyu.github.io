NSDI '23

**Hydra: Serialization-Free Network Ordering for Strongly Consistent Distributed Applications** 1

Header:

本文针对基于网络排序（Network Ordering）的分布式系统（如 NOPaxos, Eris）存在的单点序列化瓶颈，提出了一种无需序列化的多定序器架构，解决了单一硬件定序器带来的扩展性受限、单点故障恢复慢以及破坏数据中心网络负载均衡策略等物理限制 2222。

---

# 问题 (The Problems)

- **单一序列化点的扩展性瓶颈 (Scalability Bottleneck)**
    
    - 现有的网络排序方案（如 NOPaxos, Eris）要求所有发往复制组的流量必须经过**唯一**的定序器（Sequencer）进行序列化处理 3333。
        
    - **底层开销分析**：虽然可编程交换机（Switch ASIC）吞吐量高，但在分片数据库（如 Eris）场景下，单一交换机需服务所有分片，成为系统瓶颈。若使用端主机（End-host）作为定序器，CPU 处理能力的限制更加明显，难以通过横向扩展来提升定序能力 4。
        
    - **多管线架构不兼容**：现代交换机 ASIC 通常采用多管线（Multiple Pipelines）架构（例如 2-8 条管线），管线间资源不共享。现有的定序逻辑需要原子性地读取和更新全局 Sequence Number，这迫使逻辑只能部署在单条管线上，无法利用整个交换机的吞吐能力 5。
        
- **故障恢复导致的高停机时间 (Prolonged System Downtime)**
    
    - 单一序列化点意味着单点故障。
        
    - **底层开销分析**：定序器故障恢复比传统 Paxos Leader 切换更复杂，因为它耦合了**网络层重路由**和**应用层状态恢复**。网络控制平面更新转发表（Forwarding Tables）通常需要 $>200\text{ms}$ 6。在此期间，整个系统处于不可用状态 7。
        
- **破坏数据中心网络特性 (Worsened Data Center Network Properties)**
    
    - 数据中心网络依赖 ECMP 和多路径冗余来提供高带宽和容错。
        
    - 强制所有特定应用的流量经过单一物理交换机，破坏了路径多样性（Path Diversity），导致该节点的链路拥塞，无法利用现有的负载均衡策略 8。
        

> Researcher's Comment:
> 
> 这是一个非常典型的 "In-Network Computing" 的 Trade-off 问题。之前的 NOPaxos 为了获得极致的 Ordered Multicast 性能，引入了“网络序列化”这个强假设。Hydra 的敏锐之处在于指出了这个强假设在生产环境（Production Networks）中的致命弱点：网络运维人员极其讨厌这种破坏 ECMP 和引入单点故障的“黑盒”。

---

# 解决方案 (The Solutions)

Hydra 的核心贡献是设计了一种**无需物理序列化**的排序原语，允许多个定序器（Sequencers）并发工作，通过松散同步的物理时钟和逻辑序列号相结合来建立全局全序。

## 1. 基于物理时钟与多序列号的混合排序 (Hybrid Ordering Mechanism)

Hydra 部署一组定序器（$M$ 个），每个定序器维护一个单调递增的物理时钟 $c$ 和针对每个接收组（Group）的序列号 $s$ 9。

- **机制拆解**：
    
    - 当 Groupcast 消息到达任意一个定序器时，定序器原子地递增目标组的 $s$，并将当前的物理时钟 $c$ 写入包头 10。
        
    - **偏序关系**：消息 $m_1$ 排在 $m_2$ 之前 ($m_1 \prec m_2$)，当且仅当 $c_1 < c_2$ 或者 ($c_1 = c_2 \wedge i_1 < i_2$)，其中 $i$ 是定序器 ID 11。
        
    - **接收端逻辑**：接收端根据上述规则进行排序。
        

> Researcher's Comment:
> 
> 这个设计非常巧妙地将“定序”这个动作从物理上的单点（Single Register）解耦到了逻辑上的时间戳。虽然物理时钟存在 Skew，但 Hydra 并不依赖时钟的精确同步来保证 Safety（安全性仅依赖时钟单调递增），Clock Skew 仅仅影响 Liveness（消息交付延迟），这是一个极佳的工程权衡。

## 2. 丢包检测与安全交付 (Drop Detection & Safe Delivery)

仅仅依赖时钟无法检测丢包（因为时钟是不连续的）。Hydra 结合了序列号来实现丢包检测。

- **数据结构**：接收端维护每个定序器 $i$ 的最大已接收序列号 $s[i]$ 和最大已接收时钟 $c[i]$ 12。
    
- **交付规则**：
    
    1. 计算全局最小水位线：$c_{min} = \min_{i \in M} (c[i])$ 13。
        
    2. **安全条件**：只有当消息 $m$ 的时间戳 $m.c \le c_{min}$ 时，才交付该消息 14。这保证了不可能再收到来自其他定序器的、时间戳更小的消息。
        
    3. **丢包判定**：对于特定定序器 $i$，如果收到的序列号 $s > s[i] + 1$，说明中间有丢包，触发 `DROP-NOTIFICATION` 15。
        

<img src="..." alt="Hydra Ordering and Drop Detection Logic" width=100% />

> Researcher's Comment:
> 
> 这里的核心难点在于：如何在多发送源的情况下保证“无空洞”。Hydra 的做法类似于 TCP 的滑动窗口，但窗口的推进是由所有定序器的最慢时钟决定的（$c_{min}$）。这意味着系统的长尾延迟（Tail Latency）受限于最慢的那个定序器路径，这是为了获得 Scale-out 能力所付出的代价。

## 3. 活性保障与 Flush 优化 (Liveness & Optimizations)

如果某个定序器长时间没有流量，$c_{min}$ 将无法推进，导致所有消息阻塞。

- **Flush 机制**：定序器周期性发送 `Flush` 消息，包含当前时钟和各组序列号 16。
    
- **接收端请求 (Solicitation)**：为了避免广播风暴，接收端仅在有未交付消息缓存时，才主动向定序器请求 Flush 17。
    
- **网内聚合 (In-Network Aggregation)**：利用 ToR 可编程交换机，聚合来自多个定序器的 Flush 消息，仅当聚合后的 $c_{min}$ 足够大时才发往接收端服务器，大幅减少 CPU 中断开销 18181818。
    

## 4. 快速故障切换 (Fast Failover)

- **解耦路由**：当检测到定序器故障时，Hydra 不等待网络层路由收敛。
    
- **Reconfiguration 协议**：接收端通过 Configuration Service 达成一致，确定从故障定序器接收到的最后一条消息，直接在逻辑层“移除”该定序器 19191919。这使得故障恢复时间仅取决于检测超时，而与网络重路由延迟无关 20。
    

---

# 核心亮点 (Key Insights & Trade-offs)

- 线性扩展性 (Linear Scalability)：
    
    通过移除单点序列化，Hydra 的吞吐量随着定序器数量增加而线性增长。在微基准测试中，使用 8 个定序器可达到 2.5 亿次 groupcast/s 的处理能力 21。
    
    > **Researcher's Comment**: 这证明了对于 Sequencer 这种逻辑，并不一定需要强一致的共享状态。通过将排序决策推迟到 Receiver 端（Reordering Buffer），网络层获得了极大的自由度。
    
- 与 NOPaxos/Eris 的无缝兼容：
    
    Hydra 向应用层提供的原语（Ordered Unreliable Multicast）与之前的 NOPaxos 完全一致，这意味着上层应用（如 HydraPaxos, HydraTxn）几乎不需要修改代码即可获得多定序器的支持 22222222。
    
- 资源效率与复杂度的权衡：
    
    Hydra 的 Switch 实现（P4）非常轻量，仅需 1040 行代码，且利用了并行的 Register 访问（因为不同 Group 之间无依赖），不像传统方案受限于单管线 23232323。
    
    Trade-off：接收端的逻辑变得更复杂了，需要维护 $M$ 个定序器的状态并进行排序缓冲。实验显示虽然增加了一些 CPU 开销，但通过 Flush 聚合优化，吞吐量依然比 Atomic Multicast 高出 378% 24。
    
- 硬件友好的时间同步假设：
    
    Hydra 并不假设完美的时钟同步。即使存在 Clock Skew，也不会破坏 Safety（一致性），只会增加消息在接收端的排队延迟（Latency）。这使得它可以使用 NTP 或 PTP 等现有协议，而不需要昂贵的原子钟硬件 25252525。