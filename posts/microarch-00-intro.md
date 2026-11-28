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

读者应会基本的十六进制运算，知道寄存器、地址和数组的含义。通读可以按下面的目录顺序。带着具体问题查阅时，可以从三条路线进入。

- **追踪一笔访存。** 从[缓存怎样查找和替换一行](./microarch-01b-cache-mapping.md#cache-lookup)找出它所在的组和路。未命中时，跟着 [MSHR 的读请求](./microarch-01a-mshr.md#mshr-trace)看下一级返回，再看[脏行怎样移交给下一级](./microarch-01e-cache-stores.md#write-back-buffer)。
- **解释两个程序同时运行时为什么变慢。** 先读[一致性中的写权限交接](./microarch-01f-coherence.md#mesi-upgrade)，再看[内存控制器的行调度](./microarch-06b-memory-queue.md#memory-row)。这两篇分别给出缓存行交接和 DRAM 服务顺序的例题，可以结合[计时窗口](./microarch-04a-measurement.md#pmu-window)确定测量覆盖了哪段等待。
- **判断指令为何先执行、后来又撤销。** 从[页表权限](./microarch-02a-translation.md#sv39-permissions)和[乱序退休](./microarch-03-execution.md#ooo-retire)学起，再看[分支预测走错以后怎样恢复](./microarch-03a-fetch-prediction.md#predict-recovery)。有了这些先修知识，再读瞬态执行相关内容。

## 目录 {#book-contents}

### 第一章　访存局部性与高速缓存

先沿字节地址算出缓存组、标记和行内偏移，再按访问次序判断哪些块会留到下一次读取。数组扫描先练习数块，矩阵例题再加上组相联的限制。读到两张矩阵时，要把同一循环中所有落到该组的块都计入。

随后跟一笔未命中走到 MSHR，分别追踪读写权限、要取得的旧字节和随后写回的数据。两笔写能否合并，取决于发包前已经覆盖哪些字节，以及其间是否有读请求。学完单核路径，再把同一条行放进两个核的私有缓存，推导失效确认与最新数据的去向，最后用计数器例题定位伪共享的字段。

1. [缓存行与局部性](./microarch-01-cache.md)
   - [一次加载跨过缓存行边界时](./microarch-01-cache.md#locality-cross-line)
   - [同一个数组扫描两遍，为什么第二遍还要取行](./microarch-01-cache.md#locality-second-pass)
2. [缓存怎样查找和替换一行](./microarch-01b-cache-mapping.md)
   - [四路树形替换怎样选出牺牲行](./microarch-01b-cache-mapping.md#cache-tree-plru)
   - [一级缓存怎样与地址翻译并行](./microarch-01b-cache-mapping.md#cache-vipt)
   - [同一份数据有两个虚拟地址时，写入改了哪一行](./microarch-01b-cache-mapping.md#cache-vipt-alias-store)
   - [操作系统怎样让两个映射落到同一组](./microarch-01b-cache-mapping.md#cache-vipt-coloring)
3. [矩阵按行读和按列读的缓存开销](./microarch-01d-cache-traversal.md)
   - [把行跨度换成组号](./microarch-01d-cache-traversal.md#matrix-stride-sets)
   - [组号绕一圈要多少行](./microarch-01d-cache-traversal.md#matrix-set-period)
   - [不改矩阵布局，也能减少按列读取的未命中](./microarch-01d-cache-traversal.md#matrix-row-tiles)
   - [同一个循环读两张矩阵，为什么每批只能留四行](./microarch-01d-cache-traversal.md#matrix-two-stream-tiles)
4. [缓存未命中后如何合并请求](./microarch-01a-mshr.md)
   - [沿主例数一数槽位](./microarch-01a-mshr.md#mshr-capacity-trace)
   - [读请求发出后，同一行又来了写请求](./microarch-01a-mshr.md#mshr-read-then-write)
   - [取得写权限后，哪些字节真正变了](./microarch-01a-mshr.md#mshr-write-byte-trace)
   - [下一级拒收以后，重试还会发原来的包吗](./microarch-01a-mshr.md#mshr-send-retry)
   - [跨行加载的两部分可以各走一条路](./microarch-01a-mshr.md#mshr-split-load)
5. [写入缓存后，数据什么时候送到下一级](./microarch-01e-cache-stores.md)
   - [两笔局部写什么时候能省去整行读取](./microarch-01e-cache-stores.md#store-whole-line)
   - [写回被拒绝后，同一行的读取要等到哪一步](./microarch-01e-cache-stores.md#write-back-retry)
   - [被替换的行怎样重新组成写回地址](./microarch-01e-cache-stores.md#writeback-address)
   - [写缓冲报满时，已经在途的填充怎么办](./microarch-01e-cache-stores.md#write-back-reserve)
6. [缓存一致性：两个核怎样读到最新值](./microarch-01f-coherence.md)
   - [失效确认比目录回复先到怎么办](./microarch-01f-coherence.md#mesi-ack-arrival)
   - [目录记着共享者，它却已经换出了这条行](./microarch-01f-coherence.md#mesi-stale-sharer)
   - [MOESI 怎样处理多个核读取同一条脏行](./microarch-01f-coherence.md#moesi-owned)
   - [两个地址都一致，为什么还会读到两个旧值](./microarch-01f-coherence.md#coherence-store-buffering)
   - [怎样证明栅栏排除了两个旧值](./microarch-01f-coherence.md#coherence-fence-order)
   - [生产者怎样用就绪标志发布数据](./microarch-01f-coherence.md#coherence-release-acquire)
   - [网卡收包后，CPU 何时能读到新数据](./microarch-01f-coherence.md#coherence-dma)
7. [两个线程改不同字段，为什么缓存行还会来回传](./microarch-01g-false-sharing.md)
   - [字段间隔和起始地址怎样一起决定行号](./microarch-01g-false-sharing.md#sharing-alignment)
   - [数组已经对齐，为什么不同元素仍会共用一行](./microarch-01g-false-sharing.md#sharing-array-stride)
   - [四个线程怎样分工，才不会共写一行](./microarch-01g-false-sharing.md#sharing-four-workers)
   - [只读字段也会被迫重新加载](./microarch-01g-false-sharing.md#sharing-read-write)

### 第二章　硬件预取器

先从三次访问算出步长和候选地址，再跟踪候选怎样过页、排队和填充。随后到 RTL 一节，逐拍看缓存接收与拒绝怎样改变暂存槽。

1. [硬件预取器怎样工作](./microarch-02-prefetch.md)
   - [两个请求者执行同一条加载时怎样选表](./microarch-02-prefetch.md#prefetch-requestor-tables)
   - [访问方向反转后，步长表怎样重新学习](./microarch-02-prefetch.md#prefetch-reverse)
   - [MSHR 还有空项，为什么预取仍要等](./microarch-02-prefetch.md#prefetch-mshr-reserve)
   - [预取到正确的行，为什么未命中数没降](./microarch-02-prefetch.md#prefetch-pollution)
   - [综合题｜一次提前填充会多传多少数据](./microarch-02-prefetch.md#prefetch-dirty-traffic)
2. [预取器和数据缓存怎样握手](./microarch-02d-prefetch-rtl.md)
   - [核心请求重试时，步长器在哪一拍看到地址](./microarch-02d-prefetch-rtl.md#rtl-snoop-retry)
   - [两条访问流交错时，为什么历史计数还能上升](./microarch-02d-prefetch-rtl.md#rtl-interleaved-streams)
   - [逐拍跟踪一条被拒绝的请求](./microarch-02d-prefetch-rtl.md#rtl-trace)
   - [P 重试时，核心恰好又有请求](./microarch-02d-prefetch-rtl.md#rtl-retry-priority)

### 第三章　地址翻译

先用一个地址走完三级页表，再区分 TLB 保存的最终映射和[页表遍历缓存保存的中间结果](./microarch-02a-translation.md#sv39-walk-cache)。两个核都用过同一个映射时，沿着页表写入、本地同步和远端确认，判断何时可以回收旧页。

1. [处理器怎样用页表和 TLB 翻译地址](./microarch-02a-translation.md)
   - [物理地址怎样匹配一条 PMP 规则](./microarch-02a-translation.md#sv39-pmp-tor)
   - [监管者怎样读取用户页](./microarch-02a-translation.md#sv39-sum-mxr)
   - [同一页的 A、D 位怎样变化](./microarch-02a-translation.md#sv39-ad-bits)
   - [换成大页以后，哪些读取还要遍历](./microarch-02a-translation.md#sv39-tlb-reach)
   - [ASID 交给另一套页表时，要清掉哪些翻译](./microarch-02a-translation.md#sv39-asid-reuse)
   - [同一张页表，为什么两个线程的读写权限可以不同](./microarch-02a-translation.md#x86-pkru)
   - [虚拟机里的加载要查几次页表](./microarch-02a-translation.md#sv39-two-stage)
   - [用 C 逐笔数出两阶段的表项读取](./microarch-02a-translation.md#sv39-two-stage-model)
   - [设备怎样用 IOMMU 翻译 DMA 地址](./microarch-02a-translation.md#iommu-device-translation)

### 第四章　指令执行与分支预测

先跟踪一条指令从执行到退休，再用存储队列例题，判断两个不同的虚拟地址何时会[匹配同一笔存储](./microarch-03d-load-store.md#lsu-address-alias)，以及地址迟到时核心怎样重执行。把视线移到前端后，沿同一次取指学习目标、方向和历史各自怎样参与预测，再用同一个分支例题区分重定向、历史恢复与方向表训练。

1. [乱序执行怎样按程序顺序退休](./microarch-03-execution.md)
   - [发射队列怎样唤醒等数据的加法](./microarch-03-execution.md#ooo-issue-wakeup)
   - [两个发射端口空着，为什么只发出一条指令](./microarch-03-execution.md#ooo-issue-ports)
   - [ROB 还有空位，为什么新指令仍进不去](./microarch-03-execution.md#ooo-prf-pressure)
   - [用旧目的编号逐项回滚](./microarch-03-execution.md#ooo-map-rollback)
   - [两条加载都出错时，先报告哪一条](./microarch-03-execution.md#ooo-oldest-exception)
2. [加载怎样从存储队列拿到数据](./microarch-03d-load-store.md)
   - [较新的存储只覆盖加载的一部分](./microarch-03d-load-store.md#lsu-partial)
   - [存储已经退休，缓存拒绝后怎样重发](./microarch-03d-load-store.md#lsu-store-nack)
   - [两条加载读同一个字，为什么也要互相检查](./microarch-03d-load-store.md#lsu-load-load-order)
3. [分支预测器怎样决定下一次取指](./microarch-03a-fetch-prediction.md)
   - [索引相同，标记不同会怎样](./microarch-03a-fetch-prediction.md#predict-btb-conflict)
   - [两项预测栈遇到三层调用](./microarch-03a-fetch-prediction.md#predict-ras-capacity)
   - [分支改道时，返回地址栈要恢复什么](./microarch-03a-fetch-prediction.md#predict-ras-repair)
   - [x86 的 RSB 与程序栈各保存什么](./microarch-03a-fetch-prediction.md#predict-rsb)
   - [两个不同上下文查到同一个计数器](./microarch-03a-fetch-prediction.md#predict-alias)
   - [多张方向表同时命中，怎样选出预测](./microarch-03a-fetch-prediction.md#predict-tage-selection)
   - [分支历史怎样参与间接跳转目标选择](./microarch-03a-fetch-prediction.md#predict-bhb)

### 第五章　计时与证据

让两种访问顺序走同一个求和循环，先分别量程序整体和求和区间，再核对 PMU 记录的事件与实际计数时长。随后从共享缓存行和 TLB 组两种观测对象出发，逐步追踪占用、目标访问、重载与计时，判断每种读数能支持哪一步结论，以及页级活动的时间顺序怎样补足单纯组计数的不足。

1. [用计时和计数器分析访存](./microarch-04a-measurement.md)
   - [线程换了 CPU，计数跟着谁走](./microarch-04a-measurement.md#pmu-task-vs-cpu)
   - [分两段读取时，先取差值再换算](./microarch-04a-measurement.md#pmu-delta)
   - [两个事件一起计数，能代表整段执行吗](./microarch-04a-measurement.md#pmu-group-window)
   - [为什么样本会落在加法指令上](./microarch-04a-measurement.md#pmu-sampling-skid)
   - [同样四次未命中，为什么有时只等一轮](./microarch-04a-measurement.md#pmu-miss-overlap)
2. [一次缓存计时能说明什么](./microarch-04b-observation.md)
   - [访问很少发生时，快读数还有多大把握](./microarch-04b-observation.md#evidence-base-rate)
   - [不共享数据页，怎样观察同一个缓存组](./microarch-04b-observation.md#evidence-prime-probe)
   - [侧信道与隐蔽通道怎样区分](./microarch-04b-observation.md#evidence-channel-types)
3. [TLB 命中与未命中怎样留下访问痕迹](./microarch-04i-tlb-observation.md)
   - [用四路模型算出一次 TLB 驱逐](./microarch-04i-tlb-observation.md#tlb-four-way-eviction)
   - [两代 Intel 处理器的 TLB 结构实测](./microarch-04i-tlb-observation.md#tlb-intel-measurement)
   - [二级 TLB 命中和页表遍历怎样分别计数](./microarch-04i-tlb-observation.md#tlb-walk-events)
   - [地址空间标签怎样与替换竞争同时存在](./microarch-04i-tlb-observation.md#tlb-tagged-competition)

### 第六章　共享资源与隔离

把两个工作负载放在一起，先查它们共用哪些结构。沿末级缓存的数据阵列和探测过滤器，分别判断哪个槽用尽、哪一份私有副本需要撤销。请求继续走到内存控制器时，再判断它是在等入队、等打开行，还是等通道切回读方向。之后从控制组和缓存域查 CAT 掩码，推导新块能填到哪里。最后从两个逻辑 CPU 的运行队列算出同核配对，看看有任务在排队时为何仍可能空闲。

1. [不同的数据为什么会在末级缓存里互相挤掉](./microarch-06a-shared-cache.md)
   - [如果被换出的行在私有缓存里已经改过](./microarch-06a-shared-cache.md#llc-dirty-victim)
   - [LLC 还有空路，为什么私有副本也可能被撤销](./microarch-06a-shared-cache.md#llc-snoop-filter)
   - [LLC 没存这条行，哪个核心能提供最新数据](./microarch-06a-shared-cache.md#llc-home-agent)
   - [LLC 未命中是否等于 DRAM 读取？](./microarch-06a-shared-cache.md#llc-miss-source)
2. [未命中次数相同，为什么内存等待更长](./microarch-06b-memory-queue.md)
   - [一个读包跨两个存储体，要等哪一半](./microarch-06b-memory-queue.md#memory-split-bank-response)
   - [两个存储体怎样共用一条数据总线](./microarch-06b-memory-queue.md#memory-bank-overlap)
   - [写队列积满到什么程度，读请求要让路](./microarch-06b-memory-queue.md#memory-write-drain)
   - [刷新怎样打断一次本来会命中打开行的读取](./microarch-06b-memory-queue.md#memory-refresh)
   - [写队列覆盖了部分读包，还要发几次列读](./microarch-06b-memory-queue.md#memory-write-queue-read)
   - [写队列能满足一部分，入队检查为何仍可能失败](./microarch-06b-memory-queue.md#memory-forward-admission)
3. [划分末级缓存的路能隔离什么](./microarch-06c-cat.md)
   - [掩码里的路怎样分布在各个组中](./microarch-06c-cat.md#cat-sets)
   - [同一个任务取指和读数据，分别用哪份掩码](./microarch-06c-cat.md#cat-cdp)
   - [任务换组后，旧组的占用为什么还在](./microarch-06c-cat.md#cat-rmid-occupancy)
   - [把两个域的流量读数换成带宽](./microarch-06c-cat.md#cat-mbm)
4. [同核多线程怎样安排任务](./microarch-06d-core-scheduling.md)
   - [选好新配对以后，旧任务什么时候停下](./microarch-06d-core-scheduling.md#core-ipi-handoff)
   - [同组任务进入内核时，另一侧在做什么](./microarch-06d-core-scheduling.md#core-kernel-context)
   - [同一个任务迁移后，观察范围怎样变化](./microarch-06d-core-scheduling.md#core-migration)
   - [空出一侧后，吞吐会少一半吗](./microarch-06d-core-scheduling.md#core-throughput)

### 第七章　DRAM 扰动与硬件错误

先从 DRAM 单元、字线、位线和行缓冲的结构入手，再把 ACT、READ、PRE 与刷新放进同一条命令序列。用一个 bank 的教学模型区分列读和行激活，最后沿缓存、物理地址映射、TRR 与 ECC 逐项检查 Rowhammer 的成立条件和防护边界。

1. [DRAM 行被反复打开时，邻近数据为何会翻转](./microarch-07a-rowhammer.md)
   - [ACT、READ、PRE 改变的是不同状态](./microarch-07a-rowhammer.md#rowhammer-commands)
   - [用 C 模型区分列读和行激活](./microarch-07a-rowhammer.md#rowhammer-c-model)
   - [防护要对准激活次数、邻行和错误模式](./microarch-07a-rowhammer.md#rowhammer-mitigation)
   - [ECC 怎样知道该翻回哪一位](./microarch-07a-rowhammer.md#rowhammer-ecc-codeword)
   - [读到的数据已经纠正，为什么还要写回](./microarch-07a-rowhammer.md#rowhammer-ecc-scrub)

### 第八章　瞬态执行

先分别读清 Spectre v1 的边界检查和 Meltdown 的权限异常，再沿完整 C 代码逐段追踪训练、瞬态访问、缓存编码和结果恢复。之后再看页表隔离和其他变体，区分漏洞机制、缓解措施与适用范围。

1. [Spectre v1 中的边界检查为何会被越过](./microarch-04d-spectre-v1.md)
2. [Spectre v1 源码讲解](./microarch-04g-spectre-v1-source.md)
3. [Meltdown 中的越权加载如何影响后续指令](./microarch-04e-meltdown.md)
4. [Meltdown 源码讲解](./microarch-04h-meltdown-source.md)
5. [进入内核时怎样切换页表](./microarch-04f-page-table-isolation.md)
6. [Spectre 变体中的分支与访存推测](./microarch-04c-variants.md)

读到例题时，先按题设复算一次，再对照手册确认哪些条件可以用于具体处理器。更换行长、替换策略或观察条件时，可以用原来的推理方法重新计算。 [微架构安全学习资料](./microarch-05-sources.md)

## 参考文献

- [Spectre 原论文](https://spectreattack.com/spectre.pdf)
- [Meltdown 原论文](https://meltdownattack.com/meltdown.pdf)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>下一篇</small><strong>缓存行与局部性</strong></span>](./microarch-01-cache.md){.article-nav-card .article-nav-next rel=next}

</nav>
