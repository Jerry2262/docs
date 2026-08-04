---
title: "多线程共享与 Cache Miss（三）：缓存友好数据布局"
date: 2026-07-30
description: "以 cache line 与缓存层次为一阶约束的数据结构设计：哈希表、字段重排与布局变换。"
category: 体系结构
---


## 1. Cache-Conscious 数据结构

**核心思想**：在设计阶段将 cache line 大小（典型 64B）、cache 层次（L1/L2/L3）、关联度、替换策略等硬件参数作为结构设计的一阶约束，而非事后调优。

---

### 1.1 哈希表类

#### CPHash: Cache-Partitioned Hash Table

- **论文**：CPHash: A Cache-Partitioned Hash Table
- **作者**：Metreveli, Zeldovich, Kaashoek（MIT CSAIL）
- **发表**：PPoPP（ACM SIGPLAN Symposium on Principles and Practice of Parallel Programming）
- **发表年份**：约 2012--2013（PPoPP 相关工作）

**核心设计**：

| 要点 | 说明 |
|---|---|
| 分区策略 | 按 CPU core 数目 N 将哈希表均匀划分为 N 个分区 |
| 消息传递 | 读写操作通过消息通道（message channel）投递到目标核心的分区，而非跨核心共享内存直接访问 |
| 亲和性 | 每个核心主要操作自身分区，消息传递处理跨分区访问 |
| Cache 效果 | 大幅减少远程 cache line invalidation 和 L3 cache miss，因为每个核心的分区数据几乎独占其本地 cache |
| 性能 | 约 **1.6x** 吞吐量提升（相对于共享内存哈希表） |

**为什么有效**：传统共享内存哈希表在高并发下因 cache line bouncing 导致大量 coherence 流量。CPHash 通过分区 + 消息传递，让每个核心的分区数据长期驻留在其本地 L1/L2 cache 中，避免跨核心的 cache line 迁移。

---

#### Hopscotch Hashing（Lock-Free 版）

- **论文**：Lock-Free Hopscotch Hashing
- **作者**：Kelly, R., Pearlmutter, B., Maguire, P.
- **发表年份**：2019
- **会议**：SIAM Symposium on Algorithm Engineering and Experiments (ALENEX) / arXiv

**核心设计**：

| 要点 | 说明 |
|---|---|
| 邻域机制 | 每个 key 在表中有一个"邻域"（neighborhood），由固定大小的位掩码（bitmask）表示，key 只能落在邻域内的某槽中 |
| 探测链约束 | 邻域大小 H 为常数（如 32），使得任何查找最多扫描 H 个连续槽 -- cache line 友好（一个 cache line 可容纳多个槽） |
| 并发控制 | Lock-free CAS 操作逐槽移动 items，确保线性化点清晰 |
| Hash 冲突处理 | 当邻域满时，通过"hopscotch 移动"将 item 逐出到更远槽，再递归处理 |

**为什么 Cache 友好**：固定大小邻域使查找在连续内存区域内完成，最多跨越 1--2 个 cache line。对比传统开放寻址的无限长探测链，cache miss 大幅减少。

---

#### Cache-tries: Concurrent Lock-Free Hash Tries

- **论文**：Cache-tries for Concurrent Lock-Free Hash Tries
- **作者**：Aleksandar Prokopec（EPFL / Oracle Labs）
- **发表**：ACM SIGPLAN Notices / OOPSLA
- **发表年份**：2018

**核心设计**：

| 要点 | 说明 |
|---|---|
| 数据结构 | 基于 hash-array mapped trie (HAMT) 的 lock-free 变体 |
| 期望复杂度 | O(1) 期望查找/插入/删除时间 |
| Cache 友好 | trie 节点大小可控，与 cache line 大小对齐；分支因子可调 |
| 并发 | 全部操作无锁，使用 CAS 替换 trie 节点 |

**关键贡献**：在 HAMT 结构上实现了完全 lock-free 的操作集合，同时保持了 trie 结构天然的 cache 友好性（每次操作只访问路径上的少数节点）。

---

### 1.2 树结构类

#### CSB+-Tree: Cache-Sensitive B+-Tree

- **论文**：Making B+-Trees Cache Conscious in Main Memory
- **作者**：Jun Rao, Kenneth A. Ross（Columbia University）
- **发表**：SIGMOD 2000（完整版）；初版发表于 SIGMOD 1999
- **影响力**：该领域的**奠基之作**，被引超千次

**核心设计**：

| 要点 | 说明 |
|---|---|
| 关键洞察 | 传统 B+-tree 内部节点中，子指针数组使用独立指针（8B 每个），占据大量空间 |
| CSB+-Tree | 将同一节点的所有子节点**连续分配**在同一内存块中，内部节点只存储首子节点指针 + key 数组 |
| 空间节省 | 消除 > 90% 的子指针，每个内部节点可容纳更多 key（分支因子增大） |
| Cache 效果 | 更大分支因子 -> 树高降低 -> 每次查找触及更少 cache line |
| 变体 | csb+tree（使用额外槽位指向兄弟节点，用于范围扫描） |

**量化效果**：在内存驻留场景下，CSB+-Tree 比传统 B+-Tree 快约 30--50%，L2 cache miss 降低约 60%。

**设计示意图**（概念级）：

```
传统 B+-tree 节点：  [ptr0, key0, ptr1, key1, ..., ptrN, keyN, ptr(N+1)]
                     ↑ 每个 ptr 8B，空间利用率低

CSB+-tree 节点：     [key0, key1, ..., keyN] + [child_group_ptr]
                     子节点连续存放： [child0 | child1 | child2 | ...]
                     ↑ 仅首指针 + 偏移访问子节点
```

---

#### Cache-Oblivious B-Tree

- **论文**：Cache-Oblivious B-Trees
- **作者**：Bender, M., Fineman, J., Gilbert, S., Kuszmaul, B.（MIT / Stony Brook / 等）
- **发表**：SIAM Journal on Computing, 2007（journal 版）；STOC / SPAA 会议版更早

**核心思想**：与传统 cache-conscious 设计不同，cache-oblivious 算法**不需要知道** block size (B) 和 cache size (M) 等硬件参数。算法在任意 cache 层次上都渐进最优。

**三种变体**：

| 变体 | 并发模型 | 技术要点 |
|---|---|---|
| Exponential CO B-tree | 基于锁 | 使用指数增长的重平衡策略；支持动态更新 |
| Packed-Memory CO B-tree | 基于锁 | 将动态有序字典（ordered file maintenance）嵌入 B-tree，维护单元间的空隙 |
| Packed-Memory CO B-tree | **Nonblocking（Lock-Free）** | 在 packed-memory 框架中引入 lock-free 同步，使用 CAS 循环移动元素 |

**为什么重要**：在硬件参数未知的异构环境下（不同 CPU 型号、不同 cache 大小），一个单一实现可在所有平台上获得近似最优的 cache 性能。尤其对云计算环境的多样硬件部署有实际价值。

---

#### FAST: Architecture-Conscious B+-Tree

- **论文**：FAST: Fast Architecture-Sensitive Tree Search on Modern CPUs and GPUs
- **作者**：Kim, C., Chhugani, J., Satish, N., Sedlar, E., Nguyen, A., Kaldewey, T., Lee, V., Brandt, S., Dubey, P.（Intel Labs）
- **发表**：FAST 2010（USENIX Conference on File and Storage Technologies）；扩展版在 SIGMOD

**核心设计**：

| 要点 | 说明 |
|---|---|
| 节点对齐 | 内部节点大小**精确 match 一个 cache line**（64B） |
| SIMD 友好 | 节点内的 key 比较使用 SIMD 指令并行执行 |
| 结论 | 在 x86 上有 SSE / AVX 时，节点设计为 fit 一个 cache line 比更大的节点更优 |

**关键教训**：即使在内存中，一次 cache line 访问 + 寄存器内 SIMD 比较比多个 cache line 访问 + 简单比较更快。B+-tree 的 optimal node size 不是更大就好，而是要与 cache line 边界对齐。

---

### 1.3 队列类

#### Cache-Aware Lock-Free Queue

- **论文**：A Cache-Aware Lock-Free Queue
- **作者**：Gidenstam, A., Sundell, H., Tsigas, P.（Chalmers University of Technology）
- **发表**：OPODIS 2010（International Conference on Principles of Distributed Systems）

**核心设计**：

| 要点 | 说明 |
|---|---|
| 混合结构 | **链表 + 数组**的混合。每个链表节点包含一个固定大小的数组（ring buffer） |
| 本地指针 | 各线程持有"本地指针"（指向当前操作的链表节点），避免频繁触碰全局 head/tail 指针 |
| 内存模型 | 在弱内存模型（如 ARM/PowerPC）下**无需额外 fence**，因为算法本身对内存顺序依赖低 |
| Cache 行为 | 数组段大小与 cache line 对齐，enqueue/dequeue 在数组段内批量操作 |

**为什么有效**：纯链表队列每操作都要修改全局指针，引发 cache line bouncing。本地指针 + 数组段的组合将大部分操作限制在 thread-local 或近期访问的 cache line 中。

---

#### M\"obius: Lock-Free Queue for Cache Eviction

- **论文**：Mobius: A Lock-Free Queue for Cache Eviction
- **作者**：Dong, X. et al.
- **发表**：SIGMETRICS 2025
- **场景**：Cache replacement / eviction 策略中的 FIFO 队列实现

**核心设计**：

| 要点 | 说明 |
|---|---|
| 应用场景 | 替代传统 LRU/W-TinyLFU 中的 FIFO 队列，消除 eviction 路径上的锁竞争 |
| 吞吐量 | **1.2x~8.5x** 吞吐量提升（相对有锁队列），取决于并发度和 workload |
| 延迟 | tail latency 显著降低 |

---

### 1.4 其他

#### FLeeC: Lock-Free Application-Level Cache

- **论文**：FLeeC: A Fast Lock-Free Application Cache
- **作者**：Costa, D., et al.
- **发表年份**：2024
- **场景**：Memcached 兼容的 application-level cache

**核心设计**：

| 要点 | 说明 |
|---|---|
| 兼容性 | 与 Memcached 协议完全兼容，可作为 drop-in replacement |
| 淘汰策略 | 使用 **lock-free clock eviction** 嵌入哈希表内部，无需独立 LRU 链表 |
| 性能 | 高竞争场景下 **6x** 吞吐量提升 |
| Cache 友好 | 将 eviction 元数据（访问位）与哈希表 entry 共置，避免额外间接指针访问 |

---

## 2. 结构体布局优化

**核心思想**：不改变算法逻辑，仅调整数据结构在内存中的**物理排布**，使多线程共享访问模式下的 cache 利用率最大化。

---

### 2.1 SoA vs AoS 转换

**定义**：

| 布局 | 内存排布 |
|---|---|
| AoS (Array of Structs) | `{x0,y0,z0}, {x1,y1,z1}, {x2,y2,z2}` -- 每个对象的字段连续存放 |
| SoA (Struct of Arrays) | `x0,x1,x2,... y0,y1,y2,... z0,z1,z2,...` -- 每个字段的所有值连续存放 |

**决策矩阵**：

| 场景 | 推荐 | 原因 |
|---|---|---|
| 每次访问结构体的大多数字段 | **AoS** | 一次 cache line load 命中所有需要的字段 |
| 每次只访问 1--2 个字段（如 x 坐标） | **SoA** | 避免将不需要的字段（y, z）拖入 cache line（false sharing 浪费带宽） |
| SIMD 向量化处理一个字段 | **SoA** | 连续的内存布局使 SIMD load/store 无需 gather/scatter |
| 多线程分别修改不同字段 | **SoA** | 避免 false sharing（见下文） |
| 对象数量极少、字段较少 | **AoS** | 开销不大，代码可读性更高 |
| GPU / massively parallel | **SoA** | coalesced memory access 对 GPU 至关重要 |

**量化案例**：Intel 在 Embree 光线追踪库中使用 SoA 布局存储三角形数据（顶点坐标、法向量分开存储），Ray-triangle intersection 性能比 AoS 提升 2--3x。

---

### 2.2 Hot/Cold Field Splitting（热/冷字段分离）

**核心原理**：将频繁被不同线程修改的字段拆分到不同 cache line，消除 **false sharing**。

**False Sharing 的定义**：

- 两个线程分别修改结构体 S 的字段 A 和字段 B
- 设 A 和 B 落在同一 cache line（64B 内）
- Cache coherence 协议强制互相 invalidate，即使两个线程访问的是**不同的变量**
- 结果：cache line 在两个核心之间反复 bouncing，性能灾难

**Linux 内核中的实例**：

| 结构体 | 优化 |
|---|---|
| `struct page` (mm_types.h) | 将 `flags`, `_refcount`, `_mapcount` 等热字段按访问模式分组。使用 `____cacheline_aligned_in_smp` 注释分隔字段组 |
| `struct sk_buff` (skbuff.h) | 网络栈最热的结构体之一。将 head/data/tail/end 等"数据平面"字段与"控制平面"字段（timestamp, mark, priority 等）分离 |
| `struct task_struct` | 将调度器频繁访问的字段（`se`, `rt`, `dl` 调度类数据）与少访问的字段分离到不同 cache line 区域 |

**技术手段**：

```c
// Linux 内核风格
struct my_struct {
    /* 热区：频繁读写的字段 */
    int hot_counter;
    spinlock_t lock;
    // ...
} ____cacheline_aligned_in_smp;

// C++ 风格
struct alignas(64) HotPart {
    int hot_counter;
};

struct ColdPart {
    char debug_name[48];
    // 低频字段...
};
```

---

### 2.3 Padding 与 Alignment

**技术**：

| 技术 | 说明 | 典型用法 |
|---|---|---|
| `alignas(64)` (C++11) | 强制结构体或字段对齐到 64B 边界 | 确保关键结构体独占一个 cache line |
| `__attribute__((aligned(64)))` (GCC) | 等价的 GCC 扩展 | 同上，C 语言版本 |
| Padding fields | 手动插入 `char _pad[64]` 填充 | 精确控制字段间距 |
| `__cacheline_aligned` (Linux) | 内核宏，保证结构体实例在独立 cache line 上 | per-CPU 变量、锁等 |

**内存开销 tradeoff**：

- 过度 padding 导致内存膨胀（比如 `sizeof(struct)` 从 32B 膨胀到 128B）
- 策略：只对**真正竞争激烈**的热点进行 cache line 隔离
- 工具：`perf c2c` (cache-to-cache) 可以检测 false sharing，指导何时需要 padding

**权衡总结**：

```
空间 ←→ 时间
宁可浪费 64B 内存，不浪费 200 个 CPU 周期的 cache miss
但数千个结构体各浪费 64B 也是可观的 -> 精确 profile，只对热点 padding
```

---

### 2.4 Per-Core / Per-Thread 数据副本

**核心思想**：每个 CPU 核心维护自己的数据副本，仅在必要时合并。从根源上消除跨核心 cache line 共享。

**Linux 内核 Per-CPU 变量机制**：

| 机制 | 说明 |
|---|---|
| `DEFINE_PER_CPU(type, name)` | 声明 per-CPU 变量 |
| `get_cpu_var(name)` | 获取当前 CPU 的副本（同时禁用抢占） |
| `put_cpu_var(name)` | 恢复抢占 |
| `per_cpu(name, cpu)` | 访问指定 CPU 的副本 |
| `this_cpu_inc(name)` | 对当前 CPU 副本做无锁自增 |

**Per-CPU 计数器引入的性能提升**（Linux 2.6 内核，约 2003 年）：

- 统计计数器是内核中最频繁的数据争用点（网络包计数、页面分配计数等）
- 传统方案：`atomic_inc()` -- 每个操作触发一次 bus lock 或 LOCK 前缀 -> cache line bouncing
- 2.6 内核引入 per-CPU 计数器后，N 核系统的计数器写竞争从 O(N) 降至 O(1)
- 读取时需要求和所有 per-CPU 副本（如 `/proc/stat` 读取时），但读取频率远低于写入
- 量化效果：某些 workload 下 `gettimeofday` / 网络统计的系统调用开销降低 30--60%

**其他实例**：

| 系统 | 技术 |
|---|---|
| jemalloc (FreeBSD) | per-thread arena（tcache）：每个线程拥有自己的分配缓存，无锁分配 |
| tcmalloc (Google) | per-thread central cache，合并释放到 central heap |
| Linux slab allocator | per-CPU slab partial list，减少锁竞争 |

---

## 3. Lock-Free + Cache-Friendly 的融合

**核心挑战**：Lock-free 数据结构通常需要额外的元数据（版本号、标记位、dummy nodes、逻辑删除标记），这些元数据往往破坏 cache line 的自然对齐和 locality。将两者融合需要精心设计。

---

### 3.1 Locality-Conscious Lock-Free Linked List

- **论文**：A Locality-Aware Lock-Free Linked List
- **作者**：Braginsky, A., Petrank, E.（Technion -- Israel Institute of Technology）
- **发表**：ICDCN 2011（International Conference on Distributed Computing and Networking）

**核心设计**：

| 要点 | 说明 |
|---|---|
| 内存组织 | 将链表分割为**连续内存 chunk**（cache line 大小或 page 大小），chunk 内节点连续存储 |
| Chunk 管理 | 每个 chunk 有**上下界约束**（填充率），例如 chunk 必须填充 50%--100% |
| 自适应 | 当 chunk 太满 -> **split**；太空 -> **merge**（与相邻 chunk） |
| 并发操作 | split/merge 使用 lock-free freeze 机制（标志位 + CAS）保证线性一致性 |
| 查找性能 | chunk 内顺序扫描利用 cache line 的空间局部性 |

**为什么优于纯 lock-free linked list**：

- Harris (2001) 或 Michael (2004) 的经典 lock-free 链表在遍历时每个节点通常占不同 cache line（内存分配器不保证连续），导致遍历时大量 cache miss
- 此设计将逻辑上的链表映射到物理上连续的内存块，在 chunk 内的遍历几乎无 cache miss

---

### 3.2 Lock-Free Skip Tree / Dense Skip Structures

- **来源**：博士论文（Messeguer 相关，具体作者和年份待查）
- **发表**：部分发表在 DISC / OPODIS 等工作

**核心设计**：

| 要点 | 说明 |
|---|---|
| 基础结构 | 基于 skip list 的概率平衡思想，但将稀疏的 skip list 转换为更密集的树形结构 |
| 关键优化 | 消除 skip list 中的**空节点**（即只有 forward 指针、不含数据的 tower 节点），形成 dense 结构 |
| 效果 | 读密集负载下，当 working set 超出 cache 容量时，比 lock-free skip list 快 **2.3x** |
| 原因 | 密集结构减少指针间接访问次数，提高 cache line 的有效利用率 |

**比较**：

```
Lock-free skip list:
  L2: * -> * -> * -> * -> *    ← 大量空节点，cache 利用率低
  L1: * -> * -> * -> * -> * -> * -> *
  L0: * -> * -> * -> * -> * -> * -> * -> * -> *

Dense skip tree:
  实际数据节点连续排布，高层索引节点内嵌在路径上
  ← 无空节点，cache line 满载有效数据
```

---

### 3.3 Lock-Free Ring Buffer (LFRB) 的演化

Ring buffer（又称 circular buffer）是多生产者-多消费者（MPMC）场景下最广泛使用的 lock-free 结构之一。其 cache 行为直接影响吞吐量。

**演化线**：

| 工作 | 作者 | 年份 | 关键技术 | Cache 行为 |
|---|---|---|---|---|
| **LFRB** | 多个来源（Lamport 1983 起） | 1983-- | 单生产者/单消费者使用独立 head/tail 游标 | head/tail 游标在各自的 cache line 上，无 false sharing |
| **SPSC Queue (FastFlow)** | Aldinucci et al. | 2012 | 利用 cache line padding 隔离 head/tail | 精确控制 head/tail 对齐到不同 cache line |
| **MCRingBuffer** | Lee et al.（UNIST） | 2013（PPoPP） | 多生产者使用 ticket-based 分配槽位，多消费者使用 commit counter | 引入批处理减少全局计数器访问频率 |
| **bounded MPMC queue (Vyukov)** | Dmitry Vyukov | 2010-- | 广泛使用的工业级实现（被 Folly、Rust crossbeam 采用） | 每个槽位有独立的序列号 cell，自然隔离竞争 |

**Cache 设计的核心要点**：

1. **游标隔离**：head 和 tail 游标在不同 cache line，避免生产者与消费者互相 invalidate
2. **槽位对齐**：每个槽位大小对齐到 cache line 或为其约数，避免两个生产者操作相邻槽位时 false sharing
3. **批处理**：生产者/消费者一次获取多个槽位，批量写入/读取，均摊全局游标的 cache miss 开销

**量化对比**（参考 Vyukov vs LFRB）：

| 场景 | 传统 LFRB | Vyukov 优化版 | 提升 |
|---|---|---|---|
| 单生产-单消费（64B 消息） | ~30M ops/s | ~80M ops/s | ~2.7x |
| 多生产-多消费（8 cores） | ~15M ops/s | ~45M ops/s | ~3x |

提升主要来自：在同等并发度下减少了 cache line bouncing 次数。

---

### 3.4 Lock-Free + Cache-Friendly 融合的一般性设计原则

1. **局部性分组**：将逻辑上关联的节点/元素放置在连续内存中（chunk/block/slab）
2. **元数据内嵌**：将并发控制元数据（lock words, version stamps, deletion markers）与数据字段放在同一 cache line，避免额外间接访问
3. **冗余消除**：消除 lock-free 结构中不必要的间接指针（如 skip list 的空 tower 节点、链表 dummy nodes）
4. **访问模式感知**：按线程访问模式划分数据（per-thread / shared），热数据靠拢，冷数据分离

---

## 4. 当前研究前沿

---

### 4.1 持久内存（PMEM）时代的 Cache-Conscious 设计

**背景**：Intel Optane DC Persistent Memory 引入后，CPU 可字节寻址访问持久内存，但物理特性与 DRAM 迥异。

**核心矛盾**：

| 参数 | DRAM + Cache | PMEM |
|---|---|---|
| 访问粒度 | Cache line: **64B** | PMEM 内部访问粒度（XPLine / 256B 是典型值） |
| 读延迟 | ~100ns | ~300ns |
| 写延迟 | ~100ns | ~350ns + 需显式持久化（clwb + sfence） |
| 写带宽 | 80--100 GB/s | 2--6 GB/s（远低于读带宽 6--12 GB/s） |
| 持久化保证 | 无 | 需要 cache line flush（clwb/clflushopt）+ store fence |

**关键挑战**：

1. **粒度失配**：cache line (64B) 与 PMEM 内部管理粒度（256B）不一致，可能导致写放大（修改 8B 却花了 256B 的持久化带宽）
2. **flush 顺序与并发正确性**：lock-free 结构的 CAS 修改需要确保持久化顺序与逻辑顺序一致，否则崩溃恢复后可能读取到不一致的快照
3. **颠簸控制**：PMEM 写比读贵得多，cache-conscious 设计必须最小化写操作触及的 cache line 数量

**代表性工作方向**：

- **PMLock-free 数据结构**（如持久化 CAS、持久化指针），确保 lock-free 算法的线性化点在崩溃后仍可恢复
- **写优化索引**：B-tree 设计偏重写合并、减少向 PMEM 的刷写次数（如 NVTree, FPTree）
- **Cache line flush batching**：批量 flush 多个 cache line，利用 sfence 均摊开销

---

### 4.2 CXL 共享内存下的数据结构

**背景**：Compute Express Link (CXL) 协议使得多主机可以**缓存一致地**共享同一物理内存池。这引入了新的延迟层次。

**新的内存层次**：

| 层级 | 延迟 | 特性 |
|---|---|---|
| 本地 L1/L2 | ~1--10 ns | 各核心独享 |
| 本地 L3 (LLC) | ~15--40 ns | 同 socket 共享 |
| 本地 DRAM | ~100 ns | 同 NUMA node |
| 远端 NUMA DRAM | ~150--180 ns | 跨 socket |
| **CXL 附加内存** | **~170--250 ns** | 通过 PCIe/CXL 链路访问，**新延迟层次** |
| PMEM | ~300--350 ns | 持久化存储 |

**数据结构的新需求**：

1. **距离感知（Distance-Aware）**：数据结构需要知道其数据块位于本地 DRAM 还是远端 CXL 内存，并据此调整访问策略
2. **冷热分层**：将热数据置于本地 DRAM，冷数据迁移到 CXL 内存
3. **非对称访问代价模型**：传统的 uniform memory access 假设不再成立，cache-conscious 算法需要考虑非对称的 fetch 代价

**代表性工作**：仍在早期阶段（2023--2025 间陆续有相关论文出现在 ATC, OSDI, ASPLOS 等会议中），未形成成熟体系。

---

### 4.3 近数据处理（Near-Data Processing, NDP）

**背景**：当数据在内存中，CPU 需要将其搬运到 cache 再处理。NDP 思想是将部分操作**推送到内存端**执行（利用内存控制器或 PIM 芯片上的小型处理器），减少数据搬运。

**与 Cache-Conscious 设计的关系**：

| 维度 | 传统 Cache-Conscious | NDP |
|---|---|---|
| 策略 | 优化数据局部性，使数据在需要时已经在 cache 中 | **消除传输**：在数据所在位置直接计算 |
| 硬件前提 | 通用处理器 + cache 层次 | 内存内置处理单元（如 Samsung HBM-PIM, UPMEM） |
| 适用操作 | 所有操作 | 适合简单、规则、高并行的操作（选择、聚合、哈希 join parts） |
| Cache 的角色 | 核心优化对象 | **重要性降低** -- 如果计算在内存端完成，cache hierarchy 可以绕过 |

**量化示例（UPMEM PIM）**：

- 对于大表扫描 + 过滤操作（`SELECT ... WHERE ...`），PIM 芯片的每个 DPU 对自己管辖的内存 bank 进行过滤，只将匹配行返回给 CPU
- CPU 侧的数据传输量减少 10--100x，cache 压力相应降低
- 联合设计：CPU 侧的 cache-conscious 索引结构 + PIM 侧的粗粒度过滤，形成端到端优化

---

## 5. 总结与设计 Checklist

**多线程共享数据结构设计时，Cache 友好性的逐项检查**：

1. **是否消除了 false sharing？** -- 热字段是否在不同的 cache line？可使用 `perf c2c` 验证
2. **节点大小是否与 cache line 对齐？** -- 常见尺寸：64B 或其整数倍
3. **是否利用了 AoS/SoA 的最佳选择？** -- 按访问模式决定
4. **是否有 per-core 数据的空间？** -- 权衡内存开销 vs 竞争消除
5. **lock-free 的元数据开销是否可控？** -- 是否与数据在同一 cache line 内？是否过多间接指针？
6. **访问模式是否具有空间局部性？** -- 相邻元素是否常被连续访问？
7. **是否考虑未来硬件趋势？** -- PMEM 的写放大？CXL 的距离感知？NDP 的卸载机会？

---

**主要参考文献**（按文中出现顺序）：

- Metreveli, Zeldovich, Kaashoek. "CPHash: A Cache-Partitioned Hash Table." PPoPP.
- Kelly, R., Pearlmutter, B., Maguire, P. "Lock-Free Hopscotch Hashing." 2019.
- Prokopec, A. "Cache-tries for Concurrent Lock-Free Hash Tries." ACM SIGPLAN / OOPSLA 2018.
- Rao, J., Ross, K.A. "Making B+-Trees Cache Conscious in Main Memory." SIGMOD 2000.
- Bender, M., Fineman, J., Gilbert, S., Kuszmaul, B. "Cache-Oblivious B-Trees." SIAM Journal on Computing, 2007.
- Kim, C. et al. "FAST: Fast Architecture-Sensitive Tree Search." FAST 2010 / SIGMOD.
- Gidenstam, A., Sundell, H., Tsigas, P. "A Cache-Aware Lock-Free Queue." OPODIS 2010.
- Dong, X. et al. "Mobius: A Lock-Free Queue for Cache Eviction." SIGMETRICS 2025.
- Costa, D. et al. "FLeeC: A Fast Lock-Free Application Cache." 2024.
- Braginsky, A., Petrank, E. "A Locality-Aware Lock-Free Linked List." ICDCN 2011.
- Vyukov, D. "Lock-Free Bounded MPMC Queue." (工业级实现, 2010).
- Lee et al. "MCRingBuffer." PPoPP 2013.
- Aldinucci et al. "FastFlow SPSC Queue." 2012.

