# CS336 lecture3 architecture:

## 1.post-Norm&pre-Norm; pre-Norm可以keep residual stream clean。

## 2.LayerNorm&RMSNorm：LayerNorm将向量各维度的均值平移到 0，而RMSNorm只会缩放尺寸
###     dropping the bias term on both linear and RMS norm is okay

## 3. Activation
### GLU variant在控制总参数数量的前提下，一般效果会比Non-GLU variant好

## 4.serial & parallel layers
### 现在基本用的是serial layers

## 5. 我们需要的是relative position embedding---RoPE

## 6. hyperparameter
### 1.一般来说会把输入向量的尺度拓展到原来的4倍左右。例外是：1.GLU（总参数参数需要小2/3，也就是dff/dmodel差不多在8/3也就是2.6左右）2.T5用了16倍，理由是能提高硬件利用率。不过有实验证明dff/dmodel的比率在2-4之间损失曲线保持平缓。
### 2. 通常会把隐藏层纬度按head数量划分，这样在multihead计算的时候，乘以head数量正好等于隐藏纬度。\[
d_{head}=\frac{d_{model}}{h}
\]
### 3. 一般来说 dmodel/dlayer在100左右。如果layer太多很难并行，但是如果很宽的话，gpu很容易并行。
### 不过对hyperparameter的数值还是比较宽容的，更重要的是系统的资源利用率。
## 7. regularization正则化-为了防止过拟合
### 方法：dropout随机关闭一些神经元&weight decay限制模型参数的大小，避免权重变得过大，一般会和学习率进行调整。
## 8. 模型不够稳定时注意排查
### 1.softmax函数：因为指数很容易爆炸或者过于小。softmax函数出现在输出端（解决方法：调整Z（x） 因为不会改变归一化后的概率）和注意力计算（QKNorm）的时候。
## 9. MQA&GQA
## 10.全局注意力和局部注意力（带状，滑动窗口），或者两者结合使用。
