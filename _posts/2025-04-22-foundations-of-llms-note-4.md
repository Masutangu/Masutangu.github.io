---
layout: post
date: 2025-4-22T14:11:09+08:00
title: 【读书笔记】Foundations-of-LLMs 参数高效微调
tags: 
  - 机器学习
  - 读书笔记
---

<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>

本文是[《Foundations-of-LLMs》](https://github.com/ZJU-LLMs/Foundations-of-LLMs) 第四章【参数高效微调】的笔记。


主流的下游任务适配方法有两种：**上下文学习**（In-context learning）和**指令微调**（Instruction Tuning）。

上下文学习的核心思想是将不同类型的任务都转化为生成任务，通过设计 Prompt 来驱动大语言模型完成下游任务。上下文学习能有效利用大语言模型的能力，但它缺点也很明显：1）上下文学习的性能和微调依旧存在差距，并且 Prompt 设计需要花费大量的人力成本，不同 Prompt 的最终任务性能有较大差异；2）上下文学习虽然完全不需要训练，但在推理成本会随 Prompt 中样例的增多快速增加。

指令微调（Instruction Tuning）是另一种进行下游任务适配的方法。指令微调旨在对模型进行任务指令的学习，使其能更好地理解和执行各种自然语言处理任务的指令。指令微调需首先构建指令数据集，然后在该数据集上进行监督微调。然而，监督微调需要较大的计算资源，以 LLaMA2-7B 模型为例，直接进行全量微调需要近 60GB 内存，普通的消费级 GPU（如 RTX4090（24GB））无法完成微调。因此，为了在资源受限的环境中有效微调大语言模型，研究参数高效的微调技术显得尤为重要。

**参数高效微调**（Parameter-Eﬀicient Fine-Tuning，PEFT）旨在避免微调全部参数，减少在微调过程中需要更新的参数数量和计算开销，从而提高微调大语言模型的效率。主流的 PEFT 方法可以分为三类：**参数附加方法**（Additional Parameters Methods）、**参数选择方法**（Parameter Selection Methods）以及**低秩适配方法**（Low-Rank Adaptation Methods）。如下图所示：

<img src="/assets/images/foundations-of-llms-note-4/illustration-1.png" width="600" alt=""/>

## 参数附加方法

参数附加方法（Additional Parameters Methods）在模型结构中附加新的、较小的可训练模块。将原始模型参数冻结，仅微调这些新加入的模块，从而来实现高效微调。这些模块通常称为**适应层**（Adapter Layer）。它们被插入到模型的不同层之间，用于捕获特定任务的信息。由于这些新增的适应层参数量很小，所以参数附加方法能够显著减少需要更新的参数量。典型方法包括：**适配器微调**（Adapter-tuning）、**提示微调**（Prompt-tuning）、**前缀微调**（Prefix-tuning）和**代理微调**（Proxy-tuning）。

参数附加方法按照附加位置可以分为三类：加在输入、加在模型以及加在输出。

### 加在输入

加在输入的方法将额外参数附加到模型的输入嵌入（Embedding）中，其中最经典的方法是 Prompt-tuning。Prompt-tuning 在模型的输入中引入可微分的连续张量，通常也被称为软提示（Soft prompt）。软提示作为输入的一部分，与实际的文本数据一起被送入模型。在微调过程中，仅软提示的参数会被更新，其他参数保持不变，因此能达到参数高效微调的目的。

### 加在模型

加在模型的方法将额外的参数或模型添加到预训练模型的隐藏层中，其中经典的方法有 Prefix-tuning、Adapter-tuning 和 AdapterFusion。

#### Prefix-tuning

Prefix-tuning 和 Prompt-tuning 十分类似，区别在于 Prompt-tuning 仅将软提示添加到输入嵌入中，而Prefix-tuning 将一系列连续的可训练前缀（Pre-fixes，即Soft-prompt）插入到输入嵌入以及 Transformer 注意力模块中。相比 Prompt-tuning，Prefix-tuning 大幅增加了可学习的参数量。

#### Adapter-tuning

Adapter-tuning 向预训练语言模型中插入新的可学习的神经网络模块，称为适配器（Adapter）。适配器模块通常采用瓶颈（Bottomneck）结构，即一个上投影层、一个非线性映射和一个下投影层组成的全连接模块。

Adapter-tuning 在 Transformer 的每一个多头注意力层和全连接层之后添加适配器。与 Transformer 的全连接层不同，由于采用了瓶颈结构，**适配器的隐藏维度通常比输入维度小**。

### 加在输出

在微调大语言模型时，通常会面临以下问题：首先，大语言模型的参数数量可能会非常庞大，例如 LLaMA 系列最大的模型拥有 70B 参数，即使采用了 PEFT 技术，也难以在普通消费级 GPU 上完成下游任务适应；其次，用户可能无法直接访问大语言模型的权重（黑盒模型），这为微调设置了障碍。

为了应对这些问题，**代理微调**（Proxy-tuning）提供了一种轻量级的解码时（Decoding-time）算法，允许我们在不直接修改大语言模型权重的前提下，通过仅访问模型输出词汇表预测分布，来实现对大语言模型的进一步定制化调整。

## 参数选择方法

参数选择方法（Parameter Selection Methods）仅选择模型的一部分参数进行微调，而冻结其余参数。典型的方法包括：**BitFit**、**Child-tuning** 以及 **FishMask** 等。通常参数选择方法分为两类：**基于规则**的方法和**基于学习**的方法。

### 基于规则的方法

基于规则的方法根据人类专家的经验，确定哪些参数应该被更新。基于规则的方法中最具代表性的方法是 BitFit。BitFit 通过仅优化神经网络中的每一层的偏置项（Biases）以及任务特定的分类头来实现参数高效微调。由于偏置项在模型总参数中所占比例极小（约 0.08%-0.09%），BitFit 有极高的参数效率。只微调少量参数，BitFiT 依然能在 GLUE Benchmark 上与全量微调相媲美，甚至在某些任务上表现更好。此外，BitFit 方法相比全量微调允许使用更大的学习率，因此该方法整体优化过程更稳定。然而，该方法仅在小模型（如 BERT、RoBERT 等）上进行验证性能，在更大模型上的性能表现如何尚且未知。

### 基于学习的方法

基于学习的方法在模型训练过程中自动地选择可训练的参数子集。其中，最为典型方法是 Child-tuning。其通过**梯度掩码矩阵策略**实现仅对选中的择子网络进行梯度更新，而屏蔽子网络梯度以外的梯度，从而实现对微调参数的选择，达到参数高效微调的目的。

## 低秩适配方法

低秩适配方法（Low-rank Adaptation Methods）通过低秩矩阵来近似原始权重更新矩阵，并冻结原始参数矩阵，仅微调低秩更新矩阵。由于低秩更新矩阵的参数数量远小于原始的参数更新矩阵，因此大幅节省了微调时的内存开销。**LoRA** 是经典的低秩适配方法，后续有 **AdaLoRA**、**DyLoRA** 以及 **DoRA** 等变体被提出，进一步改进了 LoRA 性能。

### LoRA

低秩适配（Low-rank Adaptation, LoRA） 提出利用低秩矩阵近似参数更新矩阵来实现低秩适配。该方法将参数更新矩阵低秩分解为两个小矩阵。在微调时，通过微调这两个小矩阵来对大语言模型进行更新，大幅节省了微调时的内存开销。

给定一个密集神经网络层，其参数矩阵为 $W_0 \in \mathbb{R}^{d \times k}$，为适配下游任务，通常需要学习参数更新矩阵 $\Delta W \in \mathbb{R}^{d \times k}$，对原始参数矩阵进行更新 $W= W_0 + \Delta W$ 。对于全量微调过程，$ \Delta W$ 是需对该层所有 $d \times k$ 个参数计算梯度，这通常需要大量的 GPU 内存，成本高昂。为解决这一问题，LoRA 将 $\Delta W$ 分解为两个低参数量的矩阵 $B \in \mathbb{R}^{d \times r}$ 和 $A \in \mathbb{R}^{r \times k}$，更新过程变为：$W= W_0 + \alpha BA$。

其中，秩 $r \ll min\{d, k\}$，$B$ 和 $A$ 分别用随机高斯分布和零进行初始化，$\alpha$ 是缩放因子，用于控制 LoRA 权重的大小。在训练过程中，固定预训练模型的参数，仅微调 $B$ 和 $A$ 的参数。因此，在训练时，LoRA 涉及的更新参数数量为 $r \times(d + k)$，远小于全量微调 $d \times k$。

对于基于 Transformer 的大语言模型，密集层通常有两种类型：注意力模块中的投影层和前馈神经网络（FFN）模块中的投影层。在原始研究中，LoRA 被应用于注意力层的权重矩阵。后续工作表明将其应用于 FFN 层可以进一步提高模型性能。

LoRA 仅微调部分低秩参数，因此具有很高的参数效率，同时不会增加推理延迟。此外，低秩矩阵还可以扩展为**低秩张量**，或与 Kronecker 分解结合使用，以进一步提高参数效率。除了参数效率外，在训练后可以将 LoRA 参数与模型参数分离，所以 LoRA 还具有**可插拔性**。当有多个任务的 LoRA 插件时，可以将这些插件组合在一起，以获得良好的跨任务泛化性能。

虽然 LoRA 在一些下游任务上能够实现较好的性能，但在许多复杂的下游任务上，LoRA 与全量微调之间仍存在性能差距。为弥补这一差距，许多 LoRA 变体方法被提出，以进一步提升 LoRA 在下游任务中的适配性能。现有方法主要从以下几个角度进行改进：(1) 打破低秩瓶颈；(2) 动态秩分配；(3) 训练过程优化。

### 打破低秩瓶颈

LoRA 的低秩更新特性使其在参数效率上具有优势；然而这也限制了大规模语言模型记忆新知识和适应下游任务的能力，即存在低秩瓶颈。Biderman 等人的实验研究表明，全量微调的秩显著高于 LoRA 的秩（10-100 倍），并且增加 LoRA 的秩可以缩小 LoRA 与全量微调之间的性能差距。

ReLoRA 提出了一种合并和重置（merge-and-reinit）的方法，该方法在微调过程中周期性地将 LoRA 模块合并到大语言模型中，并在合并后重新初始化 LoRA 模块和优化器状态。具体地，合并的过程如下：

$$
W^i \leftarrow W^i + \alpha B^iA^i
$$


其中，$W^i$ 是原始的权重矩阵，$B^i$ 和 $A^i$ 是低秩分解得到的矩阵，$\alpha$ 是缩放因子。合并后，将重置 $B^i$ 和 $A^i$ 的值重置，通常 $B^i$ 会使用特定的初始化方法（如 Kaiming 初始化）重新初始化，而 $A^i$ 则被设置为零。为了防止在重置后模型性能发散，ReLoRA 还会通过幅度剪枝对优化器状态进行部分重置。合并和重置的过程允许模型在保持总参数量不变的情况下，通过多次低秩 LoRA 更新累积成高秩状态，从而使得 ReLoRA 能够训练出性能接近全秩训练的模型。

### 动态秩分配

然而，LoRA 的秩并不总是越高越好，冗余的 LoRA 秩可能会导致性能和效率的退化。并且，微调时权重的重要性可能会因 Transformer 模型中不同层而存在差异，因此需要为每个层分配不同的秩。

AdaLoRA 通过将参数更新矩阵参数化为奇异值分解（SVD）的形式，再通过奇异值剪枝动态调整不同层中 LoRA 模块的秩。具体地，AdaLoRA  使用奇异值分解重新表示 $\Delta W$ ，即

$$
W = W_0 + \Delta W= W_0 + P \Lambda Q
$$

其中 $P \in \mathbb{R}^{d \times r} $ 和 $Q \in \mathbb{R}^{r \times k} $ 是正交矩阵，$\Lambda$ 是一个对角矩阵，其中包含 $\\{\lambda_i\\}_{1 \leq i \leq r}$ 的奇异值。在训练过程中，$W_0$ 的参数被固定，仅更新 $P$、$\Lambda$ 和 $Q$ 的参数。根据梯度权重乘积大小的移动平均值构造奇异值的重要性得分，对不重要的奇异值进行迭代剪枝。此外，为了增强稳定训练性，AdaLoRA 引入一个额外的惩罚项确保 $P$ 和 $Q$ 之间的正交性：

$$
R(P, Q) = ||P^\top P − I||^2_F + ||QQ^\top - I||^2_F
$$

其中，$I$ 是单位矩阵，$\|\|\cdot\|\|_F$ 代表 Frobenius 范数。

### 训练过程优化

在实际微调过程中，LoRA 的收敛速度比全量微调要慢。此外，它还对超参数敏感，并且容易过拟合。为了解决这些问题，一些工作尝试对 LoRA 的训练过程进行优化。代表性方法 DoRA（权重分解低秩适应）提出**约束梯度更新，侧重于更新参数的方向变化**。它将预训练权重 $W_0 \in \mathbb{R}^{d \times k}$ 分解为方向和大小两个组件，并仅将 LoRA 应用于方向组件以增强训练稳定性。DoRA 将 $W_0 \in \mathbb{R}^{d \times k}$ 重新表示为：

$$
W_0 = m\frac{V}{||V||_c}=||W_0||_c\frac{W_0}{||W_0||_c}
$$

其中，$m \in \mathbb{R}^{1 \times k}$ 是大小向量，$V \in \mathbb{R}^{d \times k}$ 是方向矩阵，$\|\|\cdot\|\|_c$ 是矩阵在每一列上的向量范数。随后，DoRA 仅对方向矩阵 $V$ 施加 LoRA 进行参数化，定义为：


$$
W' = \underline{m}\frac{V + \underline{\Delta V}}{||V + \underline{\Delta V}||_c} = \underline{m}\frac{W_0 + \underline{BA}}{||W_0 + \underline{BA}||_c}
$$

其中，$\Delta V$ 是由 LoRA 学习的增量方向更新，下划线参数表示可训练参数。

### 基于 LoRA 插件的任务泛化

LoRAHub 提供了一个可用的多 LoRA 组合的方法框架。其可将已有任务上得到的 LoRA 插件进行组合，从而获得解决新任务的能力。LoRAHub 包括两个阶段：**组合阶段**和**适应阶段**。在组合阶段，LoRAHub 将已学习的 LoRA 模块通过逐元素线性加权组合为单一模块。

## 实践与应用

目前最流行的 PEFT 框架是由 Hugging Face 开发的开源库 HF-PEFT，它旨在提供最先进的参数高效微调方法。

