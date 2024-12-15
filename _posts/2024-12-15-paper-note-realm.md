---
layout: post
date: 2024-12-15T13:42:19+08:00
title: 【论文笔记】REALM-Retrieval-Augmented Language Model Pre-Training
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

本文是[《REALM-Retrieval-Augmented Language Model Pre-Training》](https://arxiv.org/abs/2002.08909)的笔记。

## 摘要

语言模型预训练已被证明能够捕捉到大量世界知识，然而这些知识隐式地存储在神经网络的参数中，需要越来越大的网络来覆盖更多的事实。为了以**更模块化**和**可解释**的方式捕捉知识，我们通过一个潜在的**知识检索器**来增强语言模型预训练，该检索器允许模型在预训练、微调和推理期间从大型语料库（如维基百科）中检索和关注文档。
我们首次展示了如何以**无监督的方式预训练这样的知识检索器**，使用掩码语言建模作为学习信号，并通过考虑数百万文档的检索步骤进行反向传播（backpropagating through a retrieval step that considers millions of documents）。
我们通过在具有挑战性的开放领域问答（Open-QA）任务上进行微调，证明了**检索增强语言模型预训练（Retrieval-Augmented Language Model，REALM）**的有效性。我们在三个流行的 Open-QA 基准测试上与当前最先进的显式和隐式知识存储模型进行了比较，发现我们的方法在绝对准确率上显著优于所有之前的方法（4-16%），同时还提供了可解释性和模块化等优势。


## 1. Introduction

最近的语言模型预训练进展表明，像 BERT、RoBERTa 和 T5 这样的模型能够从它们训练的大量文本语料库中获取惊人的世界知识。在这些语言模型中，学到的世界知识被**隐式**地存储在底层神经网络的参数中。这使得很难确定网络中存储了什么知识以及存储在哪里。
此外，**存储空间受限于网络的大小**——要捕获更多的世界知识，就必须训练越来越大的网络，这可能会非常缓慢或昂贵。

为了以更具解释性和模块化的方式捕获知识，我们提出了一个新的框架，即检索增强语言模型（REALM）预训练，它通过引入一个学习到的**文本知识检索器**来增强语言模型预训练算法。与将知识存储在其参数中的模型不同，这种方法**显式**地暴露了世界知识的作用，要求模型在推理过程中决定要检索和使用什么知识。

在进行每次预测之前，语言模型使用检索器从大型语料库（如维基百科）中检索文档，然后关注这些文档以帮助其预测。端到端学习这个模型需要通过一个考虑整个文本知识语料库的检索步骤进行反向传播，如图 1 所示。

<img src="/assets/images/paper-note-realm/illustration-1.png" width="600" alt=""/>

REALM 的关键直觉是使用来自无监督文本的基于性能的信号来训练检索器：**一个改善了语言模型困惑度的检索是有帮助的，应该得到奖励，而一个无信息的检索应该受到惩罚**。例如，在图 1 中，如果模型需要填充 *"the __ at the top of the pyramid"* 中的空白，检索器应该因为选择了一个包含 *"The pyramidion on top allows for less material higher up the pyramid"* 的文档而受到奖励。我们通过将**检索然后预测**的方法建模为一个**潜在变量**（latent variable ）语言模型并**优化边际似然**（marginal likelihood）来实现。

在预训练期间引入大规模的神经检索模块构成了一个重大的计算挑战，因为检索器必须考虑每个预训练步骤的数百万候选文档，并且我们必须通过其决策进行反向传播。为了解决这个问题，我们将检索器结构化，使每个文档的计算可以被缓存并异步更新，并且最佳文档的选择可以被构建为最大内积搜索（MIPS）。

许多先前的工作已经证明了将离散检索步骤添加到神经网络中的好处，但没有将框架应用于语言模型预训练，而是使用非学习的检索器来处理大规模文档集合。

## 2. 背景

### 语言模型预训练

语言模型预训练的目标是从未标记的文本语料库中学习有用的语言表示。然后可以对预训练的模型进行进一步的训练（微调），用于主要感兴趣的下游任务（在我们的案例中是 Open-QA），通常比从头开始训练具有更好的泛化能力。MLM 被训练来预测输入文本段落中缺失的 token。给定一个未标记的预训练语料库 X（例如，维基百科文本），可以通过在采样的文本片段中随机屏蔽 token 来生成训练样本（x, y）（例如，x = "The [MASK] is the currency [MASK] the UK"; y = ("pound", "of")）。模型使用其对屏蔽输入 x 的表示来预测每个 mask 中应填入的 token。

一个好的 MLM 必须学会编码句法和语义信息（例如，预测 "of"）以及一些世界知识（例如，预测 "pound"）。

### 开放领域问答（Open-QA）

为了衡量模型结合世界知识的能力，我们需要一个世界知识至关重要的下游任务。也许自然语言处理中最知识密集的任务之一是开放领域问答（Open-QA）：给定一个问题 x，如 "What is the currency of the UK?"，模型必须输出正确的答案字符串 y，"pound"。OpenQA 的 "开放" 部分是指**模型不会收到一个预先确定的已知包含答案的文档**，这与传统的阅读理解（reading comprehension，RC）任务（如 SQuAD）不同。RC 模型只需理解单个文档，而 Open-QA 模型必须保留来自数百万文档的知识，因为问题可能涉及其中任何一个。

我们关注的是利用文本知识语料库 Z 作为知识源的 Open-QA 系统。这些系统中的许多采用基于检索的方法：给定一个问题 x，从语料库 Z 中检索潜在相关的文档 z，然后从文档中提取答案 y。我们的方法 REALM 受此范式启发，并将其扩展到语言模型预训练。另外，一些最近的工作提出了*生成式*系统，这些系统应用序列到序列模型在 x 上直接逐个 token 生成 y。在我们的实验中，我们将与这两种范式的最先进系统进行比较。

## 3. 方法

整体框架如图 2 所示：

<img src="/assets/images/paper-note-realm/illustration-2.png" width="600" alt=""/>

*图 2：REALM 的整体框架。左图：**无监督预训练**。知识检索器和知识增强编码器在无监督语言建模任务上联合预训练。右图：**监督微调**。 在检索器（θ）和编码器（φ）的参数预训练完成后，它们将在主要关注的任务上进行监督微调，使用监督示例。*


### 3.1. Realm 的生成过程

对于预训练和微调，REALM 都会接受输入 $x$，并学习可能输出 $y$ 的分布 $p(y\|x)$。在预训练阶段，任务是掩码语言建模：$x$ 是来自预训练语料库 X 的一个句子，其中一些 token 被掩盖，模型必须预测这些缺失 token 的值 $y$。在微调阶段，任务是开放问答（Open-QA）：$x$ 是一个问题，$y$ 是答案。

REALM 将 $p(y\|x)$ 分解为两个步骤：**检索**，然后**预测**。给定一个输入 $x$，我们首先从知识语料库 $Z$ 中检索可能有用的文档 $z$。我们将这建模为一个从分布 $p(z\|x)$ 中抽样的过程。然后，我们基于检索到的 $z$ 和原始输入 $x$ 来生成输出 $y$——建模为 $p(y\|z, x)$。为了获得生成 $y$ 的整体可能性，我们将 $z$ 视为一个潜在变量，并对所有可能的文档 $z$ 进行边际化（marginalize），得到

$$
p(y\,|\,x)=\sum_{z\in \mathcal Z}p(y\,|\,z,x)\,p(z\,|\,x). \ \ (1)
$$

### 3.2 模型架构

模型架构包含两个关键组件：**知识检索器**，用于建模 $p(z\|x)$，以及**知识增强编码器**，用于建模 $p(y\|z, x)$。

#### 知识检索器

知识检索器使用密集内积（dense inner product）模型定义：

$$
\begin{aligned}
p(z\,|\,x)&=\frac{\exp f(x,z)}{\sum_{z^{\prime}}\exp f(x,z^{\prime})}, \\ f(x,z)&=\text{Embed}_{\text{input}}(x)^{\top}\text{Embed}_{\text{doc}}(z)
\end{aligned}
$$

其中，$Embed_{input}$ 和 $Embed_{doc}$ 分别是将 $x$ 和 $z$ 映射到 $d$ 维向量的嵌入函数。

$x$ 和 $z$ 之间的**相关性得分（relevance score）** $f(x, z)$ 定义为向量嵌入的内积。检索分布是对所有相关性分数进行 softmax 操作得到的。

我们使用 BERT 风格的 Transformer 实现嵌入函数。按照标准做法，我们对文本应用 wordpiece tokenization，用 $[\texttt{SEP}]$ 标记分隔它们，以 $[\texttt{CLS}]$ 标记为前缀，并在最后附加一个 $[\texttt{SEP}]$ 标记。

$$
\begin{aligned}
\mathrm{join}_{\texttt{BERT}}(x)&=[\texttt{CLS}]\,x\,[\texttt{SEP}] \\
\mathrm{join}_{\texttt{BERT}}(x_{1},x_{2})&=[\texttt{CLS}]\,x_{1}\,[\texttt{SEP}]\,x_{2}\,[\texttt{SEP}]
\end{aligned}
$$

我们将其输入 Transformer，该 Transformer 为每个 token 生成一个向量，包括对应于 $[\texttt{CLS}]$ 的向量，该向量被当作序列的“池化”表示（表示为 $BERT_{CLS}$）。最后，我们执行线性投影以减少向量的维度，表示为投影矩阵 $W$：

$$
\begin{aligned}
\text{Embed}_{input}(x) &= W_{\text{input}}\text{BERT}_{\text{CLS}}(\text{join}_{\texttt{BERT}}(x)) \quad \\ 
\text{Embed}_{doc}(z) &= W_{\text{doc}}\text{BERT}_{\text{CLS}}(\text{join}_{\texttt{BERT}}(z_{\text{title}}, z_{\text{body}}))
\end{aligned}
$$

其中 $z_{title}$ 是文档的标题，$z_{body}$ 是其正文。我们让 $\theta$ 表示与检索器相关的所有参数，包括 Transformer 和投影矩阵。

#### 知识增强编码器

给定输入 $x$ 和检索到的文档 $z$，知识增强编码器定义 $p(y\|z, x)$。我们将 $x$ 和 $z$ 连接到一个序列中，然后将其输入到 Transformer（与检索器中使用的 Transformer 不同）。这允许我们在预测 $y$ 之前在 $x$ 和 $z$ 之间执行丰富的交叉注意力。具体示例见图 1。

在这个阶段，预训练和微调的架构略有不同。

**对于掩码语言模型预训练任务，我们必须预测 $x$ 中每个 $\texttt{[MASK]}$ token 的原始值**。为此，我们使用与 BERT 相同的掩码语言建模（MLM）损失：

$$
\begin{aligned}
p(y\,|\,z,x)&=\prod_{j=1}^{J_{x}}p(y_{j}\,|\,z,x) \quad \\ 
p(y_{j}\,|\,z,x)&\propto\exp\left(w_{j}^{\top}\text{BERT}_{\text{MASK}(j)}\big{(}\text{join}_{\text{BERT}}\big{(}x,\,z_{\text{body}}\big{)}\big{)}\right).
\end{aligned}
$$

其中 $BERT_{MASK(j)}$ 表示对应于第 $j$ 个掩码 token 的 Transformer 输出向量，$J_x$ 是 $x$ 中 $[\texttt{MASK}]$ token 的总数，$w_j$ 是 token $y_j$ 的词嵌入。

**对于 Open-QA 微调，我们希望生成答案字符串 $y$**。遵循之前的阅读理解工作，我们假设答案 $y$ 可以在某个文档 $z$ 中找到作为连续 token 序列。设 $S(z, y)$ 是 $z$ 中匹配 $y$ 的 span 集合。然后我们可以定义 $p(y\| z, x)$ 如下：

$$
\begin{aligned}
p(y\,|\,z,x)&\propto\sum_{s\in S(z,y)}\exp\left(\text{MLP}\left(\left[h_{\text{START}(\mathbf{s})};h_{\text{END}(\mathbf{s})}\right]\right)\right) \\
h_{\text{START}(\mathbf{s})}&=\text{BERT}_{\text{START}(\mathbf{s})}(\text{join}_{\text{BERT}}(x,z_{\text{body}})) \\
h_{\text{END}(\mathbf{s})}&=\text{BERT}_{\text{END}(\mathbf{s})}(\text{join}_{\text{BERT}}(x,z_{\text{body}})),
\end{aligned}
$$

其中 $BERT_{START(s)}$ 和 $BERT_{END(s)}$ 分别表示 span $s$ 的开始和结束 token 对应的 Transformer 输出向量，而 MLP 表示前馈神经网络。我们将 φ 表示与知识增强编码器相关的所有参数。

### 3.3 训练

在预训练和微调阶段，我们的训练目标是最大化正确输出 $y$ 的对数似然 $log p(y\| x)$。由于知识检索器和知识增强编码器都是可微分的神经网络，我们可以计算相对于模型参数 $θ$ 和 $φ$ 的对数似然 $log p(y\| x)$ 的梯度，并使用随机梯度下降进行优化。

计算的关键挑战在于边缘概率 $p(y\| x) = \sum_{z∈Z} p(y\| x, z)p(z\| x)$ 需要对知识语料库 Z 中的所有文档 $z$ 进行求和。为了近似计算，我们改为**对概率 $p(z\| x)$ 最高的前 $k$ 个文档进行求和**，这在大多数文档的概率接近零的情况下是合理的。
即使使用了这个近似，我们仍然需要一种高效的方法来找到前 k 个文档。注意，文档概率 $p(z\| x)$ 大小的排序与相关性得分 $f(x, z) = Embed_{input}(x)^\top Embed_{doc}(z)$ 相同，即 $Embed_{input}(x)$ 和 $Embed_{doc}(z)$ 的内积。因此，我们可以使用**最大内积搜索（Maximum Inner Product Search，MIPS）**算法来找到近似的前 $k$ 个文档。

为了使用 MIPS，我们需要预先计算每个 $z ∈ Z$ 的 $Embed_{doc}(z)$ 并构建一个高效的搜索索引来存储这些嵌入。然而，如果 $Embed_{doc}$ 的参数 $θ$ 在之后进行了更新，那么这个数据结构将不再与 $p(z\| x)$ 保持一致。因此，在每次对 $θ$ 进行梯度更新后，搜索索引就会变得“过时”。我们的解决方案是每隔几百个训练步骤之后异步地重新计算嵌入和重新索引所有文档。
在检索到这些文档后，我们使用最新的 $θ$ 重新计算 $p(z\| x)$ 及其梯度。我们通过实验证明，只要刷新频率足够频繁，这个过程可以实现稳定的优化。

<img src="/assets/images/paper-note-realm/illustration-3.png" width="600" alt=""/>

*图3：REALM预训练与异步MIPS刷新。*

虽然异步刷新可以用于预训练和微调，但在我们的实验中，我们只在预训练阶段使用它。对于微调，简单起见，我们只构建一次 MIPS 索引（使用预训练的 $θ$），并且不更新 $Embed_{doc}$。请注意，我们仍然对 $Embed_{input}$ 进行微调，因此检索函数仍然会从查询侧进行更新。

#### What does the retriever learn?

由于 REALM 的知识检索是潜在的，训练目标如何鼓励有意义的检索并不明显。在这里，我们展示了如何奖励能提高预测准确性的检索结果。

对于给定的查询 $x$ 和文档 $z$，回想一下 $f(x, z)$ 是知识检索器分配给文档 $z$ 的“相关性得分”。我们可以通过分析相对于知识检索器参数 $θ$ 的梯度来了解 REALM 预训练过程中单步梯度下降如何改变这个得分：

$$
\begin{aligned}
\nabla \log p(y|x) &= \sum_{z\in\mathcal Z}r(z)\nabla f(x,z) \\ 
r(z) &= [\frac{p(y|z,x)}{p(y|x)} - 1]p(z|x)
\end{aligned}
$$

对于每个文档 $z$，梯度鼓励检索器通过 $r(z)$ 改变得分 $f(x, z)$：如果 $r(z)$ 是正数，则得分增加；如果 $r(z)$ 是负数，则得分减少。乘数 $r(z)$ 是正数当且仅当 $p(y\| z, x) > p(y\| x)$。其中，$p(y\| z, x)$ 是在使用文档 $z$ 进行预测时正确输出 $y$ 的概率。而 $p(y\| x)$ 是从 $p(z\| x)$ 随机抽样一个文档时 $p(y\| x, z)$ 的期望值。因此，只要文档 $z$ 的表现优于预期，它就会获得正向更新。

### 3.4 预训练中的归纳偏见注入

在开发 REALM 的过程中，我们发现了几种进一步引导模型实现有意义检索的策略：
- **显著跨度（salient span）掩码**：在 REALM 预训练过程中，我们希望关注那些需要世界知识来预测掩码标记的示例 $x$。
- **空文档**：即使进行了显著跨度掩码，也不是所有的掩码 token 都需要世界知识来预测。我们通过添加一个空的 *null document* 到前 $k$ 个检索到的文档中，允许在没有必要检索时适当分配权重。
- **禁止简单检索**：若预训练语料库 $X$ 和知识语料库 $Z$ 相同，如果屏蔽的句子 $x$ 来自文档 $z$，通过查看 $z$ 中未屏蔽版本的 $x$，知识增强编码器可以轻松预测 $y$。为了避免这种情况，我们在预训练期间排除这个候选。
- **冷启动**：在训练开始时，如果检索器没有良好的 $Embed_{input}(x)$ 和 $Embed_{doc}(z)$ 嵌入，检索到的文档 $z$ 可能与 $x$ 无关。为了避免这种冷启动问题，我们使用逆填空任务 (ICT) 来预热 $Embed_{input}$ 和 $Embed_{doc}$，并使用 BERT 预训练来预热知识增强编码器。


## 4. 实验

### 4.2. 比较方法

#### 基于检索的开放问答（Open-QA）

大多数现有的开放问答（Open-QA）系统通过首先从知识语料库中检索可能相关的文档，然后使用阅读理解系统从文档中提取答案来回答问题。在这种范式中，知识被**显式**地存储在语料库中。我们希望比较实现检索的不同方法。

许多方法使用非学习的启发式检索，如**稀疏词袋匹配**（sparse bag-of-words matching）或**基于问题的实体链接**（entity linking on the question ）来选择一组相关的文档。之后使用学习模型对这些文档重新排序，但覆盖率可能受到初始启发式检索步骤的限制。

还有一些最近的方法提出使用 MIPS 索引实现可学习的检索。ORQA 使用与 REALM 类似的潜在变量模型来构建开放问答，并通过最大化边际似然进行训练。然而，REALM 增加了一个新颖的语言模型预训练步骤，并将反向传播引入 MIPS 索引，而不是使用固定索引。

#### 基于生成的开放问答（Open-QA）

开放问答（Open-QA）的一种新兴替代方法是将它建模为一个序列预测任务：对问题进行编码，然后基于编码逐个 token 解码答案。


## 5. 讨论与相关工作

### 5.1 与开放问答相关的其他视角

#### 5.1.1 以语料库为上下文的语言建模

语言表示模型在进行预测时，已经逐渐融入了越来越大的上下文范围，包括周围单词、句子和段落。我们可以将 REALM 视为上述工作向更高层次范围的推广：整个文本语料库。

#### 5.1.2 学习检索的检索与编辑

为了更好地解释输入文本中的变化并实现可控生成，Guu 等人提出了一种基于检索与编辑框架的语言模型。REALM 采用了类似的方法，只是模型自己学习哪些文本对降低困惑度最有用。

#### 5.1.3 可扩展的基于神经的记忆

文档索引可以被视为一个记忆，其中键是文档嵌入。从这个角度看，我们的工作与产品键记忆等工作有着共同的动机，后者在记忆网络中实现了亚线性内存访问，允许将这些可扩展的记忆层集成到大语言模型中。一个主要区别是，我们的记忆是有根基的——每个记忆都与一个文档相关联，而不是未命名的值向量。这种解释性水平对于开放问答等应用至关重要，因为用户需要预测答案的可信来源。

#### 5.1.4 无监督语料库对齐

在具有注意力机制的序列到序列模型中，文本是通过对相关 token 进行潜在选择来生成的。这导致了一组以模型为中心的目标 token 和源 token 的无监督对齐。类似地，REALM 也通过对相关文档进行潜在选择来生成文本。我们方法的一个副产品是，我们提供了一组以模型为中心的无监督对齐，将预训练语料库 $X$ 中的文本与知识语料库 $Z$ 对齐。


## A. 知识检索器梯度推导

我们计算 REALM 预训练目标（对数似然）相对于知识检索器参数 θ 的梯度：

$$
\begin{aligned}
\nabla\log p(y|x)&=p(y|x)^{-1}\nabla p(y|x) \\
&=p(y|x)^{-1}\sum_{z}p(y|z,x) \nabla p(z|x) \\ 
&=p(y|x)^{-1}\sum_{z}p(y|z,x)p(z|x)\nabla\log p(z|x) \\ 
&=\sum_{z}p(z|y,x)\nabla\log p(z|x)
\end{aligned}
$$

其中最后一行是应用条件贝叶斯规则得出的。然后我们可以将 $\nabla \log p (z\|x)$ 展开为：

$$
\begin{aligned}
\nabla\log p(z|x)&=\nabla \log \frac{\exp f(x,z)}{\sum_{z'}\exp f(x,z')} \\
&=\nabla\left[f(x,z)-\log\sum_{z^{\prime}}\exp f(x,z^{\prime})\right] \\
&=\nabla f(x,z)-\sum_{z^{\prime}}p(z^{\prime}\,|\,x)\nabla f(x,z^{\prime}) \\
\end{aligned}
$$

将其代回第一组方程得到：

$$
\begin{aligned}
\nabla \log p (y | x) &= \sum_z p (z | y, x) \left[\nabla f(x,z) - \sum_{z'} p(z'|x)\nabla f(x, z') \right] \\
&= \sum_z p (z | y, x)\nabla f(x,z) - \sum_{z'} p(z'|x)\nabla f(x, z') \\
&= \sum_z[p (z | y, x)-p(z|x)]\nabla f(x,z) \\
&= \sum_z\left[\frac{p(y|z,x)p(z|x)}{p(y|x)}  - p(z|x) \right]\nabla f(x,z)\\
&= \sum_z\left[\frac{p(y|z, x)}{p(y|x)} - 1 \right] p(z|x) \nabla f(x,z)
\end{aligned}
$$

在第二行中，我们使用了整体表达式是相对于 $p (z \| y, x)$ 的期望这一事实，以及依赖于 $z'$ 但不依赖于 $z$ 的项可以从该期望中移出。

## B. Realm与监督学习之间的联系

从附录A中的公式可以看出：

$$
\nabla\log p\left(y\,|\,x\right)=\sum_{z}\left[p\left(z\,|\,y,x\right)-p\left(z\,|\,x\right)\right]\nabla f(x,z).
$$

假设存在一个文档 $z^\*$ 使得模型达到完美的预测准确率（即 $p(y\| z^\*, x) = 1$），而所有其他文档 $z'$ 的准确率为零（即 $p(y\| z', x) = 0$）。在这种情况下，$p(z^\*\| y, x) = 1$（前提是 $p(z^\*\| x)$ 非零），这导致梯度变为

$$
\begin{aligned}
\nabla\log p\left(y\,|\,x\right)&=\nabla f\left(x,z^{*}\right)-\sum_{z}p\left(z\,|\,x\right)\nabla f(x,z) \\
&=\nabla\log p\left(z^{*}\,|\,x\right).
\end{aligned}
$$

由此，我们可以看出，在 REALM 目标上的梯度下降等同于在 $\log p(z^\*\| x)$ 上的梯度下降。这正是监督学习中使用的典型最大似然训练目标，其中 $z^\*$ 是"黄金"文档。


## D. 检索效用

在第 3.4 节中描述的空文档 ∅ 提供了一种衡量检索文档 $z$ 重要性的方法：我们定义 $z$ 对于掩码输入 $x$ 的**检索效用**（$RU$）为在给定 $z$ 的条件下的知识增强编码器的对数似然与在给定 ∅ 的条件下的对数似然之间的差异：

$$
\mathrm{RU}(z\,|\,x)=\log p(y\,|\,z,x)-\log p(y\,|\,\varnothing,x).
$$

负的 $RU$ 表明 $z$ 对于预测 $y$ 的有用性不如空文档。这可能意味着 $z$ 与 $x$ 无关，但也可能意味着 $x$ 中的掩码标记不需要世界知识来预测，或者世界知识足够普遍，已经被烘焙到模型的参数中。实际上，我们发现 $RU$ 在预训练过程中稳步增加，并且比整体对数似然更能预测下游任务 Open-QA 的良好性能。