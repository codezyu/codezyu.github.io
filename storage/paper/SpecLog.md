Header:

OSDI '25 1。本文针对现代“Durability-First”（优先持久化）共享日志系统（如 Scalog）在追求高吞吐量和弹性伸缩时，因必须等待**全局协调（Global Coordination）而导致的高交付延迟（High Delivery Latency）**问题，提出了一种基于推测执行的低延迟架构。

**# 问题**

- **全序生成的固有延迟（The Latency Wall of Global Ordering）**：
    
    - 现代共享日志（如 Scalog）为了支持弹性伸缩和灵活的数据放置，采用“先持久化，后定序”的策略 2222。
        
    - 这就导致了**交付延迟（Delivery Latency）**过高：分片必须先对记录进行批处理（Batching），然后向定序层（Sequencing Layer）汇报，定序层计算“全局割集”（Global Cut）并达成共识（Paxos），最后分片收到割集后才能为记录分配位置并交付给下游应用 3。
        
    - 只有在全序确定后，下游应用才能开始计算，这导致端到端（E2E）延迟无法降低 4。
        
- **传统方案的刚性缺陷**：
    
    - **Order-First 方案（如 Corfu）**：虽然定序在前，但依赖固定的“位置-分片”映射。任何分片的增减都需要全系统更新映射，导致系统不可用（System-wide unavailability）5555。
        
    - **当前方案的“自由意志”（Free Will）问题**：在 Scalog 中，分片拥有“自由意志”，即每次向定序器汇报时，汇报多少条记录完全取决于该分片当前持久化了多少 6。这种不确定性使得分片无法在不与全局通信的情况下预测其记录的全局位置 7777。
        
- **底层开销分析**：
    
    - **序列化阻塞**：计算（Computation）被强制阻塞在协调（Coordination）之后。即 $Latency_{e2e} = Latency_{coord} + Latency_{compute}$ 8。
        
    - **Batching Delay**：为了摊销 RPC 和 Paxos 的开销，分片必须等待一段时间 $T_{ord}$ 累积一批日志才上报，这直接增加了基础延迟 9。
        
    - **Round-Trip Time (RTT)**：确定顺序至少需要 Shard $\to$ Sequencer $\to$ Shard 的网络往返 10。
        

**# 解决方案 (The Solutions)**

## 1. SpecLog 抽象：计算与协调的流水线化

作者提出了 SpecLog 抽象，其核心思想是**推测性交付（Speculative Delivery）**。

- **机制拆解**：
    
    - **推测执行**：一旦记录在分片内持久化，分片立即根据预测的全局顺序将其交付给下游应用 11111111。
        
    - **重叠执行（Overlapping）**：下游应用在收到推测记录后立即开始计算，此时共享日志的全局协调过程在后台并行运行 12121212。
        
    - **确认与回滚**：当真实的全局顺序确定后，SpecLog 发送 `confirm_spec` 或 `fail_spec` 13。如果预测正确（绝大多数情况），应用只需提交结果；如果失败，则回滚并重新计算 14。
        

> Researcher's Comment:
> 
> 这是一个非常经典的系统设计哲学：利用**乐观并发（Optimistic approach）**来掩盖长延迟操作。本质上，SpecLog 是将分布式系统中的“指令流水线化”了。它赌的是“网络和节点通常是正常的”，从而用极小的回滚风险换取了巨大的 $Latency_{coord}$ 隐藏收益。这对于计算密集型的下游任务（如实时欺诈检测 15）收益尤为明显。

## 2. Fix-Ante Ordering：确定性定序机制

为了让分片能够“准确预测”全局位置，必须消除 Scalog 中分片汇报的随机性。作者引入了 **Fix-Ante ("Praefixum ante") Ordering**。

<img src="[https://via.placeholder.com/800x400?text=Fix-Ante+Ordering+Mechanism](https://www.google.com/search?q=https://via.placeholder.com/800x400%3Ftext%3DFix-Ante%2BOrdering%2BMechanism)" alt="image" width=100% />

- **机制拆解**：
    
    - **预定割集（Predetermined Cuts）**：系统预先定义一系列的全局割集 $P_1, P_2, ...$ 16。
        
    - **配额（Quotas）**：每个预定割集规定了每个分片必须贡献的记录数量（即配额 $q_{ij}$） 17。
        
    - **强制对其（Adherence）**：分片必须严格遵守配额。
        
        - 如果记录不足：填充 **No-Op** 记录（下游不可见）以凑齐配额 18181818。
            
        - 如果记录过多：将多余记录**推迟（Delay）**到下一个周期 19。
            
    - **确定性预测**：因为每个分片都知道其他分片在当前割集 $P_i$ 中一定会贡献 $q_{ik}$ 条记录，所以分片 $j$ 可以本地计算出自己记录的确切全局偏移量：$StartPos = \sum_{prev\_cuts} + \sum_{k<j} q_{ik}$ 20。
        

> Researcher's Comment:
> 
> 这个设计太精妙了！它把 Scalog 的“自由市场”（各分片随意上报，定序器汇总）变成了“计划经济”（定序器下发指标，分片必须完成）。
> 
> 通过引入 No-Op 这种空间换时间的手段（消耗少量存储和带宽），换取了分布式系统中最宝贵的确定性（Determinism）。这使得分片在没有全局交互的情况下，就能拥有全局视野。

## 3. Belfast 系统实现：动态环境下的鲁棒性

Belfast 是 SpecLog 的具体实现，它解决了一个核心矛盾：**静态的配额（Quotas）与动态的负载（Workloads）之间的不匹配**。

- **基于速率的配额（Rate-based Quotas）**：定序层根据分片的实际注入速率动态设定配额，使得分片大部分时间能自然满足配额 21。
    
- **Lag-Fix 机制（处理突发流量）**：
    
    - 当某分片遭遇突发流量（Burst）并频繁上报时，定序器会检测到其他分片“落后”（Lag）22。
        
    - 定序器会指令落后分片立即填充 No-Op 并发送报告，从而快速推进全局割集，避免突发分片的确认延迟增加 23232323。
        
- **推测租约窗口（Speculation Lease Windows）**：
    
    - 为了在更改配额或增删分片时不导致预测失败，Belfast 引入了窗口机制 24。
        
    - 所有的配置变更（Quota change, Shard add/remove）仅在窗口边界生效 25。这保证了在窗口期内，所有分片对“预定割集”的视图是完全一致的，从而实现了**零停机扩缩容且无预测失败** 26。
        

> Researcher's Comment:
> 
> 这里的 Lag-Fix 机制非常有意思。在传统同步系统中，快节点通常要等慢节点（Straggler）。但在 Belfast 中，系统主动“催促”慢节点用假数据（No-Op）来“交作业”，从而不阻塞快节点的进度。这在处理**长尾延迟（Tail Latency）**时是一个非常激进但有效的策略。

**# 核心亮点 (Key Insights & Trade-offs)**

- **一致性与低延迟的解耦**：
    
    - Belfast 依然保证**线性一致性（Linearizability）**，但它巧妙地将“追加确认（Ack）”推迟到真实定序之后，而将“数据处理（Process）”提前到推测阶段 27。Appends 仍然需要等待 Global Cut 才能 Ack 给上游，但下游已经开始跑了。
        
- **几乎完美的预测率**：
    
    - 通过 Fix-Ante Ordering，除非发生整个分片彻底故障（Whole-shard failure）这种极罕见情况，分片的预测几乎总是正确的 28。这意味着回滚（Rollback）逻辑在生产环境中几乎不会被触发。
        
- **性能权衡（Trade-offs）**：
    
    - **Append Latency vs. E2E Latency**：Belfast 为了等待所有分片满足配额，其 Append Latency（向生产者确认的延迟）相比 Scalog 有轻微增加（5.8% overhead）29292929。但换来的是 E2E Latency 降低了 1.6倍 30。
        
    - **资源开销**：为了维持确定性顺序，系统需要填充 No-Op 记录，这浪费了少量的存储和带宽（实验中 $<5\%$）31。