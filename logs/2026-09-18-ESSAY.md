# Systems Research 阅读总结（2026-09-18）

## 今天的三个 Systems Research 感悟

| 感悟 | 核心意思 | 典型 research path |
|---|---|---|
| **1. Characterization → Simple Control** | 很多系统论文最后的方法其实很简单，真正困难的是先通过大量 profiling 找到“到底哪里出了问题”。 | 发现异常 → 控变量实验 → 找 root cause → 找一个 knob → 调节 |
| **2. Abstraction Mismatch** | 现有系统把某个东西当成了错误的执行或调度单位，导致效率低；论文重新定义粒度或 abstraction。 | 发现旧 abstraction 太粗或不适用 → 拆细或重定义 → 获得新的调度自由 |
| **3. Missing Primitive → Workaround Explosion** | 问题已经找到，但硬件、OS 或 runtime 没有直接控制手段，只能绕路，机制因此越来越复杂。 | 找到 root cause → 缺少合适 knob → 找 proxy mechanism → proxy 又产生问题 → 不断补机制 |

## 今天读的论文总表

| 论文 | 研究对象 | 它发现的问题 | 核心解决方法 | 属于哪类感悟 | 值得记住的 insight | 仍然没解决或脆弱处 |
|---|---|---|---|---|---|---|
| **FastServe, NSDI ’26** | 云端 LLM，多用户请求共享 GPU | output length 长尾；长请求挡住短请求，造成严重 HOL blocking，而 output length 事先未知。 | iteration-level preemption + Skip-Join MLFQ；用 input length 预测 first iteration；主动交换 KV cache。 | **2. Abstraction mismatch** | **整个 request 不应是不可分割的 scheduling unit；autoregressive token iteration 是天然的抢占点。** | queue quantum 和 latency predictor 依赖 model/hardware；复杂 batching 和 interference 下，预测是否仍稳定？ |
| **Sarathi-Serve, OSDI ’24** | 云端 LLM serving，混合 Prefill 与 Decode | 一次完整 Prefill 太重，会让 Decode 长时间 stall；只优先 Decode 又会降低 GPU 利用率。 | 将 Prefill 切成 chunks，并与 Decode 混合组成 batch。 | **2. Abstraction mismatch** | **完整 Prefill 不必是 atomic unit；切碎重 workload 后，可以填充 Decode 的空闲容量。** | chunk size 或 token budget 的选择依赖模型、GPU、batch 和 context distribution。 |
| **ExoMem, MobiCom ’26** | Jetson/UMA 上运行超大本地 LLM | framework 认为 tensor 已释放，不代表 OS 真正回收 physical pages；重复 copy 也可能造成 OOM。 | OS-governed memory；shell/payload separation；zero-copy anchor；Dormant state 与 opportunistic re-anchoring。 | **2 + 3 的混合** | **framework 的逻辑内存生命周期和 OS 的物理页生命周期不是一回事。** | 依赖特定 UMA/Jetson 环境；主要解决 capacity，不解决长期运行中的 thermal/performance 变化。 |
| **Decentralized Adaptive Scheduling, ATC ’23** | Android 上多个 DNN/App 共存 | standalone 最佳配置到了 co-running 环境不再最佳；App 也不知道其他 App 在做什么。 | 每个 App 用自己的 RL agent，根据 CPU/GPU/memory utilization 自适应选择 CPU、GPU、线程等配置。 | **1. Characterization → Control** | **不必知道竞争者是谁，只需观察它在共享资源环境里留下的状态变化。** | 主要做 spatial scheduling；temporal scheduling 交给 OS。只看 utilization 可能抓不到更深层的共享资源问题。 |
| **CORE, MLSys ’26** | 手机上的 CPU–GPU LLM inference 与 DVFS | CPU/GPU governor 各自独立调频，但 workload 实际强耦合，可能出现 frequency downward spiral。 | profile CPU/GPU frequency combinations，建立 device-model profile；runtime 根据 Prefill/Decode 和长度查表。 | **1. Characterization → Simple Control** | **局部 governor 的 utilization signal 会被另一个 component 的频率影响，因此局部最优不等于全局最优。** | 假设 device-model 的 offline optimal profile 较稳定；thermal state 或 background interference 是否会造成 profile drift 尚未充分回答。 |
| **SERENO, OSDI ’26** | 后台 mobile LLM 与前台 UI 共享 SoC memory bandwidth | 后台 NPU LLM 几乎不掉速，却把前台 UI 的 memory latency/jank 打高；NPU graph 又是 run-to-completion。 | 用 speculative decoding 和 draft-model layer subgraphs 制造小于 1 ms 的 yield points；verify 阶段配合 micro-sleep/control。 | **3. Missing primitive → Workaround explosion** | **真正缺的是 fine-grained NPU yield 或 bandwidth-control primitive；speculative decoding 被用作 scheduling substrate。** | 与 speculative decoding 强耦合、机制复杂；target verification 仍有 5–8 ms 的粗粒度；如果硬件直接提供 bandwidth QoS/yield primitive，许多机制便不再需要。 |
| **Pantheon, MobiSys ’24**（今天作为对照重新讨论） | mobile edge GPU 上多个有 deadline 的 DNN | 把整个 DNN 当作一个整体，难以实现 fine-grained preemption 和 deadline control。 | 利用 layer/early-exit boundaries 做更细粒度调度和提前退出。 | **2. Abstraction mismatch** | **把整个 DNN 当 atomic unit 太粗；可以沿 model depth 制造 control points。** | slice/exit 位置和 timing 依赖 workload；还会引入 accuracy–latency tradeoff。 |
| **FLAME, 2026 preprint**（作为 CORE 对照） | mobile CPU–GPU inference latency modeling | GPU inference latency 不只由 GPU frequency 决定；CPU kernel launch 与 GPU execution 异步耦合。 | 显式建模 CPU launch/GPU execution overlap，并按 layer 聚合，预测不同 frequency 下的 latency。 | **1. Characterization** | **CPU frequency 可以通过“喂 GPU”的速度改变 GPU latency，即使主要计算发生在 GPU 上。** | thermal、DDR、memory contention 等额外状态是否会导致模型 drift，尚未完全解决。 |
| **EnerInfer, 2026 preprint**（作为对照） | on-device LLM 的 NPU/DDR energy control | 最大频率不一定 energy-optimal；不同模型的最佳配置不同。 | throughput/power predictor + QoE constraint + thermal MPC。 | **1. Characterization → Control** | **性能最优和能效最优不是同一个 operating point。** | 没有真正解释 temperature → performance 的机制；更偏控制，而非 causal characterization。 |

> 注：表中“仍然没解决或脆弱处”是本轮阅读讨论中的归纳，部分是根据论文方法和假设提出的延伸问题，不一定是作者明确列出的 limitation。
