---
title: 微架构安全入门
description: 从缓存、取指和执行学起，再沿共享资源与瞬态执行等路线分析微架构安全问题。
date: '2026-11-25'
tags: [微架构安全, 导论, Spectre, Meltdown]
bookTitle: 微架构安全
---

两个线程各自修改不同的计数器，为什么还会等待对方？一条指令最终没有提交，为什么它先前发出的请求仍可能改变缓存？这两个问题都要追踪程序结果之外的硬件状态。前者可以从缓存行和写权限讲起，后者需要理解预测、异常和指令退休。

微架构是指实现指令集的软件可见规则所用的硬件组织。处理器可以保存数据副本、预测后续访存和下一条指令的地址，也可以先执行暂时不必交付结果的操作。这些行为平时用于提高速度。某些内部状态会影响之后的访问时间。当受保护的信息能够改变这些状态时，时间就可能成为分析安全边界的一部分。

## 本书怎样展开

**Spectre** 原始研究的一条路径从错误的控制流预测出发。处理器沿着预测路径执行了一段操作，发现方向不对后丢弃结果。**Meltdown** 原始研究则表明，在论文讨论的受影响处理器上，一次在架构上会被权限拒绝的加载，可能在报告异常前影响后续的内部操作。这两个例子共同引出一个问题：如果相关指令的结果没有正式生效，哪些内部状态还可能使它的影响留到后来？

读者应会基本的十六进制运算，知道寄存器、地址和数组的含义。要理解共享资源，先读「访存局部性与高速缓存」，从本地命中跟到多核写权限，再用第五、六章分析不同的时间差。要理解瞬态执行，可以先读地址翻译、指令执行和分支预测，再分别读两种漏洞机制。带着具体问题来查阅时，可以直接进入相应文章，按正文线索补足先修知识。

## 目录 {#book-contents}

### 第一章　访存局部性与高速缓存

先从字节地址算出缓存块，再用复用距离判断为什么同一组数据换个顺序就会多出未命中。随后用同一串地址比较 LRU 与 FIFO，追踪未命中请求和写回。学完单核缓存后，再用一条由两核共享的缓存行推导写权限、失效确认和最新数据的去向。最后把两个计数器放在同一行和不同的行，在两个物理核上比较开销，并学习怎样定位引起伪共享的字段。

1. [缓存行与局部性](./microarch-01-cache.md)
2. [缓存怎样查找和替换一行](./microarch-01b-cache-mapping.md)
3. [矩阵按行读和按列读的缓存开销](./microarch-01d-cache-traversal.md)
4. [缓存未命中后如何合并请求](./microarch-01a-mshr.md)
5. [写入缓存后，数据什么时候送到下一级](./microarch-01e-cache-stores.md)
6. [缓存一致性：两个核怎样读到最新值](./microarch-01f-coherence.md)
7. [两个线程改不同字段，为什么缓存行还会来回传](./microarch-01g-false-sharing.md)

### 第二章　硬件预取器

先从三次访问算出步长和候选地址，再跟踪候选怎样过页、排队和填充。随后到 RTL 一节，逐拍看缓存接收与拒绝怎样改变暂存槽。

1. [硬件预取器怎样工作](./microarch-02-prefetch.md)
2. [预取器和数据缓存怎样握手](./microarch-02d-prefetch-rtl.md)

### 第三章　地址翻译

先用一个地址走完三级页表，再看 TLB 缓存哪些翻译结果。两个核都用过同一个映射时，沿着页表写入、本地同步和远端确认，判断何时可以回收旧页。

1. [处理器怎样用页表和 TLB 翻译地址](./microarch-02a-translation.md)

### 第四章　指令执行与分支预测

先跟踪一条指令从执行到退休，再用存储队列例题，判断一次读取该从哪里取数，以及地址迟到时核心怎样重执行。把视线移到前端后，沿同一次取指学习目标、方向和历史各自怎样参与预测，再用同一个分支例题区分重定向、历史恢复与方向表训练。

1. [乱序执行怎样按程序顺序退休](./microarch-03-execution.md)
2. [加载怎样从存储队列拿到数据](./microarch-03d-load-store.md)
3. [分支预测器怎样决定下一次取指](./microarch-03a-fetch-prediction.md)

### 第五章　计时与证据

让两种访问顺序走同一个求和循环，先分别量程序整体和求和区间，再核对 PMU 记录的事件与实际计数时长。随后把客户端耗时与服务端记录对照，判断差异出现在排队、索引还是其他阶段，并说明给定观察者实际看到了什么。

1. [用计时和计数器分析访存](./microarch-04a-measurement.md)
2. [时间差何时算作侧信道证据](./microarch-04b-observation.md)

### 第六章　共享资源与隔离

把两个工作负载放在一起，先算末级缓存的组内替换怎样影响私有缓存，再沿内存控制器的读缓冲、拆包和重试走到 DRAM，计算行调度怎样改变返回时间。之后从控制组和缓存域查到 CAT 掩码，推导新块能填到哪里。最后从两个逻辑 CPU 的运行队列算出同核配对，看看有任务在排队时为何仍可能空闲。

1. [不同的数据为什么会在末级缓存里互相挤掉](./microarch-06a-shared-cache.md)
2. [未命中次数相同，为什么内存等待更长](./microarch-06b-memory-queue.md)
3. [划分末级缓存的路能隔离什么](./microarch-06c-cat.md)
4. [同核多线程怎样安排任务](./microarch-06d-core-scheduling.md)

### 第七章　DRAM 扰动与硬件错误

1. [DRAM 行被反复打开时，邻近数据为何会翻转](./microarch-07a-rowhammer.md){target=_self}
   - [ECC 怎样知道该翻回哪一位](./microarch-07a-rowhammer.md#rowhammer-ecc-codeword){target=_self}

### 第八章　瞬态执行

先分别读清 Spectre v1 的边界检查和 Meltdown 的权限异常，再沿完整 C 代码逐段追踪训练、瞬态访问、缓存编码和结果恢复。之后再看页表隔离和其他变体，区分漏洞机制、缓解措施与适用范围。

1. [Spectre v1 中的边界检查为何会被越过](./microarch-04d-spectre-v1.md)
2. [Spectre v1：完整 PoC 源码讲解](./microarch-04g-spectre-v1-source.md)
3. [Meltdown 中的越权加载如何影响后续指令](./microarch-04e-meltdown.md)
4. [Meltdown：完整 PoC 源码讲解](./microarch-04h-meltdown-source.md)
5. [进入内核时怎样切换页表](./microarch-04f-page-table-isolation.md)
6. [Spectre 变体中的分支与访存推测](./microarch-04c-variants.md)

读到例题时，先按题设复算一次，再对照手册确认哪些条件可以用于具体处理器。更换行长、替换策略或观察条件时，可以用原来的推理方法重新计算。 [微架构安全学习资料](./microarch-05-sources.md)

## 参考文献

- [Spectre 原论文](https://spectreattack.com/spectre.pdf)
- [Meltdown 原论文](https://meltdownattack.com/meltdown.pdf)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>下一篇</small><strong>缓存行与局部性</strong></span>](./microarch-01-cache.md){.article-nav-card .article-nav-next rel=next}

</nav>
