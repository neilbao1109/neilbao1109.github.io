---
title: Attention Is All You Need 划重点
date: 2025-11-09 10:00:00 +0800
categories: [Paper, LLM]
tags: [transformer]
render_with_liquid: false
math: true
---

这篇由 Google Brain 团队在 2017 年发表的论文不仅是一个模型架构的创新，它本质上是**对“序列建模”范式的重构**。

---

## 1. 缩放点积注意力（Scaled Dot-Product Attention）的底层逻辑

论文中提出的注意力公式如下：


### 为什么要除以 $\sqrt{d_k}$ ？

这是该论文最重要的工程细节之一。

* **数学解释**：当维度 $d_k$  很大时，$Q$和$K$的点积结果会变得非常大。这会导致经过 `softmax` 后的梯度变得极小（进入饱和区），导致模型难以训练。
* **物理意义**：除以 $\sqrt{d_k}$ 相当于对点积结果进行“方差归一化”，将分布推回梯度敏感的区域，确保训练的稳定性。

---

## 2. 多头注意力（Multi-Head Attention）：并行表征空间

论文作者认为，只进行一次注意力计算（单头）会限制模型捕捉信息的能力。

* **投影到子空间**：通过 $h$ 个不同的线性映射，模型将 $Q,K,V$ 投影到不同的低维子空间。
* **功能分工**：在实际观察中，不同的“头”会自发分工。有的头关注当前词的**语法关系**（如动词找宾语），有的头关注**代词指代**（如“他”对应到“张三”），有的头关注**局部上下文**。
* **合并信息**：最后将所有头的输出拼接（Concat）并再次线性变换。这让模型能同时在多个维度“理解”同一个词。

---

## 3. 位置编码（Positional Encoding）：赋予“时间感”

由于 Transformer 彻底抛弃了循环结构（RNN），它对词序是**完全不敏感**的。如果没有位置编码，“我爱你”和“你爱我”在 Transformer 眼里是一模一样的。

作者采用了正弦和余弦函数的组合：

$$
PE_{(pos,\, 2i)} = \sin\!\left( \frac{pos}{10000^{\,2i/d_{model}}} \right)
$$

$$
PE_{(pos,\, 2i+1)} = \cos\!\left( \frac{pos}{10000^{\,2i/d_{model}}} \right)
$$


### 为什么选这个函数？

1. **相对位置的线性表达**：对于任何固定的偏移量 $k, PE_{pos}$， 可以表示为 $PE_{pos}$ 的线性函数。这意味着模型可以很容易地学习到词与词之间的**相对距离**。
2. **无限长度外推**：理论上这种编码可以处理比训练时更长的序列，因为它具有周期性。

---

## 4. 残差连接与层归一化（Add & Norm）

每一个子层（Attention 或 FFN）之后都紧跟一个 `LayerNorm(x + Sublayer(x))`。

* **残差连接（Residual Connection）**：解决了深层网络中的**梯度消失**问题，让底层信息能直接流向高层。
* **层归一化（Layer Normalization）**：与图像处理中常用的 BatchNorm 不同，LayerNorm 在序列维度内进行归一化，非常适合处理变长的 NLP 数据，能显著加快收敛。

---

## 5. 复杂度与计算效率分析

论文对比了 Transformer 与 RNN、CNN 的计算复杂度：

| Layer Type | Complexity per Layer | Sequential Operations | Maximum Path Length |
| --- | --- | --- | --- |
| **Self-Attention** | O(n2⋅d) | O(1) | O(1) |
| **Recurrent** |O(n⋅d2) | O(n) | O(n) |
| **Convolutional** | O(k⋅n⋅d2) | O(1) | O(logk​(n)) |

* **并行性**：RNN 的串行操作是 $O\left(n\right)$，而 Transformer是 $O\left(1\right)$  ，这决定了它能跑在大规模 GPU 集群上。
* **代价**：代价是  的自注意力计算。这也是为什么现在的长文本模型（如处理 100k Token）需要各种优化（如 FlashAttention、Sparse Attention）来减轻这个平方级的负担。

---

## 6. 深度洞察：归纳偏置（Inductive Bias）的转换

这是理解这篇论文最深刻的角度：

* **RNN 的偏置**是“时间局部性”，认为距离近的词关系大。
* **CNN 的偏置**是“空间局部性”，认为滑动窗口内的词关系大。
* **Transformer 的偏置**几乎为零。它通过全连接的注意力，假设**任何两个词之间都可能有直接联系**。

**结论**：当数据量较小时，RNN/CNN 更好用；但当数据量极大时（海量互联网文本），Transformer 这种“不设限”的架构能从数据中自动学习到远比人类设计的规则更复杂的关联。
