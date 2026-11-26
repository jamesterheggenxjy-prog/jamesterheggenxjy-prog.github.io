---
title: 时间差何时算作侧信道证据
description: 用同一项服务的分组记录和阶段时间戳，判断总耗时差能支持什么结论。
date: '2026-11-25'
tags: [微架构安全, 侧信道, 威胁模型, 实验证据]
book: microarch-00-intro
---

一项服务对两类请求返回同样的确认消息。检查历史记录时，A 类请求的中位用时为 2 毫秒，B 类为 30 毫秒。单看差值，容易怀疑服务对两类请求走了不同的内部路径。先问两个更具体的问题。两组请求来自同样的网络吗？在相同网络下，差值出现在服务处理的哪一段？

[访存计时一节](./microarch-04a-measurement.md#pmu-attribution)已经说明怎样把某个总耗时和缓存、地址翻译等事件对照起来。本节从服务维护者验证隐私风险的视角，继续追踪同一组证据能支持的结论。下面的汇总和时间戳均为教学数据。

[[toc]]

## 先写清谁看得到哪些记录 {#evidence-observer}

本例中的服务读取内部业务分类 A 或 B，取回一条记录，再返回同样的确认消息。分类由服务掌握，正常回复不包含它。服务端记录收到请求、取得工作线程、结束索引查询和送出回复四个时刻。客户端只知道自己发的请求与收到的回复，也能记录自己等待了多久。

| 记录 | 在哪一侧取得 | 本节用来回答什么 |
| --- | --- | --- |
| 请求大小、连接类型和回复内容 | 客户端 | 观察者已经知道哪些条件，是否需要在这些条件内比较 |
| 端到端耗时 | 客户端 | 给定观察者真正能看见的差异 |
| 排队、索引和回复阶段的耗时 | 服务端 | 端到端的差值可能来自处理路径的哪里 |
| 索引阶段的性能计数 | 服务端 | 某种缓存或地址翻译活动是否与该阶段的变化同时发生 |

侧信道问题关心的是，观察者已经知道公开条件以后，额外可见的时间是否还提供有关受保护状态的信息。若仅内部日志能把两类区分开，日志可以指明要排查的环节。对外部观察的结论还要看客户端自己的记录。

## 总表有差值，分开以后有没有 {#evidence-stratify}

先看历史汇总。办公室网络中的每次请求耗时 2 毫秒，远程网络中的每次请求耗时 30 毫秒。两类请求各有 200 次。

| 分类 | 办公室网络 | 远程网络 | 汇总中位数 |
| --- | ---: | ---: | ---: |
| A | 180 次 | 20 次 | 2 ms |
| B | 20 次 | 180 次 | 30 ms |

A 有 90% 的样本来自较快的网络，B 有 90% 来自较慢的网络。中位数相差 28 毫秒，是两组样本构成不同的结果。拆开后，在同一网络里，A 与 B 的用时完全相同。对已知网络类型的观察者，这批用时没有提供关于分类的新信息。

也可以用平均数复算。按各自的网络比例，A 的平均值为 4.8 毫秒，B 的平均值为 27.2 毫秒。若要比较两类在同一目标网络构成下的平均值，可以给两边都用 50% 办公室、50% 远程的权重。两边都会得到 16 毫秒。这里还同时知道每个网络内部的两种时间分布完全一样。只看到加权平均值相同，在一般情况下无法断定分布也相同。

如果没有网络标签，原始差异依然存在。这样的记录无法单靠耗时解释哪一段变慢。服务维护者需要补充可比样本。后面假定两类请求在同一种网络、相同大小和相同运行版本下成对获得，并在短时间块内交错运行，避免总是一类先热身、另一类后运行。这个安排使后面的问题更具体。给定同样的外部条件，差异还会出现在哪里？

## 两份同样慢 3 毫秒的服务端记录 {#evidence-stages}

把一条已完成的请求拆开。服务端在同一个时钟上记录以下四个时刻。

| 字段 | 记录点 | 与上一项的差 |
| --- | --- | --- |
| `received` | 服务接收请求 | 服务处理区间的起点 |
| `lookup_start` | 获得工作线程，开始索引查询 | 排队时间 |
| `lookup_end` | 索引查询结束 | 索引阶段的用时 |
| `sent` | 向传输层交付回复 | 查询后处理及交付回复的用时 |

请求 ID 能把客户端记录与服务端记录对应起来。客户端耗时由客户端的发送、接收时刻相减，服务端耗时由自己的记录相减。两台机器的时间戳原点可能不同。下表把服务端 `received` 当作 0，并另记客户端的总耗时。索引查询结束以后完成后续处理，再交付回复，三个服务端阶段在题设中依次发生。

| 记录 | 分类 | 开始查询 | 查询结束 | 交付回复 | 客户端耗时 |
| --- | --- | ---: | ---: | ---: | ---: |
| 甲 | A | 1 | 3 | 6 | 15 |
| 甲 | B | 4 | 6 | 9 | 18 |
| 乙 | A | 1 | 3 | 6 | 15 |
| 乙 | B | 1 | 6 | 9 | 18 |

表中时间单位为毫秒。甲和乙代表两个独立的情形，不是同一次实验前后相邻的四条记录。仅看客户端，两者都是 A 等 15 毫秒、B 等 18 毫秒。加入服务端记录后，结论分开了。

- **甲的差值出现在排队阶段。** B 晚 3 毫秒取得工作线程，随后的索引都花 2 毫秒。应核对工作池、请求到达时的队列长度和路由策略。即使另有计数显示 B 的缓存活动不同，这份记录也没有把计数差和额外耗时连在一起。
- **乙的差值出现在索引阶段。** 两类都在第 1 毫秒开始查询，A 用 2 毫秒，B 用 5 毫秒。接下来应检查这个窗口内执行的工作。它可能多走了数据结构中的节点，也可能等了更多未命中。要往缓存解释推进，先核对加载序列和对应级别的计数，再检查是否有下级返回或[服务端共享队列](./microarch-06b-memory-queue.md#memory-evidence)的影响。

两份记录都把多出的 3 毫秒归到了服务端的一段工作。它们还没有提供对实际服务的重复观测。真实审计中，量级有多大、分布是否重叠、负载或请求条件改变后是否重现，均要由对应的数据回答。一个缩短了整体耗时的改动，也可能让两类之间的差值保持原样，所以复查时两项都要记录。

### 用代码从原始记录算出差值

下列 Python 3 程序只读取上面写在文章里的数据。`grouped_ms` 将一个分组的时间按次数展开，用于计算汇总和同网络结果。`phase_durations` 按字段顺序取相邻时间戳的差。对同一条请求，三段时间之和等于 `sent - received`。

```python
from statistics import mean, median

# 每项为（办公室次数，远程次数）。
counts = {"A": (180, 20), "B": (20, 180)}


def grouped_ms(office, remote):
    return [2] * office, [30] * remote


for label, counts_by_network in counts.items():
    office, remote = grouped_ms(*counts_by_network)
    pooled = office + remote
    balanced_mean = (mean(office) + mean(remote)) / 2
    print(label, median(pooled), mean(pooled), balanced_mean)

# 每项为（收到，开始查询，结束查询，交付回复）。
a = (0, 1, 3, 6)
b_waits = (0, 4, 6, 9)
b_lookup = (0, 1, 6, 9)


def phase_durations(stamps):
    assert len(stamps) == 4
    durations = tuple(later - earlier for earlier, later in zip(stamps, stamps[1:]))
    assert all(duration >= 0 for duration in durations)
    return durations


def phase_difference(before, after):
    return tuple(b - a for a, b in zip(phase_durations(before), phase_durations(after)))


assert phase_difference(a, b_waits) == (3, 0, 0)
assert phase_difference(a, b_lookup) == (0, 3, 0)
print("queue, lookup, reply")
print("case one:", phase_difference(a, b_waits))
print("case two:", phase_difference(a, b_lookup))
```

程序先输出每类的汇总中位数、汇总平均值和按相同权重计算的平均值。末尾两行差值按排队、查询、回复的顺序排列。甲的首项是 3，乙的第二项是 3。同样一个端到端变化，可以落在不同的接口。真实服务中的各阶段如有异步重叠，要先明确是否在同一条关键路径上，才能按本例这样拆分。

## 缓存解释要补什么证据 {#evidence-boundary}

如果维护者查到的是乙，可以先把服务端证据说完整。两类用的是同一段索引代码吗？访问的节点数是否一样？假设节点数相同，仍可能因为数据刚用过或位于不同缓存块而有不同的内存等待。可以用[缓存行与复用距离](./microarch-01-cache.md#locality-reuse)解释候选机制，再用[访存计数的对照](./microarch-04a-measurement.md#pmu-attribution)判断选定事件覆盖了哪个窗口。

举一个只用于验证解释的对照。若索引查询前的初态经过统一，随后记录到索引阶段的额外缓存未命中与耗时差一起缩小，缓存驻留的解释会得到支持。若未命中差缩小而耗时差保留，就还需要检查所计级别、CPU 执行或排队。这样做是把一个机制产生的变化追到时间窗口里。要评价对外可见性，仍要用客户端这一侧可见的量，核对差值能否在其运行条件下复现。内部计数没有自动成为外部可读的信号。

::: details 练习：改变一个阶段，复查结论

从甲开始，假定维护者调整工作线程的分配策略，使 B 与 A 都在第 1 毫秒取得工作线程。B 的索引和回复阶段保持原来的长度，服务端记录范围之外的等待也保持原样。用代码复算 B 的四个时间戳。若客户端仍看到 18 毫秒，原解释还缺哪一步？

B 的新记录是 `(0, 1, 3, 6)`，三个阶段与 A 相同，题设下客户端总耗时应从 18 变为 15 毫秒。如果客户端仍看到 18 毫秒，已有服务端记录少解释了 3 毫秒，应回到请求 ID 和两边的测量窗口，检查服务端记录以外的等待是否也发生了变化。
:::

## 参考文献

- [Intel 对侧信道与威胁模型的说明](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/best-practices/refined-speculative-execution-terminology.html)
- [NIST 工程统计手册，随机区组设计](https://www.itl.nist.gov/div898/handbook/pri/section3/pri332.htm)
- [OpenTelemetry 跟踪数据模型，操作起止时间和标识](https://opentelemetry.io/docs/specs/otel/trace/data-model/)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>上一篇</small><strong>用计时和计数器分析访存</strong></span>](./microarch-04a-measurement.md){.article-nav-card .article-nav-prev rel=prev}

[<span class="article-nav-text"><small>下一篇</small><strong>不同的数据为什么会在末级缓存里互相挤掉</strong></span>](./microarch-06a-shared-cache.md){.article-nav-card .article-nav-next rel=next}

</nav>
