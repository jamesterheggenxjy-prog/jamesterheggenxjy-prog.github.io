---
title: 写入缓存后，数据什么时候送到下一级
description: 从两条 RISC-V 存储算出覆盖的字节，跟踪写分配、脏位、替换和写回缓冲，再比较不同策略的流量。
date: '2026-11-25'
tags: [微架构安全, 高速缓存, 写分配, 写回]
book: microarch-00-intro
---

一条存储指令只改四个字节，缓存却可能先取回一整行，过一段时间又把一整行送往下一级。要追踪这两次传输，先看本级怎样合并新旧字节，再看缓存行离开数据阵列以后，谁保留最新值。

本文沿用[组相联缓存](./microarch-01b-cache-mapping.md#cache-address)的低位索引，行长 64 字节、64 组、每组两路，采用精确 LRU。例题只有一个核心。设所用虚拟地址和对应物理地址数值相同，请求已通过页权限检查，字节序为小端。先推演写分配与写回，再改变策略。涉及具体实现时，以 gem5 v24.1.0.3 的经典一致性缓存为对照。

[[toc]]

## 存储从哪里进入缓存 {#store-structure}

核心侧传来存储地址、数据和大小，标记与权限的检查决定数据阵列能否接下这笔存储。图 1 中，下端连接互连。MSHR 发出的缺失或权限请求，与写缓冲送出的写数据共用下行接口。

<figure class="lesson-figure" style="max-width: 374px">
<a href="/microarch/gem5-classic-cache.png"><img src="/microarch/gem5-classic-cache.png" alt="gem5 经典数据缓存结构，写缓冲和 MSHR 位于数据阵列与下级接口之间" width="374" height="257" loading="lazy" decoding="async"></a>
<figcaption>图 1　沿 Tags/Data 到 WriteBuffer，再到下端的互连接口看写回路径。</figcaption>
</figure>

这里从存储请求交到缓存开始。处理器后端的**存储队列**要处理地址与数据何时就绪、存储何时能提交，并可将尚未写入缓存的值转发给同核的后续加载。缓存内的**写回缓冲**在另一个接口，接手的是准备向下一级移交的数据，例如被替换的脏行。[存储队列和加载转发](./microarch-03d-load-store.md)一课会再看核心内部的部分。

## 缓存没有这一行时怎样写 {#store-allocation}

设 `a0 = 0x1000`、`t0 = 9`、`t1 = 12`。以下两条 `sw` 分别把源寄存器的低 32 位写到基址加立即数得到的地址。设下一级中 A 行的范围为 `0x1000` 至 `0x103f`，其中连续三个字的初值为 5、11、17。

```asm
sw t0, 32(a0)
sw t1, 36(a0)
```

| 行内偏移 | 下一级原有字节 | 第一条写入后，本级字节 | 第二条写入后，本级字节 |
| --- | --- | --- | --- |
| `0x20` 至 `0x23` | `05 00 00 00` | `09 00 00 00` | `09 00 00 00` |
| `0x24` 至 `0x27` | `0b 00 00 00` | `0b 00 00 00` | `0c 00 00 00` |
| `0x28` 至 `0x2b` | `11 00 00 00` | `11 00 00 00` | `11 00 00 00` |

两条指令都访问起点为 `0x1000` 的块。请求范围由行内偏移和 4 字节长度决定。若用 64 位掩码表示该行的字节写使能，第一条打开第 32 至 35 位，第二条打开第 36 至 39 位。其他字节的使能为零，所以表中的第三个字仍是 17。

### 写分配与不写分配

初始 L1 没有 A。采用**写分配**（write allocate）时，缓存记录这笔缺失事务，等待旧数据和所需的一致性写权。两条指令都是局部写入，未被覆盖的字节要保留。本题下级按整行返回，所以第一条存储要先取得 A 的 64 字节，再把四个新字节合入。第二条存储到达时，A 已经有效且可写，直接在本级更新。

在 gem5 经典缓存的常规路径中，没有有效副本且需要写权时，`Cache::createMissPacket` 可以构造整行 `ReadExReq`，取得数据和可写副本。若已经有有效数据但缺写权，可以用 `UpgradeReq` 申请许可，原有数据仍可使用。如果请求覆盖整行且实现识别出这一点，还能走省去读取旧内容的路径。本例的四字节写入仍保留传统的整行填充。

采用**不写分配**（no write allocate）时，未命中的存储交给下一级处理，本级不保留该行。若下一级支持四字节更新，就在那里应用本次存储的范围。随后第一次从本级读 A，仍会因没有本地副本而未命中。

## 脏位决定替换时怎样处理数据 {#store-dirty-bit}

**写直达**（write-through）会将本级更新也交给下一级，可以通过缓冲排队。**写回**（write-back）允许更新暂时只留在本级，用脏状态记录还需转交的修改。选择写入时是否分配，与选择何时传播更新，是缓存的两项决策。

| 行状态 | 对本例有什么影响 |
| --- | --- |
| 有效位 | 说明当前位置能否作为缓存副本，即使标记和数据阵列里仍留着旧内容 |
| 可读、可写状态 | 有效且标记匹配的行仍要具备该次访问所需的权限 |
| 脏位或等价状态 | 指出修改责任仍在本级。替换时要把数据及相应责任移交给后续路径 |

写回模式下，第一条 `sw` 写入后，A 已经是脏行，下一级仍是 5。第二条 `sw` 更新同一行的数据，脏状态保持为真。直到缓存需要回收这一行，两次修改才被放进同一个写回包。一位脏标记只说明整行有待转交的内容，无法指出是哪几个字节被改过。若接口按整行传输，两次四字节写入最后也会发出 64 字节的写回。

## 两次写入后，哪一行被换掉 {#store-trace}

令 B、C 的起点为 `0x2000`、`0x3000`。三个起点相隔 4096 字节，即 64 行，恰好绕组索引一圈，均落在第 0 组。读请求在下一次访问前完成填充。写回包可以在填充后继续等下行接口。下表采用写分配和写回，最近访问的行写在左边。

| 本次操作 | 组内状态 | 下一级中 A 的两个待改字 | 此次接口动作 |
| --- | --- | --- | --- |
| 读 B | B（干净） | 5、11 | 取回 B |
| `sw t0, 32(a0)` | A（脏）、B（干净） | 5、11 | 取回 A，写入 9 |
| `sw t1, 36(a0)` | A（脏）、B（干净） | 5、11 | 本级写入 12 |
| 再读 B | B（干净）、A（脏） | 5、11 | 本级命中，B 更新使用顺序 |
| 读 C 并完成填充 | C（干净）、B（干净） | 5、11 | 取回 C，替换 A，将 A 的整行暂存进写回缓冲 |
| 下一级处理 A 的写回 | C（干净）、B（干净） | 9、12 | 发送并处理一次 A 的整行写回 |

第五步后，A 已离开数据阵列，下一级仍存旧数据。新值 9、12 和没改过的 17 都在等待发送的副本中。写回的起始地址由被替换行的标记与第 0 组反求，得到块起点 `0x1000`。`0x1020` 只标识第一次存储覆盖的字，写回要覆盖整行。

只计数据，三次填充共向上 192 字节，A 的写回向下 64 字节，未包含地址、许可和协议控制。若换成写直达并保留写分配，仍有三次填充。假设下一级支持四字节更新且写缓冲不合并，两次存储共向下送 8 字节。若写直达又取消写分配，B 和 C 才触发本级行填充，向上共 128 字节。同一行写很多次时，写回可以把反复改写的字节合到一次整行传输。实际比较要同时给出传输粒度和统计截止位置。

## 脏行移出数据阵列以后存在哪里 {#write-back-buffer}

为了让原来的路能放 C，缓存要为 A 另留一份待写回的数据。若下级端口暂未接受发送，记录仍须保留，等待接口允许重试。

| 写回记录的内容 | 用主例核对 |
| --- | --- |
| 地址与范围 | 起点 `0x1000`，长度 64 字节 |
| 命令与状态 | 表示需要转交脏数据，并携带相应的一致性或地址空间属性 |
| 数据 | 保存被替换时的整行，包括新的 9、12 和原有的 17 |
| 调度状态 | 何时可申请端口、它与其他请求的顺序，以及下级是否已经接受 |

gem5 的 `BaseCache::writebackBlk` 从缓存块重建地址，构造 `WritebackDirty` 并把整行数据复制到包中。`Cache::doWritebacks` 将适用的包放进写缓冲。`WriteQueueEntry` 保存对齐地址、大小、最早可发时间、顺序以及指向包的记录。端口接受发送后，这一级就可以按协议移交责任。这是相邻两级的交接，后面还有几层以及何时到达主存，取决于系统的布置。

**如果此时又读 A。** 新的 A 读取若绕过等待中的写回，可能从下一级取回旧的 5。gem5 的 `getNextQueueEntry` 在 MSHR 和写缓冲之间作调度，并核对两边是否有同块冲突。它准备发缺失请求时，若有同块的待发写回，可以先服务写回。也有设计会从缓冲直接提供新数据。下面的模型采用等待写回后重试的规则。

**如果写缓冲已满。** 即使 [MSHR](./microarch-01a-mshr.md#mshr-entry) 还有位置跟踪 C 的未完成请求，另外被选中的脏行也必须有地方保存数据，才能安全回收原路。实现可为在途填充预留写回位置，或延后接受新的请求，等写缓冲腾出空间。对照[缺失表项的容量限制](./microarch-01a-mshr.md#mshr-limits)，可以定位这次等待是发生在取行一侧，还是移交脏行的一侧。

## 用同一组地址核对三种写策略 {#store-model}

下面的单核 Python 模型只模拟第 0 组的两路。`ways` 按最久使用到最近使用排列，`writeback` 留一个写回槽。程序把下级接受写回和更新其数据合为一步，并假设直达更新立即到下一级。把代码保存为 `cache_write.py` 后运行 `python cache_write.py`。

```python
from dataclasses import dataclass

LINE_BYTES = 64
A, B, C = 0x1000, 0x2000, 0x3000


def word_from(data, offset):
    return int.from_bytes(data[offset:offset + 4], "little")


@dataclass
class Line:
    data: bytearray
    dirty: bool = False


class Cache:
    def __init__(self, write_allocate=True, write_through=False):
        self.write_allocate = write_allocate
        self.write_through = write_through
        self.lower = {base: bytearray(LINE_BYTES) for base in (A, B, C)}
        for offset, value in ((0x20, 5), (0x24, 11), (0x28, 17)):
            self.lower[A][offset:offset + 4] = value.to_bytes(4, "little")
        self.ways = {}
        self.writeback = None
        self.up_bytes = 0
        self.down_bytes = 0

    def get_line(self, base):
        if base in self.ways:
            line = self.ways.pop(base)
        else:
            if self.writeback is not None and self.writeback[0] == base:
                raise BlockingIOError("同块写回尚未发送")
            if len(self.ways) == 2:
                old_base = next(iter(self.ways))
                old = self.ways[old_base]
                if old.dirty:
                    if self.writeback is not None:
                        raise BlockingIOError("写回缓冲已满")
                    self.writeback = (old_base, bytes(old.data))
                del self.ways[old_base]
            line = Line(bytearray(self.lower[base]))
            self.up_bytes += LINE_BYTES
        self.ways[base] = line
        return line

    def load_word(self, address):
        offset = address % LINE_BYTES
        return word_from(self.get_line(address - offset).data, offset)

    def store_word(self, address, value):
        offset = address % LINE_BYTES
        base = address - offset
        payload = value.to_bytes(4, "little")
        if self.write_allocate or base in self.ways:
            line = self.get_line(base)
            line.data[offset:offset + 4] = payload
            line.dirty = not self.write_through
            if not self.write_through:
                return
        self.lower[base][offset:offset + 4] = payload
        self.down_bytes += len(payload)

    def send_writeback(self):
        if self.writeback is not None:
            base, data = self.writeback
            self.lower[base][:] = data
            self.down_bytes += len(data)
            self.writeback = None


def run(cache, touch_b=True):
    cache.load_word(B)
    cache.store_word(A + 0x20, 9)
    cache.store_word(A + 0x24, 12)
    if touch_b:
        cache.load_word(B)
    cache.load_word(C)
    return cache


def first_two_words(data):
    return tuple(word_from(data, offset) for offset in (0x20, 0x24))


if __name__ == "__main__":
    wb = run(Cache())
    print("A 入队后，下一级", first_two_words(wb.lower[A]))
    print("A 入队后，缓冲内", first_two_words(wb.writeback[1]))
    wb.send_writeback()
    print("写回后，下一级", first_two_words(wb.lower[A]))

    wt = run(Cache(write_through=True))
    bypass = run(Cache(write_allocate=False, write_through=True))
    print("写分配、写回", wb.up_bytes, wb.down_bytes)
    print("写分配、写直达", wt.up_bytes, wt.down_bytes)
    print("不写分配、写直达", bypass.up_bytes, bypass.down_bytes)
```

```text
A 入队后，下一级 (5, 11)
A 入队后，缓冲内 (9, 12)
写回后，下一级 (9, 12)
写分配、写回 192 64
写分配、写直达 192 8
不写分配、写直达 128 8
```

`store_word` 的切片只覆盖 `offset` 起的四个字节。`get_line` 回收脏行时，用 `bytes(old.data)` 把数据复制到写回记录，才移除旧行。可以在 `send_writeback` 之前尝试 `wb.load_word(A + 0x20)`，模型会报告同块写回在等待。发送后重试，加载从下一级取到 9。输出中的流量在这笔重试之前统计。

::: details 练习｜少读一次 B，写回次数会变吗
在程序末尾补一句 `print(run(Cache(), touch_b=False).down_bytes)`，只去掉第二次读 B。C 会替换谁？到 C 填充结束为止，向下送了多少字节？

第二次存储后，A 是最近用过的行。C 替换更久未用的 B。B 干净，写回槽为空，观察窗口内的下行数据为零。A 仍然脏，其两个字在本级是 9、12，下一级仍是 5、11。只有统计期覆盖 A 将来的移交，才要计入那次写回。
:::

[下一篇](./microarch-01f-coherence.md)把同一个值放进双核系统，说明另一核如何取得 9，以及下一级暂时保存 5 时系统怎样保证正确的读取结果。

## 参考文献

- [RISC-V 非特权规范中的加载与存储指令](https://docs.riscv.org/reference/isa/v20260120/unpriv/rv32.html#load-and-store-instructions)
- [MIT 6.004 缓存与写入策略讲义](https://ocw.mit.edu/courses/6-004-computation-structures-spring-2017/pages/c14/c14s1/)
- 图 1 来自 Djordje Kovacevi 的 gem5 内存系统文档，本站版本转为灰度、补白底与边距。[原文与原图](https://www.gem5.org/documentation/general_docs/memory_system/gem5_memory_system/#data-cache-object) · [原站 MIT 许可](/microarch/gem5-website-license.txt)
- [gem5 v24.1.0.3 对局部写入、升级及整行写的请求选择](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/cache.cc#L492-L566)
- [gem5 的写回包构造与数据复制](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/base.cc#L1710-L1756)
- [gem5 写回队列的表项字段](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/write_queue_entry.cc#L83-L109)
- [gem5 对写缓冲和 MSHR 的仲裁](https://github.com/gem5/gem5/blob/b9da2bfe1e21f6efa2730edd4e80e58c5d965458/src/mem/cache/base.cc#L859-L906)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>上一篇</small><strong>缓存未命中后如何合并请求</strong></span>](./microarch-01a-mshr.md){.article-nav-card .article-nav-prev rel=prev}

[<span class="article-nav-text"><small>下一篇</small><strong>缓存一致性：两个核怎样读到最新值</strong></span>](./microarch-01f-coherence.md){.article-nav-card .article-nav-next rel=next}

</nav>
