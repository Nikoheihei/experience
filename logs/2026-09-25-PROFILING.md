# Profiling 记录（2026-09-25）

| 主题 | 今天做了什么 | 结果 / 结论 | 当前状态 |
|---|---|---|:---:|
| **Idea 1：NPU burst 与 UI 帧相位** | 用 **B8/B16/B32** 跑相同的 1024-token prompt，进行三轮轮换实验；分析 phase、signed φ、age、remaining 和 continuous busy。 | 没有发现稳定的相位、graph age 或 continuous busy → 卡顿关系；B8/B16/B32 的卡顿排序也不稳定。 | ❌ |
| **“缩短 prefill graph 能减少卡顿”** | 比较 B8/B16/B32 的 QNN graph 时长和 UI 卡顿。 | B32 比 B8 快约 3.6×，但三组单个 QNN layer-chunk 都约 29 ms；B 改变的是总 prefill 时长，不是不可抢占的 graph 时长。 | ❌ |
| **UFS restore 是否是新问题** | 分析 NPU 高优先级压制上下文恢复的可能性。 | 即使发生，本质仍然是内存带宽竞争问题。 | ❌ |
| **Storage restore vs. NPU KV regenerate** | 将 mzCache restore 与 NPUGen regenerate 放进同一个交叉点问题中。 | 可能随着 NPU 占用、KV 大小和 UFS locality 出现路径优劣反转，但今天还没有正式测量。 | ❓ |
| **Android 17 NPU priority** | 查询 Android 17 的 NPU priority、preempt/resume 和 Scheduling HAL。 | 有软件优先级接口，但没有证据说明它会影响 NoC/DRAM 仲裁；设备实现也不确定。 | ❓ |

