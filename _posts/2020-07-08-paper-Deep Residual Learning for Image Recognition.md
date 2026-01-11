---
title: Deep Residual Learning for Image Recognition
date: 2020-07-08 14:00:00 +0800
categories: [Machine Learning, Deep Learning]
tags: [image, resnet]
render_with_liquid: false
---

《Deep Residual Learning for Image Recognition》（深度残差学习在图像识别中的应用）是由何恺明（Kaiming He）等人在 2015 年提出的里程碑式论文。这篇论文引入了 **ResNet（残差网络）** 架构，解决了深层神经网络训练中的退化问题，并在 ILSVRC 2015 等多项视觉识别大赛中夺魁。

### 1. 核心问题：网络退化（Degradation Problem）

在 ResNet 出现之前，人们发现简单地通过堆叠层数来增加网络深度，并不能持续提升准确率。相反，当深度增加到一定程度时，准确率会达到饱和并迅速下降（并非由过拟合引起，因为训练误差也随之增加）。

* **原因**：深层网络在优化过程中极难收敛，且存在梯度消失或爆炸的挑战。

### 2. 核心创新：残差学习与捷径连接

为了解决退化问题，作者提出了**残差学习（Residual Learning）**框架：

* **原理**：与其让层直接拟合目标映射 ，不如让其拟合残差映射 。
* **捷径连接（Skip Connections/Shortcut Connections）**：通过“捷径”将输入  直接跳过几层并与输出相加，形成 。
* **优势**：如果恒等映射（Identity Mapping）是最佳的，将残差  优化为零比在多个非线性层中学习一个恒等映射要容易得多。

### 3. 网络架构：ResNet 变体

ResNet 展示了极深网络的训练可能性（最深达 152 层甚至 1000 层）：

* **基础块（Basic Block）**：常用于 ResNet-18 和 ResNet-34，由两个  卷积层组成。
* **瓶颈块（Bottleneck Block）**：常用于 ResNet-50/101/152，采用 、、 的三层设计，在减少参数量的同时增加了网络深度。

### 4. 主要成就与影响

* **性能突破**：ResNet-152 的深度是当时 VGG 网络的 8 倍，但复杂度更低。在 ImageNet 测试集上达到了 3.57% 的 top-5 误差率。
* **横扫奖项**：该技术在 ILSVRC 2015 图像分类、检测、定位以及 COCO 检测和分割任务中均获得第一名。
* **行业标准**：残差连接现已成为深度学习的基础组件，被广泛应用于 Transformer（如 GPT、BERT）等现代架构中。

