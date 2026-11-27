---
title: 用计时和计数器分析访存
description: 用相同的求和循环比较访问顺序，分别测程序整体和求和区间，再对照 PMU 事件与复用时间。
date: '2026-11-25'
tags: [微架构安全, 测量方法, PMU, 缓存, TLB]
book: microarch-00-intro
---

同一个数组求和程序，顺序版本比乱序版本快。两者读过同样的元素并算出了同样的和。差值可能来自缓存命中，也可能来自预取、页表遍历或发出的请求能否重叠。本文追问一件具体的事：**一组时间和计数，究竟可以把差异归因到哪一步？**

贯穿全文的是同一个只处理公开数据的求和程序。给它两份索引，让它按不同顺序读同一批数据。代码可以直接运行，后面的数值表是用来复算的题设。先修知识是[缓存时序](./microarch-01a-mshr.md)、[硬件预取](./microarch-02-prefetch.md#prefetch-feedback)和 [TLB](./microarch-02a-translation.md#sv39-tlb)。

[[toc]]

## 固定求和循环，只换索引顺序 {#pmu-workload}

直接比较 `sum += data[i]` 和 `sum += data[order[i]]`，会同时改变访问顺序与循环指令。前者还能更容易被编译器转换成向量加载。这里把两种输入都交给 `sum_pass`。它每轮先读 `order[i]`，再用读到的索引访问 `data`。两个版本都顺序扫描索引数组。变化的是第二次加载的地址序列。

将第一段保存为 `sum_kernel.c`。它是两个版本共同的求和循环。

```c
#include <stddef.h>
#include <stdint.h>

uint64_t sum_pass(const uint32_t *data, const uint32_t *order, size_t count)
{
    uint64_t sum = 0;
    for (size_t i = 0; i < count; ++i) {
        sum += data[order[i]];
    }
    return sum;
}
```

将下一段保存为 `main.c`。两种模式都准备同一份数据、递增索引和打乱后的索引。打乱使用固定种子，从末尾开始逐项交换，因此最终每个下标仍恰好出现一次。

```c
#ifndef _WIN32
#define _POSIX_C_SOURCE 200809L
#endif

#include <inttypes.h>
#include <stddef.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#ifdef _WIN32
#include <windows.h>
#else
#include <time.h>
#endif

enum { COUNT = 1 << 20, PASSES = 24 };
static uint32_t data[COUNT];
static uint32_t ordered[COUNT];
static uint32_t shuffled[COUNT];

uint64_t sum_pass(const uint32_t *data, const uint32_t *order, size_t count);

static uint32_t next_number(uint32_t *state)
{
    uint32_t x = *state;
    x ^= x << 13;
    x ^= x >> 17;
    x ^= x << 5;
    *state = x;
    return x;
}

static double seconds_now(void)
{
#ifdef _WIN32
    static LARGE_INTEGER frequency;
    LARGE_INTEGER counter;
    if ((frequency.QuadPart == 0 && !QueryPerformanceFrequency(&frequency)) ||
        !QueryPerformanceCounter(&counter)) {
        fputs("Performance counter unavailable\n", stderr);
        exit(EXIT_FAILURE);
    }
    return (double)counter.QuadPart / (double)frequency.QuadPart;
#else
    struct timespec now;
    if (clock_gettime(CLOCK_MONOTONIC, &now) != 0) {
        perror("clock_gettime");
        exit(EXIT_FAILURE);
    }
    return (double)now.tv_sec + (double)now.tv_nsec / 1000000000.0;
#endif
}

#ifdef WINDOW_PMU
void pmu_prepare(void);
void pmu_start(void);
void pmu_stop(void);
void pmu_report(void);
#endif

int main(int argc, char **argv)
{
    if (argc != 2 ||
        (strcmp(argv[1], "seq") != 0 && strcmp(argv[1], "shuffle") != 0)) {
        fprintf(stderr, "usage: %s seq|shuffle\n", argv[0]);
        return 2;
    }

    for (uint32_t i = 0; i < COUNT; ++i) {
        data[i] = i;
        ordered[i] = i;
        shuffled[i] = i;
    }
    uint32_t seed = 0x12345678u;
    for (uint32_t i = COUNT - 1; i > 0; --i) {
        uint32_t j = next_number(&seed) % (i + 1);
        uint32_t temporary = shuffled[i];
        shuffled[i] = shuffled[j];
        shuffled[j] = temporary;
    }

    const uint32_t *order = strcmp(argv[1], "seq") == 0 ? ordered : shuffled;
#ifdef WINDOW_PMU
    pmu_prepare();
#endif
    uint64_t sum = 0;
    double begin = seconds_now();
#ifdef WINDOW_PMU
    pmu_start();
#endif
    for (unsigned pass = 0; pass < PASSES; ++pass) {
        sum += sum_pass(data, order, COUNT);
    }
#ifdef WINDOW_PMU
    pmu_stop();
#endif
    double elapsed_ms = (seconds_now() - begin) * 1000.0;
    printf("sum=%" PRIu64 "\n", sum);
    printf("sum_window_ms=%.3f per_pass_ms=%.3f\n", elapsed_ms, elapsed_ms / PASSES);
#ifdef WINDOW_PMU
    pmu_report();
#endif
    return 0;
}
```

用 GCC 编译。下面先构建只做求和和墙钟计时的版本，关闭链接时优化。`-fno-tree-vectorize` 使教学例子的主循环保持在便于跟读的标量路径。两种运行的首行都是 `sum=13194126950400`，可以用 $24 \times (1048576-1)\times 1048576/2$ 复核。

```bash
gcc -O2 -fno-tree-vectorize -fno-lto main.c sum_kernel.c -o array_sum
./array_sum seq
./array_sum shuffle
objdump -d --disassemble=sum_pass ./array_sum
```

下面是使用相同编译选项，在 GCC 15.2.0 的 Windows x86-64 目标上得到的循环体。省去地址、机器码和循环前后的指令。Linux 目标的参数寄存器可能不同，可以用同一条 `objdump` 命令查看自己的结果。

```asm
mov    (%rdx),%r8d
add    $0x4,%rdx
mov    (%rcx,%r8,4),%r8d
add    %r8,%rax
cmp    %rdx,%r9
jne    sum_pass+0x20
```

`%rdx` 指向当前索引，第一条 `mov` 把 32 位下标读入 `%r8d`。第二条指令将索引指针向前移动四字节。第三条用 `%rcx` 中的数据基址加下标乘四，读出元素。写 `%r8d` 会将 `%r8` 的高 32 位清零，下一条 `add` 据此把无符号元素加进 64 位总和。`%r9` 保存索引数组末尾地址，`cmp` 和 `jne` 控制这一轮扫描何时结束。

一次迭代里的第二条加载要等下标，多次迭代之间仍可能有重叠。按顺序访问时，后续数据容易落入邻近缓存行。打乱以后，某行内不同元素的使用被分散到更远的迭代。两种输入同样还要消耗读取索引数组的带宽，这部分计入程序的整体代价。

## 先确定计时的对象和区间 {#pmu-window}

整个求和循环的用时是一批工作完成的耗时。循环中如果有多个互相独立的加载，多个请求可以重叠。把总时间除以加载数，得到的是批量工作中的平均单位成本，里面包含重叠执行的影响。例如每次未命中响应需 90 个周期，若三个请求可以一起发出并一起返回，这三个响应阶段合计约等 90 个周期，而单条的响应延迟仍可能是 90 个周期。

还需要明确用的时间基准。墙钟时间会包含实验窗口中的调度等待。核心周期、参考周期和时钟读数有不同的计数规则。变频、线程迁移或抢占可能使它们的变化不一致。报告“快 20%”时，一并写清所用时间基准、统计区间、每次运行处理多少元素，以及所比较的是单次访问还是整段工作的完成时间。

在 Linux 上，可以先从两个普遍可用的硬件事件开始。`:u` 选择用户态计数，花括号要求将周期与退休指令放入同一组。以下命令测量所启动的整个程序，准备、求和和输出都在这个计数窗口内。

```bash
perf stat -e '{cycles:u,instructions:u}' -- ./array_sum seq
perf stat -e '{cycles:u,instructions:u}' -- ./array_sum shuffle
```

初始化做了相同的工作，但它建立了内存映射和缓存初态，且开销仍包含在两次运行中。若问题转成「已经开始求和后，每一轮稳定用多久」，应另选只覆盖重复循环的窗口，或按轮次检查前几轮与后续轮次。即使运行 24 轮，也不能从总量倒推出初始化和求和各占多少。某些虚拟机或系统配置会让硬件事件不可用，读结果时应先确认工具报告的事件确有计数。

## 从事件选择到 PMU 计数 {#pmu-path}

**PMU** 是处理器的性能监控单元。以 Intel 的架构性能监控为例，核心内部产生退休、周期和访存等事件；硬件按配置选择哪些事件能使计数器累加。Linux 把不同机器上的事件描述组织成可供 `perf` 使用的接口。

<div role="region" aria-label="PMU 计数路径" style="max-width:100%; overflow-x:auto">

| 位置 | 本次测量中要确定什么 |
| --- | --- |
| Linux 中的性能事件 | 事件的类型、配置、所附着的任务或 CPU，以及是否排除内核态等范围 |
| 固定功能计数器 | 计数项由硬件规定，例如退休指令。内核配置它的启用与计数范围 |
| 可编程计数器与事件选择寄存器 | 按事件编号、单位掩码等字段选择要计数的活动。某些事件只能使用特定计数槽 |
| 结果读取 | 原始累计值、事件启用时长，以及真正占用计数器的时长 |

</div>

举例来说，Intel Skylake Server 的 `INST_RETIRED.ANY` 可使用固定计数器 0。该型号的 `MEM_LOAD_RETIRED.L1_MISS` 则有事件编号 `D1H` 和单位掩码 `08H`，计的是含至少一个 L1 未命中加载微操作的已退休加载指令。它的单位是指令，需要和缓存行填充数分开看。`DTLB_LOAD_MISSES.WALK_COMPLETED` 计数由需求数据加载引起、已经结束的页表遍历。该型号还把只命中二级 TLB 的情况定义为另一个事件。分析其他处理器时，按其事件表重新核对。

从要回答的问题反推该读哪一级接口。

<div role="region" aria-label="分析问题与性能事件" style="max-width:100%; overflow-x:auto">

| 想解释的对象 | 可关注的事件例子 | 仍需知道的范围 |
| --- | --- | --- |
| 程序做了多少架构工作 | 退休指令数 | 是哪个线程、核与特权范围，事件是否按 ISA 指令统计 |
| 某一级缓存发生了什么 | 只计需求加载的未命中，或统计该级缓存行填充 | 哪一级，是否合并，是否包括预取与未退休活动 |
| 翻译路径有没有变长 | 一级 DTLB 未命中、下级 TLB 命中、完成的页表遍历 | 记录的是开始还是结束，适用于何种地址翻译情况 |
| 请求在等什么 | 某类在途或阻塞周期 | 是否可能同时存在多个原因，事件能否被直接相加 |

</div>

### 为什么要看启用时长和实际运行时长 {#pmu-multiplex}

当请求的事件多于可用计数槽时，Linux 可能轮换计数。`perf_event_open` 可以在读取结果中同时返回计数值、`time_enabled` 和 `time_running`。第一个时间是事件所处的启用区间，第二个时间是它在该区间内实际运行的时长。读软件生成的报告时，要注意它展示的是原始值还是已经换算的值。

设一个事件启用了 10 ms，只有前 4 ms 获得计数槽，原始值是 400。按常用比例换算，估计值是 `400 × 10 / 4 = 1000`。若题设已知前 4 ms 正在准备数据，每毫秒恰发生 100 次，后 6 ms 进入较轻的循环，每毫秒只发生 25 次，全窗口应计 `400 + 150 = 550`。轮换到的区间恰好偏在高频阶段，所以线性换算把它的速率用到了剩余 6 ms。

再考虑要算 IPC（每周期退休指令数）的情形。若指令事件在准备阶段计数，周期事件在求和阶段计数，分别换算再相除会混进阶段差异。将两个事件组成组，是为了让它们在同一段执行上一起获得计数。组本身仍可能受到轮换，或者因为找不到合适的槽位而无法调度，所以还应核对实际运行时长。短小、阶段明确的对照通常比一次开很多事件更便于复算。

#### 分两段读取时，先取差值再换算 {#pmu-delta}

如果程序在运行期间读取同一个事件两次，得到的是该事件自建立以来的累计信息。假设第一次读取为原始计数 400、启用 10 ms、运行 4 ms。第二次读取为原始计数 700、启用 16 ms、运行 6 ms。要估计两次读取之间的事件数，应先分别相减，得到原始增量 300、启用增量 6 ms、运行增量 2 ms。按这一段的比例换算，估计值为 `300 × 6 / 2 = 900`。

若先对两次累计读数分别换算，再相减，就会得到 `700 × 16 / 6 − 400 × 10 / 4 ≈ 867`。这个结果混合了两段不同的运行比例，不能代替第二段自身的估计。若第二段的运行时长增量为 0，就没有这一段的硬件计数时间，无法按比例算出估计值。若计数器在两次读取之间被重置，也要重新建立基线。Linux 的 `PERF_EVENT_IOC_RESET` 只重置计数值，不重置启用与运行时间，因此时间字段仍需用两次读数求差。

::: details 练习｜第二段没有轮换

保留第一次读数，将第二次改为计数 1000、启用 16 ms、运行 10 ms。第二段的运行比例和原始增量分别是多少？

启用与运行增量都是 6 ms，运行比例为 100%。原始增量是 600，这一段按比例换算仍为 600。它依然只覆盖事件配置指定的线程、特权级和启用窗口。
:::

#### 两个事件一起计数，能代表整段执行吗 {#pmu-group-window}

把周期和退休指令放进同一事件组。Linux 会让整组共同占用 PMU，只有所有成员都能安排到硬件计数器上，这一组才会运行。于是指令数和周期数可以在同一段实际计数时间内相除。若整组也需要轮换，还要检查这段时间覆盖了哪些工作。

设同一个求和程序依次运行两个阶段。以下数字是题设。为了便于复算，假定两个阶段核心频率相同，线程一直运行，且事件组只在阶段 A 占用 PMU。

| 阶段 | 实际周期 | 实际退休指令 | 组是否正在计数 |
| --- | ---: | ---: | --- |
| A，索引和数据多数已缓存 | 1,000,000 | 2,000,000 | 是 |
| B，较多加载要等下级返回 | 3,000,000 | 1,500,000 | 否 |

组内原始读数给出 `2,000,000 / 1,000,000 = 2.0`，这是阶段 A 的 IPC。若把两个计数都按 25% 的运行比例放大四倍，求商仍是 2.0。换算不会补上阶段 B 的工作特征。题设已知整段的真实计数，因此可以算出全段 IPC 为 `3,500,000 / 4,000,000 = 0.875`。它也不等于两个阶段 IPC 的算术平均 1.25，因为两个阶段贡献的周期数不同。

::: details 练习｜每个阶段都计一半

保持两个阶段内部的事件速率各自恒定。若事件组在每个阶段都恰好运行一半时间，原始计数的比值是多少？

计到 2,000,000 个周期和 1,750,000 条退休指令，比值为 0.875。两个阶段在样本里保持了原有的周期权重，所以此时能还原整段的 IPC。实测只看到总运行比例时，仍无法据此确认各阶段的覆盖是否均匀。

:::

### 只给求和段打开计数器 {#pmu-region}

前面的程序在数据与索引初始化之后才取第一次单调时钟，由最后一次求和结束后的读数计算耗时。Linux 使用 `CLOCK_MONOTONIC`，Windows 使用 `QueryPerformanceCounter`。这种区间计时保留了窗口内的抢占和等待，也包含函数调用与循环控制。用 `perf stat` 启动整个进程时，即便程序自己只报求和段的时间，外部计数仍会覆盖准备数据的阶段。

下面的 Linux 组件把计数也放进同一个求和区间。它先打开周期事件当作组首，再打开退休指令事件并加入该组。`pid=0` 和 `cpu=-1` 将统计范围设为调用线程。用 `exclude_kernel` 排除内核态，再用 `exclude_hv` 排除虚拟机监控器。两个事件在这对启停调用之间一起计数。墙钟的两个读数放在启停操作的外侧，所以墙钟区间还包括启停开销。

把以下代码保存为 `pmu_window.c`。程序从组首文件描述符一次读出整组结果，并用事件 ID 辨认周期和指令计数。打开事件失败时程序会给出系统错误。可以运行前面不带 `WINDOW_PMU` 的版本，单独核对求和和计时。

~~~c
#define _GNU_SOURCE
#include <inttypes.h>
#include <linux/perf_event.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/ioctl.h>
#include <sys/syscall.h>
#include <unistd.h>

enum { CYCLES, INSTRUCTIONS, EVENT_COUNT };
static int counters[EVENT_COUNT];
static uint64_t ids[EVENT_COUNT];

struct group_read {
    uint64_t count, enabled, running;
    struct { uint64_t value, id; } events[EVENT_COUNT];
};

static void fail(const char *action)
{
    perror(action);
    exit(EXIT_FAILURE);
}

void pmu_prepare(void)
{
    static const uint64_t configs[EVENT_COUNT] = {
        PERF_COUNT_HW_CPU_CYCLES, PERF_COUNT_HW_INSTRUCTIONS
    };
    for (unsigned i = 0; i < EVENT_COUNT; ++i) {
        struct perf_event_attr attr = {
            .type = PERF_TYPE_HARDWARE,
            .size = sizeof attr,
            .config = configs[i],
            .disabled = i == CYCLES,
            .exclude_kernel = 1,
            .exclude_hv = 1,
            .read_format = PERF_FORMAT_GROUP | PERF_FORMAT_ID |
                           PERF_FORMAT_TOTAL_TIME_ENABLED |
                           PERF_FORMAT_TOTAL_TIME_RUNNING
        };
        int group = i == CYCLES ? -1 : counters[CYCLES];
        long fd = syscall(SYS_perf_event_open, &attr, 0, -1, group,
                          PERF_FLAG_FD_CLOEXEC);
        if (fd < 0)
            fail("perf_event_open");
        counters[i] = (int)fd;
        if (ioctl(counters[i], PERF_EVENT_IOC_ID, &ids[i]) < 0)
            fail("PERF_EVENT_IOC_ID");
    }
}

void pmu_start(void)
{
    if (ioctl(counters[CYCLES], PERF_EVENT_IOC_ENABLE, 0) < 0)
        fail("PERF_EVENT_IOC_ENABLE");
}

void pmu_stop(void)
{
    if (ioctl(counters[CYCLES], PERF_EVENT_IOC_DISABLE, 0) < 0)
        fail("PERF_EVENT_IOC_DISABLE");
}

void pmu_report(void)
{
    struct group_read result;
    ssize_t size = read(counters[CYCLES], &result, sizeof result);
    if (size < 0)
        fail("read perf group");
    if ((size_t)size != sizeof result || result.count != EVENT_COUNT) {
        fputs("Incomplete perf group read\n", stderr);
        exit(EXIT_FAILURE);
    }
    close(counters[INSTRUCTIONS]);
    close(counters[CYCLES]);

    uint64_t values[EVENT_COUNT] = {0};
    bool seen[EVENT_COUNT] = {false};
    for (unsigned i = 0; i < EVENT_COUNT; ++i) {
        for (unsigned j = 0; j < EVENT_COUNT; ++j) {
            if (result.events[i].id == ids[j]) {
                values[j] = result.events[i].value;
                seen[j] = true;
                break;
            }
        }
    }

    printf("enabled_ns=%" PRIu64 " running_ns=%" PRIu64 "\n",
           result.enabled, result.running);
    if (!seen[CYCLES] || !seen[INSTRUCTIONS] || result.running == 0 ||
        values[CYCLES] == 0) {
        fputs("No usable IPC for this run\n", stderr);
        exit(EXIT_FAILURE);
    }
    printf("cycles_raw=%" PRIu64 " instructions_raw=%" PRIu64 " ipc=%.3f\n",
           values[CYCLES], values[INSTRUCTIONS],
           (double)values[INSTRUCTIONS] / (double)values[CYCLES]);
}
~~~

~~~bash
gcc -O2 -fno-tree-vectorize -fno-lto -DWINDOW_PMU \
    main.c sum_kernel.c pmu_window.c -o array_sum_pmu
./array_sum_pmu seq
./array_sum_pmu shuffle
~~~

准备工作、计数器组的创建和最终打印都在计数区间外。计数区间覆盖 24 次求和，也包含启停调用之间少量用户态的控制指令。检查输出时，先核对两种顺序的总和，再看 `running_ns` 与 `enabled_ns`。例如两者是 4 ms 与 10 ms，说明组只覆盖了启用时间的一部分。周期和退休指令是在获得计数槽的同一段时间内一起数的，原始值相除得到的是这段时间的 IPC。若该组只在部分阶段运行，IPC 未必代表完整的 24 轮。把两项各乘同一个比例，IPC 也不会因此变得更有代表性。

### 为什么样本会落在加法指令上 {#pmu-sampling-skid}

上面的计数程序在窗口结束后读取累计值。如果想知道循环里哪条指令更常遇到 L1 未命中，就要采样。以固定周期为例，PMU 每累计一定数量的所选事件，就产生一次采样机会。Linux 可以把样本写入环形缓冲区，供用户态读取。样本可包含 `PERF_SAMPLE_IP`，即记录的指令地址。累计值回答窗口里数到了多少次，样本用于估计这些事件在代码里的分布。

设求和循环被编译成下面的指令序列。地址和寄存器是例题给定的，依赖关系与前文的间接索引求和相同。

```asm
0x1200  mov  eax, DWORD PTR [rsi+rcx*4]  ; 读索引
0x1203  mov  edx, DWORD PTR [rdi+rax*4]  ; 按索引读数据
0x1206  add  r8, rdx                   ; 累加
0x1209  inc  rcx                       ; 前进到下一个索引
```

假设 `0x1203` 的一次退休加载触发了 L1 未命中事件的采样。如果使用普通的溢出中断记录指令地址，事件触发后，处理器可能继续执行几条指令，到采样时才记下 `0x1206` 或 `0x1209`。这种事件位置与记录位置之间的偏移称为 **skid**。如果报告把样本记在 `add` 上，把它旁边的计数直接归为这条加法的缓存行为，就会把事件解释错。

以本篇的 Skylake Server 为例，Intel 的事件表将 `MEM_LOAD_RETIRED.L1_MISS` 标为支持 PEBS（Precise Event Based Sampling，精确事件采样）记录触发事件的指令地址。Linux 的 `precise_ip` 可以请求更严格的定位，取值 2 请求零 skid，取值 3 要求零 skid，能否使用要看 PMU 和所选事件。读原始样本时，`PERF_RECORD_MISC_EXACT_IP` 表示该条样本的 IP 对应触发事件的指令。若在这个例子里收到标为精确的 `0x1203`，就可以把样本定位到第二条加载。

即使定位精确，样本的单位仍由事件定义决定。这里统计的是满足条件的已退休加载指令。如果窗口内有 10,000 次符合事件定义的加载，采用每 1,000 次触发一次的教学设定，只会得到约 10 次采样机会。实际读报告还要核对丢失记录、采样周期和计数器运行时长。若要比较循环中两条加载各自的未命中**比例**，还需要对应的加载执行次数，不能直接拿两条指令的未命中样本数当比例。

::: details 练习｜样本集中在哪条加载，能说明什么？

一段展开后的循环里，加载 A 执行 1,000,000 次，加载 B 执行 100,000 次。假设采样没有丢失，每 1,000 次事件得到一个样本，两处分别有 40 和 20 个精确未命中样本。哪条加载的未命中总量较大？哪条加载的未命中比例较高？

按题设估算，A 约有 40,000 次未命中，B 约有 20,000 次。除以各自执行次数后，A 约为 4%，B 约为 20%。同样的采样周期支持比较事件总量，比较未命中率还要除以各自的执行次数。
:::

## 读完四组数据，能把差异解释到哪一步 {#pmu-attribution}

沿用同一访问循环，假设实验还能够控制数据缓冲区使用普通页还是大页。若要先算页大小怎样改变遍历次数，可以复习[四个地址的 TLB 容量例题](./microarch-02a-translation.md#sv39-tlb-reach)。每种条件重复多次，给出各条件的中位数，计数器均覆盖完整的测量窗口。以下是**构造数据**。上面的可运行程序只提供访问顺序，页大小和表中数值是这道归因题的额外条件。

读 PMU 表时要把三种结论分开。第一，某事件计数增加，只说明在所选范围和事件定义下记录到更多次活动。第二，时间同时增加，说明活动与变慢出现在同一测量条件中。第三，要说该活动造成了多少延迟，还需要知道它是否处在关键路径，以及它是否与其他等待重叠。比如一次页表遍历可能发生在数据加载的关键路径上，也可能与另一笔独立加载并行；遍历次数相同，暴露给程序的等待时间可以不同。反过来，页表遍历数增加而耗时几乎不变，也可能表示遍历被其他工作隐藏，不能据此断言计数器错误。

因此，归因至少要同时核对访问路径、事件定义、测量窗口和依赖关系。改变页大小后，如果遍历数下降而时间也下降，这与“翻译等待有所减少”相符；若要把时间差全算给 TLB，还需排除页数变化对缓存占用、预取行为和内存布局的影响。PMU 提供的是观察约束，程序依赖链和对照条件才帮助判断哪些活动能限制完成时间。

<div role="region" aria-label="普通页与大页的四组题设数据" style="max-width:100%; overflow-x:auto">

| 地址访问顺序 | 关联数据的页大小 | 时间 | 完成的页表遍历 | 数据需求加载的 L1 未命中 |
| --- | --- | ---: | ---: | ---: |
| 顺序 | 普通页 | 120 μs | 32 | 320 |
| 乱序 | 普通页 | 228 μs | 190 | 520 |
| 顺序 | 大页 | 111 μs | 2 | 340 |
| 乱序 | 大页 | 151 μs | 3 | 510 |

</div>

普通页下两个版本差 108 μs，大页下差 40 μs；遍历次数的差从 158 降到 1。可以支持的解读是：改变页大小后，遍历行为和时间差同时显著变化，翻译成本是值得进一步验证的解释。与此同时，L1 未命中仍然有不小差别。它可能与大页条件下残留的 40 μs 有关，但这张表还没给出从哪一级返回、请求重叠程度或预取效果。

很容易进一步写成“每次页表遍历固定花了 `68 μs / 157`”。这个结论还需要证明两个版本间的其他成本都不变，而且所有遍历都阻塞了程序的关键路径。大页还可能改变虚实布局、缓存组分布或预取所见的页边界，所以可以继续做更细的对照。测量前应分别准备可比较的缓存初态和翻译初态，并记录页大小是否确实生效。对有重复阶段的程序，可以交错运行 A、B，观察各轮差值的分布，排除总是先测某一方导致的初态偏差。

如果目标是确认“某些迟到的预取获益”，应找能够说明地址请求、在途合并或填充时序的证据。一个总缓存未命中计数可能保持不变，因为[需求仍发生了 L1 未命中](./microarch-02-prefetch.md#prefetch-fill)，它只是更早拿到同一个下级响应。选择记录时，从假设的因果链倒推所需事件，比先收集许多同名计数更有效。

### 同样四次未命中，为什么有时只等一轮 {#pmu-miss-overlap}

把上面的归因问题缩小为四次加载。假设四条加载各读不同的冷缓存行，都发生 L1 未命中，每条从发出到数据返回都要 80 周期。比较两种数据依赖。第一种访问沿链表前进，后一条要等前一条返回地址。第二种的四个地址起初都已就绪，可以连续发出，最后一条计算需要全部四个结果。题设允许每周期发一条，并忽略地址生成、退休和最后计算的额外延迟。

<div role="region" tabindex="0" aria-label="四次缺失请求的发出与返回时刻" style="max-width:100%; overflow-x:auto">

| 数据依赖与资源 | 四笔请求的发出周期 | 四笔请求的返回周期 | 从首次发出到数据全部齐备 |
| --- | --- | --- | ---: |
| 串行，后一条的地址依赖前一条 | 0、80、160、240 | 80、160、240、320 | 320 周期 |
| 地址独立，允许四笔在途 | 0、1、2、3 | 80、81、82、83 | 83 周期 |
| 地址独立，但仅允许两笔在途 | 0、1、80、81 | 80、81、160、161 | 161 周期 |

</div>

第三行假定返回当拍可以重新使用空出的名额。第 2 周期第三个地址已经就绪，但前两项仍占用在途资源。直到第 80 周期第一项返回，第三项才能发出。对应到缓存结构，可以把这个限制理解为只剩两项 [MSHR](./microarch-01a-mshr.md#mshr-limits) 可用。具体机器还可能受发射端口或下级排队限制。

三行都累计了 `4 × 80 = 320` 个请求周期，这相当于把每条请求的在途区间分别计时。多个区间重叠时，同一处理器周期会被重复计入这笔总和。完整的程序耗时还要追踪最后一项结果到来前，程序能否在其他指令上做有用工作。实际测量中，如果未命中数相同而时间不同，可以先从反汇编找出地址依赖，再核对在途请求数量和下级服务时间。[乱序执行篇](./microarch-03-execution.md#ooo-retire)解释了为什么返回与退休也可能分处不同周期。

::: details 练习｜还有一笔独立工作

沿用第二行。假设最后一条计算除了四笔读的结果，还要等另一笔与这些加载无关的计算到第 100 周期才结束。若把单笔读延迟从 80 降为 70，最后一条计算能提前吗？

四个读结果从第 83 周期全部齐备提前到第 73 周期，但另一项输入还是第 100 周期才就绪。最后一条计算的最早开始时刻仍为 100。即使测到了内存侧的改善，也要看被缩短的等待是否限制了最终完成时间。
:::

## 阈值判断要带上分母

另一个常见任务是用单次耗时区分本级有无可用数据。先通过独立方法准备两组已知的状态，在另一批样本上评估一个预先确定的阈值。下表仍是构造数据，每行 100 次：

<div role="region" aria-label="计时判据的混淆矩阵" style="max-width:100%; overflow-x:auto">

| 实际状态 | 判作数据在近处 | 判作需要下一层 |
| --- | ---: | ---: |
| 数据在近处 | 93 | 7 |
| 数据需要下一层 | 11 | 89 |

</div>

将“在近处”视为阳性，假阳性率为 11%，假阴性率为 7%。把判据用于另外 1000 次读取，若只有 100 次确实在近处，并假定条件错误率保持不变，约有 93 次真正的近处访问落入阳性。其余 900 次会贡献约 99 次假阳性。阳性中的正判比例是 `93/(93+99)`，约 48%。这解释了为何在均衡验证样本上表现良好的判据，遇到稀少状态时仍会产生许多误判。

实际实验还应保留原始分布、训练和验证的分界、初态准备方法及重复结果。当“近处”的标签本身来自同一次计时，便会让证据绕回原点。独立的初态控制或事件验证能帮助打破这种循环。到这里可以回答“时间怎样支持一个分类”。接下来还要回答这个分类提供了关于什么的新信息。

::: details 练习：相同的 L1 未命中是否意味着相同的等待
两个版本各有 50 次 L1 需求未命中。A 中有 40 次在需求前已经由预取发出对应下级请求，需求到达后合并。B 中 50 次都要在需求到达后启动。两者的 L1 未命中计数可以相同吗？至少还要对照哪两类信息？

可以。合并前的一次查找仍可能在 L1 未命中。至少要看等待来自哪一级，以及下级请求是何时开始的或有无在途合并。若要算总时间，还应考虑多个需求之间是否可以并行。
:::

## 参考文献

- [Linux v6.12 的采样精度请求与样本标志](https://github.com/torvalds/linux/blob/v6.12/include/uapi/linux/perf_event.h)
- [Intel Skylake Server 核心性能事件表](https://perfmon-events.intel.com/platforms/skylakex/core-events/core/)
- [Intel 软件开发者手册第 3 卷，性能监控](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [Linux 性能事件权限](https://docs.kernel.org/admin-guide/perf-security.html)
- [Linux perf_event_open](https://man7.org/linux/man-pages/man2/perf_event_open.2.html)
- [Linux perf stat 的计数和计时选项](https://man7.org/linux/man-pages/man1/perf-stat.1.html)
- [Linux perf list 的事件分组与可用事件](https://man7.org/linux/man-pages/man1/perf-list.1.html)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>上一篇</small><strong>分支预测器怎样决定下一次取指</strong></span>](./microarch-03a-fetch-prediction.md){.article-nav-card .article-nav-prev rel=prev}

[<span class="article-nav-text"><small>下一篇</small><strong>一次缓存计时能说明什么</strong></span>](./microarch-04b-observation.md){.article-nav-card .article-nav-next rel=next}

</nav>
