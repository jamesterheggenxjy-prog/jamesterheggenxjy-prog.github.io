---
title: 不同的数据为什么会在末级缓存里互相挤掉
description: 从多核共享范围算到同组替换，再看包含关系怎样改变核心实际发出的 LLC 请求。
date: '2026-11-25'
tags: [微架构安全, 共享资源, LLC, 高速缓存]
book: microarch-00-intro
---

两个核心各读自己的数组，物理地址没有重叠。B 读完以后，A 的三条数据中，有两条已经被换出末级缓存（Last-Level Cache，LLC）。A 再次读取这三条数据，会有两次还是三次未命中？如果 A 的私有缓存里还有副本，答案又会怎样？

这两道题要沿同一条路径顺次判断。先看两个核心是否共用 LLC，再由地址算出它们是否竞争同一个组，最后检查核心重新加载时能从哪一级取得数据。先修知识是[地址如何选组](./microarch-01b-cache-mapping.md#cache-address)和[组内替换](./microarch-01b-cache-mapping.md#cache-replacement)。

上一课跟踪的是 TLB 中保存的地址翻译。本课换成数据块，题目直接给出物理地址。这样可以先把翻译阶段的延迟放在一边，单独推导多个核心共用同一数据缓存时会发生哪些替换。

[[toc]]

## 两条访问路径在哪一级汇合 {#llc-domain}

图 1 是 gem5 课程给出的缓存层次示例。各核心先访问自己的 L1，走到互连之后才进入下方的末级缓存。图中画了两个 LLC 块。共享缓存可以有多块存储结构，实际请求按照系统的地址和路由规则被送往相应位置。图示用于说明共享接口所在的位置，后面的题目会直接指定共用一个 LLC 实例。

<figure class="lesson-figure" style="max-width: 550px">
<a href="/microarch/gem5-cache-hierarchy.png" aria-label="查看 gem5 缓存层次原图"><img src="/microarch/gem5-cache-hierarchy.png" alt="gem5 缓存层次中，多个核心先连接各自的 L1，再经互连接到 LLC" width="550" height="387" style="filter: grayscale(1); background-color: white" loading="lazy" decoding="async"></a>
<figcaption>图 1　从各自的 L1 到可能共用的 LLC。上半部是核心侧，下半部经系统互连接向内存控制器。</figcaption>
</figure>

题目比图中多设一级私有 L2。A、B 各有 L1 和 L2，二者的请求经过私有层后汇合到同一个共享 LLC。若 A 在私有 L2 找到了数据，正常的读访问已能返回。分析 LLC 替换时，先从完整访问序列中过滤出实际到了 LLC 的请求。

真实平台上要怎样确认这个汇合点？以 Linux 的缓存接口为例，从 `/sys/devices/system/cpu/` 进入选定的 `cpuN`，在其 `cache/indexK` 目录中读取 `level` 和 `type`，找出目标级别与类型。接着读 `id` 和 `shared_cpu_list`。`indexK` 是该 CPU 的缓存信息目录编号。缓存实例 ID 应在同一级别与类型下比较。下面是一组用于读配置的示意值。

| 逻辑 CPU | 目标级别和类型 | 实例 ID | 可共用这个实例的 CPU |
| --- | --- | ---: | --- |
| 0 | L3，Unified | 0 | 0–3 |
| 2 | L3，Unified | 0 | 0–3 |
| 4 | L3，Unified | 1 | 4–7 |

A 在 CPU 0、B 在 CPU 2 时，可用同一个 LLC 实例解释两者的分配。若 B 移到 CPU 4，B 的新分配使用另一个实例。这个变化能解除两者对同一 LLC 组的争用，但共享的内存节点、互连等仍要另查。若真实处理器在一个 LLC 实例内还有多个分片，也要确认访问是否竞争相同分片上的组。下文使用单片的教学模型，避免让具体平台的路由规则混入例题。

## 六个不同的块怎样争四路 {#llc-set}

设模型的 LLC 有 4 组，每组 4 路，行长为 64 字节。地址均为题设中的物理地址，块号对 4 取余选组。选择同一个组才会争这一组的四个位置。分配时先用空路，组满后换掉最久未用的行，即精确 LRU。

| 工作负载 | 块的名称 | 各块的首地址 | 组号 |
| --- | --- | --- | --- |
| A | A0、A1、A2 | `0x2000`、`0x2100`、`0x2200` | 均为 0 |
| B | B0、B1、B2 | `0x2300`、`0x2400`、`0x2500` | 均为 0 |

例如，`0x2200 / 64 = 0x88`，再对 4 取余得到 0。六条行的块号各不相同，组号却相同。整个 LLC 有 16 个行位置，其他组可以有空位，而这六个块在题设中只能使用组 0 的四路。后面只数实际到达这个 LLC 的读请求，并假定每次填充在下一次读之前完成。

A 先把 A0、A1、A2 读进 LLC。现在让 B 依次读 B0、B1、B2，然后 A 再按原顺序读一次。表中顺序从最久未用排到最近使用。

| 请求 | 结果 | 被换出的块 | 组 0 的状态 |
| --- | --- | --- | --- |
| A 完成第一轮 | 三块已填入 | 无 | A0、A1、A2 |
| B0 | 未命中 | 无，有空路 | A0、A1、A2、B0 |
| B1 | 未命中 | A0 | A1、A2、B0、B1 |
| B2 | 未命中 | A1 | A2、B0、B1、B2 |
| A0 | 未命中 | A2 | B0、B1、B2、A0 |
| A1 | 未命中 | B0 | B1、B2、A0、A1 |
| A2 | 未命中 | B1 | B2、A0、A1、A2 |

B 停止读取时，A2 仍在 LLC。A 先重读 A0 并把它填进 LLC，这一步又挤掉了 A2。于是 B 直接换出了 A 的两块，A 自己的后续填充使第三个块也未命中。用[复用距离](./microarch-01-cache.md#locality-reuse)可以重算这一点。在 A2 的两次访问之间，组 0 先后用了 B0、B1、B2、A0、A1，已经超过了四路能保留的范围。

将 A 的第二轮顺序改成 A2、A1、A0，其他条件不变。B 完成后，组 0 从旧到新为 A2、B0、B1、B2。A2 先命中并成为最近使用；A1 未命中，换出 B0；A0 未命中，换出 B1。这样第二轮是一命中、两未命中，结束时的组状态为 B2、A2、A1、A0。两轮访问涉及的块没有变化，顺序改变了尚存的 A2 能否在被替换前得到复用。

若要据此比较真实程序，还需确认这三次读取是否都到达 LLC。私有缓存若仍能满足其中某次读取，该次访问就不参与本表中的 LLC 替换顺序。下一节将私有缓存和包含关系放回模型，继续判断哪几次请求实际到达共享层。

现在只改 B 的布局，将三个首地址都增加 `0x40`。每个块号加 1，B 就转到组 1。A 不用改动顺序，下一轮三个请求都可在组 0 命中。布局保持同组时，还可以用[CAT 的四路例题](./microarch-06c-cat.md#cat-ways)重算限制填充位置后的结果。这里两个核心访问的是不同物理块。若两个字段实际落在同一物理块，且存在写入，则还需分析一致性和[伪共享](./microarch-01g-false-sharing.md#sharing-layout)。

## LLC 换出一行，私有缓存会怎样 {#llc-inclusion}

回到原布局，把此前略去的私有 L2 加回来。A 第一轮结束后，假定三条干净的数据同时在私有 L2 和 LLC，L1 的细节省略。随后 A 暂停，B 仍然读 B0、B1、B2。这时要看内存层次有没有以下约束。

- **包含式 LLC。** 私有层保留的块必须在 LLC 里也有对应缓存块。LLC 替换 A0 时，控制器要找到私有层的副本，把撤销这些副本作为本次替换的一部分。
- **非包含式 LLC。** 私有缓存可以持有当前未驻留在 LLC 的块。如果实现允许保留，B 换出 LLC 的 A0 后，A 的 L2 仍可命中。更改共享状态或写入相同块时仍需遵守一致性规则。

| 阶段 | 保持包含关系 | 允许私有层继续保留干净副本 |
| --- | --- | --- |
| B 读完三块 | LLC 去掉 A0、A1，A 的 L2 也失去这两块 | LLC 去掉 A0、A1，A 的 L2 保留三块 |
| A 再读 A0 | L2 未命中，LLC 未命中。填充 A0 换出 A2，使其 L2 副本失效 | 在 L2 命中 |
| A 继续读 A1、A2 | 两笔都到 LLC，均未命中 | 两笔均在 L2 命中 |
| A 第二轮到达 LLC 的次数 | 3 | 0 |

表里右列是针对干净只读副本的教学条件。真实非包含式处理器还可能有独立的目录资源和其他替换约束。Intel 对第一代 Xeon Scalable 的技术说明明确指出，在它的非包含式 LLC 中，某块不在 LLC 无法据此判断它也不在私有缓存。读测量结果时，需要分别核实私有层命中、LLC 请求和下级流量。

### 为什么读 B1 会让另一个核的 A0 失效 {#llc-back-invalidation}

B 读 `0x2400`（B1）时，组 0 没有空路，LRU 选中 `0x2000`（A0）。一致性控制器此时查询的是**被换出块 A0 的记录**。记录指向先前读过 A0 的私有缓存，因此控制器向它发送 A0 的失效消息。B 的加载从头到尾请求的都是 B1。比较[为写入而撤销同一行副本](./microarch-01f-coherence.md#mesi-upgrade)的过程，可以看到两种失效各有触发条件。写升级由同一物理行的写请求触发，包含式缓存的这种失效由组内替换触发，即使引发替换的访问只是一次读。

gem5 v24.1.0.3 的 Ruby `MESI_Two_Level` 用私有 L1 和共享 L2 实现了这一过程。假定 A0 是共享层中处于 `SS` 的干净行。L2 为这次换出建立事务记录，按 `Sharers` 给潜在持有者发送失效，并释放数据阵列中的表项。此后 A0 仍有一项在途状态 `I_I`，用来收取失效确认。收齐后才继续下游的替换收尾。**数据阵列的槽位可以先释放，A0 这笔事务还在等待确认。** 这个实现有助于理解前表中 A0 为什么会消失在私有缓存中。前表用的是私有 L2 与共享 LLC，层级名称随系统而变。

### 如果被换出的行在私有缓存里已经改过 {#llc-dirty-victim}

仍让 B1 的读请求选中 A0（`0x2000`）作为牺牲行。现在改变初态：A 已经取得 A0 的写权限，把行首的字从 5 改成 9，最新数据仍在 A 的私有缓存。按照 gem5 Ruby `MESI_Two_Level` 的命名，共享 L2 用 `MT` 记录已有独占持有者，它保存的数据可能已经过期。因而这次失效还必须把最新数据带回来。

| 事件 | 共享层对 A0 的状态 | 最新值 9 在哪里 |
| --- | --- | --- |
| A 写完 A0 | `MT`，目录记着私有持有者 | A 的私有缓存 |
| B1 引发替换 | 分配临时事务记录（TBE），向持有者发失效，释放 A0 的共享层阵列项，转为 `MT_I` | 仍在 A，或已随回复发出 |
| A 的脏数据回复到达 | 最新整行进入 TBE，再向内存侧发送写回，转为 `M_I` | 正由写回事务转交 |
| 内存侧确认写回 | A0 的换出事务结束 | 由下游保存 |

表中的 `MT_I` 和 `M_I` 属于 A0。它们分别等待私有持有者交还数据和内存侧确认写回。共享层腾出了存放 A0 的路，仍须靠 TBE 保留换出进度。若在这两个状态中又收到对 A0 的读请求，源码会暂缓该请求，避免用旧副本回答。B 发起的依旧只是 B1 的读，但为了回收一路，控制器另外处理了 A0 的整行写回。和[经典缓存的写缓冲](./microarch-01e-cache-stores.md#write-back-retry)一样，数据离开阵列之后，仍需追踪谁对最新值负责。

::: details 练习｜A 在 B1 到达前已经交还脏行

若私有缓存先把 A0 的最新数据交给共享 L2，共享层已经进入 `M`，而且题设仍选 A0 为牺牲行，换出还需要等哪一级？

最新数据已在共享 L2，没有私有副本需要撤销。控制器可以直接把共享层保存的整行交给内存侧，进入 `M_I` 等写回确认，省掉等待私有持有者交还数据的步骤。

:::

### LLC 还有空路，为什么私有副本也可能被撤销 {#llc-snoop-filter}

上面的例题由共享层的数据替换触发失效。另一些设计把数据和私有副本的记录分开保存。**探测过滤器**（snoop filter）根据块地址查找可能持有副本的核心及相关状态。控制器据此决定向谁发一致性探测，避免每笔请求都广播。对于同一块地址，LLC 的数据查询可以未命中，过滤器却仍记得某个核心持有该块。遇到这种状态时，要按记录和协议决定向谁取数。

两种结构也分别占空间。下面用一个简化控制器说明记录槽如何先于数据槽用满。仍用 `A0 = 0x2000`、`A1 = 0x2100`、`B1 = 0x2400`。这三个块都选中前文四路 LLC 的第 0 组。另设它们恰好由过滤器的同一个两项集合跟踪。表中只列这一组的状态，两个结构的索引和槽数都是题设。

| 事务完成后 | LLC 第 0 组中的数据 | 过滤器记录的私有副本 |
| --- | --- | --- |
| 初态 | `A1`，还空三路 | `A0 → 核 0`，`A1 → 核 0` |
| 核 1 读 B1，需要安装在私有缓存 | `A1`、`B1`，还空两路 | 换出最老的 `A0` 记录，留下 `A1 → 核 0`、`B1 → 核 1` |

题设规定控制器必须完整跟踪这组的私有副本，且记录满后按 LRU 选旧项。在用新记录覆盖 A0 之前，它根据 A0 的旧记录向核 0 发失效，并等确认。若 A0 已修改，还须先交接最新数据。这样才不会留下控制器已经找不到的副本。B1 填入 LLC 时用的是空路，牺牲的是另一张表里的记录。换一种设计，如果控制器另有广播探测路径，过滤器缺项时也可以用保守的探测弥补，实际约束由一致性协议决定。

Intel 第一代 Xeon Scalable 的事件表提供了 `UNC_CHA_SF_EVICTION`，并区分被替换的记录原先跟踪 M、E 还是 S 状态的行。该事件反映的是为了容纳新记录而发生的过滤器容量替换。核心自己腾出私有缓存位置，从而释放一条跟踪记录，不属于这项计数。因而，即使观察到此计数上升，也还要另外查看 LLC 数据替换和私有层未命中。事件表没有给出上面例题那样的索引或槽数。

::: details 练习｜跟踪 A0 的记录提前释放

核 1 读 B1 前，先让核 0 自己换出干净的 A0，并且让控制器在 B1 到达前收到通知并移除 A0 的记录。B1 仍读同一个地址，安装时还要撤销别的私有副本吗？

过滤器现在只用了一项，B1 可用空项。LLC 中也仍有空路，两个结构都无需选牺牲项。若改成 LLC 已满、过滤器仍有空项，则这次只需处理数据阵列的替换。能否保留被替换块的私有副本，要回到该设计的包含性和探测规则判断。
:::

### LLC 没存这条行，哪个核心能提供最新数据 {#llc-home-agent}

继续看 `A0 = 0x2000`。前面的例题问的是哪份记录会被换出。现在保持 LLC 的空路和过滤器容量不变，改问：核 1 第一次读 A0 时，如果 LLC 中没有这条行，控制器还要查哪里？这需要把请求者、管理一致性的代理、实际保存数据的位置分开。

在第一代 Intel Xeon Scalable 的片上网格中，LLC 分成多个 slice（一片片数据阵列），缓存与归属代理（Caching and Home Agent，CHA）也分布在片上。出自某个核心的请求，要与管理这段地址的 CHA 协调；请求来自哪个核心，并不决定地址交给哪个代理管理。Intel 的技术综述还给出一个具体的数据放置例子：一次所有缓存层都未命中的加载可以从内存取回一行，直接填充请求核的中级缓存（MLC，此处是私有 L2）。之后这条行从 MLC 逐出，如果控制器预计它还会被访问，才可能放入 LLC。处在私有层、未放入 LLC 的行，由探测过滤器记录。由此可以区分三项职责。

| 对象 | 在 A0 这笔访问里承担的职责 |
| --- | --- |
| A0 的归属 CHA | 协调对这个物理块的请求和一致性探测，跟踪这笔事务。 |
| LLC 数据阵列 | 可能存有 A0 的数据。如果只查到标签未命中，还要继续判定数据来源。 |
| 探测过滤器 | 记录 A0 可能在哪个私有缓存中，帮助 CHA 确定探测对象。过滤器记录里没有待返回的数据字节。 |

用一条缓存行推算。题设把 `0x2000` 交给 CHA 2 管理，核 0 和核 1 在同一个插槽。内存起初保存整数 5。在下面三个独立场景里，核 1 的私有缓存都缺 A0。前两个场景里，核 0 先取到这条行，随后写成 9，写入序列已经完成。若 9 仍在核 0 的脏行里，探测记录指向核 0，一致性事务就要取得这个最新值，读内存中的 5 会出错。如果核 0 已经逐出 A0，并且最新值进入 LLC，CHA 可以从 LLC 的数据阵列取得 9。第三个场景里，核 0 的写回已经到达内存，片上不再有 A0 的副本，才轮到内存供数。

| 核 1 发出读请求之前 | LLC 数据查询 | 最新数据在何处 | 核 1 的加载结果 |
| --- | --- | --- | --- |
| A0 的脏副本还在核 0 | 未命中 | 核 0 的私有缓存 | `9`，由持有者供数 |
| A0 已从核 0 逐出，9 留在 LLC | 命中 | LLC 数据阵列 | `9`，由 LLC 供数 |
| A0 的写回已到内存，片上无副本 | 未命中 | 内存 | `9`，由内存供数 |

第一行和第三行的 LLC 数据阵列都未命中，但只有第三行需要从内存读这份数据。第一行中 CHA 2 负责协调探测，核 0 才是数据持有者。核 0 交出 9 后如何降级、内存何时写回，取决于一致性协议。这里的 CHA 编号、三个初态和逐条服务是假设，用来展示职责和数据来源。Intel 的综述说明了分布式 CHA、非包含缓存和探测过滤器，实际切片选择及内部消息不会由上述题设推出。

### 用 C 核对同一次加载的三个数据来源 {#llc-home-model}

模型只走一笔核 1 的读请求。`dirty_owner` 表示过滤器记有脏行的持有者，本题只有核 0；`llc_has_data` 表示共享数据阵列有有效的当前值。主程序分别构造表里的三个初态。`source` 记录最新值从哪里取得，`dram_read` 只统计给这笔数据请求供数的内存读取。模型不执行探测时序，也不模拟目录元数据自身可能产生的内存流量。

```c
#include <stdbool.h>
#include <stdio.h>

struct Line {
    bool dirty_owner;
    bool llc_has_data;
    unsigned owner_value;
    unsigned llc_value;
    unsigned memory_value;
};

static void load_on_core1(const struct Line *line)
{
    const char *source;
    unsigned value;
    unsigned dram_read = 0;

    if (line->dirty_owner) {
        source = "core 0";
        value = line->owner_value;
    } else if (line->llc_has_data) {
        source = "LLC";
        value = line->llc_value;
    } else {
        source = "memory";
        value = line->memory_value;
        dram_read = 1;
    }

    printf("LLC data=%s, source=%s, value=%u, DRAM reads=%u\n",
           line->llc_has_data ? "hit" : "miss", source, value, dram_read);
}

int main(void)
{
    const struct Line private_dirty = {
        .dirty_owner = true, .owner_value = 9, .memory_value = 5,
    };
    const struct Line llc_resident = {
        .llc_has_data = true, .llc_value = 9, .memory_value = 5,
    };
    const struct Line memory_only = {.memory_value = 9};

    load_on_core1(&private_dirty);
    load_on_core1(&llc_resident);
    load_on_core1(&memory_only);
    return 0;
}
```

```text
LLC data=miss, source=core 0, value=9, DRAM reads=0
LLC data=hit, source=LLC, value=9, DRAM reads=0
LLC data=miss, source=memory, value=9, DRAM reads=1
```

把这个例题带到计数器上时，要按事件名称查定义。以 Skylake Server 为例，`MEM_LOAD_L3_MISS_RETIRED.LOCAL_DRAM` 统计由本地 DRAM 满足的退休加载，计数单位是**加载**。`MEM_LOAD_L3_HIT_RETIRED.XSNP_HITM` 统计得到 HitM 响应的退休加载，事件名虽然带有 `L3_HIT`，它描述的响应仍涉及其他核心持有的修改态副本。不能拿事件名前缀与模型的 `LLC data=hit` 逐字相等，也不能把退休加载数直接当成 DRAM 命令数。真实计数还取决于具体型号、数据来源分类和计数窗口。

::: details 练习｜过滤器没变，只有写回完成的位置变了

核 0 已经逐出 A0，过滤器不再跟踪核 0。假定 9 仍留在 LLC，内存还保存 5。仅凭过滤器里找不到 A0，能否去内存取 5？

不能。过滤器没有私有持有者，只排除了题设里的私有副本来源。LLC 数据阵列仍命中，核 1 要从那里取 9。只有 LLC 也不再保留最新数据，并且写回已经到达内存，才能按第三种情况直接读内存。
:::

### 用代码复算私有缓存与 LLC 的替换次数

这段 C 程序用 `private_copy[core][block]` 记录每个核手里有哪些块，用 `llc` 保存每组的块号，最久未使用的块排在最前。`read_block` 先查私有缓存，命中时直接返回，LLC 的顺序保持原样。到达 LLC 后，命中就把该块移到队尾，需要替换就从队首移走一块。在包含式模型里，移除队首还会清除对应的私有副本。这里把发失效到收齐确认合并成一步，两种情况都只模拟干净行，不计在途延迟。

将代码保存为 `llc_model.c`，用 `gcc -std=c11 -O2 -Wall -Wextra llc_model.c -o llc_model` 编译，再运行 `./llc_model`。

~~~c
#include <assert.h>
#include <stdbool.h>
#include <stdio.h>
#include <string.h>

enum { LINE_BYTES = 64, SETS = 4, WAYS = 4, BLOCKS = 256 };
enum Core { A, B, CORES };
enum Source { PRIVATE_L2, LLC, LOWER, SOURCES };

struct CacheSet {
    unsigned blocks[WAYS];
    unsigned used;
};

struct Cache {
    bool inclusive;
    bool private_copy[CORES][BLOCKS];
    struct CacheSet llc[SETS];
};

static enum Source read_block(struct Cache *cache, enum Core core, unsigned address)
{
    const unsigned block = address / LINE_BYTES;
    assert(block < BLOCKS);
    if (cache->private_copy[core][block])
        return PRIVATE_L2;

    struct CacheSet *set = &cache->llc[block % SETS];
    unsigned index = 0;
    while (index < set->used && set->blocks[index] != block)
        ++index;

    const enum Source source = index < set->used ? LLC : LOWER;
    if (source == LOWER && set->used == WAYS) {
        index = 0;
        if (cache->inclusive) {
            const unsigned victim = set->blocks[0];
            for (unsigned other = 0; other < CORES; ++other)
                cache->private_copy[other][victim] = false;
        }
    }

    if (index == set->used) {
        set->blocks[set->used++] = block;
    } else {
        for (; index + 1 < set->used; ++index)
            set->blocks[index] = set->blocks[index + 1];
        set->blocks[index] = block;
    }
    cache->private_copy[core][block] = true;
    return source;
}

static void replay(const char *label, const unsigned *b, unsigned count,
                   bool keep_private, bool inclusive)
{
    const unsigned a[] = {0x2000, 0x2100, 0x2200};
    struct Cache cache = {.inclusive = inclusive};
    unsigned hits[SOURCES] = {0};

    for (unsigned i = 0; i < 3; ++i)
        read_block(&cache, A, a[i]);
    if (!keep_private)
        memset(cache.private_copy[A], 0, sizeof cache.private_copy[A]);
    for (unsigned i = 0; i < count; ++i)
        read_block(&cache, B, b[i]);
    for (unsigned i = 0; i < 3; ++i)
        ++hits[read_block(&cache, A, a[i])];

    printf("%s: L2=%u LLC requests=%u LLC hits=%u lower=%u\n", label,
           hits[PRIVATE_L2], hits[LLC] + hits[LOWER], hits[LLC], hits[LOWER]);
}

int main(void)
{
    const unsigned same[] = {0x2300, 0x2400, 0x2500};
    const unsigned separate[] = {0x2340, 0x2440, 0x2540};
    const unsigned repeat[] = {0x2300, 0x2300, 0x2300};

    replay("same set", same, 3, false, false);
    replay("different sets", separate, 3, false, false);
    replay("inclusive", same, 3, true, true);
    replay("retain private", same, 3, true, false);
    replay("repeat B0", repeat, 3, false, false);
    replay("only B0 and B1", same, 2, true, true);
}
~~~

~~~text
same set: L2=0 LLC requests=3 LLC hits=0 lower=3
different sets: L2=0 LLC requests=3 LLC hits=3 lower=0
inclusive: L2=0 LLC requests=3 LLC hits=0 lower=3
retain private: L2=3 LLC requests=0 LLC hits=0 lower=0
repeat B0: L2=0 LLC requests=3 LLC hits=3 lower=0
only B0 and B1: L2=0 LLC requests=3 LLC hits=0 lower=3
~~~

前两行用 `keep_private=false` 清空 A 的私有副本，复算前面的 LLC 组表。第三、四行都保留 A 第一轮取得的副本，两者的区别在于共享层替换时是否清除私有副本。第四行虽然有 `LLC requests=0`，A 仍执行了三次读取，全部在私有 L2 命中。`lower` 计的是要交给 LLC 以下层级的请求，模型只算到这一步。

## 从请求数和未命中数读出变化 {#llc-evidence}

把计数窗口定在 A 的第二轮读取，只数 A0、A1、A2 这三次需求加载。假设三次加载串行执行，每次 LLC 未命中各产生一笔已被下级接收的读请求。下表前三行复用本篇模型的输出，末行是额外的排队题设。

| 第二轮开始时的条件 | A 的加载 | 到达 LLC | LLC 未命中 | 到达下级 |
| --- | ---: | ---: | ---: | ---: |
| A 的三份私有副本都保留 | 3 | 0 | 0 | 0 |
| 清空 A 的私有副本，B 只反复读 B0 | 3 | 3 | 0 | 0 |
| 清空 A 的私有副本，B 依次读 B0、B1、B2 | 3 | 3 | 3 | 3 |
| 同上一行，下级已有其他请求排队 | 3 | 3 | 3 | 3 |

第一行和第二行的 LLC 未命中都为零，但未命中率的含义不同。第一行没有请求到达 LLC，分母为零。第二行有三次请求，全部在 LLC 命中。把第二行和第三行相比，到达 LLC 的请求数仍是三，可以把新增的下行请求追到 LLC 的同组替换。

第三行和第四行的计数完全相同。若题设中下级每次需要 20 周期，从被接收到返回，三次串行加载合计在下级花 60 周期。若第四行的第一笔请求还要在下级排 6 周期，后两笔照旧，下级这部分就变为 `(6 + 20) + 20 + 20 = 66` 周期。这里的 6 周期发生在下级接收请求之后，因此不会改变表中的接收次数。请求重叠时，逐项延迟相加也无法直接给出整个循环的用时。

复查真实程序时，先固定工作量和执行位置，再按同一个窗口对照加载、LLC 请求、下级接收和返回。下级请求可能合并，硬件事件也可能将预取、写回或其他核心的活动算进去。核对计数口径可读[访存计数一节](./microarch-04a-measurement.md#pmu-path)，要继续追踪请求在哪里排队，可读[内存控制器一节](./microarch-06b-memory-queue.md#memory-controller)。

::: details 练习：B 反复读同一个块

仍在 A 已经读过 A0、A1、A2 后运行 B。保持 B 的请求和 A 落在同一组，把 B 的序列改成只反复读取 B0。若 A 的下一轮全部到达 LLC，会有几次未命中？

零次。B0 第一次填进空路。再次读取 B0 可以在已有副本命中，无须为新块另找位置。程序的 `repeat B0` 一行报告 A 的三次请求都命中 LLC。

再把条件改成包含式 LLC，且 A 的私有副本一开始都在，B 只各读一次 B0、B1。B1 换出 A0 后，A1、A2 都还在私有层。A 重新填入 A0 时换出 A1，重新填入 A1 时又换出 A2。因此 B 直接换出一块，A 最终仍出现三次 LLC 未命中。
:::

### LLC 未命中是否等于 DRAM 读取？ {#llc-miss-source}

不等于。LLC 未命中事件首先说明所选事件定义的一次查找没有在 LLC 满足请求，它本身不说明数据最终来自哪里，也不等于一条 DRAM 读命令。请求可能与已有的缺失事务合并，也可能由更下层缓存或其他转发路径满足。不同处理器的事件还可能分别统计需求加载、缓存行填充或总线事务，计数单位需要查该处理器的事件定义。

例如，LLC 标签查找未命中后，内存控制器可能用写队列中已经接收的数据满足读取，而不为被覆盖的部分发 DRAM 列读。若要判断 DRAM 流量，可以同时观察内存控制器的读命令或数据量事件，并核对时间窗口、内存域和事件定义。控制器事件通常覆盖整个控制器或通道，可能包含其他核心、设备和内核活动，不能默认按任务或核心过滤。仅凭 LLC 未命中数与程序耗时，不能计算某个任务的 DRAM 读取次数。写队列逐段覆盖的例题见[内存控制器一课](./microarch-06b-memory-queue.md#memory-write-queue-read)。

假设一次窗口中 A 的 LLC 事件报告 100 次需求读未命中，同期某控制器报告 160 个读突发。这两个数各自有明确的观察对象。前者是题设中为 A 选择的缓存事件，后者包含这个控制器处理的所有来源。即便窗口完全重合，也不能把 160 次全部归给 A。还要核对 A 的请求是否都经过该控制器、控制器是否同时处理其他请求，以及一个上游请求是否拆成多个突发。

::: details 练习｜背景流量不变时的差值

另做一次独立测量，A 不运行，保持其余条件尽可能一致，控制器报告 60 个读突发。能否将 `160 − 60 = 100` 直接认定为 A 的精确读突发数？

差值可作为分析线索，但它依赖背景流量在两次运行中相同。A 运行可能改变其他任务的排队、调度或缓存行为，两次窗口的后台流量也可能自然波动。重复测量、记录控制器范围和后台条件，才能评估差值能支持多强的结论。
:::

## 参考文献

- [Marjan Fariborz，gem5 Bootcamp《Cache Systems》第 14 页](https://gem5bootcamp.github.io/gem5-bootcamp-env/assets/slides/using-gem5-04-gem5-caches-tutorial.pdf#page=14) · [CC BY 4.0 许可](https://github.com/gem5bootcamp/gem5-bootcamp-env/blob/51590ae00b0e451c9b6a8854addbb94128ab4cac/LICENSE) · 本文复用已裁出的图并以灰度显示。
- [Intel 第一代 Xeon Scalable 技术综述，分布式 CHA、缓存层次与数据放置](https://www.intel.com/content/www/us/en/developer/articles/technical/xeon-processor-scalable-family-technical-overview.html)
- [Intel Skylake Server Uncore 事件表，CHA 探测过滤器容量替换计数](https://perfmon-events.intel.com/platforms/skylakex/uncore-events/uncore/)
- [Intel Skylake Server 核心事件表，退休加载的数据来源](https://perfmon-events.intel.com/platforms/skylakex/core-events/core/)
- [Sorin、Hill、Wood，缓存一致性教材，包含式 LLC 与失效请求](https://pages.cs.wisc.edu/~markhill/papers/primer2020_2nd_edition.pdf)
- [gem5 v24.1.0.3，Ruby 共享 L2 的 `MT`、`MT_I` 和 `M_I` 转换](https://github.com/gem5/gem5/blob/v24.1.0.3/src/mem/ruby/protocol/MESI_Two_Level-L2cache.sm)
- [gem5 v24.1.0.3 Ruby 的 L2 换出与失效确认状态机](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/ruby/protocol/MESI_Two_Level-L2cache.sm)
- [Linux v6.12 resctrl 文档中的缓存实例 ID](https://docs.kernel.org/6.12/arch/x86/resctrl.html#cache-ids)
- [Linux 内核的缓存 sysfs 接口定义](https://github.com/torvalds/linux/blob/v6.12/Documentation/ABI/testing/sysfs-devices-system-cpu)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>上一篇</small><strong>TLB 命中与未命中怎样留下访问痕迹</strong></span>](./microarch-04i-tlb-observation.md){.article-nav-card .article-nav-prev rel=prev}

[<span class="article-nav-text"><small>下一篇</small><strong>未命中次数相同，为什么内存等待更长</strong></span>](./microarch-06b-memory-queue.md){.article-nav-card .article-nav-next rel=next}

</nav>
