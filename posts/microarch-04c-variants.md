---
title: Spectre 变体中的分支与访存推测
description: 用一次 Linux 地址空间切换算出 IBPB 何时发出，再区分预测目标、分支历史和访存推测的保护范围。
date: '2026-11-25'
tags: [微架构安全, 瞬态执行, 分支预测, BHB, 缓解措施]
book: microarch-00-intro
---

两个用户进程先后在同一个逻辑 CPU 上运行。切换时，操作系统已经换了地址空间，处理器还有哪些旧的预测状态需要处理？若前一个进程与后一个进程的权限级相同，系统进入内核时采用的间接分支保护又能覆盖到哪里？

从这次切换出发，可以理解为什么 Linux 的状态报告会分别列出间接分支预测屏障、分支历史保护和推测性存储绕过保护。以下以 Intel 的接口定义和 Linux v6.12 的 x86 路径为例。表中的任务与地址空间是教学题设。

[[toc]]

## 同样是预测，处理器在等什么 {#variant-predictions}

解释器可能根据操作码调用不同的处理函数。对一个普通的间接调用，前端可先用预测的函数地址取指，等寄存器里的实际目标就绪再核对。目标不一致，错误目标上的工作会被撤销。这里需要确认的是调用地址。换成 [Spectre v1 的边界检查](./microarch-04d-spectre-v1.md)，需要确认的是条件分支的方向。还有一些推测发生在加载／存储单元里，即使分支完全正确也可能发生。

| 等待确认的条件 | 最终由哪里决定 | 与本节相关的保护对象 |
| --- | --- | --- |
| 条件分支往哪边走 | 执行端完成比较 | 检查通过前，后续操作能否继续执行 |
| 间接调用去哪个目标 | 执行端读到实际操作数 | 目标预测可以受到哪些执行历史的影响 |
| 先前的存储是否与加载重叠 | 加载／存储单元取得地址后检查 | 旧存储地址未决时，是否允许加载先执行 |

Spectre v2 涉及间接分支的预测目标。把一个错误目标放回硬件过程时，要分别说明取指去了哪里、何时恢复，以及是否有内部活动能被[给定观察者](./microarch-04b-observation.md#evidence-observer)区分。下面专门讨论系统怎样保护目标和历史的使用范围。

## 分清三条可能共享预测状态的边界 {#variant-boundaries}

Intel 用**预测模式**描述宿主／来宾、用户／特权的组合。一次系统调用从用户态进入内核会改变预测模式。同一逻辑 CPU 先后运行两个普通用户进程，虽然调度过程经过内核，但两个进程各自执行用户代码时都处在同一个用户预测模式。把这两种情况混在一起，容易把“跨权限级保护”读成“两个进程已经隔离”。

| 边界 | 可以核对的控制 | 这项控制解决的事 |
| --- | --- | --- |
| 同一个逻辑 CPU 先后运行不同任务 | IBPB，间接分支预测屏障 | 划开执行的前后两段，限制屏障前的软件影响屏障后的间接目标预测 |
| 进入更高权限的预测模式 | IBRS／eIBRS，间接分支受限推测／增强版 | 按处理器能力与启用方式保护该模式中的间接目标。增强版支持持续启用 |
| 物理核心的两个逻辑线程同时运行 | STIBP，单线程间接分支预测器控制 | 限制同核另一逻辑线程对本线程间接目标预测的影响 |

IBPB 是在需要隔离的时刻发出的命令，系统软件通过 `IA32_PRED_CMD` 下发。IBRS 和 STIBP 则是处理器支持时可以设置的控制。不同机制的覆盖会在部分情形下重合，例如 Intel 对启用 eIBRS 时的同核线程影响也有规定。阅读系统配置时，应分别核对顺序切换、权限变化与同时运行，不必预设三种接口每次都各用一次。

若要先确认哪些任务可能同核同时运行，可以用[核心调度的任务配对例题](./microarch-06d-core-scheduling.md#core-pick)把两条运行队列走一遍。

### 为什么还要单独看 BHB

[分支预测一节](./microarch-03a-fetch-prediction.md#predict-isolation)已经说明，保存目标和参与选择的历史是两类状态。即使较低权限模式训练出的目标项不能直接被使用，预测器也可能使用先前的分支历史来选择允许的项。Intel 将这类历史影响纳入 BHI（Branch History Injection，分支历史注入）的说明。

在支持 `BHI_DIS_S` 的 Intel 处理器上，这项控制限制特权模式下的间接目标选择使用用户态分支历史。基于返回栈缓冲（RSB）的返回预测另有条件。Linux 对需要软件处理的情况，也可能在适用的边界运行历史清理序列。判断措施时，先找它限制的是目标项来源、历史来源，还是返回目标的来源。硬件文档给出保护关系，不能据此反推内部表项用了哪些物理位。

## Linux 何时会发出一次 IBPB {#variant-switch}

Linux v6.12 的 x86 地址空间切换路径，用 `cond_mitigation()` 判断进程间的几类保护。例题只跟踪其中的 IBPB。用到的对象有三项。

| 内核对象 | 本例中表示什么 |
| --- | --- |
| `task_struct.mm` | 即将运行的用户任务所用地址空间。共享地址空间的线程可指向同一个 `mm` |
| `TIF_SPEC_IB` | 内核用于间接分支推测保护的任务标志。条件式检查需要考虑离开和进入的两侧 |
| `last_user_mm_spec` | 当前逻辑 CPU 上次记录的用户地址空间及相关标志。它不是上一个调度实体的任务编号 |

在切换路径进入这个函数后，内核用地址空间指针和任务标志组成 `next_mm`，与处理器本地保存的上一份用户记录比较。条件式路径要求两份记录不同，并且至少一边带有 IBPB 标志。常开路径比较的是去掉相关标志后的上一份地址空间与即将运行的地址空间。在此之后，函数更新这份逻辑 CPU 本地记录。普通的同地址空间切换另有分支。没有用户 `mm` 的任务不会在该函数里更新上一份用户记录。

假定 CPU 上最后运行的用户线程是 A，地址空间为 α，标志为 1。接下来的任务按表中顺序运行。共享同一地址空间的两条线程在题中保护标志一致。表中计算普通用户地址空间切换，内核线程一行仅说明对用户记录的影响。

| 即将运行 | 新用户地址空间与标志 | 条件式 | 常开 |
| --- | --- | --- | --- |
| A 的另一个线程 | α，1 | 无屏障 | 无屏障 |
| 内核线程 K | 无用户地址空间 | 保留 α 的记录 | 保留 α 的记录 |
| B | β，0 | 发屏障，上一份 α 有标志 | 发屏障，地址空间改变 |
| C | γ，0 | 无屏障，两侧均无标志 | 发屏障，地址空间改变 |
| D | δ，1 | 发屏障，即将运行的一侧有标志 | 发屏障，地址空间改变 |

从 K 到 B 时，判断用的是 α 和 β。K 自己没有替代那份用户记录。B 到 C 则说明状态报告中出现条件式 IBPB，只能说明这条策略已选用。具体到一次切换，还要看任务和逻辑 CPU 的实际状态。

### 用代码复算这张切换表

下面的 Python 3 教学模型只接受上面这种普通切换序列。同一地址空间的保护标志固定，内核线程用 `None` 表示。模型统计哪些步骤需要一个屏障，所有对象都是本地构造的数据。

```python
steps = [
    ("A2", ("alpha", True)),
    ("K", None),
    ("B", ("beta", False)),
    ("C", ("gamma", False)),
    ("D", ("delta", True)),
]


def barrier_steps(last_user, transitions, *, always):
    points = []
    for name, next_user in transitions:
        if next_user is None:
            continue

        previous_mm, previous_flag = last_user
        next_mm, next_flag = next_user
        changes_address_space = previous_mm != next_mm
        if changes_address_space and (always or previous_flag or next_flag):
            points.append(name)
        last_user = next_user
    return points


start = ("alpha", True)
conditional = barrier_steps(start, steps, always=False)
always = barrier_steps(start, steps, always=True)
assert conditional == ["B", "D"]
assert always == ["B", "C", "D"]
print("conditional:", conditional)
print("always:", always)
```

实际源码的条件式路径比较的是带标志的记录。本例限定同一地址空间中的标志一致，所以可以用 `changes_address_space` 直接复算。实际部署还要检查哪些路径调用该函数，以及每个逻辑 CPU 保存的记录，不能把表中一次调度的结果当成全机行为。

## 存储地址未定时，SSBD 管哪一步 {#variant-ssbd}

回到[加载与存储队列](./microarch-03d-load-store.md#lsu-replay)。设普通程序先把计数值改成 9，再从同一位置读回。较老存储的物理地址迟到，加载可能暂时取得缓存中的 5。后来发现地址重叠，核心重新执行加载，最终结果修正为 9。这条恢复链条与间接调用的地址预测无关。

Intel 把这类暂时越过较老存储的行为称为推测性存储绕过。支持 SSBD（Speculative Store Bypass Disable）的处理器在启用它时，加载会等较老存储的地址已知，再进行相应的推测执行。放到计数值例子里，变化发生在加载首次尝试取数之前。此时一旦判定同址，就需要满足这条真实依赖，等存储数据可用后转发，或在合适的时刻从缓存读回。最终读到的仍应是 9。

因此，从 `spectre_v2` 的间接分支状态推不出 SSBD 状态。Linux 分别报告这些措施。它们约束的硬件和生效时机不同。

## 怎样读 Linux 的状态报告 {#variant-status}

在自己的 Linux 系统上，运行下列只读命令可查看内核的报告。

```bash
cat /sys/devices/system/cpu/vulnerabilities/spectre_v2
cat /sys/devices/system/cpu/vulnerabilities/spec_store_bypass
```

Linux v6.12 中，`spectre_v2` 的状态可能由多个字段组成。下面列出源码里的字段示例，便于知道该去查哪个接口。这些片段分别说明字段的含义。

| 读到的字段 | 可以据此确认什么 |
| --- | --- |
| `IBPB: conditional` | 内核选用了有条件的屏障策略。逐次切换还要看上一份和下一份用户记录 |
| `BHI: BHI_DIS_S` | 内核报告使用了硬件的特权分支历史限制 |
| `BHI: SW loop, KVM: SW loop` | 内核报告了相应的软件历史处理方案 |
| `spec_store_bypass` 中的 `disabled via prctl` | 内核提供按任务控制的策略。要了解具体任务的状态，还需知道其生效设置 |

核对真实部署时，应一并记录处理器型号、微码、内核版本和运行环境。虚拟机里读到的是来宾内核根据其可见能力和配置给出的状态。它适合回答“这个内核选了什么”，而特定信任边界上的结果还取决于宿主和处理器实际提供的保证。

::: details 练习：把 B 迁移到另一个逻辑 CPU

上面的条件式例题中，B 在同一个 CPU 上接在 A 后面，所以发出了屏障。如果 B 改在另一个逻辑 CPU 上运行，该 CPU 最后一份用户记录是 γ，标志为 0。按题设，B 自己仍为 β，标志为 0。把新初态代入 `barrier_steps`，结果怎样？

对这次切换，条件式不发屏障，常开策略仍会因为地址空间从 γ 变到 β 而发屏障。改动的是负责判断的逻辑 CPU 及其上一份记录。原先的逻辑 CPU 保存的 α 不会替代这份记录。如果两个逻辑 CPU 恰好是同一物理核心的两个线程，还要另外核对同时运行时的预测控制。
:::

## 参考文献

- [Intel 推测执行缓解说明，IBRS、STIBP、IBPB 与 SSBD](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/speculative-execution-side-channel-mitigations.html)
- [Intel 关于 BHI 与特权模式历史限制的说明](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/branch-history-injection.html)
- [Linux v6.12 x86 调度代码中的条件式和常开 IBPB](https://github.com/torvalds/linux/blob/v6.12/arch/x86/mm/tlb.c#L380)
- [Linux v6.12 x86 缓解状态的输出代码](https://github.com/torvalds/linux/blob/v6.12/arch/x86/kernel/cpu/bugs.c#L2820)
- [Linux v6.12 的 Spectre 状态接口](https://docs.kernel.org/6.12/admin-guide/hw-vuln/spectre.html#spectre-system-information)

<nav class="article-nav" aria-label="系列文章导航">

[<span class="article-nav-text"><small>上一篇</small><strong>进入内核时怎样切换页表</strong></span>](./microarch-04f-page-table-isolation.md){.article-nav-card .article-nav-prev rel=prev}

[<span class="article-nav-text"><small>下一篇</small><strong>微架构安全学习资料</strong></span>](./microarch-05-sources.md){.article-nav-card .article-nav-next rel=next}

</nav>
