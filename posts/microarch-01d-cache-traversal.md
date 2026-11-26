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

## 用两个循环复算访问序列

下面的 Python 程序只生成一次求和会发出的加载地址。两段循环枚举相同的元素，区别在于哪个下标先变化。`miss_count` 用块号对 64 取余定位缓存组。每组从左到右保存最近到最久使用的块号，块命中时先移到最左边，组满时丢掉最右边。每次调用都重新创建一个空缓存，因此三次结果可直接比较。

```python
ROWS = 9
COLS = 16
BASE = 0x20000
ELEMENT_BYTES = 4
LINE_BYTES = 64
SET_COUNT = 64
WAYS = 8


def row_order(stride):
    for row in range(ROWS):
        for col in range(COLS):
            yield BASE + row * stride + col * ELEMENT_BYTES


def column_order(stride):
    for col in range(COLS):
        for row in range(ROWS):
            yield BASE + row * stride + col * ELEMENT_BYTES


def miss_count(addresses):
    groups = [[] for _ in range(SET_COUNT)]
    misses = 0
    for address in addresses:
        block = address // LINE_BYTES
        group = groups[block % SET_COUNT]
        if block in group:
            group.remove(block)
        else:
            misses += 1
        group.insert(0, block)
        if len(group) > WAYS:
            group.pop()
    return misses


print("先行后列", miss_count(row_order(4096)))
print("先列后行", miss_count(column_order(4096)))
print("增加填充", miss_count(column_order(4160)))
```

```text
先行后列 9
先列后行 144
增加填充 9
```

先行后列的前两笔地址为 `0x20000` 和 `0x20004`，落在同一行。先列后行的前两笔为 `0x20000` 和 `0x21000`，各占一行，却落在同一个组。加填充后，第二笔改为 `0x21040`，落到下一组。改变的 64 字节只出现在行跨度中，对所读元素的行、列下标没有影响。

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
