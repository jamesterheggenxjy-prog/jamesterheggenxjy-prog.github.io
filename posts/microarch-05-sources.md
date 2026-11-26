---
title: 微架构安全学习资料
description: 本书所用的教材、厂商手册和研究文献。
tags: [微架构安全, 资料, 手册]
date: '2026-11-25'
book: microarch-00-intro
---
## 教材与习题

- 王道《计算机组成原理考研复习指导》的存储系统章依次组织局部性、Cache 工作原理、映射、替换与写策略，并配有习题和解析。 [电子工业出版社公开目录](https://phei.com.cn/module/goods/wssd_content.jsp?bookid=51711)
- 唐朔飞《计算机组成原理》从存储系统和处理器功能出发，分别讲授 Cache、指令与指令流水。 [高等教育出版社书目与目录](https://www.hep.com.cn/book/show/ae8b4f15-c953-4445-a56d-ed96390fe0ae)
- 《深入理解计算机系统》第 6 章从程序访问顺序理解内存层次。作者给出了矩阵访问顺序影响缓存性能的实例。 [CS:APP 作者网站](https://csapp.cs.cmu.edu/3e/perspective.html)
- Hennessy 与 Patterson 的《计算机体系结构：量化研究方法》第 2 章讲内存层次设计，第 3 章讲指令级并行。案例和配套讲义可用于对照本书的访问代价与指令流水部分。 [Elsevier 目录](https://shop.elsevier.com/books/computer-architecture/hennessy/978-0-12-811905-1) · [配套讲义](https://shop.elsevier.com/books/book-companion/9780128119051)
- MIT 6.004 的讲义覆盖缓存查询、写策略和处理器流水线。缓存工作表可用于进一步练习地址拆分和性能计算。 [MIT 6.004 讲义](https://6191.mit.edu/_static/spring26/resources/references/6004_student_notes_FA20.pdf) · [缓存习题与解答](https://ocw.mit.edu/courses/6-004-computation-structures-spring-2017/b71446c5273c385dfcc4e50f00a26b6a_caches_answers.pdf)

## 手册与研究文献

- gem5 的经典缓存源码适合跟踪模拟模型中的未命中、合并和预取流程。本系列固定使用 v24.1.0.3。Chipyard 的 BAR-fetchers 提供可对照的 Chisel 硬件接口。 [gem5 经典缓存](https://www.gem5.org/documentation/general_docs/memory_system/classic_caches/) · [gem5 源码版本](https://github.com/gem5/gem5/tree/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache) · [BAR-fetchers](https://github.com/ucb-bar/bar-fetchers/tree/bd02692fe6cb5197df90dc39051d31369ef61fcd)

- Intel 的软件开发手册适合查指令语义和架构约定。优化参考手册适合查缓存和内存层次，以及各型号的实现差异。 [Intel 架构与优化手册](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- Arm Cortex‑A76 手册的 L1、L2 章节给出了一款具体核心的缓存设计，可以用它对照本文的 4 组、2 路教学模型。 [Arm Cortex‑A76 技术参考手册](https://documentation-service.arm.com/static/5f562083235b3560a01e03bc)
- 要读懂测量结果里的一条计数，先查该处理器对事件的定义，再核对操作系统给调用者的访问权限。 [Linux Perf events and tool security](https://docs.kernel.org/admin-guide/perf-security.html)
- Intel 的术语文档给出了从信号来源到侧信道威胁模型的区分。 [Intel 推测执行术语](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/best-practices/refined-speculative-execution-terminology.html)
- BOOM 的前端文档从取指组、快速预测、方向预测到历史恢复给出了可追踪的设计实例。Intel 的 BHI 文档可用于对照真实处理器公开的历史影响与模式边界。 [BOOM 分支预测](https://docs.boom-core.org/en/latest/sections/branch-prediction/index.html) · [Intel BHI 文档](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/branch-history-injection.html)
- 读完本单元后，可以回到两项原始研究，分别核对它们对暂态执行和可观测载体的论证。 [Spectre](https://spectreattack.com/spectre.pdf) · [Meltdown](https://meltdownattack.com/meltdown.pdf)

## 查找专题资料

先看硬件安全目录，再用具体对象的名字检索，例如 `cache replacement`、`cache occupancy`。遇到实验结论时回到手册或论文原文，确认测试的芯片、测量条件和观察者位置。 [VoltJockey 硬件安全论文库](https://papers.voltjockey.com/hw)

本书的知识条目组织参考 CTF Wiki，章节中的推导与练习安排参考公开的大学 STEM 教材编写技能。 [CTF Wiki · 栈介绍](https://ctf-wiki.org/pwn/linux/user-mode/stackoverflow/x86/stack-intro/) · [textbook-anything](https://github.com/Alchemist-Jo/textbook-anything)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>上一篇</small><strong>Spectre 变体中的分支与访存推测</strong></span>](./microarch-04c-variants.md){.article-nav-card .article-nav-prev rel=prev}

</nav>
