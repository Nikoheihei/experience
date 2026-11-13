** 1.EPD分离

** 2.Block KV（CSA）& SWA KV

** 3.EPLB解决 MoE 模型中不同 GPU 忙闲不均问题的手段：重排（把热点专家切小）+冗余（不同gpu存储相同的内容）。实测收益几乎全部来自重排。

** 4.DPLB：

** 5.Prefill分桶: 按照上下文长度分成4个桶。分桶后具体分配任务给worker。

** 6.decode平衡请求数和KV总长度。
