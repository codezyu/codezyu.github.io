Header

论文出处：2025 USENIX Annual Technical Conference (USENIX ATC '25) 1

一句话总结：针对混合存储设备（Hybrid Storage, 如 PM/NVMe SSD + SATA SSD/HDD）场景，本文指出了LSM-tree架构中“排序操作（Flush/Compaction）”导致的CPU与I/O资源深度耦合问题，并提出通过将索引排序与数据落盘解耦来消除资源碎片和写停顿（Write Stalls）。

**# 问题**

- **原有方案的不适应性 (Resource Usage Dependency)**
    
    - **操作内的资源纠缠 (Intertwined Resource Consumption)**：传统的 LSM-tree 排序操作（Flush 和 Compaction）是“单体式”的。Flush 包含 CPU 密集的序列化（生成 Bloom Filter 和 Index Block）和 I/O 密集的数据写入；Compaction 包含 CPU 密集的归并排序和 I/O 密集的读写 2。
        
    - **操作间的被动依赖 (Interdependencies among operations)**：$L_i$ 层的排序操作导致数据量增加，被动触发 $L_{i+1}$ 层的排序。这种级联触发机制使得系统无法根据当前的资源（CPU 或 I/O）空闲情况主动调度任务 3。
        
    - **混合存储上的瓶颈交替 (Alternating Bottlenecks)**：在混合存储上，快速设备（Fast Devices, FDs）通常受限于 CPU（CPU利用率高，磁盘利用率低），而慢速设备（Slow Devices, SDs）受限于 I/O 带宽。由于操作耦合，系统在不同设备间迁移数据时，性能瓶颈在 CPU 和 I/O 之间剧烈波动，导致资源碎片化和严重的写停顿 4444。
        
- **底层开销分析**
    
    - **CPU 开销**：Flush 过程中的序列化步骤，以及 Compaction 过程中的多路归并排序（Key 比较、拷贝），消耗大量 CPU 周期 5555。
        
    - **I/O 等待**：在慢速设备上，CPU 大部分时间处于 `iowait` 状态，导致计算资源闲置 6。
        
    - **现有调度缺陷**：现有的调度优化（如 ADOC）仅仅是在操作粒度上调整频率或并发度。当系统检测到阻塞并调度更多线程处理快速设备上的排序时，会加剧 CPU 争用；反之则导致 I/O 瓶颈。这种“头痛医头”的调度无法打破单个操作内部的资源绑定属性 7777。
        

**# 解决方案 (The Solutions)**

## 1. 操作解耦：索引与数据分离 (Operation Decoupling)

DecouKV 的核心在于将传统的“排序并落盘”这一原子操作，拆解为 **CPU 密集的索引归并任务** 和 **I/O 密集的数据追加/刷盘任务** 8888。

- **数据管理 (AOFs on FDs)**：在快速设备（FDs）上，数据不再维护有序性，而是直接以追加写文件（Append-only Files, AOFs）的形式存储。利用 NVMe SSD/PM 的高并发和随机读性能，AOF 放弃了排序但保持了可接受的读性能 999999999。
    
- **索引管理 (IndexTable in DRAM)**：索引以 `IndexTable`（基于可归并 Skip List 的结构）形式驻留在 DRAM 中。每个 Entry 存储 Key 和指向 FD 上数据的物理地址（File ID + Offset）10101010。
    
- **任务拆分**：
    
    - **Index Merge (CPU-intensive)**：仅在 DRAM 中合并两个 `IndexTable`，完全不涉及磁盘 I/O。这是一个纯 CPU 任务 111111。
        
    - **Data Append/Flush (I/O-intensive)**：`Data Append` 将数据写入 FD；`Data Flush` 扫描内存中的 IndexTable，从 FD 读取数据并写入慢速设备（SD）上的 SSTable。这是一个纯 I/O 任务 12121212。
        

> Researcher's Comment:
> 
> 这种设计非常巧妙地利用了“指针操作”比“数据搬运”轻量得多的特点。传统的 Compaction 需要把数据读出来、解压、排序、重写，而 DecouKV 的 Index Merge 只是在内存里玩指针游戏（Merging Skip List）。这本质上是将 Compaction 的计算代价（CPU）和数据移动代价（I/O）在时间轴上彻底撕裂开了，让系统有了独立调度两者的自由度。

<img src="" alt="DecouKV Overview showing Separation of IndexTable and AOFs" width=100% />

## 2. 资源感知型任务调度 (Queue-based Task Scheduling)

为了最大化利用混合存储的异构资源，DecouKV 引入了双队列机制来感知系统的实时瓶颈 13131313。

- **Index Merge Queue (IMQ)**：存放等待合并的 `IndexTable`。其长度 $L_{IMQ}$ 反映 **CPU 压力**。
    
- **Data Flush Queue (DFQ)**：存放等待刷入慢速设备的 `IndexTable`。其长度 $L_{DFQ}$ 反映 **I/O 压力**。
    
- **参数自动调优 (Auto-tuning)**：
    
    - **CPU Bound 时** ($Score_{IM}$ 高)：增加 `IMTN` (Index Merge Trigger Num)，减少合并频率，降低 CPU 需求；同时减小 `DFTS` (Data Flush Trigger Size)，加速数据刷盘 14。
        
    - **I/O Bound 时** ($Score_{DF}$ 高)：减小 `IMTN`，增加 `DFTS`，利用空闲 CPU 进行更多索引整理，减少写入 SD 的文件数量，从而减轻 I/O 压力 15。
        

> Researcher's Comment:
> 
> 这是一个典型的背压（Backpressure）调节系统。与 RocksDB 固定的 Write Stall 触发阈值不同，DecouKV 建立了一个动态的反馈回路。当 I/O 拥堵时，它不是简单地让写线程睡觉，而是让写线程去干 CPU 的活（整理索引），反之亦然。这种“以其人之长补其人之短”的策略在异构硬件上极其有效。

## 3. 弹性容量管理 (Elastic Capacity High Levels)

针对慢速设备上的高层级（High Levels, HLs），由于仍然采用 SSTable 结构，DecouKV 放宽了严格的层级放大系数（Level Amplification Factor）限制 16161616。

- **机制**：当 $HL_i$ 层 compaction 可能导致 $HL_{i+1}$ 层 I/O 瓶颈时，允许 $HL_i$ 层容量暂时超限，推迟排序操作。
    
- **代价权衡**：虽然这可能暂时增加读放大（Read Amplification），但由于热数据主要在快速设备上，且写入 SD 的数据已经经过预排序，整体读性能影响微乎其微 17。
    

**# 核心亮点 (Key Insights & Trade-offs)**

- **利用新硬件特性规避 Bloom Filter 开销**：
    
    - 在快速设备（FD）层级，由于采用 AOF 且索引全在内存，DecouKV **移除了 Bloom Filter**。
        
    - **Insight**：传统 LSM-tree 依赖 Bloom Filter 减少读放大，但在 NVMe SSD/PM 上，随机读性能极佳，Bloom Filter 节省的 I/O 不足以抵消其构建和查询带来的 CPU 及内存开销。DecouKV 的设计顺应了这一硬件趋势 18。
        
- **将 Compaction 转化为纯内存操作**：
    
    - **Insight**：传统的 $L_0 \rightarrow L_1$ Compaction 是写停顿的主要来源。DecouKV 将这一步转化为了内存中的 Skip List Merge。虽然这增加了 DRAM 的使用（存储 IndexTable），但消除了 FD 上的大量读写干扰 19。
        
- **Trade-off: 内存消耗 vs. 性能**：
    
    - DecouKV 需要在 DRAM 中维护较大的 `IndexTable`。论文评估显示，IndexTable 占用了约 23% (0.7GB) 的内存预算，高于 RocksDB 的 MemTable (0.3GB)。
        
    - 但是，由于去除了 FD 上的 Bloom Filter 和 Index Block 缓存，DecouKV 的 `TableCache` 占用更少（1.4GB vs RocksDB 的 1.56GB）。总体内存占用与 MatrixKV 和 ADOC 持平，略高于 RocksDB，但换来了 2.3-4.9x 的吞吐量提升 20202020。
        

> Researcher's Comment:
> 
> 这篇论文最精彩的地方在于它不仅仅是做“Key-Value 分离”（如 WiscKey），而是做了“控制流（索引排序）与数据流（数据落盘）的分离”。
> 
> 在混合存储架构中，CPU 和 I/O 的速率差异是巨大的。通过解耦，DecouKV 实际上构建了一个巨大的 CPU 缓冲区（Index Merge）和一个巨大的 I/O 缓冲区（AOF），让两者可以完全异步地运行。这解决了 LSM-tree 在异构设备上长期存在的“木桶效应”——即整体性能总是受限于当前最慢的那个资源（CPU 或 Disk）。