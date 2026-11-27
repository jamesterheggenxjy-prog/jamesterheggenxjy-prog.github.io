---
title: 矩阵按行读和按列读的缓存开销
description: 把 9 行矩阵的具体访问序列映射到缓存组，复算未命中并估计两级访问代价。
date: '2026-11-25'
tags: [微架构安全, 高速缓存, 矩阵遍历, 性能模型]
book: microarch-00-intro
---

在[组内替换的例题](./microarch-01b-cache-mapping.md#cache-replacement)里，单个组只能保留一部分活跃块。将这条规律应用到实际常见的双层循环上，可以从数组的行跨度推导哪一组容易被反复占满，再估算访问时间。

[[toc]]

## 矩阵遍历的缓存行为

设一个以行优先方式存储的矩阵，每行 1024 个 4 字节元素，例题取起始地址 `0x20000`，已对齐到 64 字节。程序要读取前 9 行、每行前 16 列的子矩形。每个元素的地址等于 `0x20000 + 行号 × 行跨度 + 列号 × 4`。采用[地址映射一节](./microarch-01b-cache-mapping.md)的 32 KiB、8 路模型，起始缓存为空，按需求填充并使用精确 LRU。忽略循环的其他访存和硬件预取。两种循环的读取总次数都是 $9\times16=144$。

| 遍历方式 | 外层循环 | 内层循环 |
| --- | --- | --- |
| 先行后列 | 依次选行 | 读取当前行的前 16 列 |
| 先列后行 | 依次选列 | 读取当前列的前 9 行 |

每行前 16 个元素恰好占一条缓存行，整个子矩形只涉及 9 条不同的行。连续两行的首地址相差 $1024\times4=4096$ 字节，相邻块号相差 $4096/64=64$。组号是块号对 64 取余，所以这 9 行都映射到同一组。

**先行后列。** 读取某一行的第一个元素时引起行填充，紧接着对该行其余 15 个元素的读取都命中。进入下一行时，即使发生替换，已经读完的行也不再需要。总计 9 次未命中，135 次命中，未命中率为 $9/144=6.25\%$。

**先列后行。** 第一列依次装入块 $A_0$ 到 $A_8$。第 9 个块进入时，第 0 个块已经被移出。到第二列，程序恰好先要访问 $A_0$。填回它又会移出 $A_1$，接下来对 $A_1$ 的请求再次未命中。也可以从任意一块看。两次使用它之间恰有另外 8 个同组块被访问，而该组的容量只有 8。等到下次使用，这块已经不在组中。144 次需求读取全部未命中。首访 9 次，其余 135 次可归为冲突未命中。

**改变布局。** 若每行增加 64 字节填充，逻辑元素仍为 1024 个，则行跨度从 4096 增到 4160 字节。前 16 列仍各占一行。相邻块号增加 65，组号每次相差 1，9 行落入连续的 9 个组。在题设下，先列后行也只需 9 次填充。这个例题把现象定位到组索引和访问顺序。单看数组总大小或访问总次数都无法得到这一结论。

### 把行跨度换成组号 {#matrix-stride-sets}

基址 `0x20000` 除以 64 得到块号 `0x800`，它对 64 取余为 0。下表只列每行第一个元素。前 16 个元素占满这条行，所以表中的组号也适用于本题读到的该行其他元素。

| 行号 | 原跨度 4096 字节的行首／组号 | 填充后跨度 4160 字节的行首／组号 |
| ---: | --- | --- |
| 0 | `0x20000`／0 | `0x20000`／0 |
| 1 | `0x21000`／0 | `0x21040`／1 |
| 2 | `0x22000`／0 | `0x22080`／2 |
| 8 | `0x28000`／0 | `0x28200`／8 |

例如填充后第 2 行的行首是 `0x20000 + 2 × 0x1040 = 0x22080`。块号为 `0x882`，对 `0x40` 取余得到组 2。这里每次跨 65 块，而 65 除以 64 余 1，所以组号逐行加一。表中的填充字节位于每个逻辑行之后，程序读取的仍是每行前 16 个元素。

::: details 练习｜每行多填充两条缓存行

只把每行的填充从 64 字节改为 128 字节。第 0、1、2 行的组号分别是什么？这 9 条被读取的行还会都落在同一组吗？

新跨度为 4224 字节，即 66 个块。组号每次增加 2，因此前三行为 0、2、4，前 9 行分别落在组 0、2、4、6、8、10、12、14、16。它们不再全挤在第 0 组。本题仅讨论给定的低位组索引，其他处理器可能采用不同映射。

继续增加矩阵行数时，还要看组号何时绕回。128 字节填充使组号每次加 2，对 64 取余后依次经过 0、2、4，直到 62，再回到 0。因此只有 32 个偶数组会被用到，第 32 行与第 0 行重新落在同一组。64 字节填充时每次加 1，经过全部 64 组后，第 64 行才与第 0 行同组。两种布局对前 9 行都消除了原来的九行同组冲突，但扩展到更多行以后，同组行数还须按各自的循环周期计算。

例如，在 128 字节填充的布局中读取前 288 行的首个缓存块，每个被用到的偶数组恰好接收 9 个不同块。如果访问次序让这 9 个块依次进入同一组，八路 LRU 组无法同时保留它们。下一轮仍按相同次序访问时，第一个块会先替换上一轮最旧的驻留块，后面的访问继续依次替换，因而该组的 9 次访问都未命中。这里仍假设冷启动、没有预取与其他访存；结论来自具体的组数、相联度和访问次序。
:::

### 组号绕一圈要多少行 {#matrix-set-period}

设缓存共有 $S$ 组，每一行矩阵的首地址比上一行多 $d$ 个缓存块。只要行跨度是缓存行长的整数倍，读各行的同一列时，块号就每次加 $d$。组号要回到原处，最少需要走多少步？如果走了 $k$ 步，块号多了 $kd$，因此要找最小的正整数 $k$，使 $kd$ 可以被 $S$ 整除。将 $S$ 和 $d$ 同除以最大公约数 $g$ 后，$S/g$ 和 $d/g$ 互质，故最小的 $k$ 是 $S/g$。这也是一圈内经过的不同组数。

用这个周期重算一个介于两例之间的尺寸。缓存仍是 64 组、八路精确 LRU，行跨度仍是 66 个块。程序依次读前 260 行的第一列，再用同样顺序读一遍。组号每 32 行绕回一次，而 $260=8\times32+4$。所以前四个组 0、2、4、6 各收到 9 个块，其余 28 个偶数组各收到 8 个块。第二遍中，前四组每次重用一块都隔着另外 8 个同组块，各有 9 次未命中，合计 36 次。其余各组恰好留得住自己的 8 个块，第二遍全部命中。未用到的奇数组里虽然有空位，这四组的块仍只能分配到各自的组内。

::: details 练习｜同样是 260 行，只填充一条缓存行
将每行跨度改为 65 个块，保留其他条件。第二遍有多少次未命中？

$\gcd(65,64)=1$，组号每 64 行绕回。$260=4\times64+4$，前四组各有 5 块，其余各组各有 4 块，全部低于八路的容量限制。题设下第二遍没有未命中。若行跨度含有不足一块的部分，就要从每次的字节地址向下取整再算块号，本节的固定块步长公式不再适用。

用后文的 C 程序复算时，将 `ROWS` 改为 260、`COLS` 改为 2。列 0 和列 1 在每行访问同一个缓存块，因此保留按列循环就能得到相同的块号序列。在 `main` 中分别打印 `count_misses(4224, true, 8, ROWS)` 和 `count_misses(4160, true, 8, ROWS)`，应得到 296 和 260。两者都包含第一遍冷缓存的 260 次未命中。
:::

### 不改矩阵布局，也能减少按列读取的未命中 {#matrix-row-tiles}

保持 4096 字节的原始行跨度。假定计算允许交换不同元素的处理先后，例如用无符号整数做求和。现在先处理第 0 至 7 行，再处理第 8 行。每批内部仍然逐列访问，变化的是一轮要反复使用多少条同组缓存行。

| 当前批 | 第 0 列读完时 | 其余 15 列 | 本批未命中 |
| --- | --- | --- | ---: |
| 第 0 至 7 行 | 第 0 组恰好存下 A0 至 A7 | 每次重用一块之前只访问另外 7 块，均命中 | 8 |
| 第 8 行 | 首次访问 A8，换出 A0 | 其余元素连续命中 A8，上一批已用完 A0 | 1 |

读取总数仍是 144，涉及的地址也完全相同，未命中从 144 降到 9。这个变化只调整了访问次序。前面的行尾填充则调整地址与组号的对应关系。两种做法在本题里得到相同的未命中数，适用条件各有不同。某些计算要求完成整列后才能处理下一列，此时跨列调整顺序要先核对依赖。

::: details 练习｜只分列，还是一次访问全部九行

将 16 列分成两批，每批 8 列，各批内部仍按列扫描全部九行。未命中会降到 9 吗？

不会。访问顺序仍是第 0 列到第 15 列，每列依次访问 A0 至 A8。相邻两次使用同一块之间仍隔着另外 8 个同组块，精确 LRU 下仍是 144 次未命中。

:::

## 用两个循环复算访问序列

下面用 C11 复算不同的遍历方式和节末的 16 路练习。用 `group` 找到组，再在该组比较标记。每组的有效路从最近使用排到最久使用。组未满时，新行占用一个空位置。组已满时，最后一个位置的旧标记被覆盖。每次遍历都从一份空缓存开始。

将代码保存为 `matrix_cache.c`，执行 `gcc -std=c11 -O2 -Wall -Wextra matrix_cache.c -o matrix_cache` 后运行。

```c
#include <stdbool.h>
#include <stdio.h>

enum { ROWS = 9, COLS = 16, BASE = 0x20000, LINE_BYTES = 64 };
enum { LINES = 512, MAX_SETS = 64, MAX_WAYS = 16 };

struct Set {
    unsigned valid;
    unsigned tag[MAX_WAYS];
};

static unsigned touch(struct Set sets[], unsigned address,
                      unsigned set_count, unsigned ways)
{
    unsigned block = address / LINE_BYTES;
    unsigned group = block % set_count;
    unsigned tag = block / set_count;
    struct Set *set = &sets[group];

    unsigned rank = 0;
    while (rank < set->valid && set->tag[rank] != tag)
        ++rank;
    bool miss = rank == set->valid;

    if (miss) {
        if (set->valid < ways)
            ++set->valid;
        else
            rank = ways - 1;
    }
    for (unsigned i = rank; i > 0; --i)
        set->tag[i] = set->tag[i - 1];
    set->tag[0] = tag;
    return miss;
}

static unsigned element_address(unsigned row, unsigned col, unsigned stride)
{
    return BASE + row * stride + col * 4;
}

static unsigned count_misses(unsigned stride, bool by_column, unsigned ways,
                             unsigned rows_per_tile)
{
    struct Set sets[MAX_SETS] = {0};
    unsigned set_count = LINES / ways;
    unsigned misses = 0;

    if (by_column) {
        for (unsigned first = 0; first < ROWS; first += rows_per_tile) {
            unsigned end = first + rows_per_tile;
            if (end > ROWS)
                end = ROWS;
            for (unsigned col = 0; col < COLS; ++col)
                for (unsigned row = first; row < end; ++row)
                    misses += touch(sets, element_address(row, col, stride),
                                    set_count, ways);
        }
    } else {
        for (unsigned row = 0; row < ROWS; ++row)
            for (unsigned col = 0; col < COLS; ++col)
                misses += touch(sets, element_address(row, col, stride),
                                set_count, ways);
    }
    return misses;
}

int main(void)
{
    printf("by row, 8 ways: %u\n", count_misses(4096, false, 8, ROWS));
    printf("by column, 8 ways: %u\n", count_misses(4096, true, 8, ROWS));
    printf("padded, by column, 8 ways: %u\n", count_misses(4160, true, 8, ROWS));
    printf("by column, 16 ways: %u\n", count_misses(4096, true, 16, ROWS));
    printf("8 rows per tile, by column: %u\n", count_misses(4096, true, 8, 8));
    return 0;
}
```

```text
by row, 8 ways: 9
by column, 8 ways: 144
padded, by column, 8 ways: 9
by column, 16 ways: 9
8 rows per tile, by column: 9
```

下表把程序中最关键的三次组内更新单独列出。按列读取且行跨度为 4096 字节时，`0x20000 / 64 = 0x800`，组号为 0，标记为 `0x20`。每向下走一行，标记加一，组号仍为 0。因此表中的 A0、A1 分别对应标记 `0x20`、`0x21`。

| 加载结束后 | 第 0 组，从最近排到最久 | 这次淘汰的行 |
| --- | --- | --- |
| 第 0 列，第 7 行 | A7、A6、A5、A4、A3、A2、A1、A0 | 无 |
| 第 0 列，第 8 行 | A8、A7、A6、A5、A4、A3、A2、A1 | A0 |
| 第 1 列，第 0 行 | A0、A8、A7、A6、A5、A4、A3、A2 | A1 |

先行后列的前两笔地址为 `0x20000` 和 `0x20004`，落在同一行。先列后行的前两笔为 `0x20000` 和 `0x21000`，各占一行，却落在同一个组。加填充后，第二笔改为 `0x21040`，落到下一组。改变的 64 字节只出现在行跨度中，对所读元素的行、列下标没有影响。

最后一行输出来自 `rows_per_tile = 8`。外层循环第一次把行范围限制为 `[0, 8)`，第二次由 `end` 截到 `[8, 9)`。两批之间缓存状态保留，因此第二批仍会换出一个旧块。被换出的块已经用完，替换不会增加之后的缺失。

## 从命中率估计访问代价

设每次先查 L1，花费 $t_h$ 周期。L1 未命中后另外等待 $p_m$ 周期。设未命中率为 $m$。在串行且各次代价固定的简化模型中，$1-m$ 比例的访问花 $t_h$，$m$ 比例花 $t_h+p_m$。每次访问的平均时间可写为

$$
T_{\mathrm{avg}}=(1-m)t_h+m(t_h+p_m)=t_h+mp_m.
$$

取 $t_h=4$、$p_m=12$。上例先行后列的平均访问时间为 $4+0.0625\times12=4.75$ 周期，先列后行为 $4+1\times12=16$ 周期。两者相差的是模型下的平均服务代价。实际循环的运行时间还会受指令执行和在途请求重叠影响。读完[在途请求](./microarch-01a-mshr.md)与[预取候选](./microarch-02-prefetch.md#prefetch-queue)后，就能分别分析哪些等待可以重叠、哪些数据可能提前开始传输。

如果 L1 未命中还要先查 L2，L2 的未命中率应以**到达 L2 的请求**为分母。令 $t_1,t_2$ 分别为查找两级缓存的代价，$m_1$ 为所有请求中未命中 L1 的比例，$m_2$ 为到达 L2 后仍未命中的比例，$p_2$ 为 L2 未命中之后的额外等待。相同的逐层计费给出 $t_1+m_1(t_2+m_2p_2)$。例如 $m_1=10\%$、$m_2=20\%$ 时，原始请求中两级都未命中的比例是 $2\%$。

::: details 练习：保持缓存容量，改变相联度
在 9 行矩阵的例子中，先列后行访问。将相联度改为 16 路，仍用 32 KiB 容量和 64 字节行。新组数是多少？前两列各有几次未命中？

512 行分成 32 组。原来的跨度是 64 个块，仍然是组数的整数倍，9 个块同组。一组现在有 16 个位置，足以同时存下它们。第一列有 9 次首访未命中，第二列全部命中。
:::

## 参考文献

- [王道组成原理复习指导目录，第 3 章](https://phei.com.cn/module/goods/wssd_content.jsp?bookid=51711)
- [CS:APP 作者的矩阵访问示例](https://csapp.cs.cmu.edu/3e/perspective.html)
- [MIT 6.004 计算结构讲义](https://6191.mit.edu/_static/spring26/resources/references/6004_student_notes_FA20.pdf)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>上一篇</small><strong>缓存怎样查找和替换一行</strong></span>](./microarch-01b-cache-mapping.md){.article-nav-card .article-nav-prev rel=prev}

[<span class="article-nav-text"><small>下一篇</small><strong>缓存未命中后如何合并请求</strong></span>](./microarch-01a-mshr.md){.article-nav-card .article-nav-next rel=next}

</nav>
