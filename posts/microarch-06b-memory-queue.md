---
title: 未命中次数相同，为什么内存等待更长
description: 用拆包例题理解读缓冲和重试，再从 DRAM 的行缓冲推算不同调度规则下的列读和返回时刻。
date: '2026-11-25'
tags: [微架构安全, 共享资源, 内存控制器, 测量方法]
book: microarch-00-intro
---

[上一节的计数例题](./microarch-06a-shared-cache.md#llc-evidence)说明，下级接收的读请求数相同，返回时间仍可能不同。本节给 A 两笔已经确定会穿过 LLC 的读取。无论 B 是否同时运行，A 都只有这两笔未命中，读出的数据也相同。接下来追踪每笔请求进入内存控制器、被选中、发出列读以及数据就绪的时刻。

[[toc]]

## 把一次等待拆成排队和服务

请求离开缓存后，还会经过互连、控制器和内存接口。某些接口每个周期能接收或发送的工作有限，控制器要保存未完成请求并决定先做哪一笔。等待可以发生在请求已进队列、尚未获得服务的阶段，也可以发生在服务开始以后。区分这两段，需要给每笔请求记三个时刻。

<div role="region" aria-label="一次请求的三个时刻" style="max-width:100%; overflow-x:auto">

| 记号 | 本节中的含义 |
| --- | --- |
| 到达 | 请求进入共享等待队列 |
| 开始 | 获得本节模型中的服务位置 |
| 完成 | 本笔服务结束，数据可返回 |

</div>

完成减到达是这段路径上的响应时间，也等于到达到开始的排队时间加上开始到完成的服务时间。这里从共享端口的入口开始计时，尚未纳入核心、缓存和互连的其他阶段。

## 按到达顺序算一个满载区间

假设只有一个服务位置，请求按先来先服务，任何一笔开始后都占四个周期。队列容量足够，已经开始的一笔不能被插队打断。时间从 0 开始，记 `[3, 7)` 为从 3 开始到 7 完成。

A 的两笔请求分别在周期 3 和 10 到达。可把它们理解为已知地址的独立读取，两次到达由题设固定。单独运行时，它们在 `[3, 7)` 和 `[10, 14)` 完成服务，响应时间各为 4。

加上 B。B 有三笔请求在周期 0、1、2 到达，按 FIFO 放在 A 的第一笔之前。

<div role="region" aria-label="固定服务时间下的队列" style="max-width:100%; overflow-x:auto">

| 请求 | 到达 | 获得服务的区间 | 排队时间 | 响应时间 |
| --- | ---: | --- | ---: | ---: |
| B0 | 0 | `[0, 4)` | 0 | 4 |
| B1 | 1 | `[4, 8)` | 3 | 7 |
| B2 | 2 | `[8, 12)` | 6 | 10 |
| A0 | 3 | `[12, 16)` | 9 | 13 |
| A1 | 10 | `[16, 20)` | 6 | 10 |

</div>

周期 3 时，B0 正在服务，B1 和 B2 在等待，A0 排在它们之后。周期 10 时，B2 正在服务，A0 又排在 A1 之前。A 发出的仍是两笔请求，每笔占用服务位置的时间也都是 4。A 的第一笔响应延长了 9 周期，第二笔延长了 6 周期，都来自到达后的排队。

把两笔响应时长相加，独占时是 8，共享时是 23。这是两个请求等待时间的和。若问题是从第一笔到达到最后一笔完成的程序窗口，则应算 `14 - 3 = 11` 与 `20 - 3 = 17`。第一笔还未返回时，第二笔已经到达，两段等待有所重叠。由两个时间口径得出的差值不同。

## 从控制器端口走到 DRAM {#memory-controller}

图 1 取自 gem5 Bootcamp。控制器在中央，读队列和写队列各自保存已经接受的请求。把图左侧的省略号接回前一课的 LLC 和互连，把右侧接到 DRAM 接口。左侧决定一笔请求能否进来，右侧决定它何时能发出命令。两边都可能使请求等待。

<figure class="lesson-figure">
<a href="/microarch/gem5-memory-controller.png" aria-label="查看 gem5 内存控制器框图"><img src="/microarch/gem5-memory-controller.png" alt="gem5 内存控制器位于内存系统中，内部有独立的读队列和写队列，左右分别连接上游和内存接口" width="820" height="260" style="filter: grayscale(1); background-color: white" loading="lazy" decoding="async"></a>
<figcaption>图 1　从 gem5 Bootcamp 讲义第 10 页裁出的内存控制器原图。数字用于区分排队请求，正文以 A0、A1、B0、B1 标记例题中的请求。</figcaption>
</figure>

下表沿 gem5 v24.1.0.3 的普通单通道 `MemCtrl` 说明接口。**突发传输**（burst）是一次列读或列写在数据总线上连续传送的一组数据。上层送来一个包，控制器可能要拆成多个突发粒度的条目。例如突发大小为 64 字节时，起始地址在突发边界上的 64 字节请求只需要一个条目，同样长度的请求若从边界后 32 字节开始，就跨了两个突发。

<div role="region" aria-label="控制器各部件保存的状态" style="max-width:100%; overflow-x:auto">

| 位置 | 保存的状态 | 如何推进请求 |
| --- | --- | --- |
| 控制器输入端 | 上层包的地址、大小和读写类型 | 计算需要的条目数，检查对应队列是否有空位 |
| 读队列与写队列 | 内部 `MemPacket`，含源请求、进入时间、地址、大小及译出的 rank、bank、row | 按读写方向、优先级及内存状态挑选，读请求也可能由覆盖它的待写条目满足 |
| DRAM 接口 | 每个存储体当前打开的行，允许读、写、激活及预充电的最早时刻 | 把请求排成行命令和列命令，并给出数据可用时间 |
| 返回队列 | 已调度读突发及其 `readyTime` | 数据就绪后处理返回。上层包拆成多个突发时，等各部分完成才回复 |

</div>

在选定实现中，读侧容量把等待调度的读项与已调度、仍在返回队列里的读项一起计入。例如读缓冲上限是 32 项，读队列占 29，返回队列占 2，再来一个需要两项的包就会被拒绝，因为 $29 + 2 + 2 > 32$。包仍留在上游，要等控制器腾出位置并通知重试。控制器内部从成功进入开始计算的延迟，不会包含这段入队以前的等待。[缓存的 MSHR](./microarch-01a-mshr.md#mshr-trace) 追踪的是更上游的缓存行事务，也可能因为该事务不能前进而占用更久。

### 一个 64 字节读包怎样分成两项 {#memory-read-retry}

沿用 64 字节突发大小。上游现在送来一个从物理地址 `0x1650` 开始的 64 字节读包。它距当前突发边界 `0x1640` 有 16 字节。余下的 48 字节落在当前突发中，另 16 字节落在下一个突发中。控制器按下面两个范围处理，区间右端不包含在请求里。

<div role="region" aria-label="64 字节读包的切分" style="max-width:100%; overflow-x:auto">

| 读项 | 要读的字节范围 | 所在的突发块 |
| --- | --- | --- |
| 第一项，48 字节 | `[0x1650, 0x1680)` | `0x1640` |
| 第二项，16 字节 | `[0x1680, 0x1690)` | `0x1680` |

</div>

一次读包的入队检查按需要的两项一起做。接着用同一个 32 项读缓冲例子。假定这个读包没有可由待写项满足的部分，且期间没有其他新请求进来。

<div role="region" aria-label="读缓冲容量与重试" style="max-width:100%; overflow-x:auto">

| 事件 | 等待调度 | 等待返回 | 两项能否同时进入 |
| --- | ---: | ---: | --- |
| 上游第一次送来读包 | 29 | 2 | 否，共需要 33 项 |
| 一条旧读项被调度 | 28 | 3 | 否，共需要 33 项 |
| 一条旧读项的数据就绪并处理返回 | 28 | 2 | 可以，共需要 32 项 |

</div>

第二行只把一条旧读从读队列移到了返回队列，因此并未腾出读缓冲容量。第三行才移除一项。这个版本的 `MemCtrl` 在处理读返回后通知等待的上游重试。通知表示可以再试一次，实际接收仍要重新核对该包需要的条目数。本例重试成功后，读队列从 28 项增到 30 项，返回队列仍有 2 项。

拆包后，两个读项分别参加调度。它们关联到原来的系统读包，并共用一份分段计数。假设两部分数据分别在第 20 和第 25 周期就绪，第一部分先把已完成数增加到 1。第 25 周期收到另一部分后，已完成数才等于 2，控制器得以安排原包的响应。上游实际看到响应的时刻还会受控制器配置的响应延迟和返回路径影响。

### 写队列覆盖了部分读包，还要发几次列读 {#memory-write-queue-read}

仍读 `[0x1650, 0x1690)`。这次假定读包已通过容量检查，控制器的写队列有一条尚未调度的待写项，覆盖 `[0x1680, 0x16a0)`。它来自一笔已经被控制器接收的写请求。一个拆出的读项能否跳过 DRAM 调度，要看其**整个字节范围**能否由一条待写项覆盖。

<div role="region" tabindex="0" aria-label="读包各项与写队列的覆盖关系" style="max-width:100%; overflow-x:auto">

| 读项 | 与待写项的关系 | 控制器动作 |
| --- | --- | --- |
| `[0x1650, 0x1680)` | 属于另一个突发块 | 加入读队列，仍要等 DRAM 调度 |
| `[0x1680, 0x1690)` | 完全落在 `[0x1680, 0x16a0)` 中 | 记为已满足，不为它发列读 |

</div>

在 gem5 v24.1.0.3 的 `addToReadQueue` 中，控制器先用对齐的突发地址查写队列索引，再比较写项的起止地址。拆包辅助项 `BurstHelper` 一开始就把写队列满足的那一项计入 `burstsServiced`。当第一项的 DRAM 时序完成，完成数从 1 增到 2，控制器才安排原来的 64 字节响应。如果两个读项都被覆盖，则直接走控制器的前端响应路径，无须为此读包安排 DRAM 列读。

#### 写队列能满足一部分，入队检查为何仍可能失败 {#memory-forward-admission}

把上面的覆盖关系放回 32 项读缓冲的容量例子。等待调度有 29 项、等待返回有 2 项，只剩一个位置。这个读包跨两个突发，理论上其中一项会由写队列覆盖，最终只需把另一项放入读队列。

在选定的 gem5 版本中，接收路径先按包的起始偏移和大小算出需要两项，再用这两项检查读缓冲容量。此时 `29 + 2 + 2 = 33`，超过上限，包被拒绝。写队列覆盖判断发生在包通过容量检查之后，所以这一次还没有机会用覆盖结果减少实际排队的读项。

一条旧读完成并退出返回队列后，假设其他状态均不变，重试时占用为 `29 + 1`。这次 `29 + 1 + 2 = 32`，可以接收。控制器随后检查覆盖关系，把第一项放入读队列，第二项记为已由写队列满足。实际新增的待读项为 1。这个例子说明接收时的容量预算和接收后实际需要排队的项数可以不同。

::: details 练习｜只剩一项时改成对齐读取

仍有 29 项等调度、2 项等返回。把新请求改为从 `0x1680` 开始的 16 字节读取，并保持写队列覆盖该范围。容量检查需要几项，结果如何？

读包只落在一个 64 字节突发块内，需要一项。检查得到 `29 + 2 + 1 = 32`，可以接收。随后覆盖检查成功，该读包无需安排 DRAM 列读。
:::

这里的写队列描述的是 gem5 的**时序模型**。接收写包时，`accessAndRespond` 已更新模拟内存的数据。稍后调度的写突发负责模拟 DRAM 的传输时序。读路径用写队列判断可省掉哪些读突发，形成响应时再对原始读包执行数据访问。要解释真实控制器，仍需核对其写数据实际保存在何处、如何交给读路径。

::: details 练习｜只覆盖后半段的 8 字节

把待写项改为 `[0x1688, 0x16a0)`。第二个读项和待写项仍在同一个突发块。这次控制器给原读包安排几次列读？

第二个读项还包含 `[0x1680, 0x1688)`。覆盖检查失败，因此两个读项都进入读队列，分别安排一个突发读。控制器不会再把第二项切成已覆盖和未覆盖的更小片段。
:::

## 打开的一行为什么会改变服务顺序 {#memory-row}

一个内存通道有一组命令与数据通路。通道内可有多个 **rank**，每个 rank 是一起响应命令的一组器件。**bank**（存储体）是 rank 内可分别打开行的单位。**row**（行）指定存储体内一整行，列地址再从该行中选出要传输的部分。同一个 bank 的**行缓冲**保留当前激活的一行。存储体能否直接访问目标列，要看请求中的 row 和当前打开的行是否相同。

<div role="region" aria-label="行状态与 DRAM 命令" style="max-width:100%; overflow-x:auto">

| 目标 bank 的状态 | 访问目标列之前需要做什么 |
| --- | --- |
| 打开的就是目标行 | 等列命令与共享总线允许，发 `RD` |
| 当前没有打开的行 | 发 `ACT` 激活目标行，等到行到列的最小间隔满足，再发 `RD` |
| 打开了另一行 | 先用 `PRE` 关闭旧行，再发 `ACT` 打开新行，最后发 `RD` |

</div>

列命令可发的时刻还会受先前命令、总线方向切换和刷新影响。例如 bank 里虽然打开了目标行，它仍可能要等列命令的最早允许时间，不能立即跟上当前可用的总线时隙。另一个 bank 的激活则可能提前完成，准备时间与正在传输的数据重叠。

gem5 默认的 `frfcfs` 会检查这些条件。在选定的读写队列和优先级内，源码遍历候选，优先找能接上目标时隙的行命中。如果暂时没有这样的命中，也会判断某些 bank 的 `PRE`／`ACT` 能否提前完成，以及哪一项最早能推进。选择策略比较的是具体时刻和资源，不能从“行命中”三个字直接推出总会先服务。下面先去掉这些额外变量，只保留后到的行命中可以先发这一条规则。

### 从物理地址算出行，再排请求 {#memory-address-mapping}

仍让 A 有两笔已经越过 LLC 的读请求，B 也提交两笔读请求。为便于手算，题设只用一个通道、一个 rank，地址低 6 位是 64 字节突发内偏移，接着 2 位选列，再用 1 位选 bank，高位选 row。这是下表使用的地址映射。实际 gem5 的映射由 `addr_mapping` 及通道、行缓冲等参数共同决定。

<div role="region" aria-label="四个读请求的地址解码" style="max-width:100%; overflow-x:auto">

| 请求 | 到达时刻 | 物理地址 | bank | row | 列 |
| --- | ---: | --- | ---: | ---: | ---: |
| A0 | 0 | `0x1600` | 0 | 11 | 0 |
| A1 | 1 | `0x1640` | 0 | 11 | 1 |
| B0 | 2 | `0x1400` | 0 | 10 | 0 |
| B1 | 3 | `0x1440` | 0 | 10 | 1 |

</div>

以 `0x1640` 为例，`0x1640 >> 6 = 0x59`。`0x59` 的最低两位是 1，下一位是 0，剩余位是 `0xb`，因此地址落在 bank 0、row 11、列 1。A0、A1 是不同的 64 字节块，却可以共用同一个已打开的 DRAM 行。

假设先前的工作使 bank 0 保留着 row 10，端口在第 4 周期才开始从这批请求选取。所有请求只读、同优先级，每笔都占一个串行服务位置直到完成。若目标行已经打开，从选中到数据可用花 2 周期。行冲突时另加 `PRE` 的 2 周期和 `ACT` 到列命令的 3 周期，总共 7 周期。处理完成后保留最后打开的一行。这组数值只用来复算调度影响。

<div role="region" aria-label="FIFO 与行命中优先的返回时刻" style="max-width:100%; overflow-x:auto">

| 规则 | 第 4 周期先选谁 | 其余完成顺序 | A0、A1 完成时刻 | 最后一笔完成 |
| --- | --- | --- | --- | ---: |
| FIFO | A0，先关闭 row 10，再打开 row 11 | A1、B0、B1 | 11、13 | 22 |
| 行命中优先，同类先到先选 | B0，直接读 row 10 的列 | B1、A0、A1 | 15、17 | 17 |

</div>

第一行先支付一次换行成本。A0 从 4 到 11，A1 沿用 row 11，于 13 完成。B0 得重新打开 row 10，于 20 完成，B1 再用 2 周期。第二行的 B0、B1 在 6、8 完成，随后只切换一次行，A0、A1 因此到 15、17 才完成。两条规则下 A 都有两次 LLC 未命中，而且读出同样的数据。行命中优先使整批提前 5 周期结束，A 的两笔却各晚了 4 周期。若只报告整批完成时间，就看不出 A 的等待增加。

### 两个存储体怎样共用一条数据总线 {#memory-bank-overlap}

上一题的请求都落在同一个 bank，并按题设逐笔完成。现在仍用原来的地址映射，在同一通道、同一 rank 内安排两个读请求。A 读 `0x1600`，它在 bank 0 的第 11 行。B 读 `0x1700`，相差的 `0x100` 恰好改变了选 bank 的那一位，因此落在 bank 1 的第 11 行。两者的行号相同，各自对应独立的行缓冲。题设 bank 0 已打开第 11 行，bank 1 尚未激活任何行。

控制器为每个 bank 记录当前打开的行及列命令最早可发的时刻。它还要为整个通道安排数据传输。以本篇对照的 gem5 为例，`Bank::openRow` 和 `Bank::rdAllowedAt` 分别对应前两项。在安排列读时，DRAM 接口会同时考虑 bank 的列命令约束和下一次突发可用的时刻。列读确定下来后，还会根据 bank group 等条件更新后续命令的最早时刻。本题只保留打开行的等待与共享数据总线的约束。

设 A、B 都在第 0 周期进入队列，A 的列读端口到第 2 周期才可用。命令接口每周期至多接收一条命令，允许在第 1 周期向 bank 1 发 `ACT`。从 `ACT` 到列读至少隔 3 周期。每条 `RD` 在发出 4 周期后开始送数据，连续占用数据总线 4 周期。区间右端表示整笔突发已经传完。

| 周期或区间 | bank 0 的 A | bank 1 的 B | 通道的数据总线 |
| --- | --- | --- | --- |
| 1 | 第 11 行保持打开 | 发 `ACT`，准备第 11 行 | 空闲 |
| 2 | 发 `RD` | 激活后的等待尚未结束 | A 预约 `[6, 10)` |
| 4 | 等数据开始返回 | 已满足激活到列读的间隔 | B 若现在发 `RD`，其 `[8, 12)` 会和 A 重叠 |
| 6 | 开始传输 A 的数据 | 发 `RD` | B 预约紧接 A 的 `[10, 14)` |
| 10 | A 的突发传完 | 开始传输 B 的数据 | 从 A 切到 B |
| 14 | 已完成 | B 的突发传完 | 空闲 |

B 的行准备在 A 传完前已经结束。计算 B 的最早列读时，要取两个下限中的较大者。行准备要求第 4 周期以后（含第 4 周期），数据总线要求 B 第一个数据不早于第 10 周期，所以 B 的 `RD` 至少要等到 `10 − 4 = 6`。第 4 到第 6 周期之间的等待来自通道上的传输安排。表中的第 14 周期只表示 DRAM 突发结束，送回上游仍要算控制器和互连的延迟。

把 A 的列读推迟到第 4 周期，其他参数和命令接口限制不变。B 的 `ACT` 仍在第 1 周期发出，第 4 周期行已准备好；A 在第 4 周期发 `RD`，预约数据总线 `[8, 12)`。若控制器按 A 先、B 后安排列读，B 的数据不能早于第 12 周期开始，因此 B 最早在第 8 周期发 `RD`，传输区间为 `[12, 16)`。B 的行准备没有变，等待却随 A 的总线预约延长。

此题把 A 先于 B 发列读写进调度条件。若允许控制器重新选择顺序，就必须先给出新的命令安排，再检查每个 bank 的约束和数据区间是否冲突；不能直接把旧表中的完成时刻搬过去。不同 bank 能重叠行准备，通道上仍只有题设的一条数据总线。

#### 写完转读，为什么总线空着还要等

沿用一条共享数据总线，将 A 改为列写，仍在第 2 周期发命令。设写命令到首批数据延迟为 2 周期，读命令到首批数据延迟仍为 4 周期，两种突发都占 4 周期。B 所在的行到第 4 周期准备好。另设本题从写数据传完到读数据开始至少要空 3 周期，以便数据总线从控制器发送切换到 DRAM 发送。这些数字是题设。

| 要检查的约束 | 算出的时刻 |
| --- | --- |
| A 何时占用数据总线 | `[2 + 2, 2 + 2 + 4) = [4, 8)` |
| B 的读数据最早何时开始 | `8 + 3 = 11` |
| 由数据总线推回 B 的列读下限 | `11 − 4 = 7` |
| 再考虑 B 的行准备 | `max(7, 4) = 7`，故读数据占用 `[11, 15)` |

第 7 周期可以发读命令，此时写数据仍在传输。读命令经命令接口送出，四周期后读数据才开始占总线。如果改成写完即可转读，B 可在第 4 周期发命令，在第 8 周期接上写数据，最后于第 12 周期传完。

在 gem5 的 DRAM 接口里，列命令安排后会更新各 bank 的 `rdAllowedAt` 和 `wrAllowedAt`。写转读和读转写分别计算，具体下限还取决于 bank group、rank 和配置。读源码时应先看一个参数约束的是命令间隔还是数据区间，再决定它加在哪个时刻上。

::: details 练习｜把 A 的列读提前到第 0 周期

只把 A 的 `RD` 提前两周期，B 仍在第 1 周期激活。B 的最早列读和突发结束时刻分别是多少？

A 改用 `[4, 8)`。B 最早在第 4 周期满足本地行约束，若此时发 `RD`，它会从第 8 周期起使用总线，正好接在 A 后面。因此 B 在第 4 周期发列读，第 12 周期传完。
:::

### 一个读包跨两个存储体，要等哪一半 {#memory-split-bank-response}

把上题的 A、B 改成**同一个上层读包**拆出的两项。包从 `0x16e0` 开始，长度 64 字节。前 32 字节位于突发块 `0x16c0`，后 32 字节位于突发块 `0x1700`。按照本例的地址划分，前一块是 bank 0、row 11、列 3，后一块是 bank 1、row 11、列 0。此处包只需要两块各自的一半，但数据总线仍要传输两个完整突发。控制器按[前文的拆包规则](#memory-read-retry)保留两条读项，以及它们共同对应的原始请求。

假定仍采用上一题的时序，bank 0 的列读在第 2 周期发出，数据总线占用 `[6,10)`。bank 1 在第 1 周期激活，虽然第 4 周期已允许列读，却要等到第 6 周期才能发出，以便占用紧随其后的 `[10,14)`。

| 时间 | 发生的事 | 控制器能否回复原包 |
| ---: | --- | --- |
| 2 | 先调度 bank 0 的列读 | 不能，两部分均未完成 |
| 6 | 调度 bank 1 的列读，bank 0 的第一批数据开始传输 | 不能，第一项尚未传完 |
| 10 | 第一项传输结束，分段完成数变成 1 | 不能，还缺后 32 字节 |
| 14 | 第二项传输结束，分段完成数达到 2 | 可以安排原包的响应 |

因此，这笔上层请求等待的是两个完成时刻中较晚的一个。不能把 A 的 10 周期和 B 的 14 周期相加，因为它们从同一时刻起算，行准备和突发等待有重叠。第 14 周期也只是控制器具备回复条件的时刻。若题设再给 2 周期的固定响应延迟，且互连没有排队，调用方在第 16 周期收到完整数据。若包的最后 32 字节可以直接由待写队列满足，那么只剩 bank 0 这一笔列读，条件分别提前到第 10 和第 12 周期。

### 刷新怎样打断一次本来会命中打开行的读取 {#memory-refresh}

DRAM 存储单元需要定期刷新。以本篇使用的 gem5 v24.1.0.3 模型为例，控制器为每个 rank 跟踪刷新状态。进入普通刷新路径时，先等这个 rank 的在途工作和各 bank 的关闭条件满足，再预充电关闭打开的行，随后进入刷新。`tREFI` 描述刷新间隔，`tRFC` 则约束刷新命令发出后，还要过多久才能恢复对这个 rank 的访问。前一个参数决定维护操作出现的频率，后一个决定一次操作占用多久。

设 rank 0 的 bank 0 已打开目标行。第 100 ns，控制器为全 rank 刷新发出预充电命令。正好在第 110 ns，有一笔要读目标行的请求到达。题设无其他请求争用，预充电等待 `tRP = 14 ns`，刷新占用 `tRFC = 260 ns`，激活到列读需要 `tRCD = 14 ns`，列读到第一批数据需要 `tCL = 14 ns`。这些数值用于演算时序。

<div role="region" tabindex="0" aria-label="全 rank 刷新与一次读取的时序" style="max-width:100%; overflow-x:auto">

| 时刻 | 控制器或 DRAM 做什么 | 这笔读请求在等什么 |
| ---: | --- | --- |
| 100 ns | 预充电，关闭打开的行 | 请求还没到达 |
| 110 ns | 接收读请求 | 先等预充电，再等刷新 |
| 114 ns | 预充电约束满足，发出刷新 | 目标 rank 进入刷新时段 |
| 374 ns | 刷新结束，对目标行发 `ACT` | 还需等激活完成 |
| 388 ns | 对目标列发 `RD` | 等待数据总线上的返回 |
| 402 ns | 首批数据就绪 | 从请求到达到这一刻用了 292 ns |

</div>

另一次运行若没有这轮刷新，且目标行仍打开、列命令和数据总线都可用，控制器在 110 ns 就能发 `RD`，124 ns 取得首批数据。两次运行都只有一笔需求读取，也都只发一笔列读。发生刷新的那次多等了 278 ns，其中 4 ns 来自剩余的预充电等待，260 ns 用于刷新，另有 14 ns 用于重新打开目标行。完整请求的返回还要算后续数据和控制器响应路径。

::: details 练习｜改为访问同一通道的另一个 rank

让请求仍在第 110 ns 到达，但地址映射到 rank 1。rank 1 未在刷新且目标行已打开。能否把 rank 0 的 `tRFC` 直接加到这笔请求的延迟上？

刷新状态属于 rank 0。rank 1 的请求是否能立即发出，要另外核对共享的命令与数据总线。若这些资源也都允许，本题就可在 110 ns 发列读。知道两个请求共用通道，还需要知道它们是否落到同一个 rank。
:::

### 用 C 复算调度、列读与数据就绪 {#memory-model}

这个 C11 教学模型沿用同一张地址表。程序先按题设切分 `0x1650` 的读包，复算读缓冲容量，再用一个串行服务位置调度 A0、A1、B0、B1。行冲突时从选中到列读前要多等 5 周期，即 `PRE` 后两周期发 `ACT`，再等三周期发 `RD`。数据就绪比 `RD` 晚两周期。每个请求完成后才调度下一项，这是本题规则。实际控制器可以重叠不同请求的命令和数据传输。

保存为 `memory_queue.c`，用 `gcc -std=c11 -O2 -Wall -Wextra memory_queue.c -o memory_queue` 编译，再运行 `./memory_queue`。Windows 上运行 `./memory_queue.exe`。

~~~c
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>

enum { BURST_BYTES = 64, READ_LIMIT = 32 };

typedef struct {
    unsigned bank, row, column;
} Location;

typedef struct {
    const char *name;
    unsigned arrival, address;
    bool from_a;
} Read;

static Location decode(unsigned address)
{
    unsigned burst = address / BURST_BYTES;
    return (Location){(burst >> 2) & 1u, burst >> 3, burst & 3u};
}

static unsigned split_read(unsigned address, unsigned size)
{
    unsigned count = 0;
    while (size != 0) {
        unsigned room = BURST_BYTES - address % BURST_BYTES;
        unsigned length = size < room ? size : room;
        printf("part %u: addr=0x%04x bytes=%u\n", ++count, address, length);
        address += length;
        size -= length;
    }
    return count;
}

static void admission(unsigned waiting, unsigned returning, unsigned needed)
{
    const char *result = waiting + returning + needed <= READ_LIMIT ?
                         "accept" : "retry later";
    printf("read=%u response=%u needed=%u: %s\n", waiting, returning,
           needed, result);
}

static void schedule(const Read reads[4], bool row_first)
{
    bool done[4] = {false};
    unsigned now = 4;
    unsigned open_row = 10;
    unsigned all_time = 0, a_time = 0, a_reads = 0;
    puts(row_first ? "Row hits first" : "FIFO");

    for (unsigned remaining = 4; remaining != 0; --remaining) {
        int chosen = -1;
        unsigned best_rank = 0;
        for (int i = 0; i < 4; ++i) {
            Location location = decode(reads[i].address);
            if (location.bank != 0 || reads[i].arrival > 4)
                exit(EXIT_FAILURE);
            if (done[i])
                continue;
            unsigned rank = row_first && location.row != open_row ? 1u : 0u;
            if (chosen < 0 || rank < best_rank ||
                (rank == best_rank && reads[i].arrival < reads[chosen].arrival)) {
                chosen = i;
                best_rank = rank;
            }
        }

        Location location = decode(reads[chosen].address);
        unsigned rd = now + (location.row == open_row ? 0u : 5u);
        open_row = location.row;
        unsigned ready = rd + 2;
        printf("%s col=%u select=%u RD=%u ready=%u\n",
               reads[chosen].name, location.column, now, rd, ready);
        unsigned elapsed = ready - reads[chosen].arrival;
        all_time += elapsed;
        if (reads[chosen].from_a) {
            a_time += elapsed;
            ++a_reads;
        }
        done[chosen] = true;
        now = ready;
    }
    printf("mean arrival-to-ready: all=%.1f A=%.1f\n",
           all_time / 4.0, (double)a_time / a_reads);
}

int main(void)
{
    unsigned needed = split_read(0x1650, 64);
    admission(29, 2, needed);
    admission(28, 3, needed);
    admission(28, 2, needed);

    const Read reads[] = {
        {"A0", 0, 0x1600, true}, {"A1", 1, 0x1640, true},
        {"B0", 2, 0x1400, false}, {"B1", 3, 0x1440, false}
    };
    schedule(reads, false);
    schedule(reads, true);
}
~~~

~~~text
part 1: addr=0x1650 bytes=48
part 2: addr=0x1680 bytes=16
read=29 response=2 needed=2: retry later
read=28 response=3 needed=2: retry later
read=28 response=2 needed=2: accept
FIFO
A0 col=0 select=4 RD=9 ready=11
A1 col=1 select=11 RD=11 ready=13
B0 col=0 select=13 RD=18 ready=20
B1 col=1 select=20 RD=20 ready=22
mean arrival-to-ready: all=15.0 A=11.5
Row hits first
B0 col=0 select=4 RD=4 ready=6
B1 col=1 select=6 RD=6 ready=8
A0 col=0 select=8 RD=13 ready=15
A1 col=1 select=15 RD=15 ready=17
mean arrival-to-ready: all=10.0 A=15.5
~~~

`schedule` 每次在未完成的项中挑出一个请求。FIFO 只比较到达时刻。行命中优先先比较是否命中已打开的行，再用到达时刻打破并列。选中后，`rd` 反映准备行所耗的时间，`ready` 再加上列读之后的两周期。以 FIFO 的 A0 为例，它在第 4 周期选中，第 9 周期才发列读，第 11 周期数据就绪。若度量从入队到选中，A0 等了 4 周期。若量到列命令，同一请求则等了 9 周期。

::: details 练习：B 改读 A 所在的行

只把 B0、B1 的地址改为 `0x1680` 和 `0x16c0`，其余条件不变。四笔请求仍然访问不同的块，分别在 bank 0、row 11 的四列。一开始没有行命中，所以先选最早到达的 A0。A0 打开 row 11 后，其余三笔都命中当前行，再按到达顺序处理。四笔依次在 11、13、15、17 周期完成，两种调度在这个输入下给出相同结果。
:::

## 怎样分辨请求变多和请求变慢 {#memory-evidence}

同一个名称在不同对象下也可能有不同定义。`MemCtrl.readBursts` 计数控制器拆出的读项，包含由写队列满足的项。`DRAMInterface.readBursts` 计数实际在 DRAM 接口调度的读突发。以本节的 64 字节读包为例，接收时 `MemCtrl.readReqs` 增加 1，`MemCtrl.readBursts` 增加 2。若两个部分都需要读 DRAM，之后 `DRAMInterface.readBursts` 增加 2。改成其中一段被待写项完整覆盖，则实际读 DRAM 的只有另一段，后一个计数只增加 1。

`DRAMInterface.avgQLat` 从读项入队算到列命令，准备行所花的时间也在区间里。`avgMemAccLat` 从入队算到突发数据就绪。这两个平均数都按实际调度的 DRAM 读突发计算。代入题设里的 A0 和 A1，FIFO 下入队到列命令分别为 `9 - 0 = 9` 和 `11 - 1 = 10` 周期。行命中优先时分别为 `13 - 0 = 13` 和 `15 - 1 = 14` 周期。

用同一组时间戳还可以算出两个不同口径的平均值。下表的延迟均为数据就绪时刻减去该请求的入队时刻，单位为周期。四项的顺序固定为 A0、A1、B0、B1。

| 调度规则 | 四笔请求各自的延迟 | 四笔的平均值 | 仅 A 的平均值 |
| --- | --- | ---: | ---: |
| FIFO | 11、12、18、19 | 15 | 11.5 |
| 行命中优先 | 15、16、4、5 | 10 | 15.5 |

行命中优先提前完成了 B 的两笔读取，全通道每笔平均少等 5 周期，A 却平均多等 4 周期。C 程序用 `from_a` 保存请求来源，并分别累加两个分子。观察 A 时，分母是 A 的两笔已完成请求。在真实测量里，应将开头和结尾的在途请求一并交代清楚，否则同一段时间内的接收计数和完成计数可能并不对应同一批请求。

这张表以突发就绪为终点。若上游请求被拆成前面例题的两段，第一段在第 20 周期就绪、第二段在第 25 周期就绪，原包最早要到第 25 周期才齐备。只把第一段提前到第 16 周期，突发平均就绪时刻从 22.5 变成 20.5，原包齐备时刻仍为 25。核对 LLC 看到的读响应时，要按原包把各段重新合起来，再计入控制器响应延迟和回程。

再给回程加入一组明确的教学条件。两段仍分别在第 20、25 周期就绪，全部齐备后控制器经过 2 周期准备响应，响应在第 27 周期到达共享返回端口。端口此时正被另一笔响应占用到第 30 周期，本响应从第 30 周期开始传输，占用 3 周期；假设回程互连随后不再排队，调用方在第 33 周期收到它。由此可把第 25 至 33 周期拆成准备 2 周期、端口等待 3 周期和传输 3 周期。各段能否这样相加取决于题设是否允许重叠，本例明确规定它们依次发生。

如果仅将第一段提前到第 16 周期，第二段、端口占用和其他时刻都不变，调用方仍在第 33 周期收到响应。如果第二段也提前到第 21 周期，响应在第 23 周期到达端口，按题设此时端口空闲，传输到第 26 周期结束。较早的数据就绪使它赶上另一笔响应占用端口之前的空档，因此调用方完成时刻从 33 提前到 26，而不是只提前 4 周期。分析真实系统时需要同时取得各阶段的时刻，不能仅凭平均 DRAM 延迟推断回程队列的变化。

假设一个公开数组求和程序，单独运行和并行运行时，都观察到近似相同的 LLC 需求读未命中次数。但并行运行时平均内存读完成时长与批处理耗时都上升。这样的观察与服务竞争相符，仍要核对延迟是从哪一级开始计的，两组请求是否去了同一内存节点，以及调度和核心频率有没有变化。

再做一组独立对照。把并行负载从大量访存改为主要做寄存器运算，同时保持求和程序与比较的时间窗口。若求和程序的下级请求数近似相同，且额外等待随着下级流量消失，服务端竞争的解释更有支持。不同处理器能提供的队列深度、请求来源或时延事件不同，可结合[计时和计数器一节](./microarch-04a-measurement.md)选择计数口径。

::: details 练习：只让 B 发一笔请求
去掉 B1、B2，B0 的到达时刻不变，仍占用四个周期。重新计算 A 的两笔完成时刻和排队时间。

B0 在 `[0, 4)`。A0 到达时还需等一周期，随后在 `[4, 8)` 完成服务。A1 在 10 到达时接口已经空出，于 14 完成。排队时间分别为 1 和 0。B 的工作虽仍存在，第二笔 A 请求已经看不到它带来的排队。
:::

## 参考文献

- [gem5 v24.1.0.3 的列命令下限、突发完成和总线方向切换](https://github.com/gem5/gem5/blob/v24.1.0.3/src/mem/dram_interface.cc#L361-L437)

- [Maryam Babaie，gem5 Bootcamp 课程讲义，第 10 页内存控制器框图](https://github.com/gem5bootcamp/gem5-bootcamp-env/blob/51590ae00b0e451c9b6a8854addbb94128ab4cac/assets/slides/extra-KVM-with-m5-and-Memory-System.pdf#page=10) · [CC BY 4.0 许可](https://github.com/gem5bootcamp/gem5-bootcamp-env/blob/51590ae00b0e451c9b6a8854addbb94128ab4cac/LICENSE) · 本文裁出原图并以灰度显示。
- [gem5 MemCtrl 实现，读取与写入队列及调度参数](https://github.com/gem5/gem5/blob/v24.1.0.3/src/mem/MemCtrl.py)
- [gem5 MemCtrl 调度与请求处理实现](https://github.com/gem5/gem5/blob/v24.1.0.3/src/mem/mem_ctrl.cc)
- [gem5 MemCtrl 的读项完成与重试通知](https://github.com/gem5/gem5/blob/v24.1.0.3/src/mem/mem_ctrl.cc#L451-L513)
- [gem5 MemCtrl 的写队列覆盖检查与分段完成计数](https://github.com/gem5/gem5/blob/v24.1.0.3/src/mem/mem_ctrl.cc#L176-L278)
- [gem5 MemPacket 的字段](https://github.com/gem5/gem5/blob/v24.1.0.3/src/mem/mem_ctrl.hh#L98)
- [gem5 DRAMInterface 的行状态、FR-FCFS 和突发统计](https://github.com/gem5/gem5/blob/v24.1.0.3/src/mem/dram_interface.cc#L60)
- [gem5 v24.1.0.3 的 rank 刷新状态机](https://github.com/gem5/gem5/blob/v24.1.0.3/src/mem/dram_interface.cc#L1160)
- [Intel 对内存带宽分配与监测的说明](https://www.intel.com/content/www/us/en/developer/articles/technical/introduction-to-memory-bandwidth-allocation.html)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>上一篇</small><strong>不同的数据为什么会在末级缓存里互相挤掉</strong></span>](./microarch-06a-shared-cache.md){.article-nav-card .article-nav-prev rel=prev}

[<span class="article-nav-text"><small>下一篇</small><strong>划分末级缓存的路能隔离什么</strong></span>](./microarch-06c-cat.md){.article-nav-card .article-nav-next rel=next}

</nav>
