---
title: 硬件预取器怎样工作
description: 用 gem5 的步长预取器跟完同一笔访存，从观察入口、预测表和发包队列走到返回填充。
date: '2026-11-25'
tags: [微架构安全, 硬件预取, 访存时序, MSHR]
book: microarch-00-intro
---

硬件预取的目的，是让一个将来才需要的访存请求尽早开始。如果当前循环会连续处理 128 字节大小的记录，某一条加载指令每次都读记录中相同的字段。处理器在观察几个地址后，可以尝试提前取下一条记录的缓存行。但猜出地址以后，还要决定地址能否转换、有没有重复请求、缓存能不能接收它，以及数据是在需求加载之前还是之后回来。理解预取器，要把这段请求的经历走完整。

本文以 gem5 v24.1.0.3 的 `StridePrefetcher` 和它连接的经典一致性缓存为主例。gem5 是时序模拟器，本文按这一版本的实现讲解。文中的周期数是教学题设。本文沿同一笔请求讲完预测、入队、仲裁、填充和效果记录。若还不熟悉块地址，先读[缓存的地址映射](./microarch-01b-cache-mapping.md)。读到并发、合并和返回时，可先用[缓存未命中与 MSHR](./microarch-01a-mshr.md#mshr-trace)把这三个概念接起来。

[[toc]]

## 预取器在缓存系统中的位置

在 gem5 经典缓存里，预取器挂在一层缓存旁边。它从本层接收访问事件，向同一层交还预取请求，再从缓存的命中和填充获得反馈。图上半部是预取器，下半部是当前缓存。MMU 和下一级存储各有自己的接口。

<figure class="lesson-figure">
  <a href="/microarch/stride-prefetcher-structure.png" aria-label="查看完整的预取器结构图">
    <img src="/microarch/stride-prefetcher-structure.png" width="820" height="644" loading="lazy" alt="gem5 步长预取器结构框图。当前缓存发出访问事件，Base 过滤并交给步长表及候选逻辑。Queued 检查候选并经待发队列交给缓存，跨页候选先经过 MMU。缓存用 MSHR 跟踪请求，接收数据并更新缓存行和使用统计。">
  </a>
  <figcaption>图 1　从左下方的访存通知看起。上半部把 0x1200 变成 0x1280、0x1300 两个候选，再沿实线跟踪 0x1280 如何进入待发队列、交给缓存并得到响应。虚线表示有 MMU 时的跨页翻译路径。图按 gem5 v24.1.0.3 的功能接口绘制。</figcaption>
</figure>

**需求请求**来自正在执行访存操作的一方。普通加载需要真正返回请求的数据。**预取请求**由预测机制生成，主要作用是让内存系统提前准备数据。在本例中，步长算法输出的是候选地址。后面的模块决定是否将这个地址变成可发给缓存的请求。缓存取到请求以后，才决定要不要跟踪一笔新的未命中事务。

<div role="region" aria-label="预取器与缓存之间传递的信息" style="max-width:100%; overflow-x:auto">

| 环节 | 读入什么 | 保存或产出什么 |
| --- | --- | --- |
| 缓存事件与观察入口 | 请求包，命中或未命中的信息 | 用于训练的地址、PC、请求者、相关属性 |
| 步长预测 | 一次合格的观察 | 按流保存的最近地址、步长和置信度；可能产出几个地址候选 |
| 预取队列和翻译接口 | 地址候选、源请求上下文、缓存查询接口 | 等待翻译的候选，以及已有物理地址、等待发送资格的请求 |
| 缓存下行仲裁和 MSHR | 合格的预取请求，以及正常未命中和写回 | 启动或跟踪一次取行事务，并记录等待这份数据的请求 |
| 缓存应答和反馈 | 下级返回的数据、等待者、之后的缓存访问 | 更新缓存行，完成需求加载，记录及时使用或未使用的情况 |

</div>

**预取队列**中的候选还可以取消。**MSHR** 中的请求已经由缓存接手，可能在等端口，也可能已经在向下一级取数据。判断某次需求加载怎样得到数据，必须知道它到来时预取处于哪一步。

## 观察入口与请求者选表 {#prefetch-observe}

在这个模型中，预取器挂在某一级缓存上，通过命中和未命中事件接收请求。即使核心执行了同样的指令，在 L1 和 L2 上也可能看到不同序列。例如，连续读取已经在 L1 里的多个元素，L2 可能只看到了把整行取回来的那一次。**预取器学习的是它的观察点收到的事件序列**。分析一个循环时，要先说明预取器挂在哪一级，以及本级哪些请求会通知它。

本版的公共入口先排除一些请求，例如软件预取和缓存维护操作，再按配置决定采用哪些访问。配置涉及读写、数据与取指、可缓存性、命中类别，以及使用虚拟还是物理地址。步长预取器默认关闭对取指的训练，并会忽略缺少 PC 的事件。PC 是触发数据访问的指令地址，数据地址是这条指令本次读到的位置。一个循环可以每次执行同一条加载，PC 保持稳定，数据地址却一直向前走。

命中也有不同口径。默认参数 `prefetch_on_pf_hit` 允许预取标记命中触发观察；普通命中是否进入观察要看命中分支和相关参数。源码在命中时优先检查这一开关。计算某段轨迹时，应从实际到达步长算法的合格观察开始，以免把被过滤掉的访问算进训练次数。

## 步长表的字段与更新 {#prefetch-stride}

设循环体里有两条加载。一条依次读每个记录的状态字段，另一条通过描述符去读不同位置的数据。如果只把两条加载的地址混成一串，第一条自己的固定步长可能很难看出来。gem5 的步长预取器先选请求者对应的表，再以 PC 和 secure 属性组成的键查询。一张表由带标记的有限条目组成。本版默认每张表 64 项、4 路，组索引由 PC 混合后取得，替换策略可配置。关闭 `use_requestor_id` 时，请求者用同一张表。这里的请求者标识由模拟系统提供，在一个真实硬件里是否有类似标识，需看该设计。

选择器和比较逻辑用表项左侧的有效状态与标记认出访问流，右侧的三列是预测所用的状态。图中 PC／S 指 PC 和 secure 属性，替换状态由相联表的管理逻辑维护。

<div role="region" aria-label="步长表项的字段" style="max-width:100%; overflow-x:auto">

| 表项信息 | 保存的值 | 什么时候读取或修改 |
| --- | --- | --- |
| 有效状态与标记 | 本项是否可用，以及对应的 PC 标记和 secure 属性 | 查表时匹配，分配或替换时更新 |
| `lastAddr` | 上次有效观察的地址 | 用来求差值，之后改成这次的地址 |
| `stride` | 历史估计的带符号步长 | 与新差值比较，置信度降到阈值以下时可替换 |
| `confidence` | 有上下限的计数 | 步长匹配时增大，不匹配时减小，生成候选前再查阈值 |

</div>

带符号步长也能表达往低地址走的流。默认按缓存行对齐，行内不同字节归到同一个训练地址。若这次与上次仍在同一行，就直接返回。源码在提前返回前已登记表项访问，替换策略仍可能看到这次命中。

训练过程可以这样读：

1. 首次从空表查到该键缺失时，分配表项并记录最近地址。起始步长是 0，置信度取初始值。
2. 下次观察时，用本次地址减去最近地址。若距离与旧步长相同，就向上调整置信度；否则向下调整。
3. 若这次距离不同，且调整后的置信度已低于阈值，就用新距离替换旧步长。更新最近地址。
4. 置信度达到阈值，才按保存的步长产生候选。计数有限，上下都饱和。

这使得一次偶发远跳可以被视为噪声，而持续的新步长可以逐渐取代旧步长。假设置信度为 6，旧步长为 128，这次突然跳了 192。计数降到 5，仍高于 50% 的阈值，预测会继续使用旧步长。若原置信度只有 4，同一次跳变使计数降到 3，新步长则会替换旧值。本次因为置信度不足，仍不会生成候选。**改变步长和允许发出候选是两次不同的判断。**

### 手算一个从空表开始的例子

行长 64 字节，使用默认 3 位置信度计数，范围 0–7，初值 4，阈值 50%。一条加载的三次合格观察如下。表内用字节地址和字节步长。为了便于追踪后续请求，本例把 `degree` 设为 2，`distance` 设为 0。

<div role="region" aria-label="步长表训练轨迹" style="max-width:100%; overflow-x:auto">

| 本次地址 | 与上次之差 | 表项里的步长 | 处理后置信度 | 候选 |
| --- | --- | --- | --- | --- |
| `0x1100` | 无 | 0 | 4，分配时写入 | 无 |
| `0x1180` | `0x80` | `0x80`，改用新值 | 3，低于一半 | 无 |
| `0x1200` | `0x80` | `0x80` | 4，超过一半 | `0x1280`、`0x1300` |

</div>

若在第二、三次之间插入一次 `0x1188`，它属于 `0x1180` 这行，按默认块地址模式不会推进训练。第三次观察仍由 `0x1200` 与 `0x1180` 计算差值。若将 `distance` 改成 1，则在产出前先跳过一个步长。本例的首个候选移到 `0x1300`，第二个是 `0x1380`。因此，`distance` 决定从离当前多远的地方开始，`degree` 决定这一轮一共提出多少候选。

50% 阈值用 3 位计数器表示，就是计数至少为 4，因为 3/7 低于一半，4/7 超过一半。首次分配没有候选，第二次虽学到 `0x80`，置信度还不够，第三次才得到两个地址。接下来跟踪这两个地址怎样成为可以发给缓存的请求。

## 候选的地址检查与待发队列 {#prefetch-queue}

候选首先按缓存行对齐，同一行只需要取一次。预取队列可以检查同地址、同属性的待发项，避免候选重复积累。若启用了 `cache_snoop`，入队前还会问本级缓存和 MSHR 是否已有这块；缓存端实际取出预取包时也有检查，补上候选等待期间状态变化的情况。比如入队时 A 尚未被取走，等轮到发出时需求加载已经给 A 分配了 MSHR，此时再取一次没有意义。

接下来是地址翻译。本例先用 **4 KiB 页、按物理地址训练、无注册 MMU** 的简化配置。同页候选可直接使用物理块地址。若最近地址为 `0x1f80`，步长仍为 `0x80`，第一个候选就是 `0x2000`，已经进入下一页。无注册 MMU 的路径会在跨页检查处跳过它。

有 MMU 的配置还要分清训练发生在哪个地址空间。若按虚拟地址训练且候选仍在原页，页内偏移的变化可以应用到原物理地址。若候选跨页，下一虚拟页的物理页号未必连续，因此要准备一次翻译请求。该实现需要能够找到源请求的上下文。按物理地址训练而跨页时，源码也需要原请求附带虚拟地址才能构造待翻译的目标。需要翻译的候选在另一条队列中等待。翻译失败会丢弃，成功后才取得发包所用的物理地址，并可再做缓存及 MSHR 去重。

准备好物理地址的项进入待发队列，记录可取出的时间、包体、优先级和用于比较的地址信息。该版队列满时会移出最低优先级中最早排入的一项，再插入新项。步长算法输出的候选使用同一优先级，所以这类候选之间可以按入队先后理解移除顺序。若需求已到同一行，且它触发了预取观察，启用 `queue_squash` 的路径可把同一行的待发项从队列取消。这样留给真正需要这份数据的请求按需求路径处理。

### 跨过页边界以后，物理地址怎样改变 {#prefetch-cross-page}

把配置换成按虚拟地址训练，并注册 MMU。取[地址翻译一课](./microarch-02a-translation.md#sv39-walk)的两张相邻虚拟页，令 VPN `0x40403` 映到 PPN `0x123`，VPN `0x40404` 映到 PPN `0x565`。两页都允许本例的读访问，缓存行长仍为 64 字节。

同一条加载在 `0x40403e80`、`0x40403f00`、`0x40403f80` 上连续通过观察。第三次观察恢复到置信度 4，步长为 `0x80`，预测的第一个虚拟块是 `0x40404000`。原加载的物理地址是 `0x123f80`。如果继续在物理地址上加 `0x80`，会得到 `0x124000`。这个计算跨过了原物理页的末尾，而目标虚拟页映到的是物理页 `0x565`。

从队列的角度看，此时有目标虚拟地址和源访问的执行上下文，还没有可送给缓存的物理块地址。候选先进入待翻译队列，之后由 MMU 对 `0x40404000` 做读访问翻译。因为新页内偏移为 0，翻译返回 `0x565000`。队列据此构造以 `0x565000` 为地址的预取包，再交给待发队列。第二个候选 `0x40404080` 的页内偏移是 `0x80`，因此翻译后是 `0x565080`。如果目标页不允许读取，失败的候选不会进入待发队列，也不会为那条预取分配 MSHR。

::: details 练习｜提前一行的同页候选
改成用 `0x40403e00`、`0x40403e80`、`0x40403f00` 训练。第三次观察的第一个候选是否要再向 MMU 提交翻译？物理块地址是多少？

第一个候选为 `0x40403f80`，仍在原虚拟页内。原请求的物理地址是 `0x123f00`，加同一页内的偏移差 `0x80` 就得到 `0x123f80`。第二个候选 `0x40404000` 仍须走跨页翻译。两个候选来自同一个步长表项，在队列却走了不同路径。
:::

## 缓存仲裁、MSHR 与返回填充 {#prefetch-fill}

候选达到队列中“可取”的时间，只意味着可以参与发送。经典缓存先考虑已可服务的未命中和写回，再视阻塞状态与留给需求请求的 MSHR 资源决定是否取出硬件预取包。被取出的包会再次查本级缓存、MSHR 和写缓冲，以处理等待期间发生的变化。需要新事务的请求才分配 MSHR，参与向下的访存。下级返回有效数据后，缓存按填充策略更新相应行，再服务匹配的等待者。

继续跟踪图中的候选 `0x1280`。设第 200 周期得到候选，第 201 周期在待发队列可取。设下级给此类请求的服务时间固定为 90 周期，并忽略返回后到核心之间的其他延迟。下面是三个独立情形。假定普通读可以跟已在途的读事务合并，且预取如成功填充，在题目所讨论的时间内保留在本级。

<div role="region" aria-label="需求到达时的三种预取状态" style="max-width:100%; overflow-x:auto">

| 条件改变 | 请求到达时的状态 | 推导结果 |
| --- | --- | --- |
| 预取第 210 周期真正发出，需求第 325 周期到达 | 第 300 周期已返回并填充 | 需求在本级命中。此处发生的是缓存查找命中。 |
| 预取仍于第 210 周期发出，需求第 260 周期到达 | MSHR 已有同块事务，数据还在路上 | 需求加入等待者，约等 40 周期。无预取且可立即发需求时，约等 90 周期。 |
| 资源不足，预取仍在待发队列，需求第 208 周期到达并触发观察 | 还没有预取事务可以合并 | 同块待发预取可被取消，需求自行分配或等待 MSHR。 |

</div>

第三种情况也提示了两个不同的问题。若候选只是由于更早的可服务请求排在前面，需求可能很快取得资源。若所有 MSHR 及等待条件都已达到限制，需求也可能等待。所以下一级“服务 90 周期”描述的是发出后的代价，还要加上发出前可能发生的排队。回头看第二行，显示一次本级未命中，并不妨碍提前发出的事务减少一部分等待。

图中的 0x1280 在第 201 周期已经能取，但第 210 周期才真正向下发出。两者之间的 9 周期来自本例的本地资源与仲裁。返回时，缓存控制器需要用 MSHR 找到地址和等待者。可缓存的响应按策略填入数据阵列，再为已经等待的需求提供所需字节。如果需求在填充后到达，就从正常的标记与数据阵列走命中路径。

### 把待发队列缩到两项

仍用 `0x1280` 和 `0x1300`。此时两项候选都已进待发队列，但尚未被缓存接手。再假设另一条已学到步长的加载产生 `0x1380`，它通过了地址检查和去重，和前两项优先级相同。容量设为两项时，入队过程按本版规则移除最早的 `0x1280`，队列留下 `0x1300` 和 `0x1380`。如果此后需求访问 `0x1280`，不能用这个已经被移除的候选缩短等待。把同一题改为 `0x1280` 已经离开队列并进入 MSHR，待发队列再满也不会使该在途事务消失。两种容量压力作用在请求的不同阶段。

## 用 C 复算候选、发出与需求到达 {#prefetch-model}

本例只保留图中的一个步长表项、两个等优先级的待发槽，以及跟踪 `0x1280` 的一笔 MSHR。输入是已经通过观察过滤的地址。模型使用物理地址、4 KiB 页、没有注册 MMU。函数调用给出仲裁和返回时机，因此输出是题设的计算结果。上文的跨页翻译路径、其他在途请求、实际缓存替换和反馈统计均由各自的部件管理。

保存为 `stride_demo.c`，用 `gcc -std=c11 -O2 -Wall -Wextra stride_demo.c -o stride_demo` 编译，再运行 `./stride_demo`。Windows 上可运行 `./stride_demo.exe`。

~~~c
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>

enum { LINE = 64, PAGE = 4096, DEGREE = 2, CAPACITY = 2, SERVICE = 90 };

typedef struct {
    bool valid;
    int last_addr, stride, confidence;
} StrideEntry;

typedef struct {
    int line, ready;
} Request;

typedef struct {
    Request slots[CAPACITY];
    unsigned count;
} Queue;

typedef struct {
    bool valid;
    int line, fill;
} Mshr;

typedef struct {
    Mshr pending;
    bool filled;
    int filled_line;
} Cache;

static unsigned observe(StrideEntry *entry, int addr, int distance,
                        int candidates[DEGREE])
{
    int line = addr / LINE * LINE;
    if (!entry->valid) {
        *entry = (StrideEntry){true, line, 0, 4};
        return 0;
    }

    int delta = line - entry->last_addr;
    if (delta == 0)
        return 0;

    if (delta == entry->stride) {
        if (entry->confidence < 7)
            ++entry->confidence;
    } else {
        if (entry->confidence > 0)
            --entry->confidence;
        if (entry->confidence < 4)
            entry->stride = delta;
    }
    entry->last_addr = line;
    if (entry->confidence < 4)
        return 0;

    for (unsigned i = 0; i < DEGREE; ++i)
        candidates[i] = line + (distance + (int)i + 1) * entry->stride;
    return DEGREE;
}

static bool remove_line(Queue *queue, int line)
{
    for (unsigned i = 0; i < queue->count; ++i) {
        if (queue->slots[i].line != line)
            continue;
        for (unsigned j = i + 1; j < queue->count; ++j)
            queue->slots[j - 1] = queue->slots[j];
        --queue->count;
        return true;
    }
    return false;
}

static void enqueue(Queue *queue, int origin, int line, int cycle)
{
    if (origin / PAGE != line / PAGE)
        return;
    for (unsigned i = 0; i < queue->count; ++i) {
        if (queue->slots[i].line == line)
            return;
    }

    if (queue->count == CAPACITY)
        remove_line(queue, queue->slots[0].line);
    queue->slots[queue->count++] = (Request){line, cycle + 1};
}

static Mshr issue(Queue *queue, int cycle)
{
    if (!queue->count || queue->slots[0].ready > cycle)
        return (Mshr){0};
    int line = queue->slots[0].line;
    remove_line(queue, line);
    return (Mshr){true, line, cycle + SERVICE};
}

static void complete_if_due(Cache *cache, int cycle)
{
    if (cache->pending.valid && cycle >= cache->pending.fill) {
        cache->filled = true;
        cache->filled_line = cache->pending.line;
        cache->pending.valid = false;
    }
}

static void demand(const char *label, Queue queue, Cache cache,
                   int line, int cycle)
{
    complete_if_due(&cache, cycle);
    printf("%s c%d: ", label, cycle);
    if (cache.filled && cache.filled_line == line) {
        puts("cache hit");
    } else if (cache.pending.valid && cache.pending.line == line) {
        printf("merge, wait %d cycles\n", cache.pending.fill - cycle);
    } else if (remove_line(&queue, line)) {
        puts("cancel queued prefetch, demand miss");
    } else {
        puts("demand miss with no matching prefetch");
    }
}

int main(void)
{
    const int addresses[] = {0x1100, 0x1180, 0x1188, 0x1200};
    StrideEntry entry = {0};
    int candidates[DEGREE];
    unsigned count = 0;
    for (unsigned i = 0; i < sizeof addresses / sizeof addresses[0]; ++i) {
        count = observe(&entry, addresses[i], 0, candidates);
        printf("access %04x: last=%04x stride=%d confidence=%d",
               (unsigned)addresses[i], (unsigned)entry.last_addr,
               entry.stride, entry.confidence);
        for (unsigned j = 0; j < count; ++j)
            printf(" %04x", (unsigned)candidates[j]);
        putchar('\n');
    }
    if (count != DEGREE)
        return EXIT_FAILURE;

    Queue queued = {0};
    for (unsigned i = 0; i < count; ++i)
        enqueue(&queued, addresses[3], candidates[i], 200);
    Queue after_issue = queued;
    Mshr flight = issue(&after_issue, 210);
    if (!flight.valid)
        return EXIT_FAILURE;
    printf("issue c210: line=%04x fill=c%d\n", (unsigned)flight.line, flight.fill);

    Cache cache = {.pending = flight};
    demand("late demand", after_issue, cache, 0x1280, 325);
    demand("early demand", after_issue, cache, 0x1280, 260);
    demand("port busy", queued, (Cache){0}, 0x1280, 208);

    Queue full = queued;
    enqueue(&full, 0x1200, 0x1380, 202);
    printf("full queue: %04x %04x\n",
           (unsigned)full.slots[0].line, (unsigned)full.slots[1].line);

    Queue crossing = {0};
    enqueue(&crossing, 0x1f80, 0x2000, 200);
    printf("cross page without MMU: %u queued\n", crossing.count);
}
~~~

~~~text
access 1100: last=1100 stride=0 confidence=4
access 1180: last=1180 stride=128 confidence=3
access 1188: last=1180 stride=128 confidence=3
access 1200: last=1200 stride=128 confidence=4 1280 1300
issue c210: line=1280 fill=c300
late demand c325: cache hit
early demand c260: merge, wait 40 cycles
port busy c208: cancel queued prefetch, demand miss
full queue: 1300 1380
cross page without MMU: 0 queued
~~~

`observe` 的四行输出把同一行的 `0x1188` 与下一行观察区分开。候选在第 200 周期进队列，下一周期达到可取时间。主程序直到第 210 周期才调用 `issue`，模拟缓存此时才接手并启动事务。它走的仍是图 1 中 `0x1280` 的那条路径。每次调用 `demand` 都从独立的缓存和队列副本开始。`complete_if_due` 先处理需求到来前已完成的填充，更新缓存行并释放 MSHR。随后按缓存行、在途请求和待发队列的顺序查找 `0x1280`。第 208 周期的场景使用发出前的队列副本，因为这一场景里预取一直没有拿到端口。最后两行复算满队列和无 MMU 的跨页丢弃。

## 从填充和命中判断预取效果 {#prefetch-feedback}

步长表的置信度回答“最近观察是否符合旧步长”。即使预取发不出去，只要距离保持一致，置信度也可能升高。发出的请求到底有没有作用，需要查看实际的缓存行和需求请求。

在所选缓存实现中，如果响应完成了预取目标，而这次响应没有服务核心目标，已填充的块可以带预取标记。后来需求命中，可以据此记一次使用，并清除标记，避免把同一行的后续访问反复记为新收益。若带标记的块在未使用前失效，会计作未使用。在途合并是另一个事件：需求在返回前加入等待者，这次响应已服务核心，标记条件与“填完才被需求命中”不同。

公共队列还可以按使用效果给候选设预算。开启 `throttle_control_percentage` 时，先按参数留下一个候选数的下限，其余额度按已使用预取与已取出的预取包的比例给出。这个参数默认是 0，默认就不根据这个比例收紧候选额度。队列最终也只允许通过地址、去重及容量条件的请求。**置信度调的是对步长的信任，节流调的是可处理的候选数，缓存仲裁调的是现在有没有资源发。** 三者的输入不同。

### 测量时先决定计数的事件

假设无预取时测得 100 次需求未命中。启用预取后，80 个已发送请求里，50 个及时填充并在需求时命中，10 个通过在途合并服务需求，另外 20 个未被需求使用。若其他需求结果不变：

- 以「无预取时的需求未命中」为分母，转成本级命中的比例是 50/100。
- 如果把及时使用和迟到但成功合并都算作使用，以「已经发送的预取请求」为分母，使用比例是 60/80。
- 还有 10 次本级未命中虽然未被消除，却可能减少其等待时间。

这是本文明确约定的口径。gem5 源码的 `pfUseful`、`pfIssued` 和 `coverage` 有自己的定义；例如 `pfIssued` 在预取队列取包时计数，缓存随后还能丢弃重复包。它的 `pfLate` 是取出的预取撞到缓存、MSHR 或写缓冲的统计，和上表“需求到来时合并”的次数也有不同的事件边界。要比较两个结果，先写清楚双方从哪个接口读到分母、哪些情况进入分子。

如果这次预取虽然正确，却先把随后会用到的另一条行挤出，会形成额外需求未命中。要归因于缓存污染，可以把带预取和不带预取的访问顺序放在一起比较，检查哪条行在两种条件下于何时被移出。同样，队列和端口的额外等待需要和缓存替换单独分析。

### 从地址提前量推到时间提前量

::: details 改变提前量与资源条件
还是每次前进 128 字节的记录流。假设观察到的需求平均每 20 周期推进一条记录，下级服务一条缓存行要 90 周期。对一条将来会用到的行，若预取在需求出现前 4 个步长真正发出，回来时需求等待了多久？提前 5 个步长又怎样？若第二种情况下另有 25 周期的队列等待，结论怎么变？

4 个步长给出 80 周期提前量，90 周期后返回，需求出现后还需等约 10 周期。5 个步长给出 100 周期提前量，数据理论上早到 10 周期。若将 5 个步长理解为「候选生成」的提前量，而出队又晚了 25 周期，真正的提前量只有 75 周期，需求出现后还需等约 15 周期。这里均假定未被替换、请求可以合并，并忽略本级其他延迟。

本例的「提前几步」是由时间线给出的实际提前量。配置中的 `distance` 控制第几个候选地址，能否达到同样的时间提前量仍取决于队列与访存服务。
:::

回到安全分析时，这一链条能解释一个常见误判。先发生的活动可能是需求加载，也可能是它带来的预取观察。看到另一行很早就命中，只说明它在测量发生时已经可用。要说明是谁请求了它、为什么被填充，需要能够排查预取的观察点、生成规则与返回时序。 可结合[计时和计数器一节](./microarch-04a-measurement.md)判断证据。

下一篇换到可综合的 Chipyard BAR-fetchers。[逐拍例题](./microarch-02d-prefetch-rtl.md#rtl-trace)用一个候选经历的两次缓存接收，说明同一项如何留在队列里等待重试。

## 参考文献

- [gem5 步长预测实现](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/prefetch/stride.cc)
- [Chipyard 预取器文档](https://chipyard.readthedocs.io/en/stable/generators/prefetchers/)
- [gem5 预取队列](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/prefetch/queued.cc)
- [gem5 缓存下行仲裁](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/base.cc#L858-L947)
- [观察条件和请求转换](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/prefetch/base.cc#L167-L273)
- [步长与通知配置](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/prefetch/Prefetcher.py#L73-L220)
- [gem5 训练与产出逻辑](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/prefetch/stride.cc#L108-L181)
- [gem5 地址产出与参数](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/prefetch/Prefetcher.py#L174-L220)
- [gem5 跨页判定、翻译回调及入队](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/prefetch/queued.cc#L198-L233)
- [gem5 去重、翻译请求和队列淘汰](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/prefetch/queued.cc#L376-L534)
- [经典缓存的目标完成路径](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/cache.cc)
- [gem5 满队列的处理](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/prefetch/queued.cc#L486-L530)
- [gem5 预取标记和清除](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/cache.cc#L900-L954)
- [gem5 命中和失效路径](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/base.cc)
- [gem5 候选额度和取包计数](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/prefetch/queued.cc#L141-L263)
- [gem5 统计公式及事件定义](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/prefetch/base.cc#L128-L165)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>上一篇</small><strong>两个线程改不同字段，为什么缓存行还会来回传</strong></span>](./microarch-01g-false-sharing.md){.article-nav-card .article-nav-prev rel=prev}

[<span class="article-nav-text"><small>下一篇</small><strong>预取器和数据缓存怎样握手</strong></span>](./microarch-02d-prefetch-rtl.md){.article-nav-card .article-nav-next rel=next}

</nav>
