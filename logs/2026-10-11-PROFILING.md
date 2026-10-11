# 2026.10.11｜Scheduling、PELT、Cgroup 与 DVFS

## 1. Scheduling Policy、Nice 与 Scheduling Weight

`policy` 决定线程采用哪种调度策略。Linux 中常见的策略包括：

- `SCHED_OTHER`：普通公平调度，根据权重分配 CPU 时间；
- `SCHED_FIFO`：实时先进先出调度；同优先级线程可以持续运行，直到主动让出 CPU、阻塞，或被更高优先级任务抢占；
- `SCHED_RR`：实时轮转调度，同优先级线程按照时间片轮流执行；
- `SCHED_IDLE`：极低优先级调度，主要用于不着急执行的后台任务。

**Fair Scheduling 要求线程按照各自的权重公平地获得 CPU 时间，而不是每个线程获得相同的 CPU 时间。**

`nice` 不是缩写，它表示一个线程在 CPU 竞争中对其他线程有多“友善”。取值范围为 `-20` 到 `19`，数值越低，对应的调度权重越大。

Linux 会把 nice 映射成具体的调度权重，例如：

- `nice = 0`：标准权重为 `1024`；
- `nice = 5`：标准权重为 `335`。

对于两个持续竞争同一个 CPU 的普通线程，理想情况下：

`CPUShare_i = w_i / Σ_j w_j`

因此，权重为 `1024` 的线程可以获得约 75% 的 CPU 时间，权重为 `335` 的线程获得约 25%。

## 2. Scheduling Entity 与 PELT

`sched_entity`（简称 `se`）是 Linux 内核中的调度实体。它不一定代表单个线程；启用 Group Scheduling 后，也可以代表一个 Task Group。

几个重要字段：

- `se.load.weight`：调度权重；
- `se.vruntime`：虚拟运行时间；
- `se.sum_exec_runtime`：累计实际 CPU 执行时间；
- `se.avg`：保存 PELT 相关的负载统计信息。

PELT 是 Per-Entity Load Tracking。**PELT 不是调度策略，而是 Linux 用来跟踪调度实体近期负载和 CPU 利用率的机制。**

`se.load.weight` 是相对静态的调度权重，反映线程在公平调度中的资源分配比例。PELT 则根据线程近期的运行状态，动态更新负载估计：

- `load_avg`：结合调度权重计算的平均可运行负载；
- `runnable_avg`：线程近期处于 Runnable 状态的程度；
- `util_avg`：线程近期实际处于 Running 状态的程度。

## 3. Cgroup、Task Group 和 CPU Shares

Cgroup（Control Groups）是 Linux 用来管理一组任务资源的机制，可以控制 CPU、内存和 I/O 等资源。

**Task Group 不会创建新的 Scheduling Policy，但会引入新的调度层次。因此，即使不改变线程本身的 nice，也可能改变最终的 CPU 分配结果。**

例如，三个权重相同、持续运行的线程 A、B、C，如果直接在同一层竞争 CPU，它们各自可以获得约 33.3% 的 CPU 时间。

但是，如果把 A、B 放在 Group 1，把 C 放在 Group 2，并且两个 Group 的权重相同：

- Group 1 先获得 50% 的 CPU 时间；
- Group 2 获得另外 50%；
- Group 1 再将自己的 CPU 份额分给 A、B。

最后 A、B 各获得 25%，C 获得 50%。这说明在 Hierarchical Scheduling 中，调度器先分配组与组之间的资源，再分配组内线程的资源。

`cpu.shares` 属于 cgroup v1，默认值是 `1024`，而不是最大值。它表示兄弟 cgroup 之间的相对 CPU 权重。cgroup v2 对应的接口是 `cpu.weight`，默认值为 `100`。

还需要区分 `cpu.weight` 和 `cpu.max`：

- `cpu.weight`：软性权重，决定竞争时的相对份额；
- `cpu.max`：硬性 CPU 配额，超出限制后可能导致 CPU throttling。

## 4. DVFS 与 Frequency Pinning

重新讨论之前阅读的 DVFS 论文 CORE 时，一个关键问题是：如果论文通过软件控制 CPU 频率，硬件在运行过程中会不会自行改变频率？如果会，软件层面的频率控制是否还有意义？

这里需要区分三个概念：

- **DVFS Governor**：根据系统负载等信息，动态选择目标 CPU 频率；
- **Frequency Pinning**：通过软件接口尝试固定目标频率，例如设置 `scaling_min_freq = scaling_max_freq`；
- **Thermal Throttling**：系统温度过高时，热管理机制限制硬件性能，可能导致实际运行频率下降。

因此，软件请求的目标频率不一定等于硬件实际运行的频率。

CORE 通过软件接口设置 CPU/GPU 频率，先进行 Offline Profiling，再选择合适的频率组合。但它的主要实验避开了严重热降频场景。因此，CORE 主要解决的是如何选择合适的运行频率，而不是如何在持续高温状态下保证频率始终稳定。

硬件频率的动态变化也不都是 Thermal Throttling 导致的。正常的 governor 调频、内核热管理和固件功耗限制，都可能改变实际频率。

修改频率时，控制链路大致是：

```text
adb/root 脚本
    ↓ 写入 scaling_min_freq / scaling_max_freq
Linux cpufreq policy
    ↓
governor（schedutil / waltgov）选择目标频率
    ↓
高通 cpufreq 驱动
    ↓
高通固件、PMIC、硬件 DVFS
    ↓
CPU 实际运行频率
```

同时还有一些并行控制者：

- Android Power HAL / vendor daemon；
- thermal 热管理；
- 电流与功耗保护 BCL；
- WALT / vendor scheduler；
- 高通固件。
