---
title: 预取器和数据缓存怎样握手
description: 对照 Chipyard BAR-fetchers，逐拍追踪预取候选入队、缓存端口仲裁、地址检查和重试。
date: '2026-11-25'
tags: [微架构安全, 硬件预取, Chipyard, RTL]
book: microarch-00-intro
---

步长器已经提出地址 `0x1288`，数据缓存却正在接收核心的请求。预取器能否在这时记下地址？如果缓存先接收了预取请求，过两拍又要求重试，地址该由谁保管？这一节读取 Chipyard 可用的 BAR-fetchers RTL，看清预测器和缓存之间的这段接口。例题沿用上一节每隔 `0x80` 字节读一个字段的访问流，地址和调度均由题设给定。

[[toc]]

## 看清观察线和请求线 {#rtl-interfaces}

BAR-fetchers 支持多种接法。本文只取接在 Rocket 核心侧与非阻塞 L1 数据缓存之间的 `HellaCachePrefetchWrapperModule`。这个包装器直接转接核心和缓存的大部分信号，同时分出观察线，把核心的访问送给步长器。步长器沿另一条线交回候选，经一项队列进入缓存原有的请求端口。

<div role="region" aria-label="预取器和数据缓存的接口" style="max-width:100%; overflow-x:auto">

| 接口或部件 | 接收什么 | 留下什么或交给谁 |
| --- | --- | --- |
| 核心侧请求入口 `io.cpu.req` | 请求地址、命令和其他普通访存字段 | 核心有有效请求时优先使用数据缓存入口 |
| 旁路观察 `snoop` | 通过包装器筛选的地址与读写标志 | 向步长器给出一次 `valid` 事件 |
| 步长器输出 `request` | 步长器内部算出的候选 | 输出地址与读写标志，由暂存队列的 `ready` 决定该拍是否接收 |
| 一项队列 `req` | 握手成功的候选 | 保管地址与读写标志，跨过缓存入口仲裁和 `s2` 校验 |
| 包装器的 `in_flight` | 预取在缓存入口发生的握手、对应的 `s2` 事件 | 标记是否已有一次尝试尚待检查 |
| 数据缓存 `cache.io.cpu` | 对齐后的地址、预取命令和请求属性 | 走缓存的地址翻译与访存路径，回报物理地址和 `s2_nack` |

</div>

有两份同名的 `req`。`io.cpu.req` 指核心提出的需求请求，源码里的局部变量 `req` 则是预取暂存队列的出队端。下文把后者简称为队列。最容易读错的信号是队列的 `req.ready`。它控制何时释放已占用的队列槽，与缓存入口的 `ready` 属于两个接口。

观察线固定延迟两拍。包装器把核心入口的 `fire`、当拍命令与地址对齐到 `s2`。已经接受的核心请求，如果本身是预取命令，或者途中被 `s1_kill` 取消、在 `s2` 收到 `nack`，就不向步长器送出有效观察。`snoop` 是 `Valid` 接口，观察方没有 `ready` 可以要求核心等一拍。缓存命中和未命中的结果也没有作为字段送入这条线。

## 单流步长器怎样提出候选 {#rtl-stride}

步长器接收的是观察地址与读写标志。接口里没有指令 PC，交错访问都进入同一份状态。对照[按 PC 分流的 gem5 步长表](./microarch-02-prefetch.md#prefetch-stride)，这里可以直接看出，决定是否能够维护多条流的不只是算法公式，还有观察接口和寄存器组织。

<div role="region" aria-label="单流步长器的状态" style="max-width:100%; overflow-x:auto">

| 状态 | 在什么时刻参与计算 |
| --- | --- |
| `last_snoop`、`last_snoop_2` | 保存最近两次观察。新地址分别与两者求差，任一差值匹配当前 `delta`，历史计数就增加 |
| `last_delta`、`last_delta_pos` | 保存与最近一次观察的差值和方向，供空闲状态更新步长、进入活跃时记下方向 |
| `history_cnt` | 有效观察到来时增减，默认阈值为 4，计数饱和到 7。低于阈值时回到空闲状态 |
| `prefetch`、`pref_far_enough`、`last_write` | 分别保存候选地址、候选是否离最近观察足够远的注册状态和最近观察的读写属性 |
| `state`、`reset_counter` | 在活跃时允许输出。候选已经走在访问流前面时可进入等待，等新观察追上来。等待过久可触发历史清零 |

</div>

设已记录的两个地址是 `0x1108` 和 `0x1188`，最近一次在 `0x1188`，保存的 `delta` 为 `0x80`。现在观察到 `0x1208`。它与最近一次的差是 `0x80`，与再前一次的差是 `0x100`，满足其中一项匹配。若旧计数为 3，且没有触发历史清零，这次更新后便是 4。候选地址生成使用最近一次观察与其差值，此时 `0x1208 + 0x80 = 0x1288`。进入活跃状态并满足注册的距离条件以后，候选才对队列提出 `valid`。这一过程从给定的寄存器初态推算，下面的逐拍例题以候选已经有效为起点。

这里距离检查涉及注册状态。`valid` 由活跃状态和 `pref_far_enough` 共同决定。正常送出时，候选沿 `delta` 向前推进，并按已经领先多远决定是否进入等待。观察仍可能在此期间更新，`request` 使用的是 `Decoupled` 接口。对这个接口来说，取样时刻是 `valid && ready` 成立的那一拍。不能把前几拍单独看见的地址当作已经接收的队列内容。

## 三个事件，三个不同的判据 {#rtl-handshake}

**候选入队。** 步长器给出 `valid`，一项队列为空时提供 `ready`。两者同时为 1 时，队列在时钟边沿保存地址和读写属性。此处 `Queue(..., 1)` 用默认的 `pipe=false` 和 `flow=false`。空槽刚接收的地址要到下一拍才从出队端可见，满槽释放的当拍也不会用另一个候选补进来。

**缓存入口接收。** 只有核心的 `req.valid` 为 0，且队列非空、`in_flight` 为 0 时，包装器才用队首向缓存端口置 `valid`。缓存入口此时也给 `ready`，才发生第二次握手。包装器把 `0x1288` 向下对齐为 64 字节块首址 `0x1280`。读候选使用 `M_PFR`，写意图候选使用 `M_PFW`，请求指定 `phys=false`，沿缓存的地址翻译路径继续。握手以后，包装器置位 `in_flight`，地址还留在原槽里。

**确认是否释放队列。** 缓存入口握手经过两拍到达 `s2`。包装器依据 `s2_paddr` 查询有没有合法的下游管理者，以及相应地址的管理者是否具备这里要求的 `AcquireT` 能力。检查失败就置 `s2_kill`。槽位释放却取决于另一根线。若缓存送出 `s2_nack`，这一项留在队列；否则队列出队，无论本次是否被 `s2_kill` 取消。两种情况下 `in_flight` 都清零，下一拍才能提出别的尝试。

因此，看到队列出队，只能确认包装器已经处理完这次入口尝试。判断是否发生缺失请求、是否分配 MSHR、何时返回填充，还要继续看[缓存的未命中和填充路径](./microarch-02-prefetch.md#prefetch-fill)。这个包装器也会屏蔽带预取命令的缓存响应，避免把它交给核心当成一次普通加载的结果。

## 逐拍跟踪一条被拒绝的请求 {#rtl-trace}

设预测器起初提出 P（`0x1288`）。第 5 拍起，它还可以提出 Q（`0x1308`），直到 Q 入队。包装器初始为空且未在尝试。除第 2 拍外，缓存入口都就绪。表中队列和 `in_flight` 是**当拍开始时**的值，写入寄存器的结果到下一拍可见。表里的合法性与 `nack` 都是为复算接口而给定的缓存侧条件。

<div role="region" aria-label="包装器处理 P 和 Q 的时序" style="max-width:100%; overflow-x:auto">

| 拍 | 核心请求有效 | 队列与 `in_flight` | 发生的事 |
| ---: | --- | --- | --- |
| 0 | 是 | 空，0 | P 可以入队。缓存入口给核心 |
| 1 | 是 | P，0 | P 留在原槽。缓存入口仍给核心 |
| 2 | 否 | P，0 | 包装器提出 P，缓存 `ready=0`，没有握手 |
| 3 | 否 | P，0 | 缓存入口接受 P 的块地址 `0x1280`，开始第一次尝试 |
| 4 | 否 | P，1 | 第一次尝试走到 `s1`，队列仍在保管 P |
| 5 | 否 | P，1 | `s2_nack=1`，留下 P，清除 `in_flight`。Q 遇到满槽 |
| 6 | 否 | P，0 | P 在缓存入口再次握手 |
| 7 | 否 | P，1 | 第二次尝试走到 `s1` |
| 8 | 否 | P，1 | `s2` 合法且无 `nack`，释放槽位。Q 仍不能在同一拍入队 |
| 9 | 否 | 空，0 | Q 终于入队 |
| 10 | 否 | Q，0 | 缓存入口接受 Q 的块地址 `0x1300` |
| 11 | 否 | Q，1 | Q 走到 `s1` |
| 12 | 否 | Q，1 | Q 的地址检查失败，且无 `nack`。置 `s2_kill` 并释放槽位 |

</div>

第 2 和第 5 拍都会导致等待，但发生在不同的路径。第 2 拍尚未握手，`in_flight` 一直是 0。第 5 拍是缓存对第 3 拍已接收的请求作出延迟拒绝，包装器清除 `in_flight`，凭保留的 P 发起下一次尝试。Q 第 12 拍被取消的条件也不同。无 `nack` 让它出队，`s2_kill` 让缓存停止处理这次尝试。

### 用 C 复算暂存槽的变化 {#rtl-model}

程序从候选已经有效开始。`slot.valid` 对应队列里是否有候选，`reply_cycle` 对应这次尝试到达 `s2` 的拍号。它为负时没有尚待检查的尝试。每轮先用拍首的状态计算两个握手，再修改寄存状态。这样即使本拍出队，下一项也要等到下一拍才可以入队。

保存为 `prefetch_handshake.c`，用 `gcc -std=c11 -O2 -Wall -Wextra prefetch_handshake.c -o prefetch_handshake` 编译，再运行 `./prefetch_handshake`。Windows 上可运行 `./prefetch_handshake.exe`。

~~~c
#include <stdbool.h>
#include <stdio.h>

typedef struct {
    bool valid;
    unsigned address;
} Slot;

static void event(int cycle, const char *name, unsigned address)
{
    printf("%2d  %-15s  0x%04x\n", cycle, name, address);
}

static void replay(int nack_cycle, int kill_cycle)
{
    Slot slot = {0};
    int reply_cycle = -1;
    bool q_pending = true;

    for (int cycle = 0; cycle <= 12; ++cycle) {
        bool core_valid = cycle <= 1;
        bool cache_ready = cycle != 2;
        bool p_valid = cycle == 0;
        bool q_valid = cycle >= 5 && q_pending;
        bool enqueue = !slot.valid && (p_valid || q_valid);
        bool fire = slot.valid && reply_cycle < 0 && !core_valid && cache_ready;
        bool at_s2 = cycle == reply_cycle;

        if (fire) {
            event(cycle, "cache-fire", slot.address & ~63u);
            reply_cycle = cycle + 2;
        }

        if (at_s2) {
            bool nack = cycle == nack_cycle;
            bool kill = cycle == kill_cycle;
            const char *outcome = kill ? "kill-and-pop" : "pop";
            if (nack)
                outcome = kill ? "kill-and-retain" : "retain";
            event(cycle, outcome, slot.address);
            slot.valid = nack;
            reply_cycle = -1;
        }

        if (enqueue) {
            slot = (Slot){true, p_valid ? 0x1288u : 0x1308u};
            event(cycle, "enqueue", slot.address);
            if (q_valid)
                q_pending = false;
        }
    }
}

int main(void)
{
    replay(5, 12);
}
~~~

~~~text
 0  enqueue          0x1288
 3  cache-fire       0x1280
 5  retain           0x1288
 6  cache-fire       0x1280
 8  pop              0x1288
 9  enqueue          0x1308
10  cache-fire       0x1300
12  kill-and-pop     0x1308
~~~

对照第 3 拍的 `cache-fire` 与第 5 拍的 `retain`。前者记录交给缓存的块首址，后者记录留在队列里的原始候选地址。对于读取模拟波形或计数器的人，这两行对应两类事件。用端口接受次数统计，本例 P 被接受两次。用候选入队次数统计，P 只入队一次。至于下级取行和 L1 填充，程序没有给定这些事件，表里也推不出它们的次数。

::: details 练习：第一次尝试没有被拒绝

把代码改为 `replay(-1, 12)`，其余输入保持不变。P 和 Q 会在哪一拍进入缓存？注意 `kill_cycle` 指定的是拍号。

P 仍于第 3 拍进入缓存，但第 5 拍就出队。第 6 拍 Q 入队，第 7 拍 Q 进入缓存，第 9 拍到达 `s2`。如果要保持 Q 的地址检查结果不变，应调用 `replay(-1, 9)`。这次变化来自 P 提前释放暂存槽，候选本身没有改变。
:::

## 参考文献

- [Chipyard 的预取器接入位置与实现](https://chipyard.readthedocs.io/en/stable/generators/prefetchers/)
- [BAR-fetchers 观察和请求接口，固定版本](https://github.com/ucb-bar/bar-fetchers/blob/bd02692fe6cb5197df90dc39051d31369ef61fcd/src/main/scala/AbstractPrefetcher.scala)
- [BAR-fetchers 单流步长器 RTL，固定版本](https://github.com/ucb-bar/bar-fetchers/blob/bd02692fe6cb5197df90dc39051d31369ef61fcd/src/main/scala/Strided.scala)
- [BAR-fetchers HellaCache 包装器 RTL，固定版本](https://github.com/ucb-bar/bar-fetchers/blob/bd02692fe6cb5197df90dc39051d31369ef61fcd/src/main/scala/HellaCachePrefetcher.scala)
- [Chisel 的 Ready/Valid 接口](https://www.chisel-lang.org/docs/explanations/interfaces-and-connections)
- [Chisel Queue 的深度、pipe 与 flow 参数](https://www.chisel-lang.org/api/latest/chisel3/util/Queue%24.html)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>上一篇</small><strong>硬件预取器怎样工作</strong></span>](./microarch-02-prefetch.md){.article-nav-card .article-nav-prev rel=prev}

[<span class="article-nav-text"><small>下一篇</small><strong>处理器怎样用页表和 TLB 翻译地址</strong></span>](./microarch-02a-translation.md){.article-nav-card .article-nav-next rel=next}

</nav>
