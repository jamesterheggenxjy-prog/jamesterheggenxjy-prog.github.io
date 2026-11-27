---
title: 两个线程改不同字段，为什么缓存行还会来回传
description: 从两个计数器的字段偏移推导伪共享，用 C 程序在不同物理核上比较同行与分行的开销，并学习如何从采样报告定位字段。
date: '2026-11-25'
tags: [微架构安全, 高速缓存, 缓存一致性, 伪共享]
book: microarch-00-intro
---

两个线程分别给自己的计数器加一。变量互不重叠，却可能让两个核轮流等同一条缓存行。原因在于，程序把数据分成字段，一致性协议按行转移写权限。本篇用两个 8 字节计数器和六次写入，把这两个粒度之间的差别算出来。

先修知识是[共享副本怎样取得写权](./microarch-01f-coherence.md#mesi-upgrade)。以下规定行长 64 字节，两个线程始终在不同物理核，同一条行的写权已经拿到后可以保留，并忽略替换与其他访问。

[[toc]]

## 两个字段落在哪些行 {#sharing-layout}

设对象从 `0x3000` 开始。布局甲的两个字段紧挨着。布局乙保持第一个字段的位置，把第二个移到下一行。块起始地址等于字节地址除以 64 的整数商乘回 64。

<div role="region" aria-label="两种布局的字段地址与缓存行" style="max-width:100%; overflow-x:auto">

| 布局 | 核 0 的字段范围 | 核 1 的字段范围 | 要申请写权的行 |
| --- | --- | --- | --- |
| 甲，字段相邻 | `0x3000` 至 `0x3007` | `0x3008` 至 `0x300f` | 两核都要取得 `0x3000` |
| 乙，字段分行 | `0x3000` 至 `0x3007` | `0x3040` 至 `0x3047` | 核 0 取得 `0x3000`，核 1 取得 `0x3040` |

</div>

行内偏移为 0 和 8，只能说明两个字段没有字节交集。要判断一致性冲突，需同时看行起点。对象的起始地址也参与计算。再设一份未强制行对齐的相邻布局，起点恰好是 `0x3038`。第一个字段在这一行末尾，第二个已经从 `0x3040` 开头，两个字段虽相邻却不共行。知道字段大小和间距之后，还需核对实际对象的对齐。

### 字段间隔和起始地址怎样一起决定行号 {#sharing-alignment}

把两个计数器看成各占 8 字节的范围，仍假设缓存行长为 64 字节。用地址除以 64 的整数商可以得到行号。下面三种布局使用相同的字段大小，基址和两个字段的间隔分别由题设给出。

| 基址 | 两个字段的起始地址 | 涉及的行 | 两字段是否同在一行 |
| --- | --- | --- | --- |
| `0x3000`，间隔 8 | `0x3000`、`0x3008` | 都在 `0x3000` 行 | 是 |
| `0x3038`，间隔 8 | `0x3038`、`0x3040` | 分别在 `0x3000`、`0x3040` 行 | 否 |
| `0x3038`，间隔 64 | `0x3038`、`0x3078` | 分别在 `0x3000`、`0x3040` 行 | 否 |

第三行给每个字段留出 64 字节间隔，即使基址本身不在行首，两次 8 字节访问也落在不同的行。程序若要确定一个字段是否跨行，则要分别检查起始地址和最后一个字节。例如从 `0x303c` 开始的 8 字节范围结束于 `0x3043`，覆盖相邻两行。这个地址只用来演算范围，实际 C 原子对象还必须满足其类型的对齐要求。

在 C 中，`offsetof` 给出字段在结构体内的偏移，实际字段地址还要加上对象基址。`_Alignas(64)` 可以要求对象从 64 字节边界开始，编译器实际采用的偏移和结构体大小仍应由 `offsetof`、`sizeof` 核对。下一个程序会打印这两项。动态分配时也需要取得满足对象对齐要求的地址，之后才能沿同样的方法判断字段落在哪些行。

## 六次写入为什么出现五次交接 {#sharing-transfers}

设初值均为 0，缓存起初均没有相关行。模型给定存储提交到缓存的顺序为 `0、1、0、1、0、1`。每步都把本核字段加一，等它完成后再开始下一步。先推演布局甲。

<div role="region" aria-label="同行交替写入时的数据与写权" style="max-width:100%; overflow-x:auto">

| 写完以后 | 同一行中的两个字段 | 写权持有者 | 这一步为何要取得写权 |
| --- | --- | --- | --- |
| 核 0 第一次 | 1、0 | 核 0 | 之前两个 L1 都缺行 |
| 核 1 第一次 | 1、1 | 核 1 | 从核 0 接过行，核 0 副本失效 |
| 核 0 第二次 | 2、1 | 核 0 | 从核 1 接过行 |
| 核 1 第二次 | 2、2 | 核 1 | 从核 0 接过行 |
| 核 0 第三次 | 3、2 | 核 0 | 从核 1 接过行 |
| 核 1 第三次 | 3、3 | 核 1 | 从核 0 接过行 |

</div>

第二行说明了整行交接的理由。核 1 只准备更新偏移 8 至 15，却必须保留核 0 刚写进偏移 0 至 7 的值 1。如果它从过期的整行副本开始自己的写入，就可能让第一个字段变回 0。一致性协议交接最新的行与写权，两个线程才都能保住各自的结果。

布局乙中，两核的第一次写分别取得各自行的写权。之后没有请求者争同一条行，剩余四次都能在各自本地完成。若布局甲仍保持共行，却把交替改成 `0、0、0、1、1、1`，则先由核 0 连写，之后只需要做一次跨核交接。

<div role="region" aria-label="三种条件下的写权事件数" style="max-width:100%; overflow-x:auto">

| 条件 | 取得写权限的次数 | 其中跨核交接 |
| --- | ---: | ---: |
| 同行，交替写 | 6 | 5 |
| 分行，交替写 | 2 | 0 |
| 同行，分批写 | 2 | 1 |

</div>

这就是**伪共享**。两个线程使用不同的数据，却因为落到同一条行而分享一致性资源。表中计的是写权限事件，单次交接实际会有哪些消息、花多少时间，还取决于协议和拓扑。

## 用 C 程序核对布局和写权限次数 {#sharing-code}

下面的 C11 程序用 `_Alignas` 控制对象和字段的最低对齐，再用 `offsetof` 取得编译器实际采用的字段偏移。三次 `simulate` 读取相同长度的写序列。程序没有创建线程，存储顺序由数组直接给出，这样可以重放表格中的六步。保存为 `sharing_layout.c` 后，可用 GCC 编译。

```bash
gcc -std=c11 -O2 -Wall -Wextra sharing_layout.c -o sharing_layout
```

```c
#include <stddef.h>
#include <stdint.h>
#include <stdio.h>

enum { LINE_BYTES = 64 };

struct Adjacent {
    _Alignas(LINE_BYTES) uint64_t first;
    uint64_t second;
};

struct Separate {
    _Alignas(LINE_BYTES) uint64_t first;
    _Alignas(LINE_BYTES) uint64_t second;
};

struct Record {
    uint64_t counter;
    uint64_t payload[2];
};

struct Result {
    unsigned acquisitions;
    unsigned handoffs;
};

_Static_assert(sizeof(uint64_t) == 8, "expected eight-byte counters");
_Static_assert(offsetof(struct Adjacent, second) == 8, "adjacent offset");
_Static_assert(offsetof(struct Separate, second) == 64, "separate offset");
_Static_assert(sizeof(struct Record) == 24, "record stride");

static struct Result simulate(size_t second_offset,
                              const int *order, size_t count)
{
    const size_t offsets[] = {0, second_offset};
    int owner[2] = {-1, -1};
    struct Result result = {0, 0};
    for (size_t i = 0; i < count; ++i) {
        int core = order[i];
        size_t line = offsets[core] / LINE_BYTES;
        if (owner[line] != core) {
            ++result.acquisitions;
            if (owner[line] != -1) {
                ++result.handoffs;
            }
            owner[line] = core;
        }
    }
    return result;
}

static void print_case(const char *name, struct Result result)
{
    printf("%s: acquisitions=%u handoffs=%u\n",
           name, result.acquisitions, result.handoffs);
}

int main(void)
{
    const int alternating[] = {0, 1, 0, 1, 0, 1};
    const int grouped[] = {0, 0, 0, 1, 1, 1};
    const size_t same = offsetof(struct Adjacent, second);
    const size_t split = offsetof(struct Separate, second);

    printf("offsets: %zu %zu\n", same, split);
    printf("sizes: %zu %zu\n", sizeof(struct Adjacent), sizeof(struct Separate));
    print_case("same/alternating", simulate(same, alternating, 6));
    print_case("split/alternating", simulate(split, alternating, 6));
    print_case("same/grouped", simulate(same, grouped, 6));
    const size_t boundaries[] = {4, 8};
    for (size_t i = 0; i < sizeof boundaries / sizeof boundaries[0]; ++i) {
        size_t right = boundaries[i] * sizeof(struct Record);
        size_t left = right - sizeof(struct Record);
        printf("split at %zu: offsets %zu %zu, line starts %zu %zu\n",
               boundaries[i], left, right,
               left / LINE_BYTES * LINE_BYTES,
               right / LINE_BYTES * LINE_BYTES);
    }
    return 0;
}
```

```text
offsets: 8 64
sizes: 64 128
same/alternating: acquisitions=6 handoffs=5
split/alternating: acquisitions=2 handoffs=0
same/grouped: acquisitions=2 handoffs=1
split at 4: offsets 72 96, line starts 64 64
split at 8: offsets 168 192, line starts 128 192
```

上面是 GCC 15.2.0 的 x86-64 Windows 输出。`owner` 的初值 `-1` 表示行还没有可写的私有副本。第一个访问这条行的核计一次取得，从另一核接过时再计一次交接。地址布局产生的空间代价也在输出中。相邻对象本身对齐到 64 字节，两个字段只用 16 字节，结构总占 64 字节。分行对象的第二个字段从第 64 字节开始，结构总占 128 字节。若系统行长与例题不同，应重新计算布局。

最后两行复算[数组的分工边界](#sharing-array-boundary)。代码只用结构体大小推导起点，题设的 64 字节基址对齐由数组对象负责。

## 只读字段也会被迫重新加载 {#sharing-read-write}

Linux 内核文档用同一个对象里的引用计数和名称说明另一类伪共享。对象初始化以后，名称保持不变。一颗核反复更新引用计数，另一颗核读取名称。沿用上一节的 64 字节行和基址 `0x3000`，设 `refs` 置于 `0x3000–0x3003`，`name` 置于 `0x3010–0x301f`。下表使用简化的写回式 MESI，初态两核都有干净的共享副本，期间没有替换和预取。假定每一步完成后才开始下一步，读者每次读取名称的一个字节。

| 操作 | 完成后核 0 的状态 | 完成后核 1 的状态 | 请求与结果 |
| --- | --- | --- | --- |
| 核 0 更新 `refs` | M | I | 核 0 申请升级，核 1 的副本失效 |
| 核 1 读取 `name[0]` | S | S | 核 1 缺行，一致性系统取得核 0 持有的最新行，两核共享 |
| 核 1 读取 `name[1]` | S | S | 仍有共享副本，本地命中 |
| 核 0 再更新 `refs` | M | I | 核 0 再次申请升级，核 1 的副本失效 |
| 核 1 读取 `name[0]` | S | S | 再次缺行，取得当前版本的整行 |

核 0 两次写入都只改了引用计数的四个字节。核 1 两次重取仍要经过整行的取数路径，因为缓存标记和一致性状态都属于 `0x3000` 这一行，名称字段没有独立的有效位。核 1 从未申请写权限。本例的开销来自两次升级和两次读缺失，可以与[两个写入者交替取得写权](#sharing-transfers)的例子对照。每一步如何获得许可、由哪一级交付数据，可回看[共享副本的升级](./microarch-01f-coherence.md#mesi-upgrade)和[脏行的读取](./microarch-01f-coherence.md#mesi-owner)。

现在只改变布局，把名称移到 `0x3040–0x304f`，引用计数仍在 `0x3000–0x3003`。假设核 1 已读入名称所在的 `0x3040` 行，核 0 更新引用计数时，申请写权限的对象是 `0x3000` 行。该事务不需要使核 1 的 `0x3040` 副本失效。后续核 1 再读名称时，只要名称行仍有效且没有被替换，就可以命中本地副本。若还有第三个字段把两核重新联系到同一行，或核 1 也读取引用计数，仍要单独追踪对应行的权限变化。

改变访问次序也会改变交接次数。保留原来的共行布局，让核 0 连续更新两次 `refs`，再由核 1 连续读取两个名称字节。初态仍为两核共享，第一次写使核 1 的行失效，第二次写在核 0 已持有 M 状态时完成；核 1 的第一次读取取得新行，第二次读取本地命中。此时题设下只需要一次升级和一次读缺失。字段的位置决定哪些访问可能互相影响，交错顺序决定这次执行实际发生多少交接。

假如把 `name` 移到从 `0x3040` 开始的下一条行，两边各有一条缓存行。保持相同的访问次序，再设初态两核都缓存了原计数行、核 1 也已缓存名称行。核 0 的第一次升级仍会让核 1 丢掉计数行的旧副本，但后面的三次名称读取都可以命中。由于这些读没有请求计数行，核 0 做第二次更新时仍有该行的写权限。

::: details 练习｜布局不动，只调整读取时间

把五次操作排成「核 0 连续写两次 `refs`，然后核 1 连续读三次 `name`」，并沿用最初两核都有共享副本的设定。要做几次升级？有几次名称读缺失？

第一次写将核 0 的状态从 S 升为 M，让核 1 的副本失效。第二次写在 M 状态下完成。核 1 第一次读取需要重取整行，随后两次本地命中。因此共有一次升级和一次读缺失。同一字段布局的开销随读写交错方式变化。

:::

### 增加只读核以后，一次写要使几份副本失效 {#sharing-read-fanout}

仍用同行的引用计数与名称，增加核 2。设三个核最初都持有这条行的 S 副本，核 0 更新一次引用计数，核 1 和核 2 随后依次各读一次名称。沿用写回式 MESI 的题设，期间没有换出，也没有别的访问。

| 操作 | 核 0 | 核 1 | 核 2 | 这一拍之后要记住的结果 |
| --- | --- | --- | --- | --- |
| 初态 | S | S | S | 三份内容相同的共享副本 |
| 核 0 写计数 | M | I | I | 一次写权限升级使两份远端副本失效 |
| 核 1 读名称 | S | S | I | 核 1 首次重取，最新行由一致性系统交付 |
| 核 2 读名称 | S | S | S | 核 2 也需要取数，取数来源由缓存组织决定 |

与两个核的例子相比，核 0 仍只升级一次，但失效的副本从一份增加到两份，名称读取中的缺失也从一次增加到两次。一致性协议可能广播、合并确认或使用不同的数据返回路径，因此表里数的是副本和请求结果，不能直接拿来当作互连消息数。相关的确认过程见[失效请求与确认](./microarch-01f-coherence.md#mesi-upgrade)。

如果核 2 在核 0 下次更新前一直不读名称，核 2 就保持 I。下一次写只需要使核 1 当前持有的共享副本失效。要判断下一轮会牵涉哪些核，应从这一轮结束时仍有效的副本出发。

## 在两个物理核上比较两种布局 {#sharing-benchmark}

六步例题规定了核之间严格交替。实际循环中，一个核可能保留写权并连写若干次，写权交接的顺序由当时的请求决定。下面用两个线程比较原来的偏移 0、8 与偏移 0、64。两组使用同一个 64 字节对齐的数组、同一段循环及同一对物理核。各线程各更新自己的计数器一千万次。

程序面向使用单处理器组的 Windows x86-64 系统。系统拓扑中，一条 `RelationProcessorCore` 记录表示一个物理核，掩码中的每一位置表示一个属于该核的逻辑 CPU。程序从两条记录中各取一个进程可用的逻辑 CPU，并检查这两个 CPU 的 L1 数据缓存行均为 64 字节。线程等待同一个启动事件。亲和设置完成后，主线程才放行。

数组元素声明为 `volatile uint64_t`，所以这个 GCC 示例会在每次循环中实际读取并写回元素。两个线程访问的是不同的数组元素，主线程只在它们结束后读取计数器。线程启动和完成由系统事件及等待接口协调。循环本身只需要普通的读、加、写，不需要处理器执行带锁的读改写指令。

保存下列代码为 `sharing_threads.c`。在 Windows PowerShell 中使用 MinGW-w64 的 GCC 和 GNU objdump。

~~~powershell
gcc -std=c11 -O2 -g -Wall -Wextra sharing_threads.c -o sharing_threads.exe
.\sharing_threads.exe
objdump -d -Mintel --disassemble=update_counter .\sharing_threads.exe
~~~

~~~c
#define WIN32_LEAN_AND_MEAN
#include <windows.h>
#include <inttypes.h>
#include <stdio.h>
#include <stdlib.h>

enum { LINE_BYTES = 64, UPDATES = 10000000 };
static _Alignas(LINE_BYTES) volatile uint64_t counters[16];

struct Worker {
    HANDLE start_event;
    volatile uint64_t *counter;
    LARGE_INTEGER start;
    LARGE_INTEGER end;
};

static void fail(const char *message)
{
    fprintf(stderr, "%s\n", message);
    exit(EXIT_FAILURE);
}

static void choose_cpus(DWORD_PTR masks[2])
{
    DWORD bytes = 0;
    if (GetActiveProcessorGroupCount() != 1)
        fail("This example requires a single processor group");
    if (GetLogicalProcessorInformation(NULL, &bytes)
        || GetLastError() != ERROR_INSUFFICIENT_BUFFER)
        fail("Cannot size the processor topology");

    SYSTEM_LOGICAL_PROCESSOR_INFORMATION *topology = malloc(bytes);
    DWORD_PTR allowed, system;
    if (!topology || !GetLogicalProcessorInformation(topology, &bytes)
        || !GetProcessAffinityMask(GetCurrentProcess(), &allowed, &system))
        fail("Cannot read the processor topology");

    size_t count = bytes / sizeof *topology;
    unsigned found = 0;
    for (size_t i = 0; i < count && found < 2; ++i) {
        DWORD_PTR eligible = topology[i].ProcessorMask & allowed;
        if (topology[i].Relationship == RelationProcessorCore && eligible)
            masks[found++] = eligible & (~eligible + 1);
    }
    if (found != 2)
        fail("Two allowed physical cores are required");

    DWORD_PTR checked = 0;
    for (size_t i = 0; i < count; ++i) {
        if (topology[i].Relationship == RelationCache
            && topology[i].Cache.Level == 1
            && (topology[i].Cache.Type == CacheData ||
                topology[i].Cache.Type == CacheUnified)
            && topology[i].Cache.LineSize == LINE_BYTES)
            checked |= topology[i].ProcessorMask;
    }
    DWORD_PTR selected = masks[0] | masks[1];
    free(topology);
    if ((checked & selected) != selected)
        fail("Both selected CPUs must report 64-byte L1 data lines");
}

static DWORD WINAPI update_counter(void *argument)
{
    struct Worker *worker = argument;
    if (WaitForSingleObject(worker->start_event, INFINITE) != WAIT_OBJECT_0
        || !QueryPerformanceCounter(&worker->start))
        return 1;

    volatile uint64_t *value = worker->counter;
    for (unsigned i = 0; i < UPDATES; ++i)
        *value += 1;

    return QueryPerformanceCounter(&worker->end) ? 0 : 1;
}

static void run_case(unsigned second, const DWORD_PTR masks[2], double ms_per_tick)
{
    HANDLE gate = CreateEventW(NULL, TRUE, FALSE, NULL);
    if (!gate)
        fail("Cannot create the start event");

    counters[0] = counters[second] = 0;
    struct Worker workers[2] = {
        {.start_event = gate, .counter = &counters[0]},
        {.start_event = gate, .counter = &counters[second]}
    };
    HANDLE threads[2];
    for (unsigned i = 0; i < 2; ++i) {
        threads[i] = CreateThread(NULL, 0, update_counter, &workers[i], 0, NULL);
        if (!threads[i] || !SetThreadAffinityMask(threads[i], masks[i]))
            fail("Cannot start and pin a worker");
    }
    if (!SetEvent(gate) ||
        WaitForMultipleObjects(2, threads, TRUE, INFINITE) != WAIT_OBJECT_0)
        fail("Cannot release or join the workers");
    for (unsigned i = 0; i < 2; ++i) {
        DWORD code;
        if (!GetExitCodeThread(threads[i], &code) || code != 0)
            fail("Worker timing failed");
        CloseHandle(threads[i]);
    }
    CloseHandle(gate);
    if (counters[0] != UPDATES || counters[second] != UPDATES)
        fail("Unexpected counter value");

    LONGLONG s0 = workers[0].start.QuadPart, s1 = workers[1].start.QuadPart;
    LONGLONG e0 = workers[0].end.QuadPart, e1 = workers[1].end.QuadPart;
    LONGLONG first_start = s0 < s1 ? s0 : s1;
    LONGLONG last_start = s0 < s1 ? s1 : s0;
    LONGLONG first_end = e0 < e1 ? e0 : e1;
    LONGLONG last_end = e0 < e1 ? e1 : e0;
    LONGLONG overlap = first_end > last_start ? first_end - last_start : 0;

    printf("offset=%2zu window=%7.3f ms overlap=%7.3f ms "
           "counters=%" PRIu64 ",%" PRIu64 "\n",
           second * sizeof counters[0], (last_end - first_start) * ms_per_tick,
           overlap * ms_per_tick, counters[0], counters[second]);
}

int main(void)
{
    DWORD_PTR masks[2];
    LARGE_INTEGER frequency;
    choose_cpus(masks);
    if (!QueryPerformanceFrequency(&frequency))
        fail("Cannot read the performance counter frequency");
    printf("cpu_masks=0x%" PRIxPTR ",0x%" PRIxPTR "\n",
           (uintptr_t)masks[0], (uintptr_t)masks[1]);

    const unsigned second_word[] = {1, 8, 8, 1};
    for (size_t i = 0; i < sizeof second_word / sizeof second_word[0]; ++i)
        run_case(second_word[i], masks, 1000.0 / frequency.QuadPart);
    return 0;
}
~~~

`second_word` 用元素下标指定第二个计数器。下标 1 和 8 对应字节偏移 8 和 64，先跑同行、分行，再调换顺序，避免每次都让同一种布局在第一轮运行。每轮结束后检查两个计数器都到了一千万，确认比较的是同样的更新次数。

下面是 GCC 15.2.0 以上述命令编译得到的 x86-64 循环体，省略了机器码，把跳转目标记作 `.loop`。进入循环时，`rcx` 已保存当前线程的计数器地址，`edx` 为 `0x989680`，即 10,000,000。

~~~asm
.loop:
    mov rax, QWORD PTR [rcx]
    add rax, 1
    sub edx, 1
    mov QWORD PTR [rcx], rax
    jne .loop
~~~

两条 `mov` 访问的是同一个 8 字节对象。`add` 增加读出的值，`sub` 更新剩余次数并设置零标志。写回的 `mov` 不改变这一标志，因此 `jne` 依据 `edx` 是否减至零决定是否继续循环。两种布局只改变第二个核的 `rcx` 指向哪里。

每个线程在循环两端读取 `QueryPerformanceCounter`。`window` 从最早进入循环的线程开始计，到最后一个线程完成。`overlap` 是两个循环时间区间的交集。如果其中一轮几乎没有重叠，两个核就很少同时争用这条行，应先检查调度再比较耗时。重叠仍可能包含抢占，因为这里用的是墙钟。

以下是 Core i5-13500H、Windows 11 x86-64 上的一次输出。`0x1` 和 `0x4` 是逻辑 CPU 0 和 2 的掩码，程序在两条不同的物理核记录中选到了它们。

~~~text
cpu_masks=0x1,0x4
offset= 8 window=  3.305 ms overlap=  2.706 ms counters=10000000,10000000
offset=64 window=  2.208 ms overlap=  2.139 ms counters=10000000,10000000
offset=64 window=  2.172 ms overlap=  2.135 ms counters=10000000,10000000
offset= 8 window=  2.922 ms overlap=  2.662 ms counters=10000000,10000000
~~~

同一环境共运行四次，每次包含四轮。其中有一轮同行的 `window` 是 4.800 ms，`overlap` 为 0。这一轮没有同时更新，不宜算入并发对照。其余有重叠的同行轮次为 2.557 至 3.629 ms，分行轮次为 2.172 至 2.263 ms。这组读数给出了该机器上两种布局的耗时差异。写权限实际交接了多少次、哪次等待最长，单从时间无法确定。

::: details 练习｜改动第二个计数器的位置
保持数组起始地址按 64 字节对齐。若把两个下标从 1 和 8 改成 7 和 15，哪一轮中的两个计数器会争同一条行？先算第二个计数器的起止地址，再用程序核对其中一轮的输出。

下标 7 占偏移 56 至 63，与第一个计数器同在第 0 行。下标 15 占偏移 120 至 127，在第 1 行。只要线程仍在不同核且同时更新，第一种布局就仍然具备伪共享的两个条件。
:::

## 怎样从采样报告找到字段 {#sharing-measurement}

总耗时反映这一轮程序完成得快还是慢。要定位哪些字段造成了跨核取数，还需要访存采样。在支持相关事件的 Linux 机器上，`perf c2c` 按数据缓存行聚合样本。报告中一个常见指标是 HITM，表示采样到的加载数据来自处于修改状态的缓存副本。它只覆盖特定平台与事件采到的加载，不能逐笔计数前面例题的写权交接。

Linux 内核文档的报告示例中，基址为 `0xff1100088366d880` 的一行在明细表里多次出现。两条明细的行内偏移分别是 `0x8` 和 `0x10`，关联到 `memcontrol.h:752` 和 `memcontrol.h:736`。把偏移加回基址，得到同一行内的两个数据地址 `0xff1100088366d888` 和 `0xff1100088366d890`。这是访问数据的位置。报告还单独给出引起访问的指令地址和源码行，用来确定加载或存储来自哪段代码。最后要用结构布局确认两个数据地址分别属于哪个字段，再用源码确认哪些访问会写入。这样才能判断是否存在两个独立字段争同一行。若干记录指向同一字段时，还可能是在直接共享同一个对象。

对本节的教学数组，字段对照已经明确。若分开字段后，重叠时间充足的多轮运行都变快，并且相关缓存行的跨核取数样本也减少，就能把布局变化与一致性开销联系起来。[计时与计数器一课](./microarch-04a-measurement.md)继续讲统计区间和事件范围。

::: details 练习｜样本少了一半，跨核取数的比例也下降了吗
三个测量窗口执行相同的更新次数，两个线程固定在同一对物理核心。移动字段后，按字段的新地址找到对应的缓存行。设采样使用同一种加载事件，表中「全部样本」只包括这两个字段的加载，且都有可用的数据来源分类。所有数字均为教学题设。

| 窗口 | 两个字段的全部加载样本 | 其中 HITM 样本 | 墙钟耗时 |
| --- | ---: | ---: | ---: |
| 字段共行 | 200 | 80 | 12 ms |
| 字段分行，第一次 | 100 | 40 | 8 ms |
| 字段分行，第二次 | 100 | 10 | 8 ms |

共行时的 HITM 样本比例是 `80 / 200 = 40%`。第一次分行虽然少了 40 个 HITM 样本，比例还是 `40 / 100 = 40%`，这组读数没有显示同类样本中的 HITM 比例下降。第二次分行的比例是 `10 / 100 = 10%`，这才与相关加载更少从修改态副本取得数据的解释相符。两个分行窗口的墙钟耗时相同，单靠时间无法看出这项差别。

实测应确认两个字段都纳入统计，并核对事件配置、采样覆盖和线程实际重叠时间。样本比例描述的是被采到且可分类的加载。由它推算整段执行的写权限交接次数还需要其他证据。
:::

分行也有成本。若字段主要由同一核一起更新，挨着放可以利用空间局部性。每个小字段占一整行，可能使对象数组和工作集增大。是否改布局，要以访问的核、写入频率与实际占用为依据。

### 只移动数组的分工边界 {#sharing-array-boundary}

再看一种可以保持结构布局的情形。一个按 64 字节对齐的数组有 16 条记录，每条占 24 字节。工作线程只更新每条记录开头的 8 字节计数器。CPU 0 处理边界左侧的所有记录，CPU 1 处理右侧的所有记录。

若从下标 4 分工，CPU 0 最后更新下标 3。其计数器占偏移 72 至 79。CPU 1 第一个更新下标 4，偏移为 96 至 103。两个字段属于不同对象，但都在偏移 64 至 127 的缓存行内。改从下标 8 分工，左侧最后写偏移 168 至 175，右侧从偏移 192 开始写。192 恰好是 64 的倍数，两个线程访问的计数器不再跨过同一条行。

| 右侧的第一个下标 | 两侧离边界最近的计数器起点 | 它们所在的行起点 | 左右两侧记录数 |
| ---: | --- | --- | --- |
| 4 | 72、96 | 64、64 | 4、12 |
| 8 | 168、192 | 128、192 | 8、8 |

在这个对齐条件下，如果希望整条记录的分界也落在缓存行边界，就解 `24 × 下标` 是否为 64 的倍数。24 和 64 的最大公约数为 8，所以每隔 8 条记录就有一个这样的分界。改变分工位置不会增加对象大小，数组仍占 384 字节。实际分工还要平衡两侧工作量，且数组基址必须满足用来推导边界的对齐条件。

### 四个线程怎样分工，才不会共写一行 {#sharing-four-workers}

仍用 16 条记录，每条 24 字节，基址按 64 字节对齐。四个线程各更新四条记录，可以连续切成四块，也可以按下标轮流分给四个线程。先列出每条缓存行里实际被写的计数器，再检查同一行内是否有不同的写入者。每个计数器的首地址为 `基址 + 24 × 下标`，八个字节都在它的起始行内，因为这里的首地址按 8 字节对齐，行长是 8 的倍数。

<div role="region" tabindex="0" aria-label="连续分块与轮流分工的缓存行归属" style="max-width:100%; overflow-x:auto">

| 相对行起点 | 这一行中被更新的记录下标 | 连续分块的写入者 | 按下标轮流分工的写入者 |
| ---: | --- | --- | --- |
| 0 | 0、1、2 | 0 | 0、1、2 |
| 64 | 3、4、5 | 0、1 | 0、1、3 |
| 128 | 6、7 | 1 | 2、3 |
| 192 | 8、9、10 | 2 | 0、1、2 |
| 256 | 11、12、13 | 2、3 | 0、1、3 |
| 320 | 14、15 | 3 | 2、3 |

</div>

连续分块时，写入者是下标整除 4，只有两行有多个写入者。轮流分工时，写入者是下标对 4 取余，六行都有多个写入者。表中数的是**可能发生写权限交接的行**。要算实际发生多少次交接，还要知道各核写入的交错顺序，以及同一个核能否保住写权限连续更新。

如果只更新计数器，可以把连续分界改到下标 3、8、11。以分界 3 为例，前一个计数器从偏移 48 开始，后一个从偏移 72 开始，中间已经跨过 64 字节边界。记录 2 自身跨行，但它跨过去的部分在题设中没人写。因此这三处分界会让每行只有一个写入者，各线程分别处理 3、5、3、5 条记录。若工作还会改记录尾部，就要按新写入范围重新计算。

::: details 练习｜保持工作量相同，也让整条记录分行

数组增至 32 条，仍按 64 字节对齐。四个线程各处理连续的 8 条记录。就算每条记录的 24 字节都会被写，内部边界还会有共写的行吗？

三个内部边界的偏移是 192、384、576，都是 64 的倍数。每个线程的写入范围由完整的缓存行组成，相邻线程没有共同写入的行。若分工仍为每线程八条，但数组基址向后错开 8 字节，每个内部边界都会落在行内，就要重新检查两侧的写入范围。
:::

## 参考文献

- [Linux 内核文档对伪共享及布局成本的说明](https://docs.kernel.org/kernel-hacking/false-sharing.html)
- [Linux 内核文档中的伪共享定位方法](https://docs.kernel.org/kernel-hacking/false-sharing.html#how-to-detect-and-analyze-false-sharing)
- [Linux perf 的 `c2c` 手册与报告字段](https://kernel.googlesource.com/pub/scm/linux/kernel/git/stable/linux-stable/+/master/tools/perf/Documentation/perf-c2c.txt)
- [Windows 处理器拓扑记录](https://learn.microsoft.com/en-us/windows/win32/api/sysinfoapi/nf-sysinfoapi-getlogicalprocessorinformation) · [线程亲和掩码](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-setthreadaffinitymask) · [高精度时间戳](https://learn.microsoft.com/en-us/windows/win32/sysinfo/acquiring-high-resolution-time-stamps)
- [gem5 Bootcamp 2024 课程中的共享、交错访问和伪共享例子](https://github.com/gem5bootcamp/2024/blob/3ea9b7e3598cd15e8c0ea688fa850249eab0fad5/slides/02-Using-gem5/05-cache-hierarchies.md)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>上一篇</small><strong>缓存一致性：两个核怎样读到最新值</strong></span>](./microarch-01f-coherence.md){.article-nav-card .article-nav-prev rel=prev}

[<span class="article-nav-text"><small>下一篇</small><strong>硬件预取器怎样工作</strong></span>](./microarch-02-prefetch.md){.article-nav-card .article-nav-next rel=next}

</nav>
