---
title: LoRA 
date: 2024-08-02 11:00:00 +0800
categories: [Paper, LLM]
tags: [tune]
render_with_liquid: false
math: true
---


《LoRA: Low-Rank Adaptation of Large Language Models》是一篇在大模型微调领域非常重要的论文，提出了一种高效的参数微调方法 **LoRA（Low-Rank Adaptation）**，主要解决 **大语言模型（LLM）微调时参数量过大、计算和存储成本高** 的问题。


***

### **1. 背景问题**

*   大模型（如 GPT、BERT）通常有数十亿参数，直接微调所有参数需要巨大的计算资源和存储空间。
*   常见替代方案：
    *   **全量微调**：更新所有参数，效果好但成本极高。
    *   **Adapter 方法**：在模型中插入小模块，减少更新参数，但推理时增加延迟。
    *   **Prefix-tuning / Prompt-tuning**：只调整输入提示，参数少但效果有限。

***

### **2. LoRA 的核心思想**

*   **低秩分解（Low-Rank Decomposition）**：假设在微调过程中，权重矩阵的更新可以用一个低秩矩阵近似。
*   对于 Transformer 中的权重矩阵 $$ W \in \mathbb{R}^{d \times k}  $$，LoRA 不直接更新 $$ W $$，而是引入两个小矩阵：

    $$ \Delta W = A B, \quad A \in \mathbb{R}^{d \times r}, B \in \mathbb{R}^{r \times k}, r \ll \min(d,k) $$

    其中 $$ r $$ 是低秩维度，通常远小于原始维度。
*   **冻结原始权重**，只训练 $$ A $$ 和 $$ B $$，这样参数量大幅减少。

***

### **3. 优势**

*   **参数效率高**：只需训练少量参数（通常减少 10^3 倍）。
*   **推理无额外延迟**：训练后可以将 ( \Delta W ) 合并到原始权重中。
*   **灵活性强**：可以在多个任务上共享同一个基础模型，只加载不同的 LoRA 模块。

***

### **4. 应用场景**

*   微调大语言模型（GPT、LLaMA、BERT）用于特定任务，如文本分类、问答、代码生成。
*   多任务场景：一个基础模型 + 多个 LoRA 模块，快速切换任务。

***

### **5. 实验结果**

*   在 NLP 下游任务中，LoRA 的性能接近全量微调，但参数量和显存消耗显著降低。
*   例如：在 GPT-3 上微调，LoRA 只需训练百万级参数，而不是数十亿。

***

### **6. 总结**

LoRA 是一种 **高效、低成本、性能优良** 的微调方法，已成为大模型微调的主流技术之一，尤其在开源社区（如 Hugging Face）广泛应用。

