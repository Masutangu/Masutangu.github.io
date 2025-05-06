---
layout: post
date: 2025-5-6T18:29:39+31:00
title: 【论文笔记】REPLUG - Retrieval-Augmented Black-Box Language Models
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

本文是[《REPLUG: Retrieval-Augmented Black-Box Language Models》](https://arxiv.org/abs/2301.12652) 的笔记。

REPLUG 是一个检索增强的语言建模框架，将语言模型（LM）视为黑盒，并使用可调节的检索模型对其进行增强。与先前的检索增强 LM 不同，先前的方法训练语言模型使用特殊的交叉注意力机制来编码检索到的文本，而 REPLUG 只是简单地将检索到的文档添加到 LM 的输入之前。这种简单的设计可以轻松应用于任何现有的语言模型。此外，我们展示了 LM 可以用来监督检索模型，帮助检索模型检索出使 LM 做出更好预测的文档。代码已 [github.com/swj0419/REPLUG](https://github.com/swj0419/REPLUG) 上公开发布。

## Introduction

在这项工作中，我们介绍了 REPLUG（Retrieve and Plug），这是一个新的检索增强 LM 框架，其中语言模型被视为黑盒，检索组件被添加为可调节的即插即用模块。给定一个输入上下文，REPLUG 首先使用现成的检索模型从外部语料库中检索相关文档。检索到的文档被添加到输入上下文中，并输入到黑盒 LM 中进行最终预测。由于 LM 上下文长度限制了可以添加的文档数量，我们还采用了一个集成方案，该方案使用相同的黑盒语言模型并行编码检索到的文档，这使我们可以轻松地在计算和准确性之间进行权衡。

我们还介绍了 REPLUG LSR（REPLUG with LM-Supervised Retrieval），这是一种训练方案，可以通过来自黑盒语言模型的监督信号进一步改进 REPLUG 中的初始检索模型。关键思想是使检索器适应 LM，这与先前的工作（使语言模型适应检索器）相反。我们使用一个训练目标，该目标更倾向于**检索可以提高语言模型困惑度的文档**，同时将 LM 视为一个冻结的黑盒评分函数。

我们的工作是首次展示了检索对大型 LM（>100B模型参数）的益处，既可以降低 LM 的困惑度，又可以提高上下文学习性能。我们总结我们的贡献如下：

* 我们介绍了 REPLUG（§3），这是第一个用于增强黑盒 LM 的检索增强语言建模框架。与以往需要更新 LM 参数的方法不同，REPLUG 可以轻松地嵌入到任何现有的 LM 中，无需额外的微调。
* 我们提出了一个训练方案（§4），进一步将现成的检索模型适应 LM，使用语言建模分数作为监督信号，从而提高检索质量。
* 我们首次证明了检索可以使大规模、最先进的 LM 在语言建模（§6）和上下文学习任务中受益。评估结果显示，REPLUG 可以提高各种语言模型的性能，如 GPT、OPT 和 BLOOM，包括具有高达 175B 参数的非常大型模型。

## REPLUG

<img src="/assets/images/paper-note-replug/illustration-1.png" width="600" alt=""/>

如图 2 所示，给定一个输入上下文，REPLUG 首先使用检索器（§3.1）从外部语料库中检索一小组相关文档。然后，我们将每个检索到的文档与输入上下文进行串联，并并行地通过 LM，最后集成预测的概率（§3.2）。

### Document Retrieval

在给定输入上下文 $x$ 的情况下，检索器旨在从语料库 $D=\\{d_1...d_m\\}$ 中检索与 $x$ 相关的一小组文档。遵循先前的工作，我们使用基于双编码器架构的密集检索器，其中一个编码器用于对输入上下文 $x$ 和文档 $d$ 进行编码。具体而言，编码器通过对 $d$ 中的 token 进行最后隐藏表示的平均池化，将每个文档 $d\in D$ 映射到一个嵌入 $E(d)$。在查询时，相同的编码器应用于输入上下文 $x$，以获得查询嵌入 $E(x)$。通过它们的余弦相似度计算查询嵌入和文档嵌入之间的相似度：

$$
s(d, x) = cos(E(d), E(x)) 
$$

与输入 $x$ 具有最高相似度分数的前 $k$ 个文档被检索出来。为了高效检索，我们预先计算每个文档 $d\in D$ 的嵌入，并在这些嵌入上构建 FAISS 索引。

### Input Reformulation

检索到的前 $k$ 个文档提供了关于原始输入上下文 $x$ 的丰富信息，可能有助于 LM 做出更好的预测。将检索到的文档作为 LM 输入的一部分的一种简单方法是将 $x$ 与所有 $k$ 个文档一起放在前面。然而，考虑到语言模型的上下文窗口大小，这种简单的方案会受文档数量（即 $k$）的限制。为了解决这个限制，我们采用了如下的集成策略。假设 $D' ⊂ D$ 包含了与 $x$ 最相关的 $k$ 个文档。我们将 $D'$ 中的每个文档 $d ∈ D'$ 放在 $x$ 的前面，传递给 LM，然后从所有 $k$ 次传递中集成输出概率。形式上，给定输入上下文 $x$ 及其前 $k$ 个相关文档 $D'$，下一个 token $y$ 的输出概率为：

$$
p(y|x, D') = \sum_{d\in D'} p(y|d \circ x) \cdot \lambda(d,x)
$$

其中 $\circ$ 表示两个序列的连接，权重 $\lambda(d,x)$ 基于文档 $d$ 和输入上下文 $x$ 之间的相似度得分：

$$
λ(d, x) = \frac{e^{s(d, x)}}{\sum_{d \in D'} e^{s(d, x)}}
$$

## REPLUG LSR: Training the Dense Retriever

<img src="/assets/images/paper-note-replug/illustration-2.png" width="600" alt=""/>

我们进一步提出了 REPLUG LSR（REPLUG with LM-Supervised Retrieval），通过 LM 本身提供检索文档的监督信号，来调整 REPLUG 中的检索器。受 Sachan 等人（2022）的启发，我们的方法可以被看作是**调整检索到的文档的概率，以匹配语言模型输出序列困惑度的概率**。换句话说，我们希望检索器找到**使困惑度分数更低的文档**。如图 3 所示，我们的训练算法包括四个步骤：

* 检索文档并计算检索概率（§4.1）
* 通过语言模型对检索到的文档进行评分（§4.2）
* 通过最小化检索概率和 LM 得分分布之间的 KL 散度来更新检索模型参数（§4.3）
* 异步更新数据存储索引（§4.4）

### Computing Retrieval Likelihood

我们从语料库 $D$ 中检索出与输入上下文 $x$ 具有最高相似度得分的 $k$ 个文档 $D'\subset D$，然后计算每个检索到的文档 $d$ 的检索似然性：

$$
P_R(d|x) = \frac{e^{s(d, x)/\gamma}}{\sum_{d \in D'} e^{s(d, x)/\gamma}}
$$

其中 $\gamma$ 是控制 softmax 温度的超参数。理想情况下，检索似然性是通过对语料库 $D$ 中的所有文档进行边际化来计算的，但实际中不可行。因此，我们通过仅对检索到的文档 $D'$ 进行边际化来近似检索似然性。

### Computing LM likelihood

我们使用语言模型（LM）作为评分函数来衡量每个文档在多大程度上可以改善语言模型的困惑度。具体来说，我们首先计算 $P_{LM}(y\|d,x)$，即在给定输入上下文 $x$ 和文档 $d$ 的情况下，语言模型对真实输出 $y$ 的概率。概率越高，文档 $d_i$ 在改善语言模型困惑度方面的效果越好。然后我们计算每个文档 $d$ 的语言模型似然性如下：

$$
Q(d|x,y) = \frac{e^{P_{LM}(y|d,x)/\beta}}{\sum_{d \in D'} e^{P_{LM}(y|d,x)/\beta}}
$$

其中 $\beta$ 是另一个超参数。

### Loss Function

给定输入上下文 $x$ 和相应的 ground truth continuation $y$，我们计算检索似然性和语言模型似然性。密集检索器通过最小化这两个分布之间的 KL 散度进行训练：

$$
L = \frac{1}{|B|} \sum_{x \in B} KL\left( P_R(d|x) \parallel Q_{LM}(d|x,y) \right),
$$

其中 $B$ 是一组输入上下文。在最小化损失时，我们只更新检索模型的参数（LM 是黑盒）。

### Asynchronous Update of the Datastore Index

由于在训练过程中检索器的参数会被更新，因此之前计算的文档嵌入不再是最新的。按照 Guu 等人（2020）的做法，我们每隔 T 个训练步骤重新计算文档嵌入，并使用新的嵌入重建搜索索引。然后，我们使用新的文档嵌入和索引进行检索，并重复训练过程。