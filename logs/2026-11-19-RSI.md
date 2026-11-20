## LLM-centric learning

1. Model offline pretraining 静态数据训练模型
2. Model online application 后训练 微调之类的
3. multi-agent orchestration: agent topology；组织结构（中心化，去中心化，多层次）；通信方式（自然语言，结构化语言，通信协议）
4. multi-agent self-improving

## self-evolving process

组成部分：

1. system inputs:优化维度：task-level&instance-level
2. agent system：单agent & 多agent
3. environment：a.benchmark或者real world setting; b.proxy metrics
4. optimiser: search space(system有哪些地方可以优化) & optimisation algorithm

## 有关optimiser的具体优化手段

### single agent优化：

#### 1.behavior优化：

- a.training: SFT, RL;
- b.test-time（因为用API）:1.feedback-based(model as verifier;不只看结果还看过程是否正确)；2.search-based允许不同推理路径并行，再通过投票、评分或搜索选出更好的结果。（CoT-SC,Beam Search,Tree/Forest/Graph/Buffer-of-Thought等）

| 方法 | 通俗理解 |
|---|---|
| **CoT-SC / Self-Consistency** | 独立解多次，推理过程可以不同，最后按答案投票 |
| **DBS / Beam Search** | 每推理一步，都扩展若干候选，再根据评分保留最有希望的几条 |
| **Tree-of-Thoughts** | 把思路展开成树：一个步骤有多个后续选择，可以评估、剪枝、回退 |
| **Forest-of-Thought** | 同时维护多棵思路树，再协调、筛选结果 |
| **Graph-of-Thoughts** | 思路可以分支，也可以合并、交叉利用，不必局限于树 |
| **Buffer-of-Thoughts** | 从记忆缓冲区调用可复用的解题模板，针对当前问题展开 |

#### 2.prompt优化：

##### a.直接编辑prompt（插入删除重写甚至强化学习。。）

| 方法 | 做法 |
|---|---|
| **GRIPS** | 把指令拆成短语，通过删除、交换、改写、添加来搜索 |
| **Plum** | 加入模拟退火、变异、交叉等搜索策略，避免只围绕当前最优版本打转 |
| **TEMPERA** | 用强化学习训练一个“编辑策略”，根据当前问题决定怎么编辑 prompt |

##### b.基于已有prompt生成新的:

| 方法 | 核心做法 |
|---|---|
| **OPRO** | 把历史 prompt 及其得分提供给 LLM，让它提出更好的版本 |
| **StraGo** | 从成功和失败案例中总结经验，指导生成 |
| **PromptAgent** | 用 MCTS 蒙特卡洛树搜索 搜索不同的 prompt 修改方向 |
| **MIPRO** | 搜索指令与示例的组合 |
| **RL 方法** | 进一步训练“生成或修改 prompt 的模型”，让它更擅长优化 |

##### c.文本梯度(用自然语言描述问题和修改方向):

| 方法 | 重点 |
|---|---|
| **ProTeGi** | 根据错误案例批评当前 prompt，生成修改候选，再通过搜索和评测筛选 |
| **TextGrad** | 把最终错误，沿执行流程向前追溯，给可修改的部分提意见。优化 prompt、代码等文本变量 |
| **Agent symbolic learning** | 把 Agent 的提示词、工具和工作流视为可优化的组件，做类似“反向传播”的反馈与更新 |

##### d.evolutionary prompt optimization:

把一批 prompt 当成一个“种群”，通过变异、交叉、筛选，逐代生成效果更好的prompt。

EvoPrompt把遗传算法（GA）、差分进化（DE）的思路引入 prompt 搜索。

Promptbreeder涉及两层 prompt：

| 类型 | 作用 | 示例 |
|---|---|---|
| **Task prompt** | 指导模型完成任务 | “先分析约束，再编写代码” |
| **Mutation prompt** | 指导模型如何修改 task prompt | “根据失败案例，为这条指令增加遗漏的检查步骤” |

#### 3.memory优化：

##### a.short-memory

| 方法 | 具体做法 | 解决的问题 |
|---|---|---|
| **递归摘要**（Wang et al., 2025d） | 把旧摘要和新内容合并，持续更新摘要 | 对话越来越长，无法保留全部历史 |
| **MemoChat** | 从历史对话中整理出需要记住的信息，在后续回复时使用 | 前后回复不一致，忘记用户说过的事情 |
| **COMEDY / ReadAgent** | 提取或压缩对话、文档内容，再提供给模型 | 内容太长，超出上下文窗口 |
| **MoT** | 保存并检索之前生成的推理过程，供当前问题参考 | 相似问题需要重复推理 |
| **StructRAG** | 将信息组织成结构化形式，按需检索，辅助分步推理 | 信息分散，难以串联多个线索 |
| **MemoryBank** | 分层总结历史事件，根据时间和相关性保留或遗忘信息 | 历史记录不断积累，重要信息容易被淹没 |
| **Reflexion** | 根据任务结果总结错误和经验，在下一次尝试时参考 | 反复犯同样的错误 |

##### b.long-memory

| 方法 | 具体做法 | 解决的问题 |
|---|---|---|
| **RAG** | 从外部知识库检索相关内容，提供给模型参考 | 所需知识不在当前上下文里 |
| **EWE** | 用额外的工作记忆暂存检索内容的内部表示，在生成过程中使用 | 如何在生成答案时利用检索到的信息 |
| **A-MEM** | 为记忆建立索引和关联；加入新记忆时，也更新相关旧记忆 | 记忆不断增加，需要维护它们之间的关系 |
| **基于 RAG 的遗忘方法**（Wang et al., 2024e） | 修改外部知识库，让某些内容不再被检索，无须修改模型参数 | 让系统停止使用指定知识 |
| **MemGPT / NeuroCache / G-Memory / AWESOME** | 各自采用不同的记忆管理方式，保存并复用历史信息 | 让过去的信息和经验用于后续任务 |
| **Mem0** | 从交互中持续提取值得保存的信息，在需要时检索 | 减少重复提供背景信息 |
| **HippoRAG** | 用知识图谱连接信息，建立受海马体机制启发的检索方式 | 找到分散但彼此相关的知识 |
| **GraphReader / Mem0ᵍ** | 用图结构记录信息之间的关系，并据此检索 | 单独检索文本片段容易遗漏关联 |
| **ChatDB** | 将信息放在结构化数据库中，通过 SQL 查询和操作 | 需要准确查找、统计或更新记录 |
| **神经符号记忆方法**（Wang et al., 2024f） | 同时用自然语言和符号形式保存事实、规则 | 支持规则推理，并追踪所用的信息 |
| **MATTER** | 从多种记忆来源中选择与当前问题相关的片段 | 记忆来源多，需要筛选有用内容 |
| **AWM** | 从任务经历中提取可复用的工作流程，支持离线构建和在线更新 | 避免每次遇到类似任务都重新摸索步骤 |
| **MyAgent** | 回忆历史信息时考虑时间关系 | 分清事情发生的先后和随时间产生的变化 |
| **MemoryBank** | 根据遗忘机制管理记忆，被再次使用的信息得到强化 | 长期保留重要信息，逐渐淡化不常用的信息 |
| **MEM1** | 用强化学习训练记忆管理能力，整合新信息、舍弃无关内容 | 长任务中历史越来越多，需要保留精简且有用的状态 |
| **MrSteve** | 按“发生了什么、在哪里、什么时候”组织经历 | 为后续规划和行动提供历史依据 |
| **MIRIX** | 设置六种记忆类型，由多个 Agent 协作管理和检索 | 长任务需要同时使用不同类型的记忆 |
| **Agent KB** | 通过共享知识库和教师—学生两阶段检索，复用策略与执行经验 | 让不同 Agent、不同领域的任务共享经验 |

#### 4.工具：

角度：优化模型和工具的交互或者优化工具本身

##### a.training-based

监督微调（SFT）：先收集工具使用示例，再让模型学习。 一条完整示例通常包括：用户问题 → 中间步骤 → 工具调用及返回结果 → 最终回答。

| 方法 | 具体做法 |
|---|---|
| **ToolLLM / GPT4Tools** | 用更强的模型生成任务指令和对应的工具调用过程，作为训练数据 |
| **STE** | 通过模拟试错，让模型尝试使用工具，收集训练示例 |
| **TOOLEVO** | 用 MCTS 探索不同的工具调用路径，收集质量更高的示例 |
| **T3-Agent** | 生成并验证多模态工具使用示例，用来训练视觉语言模型 |
| **Magnet** | 围绕工具生成问题和可执行的调用序列，用图组织调用之间的依赖，构造多轮训练数据 |
| **BUTTON** | 先把简单任务组合成复杂任务，再让多个 Agent 模拟用户、助手和工具，生成完整交互记录 |
| **APIGen-MT** | 先生成工具调用序列，再模拟用户与 Agent 的对话，把序列扩展成多轮交互 |
| **Confucius** | 按从简单到复杂的顺序安排训练任务，让模型逐步学习更复杂的工具使用 |
| **Gorilla** | 在训练流程中加入 API 文档检索，让模型根据查到的文档调用工具，适应工具变化 |

强化学习（RL）：让模型实际尝试调用工具，再根据结果获得奖励、调整策略。

| 方法 | 具体做法 |
|---|---|
| **ReTool / Nemotron-Research-Tool-N1** | 在交互环境中训练，让模型根据执行反馈学习工具使用，减少对完整人工示例的依赖 |
| **Tool-Star** | 结合工具使用数据合成和两阶段训练，提升模型自主配合使用多个工具的能力 |
| **SPORT** | 面向多模态任务，让模型自行生成任务、探索并验证工具使用，通过逐步偏好优化进行训练 |
| **ARPO** | 根据模型输出的不确定性调整采样，并细分各步骤对结果的贡献，改进长推理和多轮工具调用的训练 |
| **其他奖励与数据方法** | 改进奖励设计，或生成、筛选训练数据，提高训练稳定性和效率 |

##### b.inference-based

改进工具说明：让模型看懂工具做什么、什么时候用、参数怎么填。

| 方法 | 具体做法 |
|---|---|
| **EASYTOOL** | 把格式各异、内容冗长的工具文档整理成统一、简洁的说明 |
| **DRAFT / PLAY2PROMPT** | 让 Agent 尝试使用工具，根据反馈反复修改工具说明 |
| **联合优化方法**（Wu et al., 2025a） | 同时调整 Agent 的任务指令和工具描述，让两者配合得更好 |

改进调用过程：让 Agent 更合理地选择工具、安排步骤，并在失败后调整。

| 方法 | 具体做法 |
|---|---|
| **ReAct** | 交替进行推理、调用工具和查看结果，根据返回信息决定下一步 |
| **ToolLLM** | 用深度优先树搜索探索调用路径；遇到失败时回退到之前的有效状态，尝试其他分支 |
| **ToolChain** | 估计各条调用路径后续的成本，提前排除不值得继续的分支，减少无效尝试 |
| **Tool-Planner** | 把功能相近的工具分组，再通过树形规划选择工具；需要时换用同组工具 |
| **MCP-Zero** | 让 Agent 判断自己缺少什么能力，再按需请求相应工具 |

##### c.tool

| 方法                 | 核心思想         | 具体实现                                                   |
| ------------------ | ------------ | ------------------------------------------------------ |
| **CREATOR**        | 自动创建新工具      | 为新任务生成工具文档（Tool Documentation）和可执行代码（Executable Code）。 |
| **LATM**           | 通过语言模型创建工具   | 为新任务生成所需的工具及其实现。                                       |
| **CRAFT**          | 复用已有代码创建工具   | 从历史任务中提取可复用代码片段，组合成适用于新场景的工具。                          |
| **AgentOptimiser** | 将工具视为可学习参数   | 把工具和函数视为 Learnable Weights，通过 LLM-based Updates 迭代优化。  |
| **Alita**          | 多组件程序组织工具    | 将工具创建扩展为 Multi-Component Program（MCP）格式，提高可复用性和环境管理能力。 |
| **CLOVA**          | 基于人类反馈优化视觉工具 | 采用 Inference、Reflection、Learning 三阶段闭环，根据人类反馈持续调整视觉工具。 |

### 多agent优化：

我把表格里的语言统一调整得更自然一些：尽量使用直白的中文描述具体做法，保留必要的英文术语，不再堆叠诸如“范式迁移”“概率化架构采样”“语义有效区域”等不容易理解的表达。

#### Manually Designed Multi-Agent Systems

| 类型 | 核心思想 | 具体做法 |
|---|---|---|
| Parallel Workflows | 多个 Agent 并行完成任务 | 多个 Agent 独立生成答案，再通过多数投票选出最终结果。也可以使用多层投票、动态任务图和异步执行来提高效率。 |
| Hierarchical Workflows | 按层级分配任务 | 将 Agent 组织成多层结构，由上层负责任务分解，下层负责具体子任务，也可以按照顺序组成执行 Pipeline。 |
| Multi-Agent Debate | 通过辩论改进答案 | 多个 Agent 从不同角度讨论、质疑和修正彼此的推理结果，最后由仲裁机制确定答案。 |
| Confidence-Gated Debate | 根据模型置信度决定是否辩论 | 当单个模型对答案不够确定时，才启动多 Agent Debate，避免不必要的计算开销。 |

#### Self-Evolving Multi-Agent Systems

##### Multi-Agent Prompt Optimisation

| 方法 | 核心思想 | 具体做法 |
|---|---|---|
| DSPy Assertions | 根据执行反馈自动修正输出 | 使用 Assertion 检查 Pipeline 中间结果。如果结果不符合预设条件，就通过 Backtracking 和反馈让 LLM 重新生成。 |
| AutoAgents | 自动优化 Agent 团队的配置 | 通过专门的 Meta-Agent 进行讨论，调整不同 Agent 的角色分工和执行计划。 |

##### Topology Optimisation

###### A. Code-level Workflows

| 方法 | 核心思想 | 具体做法 |
|---|---|---|
| AutoFlow | 搜索更好的工作流程序 | 将自然语言程序作为搜索对象，通过强化学习训练模型生成不同的工作流。 |
| AFlow | 使用代码算子构建工作流 | 将可复用的 Typed Operators 组合成 Code Graph，并使用 MCTS 和 LLM 引导搜索更好的组合方式。 |
| ScoreFlow | 通过梯度优化工作流生成器 | 将代码表示映射到连续空间，利用 Gradient-based Optimisation 和 Score-DPO 改进工作流生成效果。 |
| MAS-GPT | 直接生成完整的多 Agent 系统 | 使用经过一致性设计的数据集进行 SFT，使模型能够一次性生成完整、可执行的 MAS 代码。 |

###### B. Communication-graph Topologies

*静态通信图优化*

| 方法 | 核心思想 | 具体做法 |
|---|---|---|
| GPTSwarm | 优化 Agent 之间的通信连接 | 将 Agent 之间是否建立连接表示为概率，并通过强化学习学习更合适的通信结构。 |
| DynaSwarm | 根据任务选择不同的通信结构 | 维护多种通信图，通过 Actor-Critic（A2C）训练图选择器，为不同任务选择合适的结构。 |
| G-Designer | 根据任务生成通信图 | 使用 Variational Graph Autoencoder 生成 Agent 之间的通信连接，并根据任务调整图的复杂程度，在效果和 Token Cost 之间取得平衡。 |
| MermaidFlow | 在满足约束的条件下搜索通信图 | 使用带类型的图表示 Agent 之间的关系，并通过静态检查和进化搜索，避免生成不符合要求的通信结构。 |

*动态通信图优化*

| 方法 | 核心思想 | 具体做法 |
|---|---|---|
| DyLAN | 自动筛选有价值的 Agent | 使用 LLM 对 Agent 的重要性进行评分，逐步移除贡献较小的 Agent，并在必要时提前结束执行。 |
| Captain Agent | 根据子任务调整 Agent 团队 | 针对不同子任务检索和筛选 Agent 及工具，必要时创建新的 Agent 或工具，并通过讨论和反思调整团队组成。 |
| Flow | 根据执行情况调整工作流 | 先根据任务之间的并行关系和依赖关系建立初始工作流，再根据执行情况调整任务分配和执行顺序。 |

*通信图剪枝*

| 方法 | 核心思想 | 具体做法 |
|---|---|---|
| AgentPrune | 删除不必要的通信 | 同时检查单次对话内部和不同轮次对话之间的通信连接，使用可训练的 Graph Mask 删除冗余连接，减少 Token 消耗。 |
| AGP | 同时优化 Agent 数量和通信连接 | 通过两阶段训练，分别调整 Agent 数量和 Agent 之间的通信连接，使团队规模和通信结构适应不同任务。 |
| G-Safeguard | 通过删除通信连接提高安全性 | 使用 GNN 识别存在风险的节点，再通过规则切断这些节点向外发送信息的连接，以降低攻击风险。 |
| NetSafe | 分析多 Agent 通信结构中的安全风险 | 总结通信拓扑可能带来的安全问题，并提出基于图结构的风险检测和干预方法。 |

##### Unified Optimisation

###### A. Code-based Approaches

| 方法 | 核心思想 | 具体做法 |
|---|---|---|
| ADAS | 用代码统一表示 Prompt 和工作流 | 将 Agent 的 Prompt、工作流和工具调用都表示为 Python 代码，通过 Meta Agent Search 不断生成和评估新的 Agent 系统。 |
| FlowReasoner | 针对每个 Query 生成多 Agent 系统 | 不再为一类任务设计固定的 MAS，而是针对每个 Query 单独生成系统，并利用 GRPO 和外部执行反馈改进生成过程。 |

###### B. Search-based Approaches

| 方法 | 核心思想 | 具体做法 |
|---|---|---|
| EvoAgent | 通过进化算法优化 Agent 配置 | 将 Agent 的角色、技能和 Prompt 作为优化对象，通过 Mutation、Crossover 和 Selection 不断生成和筛选新的配置。 |
| EvoFlow | 通过进化算法优化工作流 | 在由算子组成的工作流图上进行搜索，利用预定义的算子库限制搜索范围，同时考虑任务效果和执行成本。 |
| MASS | 分阶段优化 Prompt 和工作流 | 先分别优化每个 Agent 的 Prompt，再搜索合适的工作流结构，最后针对选定的结构进一步调整 Prompt。 |
| DebFlow | 通过多 Agent 辩论优化工作流 | 让多个 Agent 参与工作流优化，并利用执行失败后的反思来指导下一轮搜索。 |
| MAS-ZERO | 在推理阶段直接搜索更好的 Agent 结构 | 不需要额外训练或梯度更新，而是根据任务能否成功完成，不断调整 Agent 团队和任务分解方式。 |

###### C. Learning-based Approaches

| 方法 | 核心思想 | 具体做法 |
|---|---|---|
| MaAS | 学习不同任务对应的 Agent 架构 | 不再只优化一个固定架构，而是训练一个 Controller，根据具体 Query 从多种可能的 MAS 架构中选择合适的结构。 |
| ANN | 用类似神经网络的结构组织 Agent | 将每一层设计为一组专门负责特定任务的 Agent，通过前向任务分解和反向文本梯度优化，同时调整 Agent 角色、Prompt 和层间连接。 |

##### LLM Backbone Optimisation

###### A. Reasoning-oriented Optimisation

| 方法 | 核心思想 | 具体做法 |
|---|---|---|
| Multi-Agent Finetuning | 利用多 Agent 协作数据提升模型推理能力 | 从多 Agent 辩论中收集高质量的协作过程，再通过 SFT 训练模型，使模型学会不同角色的推理方式和协作能力。 |
| Sirius | 通过 Self-play 训练模型 | 让 Agent 自己生成协作轨迹，通过 SFT 学习这些轨迹，并将经过自我纠错的错误轨迹也加入训练数据。 |
| MALT | 通过 Self-play 和偏好学习提升协作能力 | 收集 Agent 自我博弈产生的协作轨迹，使用 DPO 学习正面和负面的训练样本。 |
| MaPoRL | 通过强化学习鼓励 Agent 协作 | 设计与具体任务相关的奖励机制，鼓励 Agent 进行更多有效的信息交流和合作。 |
| MARFT | 将传统多 Agent 强化学习引入 LLM 训练 | 建立传统 Multi-Agent Reinforcement Learning 与 LLM-based Multi-Agent Reinforcement Tuning 之间的联系。 |
| MARTI | 提供更灵活的多 Agent 强化微调方式 | 允许研究者自定义 Agent 结构和奖励函数，再通过强化学习优化模型的协作能力。 |

###### B. Collaboration-oriented Optimisation

| 方法 | 核心思想 | 具体做法 |
|---|---|---|
| COPPER | 改进 Agent 的反思能力 | 使用 PPO 训练一个共享的 Reflector，为不同角色的 Agent 生成更有针对性的反思内容，帮助它们改进协作过程。 |
| OPTIMA | 提高 Agent 之间的通信效率 | 关注通信所需的 Token 数量和信息表达质量，通过 SFT、DPO 等方法减少通信开销，同时尽量保持协作效果。 |

其实感觉优化使用的方法都差不多，重点是具体场景和约束下实现完整闭环。

## evaluation

### 1.层出无穷的benchmark

### 2.LLM-as-judge/agent-as-judge:后者有很高的复杂度

### 3.对于MASE，希望safety evaluation是动态的，而现在大部分都是snapshot。

## 后续发展方向
  1.Simulated Environments for Fully Autonomous Self-Evolution (MASE). 
  2.Advancing Tool Use and Creation (MAO→MASE).
  3.Real-World Evaluation and Benchmarking (Cross-stage).
  4.Effectiveness–Efficiency Trade-offs in MAS Optimisation (MAO). 
  5.Domain-Aware Evolution for Scientific and Specialised Applications (MASE). 
  
