---
layout: post
date: 2024-11-23T15:39:55+08:00
title: 【论文笔记】Bag of Tricks for Efficient Text Classification
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

本文是 [《Bag of Tricks for Efficient Text Classification》](https://arxiv.org/pdf/1607.01759) 的笔记。

## Abstract

这篇论文探索了一种简单高效的文本分类基线。实验表明，我们的快速文本分类器 fastText 在准确性方面通常与深度学习分类器相当，而在训练和评估速度上快几个数量级。我们可以在不到十分钟的时间内使用标准多核 CPU 训练 fastText 处理超过十亿个单词，并在一分钟内对 312K 类别中的五十万个句子进行分类。

## 1 Introduction

文本分类是自然语言处理中的一个重要任务，具有许多应用，如网络搜索、信息检索、排名和文档分类。近年来，基于神经网络的模型越来越受欢迎。尽管这些模型在实践中取得了非常好的性能，但它们在训练和测试时间上往往相对较慢，限制了它们在非常大的数据集上的使用。

与此同时，**线性分类器**通常被认为是文本分类问题的强大基线（Joachims, 1998; McCallum and Nigam, 1998; Fan et al., 2008）。尽管它们很简单，但如果使用正确的特征，它们通常能够获得最先进的性能（Wang and Manning, 2012）。它们还有潜力扩展到非常大的语料库（Agarwal et al., 2014）。

在这项工作中，我们探讨了如何在文本分类的背景下，将这些基线**扩展到具有大规模输出空间的大语料库**。受到最近在**高效词表示学习**（efficient word representation
learning）方面的工作的启发（Mikolov et al., 2013; Levy et al., 2015），我们展示了具有**秩约束**（rank constraint）和**快速损失近似**（fast loss approximation）的线性模型可以在十分钟内在一亿个词上进行训练，同时达到与最先进水平相当的性能。我们在两个不同的任务上评估了我们方法 fastText 的质量，即标签预测和情感分析。

## 2 模型架构

句子分类的一个简单而高效的基线是将句子表示为**词袋**（bag of word BoW），并训练一个线性分类器，例如逻辑回归或支持向量机（SVM）。然而，**线性分类器在特征和类别之间不共享参数**。这可能在输出空间很大且某些类别具有非常少的示例的情况下限制了它们的泛化能力。解决这个问题的常见方法是**将线性分类器分解为低秩矩阵**（Schutze，1992；Mikolov等人，2013）或**使用多层神经网络**（Collobert和Weston，2008；Zhang等人，2015）。

一个具有秩约束的简单线性模型，可以通过最小化以下负对数似然来优化模型的训练：

$$
-\frac{1}{N}\sum_{n=1}^{N}y_{n}\log(f(BAx_n)),
$$

其中 $x_n$ 是第 $n$ 个文档的标准化特征包，$y_n$ 是标签，$A$ 和 $B$ 是权重矩阵。

<img src="/assets/images/paper-note-fasttext/illustration-1.png" width="600" alt=""/>


### 2.1 分层 Softmax

当类别数量很大时，计算线性分类器的计算成本很高。具体来说，计算复杂度为 $O(kh)$，其中 $k$ 是类别数量，$h$ 是文本表示的维度。为了提高我们的运行时间，我们使用了基于**霍夫曼编码树**（Huffman coding tree）的**分层 Softmax**（Goodman, 2001）。在训练期间，计算复杂度降低到 $O(h \log_2(k))$。

在测试时寻找最可能的类别时，分层 Softmax 也有优势。每个节点都与一个概率相关联，即从根节点到该节点的路径的概率。如果节点位于深度 $l + 1$，其父节点为 $n_1，...，n_l$，其概率为

$$P(n_{l+1})=\prod_{i=1}^{l}P(n_{i})$$

这意味着节点的概率总是低于其父节点的概率。**通过深度优先搜索树并跟踪叶子中的最大概率，我们可以丢弃与小概率的分支**。我们观察到测试时的复杂度降低到 $O(h \log_2(k))$。使用二叉堆计算 top T，可以进一步降低到 $O(\log(T))$。

### 2.2 N-Gram 特征

词袋模型对词语顺序不敏感，但显式地考虑顺序通常计算成本非常高。因此，我们**使用 n-gram 的集合**作为额外的特征来捕获一些关于局部词序的部分信息。这在实践中非常高效，同时达到了与显式使用顺序的方法（Wang and Manning, 2012）相当的结果。

我们通过使用**哈希技巧**（hashing trick）（Weinberger et al., 2009）来维护 n-gram 的高效且内存高效的映射，使用的哈希函数与 Mikolov et al. (2011) 中的相同。

## 4 讨论与结论

在这项工作中，我们提出了一种简单的文本分类基线方法。与 word2vec 中无监督训练的词向量不同，我们的词特征可以平均在一起形成良好的句子表示。在几项任务中，fastText 在性能上与最近受深度学习启发的方法相当，同时速度要快得多。
