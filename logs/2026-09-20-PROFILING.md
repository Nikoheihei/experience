# Profiling 结论与后续方向（2026-09-20）

| 我们已经确定的事 | 对后续方向的直接指引 |
|---|---|
| **8B 在小米 14 的 16 GB 内存上没有足够余量做 speculative profiling。** Target-only 已出现 critical memory pressure / 重启；8B target + Draft、多 batch target、3DMark 组合还会触发低内存、LMK 或前台被系统停止。 | **停止把 8B 用作复杂 speculative、SERENO、GPU overlap 的实验模型。** 8B 只保留为“该部署组合的内存边界”证据；后续统一换成 **3B target + 0.5B Draft**。 |
| **温度不是一个应被简单“卡在 38°C 以下”的无效条件，而是会经由 OPP/DVFS、调度与功耗限制影响性能状态。** | 后续不再用“温度超过某值就丢掉数据”的规则。应记录全过程温度、CPU/GPU/DDR OPP/频率、thermal status，并按相近起始热状态配对。研究问题应是：**频率相近时，thermal state 是否仍解释额外 latency/QoS 差异？** |
| **轻量 FrameProbe 不能作为主前台 workload。** 它有时太轻，后台计算触发系统 boost 后反而更顺。 | 不再用 FrameProbe 证明“LLM 损害前台 QoS”；它仅保留作仪器和 trace sanity check。主 workload 用 **Bilibili 固定 StoryVideo 滑动**。 |
| **Bilibili 在后台 8B 推理下确实出现 QoS 下降。** Target-only 已提高 deadline miss。 | 现象成立。后续应围绕真实 App 的 **App Deadline Missed、长帧、UI/RenderThread 延迟** 做因果定位。Buffer Stuffing 只作为生产—消费节奏指标，不当作 QoS 损伤本身。 |
| **当前最强的资源方向是 memory subsystem。** PMU 还看到 memory stall、membound stall、L3 refill、TLB walk 上升，内存 workload 被显著损伤；CPU/GPU isolated workload 还没测，因为模型没搞好。 | 继续围绕 memory-subsystem contention 设计隔离实验，并补齐 CPU/GPU isolated workload，避免在模型配置未稳定前过早下结论。 |
| **但“已证明 DDR 带宽饱和”不成立。** 目前没有直接 HTP/DDR byte 或 DRAM latency counter。 | 论文里只能写“memory-subsystem contention / memory waiting 增强”，不能写死“DDR bandwidth saturation”。若要再深挖，要补直接 DDR/HTP 指标，或设计能区分“流量增加”和“DRAM 延迟增加”的实验。 |
| **Target round pacing / micro-sleep 有候选收益。** 温度匹配的重复中，2 ms 请求 sleep 通常显著降低 Stuffing，deadline miss 也小幅下降，TPS 代价较小。 | 值得保留为机制线索，但不能写“micro-sleep 已解决 QoS”。下一步应把它和真实 NPU burst、关键帧窗口做时间对齐，证明它为什么有效。 |
| **“LLM 停止后，余热一定继续伤害前台”尚未成立。** 余热状态明显存在，但一次 FrameProbe 和一次 Bilibili 后测都没有稳定 QoS 下降。 | 不要把 thermal residual 当现成论文结论。它仍是一个可证伪问题：固定 OPP/频率后，温度是否还有额外解释力。 |
| **SERENO 中的 CPLM 是 system-wide proxy，不是直接 DDR 真值。** 还未证明 CPLM 和 DDR 的正相关性。 | 后续需要补 CPLM 与直接 DDR/HTP 指标的相关性验证；在证据补齐前，只把 CPLM 作为系统级代理指标使用。 |

