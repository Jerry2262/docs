---
title: "面向多线程共享导致 Cache Miss 的编译器优化技术"
date: 2026-07-30
description: "编译器视角的 cache miss 优化：数据布局变换、循环访存优化与 PGO 反馈闭环。"
category: 体系结构
---


## 1. 数据布局变换 (Data Layout Transformations)

编译器在不改变程序语义的前提下，自动优化数据结构的内存排布以减少 false sharing。

### 1.1 结构体字段重排序 (Field Reordering)

**核心思想**：将频繁被不同线程访问的字段拆分到不同 cache line，将同一线程内频繁共访的字段聚合到同一 cache line。

**LLVM 现状**：
- LLVM 中**尚无专门的 field reordering pass**（根据 2021 年 12 月 LLVM-dev 讨论确认）。
- 对局部 `alloca` 的拆分由 **SROA (Scalar Replacement of Aggregates)** pass 完成——将聚合类型递归拆分为细粒度分区，每个分区单独 `alloca` 并通过 `mem2reg` 提升为 SSA 标量。但 SROA 不负责"重排"字段顺序。
- 全局常量的拆分由 **GlobalOpt** 完成。
- 社区共识：跨函数边界的结构体变换需要 LTO 支持，且面临 ABI 复杂性（不同调用约定的参数传递方式不同）。

**Rust 编译器**：Rust **默认重排结构体字段以消除 padding**（非按声明顺序），但此重排仅服务于对齐/padding 消除，不基于 profiling 或 false sharing 检测。

**C/C++ 限制**：C/C++ 语言标准要求 struct 字段保持声明顺序（因 `offsetof`、分离编译等），编译器不能自动做字段重排。程序员需手动调整或使用 `alignas`。

### 1.2 AoS -> SoA 自动转换

**问题定义**：将 `struct S { int a; float b; } arr[N]` 转为 `struct S_SoA { int a[N]; float b[N]; }`。

**转换条件（何时合法）**：
- 需要分析数组元素的访问模式：元素间是否真正独立？是否存在跨元素的数据依赖？
- 指针算术和取地址操作（如 `&arr[i].a` 被传递给其他函数）可能阻止转换。

**相关工作**：
- **Hwu et al., IMPACT 编译器**（1990s, UIUC）：早期探索 AoS→SoA 转换，但细节在现有搜索结果中未直接呈现。IMPACT 以 ILP 编译为特色，其内存消歧和依赖分析为后续工作奠基。
- **属性驱动的半自动转换**（Durham University, 2025）：使用 `[[clang::]]` attribute 标注需要转换的结构体和字段，编译器自动生成 SoA 临时副本、重定向访问、可选回写。非全自动，需程序员介入。
- **ASA Transformation**（LLVM/Clang, UIUC）：`__attribute__((array))` 注解自动将 AoS 转 ASA（Array of Structures of Arrays），字段内部展开为小数组，下标表达式在 AST 层重写。CUDA benchmark 上达到手动优化 95% 的性能。
- **Reuse 驱动的区域化布局变换**（Rohwedder et al., CC '24）：RebaseDL 框架在 LLVM 中基于数据重用分析识别局部代码区域，进行结构体拆分、字段重排、打包，避免全局变换的 profiling 开销。SPEC CPU 上最大 1.34x 加速。

### 1.3 自动 Padding 插入

**目标**：识别跨线程并发访问的全局/堆变量，在它们之间自动插入 padding 至 cache line 边界。

**现状**：
- GCC/LLVM 支持 `__attribute__((aligned(64)))` 和特定平台宏（如 Linux 内核的 `____cacheline_aligned`），但均为**手动标注**。
- **尚无主流编译器自动插入 padding 的通用 pass**。
- 技术上可实现：通过静态分析识别被多个线程写入的相邻全局变量（特别是同一并行区域的 OpenMP 变量），在其间注入对齐填充。挑战在于判断哪些变量"会被不同线程同时写入"的精度。

### 1.4 结构体拆分 (Structure Splitting / SROA)

**LLVM SROA pass**（`llvm/lib/Transforms/Scalar/SROA.cpp`）：
- 四阶段流程：分析（递归遍历 `alloca` 的所有 use，构建内存范围表达）-> 分区（将使用的范围划分为最小分裂数） -> use 分配（每个访存分配至碰到的所有分区） -> 重写与提升（为每个分区构建新的更小 `alloca`，通过 `mem2reg` 提升为 SSA 标量）。
- **局限**：仅作用于局部非逃逸 `alloca`。Roman Lebedev（2021）在扩展 SROA 以处理通过函数调用的指针逃逸和变量 GEP 下的 alloca 拆分。
- **GlobalOpt** 处理全局常量的拆分。
- 结构体"剥离"(peeling) --- 将热字段与冷字段分开 --- 在 LLVM 中尚未实现。

---

## 2. 循环与访存优化 (Loop and Memory Access Optimizations)

改善程序在共享数据上的访存模式，提高 cache 局部性并降低 coherence 开销。

### 2.1 Loop Tiling / Blocking

**原理**：将嵌套循环的迭代空间分块为 tile，使数据在 cache 中充分复用后再被替换。

**对 false sharing 的缓解**：
- Tiling 将不同线程的访问限制在不同的数据块上，减少了多个线程对同一 cache line 的竞争概率。
- 临界：tile size 应与 cache line size 对齐；若 tile 边界跨 cache line，效果大打折扣。

**关键工作**：
- **Wei Li (1993), "Affinity Tiling"**：同时利用 tile 内部和 tile 间的数据局部性，对 tile size 选择不敏感。通过扩展 tile 概念显式消除 false sharing。
- **Gupta & Padua**：将循环 strip-mine 到 cache block size，每个 strip 分配给不同处理器，miss 率降低 4%~60%。
- **TileMind** (TACO '24)：解耦分析模型，通过非线性优化 + 线性化求解自动选择 tile size。PolyBench 上 1.49x (串行) / 1.33x (并行) 加速，DL 负载 2.08-3.54x。

### 2.2 Loop Fission / Distribution

**原理**：将一个循环拆成多个循环，各自访问数据的不同部分。

**对 coherence 的优化**：
- 将读为主的阶段与写为主的阶段分离，减少来自其他核的 cache 失效通知(coherence invalidation)。
- 在 false sharing 场景中，fission 可隔离写入操作，减少不同线程同时修改同一 cache line 的概率。

**关键工作**：
- **Ju & Dietz (LCPC '91)**：《Reduction of Cache Coherence Overhead by Compiler Data Layout and Loop Transformation》。系统性地将数据布局优化和循环变换结合，通过 coherence 代价函数驱动 fission 决策。效果：false sharing miss 减少 56%-93.5%，执行时间改善 2%-58%。

### 2.3 Loop Fusion

**原理**：将多个循环合并为一个，减少数据在内存与 cache 间的往返次数。

**trade-off**：融合后循环体更大、变量活跃期更长，可能增加 cache 压力并恶化 false sharing（如果不同原循环中有不同线程写入相邻数据）。因此需与 fission 形成双向权衡——Basteln 策略 (CC '24) 将 tiling 与 fusion 的交互建模到 polyhedral 编译框架中，联合优化。

### 2.4 Loop Interchange

**原理**：改变循环嵌套顺序以匹配数据的内存布局（行优先 vs 列优先）。

**对 false sharing 的影响**：间接的——通过将并行循环移动到最内/外层，可改变哪些数据被不同线程同时访问，从而影响 cache line 竞争模式。

### 2.5 Cache Bypassing (Non-temporal Hints)

**原理**：编译器在识别到"只写一次、不再读取"的数据时，插入 non-temporal store 指令（如 x86 `MOVNTI`、`VMOVNTDQ`），绕过 cache 直接写回内存，避免污染 cache。

**编译器支持现状**：

| 特性 | Clang/LLVM | GCC |
|------|-----------|-----|
| `__builtin_nontemporal_store` | 有，但仅生成标量 `movnti`，**破坏自动向量化** | 无 |
| OpenMP `nontemporal` pragma | 未实现 | 解析但不生成 MOVNT |
| 自动生成 MOVNT | **否**——LLVM 明确保守，只在显式请求时生成 | **否** |
| 向量 non-temporal store | 仅通过 `_mm*_stream_*` intrinsics | 仅通过 intrinsics |

**正确性陷阱**：x86 上 non-temporal store 使用 write-combining buffer，绕过 TSO 规则。`fence release` 在 x86 上被降低为仅编译器屏障（假定 TSO），但 MOVNT 绕过 TSO，导致 release 前的 non-temporal store 可能还未完成——需要显式 `SFENCE` 指令。LLVM 当前 lowering 不会自动插入 `SFENCE`。

---

## 3. 线程感知优化 (Thread-Aware Optimizations)

编译器感知多线程语义，进行针对性优化。

### 3.1 Thread-Local Storage 的编译器支持

**`__thread` / `thread_local` 的实现**：
- GCC 和 LLVM 均支持。
- `thread_local` 变量通常放入 `.tdata`/`.tbss` ELF section，由运行时通过 TLS 基址寄存器快速寻址（x86-64 用 `fs` 段寄存器）。
- LLVM 中对应的 IR 概念是 `thread_local` 全局变量，生成 `@llvm.threadlocal.address` intrinsic。

**自动推断的潜力**：
- 编译器理论上可通过静态分析判断：某全局变量是否在每种执行路径上均由同一线程独占访问（无跨线程的写入交错）？
- 但**目前无主流编译器实现此项自动推断**。挑战在于需要过程间分析和跨 TU 可见性，本质上是 alias analysis 在多线程语义下的扩展。

### 3.2 OpenMP 指导语句的编译器实现

**`private` vs `firstprivate` vs `threadprivate` 的映射**：

| 指令 | 存储位置 | 初始化 | 持久性 | Cache 相关 |
|------|---------|--------|--------|------------|
| `threadprivate` | TLS/全局 per-thread 存储 | `copyin` from master | 跨并行区 | LLVM runtime 使用缓存复用机制；每次并行区仅调用一次 `__kmpc_threadprivate_cached` |
| `private` | 栈或通过 `GOMP_alloc` 分配 | 未初始化 | 单构造 | 相邻线程的栈上变量可能共享 cache line（false sharing 风险） |
| `firstprivate` | 栈或分配+复制 | 从 master 复制 | 单构造 | LLVM 的 packing 优化：<1024B 的多个 firstprivate 参数打包为单次传输（D86307） |

**per-thread 数组的 false sharing**：并发线程各自拥有 `private` 数组元素时，若数组按线程 ID 为第一维（如 `a[thread_id][N]`），需确保每个线程的元素跨 cache line 边界（`a[thread_id][cache_line]` 而非 `a[thread_id]`）。

### 3.3 False Sharing 编译器自动检测

**Intel 编译器 (icx/icc)**：**无专门的编译器 warning 标志**来检测 false sharing。Intel 官方文档指出编译器"在优化中会消除一部分 false sharing"（如使用 thread-private 临时变量），但无诊断机制。检测依赖 **Intel VTune Profiler**（基于 PMU counter 如 `MEM_UNCORE_RETIRED.OTHER_CORE_L2_HITM`）和 **PTU**（Performance Tuning Utility，提供"Contested Usage"/"False-True Sharing"预配置）。

### 3.4 Lshaz -- Clang/LLVM 静态分析器

**全称**：Lshaz (Latency-Sensitive HAzard analyzer)

**作者**：Ahmed Bokhalil (独立开发)

**技术架构**（双重分析管道）：
1. **AST 层**：用 `RecursiveASTVisitor` 遍历 AST，通过 `ASTRecordLayout` 计算每个字段的字节偏移。应用 15 条独立规则（每条对应特定硬件机制，如 false sharing FL002、cache line 跨越、原子争用等）。同时做 per-TU escape analysis 判断哪些类型真正逃逸到多线程。
2. **IR 层交叉验证**：生成 LLVM IR（可配置优化级别，默认 O2），对照 AST 发现检查优化器是否消除了风险（如 `seq_cst` atomic 被提升为寄存器、调用被 devirtualize、分配被 inline）。为每个诊断赋予置信度 score。

**架构自适应 cache line size**：

| 架构 | 默认 Cache Line |
|------|----------------|
| x86-64 | 64B |
| AArch64 | 64B |
| Apple ARM | 128B |

可通过 `--cache-line-size=<N>` 覆盖。

**实际效果**：
- LLVM 自身代码库（4502 TU）：最高置信度发现是 `TrackingStatistic` -- 3 个冷 `const char*` 指针与 1 个 hot `atomic<uint64_t>` 共享同一 cache line。
- Abseil 库（157 TU）：18 个 FL002 发现，精度 100%。如 `HashtablezInfo`（9 个 atomic 塞进 3 个 cache line）。

**设计取舍**：作为独立可执行文件编译，不修改 LLVM 本身；AST-IR 关联通过 debug info metadata 而非自定义 pass。

- **仓库**：https://github.com/abokhalill/lshaz

---

## 4. PGO 与反馈优化 (Profile-Guided and Feedback-Directed Optimization)

### 4.1 Perf c2c 驱动的反馈循环

**`perf c2c`**（Linux `perf` 工具的 cache-to-cache 子命令）：基于 PMU 硬件计数器采样，检测因 false sharing 和 true sharing 导致的 HITM（Hit Modified）事件，定位跨 CPU 的 cache line 争用。

**目前的端到端流程**（大部分仍为手动）：
```
perf c2c record -> perf c2c report -> 人工识别热点 -> 手动修改代码/布局 -> 重新编译 -> perf c2c record (验证)
```

**自动化是研究热点**：如何将 `perf c2c` 的结果自动反馈给编译器/链接器，自动执行重布局（padding 插入、字段移动等），然后自动重新 profile 验证闭环。

### 4.2 AutoFDO + Propeller (Google)

**AutoFDO**（Automatic Feedback-Directed Optimization）：
- 基于 CPU PMU 硬件采样（而非代码插桩），在**生产环境**直接收集 profile。
- 避免插桩的 ~10x 运行时开销。
- 数据流：`生产内核 perf record` -> `llvm-profgen 转换` -> 编译器读取 profile -> 优化重编译。
- 主要优化：函数内联、分支 fall-through 改进、间接调用去虚拟化。
- 2024 年 10 月提出 upstream 到 Linux 内核（Rong Xu, Han Shen, Google）。LLVM 17+ 可用。

**Propeller**（Post-Link Optimizer）：
- 与传统 post-link 优化器（如 BOLT）不同，Propeller 不直接修改二进制，而是将 profile 写回编译器/链接器，**从源码重新编译**。
- 优点：编译器级别的变换精度最高、可与分布式 build cache 兼容、可优化内核模块。
- 优化：basic-block 布局重排、路径克隆、inter-procedure register allocation（进行中）。
- 限制：要求 profile 时和重编译时的源码和构建选项完全一致。

**性能数据（2024）**：
- Neper/tcp_rr 延迟：10.6% 降低（AutoFDO）
- Neper/tcp_stream 吞吐：6.1% 改善
- Google 数据库应用：2.6% CPU 减少
- AutoFDO + ThinLTO + Propeller 叠加：比单独 AutoFDO 再降 4.5% 延迟

**数据布局扩展潜力**：目前 AutoFDO 和 Propeller 都主要面向**指令 cache 优化**（basic block 排列）。方法论可扩展到数据布局——基于 LBR/perf mem 采样的数据访问 profile 指导数据对象重排，但尚未成熟。

### 4.3 BOLT (Meta)

**BOLT** (Binary Optimization and Layout Tool)：
- 2018 年开源，2022 年起成为 LLVM 的一部分。
- 操作方式：读取未 strip 的 ELF 二进制 + perf 采样 profile -> 反汇编 -> 构建带权重边的 CFG -> 重排 basic block -> 将函数拆分为 hot/cold 片段 -> 重写二进制。
- 效果：HHVM 7% 性能提升；Meta 生产服务 kernel 优化后 2% QPS 增长。
- **局限**：专注于**代码布局**。目前**不支持 data layout 或 false sharing 的反馈优化**。BOLT 无法直接替换二进制中的数据段布局。
- x86_64、ARM64、RISC-V 均支持。profile 可通过 Intel LBR 或其他架构分支采样收集。

### 4.4 理想化的反馈优化闭环（研究展望）

```
第 0 轮: 采样型 profiler (perf c2c / LBR) 在目标负载上收集数据
     |
第 1 轮: 热点识别 -> 编译器自动修复 (padding/重排/AoS->SoA)
     |
第 2 轮: 重新 profile -> 效果验证 -> 迭代
```

目前此闭环中每个箭头仍是手动的。实现自动化的关键技术挑战：profile 热点到源码变量的反向映射（当优化器已完成大量变换后）、合法性的自动证明、布局变化的全局影响评估。

---

## 5. 语言层面的编译器支持 (Language-Level Compiler Support)

### 5.1 Rust 所有权信息利用

**Rust 编译器已知信息**：
- 每个堆分配的唯一 owner（`Box<T>`, `Vec<T>` 等）。
- 引用借用的精确作用域：`&T`（共享不可变借用，多 reader 无 writer）与 `&mut T`（独占可变借用，单 writer）。
- 生命周期标注（`'a`）派生的别名关系。
- `Send` / `Sync` trait 标记跨线程安全性。

**当前 rustc 对这些信息的利用**：
- 所有权信息**主要用于 borrow checker（安全性检查）和 drop 插入**，对 LLVM 后端有一定影响（生成 `noalias` 属性，改善 alias analysis），但**不用于数据布局优化**。
- struct 字段重排仅服务于 padding 消除，不区分热/冷字段或线程访问模式。
- rustc 中无专门利用 `Send + Sync` 信息指导 false sharing 预防的优化 pass。

**DRust (OSDI '24) 的启发**：
- **论文**：Haoran Ma, Yifan Qiao et al. (UCLA/UCSD/Alibaba), "DRust: Language-Guided Distributed Shared Memory with Fine Granularity, Full Transparency, and Ultra Efficiency."
- **核心洞察**：Rust 的所有权模型天然 enforce **单写多读 (SWMR)** 语义——唯一 owner、单一 `&mut`、共享 `&`。DRust 利用编译器提取此信息，构建**无需 explicit invalidation 的分布式一致性协议**——写操作时转移全局地址（自然使远端 cache 失效），读操作时安全复制。
- **性能**：8 节点集群上 vs 传统 DSM 实现 GAM 最高 2.64x 吞吐加速，vs 单机 Rust 仅 2.42% 平均开销。
- **对本地场景的启示**：Rust 编译器已知的 aliasing 信息如果被传递给 LLVM 后端并用于指导字段布局和 padding 决策，理论上有望在无需程序员标注的情况下自动消除本地 false sharing。**但此路径尚未被探索**。

### 5.2 C++ 的标准化工具

**`std::hardware_destructive_interference_size` 与 `std::hardware_constructive_interference_size`（C++17）**：

这是语言标准对 false sharing 问题的直接回应：

- **`hardware_destructive_interference_size`**：两个对象若要避免因共享 cache line 导致的 false sharing，它们之间的最小偏移量。典型值 64 或 128。
- **`hardware_constructive_interference_size`**：为促进 true sharing（即真正共享数据），连续内存的最大大小。典型值 64。

**编译器支持状态**：

| 编译器 | 状态 | 宏/参数 |
|--------|------|---------|
| GCC | 2021 年起支持 (`libstdc++`) | `__GCC_DESTRUCTIVE_SIZE`, `__GCC_CONSTRUCTIVE_SIZE`; `-Winterference-size` 警告 ABI 不安全使用 |
| Clang | 2024 年起支持 (PR #89446, commit 72c373b) | 同样暴露 `__GCC_DESTRUCTIVE_SIZE`, `__GCC_CONSTRUCTIVE_SIZE`; `libc++` 标准库实现 blocked 至此 |

**关键 caveat**：这些值**不是 ABI 稳定的**——不同 `-mtune`/`-march` 可能产生不同值，跨 Release 也可能变化。公共头文件中使用会导致 ODR 违规风险。WG21 接受了此 ABI 风险以换取实用价值。

**其他 C++ 工具**：
- `alignas`：手动对齐到 cache line 边界。`alignas(64) char padding[64]` 是典型模式。
- `[[likely]]` / `[[unlikely]]`：帮助分支预测，对数据 cache 影响间接。
- `struct` 字段顺序：在 C++ 中由声明顺序保证，编译器不能重排。

---

## 6. 当前研究前沿 (Current Research Frontiers)

### 6.1 AI 辅助的编译器优化决策

传统编译器依赖静态启发式做数据布局、tile size、fusion/fission 决策。ML 正逐步介入：

- **ML 预测 tile size + loop order**（IISc）：SVM 集成模型在 Intel Xeon Cascadelake 上找到 6.54-8.80% 内最优的 loop order，层次 SVM 预测最佳 `<tile_size, loop_order>` 组合。
- **TileMind (TACO '24)**：非线性优化模型 linearize 后高效求解 tile size。2-4 个数量级快于 TVM MetaSchedule 的 autotuning 开销。
- **LayoutFusion (2025)**：张量数据布局对 tile configuration 选择的影响——layout-aware filtering，RTX 3090 上最高 5.55x。
- **Axon (2025)**：基于 synthesis + SMT equivalence checking 的超优化器，无手写 rewrite rule。在 Amazon Trainium 上最高 3.7x (单 op)、19x (multi-op LLM kernel)。
- **Hexcute (2025)**：基于类型推断的自动布局和 task-mapping 合成，混合精度算子 1.7-11.28x。

**false sharing 相关的 ML 应用**：目前仍缺专门用 ML 预测 false sharing 风险和优化布局的成熟工作，是明显的 research gap。

### 6.2 跨模块优化 (LTO/ThinLTO/WPO) 在数据布局上的应用

- **LTO (Link-Time Optimization)**：全程序 IR 可见后，编译器可以安全地重排结构体字段（因为所有 field access 站点都可见并可重写）。LLVM 的 ThinLTO 已支持跨 TU 内联和间接调用去虚拟化，但数据布局变换 pass 尚未集成进 LTO 管线。
- **挑战**：外部库边界（已编译成机器码的第三方库）形成布局变换的硬边界——重新排列其使用的结构体会破坏 ABI。
- **WPO (Whole-Program Optimization)**：Microsoft Visual C++ 的 `/GL` + `/LTCG` 支持类似概念，社区对数据布局的 WPO 探索较少。
- **AutoFDO + ThinLTO 叠加** (Google, 2024)：已验证组合收益，但目前仍主要面向**代码布局**而非数据布局。

### 6.3 JIT 编译器的运行时重优化

**Azul Zing (Prime) JVM** 提供了一套分层防御：

1. **`@Contended` 注解**：运行时动态插入 padding（默认 128B）到标注字段前后以避免 false sharing。可通过 `-XX:ContendedPaddingWidth` 配置。
2. **ObjectLayout 项目**：将 C struct 风格的连续内存布局引入 Java 堆。引入 "contained" / "container" 对象概念，支持**dead reckoning**（无指针解引用的字段访问）和 streaming（连续内存上的高效遍历）。通过 CHA (Class Hierarchy Analysis) 和 inline-caching 的编译器支持实现。
3. **Compile Stashing**：缓存 JIT 编译结果供后续运行重用，减少 warm-up 时间。`-XX:+FalconUseCompileStashing`。

**JIT 对 false sharing 的优势**：JIT 编译器拥有运行时采样的实际访问模式，可以做出比 AOT 编译器更精准的优化决策。一个 MPhil 项目在 JVM 上做动态 profiling 检测 false sharing 并拆分 struct 到不同 cache line，microbenchmark 上 50% 加速——但真实应用中差异不可测，说明程序员已在手动规避。

### 6.4 编译器与硬件 CAT 的协同

**Intel CAT (Cache Allocation Technology)**：
- 通过 way-partitioning 对 LLC 进行软件可编程的分区。每个 CLOS (Class of Service) 通过 CBM (Capacity Bitmask) 指定其可用的 cache way。
- 传统用法是 OS 层面为进程隔离 cache 资源。

**编译器指导的 CAT 调度 -- Com-CAS (CC '21)**：
- **LLVM 前端** (Probes)：插入 marker ("probe") 到每个最外层循环嵌套入口。用 polyhedral 分析静态计算访存 footprint，SRD (Static Reuse Distance) 分类为 streaming vs reuse。
- **运行时后端** (BCache)：编译器预估的 footprint 和 reuse 性质指导 proactive cache 分区决策。每当应用进入不同 reuse 行为的循环阶段，CBM 通过 `resctrl` 文件系统更新。
- **效果**：vs 无分区提升 15%，vs KPart（SOTA 响应式分区）提升 20%。
- **数据布局层面的协同潜力**：编译器可进一步指导 CAT 为不同类型的数据结构（如热元数据 vs 冷 bulk data）分配专属 cache way，避免跨类型污染。这是编译器优化与硬件资源管理深度融合的长远方向。

### 6.5 编译器自动检测 + 修复 False Sharing（总结与展望）

**现有工作层次**：

| 层次 | 工具 | 成熟度 |
|------|------|--------|
| 静态分析 | Lshaz (LLVM/AST+IR) | 原型，高精度 |
| 动态 profiling | perf c2c, VTune, PTU | 生产可用 |
| 手动修复 | `alignas`, `__cacheline_aligned`, padding | 标准实践 |
| 编译器自动修复 | 无 | **研究空白** |

**关键技术挑战**：
1. **精度**：静态分析必须区分 true sharing 与 false sharing，但跨线程的 aliasing 分析是 undecidable 的。
2. **合法性**：任何布局变换都不能改变程序语义（C/C++ 中 struct 布局语义强，Rust 中较灵活）。
3. **全局影响**：局部布局优化可能增加整体内存占用（padding 放大对象）、引入新的 cache 竞争热点。
4. **反馈闭环**：从 profile 到源码的精确反向映射，特别是在编译器已做大量变换后。

---

**编制日期**：2026 年 7 月
