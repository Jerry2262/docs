---
title: "同步原语的缓存友好性优化"
date: 2026-07-30
description: "锁原语、RCU、Lock-Free 与硬件事务内存的缓存行为分析与优化调研。"
category: 体系结构
---

## 目录
1. [锁原语的 Cache 行为分析](#1-锁原语的-cache-行为分析)
2. [Read-Copy-Update (RCU)](#2-read-copy-update-rcu)
3. [Lock-Free 与 Wait-Free 同步原语的 Cache 行为](#3-lock-free-与-wait-free-同步原语的-cache-行为)
4. [硬件事务内存（HTM）与 Cache 交互](#4-硬件事务内存htm与-cache-交互)
5. [当前研究前沿](#5-当前研究前沿)

---

## 引言

现代多核/多插槽系统中，同步原语的性能瓶颈已从"锁竞争导致的 CPU 浪费"转变为 **cache coherence 流量导致的存储墙问题**。一次远程 cache line 传输在 NUMA 系统上可能需要 100–300 个周期，而本地 L1 cache 访问仅需 3–4 个周期。同步原语的"缓存友好性"——即最小化跨核 coherence 流量、最大化本地 cache 命中——已成为决定多核可扩展性的核心因素。

本笔记系统性地调查四类主流同步原语（锁、无锁、RCU、HTM）在 cache 友好性方面的设计演进与核心洞察。

---

## 1. 锁原语的 Cache 行为分析

### 1.1 朴素自旋锁（TAS Lock）的 Cache Bouncing 问题

#### 机制分析

最朴素的 TAS（Test-and-Set）自旋锁操作如下：

```
acquire: while (test_and_set(&lock->flag)) ; // 循环原子写
release: lock->flag = 0;                     // 简单写
```

#### Coherence 流量量化

**Mellor-Crummey & Scott, "Algorithms for Scalable Synchronization on Shared-Memory Multiprocessors", ACM TOCS 1991**

- 每个等待线程在自旋循环内对同一个地址执行 `test_and_set`（原子 RMW 操作，必然获取 cache line 的 Exclusive 状态）
- 当持有者释放锁时（写入 `flag = 0`），该 cache line 被 invalidated，所有 N 个竞争者的 cache line 副本全部失效
- 每个竞争者随后依次获取 Exclusive 所有权，尝试 CAS 抢占，产生 **O(N) 次 cache line 传输**
- 在 N 个核竞争一把锁的稳态下，每次 lock release 到下一个 acquire 完成之间，cache line 在总线上至少传输 **O(N) 次**
- 即使在无竞争时，也至少需要 2 次 cache line 传输（持有者写 invalidation + 获取者读/写）

#### Cache Line Ping-Pong 示意图

```
Core 0 (holder) 释放 → invalidates Core 1,2,3 的 copies
Core 1 (acquires) → invalidates Core 0,2,3 的 copies
Core 2 (waits, spins) → issues RFO, invalidates Core 1's copy
... 在 N 个核之间弹跳
```

#### 关键结论

- TAS 锁的 coherence 流量与竞争者数量 **线性正比**（O(N)），使其在核数超过 8–16 时急剧退化
- 即使是 `test-and-test-and-set` (TATAS) 优化（先读后 CAS），仅在低竞争时有所改善，高竞争下退化行为相同

### 1.2 Ticket Lock 的改进与局限

#### 设计

**Linux 内核长期使用的默认自旋锁（x86 上被 qspinlock 取代）**：

```
acquire: my_ticket = fetch_and_add(&lock->next_ticket, 1);
         while (lock->now_serving != my_ticket) ; // 自旋读
release: lock->now_serving++;
```

#### 改进点

- 将"争抢模式"变为 **FIFO 排队模式**：每个线程获得唯一的 ticket 号，等待轮到自己的号被服务
- 公平性得到保证：不会出现某个线程永远抢不到锁（starvation 消除）

#### 局限

- 所有等待者自旋读取 **同一个 `now_serving` 字段**，仍然位于同一条 cache line 上
- 持有者释放时写入 `now_serving++`，invalidate 所有等待者的 cache line，然后所有等待者再次发起读请求
- 虽然 TAS 的原子写操作被降为普通读，但 coherence 流量仍为 **O(N)**（所有 N 个等待者需要重新读取同一 cache line）
- 在 4+ 插槽的 NUMA 系统中，跨插槽的 cache line bouncing 严重限制扩展性

### 1.3 MCS Lock（经典队列锁）

**Mellor-Crummey & Scott, "Algorithms for Scalable Synchronization on Shared-Memory Multiprocessors", ACM TOCS, Vol. 9, No. 1, pp. 21-65, February 1991**

**2006 年 Edsger Dijkstra 分布式计算奖** —— "probably the most influential practical mutual exclusion algorithm ever"

#### 核心设计

```
type qnode = record
    next   : ^qnode          // 指向后继节点
    locked : Boolean         // 本地自旋标志
type lock = ^qnode           // 指向队尾（初始 nil）

acquire_lock(L, I):
    I->next := nil
    predecessor := fetch_and_store(L, I)    // 入队
    if predecessor != nil                   // 如果队列非空
        I->locked := true
        predecessor->next := I               // 链接前驱
        repeat while I->locked               // 在自己的节点上自旋

release_lock(L, I):
    if I->next = nil
        if compare_and_swap(L, I, nil)       // 队列为空，尝试重置
            return
        repeat while I->next = nil           // 等待后继链接（罕见竞争）
    I->next->locked := false                 // 唤醒后继
```

#### Cache 友好性分析

| 维度 | TAS Lock | Ticket Lock | MCS Lock |
|------|----------|-------------|----------|
| **自旋位置** | 全部在同一 cache line | 全部在同一 cache line | 每个人在自己的 cache line |
| **释放时的 invalidation** | O(N) | O(N) | **O(1)** — 仅 invalidate 直接后继 |
| **公平性** | 无 | FIFO | FIFO |
| **跨 NUMA 自旋** | 取决于持有者 | 取决于持有者 | 自旋总是在本地 |

- **释放锁时只 invalidate 直接后继的 cache line**，将 O(N) 的 coherence 流量降低到 O(1)
- 每个等待者在自己的 `qnode->locked` 上自旋，该节点由线程自己分配（通常在本地 cache 中）
- 空间开销：O(p + n)，p 个进程，n 把锁（相比 ticket lock 的 O(1) 有所增加）

#### 内存开销

- 每锁一个指针（8 字节）
- 每线程一个 `qnode` 结构（在典型实现中约 16–32 字节，含 `next` 指针和 `locked` 标志）
- 代码复杂度高于 ticket lock，但解除了可扩展性瓶颈

### 1.4 CLH Lock

**Craig, Landin, and Hagersten, 1993**

#### 与 MCS 的关键差异

| 特性 | MCS Lock | CLH Lock |
|------|----------|----------|
| **自旋位置** | 在自己的 node 上 | 在 **前驱 node** 上 |
| **链表方向** | 前驱指向后继 (`next` 指针) | 隐式链表（仅指向前驱） |
| **锁传递方式** | 前驱写入后继的 `locked` | 前驱写入**自己的** `locked`，后继自旋观察到 |

#### NUMA 下的行为差异

- **SMP 系统**：CLH 优——锁传递所需远程访问少一次
- **NUMA 系统**：MCS 优——线程自旋**自己的**本地 node（始终 local），而 CLH 线程自旋**前驱的** node（可能位于不同 NUMA 节点）
- CLH 的空间开销略小于 MCS（不需要显式 `next` 指针）
- Java AQS（AbstractQueuedSynchronizer）采用 CLH 变体

#### 重要观察：FIFO vs 数据局部性

- Radovic & Hagersten (2002) 指出：CLH 和 MCS 的严格 FIFO 特性在 **NUCA**（Non-Uniform Cache Architecture）上反而**降低了数据局部性**
- 相比简单的 TAS 锁（持有者倾向于在同一插槽内传递），FIFO 队列锁可能强制跨插槽传递，破坏因数据局部性带来的自然亲和性
- 这直接催生了层次化锁的设计动机

### 1.5 Linux 内核中的演化

#### Ticket Lock → MCS → qspinlock

**Waiman Long (HP/HPE), Peter Zijlstra, 2013–2015**

时间线：
1. **x86 上 ticket lock 是默认锁** → 多插槽 NUMA 系统下扩展性差
2. **引入 paravirt ticketlock** → 为虚拟化场景添加 paravirt hook（vCPU 取消调度等待）
3. **qspinlock（Waiman Long, v1–v16 patch series, 2013–2015）** → 基于 MCS 的 **4 字节队列自旋锁**（与 ticket lock 相同的内存占用量）

#### qspinlock 设计要点

- 将 MCS 队列编码进 **4 字节**的原子变量（通过位域）：`locked` 字节 + `pending` 位 + `tail` 索引
- **fast path**：直接在 `locked` 字节上 CAS，成功即获得（无竞争时 0 额外开销）
- **slow path**：入队进入 MCS 队列，使用 per-CPU 的 `mcs_nodes[]` 数组
- 与 ticket lock 相同的内存占用量，不增加内核关键数据结构（如 `struct page`、`struct inode`）的大小

#### 层次化 qspinlock：CNA 与 HQ-Spinlock

**CNA（Compact NUMA-Aware Lock），Alex Kogan (Oracle), 2019–2021**
- 在 qspinlock 的 MCS 队列基础上，将等待者分为 **主队列**（与锁持有者同 NUMA 节点）和 **次队列**（其他节点）
- 锁在释放时倾向于在同节点内传递，减少跨节点 cache line 传输
- 通过 `CONFIG_NUMA_AWARE_SPINLOCKS` 和 `numa_spinlock=` 内核参数控制

**HQ-Spinlock（Hierarchical Queued NUMA-aware Spinlock），Anatoly Stepanov & Nikita Fedorov (Huawei), RFC Nov 2025**
- 借鉴 Dave Dice 的 **Cohort Lock** 思想，每个 NUMA 节点拥有独立的 FIFO 队列
- **两级传递**：先在节点内局部传递（intra-node handoff），仅在必要时跨节点传递
- 支持 **动态模式切换**：默认使用普通 qspinlock，仅在跨节点竞争检测到时切换到 NUMA-aware 模式
- 通过 `spin_lock_init_hq()` 按锁粒度决定是否启用

---

## 2. Read-Copy-Update (RCU)

### 2.1 核心思想

**Paul E. McKenney, Andrea Arcangeli, Dipankar Sarma et al., Ottawa Linux Symposium 2002**
**McKenney, "Exploiting Deferred Destruction: An Analysis of Read-Copy-Update Techniques in Operating System Kernels", PhD Dissertation, OGI School of Science & Engineering, 2004**

RCU 是专门为"读多写少"场景设计的同步机制，其核心洞察：

> **读端完全无锁、无共享写入、无原子操作，实现"零 cache coherence 开销"的读路径**

#### 关键机制

1. **Quiescent State（静止态）**：线程不在 RCU 读临界区内执行的时刻（如上下文切换、进出内核态、idle）
2. **Grace Period（宽限期）**：从写者完成指针更新到所有 CPU 都经过至少一个 quiescent state 的时间段
3. **Deferred Destruction**：写者不直接释放旧数据，而是等到 grace period 结束后通过 `call_rcu()` 回调释放

#### Cache 友好性的本质

- 读端代码（`rcu_read_lock()` / `rcu_read_unlock()` / `rcu_dereference()`）在非抢占内核中**编译为空操作**（x86 上零指令）
- 读操作不涉及任何共享变量的写入，所有 coherence 事件由**写者引发**（限于指针更新 + 回收）
- 读者所做的 cache miss 是 **capacity/compulsory miss**（数据本身不命中），而非 **coherence miss**（因其他核写入导致 invalidation）
- 将读路径从"coherence miss 主导"转变为"访问模式主导"

### 2.2 Linux 内核中的经典应用与量化数据

| 内核子系统 | 性能提升 | 测试负载 | 硬件配置 |
|-----------|---------|---------|---------|
| **FD 管理** | **>30%** 吞吐量提升 | Chat benchmark | 4 路 PIII Xeon |
| **dcache（目录项缓存）** | **最高 26%** | SDET 多用户 benchmark | 16 CPU NUMA-Q |
| **dcache** | **12%** | SPECweb99 (2258→2530) | 8 CPU PIII Xeon |
| **内核构建** | **>10%** 系统时间减少 | kernel build (47.5→42.5s) | 16 CPU NUMA-Q |
| **System V IPC** | **>1 个数量级** | 多种 IPC 负载 | — |
| **IP 路由缓存** | 显著降低 | 包发送负载 | 8 CPU Xeon |

#### 关键发现

- RCU 在**单处理器系统上没有性能退化**（匹配或略优于非 RCU）
- 在多核上，rwlock 保护的读路径随核数增长退化严重（POWER5+ 上 64 核比 8 核 **慢 10 倍**），RCU 读路径 **近乎理想的线性扩展**
- 性能提升的根本来源：将 coherence miss 替换为 capacity/compulsory miss

### 2.3 URCU（用户态 RCU）

**Mathieu Desnoyers (EfficiOS), LGPLv2.1 许可**

#### 核心特性

- 将内核 RCU 机制移植到用户态
- 提供多种 flavor：`urcu`（需 `sys_membarrier`）、`urcu-qsbr`（Quiescent State Based Reclamation）、`urcu-bp`（bulletproof，无需显式 quiescent state）、`urcu-signal`（基于信号）
- 配套库 `liburcu-cds` 提供基于 RCU 的并发数据结构（哈希表、队列、栈、链表）

#### `sys_membarrier` 的关键作用

- Mathieu Desnoyers 向 Linux 内核贡献了 `membarrier` 系统调用
- 将内存屏障的开销从**读端（每个操作）**转移到**写端（每个 grace period）**
- 基准测试：`sys_membarrier` 方案在 6 读/2 写的配置下 **10 秒内完成 ~99.5 亿次读取**，而写者开销从数千次操作降低到数百次

#### 应用场景

- **DPDK**：高吞吐量网络数据面
- **LTTng**：内核/用户态追踪（Desnoyers 自用场景）
- **Knot DNS**、**Sheepdog** 分布式存储、**GlusterFS**、金融交易软件

### 2.4 RCU 的局限

| 局限 | 描述 |
|------|------|
| **写端开销** | 写端仍需同步等待 grace period，`synchronize_rcu()` 在无加速时为毫秒级 |
| **不适合写密集** | 频繁的更新导致频繁的 grace period 等待，累积累积开销远超简单锁 |
| **Grace period 延迟 vs 吞吐量 tradeoff** | 缩短 grace period 提高响应性但增加开销；延长 grace period 提升写吞吐量但增加内存占用 |
| **内存占用** | 旧版本数据在 grace period 结束前不能释放，写密集型场景下内存压力大 |
| **编程复杂性** | 需要理解 quiescent state、grace period 语义，正确放置 `rcu_dereference` 和 `rcu_assign_pointer` |

### 2.5 Hazard Pointers vs Epoch-Based Reclamation (EBR)

**背景**：RCU 是内核方案；在纯粹的用户态 lock-free 编程中，两大主流内存回收方案是 Hazard Pointers 和 EBR。

#### Hazard Pointers (Maged Michael, 2004)

- 每个线程发布（publish）它当前正在访问的指针到全局可见的 "hazard pointer" 数组中
- 释放内存前，检查是否有任何线程的 hazard pointer 指向待释放的内存
- **Cache 行为**：每次读取共享指针后需要**写**（publish）到全局数组，随后 **内存屏障 + 重验证**（防止 ABA）
- 重验证中需要检查其他所有线程的 hazard pointer 列表 → **O(H) 次 cache miss**（H = 活跃 hazard pointer 数）
- **优势**：有界内存（最多 k+R 个未回收对象）、容错（崩溃线程不阻塞回收）
- **劣势**：每次指针读取后的 fence 开销大

#### Epoch-Based Reclamation (Keir Fraser, 2004)

- 维护全局 epoch 计数器，线程声明自己处于某 epoch
- 删除操作时，将待回收对象放入当前 epoch 的 limb bag
- 只有当所有线程都离开某 epoch 后，该 epoch 的 limb bag 才能安全回收
- **Cache 行为**：读操作只需设置线程局部标志（local write），但需要 O(T) 次扫描检查所有其他线程的 epoch 状态
- [Hart et al., "Making lockless synchronization fast: performance implications of memory reclamation", IPDPS 2016] 提出的 **NER (New Epoch-based Reclamation)** 将临界区扩大到多个操作，**摊销 fence 开销**
- **Brown, "DEBRA/DEBRA+", ACM PPoPP-like**: DEBRA+（容错 EBR 变体）在性能上**平均优于高度优化的 HP 实现 75%**

#### 混合方案：Hazard Eras

- **Ramalhete & Correia, "Hazard Eras", 类似 "Hazard Eras--Non-Blocking Memory Reclamation" (PODC/PPoPP 相关工作)**
- 结合 EBR 的低同步开销和 HP 的有界内存性质
- 读端快速路径仅执行两个 acquire load（无 store），显著优于 HP 的 fence-heavy 协议
- 微基准测试中**比 HP 快 5 倍**

#### 方案对比

| 属性 | HP | EBR (classical) | DEBRA+ | Hazard Eras |
|------|-----|------|--------|-------------|
| **吞吐量** | 较慢 | 最快 | 远超 HP | 接近 EBR |
| **Cache 效率** | 中等（O(H) 扫描） | 通过小内存占用改善局部性 | NUMA-aware private limb bag | 快速路径无 store |
| **有界内存** | 是 | 否 | 是 | 是 |
| **容错** | 是 | 否 | 是 | 是 |
| **每对象开销** | 1 word | 1 word | 同 EBR | 3 words |

---

## 3. Lock-Free 与 Wait-Free 同步原语的 Cache 行为

### 3.1 CAS 重试循环的 Cache Bouncing 退化

#### 问题机制

```
lock_free_push(Q, x):
    do {
        old_head = Q->head;               // 读
        x->next = old_head;
    } while (!CAS(&Q->head, old_head, x)); // CAS 重试
```

- 多个线程同时竞争 CAS 同一个地址 → 每次 CAS（无论成败）都需要该地址的 Exclusive cache line
- 失败者立即重试，再次请求 Exclusive → cache line 在所有竞争者的 cache 之间弹跳（bouncing）
- 在高竞争时，CAS 重试循环的 cache 行为退化与 **TAS 自旋锁相同**（O(N) coherence 流量）

#### 量化差异

- CAS 指令本身**总是执行一个写操作**（即使 CAS 失败，某些实现也写入），相比 TATAS 自旋锁的纯读自旋可能更差
- `cmpxchg` 在 x86 上总是获取 cache line 的 Exclusive 状态（因语义为 RMW），而 TATAS 在自旋时可以保持在 Shared 状态
- 但 CAS 循环**不需要"释放"操作的反向通信**（成功即完成），锁需要 release→acquire 往返

### 3.2 Michael & Scott Lock-Free Queue 的 Cache 行为

**Michael & Scott, "Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms", PODC 1996**

#### 算法结构

- `head` 和 `tail` 两个指针，通过 cache line padding 分离
- enqueue：CAS `tail->next`，然后帮助推进 `tail`
- dequeue：CAS `head`

#### Cache 行为详细分析

| 操作 | 竞争点 | 导致 bounce 的场景 |
|------|--------|-------------------|
| enqueue (多生产者) | `tail->next` | 所有生产者竞争同一 cache line |
| enqueue (帮助机制) | `tail` 指针 | 生产者 + 消费者同时 CAS `tail` |
| dequeue (多消费者) | `head` 指针 | 所有消费者竞争同一 cache line |
| 空队列时 | `head` == `tail` | 一个线程读 `head`、一个写 `tail` |

#### Cache Line Padding 的必要性

- 如果不 padding，`head` 和 `tail` 处于同一 64B cache line → 生产者写 `tail` 会使消费者核上包含 `head` 的 cache line 失效（false sharing）
- 典型 Rust crossbeam 实现：`CachePadded<Atomic<Node<T>>>` 包装 `head` 和 `tail`
- 即使 padding 消除了 head/tail 之间的 false sharing，**同端竞争者的 bouncing 仍存在**（多生产者对 `tail` 的竞争、多消费者对 `head` 的竞争）

#### 帮助机制的代价

- 当一个线程观察到 `tail` 滞后（已有新节点链接但 `tail` 未推进），会"帮助"CAS 推进 `tail`
- 消费者在 `head == tail` 时也帮助推进 `tail`
- **增加了 `tail` 指针上的 CAS 流量**，在高并发下可能加重 bouncing

### 3.3 Lock-Free vs Lock-Based：Cache 维度的重新审视

#### 核心悖论

> Lock-free 算法消除阻塞，但并非消除 contention——CAS 失败后的重试循环在 cache 维度可能比阻塞锁更差

#### 实证数据

**[来自对多篇研究的综合]**

| 竞争水平 | 胜出方案 | Cache 层面的原因 |
|---------|---------|----------------|
| **低竞争** | 无锁 CAS / 原子操作 | 无锁开销，单条指令即完成 |
| **中竞争** | MCS 队列锁 | 排队自旋避免 coherence 风暴，保持响应性 |
| **高竞争** | 精心设计的锁（现代 x86）| CAS 重试循环淹没内存子系统；锁序列化更干净 |

#### 关键数据点

- **无竞争延迟**：自旋锁 `xchg` (17 cycles) vs 无锁 CAS (25 cycles)——TSO 架构上自旋锁更轻量（Intel Core i7-3615QM, ACM 2013）
- **高竞争下**：基于 delegation 的队列 ~3000 万 ops/s vs 锁方案 ~850 万 ops/s（Intel Xeon E7-4870, CACM 2016）— 避免任何 CAS 竞争反而最快
- **CppNow 2026 talk "Lock-Free Programming is Dead"** 报告：现代 Intel 芯片上，精心设计的自旋锁在**高竞争下持续显著优于**无锁 CAS 循环——后者淹没内存子系统的 coherence 流量

#### 深层原因

1. CAS 在高竞争下导致 **cache line bouncing ∝ N**（N 个竞争者）
2. MCS 锁将 bouncing 降低到 **O(1)**（只传递后继）
3. 锁序列化允许其他核做有意义的工作，CAS 重试循环占据所有核使其空转
4. 现代 Intel CPU 对锁获取模式有极深的流水线优化

### 3.4 Wait-Free 的 Cache 成本

#### 更强的进度保证的代价

Wait-free 算法保证每个线程在有限步数内完成操作（不依赖其他线程），这通常意味着：

1. **多版本机制**：保持多个数据副本，读者不用阻塞等待写者
   - VLock 方案：每个版本需要 1 `usize` 元数据，外加全局原子状态
   - Left-Right 模式：双副本 + 读者计数器
   - **Cache footprint 扩大**：多版本 = 多倍数 cache 占用

2. **帮助机制**：某些 wait-free 算法要求线程之间互相帮助
   - 线程可能需要 CAS 多个其他线程的操作描述符 → 增加 cache line bouncing

3. **操作日志/描述符**：每个操作需要一个内存描述符
   - 增加 cache footprint 和跨线程读取 → 引入 coherence 写入

#### Cache 友好性权衡

| 属性 | Lock-Free (CAS 循环) | Wait-Free (多版本等) |
|------|---------------------|-------------------|
| **元数据占用** | 低（通常只需原子指针） | 高（多版本、描述符数组） |
| **Cache footprint** | 紧凑 | 更大（多副本） |
| **Coherence 写入** | 高（CAS 失败→重试） | 取决于实现（帮助机制引入远程写） |
| **读路径开销** | 通常只需读 | 可能需要写（版本注册/引用计数） |
| **适用场景** | 竞争不太激烈 | 需要硬实时保证 |

---

## 4. 硬件事务内存（HTM）与 Cache 交互

### 4.1 Intel TSX 的 Cache-Level 冲突检测

**Intel 从 Haswell (2013) 开始引入 TSX（Transactional Synchronization Extensions）**

#### 两个接口

| 接口 | 全称 | 特点 |
|------|------|------|
| **HLE** | Hardware Lock Elision | 基于 `XACQUIRE`/`XRELEASE` 前缀，对旧 CPU 透明 |
| **RTM** | Restricted Transactional Memory | `XBEGIN`/`XEND`/`XABORT` 指令，软件控制回退策略 |

#### 冲突检测机制

- **粒度**：64 字节 cache line
- **Read Set**：事务内读过的所有 cache line 集合
- **Write Set**：事务内写过的所有 cache line 集合（数据保存在 L1 中，不传播到外层）
- **冲突条件**：另一逻辑处理器**写**入 read-set 内任何 cache line，或**读或写** write-set 内任何 cache line → abort
- **利用 MESIF 缓存一致性协议**：事务期间尝试保持访问过的 cache line 在 E 或 F 状态；任何导致降级或 invalidation 的 coherence 请求都会触发冲突检测

#### Cache 容量限制

- **L1 数据缓存**作为事务读/写集的跟踪硬件
- 事务访问的 cache line 超过 L1 容量 → **capacity abort**
- Haswell 的 L1D 为 32KB（每核），实际可容纳的 cache line 在超线程共享下更少
- 这是 TSX 事务不能在事务内访问过多内存的根本原因

### 4.2 False Sharing 引发的 TSX Abort：正确性问题

#### 问题本质

> False sharing 在 TSX 事务中不仅是性能退化，更是**正确性问题**——无关数据放在同一 cache line 上会导致不应有的 abort，使本应成功的事务回滚

#### 示例

```
cache line (64B): [X (4B) | Y (4B) | ... 剩余56字节]
```
- 线程 A 的事务读取 X → X 所在的 cache line 进入 read-set
- 线程 B（非事务或另一事务）写入 Y → 同一条 cache line 被写入 → 线程 A 收到 coherence invalidation → **abort**
- 逻辑上 X 和 Y 完全无关，但因为 false sharing 导致 abort

#### 缓解策略

- **Intel TBB `speculative_spin_mutex`**：占用两条 cache line，lock 变量单独占一条 cache line（`alignas(64)`），确保与用户数据隔离
- 事务内数据结构应当：大小为 64 字节的倍数，对齐到 64 字节边界
- 每核心独立副本避免 false sharing（见 4.4 SAP HANA 案例）

### 4.3 Sub-Block 冲突跟踪（ASF 研究，AMD）

**Nai & Lee (Georgia Tech), "Reducing False Transactional Conflicts with Speculative Sub-Blocking State -- An Empirical Study for ASF Transactional Memory System", IPDPSW 2013**

#### 背景：AMD ASF

- AMD 的 **Advanced Synchronization Facility (ASF)** 是实验性的 HTM ISA 扩展
- 每 cache line 使用两个额外位：speculative-read 位和 speculative-write 位
- 冲突检测同样基于 64B cache line 粒度 → 面临与 Intel TSX 相同的 false sharing 问题
- 实验发现 **高达 46.7%** 的事务冲突为 false conflicts（误判）

#### Speculative Sub-Blocking State 技术

核心思想：将每条 cache line (64B) 划分为 **4 个子块** (每块 16B)，在子块粒度上进行冲突跟踪。

| 特性 | 标准 ASF（64B 粒度） | Sub-Blocking ASF（16B 粒度） |
|------|---------------------|---------------------------|
| **冲突检测粒度** | 64 字节 | 16 字节（4 子块） |
| **额外硬件** | 每 cache line 2 位 | 每 cache line 8 位（每子块 2 位） |
| **一致性协议影响** | 无 | **保持原有协议不变** |

#### 关键结果

- **56.4% 的 false 冲突被消除**
- **31.3% 的总冲突被消除**（包括 true conflicts）
- 性能接近 **理想系统**（无 false conflicts 的理论上限）
- 评估使用 PTLsim-ASF 模拟器和 STAMP / RMS-TM 基准套件

#### 设计优势

- 侵入性低——仅增加 cache line 内的状态位，不修改 coherence 协议状态机
- 简单可实现的硬件修改，性价比极高
- 解决了 HTM 中 false sharing 导致 abort 这个根本性痛点

### 4.4 DPTM（解耦事务内存）

**Tabba, Hay, & Goodman, "Transactional Conflict Decoupling and Value Prediction", ICS 2011**

#### 核心创新

> 将事务冲突检测与 cache coherence invalidation 解耦：收到 coherence invalidation 不必导致 abort，等到 commit 时再验证是否真正冲突

#### 机制对比

| 阶段 | 传统 HTM | DPTM |
|------|----------|------|
| **Load** | 标记 read-set | 标记 read-set + **保存值**（read mark bits） |
| **收到 invalidation** | → abort | **不 abort**，重新发出数据请求 |
| **数据返回时** | — | 比较新值与旧值：**值没变 → 继续；值变了 → abort** |
| **Store** | 需获取 Exclusive 权限 | 写入 write buffer，**不要求 Exclusive 权限**（无此阶段延迟） |
| **Commit** | 以传统方式执行 | 阶段 1：验证 read-set 数据；阶段 2：flush write buffer |

#### Cache 友好性优势

- 消除因 false sharing 引起的误判 abort（silent store、same-value writes 等不再导致 abort）
- Store 时不阻塞等待 Exclusive 权限 → 减少延迟
- 冲突检测从"eager coherence-based"变为"lazy value-based"
- **不修改底层 cache coherence 协议**，仅需处理器内部修改

### 4.5 数据结构变换适配 TSX（SAP HANA 案例研究）

**"Improving in-memory database index performance with Intel Transactional Synchronization Extensions"**, 发表于 HPCA 相关会议/期刊（SAP & Intel 合作）

#### 背景

SAP HANA 的列存储 Delta Storage Index 是一种树索引，首次直接应用 TSX 锁省略时，**性能比传统 rwlock 更差**。

#### 两个根因

1. **容量 abort**：树遍历需要 ~93 条 cache line（节点扫描 + 多次字典查找），超出 L1 容量（超线程进一步减半）
2. **意外冲突 abort**：全局字典的追加路径（共享 `size` 计数器、数组）和内存分配器元数据导致的 false conflict

#### 三项数据结构变换

| 变换 | 描述 | Cache 效果 |
|------|------|-----------|
| **Footprint 缩减** | 节点内线性搜索→二分搜索 | cache line 从 ~93 降到 ~47 |
| **Per-Core 字典** | 每核独立字典，ID 编码核号+位置 | 消除全局字典追加的冲突 abort |
| **Per-Core 内存分配器** | 每核独立内存池 + free list | 消除分配器元数据上的冲突 |

#### 最终结果

- **TSX-RTM 在 100% 插入负载上达到最优 rwlock 配置的 4.6x 吞吐量**
- 回退率降至 ~1–2%（调优前极高频回退）
- 读写混合负载接近"无锁"的性能水平
- 读场景略低于 rwlock（因残留 abort 回退开销），但作者预期在更大规模多插槽服务器上 TSX 会因消除 rwlock 读者计数器的跨插槽 coherence 流量而获胜

#### 启示

精心设计的数据结构变换（使其"TSX 友好"）远比直接用 TSX 包装已有限更有效。变换原则（per-core 辅助结构、footprint 缩减）对一般内存数据库数据结构具有普适性。

---

## 5. 当前研究前沿

### 5.1 NUMA 感知的层次化同步原语

#### Cohort Lock

**David Dice, Virendra J. Marathe, Nir Shavit, "Lock Cohorting: A General Technique for Designing NUMA Locks", PPoPP 2012 → ACM Transactions on Parallel Computing, 2015**

核心思想：
- **两层结构**：全局锁 G（thread-oblivious）+ 每 NUMA 节点的本地锁 S（cohort-detection property: 释放线程可检测同 cohort 内是否有等待者）
- 一旦某线程通过 G 获得锁，就可在**同 NUMA 节点的 cohort 内充分传递**（使用高效的本地锁），不释放 G
- 可将任意自旋锁转换为 NUMA-aware 版本
- 在 memcached 基准测试中**应用性能提升超 25%**；libc 分配器压力测试上实现**近 6x 扩展**

#### HMCS Lock

**Chabbi, Fagan, Mellor-Crummey, "High Performance Locks for Multi-level NUMA Systems", PPoPP 2015**
- 将 MCS 锁层次化：每个 NUMA 层级一个 MCS 队列
- 锁在最高局部层级上传递，仅必要时 fallback 到更高（更全局）层级

#### 层次化在 Linux 中的体现

- **CNA (Compact NUMA-Aware qspinlock)**：MCS 队列 → 主队列（同节点）+ 次队列（跨节点）
- **HQ-Spinlock (2025 RFC)**：显式借鉴 Dice 的 Cohort Lock，每 NUMA 节点独立 FIFO 队列 + 动态模式切换

### 5.2 非对称同步

#### 核心洞察

> 读端和写端的通信成本不必对称——将开销推向较少执行的一侧

#### Linux `percpu_ref`（Per-CPU 引用计数）

**Tejun Heo, Kent Overstreet et al., Linux Kernel, 2014+**

- **Percpu 模式（快速路径）**：每个 CPU 对自己的本地计数器执行非原子 inc/dec（写本地 cache line，无 bouncing）
- **Atomic 模式（慢速路径）**：当需要销毁/drain 时，通过 RCU 同步将所有 per-CPU 计数器合并到一个原子计数器
- **极端不对称**：`get`/`put` 在快速路径为纯本地非原子操作；读全局计数值需要遍历所有 per-CPU 计数器（慢）
- **Cache 优势**：避免了传统的单一原子引用计数器的 cache line bouncing

#### 权衡（USENIX 相关论文分析）

| 权衡 | 描述 |
|------|------|
| **Counting-Query Tradeoff** | 快速非原子的局部更新 vs 昂贵的全局求和 |
| **Space-Time Tradeoff** | 每对象、每核独立计数器占用大量内存 |
| **RefCache/OpLog/Paygo** | 后续研究通过 per-core hashing 而非 per-object per-core array 缓解空间开销 |

### 5.3 持久内存（PMEM）场景下的同步

#### 新约束

持久内存（Intel Optane PMEM、CXL-attached PM）引入的新问题：

1. 数据从 CPU cache 到持久介质的路径不再 "自动"
2. 需要显式的 **cache flush**（`clwb` / `clflushopt`）将脏 cache line 写回持久域
3. 需要 **memory fence**（`sfence`）保证 flush 的完成顺序
4. 崩溃一致性（crash consistency）要求**严格的写顺序**

#### 关键指令

| 指令 | 作用 | Cache 行为 |
|------|------|-----------|
| `clflush` | 刷新并逐出 cache line | Eviction + Writeback |
| `clflushopt` | 优化的刷新（乱序允许） | Eviction + Writeback |
| `clwb` | 写回 cache line（不逐出） | Writeback only, 保留在 cache |
| `sfence` | 等待之前的所有 store 完成 | 仅排序，不触发写回 |

#### Cache 影响

- PMEM 的 cache flush 开销比 DRAM 中的 cache miss 更昂贵（PMEM 写延迟 ~350ns，DRAM ~80-100ns）
- 过多 flush 导致性能下降；过少或不正确排序导致崩溃后数据不一致
- **eADR（extended Asynchronous DRAM Refresh）** 将 CPU cache 纳入持久域（通过电容实现在掉电时回刷脏 cache line），消除了 `clwb`/`sfence` 的必要性——但也消除了精细控制

#### 自动化工具

- **PMRobust** 编译器：静态分析自动插入 flush/fence，仅 0.26% 几何平均开销
- **ENTS (Extricated Non-Temporal Store)**：利用非时间存储（绕过 cache）和现有 fence-like 指令，避免显式 flush/fence，实现 **1.8x-2.1x** 于 PMDK 和 Clobber-NVM 的吞吐量

### 5.4 CXL 下的锁设计

#### CXL 共享内存的特性

- **CXL.mem**：允许主机 CPU 以 load/store 方式访问设备内存
- **CXL.cache**：设备可以缓存系统内存（MESI coherence）
- **Type 3 设备**：CXL.io + CXL.mem，实现**多主机 cache-coherent 共享持久内存**

#### 延迟特征

- CXL 内存延迟约为本地 DRAM 的 **1.5x-2x**
- 跨 CXL 拓扑（多跳）延迟更高
- 传统的 NUMA 距离模型不直接对等：CXL 设备在 coherence 域内但延迟模型不同

#### 对锁设计的影响

1. **NUMA-aware 锁需重新校准**：CXL 节点间的延迟与带宽特征不同于传统 NUMA 节点间
2. **Cohort lock / CNA 的重映射**：本地节点 → 最近 CXL 节点 → 远距离 CXL 节点 的三级层次
3. **CXL-SHM 的参考计数方案**：使用非阻塞分布式事务 + 单写者多读者模型
4. **GPF (Global Persistent Flush)**：CXL 的两阶段持久化机制在分布式拓扑中面临可靠性和能量储备挑战

#### 开放问题

- CXL 延迟下"本地 spinning" vs "远程 spinning"的阈值如何确定
- CXL 交换结构（switch fabric）中的拥塞对锁公平性的影响
- GPF 失败模式下的锁状态恢复协议

---

## 总结：Cache 友好性优化技术谱系

| 技术大类 | 核心理念 | 代表性工作 | Coherence 流量 | 适用场景 |
|---------|---------|-----------|---------------|---------|
| **朴素自旋锁** | 争抢 | TAS/Ticket | O(N) | 低竞争/N<8 |
| **队列锁** | 排队+本地自旋 | MCS/CLH | **O(1)** 每次释放 | 中高竞争 |
| **层次化队列锁** | NUMA 局部性优先 | Cohort/HMCS/CNA | O(1) + 节点内通信 | >2 插槽 NUMA |
| **RCU** | 读写分离+延迟回收 | Linux RCU/URCU | **读端零** | 读多写少 |
| **Lock-Free** | 无阻塞 CAS | M&S Queue/lock-free lists | 退化到 O(N) 高竞争 | 低竞争无锁 |
| **Wait-Free** | 有限步完成 | 多版本/Left-Right | 取决于实现 | 硬实时 |
| **HTM (TSX)** | 投机执行+冲突检测 | Intel TSX/ASF | 取决于冲突率 | 真冲突极少 |
| **HTM+变换** | 数据结构适配 HTM | SAP HANA/SF1 | 接近最优锁 | 中冲突+可变换 |
| **非对称同步** | 开销推向慢端 | percpu_ref/sloppy counter | 快速路径 = local | 读多写少/计数密集 |

---

## 关键参考文献索引

| 文献 | 作者 | 年份 | 会议/期刊 | 主题 |
|------|------|------|-----------|------|
| Algorithms for Scalable Synchronization on Shared-Memory Multiprocessors | Mellor-Crummey, Scott | 1991 | ACM TOCS | MCS Lock |
| Lock Cohorting: A General Technique for Designing NUMA Locks | Dice, Marathe, Shavit | 2012/2015 | PPoPP/ACM TOPC | Cohort Lock |
| High Performance Locks for Multi-level NUMA Systems | Chabbi, Fagan, Mellor-Crummey | 2015 | PPoPP | HMCS Lock |
| Exploiting Deferred Destruction (PhD Dissertation) | McKenney | 2004 | OGI | RCU 体系 |
| Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms | Michael, Scott | 1996 | PODC | Lock-Free Queue |
| Reducing False Transactional Conflicts with Speculative Sub-Blocking State | Nai, Lee | 2013 | IPDPSW | Sub-Block Conflict Tracking |
| Transactional Conflict Decoupling and Value Prediction | Tabba, Hay, Goodman | 2011 | ICS | DPTM |
| qspinlock (Linux kernel patch series v1-v16) | Waiman Long, Peter Zijlstra | 2013-2015 | Linux Kernel ML | qspinlock |
| CNA: Compact NUMA-Aware qspinlock | Alex Kogan | 2019-2021 | Linux Kernel ML | CNA |
| HQ-Spinlock RFC | Stepanov, Fedorov | 2025 | Linux Kernel ML | HQ-Spinlock |
| Userspace RCU | Mathieu Desnoyers | 2009+ | LGPL Library | URCU |
| DEBRA/DEBRA+ | Trevor Brown | 2015 | PPoPP-related | EBR 改进 |
| Hazard Eras | Ramalhete, Correia | ~2017 | PPoPP/PODC-related | HE 混合方案 |
| Improving in-memory database index performance with Intel TSX | SAP/Intel | ~2014 | HPCA Workshop/Related | TSX + 数据结构变换 |
| Making lockless synchronization fast (NER) | Hart et al. | 2016 | IPDPS | EBR fence 摊销 |

---

*本笔记编译于 2026 年 7 月，基于所引文献的研究成果。*
