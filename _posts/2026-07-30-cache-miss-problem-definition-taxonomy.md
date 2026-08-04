---
title: "多线程共享数据结构导致 Cache Miss 的问题定义与分类学"
date: 2026-07-30
description: "多处理器 cache miss 分类体系的演进：3C 模型、一致性缺失与问题分类学。"
category: 体系结构
---


## 1. 多处理器 Cache Miss 分类体系的演进

### 1.1 经典单核 3C 模型 (Hill, ISCA 1987)

Mark Hill 在其博士论文工作中提出了经典的 **3C 模型**，将单处理器系统中的 cache miss 分为三类：

| 类型 | 定义 | 消除方式 |
|------|------|----------|
| **Compulsory (Cold)** | 数据第一次被访问时必然发生的 miss | 无法消除（仅可通过预取隐藏延迟） |
| **Capacity** | 工作集超过 cache 容量导致的 miss | 增大 cache 容量 |
| **Conflict** | 多个块映射到同一 cache 组导致的 miss | 增加相联度或改进替换策略 |

**关键论文**：
- Mark D. Hill, "A Case for Direct-Mapped Caches," *IEEE Computer*, 1988
- Mark D. Hill and Alan Jay Smith, "Evaluating Associativity in CPU Caches," *IEEE TC*, 1989

该模型的贡献在于**给出了 cache miss 的可操作分类**：程序员或编译器的优化方向直接对应于某种 miss 类型的消除。

### 1.2 从 3C 到 4C：引入 Coherence Miss

在多处理器环境中，3C 模型不足以解释由于缓存一致性协议引发的 miss。Dubois et al. 在 ISCA 1993 的奠基性工作中提出了 **useless miss** 概念，并系统性地将 coherence miss 引入分类体系。

**Coherence miss**（也称 communication miss）的定义：某个处理器对某 cache line 的访问由于另一个处理器对该行的写入而失效，导致的 miss。

由此形成了扩展的 **4C 模型**：

$$\text{Cache Miss} = \text{Compulsory} + \text{Capacity} + \text{Conflict} + \text{Coherence}$$

**关键论文**：
- Michel Dubois, J. Skeppstedt, L. Ricciulli, K. Ramamurthy, and Per Stenström, "The Detection and Elimination of Useless Misses in Multiprocessors," *ISCA 1993*

### 1.3 Coherence Miss 的进一步分解：True Sharing vs False Sharing

Coherence miss 根据共享行为的本质被进一步分解为两类（此分类后来被 Culler & Singh 的经典教材采纳和标准化）：

**True Sharing Miss**：
- 定义：处理器 P1 访问的字与处理器 P2 写入的字**相同**（或存在数据依赖意义上的真实共享）
- 判定标准：一个线程访问了另一线程最近写入并使其缓存行失效的那个确切数据字
- 本质：程序逻辑要求的通信，是 correctness-necessary

**False Sharing Miss**：
- 定义：处理器 P1 和 P2 访问同一 cache line 中的**不同字**，但由于 coherence 以 line 为粒度维护，导致虚假的失效
- 判定标准：一个线程访问的数据字与导致该行失效的写操作所写入的字不同，且该访问的字未被另一线程修改
- 本质：行粒度过粗导致的"冤枉"通信，是 implementation artifact

**关键教材**：
- David E. Culler and Jaswinder Pal Singh, "Parallel Computer Architecture: A Hardware/Software Approach," Morgan Kaufmann, 1998（第 5-6 章）

### 1.4 Essential vs Useless Misses 的奠基分类 (Dubois et al., ISCA 1993)

Dubois 等人的核心贡献是提出了从**程序语义角度**而非硬件机制角度来划分 cache miss 的框架：

**Essential miss**：
- 定义：用于完成处理器间同步或数据通信所必需的 miss
- 特征：若消除该 miss，程序或出错，或进度无法推进
- 子类型：
  - **同步 miss**：与锁、屏障等同步原语相关
  - **通信 miss**（或真正的数据共享 miss）：用于传递生产者-消费者共享数据

**Useless miss**：
- 定义：并非程序正确性或通信所必需的 miss，消除后不影响功能
- 子类型：
  - **Replacement miss**：类似传统 3C 中的 capacity/conflict miss，可通过增大 cache 消除
  - **False sharing miss**：行粒度假共享
  - **Dead sharing miss**：一行虽在处理器间传输，但传输后该处理器从未访问被传输的特定字

**更细粒度的扩展概念**：

| 概念 | 定义 | 来源/年代 |
|------|------|-----------|
| **Dead sharing** | Coherence 传送了一个 cache line，但接收处理器仅访问了与该次传送无关的字节，或根本未访问该行中"被共享"的字 | Skeppstedt & Dubois (ISCA 1993) 中描述 |
| **Replication miss** | 一行的数据被多个处理器只读访问，但其间的 coherence 维护仍产生不必要的 miss | Dubois et al. 的框架中隐含 |
| **Ping-pong effect** | 一行在两个处理器之间反复来回，但每次只有单方向的有效通信 | Torrellas et al. 的量化研究中命名 |

**判断标准表格**（Dubois 框架）：

| 场景 | 分类 | 可否消除 |
|------|------|----------|
| 第一次访问某数据 | Compulsory / 部分 essential | 无法消除，可预取隐藏 |
| P1 写 x, P2 读 x | Essential (true sharing) | 不可消除，可优化频率 |
| P1 写 x, P2 读 y (x 和 y 同 line) | Useless (false sharing) | 可通过 padding/重构消除 |
| P1 读 x, P2 写 y (同 line, 无依赖) | Useless (false sharing) | 同上 |
| P1 写 x, line 传给 P2, P2 不访问 x | Useless (dead sharing) | 可消除 |

---

## 2. Cache Coherence 协议与 Miss 的因果关系

### 2.1 MESI/MOESI 协议的状态转换与 Miss 类型的关系

**MESI 协议（4 状态）**：

| 状态 | 含义 | 该行在被另一处理器写入时的命运 |
|------|------|-------------------------------|
| **M** (Modified) | 仅本 cache 拥有，已修改，与 memory 不一致 | 需写回 memory，然后 Invalid |
| **E** (Exclusive) | 仅本 cache 拥有，与 memory 一致 | 降级为 Invalid |
| **S** (Shared) | 可能多个 cache 共享，与 memory 一致 | 降级为 Invalid |
| **I** (Invalid) | 不持有该行（或已失效） | — |

**MOESI 协议**增加了 **O 状态**（Owned），允许"脏共享"，减少了从 M 写回时的带宽消耗。

**Invalidation-based 协议对 cache miss 的影响**：
- 每次写操作导致所有共享方的行失效
- 失效率随处理器数增加（属于 coherence miss）
- 即使只有一个读-写对的 true sharing，也可能引发连锁失效

**Update-based / Write-broadcast 协议**：
- 写操作直接推送新值给所有共享方
- 减少后续读的 cache miss（数据已在该行中）
- 代价：网络带宽消耗大，且对于后续不读该行的处理器，更新是浪费
- 在 false sharing 场景下，update-based 协议同样会推送无用的更新

**关键论文**：
- Papamarcos & Patel, "A Low-Overhead Coherence Solution for Multiprocessors with Private Cache Memories," *ISCA 1984*（MESI 原论文）
- Sweazey & Smith, "A Class of Compatible Cache Consistency Protocols and Their Support by the IEEE Futurebus," *ISCA 1986*（MOESI）
- Agarwal, Simoni, Hennessy, Horowitz, "An Evaluation of Directory Schemes for Cache Coherence," *ISCA 1988*（invalidation vs update 的量化比较）

### 2.2 Cache Line Size 与 False Sharing 的张力学

**为什么 false sharing 随 line 增大而加剧**：

假设两个独立的变量 `x` 和 `y` 分别由线程 T1 和 T2 频繁写入。若它们落入同一 cache line 的概率为：

$$P_{\text{false sharing}} \propto \frac{\text{变量大小}}{\text{cache line size}} \times \frac{1}{\text{变量间距离}}$$

当 line size 从 32B 增大到 64B 乃至 256B 时，两个本应独立的变量落入同一行的概率大幅上升。每一次 T1 写入 `x` 都会使 T2 缓存中的该行失效，导致 T2 下一次访问 `y` 时产生 coherence miss——尽管 T2 从未读取也从不关心 `x` 的值。

**为什么不能简单地缩小 line size**：

这是经典的 **spatial locality tradeoff**：

- **缩小 line size（如 16B）**：减少 false sharing，但降低空间局部性利用效率。连续数据的访问（如数组遍历）将产生更多 compulsory miss，因为每次 prefetch 的有效数据量减少。

- **增大 line size（如 128B/256B）**：提升空间局部性收益，但加剧 false sharing。

- **现代趋势**：x86 的主流 line size 稳定在 64B（为综合权衡点），但也有架构探索 sub-blocking 协议——line 在 coherence 层面用大粒度（128B），但目录跟踪和失效传播以子块（32B）为粒度。

**量化数据**：
- 64B line size 下，两个 4B 整型变量被放入同一行的概率在对齐不良的情况下可达 10-20%（4/64 × 变量数量因子）
- 在一些 HPC 基准中，从 64B 扩展到 128B line，false sharing miss 增加了 1.5-3 倍（Dubois et al., 1993）

### 2.3 Communication Ratio：有效通信 vs 传输浪费

**定义**：

$$\text{Communication Ratio} = \frac{\text{在一次 coherence 事务中被接收方实际读取的有效字节数}}{\text{该事务传输的总 cache line 字节数}}$$

**含义**：
- True sharing 场景下，该比例通常与共享粒度相关：若两个线程只共享一个 8B 的统计数据但 coherence 以 64B line 为粒度传输，则 communication ratio = 8/64 = 12.5%。
- False sharing 场景下，有效通信字节数为 0，communication ratio = 0。
- 该指标直接量化了 coherence 机制的通信效率损失。

**延伸概念——消息效率**：Agarwal et al. (ISCA 1988) 在目录协议评估中引入的类似概念，衡量一次 coherence 消息中有多少比例的信息是"有生产力"的（即后续会被接收方使用）。

---

## 3. True Sharing 问题的本质

### 3.1 定义与特征

True sharing miss 是**程序正确性所必需的通信**在 cache 层面的表现形式。当线程 A 写入的共享数据必须被线程 B 读取时，由 coherence 协议引发的 cache miss 属于 true sharing miss。

**关键特征**：
- 不可消除（消除等于程序错误），但可以优化其发生率
- 优化方向：减少共享粒度、批量化通信、延迟合并写操作
- 本质是并行编程模型中"通信"概念的硬件实现

### 3.2 经典案例

**案例 1：共享计数器 / 全局屏障计数器**

```c
// 线程 0..N-1 同时执行
global_counter++;
```
即使仅为 `+=1`，每次线程写入这一共享变量都会导致所有其他线程对应 cache line 的失效，形成 "bouncing line"——一行在多个处理器 cache 间反复迁移。

**案例 2：共享工作队列**

```c
while (!done) {
    task = dequeue(global_queue);  // 写入 head 指针
    process(task);
}
```
每次出队操作都修改队列的 head/tail 指针，引起一行在竞争该队列的所有处理器间的逐行迁移。在大量线程竞争的场景下，该行的 E→I→M→S 状态转换所引发的 coherence 流量可占总 cache miss 的 30% 以上。

**案例 3：Producer-Consumer 环形缓冲区**

生产者写 `buffer[write_ptr]`，消费者读 `buffer[read_ptr]`。如果 `write_ptr` 和 `read_ptr` 以及 `buffer[]` 数据落在同一 cache line 中，造成混合了 true sharing 和 false sharing 的复杂 miss 模式。

### 3.3 量化研究 (Torrellas et al.)

**关键论文**：
- Josep Torrellas, Monica S. Lam, and John L. Hennessy, "False Sharing and Spatial Locality in Multiprocessor Caches," *IEEE TC*, 1994

**核心发现**：
- 在多种并行科学计算和系统负载中，**sharing miss（true + false）占数据 cache miss 的 30-50%**
- 其中 true sharing miss 在典型 SPMD 程序中占 sharing miss 的 40-70%（取决于共享粒度和同步频率）
- 在以细粒度锁为基础的操作系统中，true sharing miss 占比可达 60% 以上

**后续量化**（Hennessy & Patterson, Computer Architecture: A Quantitative Approach, 第 5 版）：
- 32 处理器 OLTP（数据库）负载中，coherence miss 占总数据 cache miss 的 30-60%
- 商业负载的 coherence miss 比率普遍高于 HPC 负载，因为共享数据结构的访问模式更加不规则

### 3.4 优化方法（经典）

| 方法 | 原理 | 适用场景 |
|------|------|----------|
| 本地副本累加后合并 | 延迟全局更新，用本地变量累加结果后再写入共享计数 | 计数器、直方图 |
| Hierarchical synchronization | 树形屏障/树形锁减少单行竞争度 | 大规模并行屏障/归约 |
| 无锁数据结构（lock-free） | 使用 CAS/LL-SC 原子操作避免锁的 cache line bouncing | 队列、栈、哈希表 |
| 通信避免算法 | 重新设计算法减少共享状态访问频率 | 数值并行算法 |

---

## 4. False Sharing 问题的本质

### 4.1 定义与产生条件

**形式化定义**：两个或多个处理器访问同一 cache line 中的不同独立变量，且至少有一个写操作。它导致该 cache line 在处理器间发生 coherence 失效，但失效的原因（被写入的字）与失效的后果（被失效的处理器所要访问的字）之间没有数据依赖关系。

**产生的三个充要条件**：
1. **空间条件**：两个或多个无数据依赖的变量位于同一 cache line
2. **并发条件**：不同处理器对这些变量执行了交错读写（至少一个写）
3. **通信冗余**：coherence 协议以 line 为粒度维护一致性

### 4.2 为什么被称为"最隐蔽的并行性能杀手"

1. **代码逻辑完全正确**：不使用任何数据竞争，编译器优化不会发现问题
2. **性能退化可能达数倍甚至数十倍**：一行在处理器间反复 ping-pong 产生的 coherence 流量可以使得原本计算密集的代码变为延迟密集
3. **对输入和线程数敏感**：在某规模下表现良好，在另一规模下突然退化（陡峭的性能拐点）
4. **难以用 profiler 定位**：常规 profiler 报告 cache miss 地址，但不会自动推断 miss 是否属于 false sharing
5. **对微小的代码修改敏感**：增加一个 padding 字段或改变结构体对齐就可能将性能提高数倍

### 4.3 实证研究：Rosenblum et al. 的发现

**关键论文**：
- Torrellas, Lam, Hennessy, "Measurement and Analysis of Memory Sharing in Shared-Memory Multiprocessors," Stanford CSL-TR-92-1673（早期量化研究）
- 更直接地：Rosenblum, Bugnion, Herrod, Witchel, Gupta, "The Impact of Architectural Trends on Operating System Performance," *SOSP 1995*

**核心发现**：
- 在 OS 内核代码中（基准程序为模拟的 UNIX 内核负载），**一枚包含高竞争锁的 cache line 贡献了 18% 的全部 coherence miss**（对应出处：Rosenblum 等人发现，在锁语句周围的几个相邻字段与锁共享一行，导致持有锁的处理器在释放锁时，与该锁同行的其他数据访问在其他处理器上也产生大量 coherence miss）
- 单行问题可以同时是 false sharing（锁与邻居数据）与 true sharing（锁本身在处理器间流转）的叠加

**其他关键量化**（Torrellas et al., IEEE TC 1994）：
- 在 8 处理器系统中，个别 benchmark 的 false sharing miss 占总 cache miss 的 30%
- 通过 padding 消除 false sharing 后，部分程序的执行时间减少了 20-40%

### 4.4 检测与缓解方法

**编译时检测**：
- 静态分析：通过数据流分析推算出哪些共享变量会落入同一 cache line（需要了解 target 的 cache line size）
- 例如：数组`shared[4]`中，`shared[0]` 和 `shared[1]` 若大小为 4B 则在 64B line 中必然在同一行
- HPCToolkit、CrayPat 等工具在部分场景下可辅助分析

**运行时检测**：
- 采样硬件计数器，记录哪些 cache line address 频繁发生 coherence miss
- 结合线程 ID 和时间戳判断是否为 false sharing pattern
- Intel VTune、AMD uProf 提供此类分析

**缓解方法**：

| 方法 | 原理 | 适用场景 |
|------|------|----------|
| **Padding / Alignment** | 使用 `__attribute__((aligned(64)))` 或 `alignas(64)` 确保变量独占一行 | 频繁访问的独立变量 |
| **Struct reordering** | 将热写字段分散到不同 cache line 中 | 包含多个热字段的结构体 |
| **Thread-local storage + 归约** | 先在线程本地累加，定期归约到共享结构 | 累加型计数器 |
| **Array privatization** | 每个线程分配独立 array chunk，消除行共享 | 循环中的数组访问 |
| **Compiler padding** | 编译器自动插入 padding 字节（需基于 profile 指导） | GCC attribute packed 的反面 |

**已知局限**：
- 手动 padding 脆弱：改变 line size 或架构后 padding 大小不再有效
- 过度 padding 浪费 cache 容量
- 与空间局部性优化互相矛盾

---

## 5. 当前研究前沿

### 5.1 从定性分类走向运行时精确定量与归因

**挑战**：
- 现代软件栈极深（hypervisor → OS → 运行时 → 框架 → 应用），一次 cache miss 的根因可能跨越多层
- 在生产环境中，无法大幅插桩或暂停应用进行离线分析

**代表性工作**：

- **Xu Liu and John Mellor-Crummey**, "A Data-Centric Profiler for Parallel Programs," *SC 2013*（以及后续的 HPCToolkit 相关扩展）
  - 提出数据为心的 profiling 方法，将 cache miss 归因到具体的数据结构而非代码行
  - 结合硬件采样与二进制分析，可识别 false sharing、dead sharing 等模式

- **Google 的 CacheLineConscious 数据类型和运行时**（C++ Abseil 库中部分数据结构的设计哲学体现了对 false sharing 的运行时自动缓解）

- **Dice, Lev, Moir (Oracle Labs)**, "Scalable Statistic Counters," 系列工作
  - 在 JVM 和数据库运行时层面，通过数据布局方案动态消除 false sharing
  - 属于面向编译器和运行时的自动优化前沿

### 5.2 异构计算（CPU+GPU）与新型 Miss 分类需求

**新挑战**：

- CPU-GPU 的 coherence 边界：传统的 4C 模型假设所有处理器共享同一物理地址空间和 coherent domain。但在 CPU+GPU 系统中：
  - GPU 的 L1/L2 可能不与 CPU 的 L1/L2 在同一 coherence domain 内
  - 数据通过 PCIe/NVLink 显式传输时，coherence miss 的概念需要重新定义

- **Unified Memory / Managed Memory**：NVIDIA 的 UVM（Unified Virtual Memory）引入了**page fault 驱动的迁移**，其 miss 本质上是 coarse-grained coherence（以 4KB/2MB 为粒度），与传统的 cache line 粒度 coherence 有根本不同

- **假共享问题的跨设备延伸**：一个 CPU 写的变量与同一内存页中的另一个 GPU 写变量同 page，触发整个 page 的迁移，产生"page-level false sharing"

**代表性工作**：

- Neha Agarwal, David Nellans et al., "Unlocking Bandwidth for GPUs in CC-NUMA Systems," *HPCA 2015*
- Ang Li et al., "Analysis and Optimization of GPU Unified Memory," *SC 2019*（讨论 UVM 下 page-fault-driven coherence 的 miss 特征）
- 多个 GPU 厂商白皮书对 heterogeneous coherence 的内部实现描述

### 5.3 数据中心负载与 Tail Latency 视角的 Cache Miss 分析

**新维度**：

- 传统 cache miss 研究以**平均吞吐量**为优化目标；数据中心负载关注**尾延迟 (P99/P99.9)**
- 共享数据结构的 cache line bouncing 在单次 miss 上的延迟惩罚可能为纳秒级，但在请求链中叠加后，对尾延迟的影响是微秒级
- 内存层级的非均匀性（NUCA——Non-Uniform Cache Access）使得 cache miss 的惩罚依赖于 cache line 在多级 cache 层级中的具体位置

**代表性工作**：

- **Ferdman et al.**, "Clearing the Clouds: A Study of Emerging Scale-out Workloads on Modern Hardware," *ASPLOS 2012*（数据中心负载的微架构特征描述）
- **Lozi et al.** (Google), "The Linux Scheduler: A Decade of Wasted Cores," *EuroSys 2016*（揭示内核数据结构的 false sharing 在大规模数据中心场景下对 CPU 利用率的实际影响）

### 5.4 CXL/CCIX 等新型互连技术对传统分类体系的挑战

**核心变化**：

- **CXL (Compute Express Link)**：基于 PCIe 物理层但提供缓存一致性的互连，使得 CPU、GPU、FPGA、内存扩展器等异构设备进入同一 coherence domain
- CXL.cache 协议允许设备 cache host memory，引入了新的 coherence miss 类别：
  - 跨连接拓扑的 miss 延迟远大于片内（200ns vs ~50ns 或更高）
  - 协议类型（ESM / MHDG / SFS）之间的不同失效策略影响 miss 模式

- **CXL.mem (Type 3 内存扩展器)**：允许通过 CXL 连接远端的"惰性"内存。远端内存的访问失败不是传统意义上的 cache miss，而是分布式共享内存中的**远端访问 miss**

**新分类需求**：
- 传统的 4C 模型不区分远端 miss 和本地 miss（两者都是 coherence miss，但延迟相差一个数量级）
- 需要引入 **Locality-annotated Miss Taxonomy**：将 miss 按拓扑距离分层（L1 → L2 → L3 → local memory → CXL-attached memory → socket-socket → 跨节点）

**代表性工作**：

- CXL 3.0 规范中描述的多层交换机和多 host 之间的 coherence 语义
- SK hynix / Samsung 的 CXL 内存扩展器白皮书中对延迟层级的讨论
- Intel / AMD 技术报告中对 CXL 引入后"近端 vs 远端" cache miss 区分的方法论探索
- 学术界：多篇 Workshop on CXL-Based Systems (HPCA/ISCA 2020s 的 Workshop) 讨论 CXL 下的新 performance model

### 5.5 基于机器学习的 Cache Miss 模式识别与预测

**前沿探索**：

- 深度学习模型从硬件性能计数器序列中识别 false sharing 模式
- 结合 NLP 风格的技术建模"访问序列"以自动分类 miss 为 true/false/dead sharing
- 代表性早期工作：MIT/Google 等组使用 Graph Neural Networks 建模 cache line 访问图

---

## 核心概念关系图（文字版）

```
Cache Miss
├── Compulsory (Cold) — 首次访问，无法消除
├── Capacity — 工作集 > cache 容量
├── Conflict — 相联度限制
└── Coherence — 多处理器一致性
    ├── True Sharing — 必要的通信
    │   └── 优化：减少频率/粒度，不可消除
    ├── False Sharing — 行模粒度假共享
    │   └── 消除：padding, struct reorder, privatization
    └── Dead Sharing — 传输但从未访问
        └── 消除：优化数据布局

新增维度：
├── 拓扑距离注解：LOCAL vs REMOTE (CXL/CCIX)
├── 异构域注解：CPU vs GPU vs Accelerator
└── 时间敏感性：P99 latency impact (data center)
```

---

## 参考文献速查

| 编号 | 论文/著作 | 作者 | 出版源 | 年份 | 贡献 |
|------|----------|------|--------|------|------|
| [1] | A Case for Direct-Mapped Caches | M.D. Hill | IEEE Computer | 1988 | 3C 模型提出 |
| [2] | The Detection and Elimination of Useless Misses in Multiprocessors | Dubois et al. | ISCA | 1993 | Essential vs Useless miss 分类，false sharing 形式化 |
| [3] | Parallel Computer Architecture | Culler & Singh | Morgan Kaufmann | 1998 | 4C 模型标准化，true/false sharing 教学级定义 |
| [4] | False Sharing and Spatial Locality in Multiprocessor Caches | Torrellas, Lam, Hennessy | IEEE TC | 1994 | False sharing 的量化研究，50% sharing miss 论断 |
| [5] | The Impact of Architectural Trends on Operating System Performance | Rosenblum et al. | SOSP | 1995 | 单一 cache line 贡献 18% coherence miss 的发现 |
| [6] | An Evaluation of Directory Schemes for Cache Coherence | Agarwal et al. | ISCA | 1988 | Invalidation vs update 协议的 cache miss 特征比较 |
| [7] | Computer Architecture: A Quantitative Approach (5th/6th Ed.) | Hennessy & Patterson | Morgan Kaufmann | 2012/2017 | 教材级总览，包含 OLTP/数据中心负载 miss 特征 |
| [8] | A Data-Centric Profiler for Parallel Programs | Liu & Mellor-Crummey | SC | 2013 | 运行时 miss 归因到数据结构 |
| [9] | Clearing the Clouds | Ferdman et al. | ASPLOS | 2012 | 数据中心负载的微架构 miss 特征 |
| [10] | The Linux Scheduler: A Decade of Wasted Cores | Lozi et al. | EuroSys | 2016 | 内核 false sharing 在大规模部署中的影响 |

---

*笔记整理日期：2026 年 7 月。当前的划分中，「1.-4.」属于经典工作，已在体系结构教学和工业实践中形成共识；「5.」属于当前研究前沿，正在活跃发展中。*
