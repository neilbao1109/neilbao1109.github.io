---
title: Attention Is All You Need（Transformer）深度笔记
date: 2025-11-08 10:00:00 +0800
categories: [Paper, LLM]
tags: [transformer]
render_with_liquid: false
math: true
---

- 论文：Ashish Vaswani et al., *Attention Is All You Need*, NeurIPS 2017（arXiv:1706.03762v5）
- 任务：序列到序列（重点：机器翻译 EN-DE / EN-FR），并展示可迁移到句法分析
- 核心主张：完全移除 RNN/CNN 的序列建模依赖，仅用注意力（Self-Attention + Cross-Attention）+ 前馈网络完成序列转导

---

## 1. 一句话总结
Transformer 用“全局自注意力”替代 RNN 的时间展开与 CNN 的局部卷积堆叠，使得：

- 训练可对序列维度并行（不再按时间步串行）
- 任意两位置的信息交互路径长度降为 $O(1)$（单层内即可互相看到）
- 通过多头注意力（Multi-Head）抵消“加权平均导致分辨率下降”的问题

---

## 2. 论文解决了什么痛点

### 2.1 RNN 的痛点
- **计算依赖链长**：隐藏状态 $h_t$ 依赖 $h_{t-1}$，训练无法在单样本内并行；序列越长越慢
- **长程依赖难学**：梯度路径长（最大路径 $O(n)$），优化更难

### 2.2 CNN（ConvS2S/ByteNet 等）的痛点
- 可并行，但要覆盖远距离依赖，需要堆叠多层（路径长度 $O(n/k)$ 或 $O(\log n)$）
- 局部感受野扩展需要增加层数/空洞卷积，结构更复杂

### 2.3 Transformer 的取舍
- 自注意力每层复杂度约 $O(n^2 d)$（对长序列昂贵）
- 但在翻译常见的子词序列长度范围内，$n$ 往往小于表示维度 $d$，且并行效率显著更高

---

## 3. 模型总览：Encoder-Decoder + 堆叠层
标准 encoder-decoder 框架不变：

- Encoder：输入序列 $x_{1:n}$ → 表示序列 $z_{1:n}$
- Decoder：自回归生成 $y_{1:m}$，并可对 $z$ 做 cross-attention

论文默认（Base）配置：$N=6$ 层 encoder + $N=6$ 层 decoder，模型维度 $d_{model}=512$。

### 3.1 Encoder Layer 结构
每层两块：
1) Multi-Head Self-Attention
2) Position-wise FFN（逐位置两层 MLP）

每个子层外都带：Residual + LayerNorm：
$$\text{LayerNorm}(x + \text{Sublayer}(x))$$

### 3.2 Decoder Layer 结构
每层三块：
1) Masked Multi-Head Self-Attention（遮住未来位置）
2) Encoder-Decoder Attention（cross-attention：Q 来自 decoder，K/V 来自 encoder 输出）
3) Position-wise FFN

同样每个子层都用 Residual + LayerNorm。

---

## 4. 注意力机制：Scaled Dot-Product Attention

### 4.1 定义
输入为 Query/Key/Value：$Q, K, V$。

论文使用“缩放点积注意力”：
$$\text{Attention}(Q,K,V) = \text{softmax}\left(\frac{QK^\top}{\sqrt{d_k}}\right)V$$

**为什么要除以 $\sqrt{d_k}$？**
- 若 $q,k$ 的分量独立同分布、方差为 1，则 $q\cdot k$ 的方差随 $d_k$ 增大而增大
- 点积过大使 softmax 进入饱和区，梯度变小 → 学不动
- 缩放相当于把 logits 的尺度归一化，改善训练稳定性

### 4.2 Masked Attention（Decoder 自注意力）
在 decoder 自注意力里，对非法连接（$j>i$）的 logits 加上 $-\infty$（实现上是一个上三角 mask），确保自回归：
$$p(y_i | y_{<i}, x)$$

---

## 5. Multi-Head Attention：为什么“多头”关键

### 5.1 公式
对每个 head 做不同的线性投影：
$$

\begin{align}
	ext{head}_i &= \text{Attention}(QW_i^Q, KW_i^K, VW_i^V)\\
	ext{MultiHead}(Q,K,V) &= \text{Concat}(\text{head}_1,\dots,\text{head}_h)W^O
\end{align}

$$

其中 Base 模型：
- 头数 $h=8$
- $d_k=d_v=d_{model}/h=64$

### 5.2 直觉
- 单头注意力本质是对 $V$ 做一次“加权平均”，表达能力受限（尤其当多种关系需要同时编码）
- 多头让模型在不同子空间里并行学习：
	- 句法依赖（主谓宾、修饰关系）
	- 指代消解（its→The Law）
	- 局部对齐 vs 长程对齐

论文附录可视化显示：不同 head 学到不同“功能”，并且在第 5 层已出现明显结构性行为。

---

## 6. Position-wise FFN：逐 token 的非线性变换

每个位置独立、共享参数的两层 MLP：
$$\text{FFN}(x)=\max(0, xW_1+b_1)W_2+b_2$$

Base：$d_{ff}=2048$，输入输出维度为 $d_{model}=512$。

可把它看成 kernel size=1 的卷积：负责“通道混合/特征变换”，而 attention 负责“跨位置混合”。

---

## 7. Positional Encoding：没有递归/卷积如何注入顺序

Transformer 结构本身对置换等变（没有位置概念），所以必须显式注入位置信息。

论文用固定正弦/余弦位置编码（与 embedding 相加）：
$$
\begin{align}
PE(pos,2i) &= \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right)\\
PE(pos,2i+1) &= \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)
\end{align}
$$
关键点：
- 不同维度对应不同频率，波长从 $2\pi$ 到 $10000\cdot 2\pi$ 几何递增
- 论文假设：固定 offset 的相对位移可由 $PE(pos+k)$ 表示成 $PE(pos)$ 的线性函数，便于学相对位置关系
- 实验中“学习的位置 embedding”和正弦位置编码效果几乎相同（Table 3 (E)）

---

## 8. 复杂度与“最长路径”：Transformer 为什么更易学长依赖

论文用三指标比较层类型（Table 1）：

- **每层复杂度**（大致）：
	- Self-attention：$O(n^2 d)$
	- RNN：$O(n d^2)$（且顺序依赖）
	- CNN：$O(k n d^2)$（取决于卷积形式）
- **最少串行操作数**：
	- Self-attention：$O(1)$（整段并行算矩阵乘）
	- RNN：$O(n)$
	- CNN：$O(1)$
- **最大路径长度**（任意两 token 信息交互所需层数的上界）：
	- Self-attention：$O(1)$
	- RNN：$O(n)$
	- CNN：$O(\log n)$（空洞卷积）或 $O(n/k)$（常规堆叠）

这解释了：
- Transformer 更适合建模跨句长距离依赖（路径短，梯度与信息传播更直接）
- 但注意：$n^2$ 成本导致超长序列任务需要“稀疏/局部/块状注意力”等后续改进（论文也在结论里提到 restricted attention 设想）

---

## 9. 训练细节：决定“能不能复现”的关键超参

### 9.1 数据与 batching
- WMT14 EN-DE：约 4.5M 句对，BPE 共享词表约 37K
- WMT14 EN-FR：36M 句对，word-piece 词表 32K
- batch：按长度分桶，每 batch 约 25K source tokens + 25K target tokens

### 9.2 优化器与学习率计划（很重要）
Adam：$\beta_1=0.9,\ \beta_2=0.98,\ \epsilon=10^{-9}$。

学习率 schedule（带 warmup）：
$$\text{lrate}=d_{model}^{-0.5}\cdot \min(step^{-0.5},\ step\cdot warmup^{-1.5})$$
其中 $warmup=4000$。

直觉：
- 前期线性升温避免不稳定
- 后期按 $1/\sqrt{step}$ 衰减

### 9.3 正则化
- Residual Dropout：Base 用 $P_{drop}=0.1$
- Label Smoothing：$\epsilon_{ls}=0.1$（提升 BLEU，牺牲 PPL：模型更“不自信”但泛化更好）

---

## 10. 结果与消融：哪些组件真的重要

### 10.1 机器翻译主结果（newstest2014）
- EN-DE：Transformer Big 达到 28.4 BLEU，超过当时 SOTA（含 ensemble）>2 BLEU
- EN-FR：单模型 41.8 BLEU（论文摘要；正文也给出 41.0 的表述，取决于训练设置/汇报口径）

训练成本：在 8×P100 上，Base 约 12 小时（100K steps），Big 约 3.5 天（300K steps）。

推断：beam size 4，length penalty $\alpha=0.6$；Base 平均最后 5 个 checkpoint，Big 平均最后 20 个。

### 10.2 结构消融（Table 3 的要点解读）
1) **多头 vs 单头**：单头会掉约 0.9 BLEU → 多头确实缓解“平均化”问题
2) **$d_k$ 太小会伤性能**：说明仅靠点积做兼容性判断并不“容易”，需要足够的 key/query 表达维度
3) **更大模型更好**：$d_{model}$、$d_{ff}$ 增大普遍提升
4) **Dropout 很关键**：无 dropout 会过拟合；适当增大 dropout 能稳定训练
5) **正弦位置编码 vs 学习位置 embedding**：差别不大 → 核心不是“哪种位置编码”，而是“必须有位置”

---

## 11. 迁移到句法分析：Transformer 不只是翻译

英语成分句法分析（WSJ Penn Treebank）：
- 4-layer Transformer（$d_{model}=1024$）
- WSJ-only 40K 句：F1 约 91.3，已超过 Berkeley Parser（90.4）
- 半监督（~17M 句）：F1 约 92.7

要点：
- 输出比输入长很多且结构强约束，证明该架构可泛化到“结构化生成”

---

## 12. 这篇论文最“本质”的思想拆解

### 12.1 注意力 = 可微检索/路由（routing）
可以把 $\text{softmax}(QK^\top)$ 看成“内容寻址”的路由矩阵：
- Query 决定“我要找什么”
- Key 决定“我能提供什么”
- Value 是“我提供的内容”

与 RNN 不同：信息不是沿时间步被动传递，而是每个位置主动从全局检索需要的信息。

### 12.2 架构解耦：
- **Attention 负责 token 之间的信息交互**（跨位置混合）
- **FFN 负责每个 token 内部的非线性变换**（通道混合）
这种解耦为后续大量改进奠定了“模块化”基础（更换 attention/FFN/位置编码即可）。

### 12.3 并行性是“性能红利”的来源之一
论文不仅靠“更强表达”，也靠“更快训练”实现更充分的调参和更大模型的可行性。

---

## 13. 站在今天（2025）回看：影响与局限

### 13.1 影响
- 直接奠定了后续大规模预训练语言模型（GPT/BERT/…）的主干结构
- “多头注意力 + 残差 + 归一化 + 大 FFN”成为默认配方
- 引发了对位置编码、注意力稀疏化、长上下文建模、KV cache 等方向的长期研究

### 13.2 局限（也是后来工作的主战场）
- $O(n^2)$ 注意力在长序列上昂贵（算力与显存都随 $n^2$ 增长）
- 位置编码方案本身并非完美：固定/学习、绝对/相对、RoPE/ALiBi 等改进都来自这一缺口
- 多头并不总是“越多越好”，存在冗余 head（后续有 head pruning/共享等工作）
- Decoder 自回归生成仍是串行瓶颈（论文结论也提到希望“让生成更不串行”）

---

## 14. 复现/实现备忘（给工程实现用）

1) 形状约定（批量矩阵）
- $Q\in\mathbb{R}^{B\times n\times d_{model}}$，投影后 $Q\in\mathbb{R}^{B\times h\times n\times d_k}$
- 注意力权重 $A=\text{softmax}(QK^\top/\sqrt{d_k})\in\mathbb{R}^{B\times h\times n\times n}$
- 输出 $AV\in\mathbb{R}^{B\times h\times n\times d_v}$

2) Mask 的实现
- decoder self-attention：对上三角（未来）位置置为 $-\infty$（或一个很大的负数）

3) 训练稳定性关键项
- 学习率 warmup schedule（式 (3)）
- dropout + label smoothing
- 残差与 LayerNorm 的位置（原文是 Post-LN：Sublayer 后再 LayerNorm；后续很多实现改成 Pre-LN，训练更稳，但这已超出原文复现范围）

---

## 15. 我读完后的问题清单（建议你继续追的点）

- Multi-Head 的“分辨率下降”到底是什么数学意义？是否可视为低秩/平均化导致的信息丢失？
- 位置编码为什么“学的”和“正弦的”差不多：是任务短程为主，还是模型能自己学习相对关系？
- 如果把 $QK^\top$ 看成图的邻接矩阵，Transformer 是否就是一种数据依赖的图信息传播？与 GNN 的关系是什么？
- 对长序列，restricted attention 会如何影响 BLEU 与速度？后续工作用了哪些稀疏模式最有效？

---

## 参考速记
- Attention：$\text{softmax}(QK^\top/\sqrt{d_k})V$
- FFN：$\max(0, xW_1+b_1)W_2+b_2$
- LR schedule：$d_{model}^{-0.5}\cdot \min(step^{-0.5}, step\cdot warmup^{-1.5})$


