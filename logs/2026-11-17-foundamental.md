## 1.正则化的目的是为了防止数据过拟合
    L1正则化：权重的绝对值相加；L2正则化：权重的平方和
    从几何的角度理解，L1正则化容易得到稀疏矩阵，因为假设只有两个参数，其余地方gradient的绝对值为1，L2正则化不容易。
## 2.MoE：针对于FFN层
    DeepSeekMoE：top-k->shared experts（不管什么token都必须经过某个expert）&fine-grained experts（将n个专家细化成2n个专家而总参数保持不变）
    routing的方式：token choice&expert choice。前者容易出现专家负载均衡问题，后者可能出现token遗漏问题（这个token直接不经过FFN层和原始输入保持一致）且在自回归推理时很难起作用。
    
