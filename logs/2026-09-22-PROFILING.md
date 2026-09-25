# mzCache 阅读与实验记录（2026-09-22）

## 1. mzCache 论文

| 论文                        | 研究对象                                                                        | 核心方法                                                                                                                 | 主要结论                                                                                             | 未解决处                                                                                                               |
| ------------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| **mzCache, MobiCom 2026** | 手机上的本地 LLM 在用户切去其他 App 后，weights / KV cache 因内存压力被回收、压缩甚至进程被杀，用户回来继续使用时恢复很慢 | 将 weight 按 layer、KV 按 chunk 管理；CPU 从 UFS / 压缩内存恢复，GPU 同时进行 inference；一边 prefill 新 token 一边恢复后续数据，把 restore 尽量隐藏在计算后面 | Android 通用 paging 不理解 LLM memory semantics；利用 Transformer 已知的数据访问顺序，可以明显降低 resume TTFT，并隐藏大量恢复开销 | 论文主要在 **0.6B / 1.2B + CPU/GPU** 环境验证，并假设 restore 与 inference 可以很好 overlap；长 KV、更大模型以及 **NPU inference** 下是否仍成立并不清楚 |

Bandwidth contention 不是 mzCache 的原始问题，而是它采用“restore 和 inference 并发执行”后引出的第二层问题：CPU/UFS restore 和 GPU/NPU inference 都会访问 shared DRAM，如果恢复能始终跑在 inference 前面，restore 就能被隐藏；如果资源竞争导致恢复跟不上，zero-wait assumption 才可能失效。

## 2. 3B QNN：人为 DRAM 竞争

先在 **3B QNN、1024-token Prefill** 上人为制造 memory traffic，确认如果 shared DRAM 真的被压得很重，NPU inference 是否会受到影响。

| 条件                  | Prefill | 相对变化 |
| ------------------- | ------: | ---: |
| Baseline            | ~1.32 s |    — |
| 1 个 memory stressor | ~1.39 s |  +6% |
| 2 个 memory stressor | ~1.81 s | +38% |

结论：重 DRAM 竞争确实可以明显拖慢 3B NPU Prefill，但这种人工 stress 不能直接代表 mzCache 的真实 restore workload。

## 3. 3B QNN：真实 mzCache-style restore

随后改成真实 `O_DIRECT` UFS 读取 + memory materialize，并与 3B QNN Prefill 并发。4 路恢复约达到 **3 GiB/s UFS read** 和 **9 GiB/s memory traffic**。

| 条件         |  Prefill |
| ---------- | -------: |
| Baseline   | ~1.317 s |
| Restore 并发 | ~1.315 s |

结论：普通 mzCache 式“从 UFS 读回再放入内存”几乎完全可以被 3B Prefill 隐藏，因此最开始“3B 就会打破 mzCache overlap assumption”的猜测不成立。

## 4. KV 解压 / 展开式 memory traffic

随后使用 root PMU 增加更接近 KV 解压和 materialization 的内存流量。

| 条件          | Prefill | Memory-backend stall |
| ----------- | ------: | -------------------: |
| Baseline    | 1.325 s |             91.6 M/s |
| 普通 restore  | 1.327 s |            317.9 M/s |
| 6× 不同输出块扩展  | 1.359 s |            417.8 M/s |
| 强 memcpy 干扰 | 1.513 s |            576.4 M/s |

结论：退化不是突然出现的 bandwidth cliff，而是随着 memory pressure 增大，memory-backend stall 连续增加并逐渐暴露到端到端 latency。尤其是普通 restore 已经让 stall 从 91.6 M/s 增加到 317.9 M/s，但 Prefill 几乎不变，说明系统本身有较强的 overlap / latency hiding 能力。

## 5. 真实 KV / Context Sweep

随后按照真实 historical KV 大小恢复，而不是继续使用纯 memcpy stress。

| Context | KV Restore |   相对基线 |
| ------: | ---------: | -----: |
|     256 |      9 MiB | +3.36% |
|     512 |     18 MiB | -0.08% |
|    1024 |     36 MiB | +1.72% |
|    1536 |     54 MiB | -0.24% |
|    1792 |     63 MiB | -3.61% |

没有观察到“Context 越长 → Restore contention 越严重”的稳定趋势。将 historical KV 和新 prompt 大小解耦后，63 MiB restore 约需 **43 ms**，固定短 Prefill 约需 **162 ms**，因此在当前约 2K QNN graph 下，9–63 MiB restore 仍基本可以被隐藏。

## 6. llama.cpp CPU：8K / 16K Historical KV

由于现有 QNN graph 的 KV cache 只有约 1920 token，进一步使用 stock llama.cpp CPU 路径扩大 historical KV，观察 restore 时间与固定短前台计算的关系。

| Historical KV |  State 大小 |  Restore |    孤立 8t |   孤立 16t |
| ------------: | --------: | -------: | -------: | -------: |
|            2K |    72 MiB |  28.9 ms | 195.2 ms | 283.0 ms |
|            4K |   144 MiB |  34.1 ms | 195.2 ms | 283.0 ms |
|            6K |   216 MiB |  60.6 ms | 195.2 ms | 283.0 ms |
|          8176 | 287.6 MiB |  85.6 ms | 195.2 ms | 283.0 ms |
|        16,352 | 575.1 MiB | 153.9 ms | 182.7 ms | 287.3 ms |

8K historical KV 下，restore 约 **86 ms**，仍明显短于 8-token 前台计算的约 **195 ms**，所以依然可以完全隐藏。8K historical KV 后继续生成 8 / 16 token 分别约需 **721 / 1390 ms**，但这是 long-context attention 本身的代价，而不是 restore 失败。

到 16K 时，restore 已增加到 **153.9 ms**，而固定 8-token 前台计算约为 **182.7 ms**，两者已经比较接近。也就是说，16K / 575 MiB historical KV 开始逼近 overlap window，但 restore 仍没有真正超过 foreground computation。

## 7. 当前结论与下一步

| 已验证的问题                                          | 当前结论                                     |
| ----------------------------------------------- | ---------------------------------------- |
| 3B 是否天然打破 mzCache overlap                       | **否**，普通真实 restore 基本可以完全隐藏              |
| 强 DRAM stress 是否会拖慢 NPU Prefill                 | **会**，但人工 memcpy stress 不能直接代表真实 restore |
| 真实 restore 是否存在明显 bandwidth knee                | **暂未发现**，目前更像连续增加的 memory stall          |
| 9–63 MiB KV restore 是否随 context 增大而系统性恶化        | **否**                                    |
| 8K / 288 MiB historical KV 是否让 restore 成为瓶颈     | **否**                                    |
| 16K / 575 MiB historical KV 是否越过 overlap window | **还没有，但已经接近**                            |

目前最值得继续验证的 setting 变化不是继续单纯扩大模型，而是将 mzCache 原本的 **CPU restore + GPU inference** 换成 **CPU restore + Qualcomm NPU inference**，看 NPU 特殊的 memory arbitration 是否会改变 restore / inference 的 overlap 关系。

需要注意，**“NPU 几乎不变慢，而 CPU restore 被抢带宽”本身不能作为 novelty**，因为 SERENO 已经观察过类似的 NPU asymmetric interference。真正要验证的是：在 mzCache 这种 producer–consumer pipeline 中，NPU 是否会让 CPU restoration 落后，从而缩小甚至打破原本的 zero-wait overlap window。如果出现这种现象，再继续研究它是否会导致新的 resume bottleneck 或需要不同于 GPU 路径的恢复策略。
