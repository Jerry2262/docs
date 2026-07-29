---
title: "工程工作日志"
date: 2026-07-29
description: "日常工程工作中积累的短平快经验，持续更新。"
category: 工程
---

日常搬砖过程中遇到的小经验，每条几分钟能讲完。不展开，不引论文，用到的时候能想起来就行。

## 2026-07-29 — 研究一个库之前，先去搜 issue

文档可能是过时的，源码太花时间。

性价比最高的动作：打开 GitHub Issues，搜你关心的问题。

- 有没有人踩过你即将踩的坑？
- 维护者怎么回复的——积极修还是已读不回？
- bug 多么、严重么？
- feature request 的处理风格——社区驱动的还是作者说了算？

十分钟，基本能判断这库能不能用、好不好用。比看完文档发现不对劲再换，划算太多了。

## 2026-07-29 — ARM64 上 profiling 遇到 memory bound：上 SPE 和 C2C

在 ARM64 平台上做性能 profiling，如果发现瓶颈是 memory bound，常规的采样可能不够细。这时候 ARM 的两个硬件特性很好用：

**SPE（Statistical Profiling Extension）**：能抓到具体是哪条汇编指令导致的 cache miss，而不只是停在函数级别。用 `perf` 抓取：

```bash
perf record -e arm_spe_0/load_filter=1,store_filter=1/ -a -- your_workload
perf report
```

**C2C（Cache-to-Cache）**：当怀疑性能损失来自多个线程抢同一根 cache line 时，用 perf c2c 来分析 false sharing 或 true sharing。

```bash
perf c2c record -- your_workload
perf c2c report
```

组合拳：SPE 定位 hot 指令 → C2C 判断是不是跨线程的 cache line 竞争。很多时候优化方向不是减少计算，而是换个数据结构布局。
