---
title: Deep Interest Evolution Network for Click-Through Rate Prediction
date: 2024-10-10 14:00:00 +0800
categories: [Paper, Ad Click Prediction]
tags: [dien]
render_with_liquid: false
---

《Deep Interest Evolution Network for Click-Through Rate Prediction》（简称 **DIEN**）是**阿里巴巴**团队在 2019 年（AAAI 会议）发表的重磅论文。

如果说 DIN 解决了“用户兴趣的多样性”问题，那么 **DIEN 则是为了解决“用户兴趣的动态演化”问题**。它是对 DIN 的直接继承和升华，重点在于捕捉**时间**带来的变化。


---

### 1. 核心动机：兴趣是会“漂移”的

DIN 模型虽然引入了 Attention 来捕捉相关性，但它有一个缺陷：它把用户的历史行为看作一个**无序的集合**（Set），忽略了行为发生的**先后顺序**。

**现实情况是：**

* **兴趣漂移**：用户上个月喜欢穿卫衣，这个月可能开始看衬衫了（时尚趋势改变）。
* **兴趣依赖**：用户买了手机，接着买手机壳，然后买蓝牙耳机（具有因果关联的序列）。

**DIEN 的核心思想：**
用户的**当前兴趣**是由**过去兴趣**逐步演化而来的。我们需要建模这种“演化过程”，并且这种演化是相对于目标商品（Target Item）而言的。

---

### 2. 模型架构：两步走的策略

DIEN 的网络结构非常清晰，主要包含两个核心模块，像接力棒一样工作：

#### A. 兴趣提取层 (Interest Extractor Layer)

这一层负责从原始的行为序列中提取出**潜在的兴趣状态**。

* **基础结构**：使用了 **GRU（门控循环单元）** 来处理序列数据。
* **关键创新：辅助损失 (Auxiliary Loss)**
* **痛点**：通常的 CTR 模型只在最后输出层有一个 Loss（点击/不点击）。对于长序列来说，梯度的反向传播很难有效地指导每一步的 GRU 状态学习（梯度消失问题），导致中间的隐层状态（Hidden State）可能并不能真实代表当时的“兴趣”。
* **解法**：DIEN 在每一个时间步都引入了一个监督信号。它强制要求第  步的隐层状态  能够预测出第  步的真实行为。
* **效果**：这就像老师不只期末考试（Final Loss），还在每节课后都要测验（Auxiliary Loss），确保模型时刻都在通过历史行为捕捉真实的兴趣。



#### B. 兴趣演化层 (Interest Evolving Layer)

提取出兴趣序列后，最大的挑战来了：用户的兴趣是多样的，**只有与当前目标广告相关的兴趣演化才是重要的**。

* **基础结构**：再次使用 GRU 建模演化。
* **关键创新：AUGRU (Attention Update GRU)**
* **结合点**：将 DIN 的 **Attention 机制** 嵌入到 **GRU 的更新门（Update Gate）** 中。
* **原理**：
* 计算历史行为与目标广告的 Attention 分数。
* 如果 Attention 分数高（相关性强），GRU 的更新门打开，让这个兴趣信息更多地传递下去，更新当前的演化状态。
* 如果 Attention 分数低（比如买手机时看过的“洗面奶”行为），GRU 就会忽略这个输入，保持之前的状态不变。


* **公式直觉**：



其中更新门  被 Attention 分数加权调整了。



---

### 3. DIEN 与 DIN 的本质区别

| 特性 | DIN (Deep Interest Network) | DIEN (Deep Interest Evolution Network) |
| --- | --- | --- |
| **看待历史行为** | 视作**无序集合** (Set) | 视作**有序序列** (Sequence) |
| **核心机制** | Attention + Sum Pooling | GRU + Auxiliary Loss + AUGRU |
| **捕捉能力** | 捕捉**不同兴趣的权重** (Diversity) | 捕捉**特定兴趣随时间的演变** (Evolution) |
| **适用场景** | 兴趣相对稳定，主要看匹配度 | 兴趣变化快，或购买行为有明显先后依赖 |

---

### 4. 论文的贡献总结

1. **辅助损失 (Auxiliary Loss)**：这是一个通用的 Trick，极大地增强了序列模型（RNN/GRU/LSTM）在处理长序列时的表达能力，确保隐层状态具有语义含义。
2. **AUGRU**：成功地将注意力机制融合进了循环神经网络的内部结构中，克服了传统 RNN 容易受序列中“噪声”（无关行为）干扰的问题。

---

### 总结

DIEN 是推荐系统从“特征工程时代”迈向“序列建模时代”的重要一步。它证明了：**不仅要关注用户买过什么，还要关注用户购买行为背后的时间逻辑和演变趋势。**

在 DIEN 之后，工业界的推荐模型基本确立了 **Embedding -> Sequence Modeling (Transformer/GRU) -> Attention -> MLP** 的标准范式。
