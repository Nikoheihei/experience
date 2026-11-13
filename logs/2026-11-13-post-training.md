## post-traing:判断模型回答的好或者不好并纠正。
## 1.如何学习answer问题：SFT。
    chat template(给句子打标签从而区分到底是谁讲的); 
    next-token prediction（对于实际的下一个token给出概率）;
    cross-entropy loss to calculate the loss接着反向传播调整权重。
    数据来源：人工标注；现有数据集；其他模型蒸馏

## 2.如何学习人们的偏好：
