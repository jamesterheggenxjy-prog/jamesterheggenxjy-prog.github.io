---
title: 缓存未命中后如何合并请求
description: 沿三笔加载追踪 MSHR 分配、发包、同块合并与返回，区分表项、等待者和缓存行各自保存的状态。
date: '2026-11-25'
tags: [微架构安全, 高速缓存, MSHR, 访存时序]
book: microarch-00-intro
---

缓存查不到某个地址的数据，就要向下一级请求一整行。在数据回来之前，核心可能又要读这行的另一个位置，也可能要读取其他块。要允许这些访问重叠，缓存必须保存已经受理但尚未完成的请求，还要避免同一行被重复请求。

下面沿用[组相联缓存的查找过程](./microarch-01b-cache-mapping.md#cache-lookup)，重点看标记查找未能满足请求以后发生什么。实现对照采用 gem5 v24.1.0.3 的经典一致性缓存。所用地址、数据和周期由例题给定，适用于普通可缓存读，假设响应成功、按需填充，且返回的数据和读权限足以满足所有读等待者。

[[toc]]

## MSHR 位于缓存的哪条路径上 {#mshr-structure}

阻塞式缓存可以把一次未命中处理完再接收下一次访问。非阻塞缓存允许部分访问先完成，例如在等第一个块时，先返回另一个已经驻留的块，也可以让两个不同块的未命中同时在途。能重叠多少，要看缓存的查找端口和用于保留请求的资源。

图 1 中，上端通向核心，下端通向互连和下一级。**Tags/Data** 是平时查标记和取数据的阵列。**MSHR**（Miss Status Holding Register，未命中状态保持寄存器）所在的结构，为尚未完成的请求保存状态。**WriteBuffer** 是另一条下行路径，保存需要发往下一级的写数据，例如被移出的脏行。后面一课会跟踪[写回缓冲怎样留住脏行](./microarch-01e-cache-stores.md#write-back-buffer)。图沿用旧文档的端口称呼，以下按连接方向称为核心侧和下级侧。

<figure class="lesson-figure" style="max-width: 374px">
<a href="/microarch/gem5-classic-cache.png"><img src="/microarch/gem5-classic-cache.png" alt="gem5 数据缓存结构，上端连接核心，下端连接互连，内部有标记与数据阵列、MSHR 和写缓冲" width="374" height="257" loading="lazy" decoding="async"></a>
<figcaption>图 1　gem5 经典数据缓存中的阵列、MSHR 和写缓冲。</figcaption>
</figure>

理解它们的分工，可以先问三个问题。

| 要判断什么 | 查哪一份状态 |
| --- | --- |
| 数据已经到了吗，当前能直接读吗 | 缓存阵列的标记、数据和有效、可读状态 |
| 这一块是不是已经在取 | MSHR 表项中的块地址和发包状态 |
| 数据回来后要回复哪些请求，各取哪些字节 | 该表项保存的原始请求列表 |

下一级的流量以块为单位，核心侧的返回仍以原始加载为单位。一个 MSHR 表项通常按一个块来跟踪，但一块中有多个加载等待数据时，每个加载都要保留自己的地址、大小和响应上下文。

## 一条 MSHR 表项保存什么 {#mshr-entry}

图 2 放大了 MSHR 队列与其中的等待者。以 `Cached Memory Block #1001` 为例，中间的大框是一个块的在途记录，向右串着的 `#1`、`#5`、`#10` 是原始请求。图中只保留原图的 MSHR 部分，请求类型和编号沿用原文档。下面只借用它的结构，用三个读请求看数据怎样返回。

<figure class="lesson-figure" style="max-width: 594px">
<a href="/microarch/gem5-mshr-queues.png"><img src="/microarch/gem5-mshr-queues.png" alt="gem5 官方队列图中的 MSHR 部分，每个块表项可保存多个原始请求" width="594" height="363" loading="lazy" decoding="async"></a>
<figcaption>图 2　MSHR 的块表项及其等待请求。点击查看大图。</figcaption>
</figure>

在 gem5 这一版中，表项和等待者分别保存如下状态。这里的“可发”描述调度资格，“在服务中”描述本级已经把请求交给下一级，二者发生在不同的时刻。

| 字段 | 保存的内容 | 什么时候使用 |
| --- | --- | --- |
| `blkAddr`、`blkSize` | 对齐的块起点与块大小 | 判定新请求是否指向同一块，构造向下的整块请求 |
| `isSecure` | 请求所属的安全地址空间属性 | 这一实现同时比较块地址和该属性 |
| `readyTime`、`order` | 最早可调度时刻、用于协调请求顺序的编号 | 选择何时尝试发出以及怎样处理有顺序约束的请求 |
| `inService` | 下级侧是否已接受本轮请求 | 区分还在等发包与已经交给下一级 |
| `targets` | 可由当前阶段处理的请求 | 返回时从中选出本次响应能够满足的请求 |
| `deferredTargets` | 需要额外条件或后续一轮处理的请求 | 当前响应到达后重新判断，必要时继续请求 |
| 每个 `target` 的 `pkt` 和时间、来源等 | 原始请求包与等待者自身的信息 | 从原地址确定返回的字节范围，并为原请求准备响应 |

`readyTime` 到了，端口仍可能忙。gem5 在发送成功后才设置 `inService`。若下一级不接受这次发送，表项仍然持有等待者，等下级端口允许重试时再构造并发送请求。这样，上游已把加载交给本级和下级已接受本级请求可以相隔若干周期。

## 同一份返回数据怎样交给多条加载 {#mshr-trace}

行长为 64 字节。主例有 R1、R2、R3 三笔 4 字节加载。下表中的字节用于核对数据通路，假定小端。另设 H 读取已经在本级的块 C。H 只用来观察一次命中能否越过未完成的未命中。

| 加载 | 字节地址 | 对齐后的块地址 | 行内偏移 | 读到的四个字节 |
| --- | --- | --- | --- | --- |
| R1 | `0x1044` | A，`0x1040` | 4 至 7 | `04 05 06 07` |
| R2 | `0x1064` | A，`0x1040` | 36 至 39 | `24 25 26 27` |
| R3 | `0x10c8` | B，`0x10c0` | 8 至 11 | `48 49 4a 4b` |
| H | `0x110c` | C，`0x1100` | 12 至 15 | `8c 8d 8e 8f` |

R2 的地址比 R1 大 32 字节，两者仍落在同一个对齐区间内。两个请求都得到自己的四字节数据，所以只按块地址保留一个请求包仍不够。要合并的是对下一级的读事务，上行仍要分别回应 R1 和 R2。

假设 A、B 开始时都未缓存。这里有两项用于未命中的槽位，每项至少能保存两个等待者。各次填充在收到整行时立即完成，不计本级的响应排队。

| 周期 | 发生的事件 | 状态或数据怎样变化 |
| ---: | --- | --- |
| 10 | R1 读 `0x1044` | 查标记未命中，为 A 分配表项，记下 R1，本轮尚未发出 |
| 12 | 下一级接受 A 的读请求 | A 变为在服务中，等待者仍是 R1 |
| 15 | R2 读 `0x1064` | 标记仍未命中，查到 A 的 MSHR，将 R2 追加到当前等待者 |
| 16 | H 读 `0x110c` | C 已驻留且可读，直接返回 `0x8f8e8d8c` |
| 18 | R3 读 `0x10c8` | 为 B 分配另一项，下一级当拍接受 B 的读请求 |
| 60 | B 的整行回来 | 填入 B，取出偏移 8 至 11 回应 R3，释放 B 的表项 |
| 80 | A 的整行回来 | 填入 A，用偏移 4 至 7 和 36 至 39 分别回应 R1、R2，释放 A 的表项 |

R1、R2、R3 都发生了标记查找未命中，但只有两笔下行读事务。第 15 周期，R2 查到的是已有的 MSHR，仍要等到第 80 周期才能获得自己的数据。H 在未命中尚未完成时直接命中，称为 **hit under miss**。B 与 A 的缺失事务重叠，称为 **miss under miss**。R3 返回得比 R1 早，指令最终以何种顺序更新程序可见状态，还要看[执行与退休](./microarch-03-execution.md#ooo-retire)。

对照源码，`sendMSHRQueuePacket` 发出请求时会在包的发送上下文中保留 MSHR。收到响应时，`recvTimingResp` 取回这份上下文，在常规读填充路径更新缓存块，再让 `serviceMSHRTargets` 服务这次能满足的等待者。真实实现还可能计入首个数据与后续数据的时序、端口排队，不能从这个例题推出各等待者必定同拍收到响应。

## 表项满了，和单项的等待者满了有什么区别 {#mshr-limits}

把表项总数改成一项，R3 在第 18 周期找不到自己的块，也找不到可分配的项。在允许同块请求继续进入的教学模型中，R2 仍能合并，因为 A 已经有表项。若总数仍为两项，但 A 的等待者容量为一，R2 则会因为 A 的那一项没有空位被推迟。第一个条件限制在途块数，第二个条件限制一笔块事务能保留的原始请求数。

gem5 将这两种容量分别暴露为 `mshrs` 和 `tgts_per_mshr`。它的经典缓存还会在资源达到阻塞条件时，通过核心侧端口拒绝暂时无法接收的新请求，解除阻塞后再通知重试。此时连原本可命中的请求也可能受端口阻塞影响。不能仅从“有一个块已缓存”推断它在别的请求受阻时始终能通过。下方代码把容量约束放在表项和等待者上，便于分别把两个限制改一遍。

还有一种无法马上合并的情况与空间无关。假设 R2 改成需要可写副本的存储，而 A 已经发出请求，预期只返回可读副本。这次响应能够满足 R1 的加载，却还不足以完成 R2 的存储。该版 gem5 会根据在途状态和当前响应能给出的权限判断是否把后到的请求放在 `deferredTargets`。如果回包实际已有所需权限，可以提升并处理这些等待者，否则还须继续请求。这里的可写状态来自缓存一致性，与页表决定程序能否写该地址是两个层面的检查。获取可写副本的过程见[缓存一致性一课](./microarch-01f-coherence.md#mesi-upgrade)。

## 用代码重放例题 {#mshr-model}

下面是可独立运行的 Python 教学模型。`pending` 以块地址为键存放表项，`cached` 持有已经可读的行。`agenda` 是按时间处理的事件队列。行里的内容用连续字节初始化，所以每条加载的结果都能按偏移复算。把代码保存为 `mshr_read.py` 后运行 `python mshr_read.py`。

```python
from dataclasses import dataclass
from heapq import heappop, heappush
from itertools import count

LINE = 64
A, B, C = 0x1040, 0x10C0, 0x1100
MEMORY = {A: bytes(range(64)), B: bytes(range(64, 128)), C: bytes(range(128, 192))}
RETURN_DELAY = {A: 68, B: 42}


@dataclass(frozen=True)
class Read:
    name: str
    address: int

    @property
    def block(self):
        return self.address // LINE * LINE


@dataclass
class Miss:
    targets: list[Read]
    in_service: bool = False


def simulate(entries=2, targets_per_entry=2, r2_arrival=15):
    agenda, sequence = [], count()
    pending = {}
    cached = {C: MEMORY[C]}
    retry, sent, replies = [], [], []

    def schedule(cycle, kind, item):
        heappush(agenda, (cycle, next(sequence), kind, item))

    def respond(cycle, request, source):
        offset = request.address - request.block
        data = cached[request.block][offset:offset + 4]
        word = int.from_bytes(data, "little")
        replies.append((cycle, request.name, source, word))

    schedule(10, "load", Read("R1", 0x1044))
    schedule(r2_arrival, "load", Read("R2", 0x1064))
    schedule(16, "load", Read("H", 0x110C))
    schedule(18, "load", Read("R3", 0x10C8))

    while agenda:
        cycle, _, kind, item = heappop(agenda)
        if kind == "issue":
            pending[item].in_service = True
            sent.append((cycle, item))
            schedule(cycle + RETURN_DELAY[item], "fill", item)
            continue

        if kind == "fill":
            miss = pending.pop(item)
            assert miss.in_service
            cached[item] = MEMORY[item]
            for request in miss.targets:
                respond(cycle, request, "fill")
            for request in retry:
                schedule(cycle, "load", request)
            retry.clear()
            continue

        request = item
        block = request.block
        if block in cached:
            respond(cycle, request, "hit")
        elif block in pending:
            miss = pending[block]
            if len(miss.targets) < targets_per_entry:
                miss.targets.append(request)
            else:
                retry.append(request)
        elif len(pending) < entries:
            pending[block] = Miss([request])
            issue_cycle = max(cycle, 12) if block == A else cycle
            schedule(issue_cycle, "issue", block)
        else:
            retry.append(request)

    return sent, replies


if __name__ == "__main__":
    sent, replies = simulate()
    for cycle, block in sent:
        print(f"下行 {cycle} {block:#06x}")
    for cycle, name, source, word in replies:
        print(f"上行 {cycle} {name} {source} {word:#010x}")
```

```text
下行 12 0x1040
下行 18 0x10c0
上行 16 H hit 0x8f8e8d8c
上行 60 R3 fill 0x4b4a4948
上行 80 R1 fill 0x07060504
上行 80 R2 fill 0x27262524
```

代码让 A 的请求等到第 12 周期才能发出。B 在第 18 周期到达时，模型允许它直接使用同一个下行接口。加载先查 `cached`，找不到时再查 `pending`，这正对应“缓存命中”和“在途表项命中”。填充时从等待列表逐笔提取字节，并将题设中暂时受阻的请求放回事件队列。这个模型没有端口级的全局阻塞，也没有一致性消息、替换或脏行，因此它返回的周期是用来核算例题的事件时刻。

把调用改成 `simulate(entries=1)`，第二笔下行变为第 80 周期发送 B，第 122 周期完成，因为 B 的 42 周期等待时间从实际发出开始。改成 `simulate(targets_per_entry=1)`，R2 在第 80 周期重试，填充已先完成，它这时从本级命中。两种改法都没有多发一笔 A 的下行请求。还可以用 `simulate(r2_arrival=11)` 让 R2 在 A 已分配、尚未发出时到达。R2 同样能加入 A 的等待者列表，原定的下行时刻不变。

## 怎样辨认预取是否赶上需求 {#mshr-prefetch}

若 A 起初由硬件预取发出，需求加载随后到达，同样可以使用已有表项。此时需求仍要等响应，它可能比从需求到达才发包更快，但不会直接从本级阵列命中。若预取已经填进本级，需求才到达，就会从阵列读取。两组下行请求数可以相同，等待时间却不同。

在 gem5 的这条路径，`serviceMSHRTargets` 会区别“一个预取带来的填充还没有被本次核心请求使用”和“响应已经满足了合并进来的核心请求”。前一种条件下可将有效的块标为 `prefetched`，等后续命中时识别预取的使用。进一步把观察、发包与及时性放在一起看，可读[硬件预取器怎样工作](./microarch-02-prefetch.md#prefetch-feedback)。

::: details 练习｜同样两笔下行，R2 还等 A 吗
把 R2 移到第 90 周期，假设从第 80 周期填入以后 A 一直有效。R2 会用哪条返回路径？下一级仍被请求几次？

R2 直接从本级取出偏移 36 至 39，这时 A 的 MSHR 表项已经释放。R1 与 R3 仍各引起一笔下行取行，总数是两笔。用 `simulate(r2_arrival=90)` 可以核对事件里 R2 变成第 90 周期的 `hit`。如果只看下行流量，无法分辨这次命中与主例中第 15 周期的同块合并。
:::

## 参考文献

- [gem5 经典缓存的组织](https://www.gem5.org/documentation/general_docs/memory_system/classic_caches/)
- 图 1、图 2 来自 Djordje Kovacevi 的 gem5 内存系统文档，本站版本转为灰度、补白底与边距，图 2 截取原图的 MSHR 部分。[原文与原图](https://www.gem5.org/documentation/general_docs/memory_system/gem5_memory_system/#mshr-and-write-buffer-queues) · [原站 MIT 许可](/microarch/gem5-website-license.txt)
- [gem5 v24.1.0.3 的 QueueEntry 和等待者字段](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/queue_entry.hh#L57-L114)
- [gem5 的 MSHR 分配、合并和延后等待者](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/mshr.cc#L303-L429)
- [gem5 核心侧的阻塞与重试](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/base.cc#L130-L185)
- [gem5 下行请求的发送与状态更新](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/base.cc#L1907-L1968)
- [gem5 响应填充与等待者处理](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/base.cc#L515-L619)
- [gem5 响应、等待者和预取标记](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/cache.cc#L699-L947)
- [gem5 的表项与等待者容量配置](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/Cache.py#L99-L105)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>上一篇</small><strong>矩阵按行读和按列读的缓存开销</strong></span>](./microarch-01d-cache-traversal.md){.article-nav-card .article-nav-prev rel=prev}

[<span class="article-nav-text"><small>下一篇</small><strong>写入缓存后，数据什么时候送到下一级</strong></span>](./microarch-01e-cache-stores.md){.article-nav-card .article-nav-next rel=next}

</nav>
