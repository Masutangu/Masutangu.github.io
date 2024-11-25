---
layout: post
date: 2024-11-25T22:56:43+08:00
title: 【论文笔记】DistilBERT, a distilled version of BERT-smaller, faster, cheaper and lighter
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

本文是 [《DistilBERT, a distilled version of BERT: smaller, faster, cheaper and lighter》](https://arxiv.org/pdf/1910.01108) 的笔记。

## 摘要

随着大规模预训练模型在自然语言处理（NLP）中的迁移学习变得越来越普遍，**在边缘计算和/或在受限的计算训练或推理预算下运行这些大型模型仍然具有挑战性**。在这项工作中，我们提出了一种方法来预训练一个较小的通用语言表示模型，称为 DistilBERT，然后可以在广泛的任务上进行微调，性能与其较大的同类模型一样好。

大多数以往的研究使用蒸馏来构建特定任务的模型，而我们在**预训练阶段利用知识蒸馏**，能够将 BERT 模型的大小减少 40%，同时保留其 97% 的语言理解能力，且速度快 60%。为了利用大型模型在预训练期间学到的归纳偏见（inductive biases），我们引入了一个**结合语言建模、蒸馏和余弦距离损失的三重损失**。

## 1 Introduction

近两年来，自然语言处理（NLP）中的迁移学习方法逐渐兴起，大规模预训练语言模型已成为许多 NLP 任务的基本工具。这些模型通常具有数亿个参数。当前研究表明，训练更大的模型仍然可以在下游任务上获得更好的性能。但更大模型的趋势也引发了几个问题：
- 指数级扩展这些模型的计算需求所带来的环境成本
- 不断增长的计算和内存需求可能会阻碍其广泛采用

本文展示了使用知识蒸馏预训练的更小的语言模型在许多下游任务上可以达到类似的性能，这些模型在推理时间上更快、更轻量，并且训练所需的计算预算也更小。我们的通用预训练模型可以在多个下游任务上进行微调，保持较大模型的灵活性。我们还展示了我们的压缩模型足够小，可以在边缘设备上运行，例如移动设备。

通过使用三重损失，我们展示了使用较大的 Transformer 语言模型进行监督，蒸馏预训练出的比 Transformer 小 40% 的模型，可以在各种下游任务上实现类似的性能，同时在推理时速度提高了 60%。进一步的消融研究表明，三重损失的所有组成部分对于获得最佳性能都是重要的。

## 2 知识蒸馏

**知识蒸馏**（knowledge distillation）是一种压缩技术，其中一个**紧凑的模型（学生模型）被训练来复现一个更大模型（教师模型）或模型集合**的行为。

在监督学习中，分类模型通常通过最大化金标签（gold labels）的估计概率来训练以预测实例类别。因此，标准的训练目标涉及最小化模型的预测分布与训练标签的独热（one-hot）经验分布之间的交叉熵。在训练集上表现良好的模型将在正确类别上预测具有高概率的输出分布，并在其他类别上预测接近零的概率。

学生模型通过教师模型的**软目标概率**（soft target probabilities）进行蒸馏损失训练：

$$
L_{ce} =\sum_i t_i * \log(s_i)
$$

其中 $t_i$ 是教师估计的概率，$s_i$ 是学生估计的概率。这个目标通过利用完整的教师分布产生丰富的训练信号。遵循 Hinton 等人 [2015] 的方法，我们使用了*softmax-temperature*：$p_i = \frac{\exp(z_i/T) }{\sum_j \exp(z_j /T)}$，其中 $T$ 控制输出分布的平滑度，$z_i$ 是类别 $i$ 的模型得分。

在训练时，学生和教师都应用相同的温度 $T$，而在推理时，$T$ 设置为 1 以恢复标准的*softmax*。

最终的训练目标是蒸馏损失 $L_{ce}$ 与监督训练损失的线性组合，在我们的案例中是**掩码语言建模**损失 $L_{mlm}$ [Devlin et al., 2018]。我们发现添加一个**余弦嵌入**损失（[CosineEmbeddingLoss](https://pytorch.org/docs/stable/generated/torch.nn.CosineEmbeddingLoss.html)）（$L_{cos}$）是有益的，这将趋向于**对齐学生和教师隐藏状态向量的方向**（align the directions of the student
and teacher hidden states vectors）。


## 3 DistilBERT: A Distilled Version Of Bert

### 学生架构
在本研究中，学生模型 - DistilBERT - 的总体架构与 BERT 相同。去掉了 *token-type embeddings* 和 *pooler*，并且层数减少了2倍。Transformer 架构中使用的大多数操作（线性层和 *layer normalisation*）在现代线性代数框架中已经高度优化，我们的研究表明，在固定参数预算的情况下，**张量的最后一个维度（隐藏大小维度）的变化对计算效率的影响小于层数等其他因素的变化**。因此，我们专注于**减少层数**。

### 学生初始化
除了上述优化和架构选择外，我们训练过程中的一个重要元素是为子网络找到正确的初始化以便收敛。利用教师和学生网络之间的共同维度，我们通过取教师网络的两层中的一层来初始化学生网络。

### 蒸馏
我们应用了 Liu 等人 [2019] 最近提出的训练 BERT 模型的最佳实践。因此，DistilBERT 是在**非常大的批次**上进行蒸馏的，利用**梯度累积**（gradient accumulation）（每个批次高达 4K 个例子），使用**动态掩码**，并且**没有下一句预测目标**。

### 数据和计算能力
我们在与原始 BERT 模型相同的语料库上训练 DistilBERT：英语维基百科和 Toronto Book Corpus。DistilBERT 在 8 个 16GB V100 GPU 上训练了大约 90 小时。作为比较，RoBERTa 模型 [Liu et al., 2019] 需要在 1024 个 32GB V100 上训练 1 天。

### 性能对比

DistilBERT 在下游任务上表现出可比的性能：

| 模型      | IMDb   | SQuAD     |
|------------|--------|-----------|
| BERT-base  | 93.46  | 81.2/88.5 |
| DistilBERT     | 92.82  | 77.7/85.8 |

### 模型大小和推理时间
DistilBERT 显著更小，同时始终更快。在 CPU 上以批处理大小为 1 完成 GLUE 任务 STS-B（情感分析）的全过程的推理时间如下：

| 模型      | # 参数（百万） | 推理时间（秒）   |
|------------|----------------|------------------|
| ELMo       | 180            | 895              |
| BERT-base  | 110            | 668              |
| DistilBERT | 66             | 410              |

DistilBERT 在保持较高性能的同时，显著减小了模型大小，并提高了推理速度。

## 4 实验

### 4.2 消融（Ablation）研究

在本节中，我们研究了三重损失和学生初始化对蒸馏模型性能的影响。表4展示了与完整三重损失的差异：移除*掩码语言建模*损失影响不大。


| Ablation                                    | GLUE macro-score 变化   |
|---------------------------------------------|------------------|
| ∅ - Lcos - Lmlm                             | -2.96            |
| Lce - ∅ - Lmlm                              | -1.46            |
| Lce - Lcos - ∅                              | -0.31            |
| 三重损失 + 随机权重初始化                   | -3.69            |

*表 4：macro-score 变化是相对于使用三重损失和教师权重初始化训练的模型。*
