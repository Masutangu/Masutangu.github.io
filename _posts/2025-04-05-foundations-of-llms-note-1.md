---
layout: post
date: 2025-4-5T15:58:10+08:00
title: 【读书笔记】Foundations-of-LLMs 语言模型基础
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

本文是[《Foundations-of-LLMs》](https://github.com/ZJU-LLMs/Foundations-of-LLMs) 第一章【语言模型基础】的笔记。

## 基于统计方法的语言模型

n-grams 语言模型是在 **n 阶马尔可夫假设**下，对语料库中出现的长度为 n 的词序列出现概率的**极大似然估计**。

n-grams 语言模型通过依次统计文本中的 n-gram 及其对应的 (n-1)-gram 在语料库中出现的相对频率来计算文本 $w_{1:N}$ 出现的概率。计算公式如下：

$$
P_{n-grams}(w_{1:N}) = \prod_{i=n}^N\frac{C(w_{i-n+1:i})}{C(w_{i-n+1:i-1})}
$$

其中 $C(w_{i-n+1:i})$ 为词序列 $\\{w_{i-n+1},..., w_i\\}$ 在语料库中出现的次数。$C(w_{i-n+1:i-1})$ 同理。 

接下来证明 $\frac{C(w_{i-n+1:i})}{C(w_{i-n+1:i-1})}$ 近似 $P(w_i\|w_{i-n:i-1})$。以 bigrams 为例，假设语料库中共涵盖 $M$ 个不同的单词，$\\{w_i, w_j\\}$ 出现的概率为 $P(w_i, w_j)$, 对应出现的次数为 $C(w_i, w_j)$，则其似然函数为：

$$
L(\theta) = \prod_{i=1}^M\prod_{j=1}^MP(w_i, w_j)^{C(w_i, w_j)}
$$

根据条件概率公式 $P(w_i, w_j) = P(w_j\|w_i)P(w_i)$，有：

$$
L(\theta) = \prod_{i=1}^M\prod_{j=1}^MP(w_j|w_i)^{C(w_i, w_j)} P(w_i)^{C(w_i, w_j)}
$$

其对应的对数似然函数为：

$$
L_{\log}(\theta) = \sum_{i=1}^M\sum_{j=1}^MC(w_i, w_j)\log P(w_j|w_i) + \sum_{i=1}^M\sum_{j=1}^MC(w_i, w_j)\log P(w_i)
$$

因为 $\sum_{j=1}^MP(w_j\|w_i) = 1$，所以最大化对数似然函数可建模为如下的约束优化问题：

$$
\max L_{\log}(\theta)~\text{s.t.}~\sum_{j=1}^M P(w_j|w_i) = 1~\text{for}~i\in [1, M]
$$

对其求关于 $P(w_j\|w_i)$ 的偏导，可得：

$$
\frac{\partial L(\lambda, L_{log})}{\partial P(w_j|w_i)} = \sum_{i=1}^M\frac{C(w_i, w_j)}{P(w_j|w_i)} + \sum_{i=1}^M\lambda_i
$$

当导数为 0 时，有：

$$
P(w_j|w_i) = -\frac{C(w_i, w_j)}{\lambda_i}
$$

因 $\sum_{j=1}^M P(w_j\|w_i) = 1$，$\lambda_i$ 可取值为 $-\sum_{j=1}^M C(w_i, w_j)$，即

$$
P(w_j|w_i) = \frac{C(w_i, w_j)}{\sum_{j=1}^M C(w_i, w_j)} = \frac{C(w_i, w_j)}{C(w_i)}
$$

上述分析表明 bigram 语言模型中的 $\frac{C(w_i, w_j)}{C(w_i)}$ 是对语料库中的长度为 2 的词序列的 $P(w_j\|w_i)$ 的极大似然估计。该结论可扩展到 n > 2 的其他 n-grams 语言模型模型中。

**n 阶马尔可夫假设**：对序列 ${w_1, w_2, w_3, ..., w_N}$，当前状态 $w_N$ 出现的概率只与前 n 个状态 ${w_{N−n}, ..., w_{N−1}}$ 有关，即：

$$
P(w_N|w_1, w_2, ..., w_{N-1}) \approx P(w_N|w_{N-n}, ..., w_{N-1})
$$

设文本 $w_{1:N}$ 出现的概率为 $P(w_{1:N})$。根据条件概率的链式法则，$P(w_{1:N})$ 可由下式进行计算：

$$
P(w_{1:N}) = P(w_1)P(w_2|w_1)P(w_3|w_{1:2})...P(w_N|w_{1:N-1}) = \prod_{i=1}^N P(w_i|w_{1:i-1})
$$

根据 n 阶马尔可夫假设，$P(w_i\|w_{i-n:i-1})$ 近似 $P(w_i\|w_{1:i-1})$，同时 $\frac{C(w_{i-n+1:i})}{C(w_{i-n+1:i-1})}$ 近似 $P(w_i\|w_{i-n:i-1})$，可得：

$$
\begin{aligned}
P_{n-grams}(w_{1:N}) &= \prod_{i=n}^N\frac{C(w_{i-n+1:i})}{C(w_{i-n+1:i-1})} \\
&\approx \prod_{i=n}^N P(w_i|w_{i-n:i-1}) \\
&\approx \prod_{i=n}^N P(w_i|w_{1:i-1}) \\
&= P(w_{1:N})
\end{aligned}
$$

即 $P_{n-grams}(w_{1:N}) \approx P(w_{1:N})$。

n-grams 语言模型通过统计词序列在语料库中出现的频率来预测语言符号的概率。其对未知序列有一定的泛化性，但也容易陷入“零概率”的困境。

## 基于 RNN 的语言模型

训练 RNN 时，涉及大量的矩阵联乘操作，容易引发梯度衰减或梯度爆炸问题。具体分析如下：

设 RNN 语言模型的训练损失为：

$$
L= L(x, o, W_I, W_H, W_O) = \sum_{i=1}^tl(o_i, y_i)
$$

其中 $l$ 为损失函数，$y_i$ 为标签。损失 $L$ 关于参数 $W_H$ 的梯度为：

$$
\frac{\partial L}{W_H} = \sum_{i=1}^t \frac{\partial l_t}{\partial o_t}\frac{\partial o_t}{\partial h_t}\frac{\partial h_t}{\partial h_i}\frac{\partial h_i}{\partial W_H}
$$

其中

$$
\frac{\partial h_t}{\partial h_i} = \frac{\partial h_t}{\partial h_{t-1}}\frac{\partial h_{t-1}}{\partial h_{t-2}}...\frac{\partial h_{i+1}}{\partial h_i} = \prod_{k=i+1}^t\frac{\partial h_k}{\partial h_{k-1}}
$$

且 $\frac{\partial h_k}{\partial h_{k-1}} = \frac{\partial g(z_k)}{\partial z_k}W_H$。综上可得：

$$
\frac{\partial L}{W_H} = \sum_{i=1}^t \frac{\partial l_t}{\partial o_t}\frac{\partial o_t}{\partial h_t}\prod_{k=i}^t\frac{\partial g(z_k)}{\partial z_k}W_H \frac{\partial h_i}{\partial W_H}
$$

从上式中可以看出，求解 $W_H$ 的梯度时涉及大量的矩阵级联相乘。这会导致其数值被级联放大或缩小。当 $W_H$ 的最大特征值小于 1 时，会发生梯度消失；当 $W_H$ 的最大特征值大于 1 时，会发生梯度爆炸。梯度消失和爆炸导致训练 RNN 非常困难。另外由于 RNN 模型循环迭代的本质，不易进行并行计算，导致其在输入序列较长时，训练较慢。

## 基于 Transformer 的语言模型

Transformer 是由两种模块组合构建的模块化网络结构。两种模块分别为：
* 注意力（Attention）模块
* 全连接前馈（Fully-connected Feedforwad）模块

其中，自注意力模块由自注意力层（Self-Attention Layer）、残差连接（Residual Connections）和层正则化（Layer Normalization）组成。全连接前馈模块由全连接前馈层，残差连接和层正则化组成。

全连接前馈层占据了 Transformer 近三分之二的参数，可以看作是一种 Key-Value 模式的记忆存储管理模块（[《Transformer Feed-Forward Layers Are Key-Value Memories》](https://arxiv.org/pdf/2012.14913)）

引入残差连接可以有效解决梯度消失问题。将层正则化置于残差连接之后的网络结构被称为 Post-LN Transformer。与之相对的，还有一种将层正则化置于残差连接之前的网络结构，称之为 Pre-LN Transformers。Post-LN Transformer 应对表征坍塌（Representation Collapse）的能力更强，但处理梯度消失略弱。而 Pre-LN Transformers 可以更好的应对梯度消失，但处理表征坍塌的能力略弱（[《ResiDual: Transformer with Dual Residual Connections》](https://arxiv.org/abs/2304.14802)）。

## 语言模型的采样方法

### 概率最大化方法

#### 贪心搜索（Greedy Search）

贪心搜索在在每轮预测中都选择概率最大的词，贪心搜索容易陷入局部最优，难以达到全局最优解。

#### 波束搜索（Beam Search）

波束搜索在每轮预测中都先保留 $b$ 个可能性最高的词 $B_i = \\{w_{i+1}^1, w_{i+1}^2, ..., w_{i+1}^b\\}$。

### 随机采样方法

为了增加生成文本的多样性，随机采样的方法在预测时增加了随机性。在每轮预测时，其先选出一组可能性高的候选词，然后按照其概率分布进行随机采样，采样出的词作为本轮的预测结果。在采样方法中加入 Temperature 机制可以对候选词的概率分布进行调整。

#### Top-K 采样

Top-K 采样在每轮预测中都选取 K 个概率最高的词作为本轮的候选词集合，然后对这些词的概率用 softmax 函数进行归一化得到分布函数，根据该分布采样出本轮的预测的结果。

然而，当候选词的分布的方差较大的时候，可能会导致本轮预测选到**概率较小、不符合常理的词**，从而产生“胡言乱语”。而当候选词的分布的方差较小的时候，甚至趋于均匀分布时，固定尺寸的候选集中**无法容纳更多的具有相近概率的词，导致候选集不够丰富**。

#### Top-P 采样

为了解决上述问题，我们可以使用 Top-P 采样，也称 Nucleus 采样。其设定阈值 $p$ 来对候选集进行选取。候选集可表示为 $S_p = \\{w_{i+1}^1, w_{i+1}^2, ..., w_{i+1}^{\|S_p\|}\\}$，其中，对 $S_p$ 有，$\sum_{w \in S_p} o_i[w] \geq p$。

应用阈值作为候选集选取的标准之后，**Top-P 采样可以避免选到概率较小、不符合常理的词**，从而减少“胡言乱语”。并且，其还可以容纳更多的具有相近概率的词，增加文本的丰富度。

#### Temperature 机制

在不同场景中，我们对于随机性的要求可能不一样。比如在开放文本生成中更倾向于生成更具创造力的文本，所以需要采样具有更强的随机性。而在代码生成中希望生成的代码更为保守，所以需要较弱的随机性。引入 Temperature 机制可以对解码随机性进行调节。Temperature 机制通过对 Softmax 函数中的自变量进行尺度变换，然后利用 Softmax 函数的非线性实现对分布的控制。

设 Temperature 尺度变换的变量为 $T$。当 $T > 1$ 时，Temperature 机制会使得候选集中的词的概率差距减小，**分布变得更平坦**，从而增加随机性。当 $0 < T < 1$ 时，Temperature 机制会使得候选集中的元素的概率差距加大。

## 语言模型的评测

评测语言模型生成能力的方法可以分为两类。第一类方法不依赖具体任务，直接通过语言模型的输出来评测模型的生成能力，称之为**内在评测（Intrinsic Evaluation）**。第二类方法通过某些具体任务，如机器翻译、摘要生成等，来评测语言模型处理这些具体生成任务的能力，称之为**外在评测（Extrinsic Evaluation）**。

### 内在评测

在内在评测中，测试文本通常由与预训练中所用的文本独立同分布的文本构成，不依赖于具体任务。最为常用的内部评测指标是困惑度（Perplexity）。由于测试文本和预训练文本同分布，预训练文本代表了我们想要让语言模型学会生成的文本，如果语言模型在这些测试文本上越不“困惑”，则说明语言模型越符合我们对其训练的初衷。因此，困惑度可以一定程度上衡量语言模型的生成能力。

困惑度可以改写成如下等价形式：

$$
\text{PPL}(s_{test}) = \exp (-\frac{1}{N}\sum_{i=1}^N \log P(w_i|w_{<i}))
$$

其中 $-\frac{1}{N}\sum_{i=1}^N \log P(w_i\|w_{<i})$ 可以看作是生成模型生成的词分布与测试样本真实的词分布间的交叉熵，因为 $P(w_i\|w_{<i}) \leq 1$，所以此交叉熵是生成模型生成的词分布的信息熵的上界，即：

$$
-\frac{1}{N}\sum_{i=1}^N P(w_i|w_{<i}) \log P(w_i|w_{<i}) \leq -\frac{1}{N}\sum_{i=1}^N \log P(w_i|w_{<i})
$$

困惑度减小也意味着熵减少，模型“胡言乱语”的可能性降低。

### 外在评测

外在
评测方法通常可以分为基于统计指标的评测方法和基于语言模型的评测方法两类。

#### 基于统计指标的评测

基于统计指标的方法构造统计指标来评测语言模型的输出与标准答案间的契合程度，并以此作为评测语言模型生成能力的依据。**BLEU（BiLingual Evaluation Understudy）** 和 **ROUGE（Recall-Oriented Understudy for Gisting Evaluation）** 是应用最为广泛的两种统计指标。其中，BLEU 是精度导向的指标，而 ROUGE 是召回导向的指标。

**BLEU 被提出用于评价模型在机器翻译（Machine Translation, MT）任务上的效果。**其**在词级别上计算生成的翻译与参考翻译间的重合程度**。具体地，BLEU 计算多层次 n-gram 精度的几何平均。

**ROUGE 被提出用于评价模型在摘要生成（Summarization）任务上的效果。**常用的 ROUGE 评测包含 ROUGE-N, ROUGE-L, ROUGE-W, 和 ROUGE-S 四种。其中，ROUGE-N 是基于 n-gram 的召回指标，ROUGE-L 是基于最长公共子序列（Longest Common Subsequence, LCS）的召回指标。ROUGE-W 是在 ROUGE-L 的基础上，引入对 LCS 的加权操作后的召回指标。ROUGE-S 是基于 Skip-bigram 的召回指标。

#### 基于语言模型的评测

目前基于语言模型的评测方法主要分为两类：
* 基于上下文词嵌入（Contextual Embeddings）的评测方法
* 基于生成模型的评测方法

基于上下文词嵌入典型的评测方法是 BERTScore。BERTScore 在 BERT 的上下文词嵌入向量的基础上，计算生成文本 $s_{gen}$ 和参考文本 $s_{ref}$ 间的相似度来对生成样本进行评测。BERTScore 从精度（Precision），召回（Recall）和 F1 量度三个方面对生成文档进行评测。相较于统计评测指标，BERTScore 更接近人类评测结果。但是，BERTScore 依赖于人类给出的参考文本。

基于生成模型典型的评测方法是 G-EVAL。与 BERTScore 相比，G-EVAL 无需人类标注的参考答案，可以更好的适应到缺乏人类标注的任务中。G-EVAL 通过提示工程（Prompt Engineering）引导 GPT-4 输出评测分数。

除 G-EVAL 外，近期还有多种基于生成模型的评测方法被提出。其中典型的有 InstructScore ，其除了给出数值的评分，还可以给出对该得分的解释。