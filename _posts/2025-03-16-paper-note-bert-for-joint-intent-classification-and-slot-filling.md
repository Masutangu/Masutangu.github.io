---
layout: post
date: 2025-3-16T15:58:42+08:00
title: 【论文笔记】BERT for Joint Intent Classification and Slot Filling
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

本文是 [《BERT for Joint Intent Classification and Slot Filling》](https://arxiv.org/pdf/1902.10909) 的笔记。


在自然语言理解中，**意图分类（Intent Classification）**和**槽填充（Slot Filling）**是两个重要的任务。它们通常受限于规模较小的人工标注训练数据，导致泛化能力较差，特别是对于罕见的词汇。最近，一种新的语言表示模型 BERT 推动了在大规模未标注语料库上预训练，并在简单微调后为各种自然语言处理任务创建了最先进的模型。在这项工作中，我们提出了一种基于 BERT 的联合意图分类和槽填充模型。实验结果表明，与基于注意力的 RNN 模型和 slot-gated 模型相比，我们提出的模型在多个公共基准数据集上的**意图分类准确性**、**槽填充 F1 值**和**句子级语义框架准确性**方面取得了显著的改进。


## Introduction

近年来，各种智能音箱已经部署并取得了巨大的成功，例如Google Home、Amazon Echo、天猫精灵等，它们通过语音交互促进目标导向的对话，并帮助用户完成任务。自然语言理解（NLU）对于目标导向的口语对话系统的性能至关重要。NLU通常包括意图分类和槽填充任务，旨在为用户的话语形成语义解析。意图分类专注于预测查询的意图，而槽填充则提取语义概念。表1显示了用户查询“找一部史蒂文·斯皮尔伯格的电影”的意图分类和槽填充的示例。

NLU（Natural language understanding）通常包括意图分类和槽填充任务，旨在对用户的话语进行语义解析。**意图分类专注于预测查询的意图，而槽填充则提取语义概念**。表 1 显示了用户查询“找一部史蒂文·斯皮尔伯格的电影”的意图分类和槽填充的示例。

|**Query** | Find me a movie by Steven Spielberg|
|**Frame** |**Intent** find movie<br>**Slot** genre = movie, directed by = Steven Spielberg|

*表 1：意图分类和槽填充示例*

**意图分类是一个分类问题**，用于预测意图标签 $y^i$，而**槽填充是一个序列标注任务**，用于给输入的词序列 $x = (x_1, x_2,···, x_T)$ 打上槽标签序列 $y^s = (y^s_1, y^s_2,···, y^s_T)$。

由于 NLU 和其他 NLP 任务缺乏人工标注数据，导致泛化能力较差。本工作的技术贡献有两个方面：
* 我们探索了 BERT 预训练模型，以解决 NLU 的泛化能力不足；
* 我们提出了一种基于 BERT 的联合意图分类和槽填充模型，并证明与基于注意力的 RNN 模型和 slot-gated 模型相比，在意图分类准确性、槽填充 F1 值和句子级语义框架准确性上取得了显著的改进。

## Proposed Approach

首先，我们简要介绍 BERT 模型，然后介绍基于 BERT 的联合模型。图 1 展示了所提出模型的概括性视图。

<img src="/assets/images/paper-note-bert-for-joint-intent-classification-and-slot-filling/illustration-1.png" width="600" alt=""/>

* 图 1：输入的查询是“play the song little robin redbreast”

### BERT

BERT 模型的架构是基于原始 Transformer 模型的多层双向 Transformer 编码器。输入表示由 WordPiece 嵌入、位置嵌入和段落嵌入构成。一个特殊的分类嵌入（[CLS]）被插入为第一个标记，一个特殊的标记（[SEP]）被添加为最后一个标记。给定一个输入标记序列 $x = (x_1, . . . , x_T)$，BERT 的输出是 $\mathbf{H} = (\mathbf{h}_1, . . . , \mathbf{h}_T)$。

BERT 模型通过两种策略在大规模未标注文本上进行预训练：**掩码语言模型**（masked language model）和**下一个句子预测**（next sentence prediction）。预训练的 BERT 模型提供了强大的上下文相关句子表示，并可以通过微调适应各种目标任务。


### Joint Intent Classification and Slot Filling

BERT 可以很容易地扩展为一个联合的意图分类和槽填充模型。基于第一个特殊标记（[CLS]）的隐藏状态（表示为 $h_1$），可以预测出意图：

$$
y^i = softmax(\mathbf{W}^i\mathbf{h}_1 + \mathbf{b}^i),
$$

对于槽填充，我们将其他 token 的最终隐藏状态 $\mathbf{h}_2, ..., \mathbf{h}_T$ 输入到一个 softmax 层中，以对槽填充标签进行分类。为了使这个过程与 WordPiece 分词兼容，我们将每个分词后的输入词汇输入到 WordPiece 分词器中，并使用与第一个 sub-token 对应的隐藏状态作为输入传递给 softmax 分类器。

$$
y^s_n = softmax(\mathbf{W}^s\mathbf{h}_n + \mathbf{b}^s) , n \in 1 . . . N
$$

其中 $\mathbf{h}_n$ 是与单词 $x_n$ 的第一个 sub-token 对应的隐藏状态。为了联合建模意图分类和槽填充，目标函数被定义为：

$$
p(y^i, y^s|\mathbf{x}) = p(y^i|\mathbf{x})\prod^N_{n=1}p(y^s_n|\mathbf{x}),
$$

学习目标是最大化条件概率 $p(y^i, y^s\|\mathbf{x}) $。通过最小化交叉熵损失，模型进行端到端的微调。

### Conditional Random Field

槽标签的预测依赖于周围单词的预测。已经证明，结构化预测模型如 conditional random fields（CRF）可以提高槽填充的性能。在这里，我们研究了在联合 BERT 模型之上添加 CRF 来建模槽标签之间的依赖关系的有效性。

## Experiments and Analysis

我们在两个公开的基准数据集 ATIS 和 Snips 上评估了提出的模型。

### Data

ATIS 数据集是 NLU 研究中广泛使用的数据集，其中包括人们进行航班预订的音频记录。我们对两个数据集使用了与 Goo 等人（2018）相同的数据划分。训练集、开发集和测试集分别包含 4,478、500 和 893 个语句。训练集中有 120 个槽标签和 21 个意图类型。我们还使用了 Snips 数据集，该数据集是从 Snips 个人语音助手中收集的。训练集、开发集和测试集分别包含 13,084、700 和 700 个语句。训练集中有 72 个槽标签和 7 个意图类型。


### Training Details

我们使用英文的 uncased BERT-Base 模型，该模型有 12 层、768 个隐藏状态和 12 个头部。在微调过程中，所有超参数都在开发集上进行了调优。最大长度为 50。批量大小为 128。我们使用 Adam 进行优化，初始学习率为 5e-5，dropout 概率为 0.1，最大迭代次数从 [1, 5, 10, 20, 30, 40] 中选择。

### Results

实验结果表明，联合 BERT 模型在两个数据集上明显优于 baseline 模型。在 Snips 数据集上，联合 BERT 的意图分类准确率达到了 98.6%（从 97.0%），槽填充的 F1 值为 97.0%（从 88.8%），句子级语义框架准确率为 92.8%（从 75.5%）。在 ATIS 数据集上，联合 BERT 的意图分类准确率为 97.5%（从 94.1%），槽填充的 F1 值为 96.1%（从 95.2%），句子级语义框架准确率为 88.2%（从 82.6%）。联合 BERT+CRF 将 softmax 分类器替换为 CRF，其性能与 BERT 相当，可能是由于 Transformer 中的自注意机制已足够建模标签结构。

与 ATIS 相比，Snips 包含多个领域，并且具有更大的词汇量。对于更复杂的 Snips 数据集，联合 BERT 在句子级语义框架准确率上取得了很大的提升，从 75.5% 提高到 92.8%（相对提升 22.9%）。考虑到联合 BERT 模型是在来自不匹配领域和体裁（图书和维基百科）的大规模文本上进行预训练的，这表明其具有很强的泛化能力。在 ATIS 上，联合 BERT 在句子级语义框架准确率上也取得了显著提升，从 82.6% 提高到 88.2%（相对提升 6.8%）。


### Ablation Analysis and Case Study

我们在 Snips 上进行了消融分析，如表 3 所示。在没有联合学习的情况下，意图分类的准确率下降到 98.0%（从 98.6%），槽填充的 F1 值下降到 95.8%（从 97.0%）。我们还比较了联合 BERT 模型不同微调迭代次数。只进行 1 次迭代的联合 BERT 模型已经超过了 baseline 模型。

|Model | Epochs |Intent | Slot|
|-----| ---- | ----- | ---- |
|Joint BERT | 30 | 98.6 | 97.0 |
|No joint | 30 | 98.0 | 95.8 |
|Joint BERT | 40 | 98.3 | 96.4 |
|Joint BERT | 20 | 99.0 | 96.0 |
|Joint BERT | 10 | 98.6 | 96.5 |
|Joint BERT | 5 | 98.0 | 95.1 |
|Joint BERT | 1 | 98.0 | 93.3 |

*表 3：Snips数据集的消融分析。*