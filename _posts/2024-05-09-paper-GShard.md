---
title: GShard - Scaling Giant Models with Conditional Computation and Automatic Sharding
date: 2024-05-09 11:00:00 +0800
categories: [Paper, Computation]
tags: [automatic sharding]
render_with_liquid: false
math: true
---


《GShard: Scaling Giant Models with Conditional Computation and Automatic Sharding》是由 Google 研究团队（Dmitry Lepikhin 等人）于 2020 年发表的重要论文。这篇论文解决了如何将神经网络模型（特别是 Transformer）扩展到数千亿乃至万亿参数规模的工程与算法难题。


### 1. 核心目标：扩展模型规模而不成倍增加计算量

在深度学习中，增加模型参数通常能提升效果，但传统的“稠密（Dense）”模型会导致计算成本随参数量线性增长。GShard 的目标是实现**超大规模参数量**的同时，保持**较低的计算开销**。

### 2. 核心技术一：稀疏门控专家混合网络（MoE）

GShard 采用了 **Mixture-of-Experts (MoE)** 架构：

* **专家（Experts）**：将 Transformer 中的全连接层（FFN）替换为多个并行的“专家”层。
* **门控机制（Gating Network）**：对于每一个输入的 Token，门控网络会选择最相关的 1 到 2 个专家进行处理，而不是激活所有专家。
* **效果**：这使得模型可以拥有数千亿参数（分布在不同专家中），但每个 Token 实际经过的计算路径依然很短，从而在增加容量的同时控制了计算成本。

### 3. 核心技术二：自动分片（Automatic Sharding）

在大规模分布式训练中，手动编写数据并行或模型并行的代码极其复杂。GShard 提供了一套**编译器注解（Compiler Annotations）**：

* **编程简便**：开发者只需在张量上添加简单的注释（如指定某个维度如何分布在硬件集群上），而无需关心底层的通信细节（如 `All-to-All` 操作）。
* **XLA 编译器优化**：Google 的 XLA 编译器会自动将这些注解转化为高效的执行计划，处理跨设备的数据分片和梯度同步。

### 4. 关键改进与工程实践

为了让 MoE 在大规模运行时保持稳定和高效，论文提出了几项关键创新：

* **平衡损耗（Balance Loss）**：防止大部分 Token 涌向少数几个“明星专家”，导致计算负载不均。
* **随机路由（Random Routing）**：在选择专家时引入一定的随机性，提高模型的鲁棒性。
* **位置辅助路由（Position-based Routing）**：优化长序列处理时的效率。

### 5. 实验成果

* **规模突破**：作者展示了一个拥有 **6000 亿参数**的翻译模型。
* **效率提升**：该模型在多语言翻译任务（如 100 种语言互译）上取得了显著的质量提升，且训练速度比同等规模的稠密模型快得多。
* **收敛速度**：实验证明，超大模型不仅最终效果更好，而且在达到相同精度时所需的训练步数更少。

### 6. 论文的深远影响

GShard 是现代“巨型模型”时代的奠基石之一：

* 它是 Google 后来推出 **Switch Transformer**（1.6 万亿参数）和 **PaLM** 模型的技术前身。
* 它证明了 **MoE + 自动并行化编译器** 是通往万亿参数 AGI 的可行路径。
* 目前流行的开源大模型（如 Mixtral 8x7B）以及许多闭源大模型（如 GPT-4 传闻中也采用了 MoE）都受到了 GShard 思想的影响。

**总结：** GShard 的伟大之处在于它不仅提出了**稀疏激活**的算法优化，还通过**编译器技术**简化了超大规模并行计算的实现难度，真正让“巨型模型”走出了理论，进入了工业实践。
