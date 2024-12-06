---
layout: post
date: 2024-12-1T21:20:25+08:00
title: 【论文笔记】Unifying Large Language Models And Knowledge Graphs-A Roadmap
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

本文是[《Unifying Large Language Models And Knowledge Graphs: A Roadmap》](https://arxiv.org/pdf/2306.08302)的笔记

## 摘要

大型语言模型（LLMs），如 ChatGPT 和 GPT4，由于其涌现能力（emergent ability）和泛化性，在自然语言处理和人工智能领域引起了新的浪潮。然而，LLMs 是黑盒模型，往往无法捕捉和访问事实知识。相比之下，知识图谱，例如维基百科和华普，是结构化的知识模型，显式地存储丰富的事实知识。知识图谱可以通过提供外部知识来增强 LLMs 的推理能力和可解释性。与此同时，知识图谱的构建和演化具有困难，这对现有的知识图谱方法在生成新的事实和表示未见过的知识方面提出了挑战。因此，将 LLMs 和知识图谱统一起来并同时利用它们的优势是互补的。在本文中，我们提出了一个前瞻性的 LLMs 和知识图谱统一的路线图。我们的路线图包括三个通用框架，即：
* **知识图谱增强的LLMs（KG-enhanced LLMs）**：在 LLMs 的预训练和推理阶段中引入知识图谱，或者为了增强对 LLMs 学到的知识的理解
* **LLM 增强的知识图谱（LLM-augmented KGs）**：利用 LLMs 进行不同的知识图谱任务，如嵌入、补全、构建、图文生成和问答
* **协同的 LLMs + 知识图谱（Synergized LLMs + KGs）**：LLMs 和 KGs 发挥同等作用，并以相互促进的方式增强 LLMs 和知识图谱，以数据和知识双向驱动推理

我们在路线图中回顾和总结了这三个框架中的现有工作，并指出了它们未来的研究方向。

## 1. 引言

尽管在许多应用中取得了成功，但 LLMs 因**缺乏事实知识**（factual knowledge）而受到批评。LLMs 会记忆训练语料库中包含的事实和知识。然而进一步的研究表明，LLMs 无法回忆事实，并且经常产生幻觉，生成与事实不符的陈述，这严重损害了 LLMs 的可信度。

作为黑盒模型，LLMs 也因其**缺乏可解释性**而受到批评。LLMs 在其参数中隐式地表示知识。很难解释或验证 LLMs 获得的知识。此外，LLMs 通过概率模型进行推理，这是一个不确定的过程。LLMs 用于预测或决策的具体模式和功能对我们来说不是直接可访问或可解释的。尽管一些 LLMs 可以通过应用思维链来解释它们的预测，它们的推理解释也受到幻觉问题的困扰。这严重影响了 LLMs 在高风险场景中的应用。这引发了另一个问题，即 LLMs 在一般语料库上训练可能无法很好地泛化到特定领域或新知识，因为缺乏特定领域的知识或新的训练数据。

为了解决上述问题，一个潜在的解决方案是将知识图谱整合到 LLMs 中。知识图谱以三元组（*头实体，关系，尾实体*）的形式存储大量事实，是一种**结构化**和**明确**的知识表示方式。知识图谱对于各种应用至关重要，因为提供了准确的显式知识。此外，知识图谱**以符号推理能力而闻名，这产生了可解释的结果**。知识图谱还可以随着新知识的不断添加而积极演变 。此外，专家可以构建特定领域的知识图谱，以提供精确可靠的特定领域知识。

尽管如此，**知识图谱难以构建**，当前知识图谱的方法不足以处理现实世界知识图谱不完整和动态变化的性质。这些方法未能有效地建模未见过的实体并表示新事实。此外，它们经常忽略知识图谱中丰富的文本信息。此外，现有的方法通常针对特定的知识图谱或任务进行定制，这不够通用。因此，有必要利用 LLMs 来解决知识图谱（KGs）面临的挑战。我们在图 1 中分别总结了 LLMs 和 KGs 的优缺点。

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-1.png" width="600" alt=""/>
*图1：LLMs 的优点：通用知识、语言处理、泛化能力；LLMs 的缺点：隐性知识、幻觉、犹豫不决、黑盒、缺乏领域特定/新知识。KGs 的优点：结构化知识、准确性、确定性、可解释性、领域特定知识、不断演化的知识；KGs 的缺点：不完整性、缺乏语言理解、未知事实。*


最近，将 LLMs 与 KGs 统一的可能性引起了研究人员和实践者的日益关注。LLMs 和 KGs 本质上是相互联系的，可以相互增强。在 **KG-enhanced LLMs** 中，KGs 不仅可以整合到 LLMs 的预训练和推理阶段以提供外部知识，还可以用于分析 LLMs 并提供可解释性。在 **LLM-augmented KGs** 中，LLMs 已被用于各种与 KG 相关的任务，例如 KG embedding、KG completion、KG construction、KG-to-text generation 和 KGQA，以提高性能并促进 KGs 的应用。在 **Synergized LLM + KG** 中，研究人员结合了 LLMs 和 KGs 的优点，以相互增强知识和推理方面的性能。尽管有一些关于 KG-enhanced LLMs 的研究，但主要关注使用 KGs 作为外部知识来增强 LLMs，忽略了将 KGs 整合到 LLMs 的其他可能性以及 LLMs 在 KG 应用中的潜在作用。

在本文中，我们提出了一个将 LLMs 和 KGs 统一的前瞻性路线图，以利用它们各自的优势并克服每种方法的局限性，用于各种下游任务。我们提出了详细的分类，进行了全面的回顾，并指出了这些快速发展的领域中的新兴方向。我们的主要贡献总结如下：

(1) **路线图**：我们提出了一个将 LLMs 和 KGs 统一的前瞻性路线图。我们的路线图包括三个统一 LLMs 和 KGs 的通用框架，即 KG-enhanced LLMs、LLM-augmented KGs 和 Synergized LLMs + KGs，为这两种独立但互补的技术的统一提供了指导。

(2) **分类和回顾**：对于我们路线图的每个集成框架，我们提出了详细的分类和新颖的组织方法，用以统一 LLMs 和 KGs。

(3) **新兴进展的覆盖**：我们涵盖了 LLMs 和 KGs 中的先进技术。包括了对 ChatGPT 和 GPT-4 等最先进的 LLMs 以及例如多模态知识图谱等新颖 KGs 的讨论。

(4) **挑战和未来方向的总结**：我们突出了现有研究中的挑战，并提出了几个有前景的未来研究方向。

## 2 背景

### 2.1 大型语言模型（LLMs）

大型语言模型在大规模语料库上进行预训练，在各种自然语言处理（NLP）任务中显示出巨大的潜力。大多数 LLMs 都源自 Transformer 设计，它包含编码器和解码器模块，这些模块由自注意力机制赋能。基于架构结构，LLMs 可以分为三组：

* **Encoder-only LLMs**
* **Encoder-decoder LLMs**
* **Decoder-only LLMs**

如图 2 所示，我们总结了近年来具有不同模型架构、模型大小和开源可用性的几种代表性 LLMs。

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-2.png" width="600" alt=""/>

*图2：近年来的代表性大型语言模型。开源模型由实心方块表示，而闭源模型由空心方块表示。*

#### 2.1.1 Encoder-only LLMs

Encoder-only LLMs 仅使用编码器来编码句子并理解单词之间的关系。这些模型的常见训练范式是**预测输入句子中的掩码词**。这种方法是无监督的，可以在大规模语料库上进行训练。Encoder-only LLM 例如 BERT、ALBERT、RoBERTa 和 ELECTRA 等**需要添加额外的预测头来解决下游任务。这些模型对于需要理解整个句子的任务最为有效，例如文本分类和命名实体识别**。

#### 2.1.2 Encoder-decoder LLMs

Encoder-decoder LLMs 使用了编码器和解码器两个模块。编码器模块负责将输入句子编码到隐藏空间，而解码器用于生成目标输出文本。Encoder-decoder LLMs 的训练策略可以更加灵活。例如 T5 通过掩码和预测掩码词的片段进行预训练。Encoder-decoder LLMs（如 T0、ST-MoE 和 GLM-130B）能够**直接解决基于某些上下文生成句子的任务**，例如摘要生成、翻译、问答等。

#### 2.1.3 Decoder-Only LLMs

Decoder-only LLMs 仅采用解码器模块来生成目标输出文本。这些模型的训练范式是**预测句子中的下一个单词**。大规模的 decoder-only LLMs 通常可以**从少量示例或简单指令执行下游任务，无需添加预测头或微调**。许多最先进的 LLMs（例如，Chat-GPT 和 GPT-4）遵循 decoder-only 架构。这些模型是闭源的，Alpaca 和 Vicuna 作为开源 decoder-only LLMs 发布，这些模型基于 LLaMA 进行微调，性能与 Chat-GPT 和 GPT-4 相当。

#### 2.1.4 提示工程

提示工程是一门新兴领域，专注于创建和完善提示，以**最大化 LLMs 在各种应用和研究领域中的有效性**。提示是一个为特定任务指定的自然语言输入序列，可以包含以下几个元素：

- **指令（Instruction）**：一个简短的句子，指示模型执行特定任务
- **上下文（Context）**：为输入文本提供上下文或少量示例
- **输入文本（Input Text）**：需要由模型处理的文本

**提示工程旨在提高大型语言模型（如 ChatGPT）在各种复杂任务中的能力**，如问答、情感分类和常识推理。链式思维（CoT）提示通过中间推理步骤实现复杂的推理能力。提示工程还使结构化数据，例如 KGs 能够集成到 LLMs 中。Li 等人简单地将 KGs 线性化，并使用模板将 KGs 转换为段落。Mindmap 设计了一个 KG prompt，将图结构转换为思维导图，使 LLMs 能够在其上进行推理。Prompt 提供了一种简单的方法来利用 LLMs 的潜力，而无需微调。精通提示工程有助于更好地理解 LLMs 的优势和劣势。


### 2.2 知识图谱（Knowledge Graphs, KGs）

KGs 将结构化知识存储为一系列三元组，形式为 $KG = \\{ (h, r, t) ⊆ E × R × E \\}$，其中 $E$ 和 $R$ 分别表示实体集和关系集。

现有的 KGs 可以根据存储的信息分为四类：

1. **百科知识图谱**：存储广泛的主题和事实信息
2. **常识知识图谱**：存储日常生活中的常识和通用知识
3. **领域特定知识图谱**：针对特定领域或行业的知识
4. **多模态知识图谱**：结合文本、图像、音频等多种模态的信息


#### 2.2.1 百科知识图谱

百科知识图谱是最普遍的知识图谱类型，代表了现实世界中的一般知识。百科知识图谱通常通过整合来自多种广泛来源的信息来构建，包括人类专家、百科全书和数据库。Wikidata 是最广泛使用的百科知识图谱之一，它整合了从维基百科文章中提取的各种知识。其他典型的百科知识图谱，如Freebase、Dbpedia 和 YAGO 也是从维基百科衍生出来的。[NELL](https://paperswithcode.com/dataset/nell) 是一个不断改进的百科知识图谱，自动从网络中提取知识，并利用这些知识持续提升性能。

#### 2.2.2 常识知识图谱

常识知识图谱构建了关于日常概念（例如物体和事件）及其关系的知识。与百科知识图谱相比，**常识知识图谱通常对从文本中提取的隐性知识进行建模**，如（Car, UsedFor, Drive）。ConceptNet 包含了广泛的常识概念和关系，可以帮助计算机理解人们使用的词语的含义。ATOMIC 和 ASER 关注事件之间的因果效应，可用于常识推理。其他一些常识知识图谱，如 TransOMCS 和 CausalBanK 是自动生成的，以提供常识知识。

#### 2.2.3 领域特定知识图谱

领域特定的知识图谱通常是为了表示特定领域的知识而构建的，例如医学、生物学和金融。与百科全书式的知识图谱相比，领域特定的知识图谱通常规模较小，但更加准确和可靠。例如，UMLS 是一个医学领域的领域特定知识图谱，包含了生物医学概念及其关系。

#### 2.2.4 多模态知识图谱

与传统知识图谱仅包含文本信息不同，多模态知识图谱可以表示包括图像、声音和视频等多种模态的事实。例如 IMGpedia、MMKG、Richpedia 等。这些知识图谱结合了文本和图像信息。它们可以应用于各种多模态任务，如图像-文本匹配、视觉问答和推荐系统。

### 2.3 应用

LLMs 作为 KGs 已经在各种现实世界的应用中得到了广泛的应用。我们在表 1 中总结了一些使用 LLMs 和 KGs 的代表性应用。


| 名称 | 类别 | 大型语言模型 | 知识图谱 | 网址 |
|------|------|--------------|----------|------|
| ChatGPT/GPT-4 | 聊天机器人 | ✓ | | [链接](https://shorturl.at/cmsE0) | 
| ERNIE 3.0 | 聊天机器人 | ✓ | ✓ | [链接](https://shorturl.at/sCLV9) |
| Bard | 聊天机器人 | ✓ | ✓ | [链接](https://shorturl.at/pDLY6) |
| Firefly | 照片编辑 | ✓ | | [链接](https://shorturl.at/fkzJV) |
| AutoGPT | AI 助手 | ✓ | | [链接](https://shorturl.at/bkoSY) |
| Copilot | 编码助手 | ✓ | | [链接](https://shorturl.at/lKLUV) |
| New Bing | 网络搜索 | ✓ | | [链接](https://shorturl.at/bimps) |
| Shop.ai | 推荐系统 | ✓ | | [链接](https://shorturl.at/alCY7) |
| Wikidata | 知识库 | | ✓ | [链接](https://shorturl.at/lyMY5) |
| KO | 知识库 | | ✓ | [链接](https://shorturl.at/sx238) |
| OpenBG | 推荐系统 | | ✓ | [链接](https://shorturl.at/pDMV9) |
| Doctor.ai | 医疗助手 | ✓ | ✓ | [链接](https://shorturl.at/dhlK0) |

*表1：使用大型语言模型和知识图谱的代表性应用*

- **ChatGPT/GPT-4**：基于 LLM 的聊天机器人，可以与人类以自然对话的形式进行交流。
- **Firefly**：开发了一款照片编辑应用，允许用户通过自然语言描述来编辑照片。
- **Copilot**、**New Bing** 和 **Shop.ai**：分别使用 LLMs 来增强他们在编码助手、网络搜索和推荐领域的应用。
- **Wikidata** 和 **KO**：是两个代表性的知识图谱应用，用于提供外部知识。
- **OpenBG**：为推荐而设计的知识图谱。
- **Doctor.ai**：一款结合了 LLMs 和 KGs 的医疗助手，用于提供医疗建议。

## 3 路线图与分类

### 3.1 路线图

统一 KGs 和 LLMs 的路线图如图 6 所示。在路线图中，我们确定了三个统一 LLMs 和 KGs 的框架，包括**KG 增强 LLMs**、**LLM 增强的 KGs**和**协同的 LLMs + KGs**。

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-3.png" width="600" alt=""/>

*图6：将 KGs 和 LLMs 统一的一般路线图如下： KG 增强的 LLMs；LLM 增强的 KGs；协同作用的 LLMs + KGs。*

KG 增强 LLMs 和 LLM 增强 KGs 是两个并行的框架，分别旨在增强 LLMs 和 KGs 的能力。在这些框架的基础上，Synergized LLMs + KGs 是一个统一的框架，旨在使 LLMs 和 KGs 相互协同增强。

#### 3.1.1 知识图谱增强大型语言模型（Kg-Enhanced LLMs）

为了解决 LLMs 幻觉和缺乏可解释性的问题，研究人员提出用 KGs 来增强 LLMs。

KGs 以显式和结构化的方式存储大量知识，可以用来增强 LLMs 的知识认知。一些研究人员提出在 LLMs 的预训练阶段将 KGs 纳入其中，这可以帮助 LLMs 从 KGs 中学习知识。其他研究人员提出在 LLMs 的推理阶段将 KGs 纳入其中。通过从 KGs 中检索知识，可以显著提高 LLMs 访问特定领域知识的性能。为了提高 LLMs 的可解释性，研究人员还利用 KGs 来解释事实和 LLMs 的推理过程。

#### 3.1.2 LLM 增强知识图谱（LLM-Augmented Kgs）

现有 KGs 方法在处理不完整的知识图谱和处理文本语料库以构建知识图谱方面存在不足。许多研究人员正尝试利用 LLMs 解决与知识图谱相关的任务。

将 LLMs 作为文本编码器应用于与 KGs 相关的任务是最直接的方法。研究人员利用 LLMs 处理 KGs 中的文本语料库，然后使用文本的表示来丰富 KGs 的表示。一些研究还使用 LLMs 处理原始语料库并提取关系和实体以构建知识图谱。最近的研究尝试设计一种知识图谱 prompt，可以有效地将结构化知识图谱转换为 LLMs 可以理解的格式。通过这种方式，LLMs 可以直接应用于与知识图谱相关的任务，例如知识图谱补全和知识图谱推理。

#### 3.1.3 协同作用的大语言模型与知识图谱（Synergized LLMs + KGs）

近年来，大语言模型（LLMs）和知识图谱（KGs）的协同作用引起了研究人员的日益关注。LLMs 和 KGs 是两种本质上互补的技术，应统一到一个通用框架中，以相互增强。

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-4.png" width="600" alt=""/>

*图7：协同 LLMs + KGs* 的一般框架，包含四层：1）数据，2）协同模型，3）技术，4）应用。

为了进一步探索统一，我们提出了一个统一的协同作用 LLMs + KGs 框架，如图 7 所示。统一框架包含四层：1）**数据**，2）**协同模型**，3）**技术**，4）**应用**。在数据层，LLMs 和 KGs 分别用于处理文本和结构化数据。随着多模态 LLMs 和 KGs 的发展，该框架可以扩展到处理多模态数据，如视频、音频和图像。在协同模型层，LLMs 和 KGs 可以相互协同以提高它们的能力。在技术层，可以将 LLMs 和 KGs 中使用的相关技术纳入此框架以进一步提高性能。在应用层，可以将 LLMs 和 KGs 集成以解决各种现实世界的应用，如搜索引擎、推荐系统和 AI 助手。

### 3.2 分类

为了更好地理解统一 LLMs 和 KGs 的研究，我们进一步为路线图中的每个框架提供了细粒度的分类。研究的细粒度分类如图 8 所示：

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-5.png" width="600" alt=""/>

*图8：统一大语言模型（LLMs）与知识图谱（KGs）研究的细粒度分类*

#### KG 增强 LLMs

将 KGs 集成可以增强 LLMs 在各种下游任务中的性能和可解释性。我们将 KG 增强 LLMs 的研究分为三组：

* **KG 增强 LLM 预训练**：包括在预训练阶段应用 KGs 并提高 LLMs 知识表达的工作。
* **KG 增强 LLM 推理**：包括在 LLMs 推理阶段利用 KGs 的研究，这使得 LLMs 能够在不重新训练的情况下访问最新知识。
* **KG 增强 LLM 可解释性**：包括使用 KGs 理解 LLMs 学到的知识并解释 LLMs 推理过程的工作。

#### LLM 增强 KGs

LLMs 可以应用于增强各种与 KG 相关的任务。我们根据任务类型将 LLM 增强 KGs 的研究分为五组：

* **LLM 增强 KG 嵌入**：包括应用 LLMs 通过编码实体和关系的文本描述来丰富 KGs 表示的研究。
* **LLM 增强 KG 补全**：包括利用 LLMs 编码文本或生成事实以提高 KG 补全性能的论文。
* **LLM 增强 KG 构建**：包括应用 LLMs 解决 KG 构建中的**实体发现**、**指代消解**（coreference resolution）和**关系提取**任务的工作。
* **LLM 增强 KG 到文本生成**：包括利用 LLMs 生成描述 KGs 事实的自然语言的研究。
* **LLM 增强 KG 问答**：包括应用 LLMs 弥合自然语言问题和从 KGs 检索的答案之间的差距的研究。

#### 协同作用 LLMs + KGs

LLMs 和 KGs 的协同作用旨在将 LLMs 和 KGs 集成到一个统一框架中，以相互增强。我们将从知识表示和推理的角度回顾最近的协同作用 LLMs + KGs 的尝试。

## 4 Kg-Enhanced LLMs

### 4.1 知识图谱增强的语言模型预训练

现有的大型语言模型主要依赖于对大规模语料库的无监督训练。虽然这些模型在下游任务上表现出色，但它们往往缺乏与现实世界相关的实际知识。

将 KGs 集成到大型语言模型中的先前工作可以分为三个部分：1）将 KGs 集成到训练目标中，2）将 KGs 集成到 LLM 输入中，以及 3）KGs 指令微调。

#### 4.1.1 将知识图谱整合到训练目标中

这一类别的研究工作主要集中在**设计新颖的知识感知训练目标**（designing novel knowledge-aware training objectives）。一个直观的想法是**在预训练目标中暴露更多的知识实体**。GLM **利用知识图谱结构来分配掩码概率**。具体来说，**在一定的跳数内可以到达**（can be reached within a certain number of hops）的实体是最重要最需要学习的实体，在预训练期间给予更高的掩码概率。此外，E-BERT 进一步控制了 **token 级和实体级训练损失之间的平衡**。训练损失值被用作 token 和实体学习过程的指示，动态地决定了下一个训练周期的比率。SKEP 也遵循类似的融合，在 LLMs 预训练期间注入情感知识。SKEP 首先利用 PMI 以及预定义的一组种子情感词来确定具有积极和消极情感的词，然后为**这些识别出的情感词在词语掩码目标中分配更高的掩码概率**。

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-6.png" width="600" alt=""/>
*图9：通过文本知识对齐损失将 KG 信息注入到 LLMs 的训练目标中，其中 h 表示 LLMs 生成的隐藏表示*

另一条研究线明确利用了知识与输入文本之间的联系。如图 9 所示，ERNIE 提出了一种新颖的**词-实体对齐训练目标**作为预训练目标。具体来说，ERNIE 将句子和文本中提到的相应实体一起输入 LLMs，然后训练 LLMs **预测文本 token 和知识图谱中的实体之间的对齐链接（alignment links）**。类似地，KALM 通过结合实体嵌入来增强输入 token，并在 token-only 的预训练目标之外**增加了实体预测的预训练任务**。这种方法旨在提高 LLMs 捕捉与实体相关知识的能力。最后，KEPLER 直接**将知识图谱嵌入训练目标和掩码 token 预训练目标整合到一个共享的基于 Transformer 的编码器**中。确定性 LLM 专注于预训练语言模型以**捕捉确定性的事实知识**。它只对具有确定性实体作为问题的 span 进行掩码，并引入额外的**线索对比学习**（clue contrast learning）和**线索分类目标**（clue classification objective）。WKLM 首先用其他同类型的实体替换文本中的实体，然后将它们输入 LLMs。模型进一步预训练以区分实体是否已被替换。


#### 4.1.2 将 KGs 集成到 LLMs 的输入中

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-7.png" width="600" alt=""/>

*图10：通过图结构将知识图谱信息注入到 LLMs 的输入中。*

如图 10 所示，这类研究侧重于将相关的知识子图引入到 LLMs 的输入中。给定一个知识图谱三元组和相应的句子，ERNIE 3.0 将三元组表示为 token 序列，并直接将其与句子连接起来。它进一步**随机掩码三元组中的关系 token 或句子中的 token**，以更好地将知识与文本表示相结合。但这种知识三元组直接连接的方法，允许句子中的 token 与知识子图中的 token 密集交互，这可能**导致知识噪声（Knowledge Noise）**。为了解决这个问题，K-BERT 首先通过一个可见矩阵将知识三元组注入到句子中，只有知识实体可以访问知识三元组的信息，而句子中的 token 只能在自注意力模块中相互关注。为了进一步减少知识噪声，Colake 提出了一个统一的词-知识图（word-knowledge graph）（见图10），其中输入句子中的 token 形成一个完全连接的词图，与知识实体对齐的 token 与其相邻实体相连。

上述方法确实可以将大量的知识注入到 LLMs 中。然而，它们大多集中在热门实体上，忽视了低频和长尾实体。DkLLM 旨在改进 LLMs对这些实体的表示。DkLLM 首先提出了一种新颖的度量方法来确定长尾实体，然后用 pseudo token 嵌入替换文本中的这些选定实体，作为大型语言模型的新输入。Dict-BERT 提出**利用外部词典**来解决这个问题。具体而言，Dict-BERT 通过在输入文本末尾附加来自词典的定义，提高了罕见词的表示质量，并训练语言模型局部对齐输入句子中罕见词表示和词典定义，并判断输入文本和定义是否正确映射。

#### 4.1.3 知识图谱指令调优

与将事实知识注入LLMs不同，**知识图谱指令调优**（KGs Instruction-tuning）的目标是微调 LLMs 以更好地理解 KGs 的结构，并有效地遵循用户指令来执行复杂任务。KGs Instruction-tuning 利用事实和 KGs 的结构来创建指令调优数据集。在这些数据集上微调的 LLMs 可以从 KGs 中提取事实和结构知识，从而增强 LLMs 的推理能力。

KPPLM 首先设计了几个 prompt 模板，将结构图转化为自然语言文本。然后，提出了两个自监督任务，利用这些 prompt 生成的知识来微调 LLMs。OntoPrompt 提出了一种增强本体的 prompt 微调方法，可以将实体的知识置于 LLMs 的上下文中，并在几个下游任务上进行进一步微调。ChatKBQA 在 KG 结构上对 LLMs 进行微调，以生成逻辑查询，这些查询可以在 KGs 上执行以获取答案。为了更好地在图上进行推理，RoG 为 LLMs 提供了从 KGs 中进行如实推理和生成可解释结果的推理路径。

KGs Instruction-tuning 可以更好地利用 KGs 中的知识来进行下游任务。然而，这需要重新训练模型，耗时且需要大量资源。

### 4.2 知识图谱增强语言模型推理

上述方法可以有效地将知识融合到大型语言模型（LLMs）中。然而，现实世界的知识是不断变化的，这些方法的局限性在于**不允许在不重新训练模型的情况下更新所融合的知识**，在推理过程中可能无法很好地泛化到未见过的知识。因此，相当多的研究致力于**将知识空间和文本空间分开，并在推理过程中注入知识**。这些方法大多关注在问答（QA）任务，因为 QA 任务要求模型捕捉文本语义和最新的现实世界知识。

#### 4.2.1 检索增强知识融合

**检索增强知识融合**（Retrieval-Augmented Knowledge Fusion）是一种在推理过程中向大型语言模型（LLMs）注入知识的流行方法。其核心思想是从大量语料库中检索相关知识，然后将检索到的知识融合到 LLMs 中。

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-21.png" width="600" alt=""/>

*图11：检索外部知识以增强 LLM 生成*

如图 11，RAG（Retrieval-Augmented Knowledge Fusion）提出了**结合非参数化和参数化模块**来处理外部知识的方法。给定输入文本，RAG 首先通过**最大内积搜索（MIPS）在非参数化模块中搜索相关的知识图谱**，以获取多个文档。然后，RAG 将这些文档视为隐藏变量 $z$，并将它们作为额外的上下文信息输入到由 Seq2Seq LLMs 支持的输出生成器中。实验结果显示，RAG 在开放域问答中优于其他仅参数化或仅非参数化的基准模型。RAG 还能够生成比其他参数化基准模型更具体、多样化和真实的文本。Story-fragments 通过添加一个额外的模块来确定显著的知识实体，并将它们融合到生成器中，以提高生成长篇故事的质量。EMAT 通过将外部知识编码为键值内存，并利用快速最大内积搜索进行内存查询，进一步提高了这种系统的效率。REALM 提出了一种新颖的知识检索器，在预训练阶段帮助模型从大型语料库中检索和关注文档，成功提高了开放域问答的性能。KGLM 根据当前上下文从知识图谱中选择事实，生成事实性句子。借助外部知识图谱的帮助，KGLM 可以使用领域外的词汇或短语来描述事实。

#### 4.2.2 KGs Prompting

为了在推理过程中更好地将 KG 结构输入到LLM 中，KGs Prompting 旨在设计一种精心制作的 prompt ，将结构化的 KG 转换为文本序列，这些文本序列可以作为上下文输入到 LLM 中。通过这种方式，LLM 可以更好地利用 KG 的结构进行推理。Li 等人采用预定义模板将每个三元组转换为短句，以便 LLM 进行推理。Mindmap 设计了一种 KG prompt，将图结构转换为思维导图，使 LLM 能够通过整合 KG 中的事实和 LLM 中的隐含知识来进行推理。ChatRule 从 KG 中采样几个关系路径，将其表述出来并输入到 LLM 中，然后使用 prompt 让 LLM 生成有意义的逻辑规则用于推理。CoK 提出了一种知识链的 prompt，使用一系列三元组来激发 LLM 的推理能力，以达到最终答案。

KGs Prompting 为 LLM 和 KG 的协同提供了一种简单的方式。通过使用 prompt，我们可以轻松地利用 LLM 基于 KG 进行推理的能力，而无需重新训练模型。不过 prompt 通常是手动设计的，这需要大量的人力。

### 4.3  KG-enhanced LLM 预训练与推理的比较

KG-enhanced LLM 的预训练方法通常**通过语义相关的真实世界知识丰富大量未标记的语料库**。这些方法允许知识表示与适当的语言上下文对齐（knowledge representations to be aligned with appropriate linguistic context），并明确训练 LLMs 从零开始利用这些知识。当将结果 LLMs 应用于下游知识密集型任务时，它们应该达到最佳性能。相比之下，KG-enhanced LLM  的推理方法只在推理阶段向 LLMs 呈现知识，而底层的 LLMs 可能没有训练充分地利用这些知识进行下游任务，可能导致次优的模型性能。

然而，现实世界的知识是动态的，需要频繁更新。尽管有效，但 KG-enhanced LLM 预训练方法不允许在不重新训练模型的情况下更新或编辑知识。因此，**预训练方法可能对最近或未见过的知识泛化能力较差。KG-enhanced LLM 的推理方法可以通过更改推理输入轻松维护知识更新。**这些方法有助于提高 LLMs 在新知识和领域上的性能。

总之，何时使用这些方法取决于应用场景。如果希望将 LLMs 应用于特定领域的实时不敏感知识（例如常识和推理知识），应考虑预训练方法。否则，可以使用推理方法来处理频繁更新的开放领域知识。

### 4.4 KG-enhanced LLM 的可解释性

大型语言模型的可解释性是指**理解和解释大型语言模型的内部运作和决策过程**。这可以提高 LLMs 的可信度，并促进它们在高风险场景（如医疗诊断和法律判断）中的应用。KGs 结构性地表示知识，可以为推理结果提供良好的可解释性。因此，研究人员尝试利用 KGs 来提高 LLMs 的可解释性，这大致可以分为两类：

- **KGs 用于 LLM 探测**（KGs for language model probing）
- **KGs 用于 LLM 分析**（KGs for language model analysis）


#### 4.4.1 KGs 用于 LLM 探测

LLM 探测旨在理解 LLMs 中存储的知识。LLMs 在大规模语料库上进行训练，通常被认为包含了大量的知识。然而，LLMs 以隐藏的方式存储知识，这使得很难弄清楚存储的知识是什么。此外，LLMs 还受到幻觉问题的困扰，这导致生成与事实相矛盾的陈述，严重影响了 LLMs 的可靠性。因此，有必要探测和验证 LLMs 中存储的知识。

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-8.png" width="600" alt=""/>

*图12：使用知识图谱进行语言模型探测的一般框架。*

LAMA 是第一个通过使用 KGs 来探测 LLMs 中的知识的工作。如图 12 所示，LAMA 首先通过预定义的提示模板将 KGs 中的事实转换为完形填空语句，然后使用 LLMs 来预测缺失的实体。预测结果用于评估 LLMs 中存储的知识。例如，我们尝试探测 LLMs 是否知道事实(Obama, profession, president)。我们首先将事实三元组转换为完形填空问题 "Obama's profession is _."，并将对象屏蔽。然后，我们测试 LLMs 是否能正确预测对象 "president"。

然而，LAMA 忽略了 prompt 不合适的事实。例如，"Obama worked as a _" 可能比 "Obama is a _ by profession" 更有利于语言模型预测。

因此，LPAQA 提出了一种**基于挖掘和释义**（mining and paraphrasingbased）的方法，**自动生成高质量和多样化的 prompt**，以更准确地评估语言模型中包含的知识。此外，Adolphs 等人尝试使用示例，让语言模型理解查询。他们在 TREx 数据上对 BERT-large 取得了显著的改进。与使用手动定义的提示模板不同，Autoprompt 提出了一种**基于梯度引导搜索**（gradient-guided search）的自动化方法来创建 prompt。LLM-facteval 设计了一个系统框架，从 KGs 自动生成探测问题。生成的问题随后用于评估 LLMs 中存储的事实知识。

Alex 等人研究了 LLMs 保留不太流行的事实知识的能力。他们从 Wikidata 知识图谱中选择点击频率低的实体，这些事实随后用于评估，结果表明 LLMs 在处理这类知识时遇到困难，而且扩展规模也未能显著提高尾部事实知识的记忆。

#### 4.4.2 KGs 用于 LLM 分析

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-9.png" width="600" alt=""/>

*图13：使用知识图谱进行语言模型分析的通用框架。*

KGs 用于预训练语言模型 LLMs 分析旨在回答以下问题，例如 “LLMs 是如何生成结果的？” 以及 “LLMs 中的功能和结构是如何工作的？”。为了分析 LLMs 的推理过程，如图 13 所示，KagNet 和 QA-GNN 使用知识图谱来支撑 LLMs 在每个推理步骤中生成的结果。这种方式可以通过从知识图谱中提取图结构来解释 LLMs 的推理过程。Shaobo 等人研究了 LLMs 是如何正确生成结果的。他们采用了从 KGs 中提取的事实进行因果启发式（causal-inspired）分析。这种分析定量地衡量了 LLMs 生成结果所依赖的词模式。结果表明，**LLMs 更多地通过位置上接近的词而不是知识依赖的词来生成缺失的事实**。因此，他们声称由于不准确的依赖，LLMs 不足以记住事实知识。为了解释 LLMs 的训练，Swamy 等人在预训练期间采用语言模型生成知识图谱。LLMs 在训练期间获得的知识可以通过 KGs 中的事实来揭示。为了探索隐含知识如何在 LLMs 的参数中存储，Dai 等人提出了知识神经元的概念。具体来说，已识别的知识神经元的激活与知识表达高度相关。因此，他们通过抑制和放大知识神经元来探索每个神经元所代表的知识和事实。


## 5 LLM-Augmented KGs

传统的知识图谱往往是不完整的，现有的方法通常没有考虑文本信息。为了解决这些问题，最近的研究探索了将 LLMs 集成到知识图谱中，以考虑文本信息并提高下游任务的性能。我们将分别介绍将 LLMs 整合到 KG embedding、KG completion、KG construction、KG-to-text generation 和 KG question answering。

### 5.1 LLM 增强知识图谱嵌入

知识图谱嵌入（KGE）旨在**将每个实体和关系映射到低维向量（嵌入）空间中**。这些嵌入包含了知识图谱的语义和结构信息，可以用于各种任务，如问答、推理和推荐。传统的知识图谱嵌入方法主要依赖于知识图谱的结构信息，通过在嵌入上定义一个评分函数（例如 TransE 和 DisMult）来进行优化。然而，由于结构连接性有限，这些方法在表示未见过实体和长尾关系时往往表现不佳。为了解决这个问题，如图 14 所示，最近的研究采用 LLMs，通过编码实体和关系的文本描述来丰富知识图谱的表示：

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-10.png" width="600" alt=""/>

*图14：LLM 作为知识图谱嵌入（KGE）的文本编码器。*

#### 5.1.1 LLMs 作为文本编码器

Pretrain-KGE 是一个代表性的方法，遵循图14所示的框架。给定来自知识图谱的三元组 $(h, r, t)$，它首先使用 LLM 编码器将实体 $h$、$t$ 和关系 $r$ 的文本描述编码为如下表示形式：

$$
e_h = LLM(Text_h), e_t = LLM(Text_t), er = LLM(Text_r)
$$

其中 $e_h$、$e_r$ 和 $e_t$ 分别表示实体 $h$、$t$ 和关系 $r$ 的初始嵌入。Pretrain-KGE 在实验中使用 BERT 作为 LLM 编码器。然后，将初始嵌入输入到 KGE 模型中生成最终的嵌入 $v_h$、$v_r$ 和 $v_t$。在 KGE 训练阶段，他们通过遵循标准的 KGE 损失函数来优化 KGE 模型：

$$
L = [γ + f(v_h, v_r, v_t) − f(v'_h, v'_r, v'_t)]
$$

其中，$f$ 是 KGE 评分函数，$γ$ 是边界超参数（margin hyperparameter），$v'_h$, $v'_r$ 和 $v'_t$ 是负样本。
在这种方式下，KGE 模型可以学习到足够的结构信息，同时保留了 LLM 中的部分知识，从而实现更好的知识图谱嵌入。KEPLER 提供了一个统一的模型，用于知识嵌入和预训练语言表示。该模型不仅使用强大的 LLMs 生成有效的文本增强知识嵌入，还无缝地将事实知识集成到 LLMs 中。Nayyeri 等人使用 LLMs 生成世界级、句子级和文档级表示。他们将这些表示与图结构嵌入整合到一个统一的向量中，使用四维超复数的二面体和四元数表示。Huang 等人将 LLMs 与其他视觉和图形编码器相结合，学习多模态知识图嵌入，提高下游任务的性能。CoDEx 提出了一种由 LLMs 赋能的新型损失函数，通过考虑文本信息来指导 KGE 模型衡量三元组的可能性，其提出的损失函数不受限于模型结构，可以与任何 KGE 模型结合使用。


#### 5.1.2 使用 LLM 进行联合文本和 KG 嵌入

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-11.png" width="600" alt=""/>

*图15：用于联合文本和知识图谱嵌入的 LLMs。*

另一种方法直接利用 LLMs 将图结构和文本信息同时纳入嵌入空间。如图15所示，kNN-KGE **将实体和关系视为 LLM 中的特殊标记**。在训练过程中，它将每个三元组 $(h, r, t)$ 及其对应的文本描述转换为一个句子 $x$，形式如下：

$$
x = [\texttt{CLS}]~h~\text{Text_h}~[\texttt{SEP}]~r~[\texttt{SEP}]~[\texttt{MASK}]~\text{Text_t}~[\texttt{SEP}]
$$

其中，尾部实体被替换为[MASK]。将该句子输入 LLM，然后微调模型以预测被掩码的实体，表示为：

$$
P_{LLM}(t|h, r) = P([\texttt{MASK}]=t|x, Θ), (4)
$$

其中，$Θ$ 表示 LLM 的参数。LLM 被优化以最大化正确实体 $t$ 的概率。训练完成后，LLM 中相应的 token 表示被用作实体和关系的嵌入。类似地，LMKE 提出了一种对比学习方法，以改进 LLMs 生成的嵌入在 KGE 中的学习。同时，为了更好地捕捉图结构，LambdaKG 采样 1-hop 邻居实体，并将它们的标记与三元组连接成一个句子，输入 LLMs 进行处理。

### 5.2 LLM 增强知识图谱补全

知识图谱补全（KGC）是指**在给定的知识图谱中推断缺失事实的任务**。与知识图谱嵌入类似，传统的 KGC 方法主要关注知识图谱的结构，而没有考虑大量的文本信息。最近 LLMs 的整合使得 KGC 方法能够对文本进行编码或生成事实，以提高 KGC 的性能。这些方法根据它们的利用方式可以分为两个不同的类别：1）**LLM 作为编码器**（PaE），2）**LLM 作为生成器**（PaG）。

#### 5.2.1 LLM 作为编码器（PaE）

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-12.png" width="600" alt=""/>

*图16.：将 LLMs 作为编码器（PaE）用于知识图谱补全的通用框架。*

如图16（a）、（b）和（c）所示，这一方法首先使用 encoder-only LLMs 对文本信息和 KG 事实进行编码。然后，它们通过将编码表示输入到预测头部（可以是简单的多层感知机或传统的 KG 评分函数，如 TransE 和TransR）来预测三元组或掩码实体的可信度。

##### 联合编码（Joint Encoding）

由于 encoder-only LLMs（例如 Bert）在编码文本序列方面表现出色，KG-BERT 将三元组 $(h, r, t)$ 表示为文本序列，并使用 LLM 进行编码，如图16（a）所示。

$$
x = [\texttt{CLS}]~Text_h~[\texttt{SEP}]~Text_r~[\texttt{SEP}]~Text_t~[\texttt{SEP}]
$$

将 $[\texttt{CLS}]$ 标记的最终隐藏状态输入分类器，以预测三元组的可能性，公式化为：

$$
s = σ(MLP(e_{[\texttt{CLS}]}))
$$

其中 $σ(·)$ 表示 sigmoid 函数，$e_{[\texttt{CLS}]}$ 表示由 LLMs 编码后的表示。为了提高 KG-BERT 的效果：
* **MTL-KGC** 提出了一种多任务学习的 KGC 框架，将额外的辅助任务纳入模型的训练中，即**预测**（RP）和**相关性排序**（RR）。
* **PKGC** 通过使用预定义模版将三元组及其支持信息转化为自然语言句子来评估三元组 $(h, r, t)$ 的有效性。这些句子通过 LLMs 进行二分类处理。三元组的支持信息是通过一个语言化函数从 $h$ 和 $t$ 的属性中得出的。例如，如果三元组是 (Lebron James，member of sports team，Lakers)，那么关于 Lebron James 的信息将被语言化为 “Lebron James: American basketball player”。
* **LASS** 观察到语言语义和图结构对于 KGC 同样重要。因此，LASS 提出了同时学习两种类型的嵌入：**语义嵌入**和**结构嵌入**。在这种方法中，将三元组的完整文本传递给 LLM，并分别计算 $h$、$r$ 和 $t$ 的相应 LLM 输出的平均池化。然后，将这些嵌入传递给基于图的方法，即 TransE，以重构知识图谱的结构。


##### MLM 编码（MLM Encoding）

不同于对三元组的完整文本进行编码，许多研究引入了**掩码语言模型**（MLM）的概念来编码 KG 文本（图16（b））。MEMKGC 使用**掩码实体模型**（MEM）分类机制来预测三元组中的掩码实体。输入文本的形式为：

$$
x = [\texttt{CLS}]~Text_h~[\texttt{SEP}]~Text_r~[\texttt{SEP}]~[\texttt{MASK}]~[\texttt{SEP}], (7)
$$

类似于公式4，它试图最大化掩码实体是正确实体 $t$ 的概率。此外，为了使模型能够学习未见过的实体，MEM-KGC 结合了基于实体文本描述预测实体和超类的多任务学习：

$$
x = [\texttt{CLS}]~[\texttt{MASK}]~[\texttt{SEP}]~Text_h~[\texttt{SEP}], (8)
$$

OpenWorld KGC 将 MEM-KGC 模型扩展到应对开放世界 KGC 的挑战，采用了一个流水线框架，其中定义了两个顺序的基于 MLM 的模块：**实体描述预测**（EDP）和**不完整三元组预测**（ITP）。EDP 是一个辅助模块，根据给定的文本描述预测相应的实体；ITP 是目标模块，根据给定的不完整三元组 $(h, r, ?)$ 预测一个合理的实体。EDP 首先使用公式8对三元组进行编码，并生成最终的隐藏状态，然后将其作为公式7中头实体的嵌入传递给 ITP，以预测目标实体。

##### 分离编码（Separated Encoding）

如图16（c）所示，这些方法涉及将三元组 $(h, r, t)$ 分割为两个不同的部分，即 $(h, r)$ 和 $t$，可以表示为：

$$
\begin{align}
x_{(h,r)} &= [\texttt{CLS}]~Text_h~[\texttt{SEP}]~Text_r~[\texttt{SEP}] \\
x_t &= [\texttt{CLS}]~Text_t~[\texttt{SEP}]
\end{align}
$$

然后，这两个部分分别由 LLMs 进行编码，并使用 $[\texttt{CLS}]$ 标记的最终隐藏状态作为 $(h, r)$ 和 $t$ 的表示。然后，将这些表示输入到评分函数中，以预测三元组的可能性，公式化为：

$$
s = f_{score}(e(h,r), e_t)
$$

其中 $f_{score}$ 为类似于 TransE 的评分函数。

StAR 在其文本上应用了 Siamese 风格的文本编码器，将其编码为单独的上下文表示。为了避免文本编码方法（例如 KG-BERT）的组合爆炸，StAR 采用了一个评分模块，其中包括**确定性分类器**和**空间测量**，分别用于**表示**和**结构学习**，从而通过探索空间特征来增强结构化知识。SimKGC 也利用了 Siamese 文本编码器来编码文本表示。在编码过程之后，SimKGC 应用**对比学习**来处理这些表示。该过程涉及计算给定三元组及其正负样本之间的编码表示的相似性。即**最大化三元组编码表示与正样本之间的相似性，同时最小化三元组编码表示与负样本之间的相似性**。这使 SimKGC 能够学习一个将可信和不可信三元组分开的表示空间。为了避免过度拟合文本信息 CSPromp-KG 采用了参数高效的 prompt 学习（parameter-efficient prompt learning）方法用于 KGC。

LP-BERT 是一种混合 KGC 方法，结合了 MLM 编码和分离编码。该方法包括两个阶段，即预训练和微调。在预训练阶段，该方法利用标准的 MLM 机制对带有 KGC 数据的 LLM 进行预训练。在微调阶段，LLM 对两个部分进行编码，并使用对比学习策略进行优化（类似于 SimKGC）。

#### 5.2.2 LLM 作为生成器（PaG）

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-13.png" width="600" alt=""/>

*图17：显示了将 LLMs 作为 KG 完成的解码器（PaG）的一般框架。En. 和 De. 分别表示编码器和解码器。*

最近的研究在 KGC 中使用 LLMs 作为序列到序列生成器。如图17（a）和（b）所示，这些方法涉及 encoder-decoder 或 decoder-only 的 LLMs。LLMs 接收查询三元组 $(h, r, ?)$ 的序列文本输入，并直接生成尾实体 $t$ 的文本。

GenKGC 使用大型语言模型 BART 作为骨干模型。受 GPT-3 中的上下文学习（in-context learning）方法的启发，其中模型将相关样本连接起来学习正确的输出答案，GenKGC 提出了一种关系引导的演示技术（relation-guided demonstration technique），包括具有相同关系的三元组，以促进模型的学习过程。在生成过程中提出一种实体感知的分层解码方法，以降低时间复杂度。

KGT5 引入了一种新颖的 KGC 模型，满足这些模型的四个关键要求：可扩展性、质量、多功能性和简易性。为了实现这些目标，所提出的模型采用了简单直接的 T5 small 架构。该模型与先前的 KGC 方法不同，它是随机初始化的，而不是使用预训练模型。

KG-S2S 是一个综合性框架，可应用于各种类型的 KGC 任务，包括 Static KGC、Temporal KGC 和 Few-shot KGC。为了实现这个目标，KG-S2S 通过引入一个额外的元素来重新定义标准的三元组 KG 事实，形成一个四元组 $(h, r, t, m)$，其中 $m$ 表示额外的“条件”元素。尽管不同的 KGC 任务可能涉及不同的条件，但它们通常具有类似的文本格式，能够在不同的 KGC 任务之间进行统一。KG-S2S 方法结合了各种技术，如实体描述、软提示和 Seq2Seq Dropout，以提高模型的性能。此外，它利用约束解码（constrained decoding）来确保生成的实体是有效的。

对于闭源 LLMs（例如 ChatGPT 和 GPT-4），AutoKG 采用提示工程来设计定制的提示。如图18所示，这些提示包含任务描述、少样本示例和测试输入，指导 LLMs 预测 KG 完成的尾实体。

##### PaE 与 PaG 的比较

**PaE（LLMs 作为编码器）**

**优点**
* 在 LLMs 编码的表示之上应用额外的预测头，更易于微调，只需优化预测头并冻结 LLMs
* 预测的输出可以轻松地指定并与现有的 KGC 函数集成，用于不同的 KGC 任务

**缺点**
  * 在推理阶段需要为 KGs 中的每个候选计算分数，可能计算成本高
  * 不能泛化到未见过的实体
  * 需要 LLMs 的表示输出，而一些最先进的 LLMs（例如 GPT-4）是闭源的，不允许访问表示输出

**PaG（LLMs 作为生成器）**

**优点**
* 不需要预测头，无需微调或访问表示，适用于所有类型的 LLMs
* 直接生成尾实体，推理效率高，无需对所有候选进行排序，能泛化到未见过的实体

**缺点** 
* 生成的实体可能多样且不在 KGs 中
* 自回归生成，单次推理时间较长
* 如何设计强大的 prompt 以将知识图谱输入 LLMs 仍是一个开放性问题

#### 5.2.3 模型分析

Justin 等人对集成了 LLMs 的 KGC 方法进行了全面分析。他们的研究调查了 LLM 嵌入的质量，并发现它们对于有效的实体排名来说并不理想。为此，他们提出了几种处理嵌入的技术，以提高它们用于候选检索的适用性。该研究还比较了不同的模型选择维度，如 Embedding Extraction、Query Entity Extraction 和 Language Model Selection。最后，作者提出了一个框架，有效地将 LLM 适配 KGC。

### 5.3 LLM 增强知识图谱构建

**知识图谱构建涉及在特定领域内创建知识的结构化表示**，这包括识别实体及其彼此之间的关系。知识图谱构建的过程通常包括多个阶段，包括 1）**实体发现**，2）**指代消解**和 3）**关系抽取**。图 19 展示了在知识图谱构建的每个阶段应用 LLMs 的一般框架。最近的方法还探索了 4）端到端的知识图谱构建，即在一步中构建完整的知识图谱，或者直接 5）从 LLMs 中提炼知识图谱。

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-14.png" width="600" alt=""/>

*图19：基于 LLMs 的知识图谱构建的一般框架。*

#### 5.3.1 实体发现

在 KG 构建中，实体发现是指从非结构化数据源（如文本文档、网页或社交媒体帖子）中识别和提取实体，并将其纳入知识图谱的过程。

##### 命名实体识别（Named Entity Recognition (NER)）
命名实体识别涉及在文本数据中识别并标记命名实体的位置和分类。命名实体包括人名、组织、地点和其他类型的实体。最先进的 NER 方法通常利用 LLMs 的上下文理解和语言知识进行准确的实体识别和分类。基于识别的 NER 跨度的类型，有三个 NER 子任务，即 **flat NER**、**nested NER** 和 **discontinuous NER**：

- **Flat NER**：从输入文本中识别不重叠的命名实体。通常将其概念化为序列标注（sequence labelling）问题，其中文本中的每个 token 根据其在序列中的位置被分配一个唯一的标签。
- **Nested NER**：考虑允许一个 token 属于多个实体的复杂场景。
- **Discontinuous NER**：识别文本中可能不连续的命名实体。为了解决这一挑战，使用 LLM 输出来识别实体片段并确定它们是重叠还是连续。

与任务特定方法不同，GenerativeNER 使用带有指针机制的序列到序列 LLM 来生成实体序列，能够解决三种类型的 NER 子任务。

##### 实体类型（ET）

**实体类型（ET）旨在为上下文中提及的实体提供细粒度和超细粒度的类型信息**。这些方法通常**利用 LLM 来编码提及（mention）、上下文和类型**。LDET 使用预训练的 ELMo 嵌入进行词表示，并采用 LSTM 作为其句子和 mention 编码器。BOX4Types 认识到类型依赖的重要性，并使用 BERT 在超矩形（盒子）空间中表示隐藏向量和每个类型。LRN 考虑标签之间的外在和内在依赖关系。它使用 BERT 对上下文和实体进行编码，并利用这些输出嵌入进行演绎和归纳推理。MLMET 使用预定义的模式构建 BERT MLM 的输入样本，并使用 $[\texttt{MASK}]$ 来预测 mention 的上下文相关的上位词（hypernyms），这可以视为类型标签。PL 和 DFET 利用 prompt learning 进行实体类型标注。LITE 将实体类型标注形式化为文本推理，并使用 RoBERTa-large-MNLI 作为骨干网络。

##### 实体链接（EL）
**实体链接（EL），也称为实体消歧，涉及将文本中出现的 entity mentions 与知识图谱中相应的实体进行链接。**[《Investigating Entity Knowledge in BERT with Simple Neural End-To-End Entity Linking》](https://arxiv.org/abs/2003.05473)提出了基于 BERT 的端到端 EL 系统，可以同时发现和链接实体。ELQ 采用了快速的双编码器架构，可以在一次遍历中同时进行 mention 检测和链接，用于下游的问答系统。与以前将 EL 框架视为向量空间匹配的模型不同，GENRE 将其形式化为序列到序列问题，自回归地生成一个带有自然语言中实体唯一标识符的输入标记版本。ReFinED 通过利用基于 LLM 的编码器处理的细粒度实体类型和实体描述，提供了一种高效的 zero-shot 能力的 EL 方法。

#### 5.3.2 指代消解（Coreference Resolution, Cr）

指代消解是为了找到文本中所有指代同一实体或事件的表达式（即 mention）。

##### 文档内指代消解（Within-document CR）

文档内指代消解是指所有这些 mention 都在单个文档中的指代消解子任务。Mandar 等人用 BERT 替换之前的 LSTM 编码器来初始化基于 LLM 的指代消解。这项工作之后引入了 SpanBERT，它是在 BERT 架构上预训练的，使用基于 span 的掩码语言模型（MLM）。受这些工作的启发，Tuan Manh 等人通过将 SpanBERT 编码器整合到非 LLM 方法 e2ecoref 中，提出了一个强大的基线。CorefBERT 利用 Mention Reference Prediction (MRP) 任务，该任务掩码一个或多个 mention，并要求模型预测被掩码 mention 的相应指称对象。CorefQA 将指代消解表述为一个问答任务，为每个候选提及生成上下文查询，并使用这些查询从文档中提取共指 span。Tuan Manh 等人引入了一个门控机制（gating mechanism）和一个噪声训练方法（noisy training method），使用 SpanBERT 编码器从事件 mention 中提取信息。

为了减少基于 LLM 的 NER 模型面临的大内存占用问题，Yuval 等人和 Raghuveer 等人分别提出了端到端和近似模型，两者都利用双线性函数来计算 mention 和先行得分，减少对 span-level 表示的依赖。

##### 跨文档指代消解（Cross-document CR）

跨文档指代消解是指提及指向同一实体或事件可能跨越多个文档的子任务。CDML 提出了一种跨文档语言建模方法，该方法在拼接的相关文档上预训练 Longformer 编码器，并使用 MLP 进行二元分类，以确定一对 mention 是否共指。CrossCR 利用一个端到端模型进行跨文档指代消解，该模型在 gold mention spans 上预训练评分器。CR-RL 提出了一个基于 actor-critic 的深度强化学习的跨文档指代消解器。

#### 5.3.3 关系提取（Relation Extraction, RE）

关系提取涉及识别自然语言文本中提到的实体之间的语义关系。关系提取方法有两种类型：

- **句子级 RE**：关注于识别单个句子内实体之间的关系。Peng 等人和 TRE 引入了 LLM 来提高关系提取模型的性能。BERT-MTB 通过执行填空任务并结合提取关系的目标来学习基于 BERT 的关系表示。Curriculum-RE 利用课程学习，通过逐渐增加训练数据的难度来改进关系提取模型。RECENT 引入了 SpanBERT 并利用实体类型限制来减少噪声。Jiewen 通过将实体信息和标签信息结合到句子级嵌入中，扩展了 RECENT，使得嵌入能够感知实体标签。

- **文档级 RE（DocRE）**：旨在提取文档内跨多个句子的实体之间的关系。Hong 等人提出了一个强大的 DocRE 基线，通过用 LLM 替换 BiLSTM 主干。HIN 使用 LLM 对不同层次（实体、句子和文档层次）的实体表示进行编码和聚合。GLRE 是一个全局到局部网络，它使用 LLM 来编码文档信息，包括实体的全局和局部表示以及上下文关系表示。SIRE 使用两个基于 LLM 的编码器来提取句子内和句子间的关系。LSR 和 GAIN 提出了基于图的方法，在 LLM 之上引入图结构以更好地提取关系。DocuNet 将 DocRE 构建为一个语义分割任务，并在 LLM 编码器上引入了一个 U-Net 来捕捉实体之间的局部和全局依赖性。ATLOP 关注于 DocRE 中的多标签问题，可以用两种技术处理，即分类器的自适应阈值（adaptive thresholding for classifier）和 LLM 的局部上下文池化（localized context pooling for LLM）。DREEAM 进一步扩展和改进了 ATLOP，通过结合证据信息。

**端到端知识图谱构建**
目前，研究人员正在探索使用 LLM 进行端到端的知识图谱构建。[Kumar 等人](https://ieeexplore.ieee.org/document/9231227)提出了一种从原始文本构建知识图谱的统一方法，其中包含两个由 LLM 驱动的组件。他们首先在命名实体识别任务上对 LLM 进行微调，使其能够识别原始文本中的实体。然后，他们提出了另一个 “2-model BERT” 来解决关系提取任务，其中包含两个基于 BERT 的分类器。第一个分类器学习关系类别，而第二个二分类器学习两个实体之间关系的方向。预测的三元组和关系随后用于构建知识图谱。Guo 等人 提出了一个基于 BERT 的端到端知识提取模型，可应用于从古典中文文本构建知识图谱。Grapher 提出了一个新颖的端到端多阶段系统。它首先利用 LLM 生成KG 实体，然后通过一个简单的关系构建头，实现从文本描述中高效构建知识图谱。PiVE 提出了一个带有迭代验证框架的提示方法，该方法利用较小的 LLM（如 T5）来纠正较大的 LLM（例如 ChatGPT）生成的知识图谱中的错误。为了进一步探索先进的 LLM，[AutoKG](https://arxiv.org/pdf/2305.13168) 为不同的知识图谱构建任务（例如实体类型、实体链接和关系提取）设计了几个 prompt。然后采用这些 prompt 使用 ChatGPT 和 GPT-4 进行知识图谱构建。

#### 5.3.4 从 LLMs 中提取知识图谱

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-15.png" width="600" alt=""/>

*图20：从 LLMs 中提取知识图谱的一般框架。*

LLMs 已经被证明隐式地编码了大量的知识。如图 20 所示，一些研究旨在从 LLMs 中提取知识以构建 KGs。

- **COMET** 提出了 commonsense transformer model，通过使用现有的元组作为知识的种子集合进行训练，构建常识知识图谱（KGs）。利用这个种子集合，LLM 学习适应其学习到的表示以进行知识生成，并生成高质量的新元组。
- **BertNet** 提出了一个由 LLMs 驱动的自动 KG 构建框架。它只需要输入关系的最小定义，自动生成多样化的 prompt，并在给定的 LLM 内进行高效的知识搜索以获得一致的输出。构建的 KGs 在质量、多样性和新颖性方面表现出竞争力，包含了一系列以前方法无法提取的新的复杂关系。
- **West et al.** 提出了一个符号知识蒸馏框架，从 LLMs 中提取符号知识。他们首先通过从像 GPT-3 这样的大型 LLM 中提取常识性事实来微调一个小型学生 LLM。然后，利用学生 LLM 生成常识性 KGs。

### 5.4 LLM 增强知识图谱到文本生成

**知识图谱到文本（KG-to-text）生成的目标是生成高质量文本，准确且一致地描述输入的知识图谱信息**。KG-to-text 生成将知识图谱与文本连接起来，显著提高了知识图谱在更现实的自然语言生成（NLG）场景中的应用性。然而，收集大量的图谱-文本并行数据既具挑战性又成本高昂，导致训练不足和生成质量差。因此，许多研究工作采取以下两种方法来解决这个问题：1）**利用 LLM 的知识** 和 2）**构建大规模弱监督 KG-text 语料库**。

#### 5.4.1 利用来自 LLM 的知识

在使用 LLM 进行 KG-to-Text 生成的开创性研究工作中，Ribeiro 等人和 Kale 与 Rastogi 直接对各种 LLM（包括 BART 和 T5）进行了微调，目的是将 LLM 的知识转移到这项任务中。

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-16.png" width="600" alt=""/>

*图21：展示了知识图谱到文本生成的一般框架。*

如图 21 所示，这两项工作都**将输入 KG 表示为线性遍历**，这种简单的方法成功地超过了许多现有的最先进的 KG-to-text 生成系统。有趣的是，Ribeiro 等人还发现继续预训练可以进一步提高模型性能。然而，这些方法无法明确地将 KG 中丰富的图形语义纳入其中。为了增强 LLMs 的 KG 结构信息，JointGT 提出将保持 KG 结构的表示注入到 Seq2Seq 大型语言模型中。JointGT 还使用额外的预训练目标，包括给定掩码输入重构 KG 和文本的任务，以改善文本和图形信息之间的对齐。Li 等人关注少样本情况。它首先采用一种新颖的广度优先搜索策略来更好地遍历输入的 KG 结构，将增强的线性化 KG 表示输入到 LLMs 中以生成高质量的输出，然后对齐基于 GCN 和基于 LLM 的 KG 实体表示。Colas 等人首先将 KG 转换为适当的表示，然后再将 KG 线性化。接下来，通过全局注意机制（global attention mechanism）对每个 KG 节点进行编码，再通过图形感知注意模块（graph-aware attention module），最终解码为一系列 token。与这些工作不同，KGBART 保持了 KG 的结构，并利用图形注意力（graph attention）来聚合子 KG 中丰富的概念语义，从而增强了模型对未见概念集的泛化能力。

#### 5.4.2 构建大型弱知识图谱-文本对齐语料库

LLMs 的无监督预训练目标并不一定与 KG-to-text 的生成任务很好地对齐，这促使研究人员开发大规模的 KG-text 对齐语料库。Jin 等人从维基百科提出了一个 130 万的无监督知识图谱到文本的训练数据。具体来说，他们首先通过超链接和命名实体检测器检测文本中出现的实体，然后只添加与相应知识图谱共享一组共同实体的文本，这与关系提取任务中的距离监督思想类似。他们还提供了 1000 多个人工标注的知识图谱到文本的测试数据，以验证预训练的知识图谱到文本模型的有效性。同样，Chen 等人也提出了一个从英文 Wikidump 收集的知识图谱基础文本语料库。为了确保知识图谱和文本之间的联系，他们只提取至少有两个维基百科锚链接的句子。然后使用这些链接中的实体在 WikiData 中查询它们的邻居，并计算这些邻居与原始句子之间的词汇重叠。最后只选择重叠度高的对。

### 5.5 LLM 增强知识图谱问答

**知识图谱问答（KGQA）旨在基于存储在知识图谱中的结构化事实找到自然语言问题的答案。**KGQA 中不可避免的挑战是检索相关事实并将知识图谱的推理优势扩展到问答任务中。因此最近的研究采用 LLMs 来弥合自然语言问题和结构化知识图谱之间的差距。应用 LLMs 进行 KGQA 的一般框架如图 22 所示：

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-17.png" width="600" alt=""/>

*图22：使用 LLMs 进行知识图谱问答（KGQA）的通用框架。*

LLMs 可以用作：1）**实体/关系提取器**：和 2） **答案推理器**。


#### 5.5.1 LLMs 作为实体/关系提取器

实体/关系提取器旨在识别自然语言问题中提到的实体和关系，并在 KGs 中检索相关事实。Lukovnikov 等人首次利用 LLMs 作为关系预测的分类器，与浅层神经网络相比，性能显著提高。Nan 等人引入了两个基于 LLM 的 KGQA 框架，采用 LLMs 检测提到的实体和关系，然后使用提取的实体-关系对在 KGs 中查询答案。QA-GNN 使用 LLMs 对问题和候选答案对进行编码，用于估测相关 KG 实体的重要性。检索到的实体形成一个子图，通过图神经网络进行答案推理。Luo 等人使用 LLMs 计算关系和问题之间的相似性以检索相关事实，公式如下：
  
$$
s(r,q)=LLM(r)^{\top}LLM(q),   
$$

其中 $q$ 表示问题，$r$ 表示关系，$LLM(·)$ 分别为 $q$ 和 $r$ 生成表示。

Zhang 等人提出了一个基于 LLM 的路径检索器，逐跳检索与问题相关的关系并构建多条路径。每条路径的概率可以计算为：
  
$$
P(p|q)=\prod_{t=1}^{|p|}s(r_{t},q),   
$$

其中 $p$ 表示路径，$r_t$ 表示 $p$ 的第 $t$ 跳的关系。检索到的关系和路径可以用作上下文知识，以提高答案推理器的性能：
  
$$
P(a|q)=\sum_{p\in{\mathcal P }}P(a|p)P(p|q),   
$$

其中 ${\mathcal P}$ 表示检索到的路径，$a$ 表示答案。

#### 5.5.2 LLMs 作为答案推理器

答案推理器旨在对检索到的事实进行推理并生成答案。LLMs 可以作为答案推理器直接生成答案。例如，如图 22 所示，DEKCOR **将检索到的事实、问题和候选答案连接起来**：

$$ 
x = [\texttt{CLS}]~q~[\texttt{SEP}]~Related~Facts~[\texttt{SEP}]~a~[\texttt{SEP}]
$$

其中 $a$ 表示候选答案。然后输入到 LLMs 中来预测答案得分。在利用 LLMs 生成 $x$ 的 QA 上下文表示后，DRLK 提出一个动态层次推理器来捕捉 QA 上下文和答案之间的交互以预测答案。Yan 等人提出一个基于 LLM 的 KGQA 框架，包括两个阶段：(1) 从 KGs 中检索相关事实；(2) 基于检索到的事实生成答案。第一阶段类似于实体/关系提取器。给定一个候选答案实体 $a$，它从 KGs 中提取一系列路径 $p_1, . . . , p_n$。但第二阶段是一个基于 LLM 的答案推理器。它首先使用 KGs 中的实体名称和关系名称来表述路径。然后，它将问题 $q$ 和所有路径 $p_1, . . . , p_n$ 连接起来，形成一个输入样本

$$
x=\ [\texttt{CLS}]\ q\ [\texttt{SEP}]\ p_{1}\ [\texttt{SEP}] ~· · ·~ [\texttt{SEP}]~p_n~[\texttt{SEP}]
$$

这些路径被视为候选答案 $a$ 的相关事实。最后，它使用 LLMs 来预测假设：“a 是 q 的答案” 是否由这些事实支持，其公式为

$$
\begin{align}
e_{[\texttt{CLS}]} & =\mathrm{LLM}(x), \\
s & =\sigma(\mathrm{MLP}(e_{[\texttt{CLS}]})),
\end{align}
$$

使用 LLM 对 $x$ 进行编码，并将编码后 $[\texttt{CLS}]$ token 对应的表示输入到二元分类器中，$σ(·)$ 表示 sigmoid 函数。

为了更好地指导 LLMs 通过 KGs 进行推理，OreoLM 提出了一个知识交互层（Knowledge Interaction Layer，KIL），该层插入到 LLM 层之间。KIL 与 KG 推理模块交互，以发现不同的推理路径，然后推理模块可以对这些路径进行推理以生成答案。GreaseLM 融合了来自 LLMs 和图神经网络的表示，以有效地对 KG 事实和语言上下文进行推理。UniKGQA 将事实检索和推理统一到一个框架中。类似地，ReLMKG 在大型语言模型和相关知识图谱上进行联合推理。问题和表述的路径由语言模型编码，语言模型的不同层产生输出，指导图神经网络执行消息传递。这个过程利用结构化知识图谱中包含的显式知识进行推理。StructGPT 采用定制接口，允许大型语言模型（例如，ChatGPT）直接在 KGs 上进行推理以执行多步问答。

## 6 Synergized LLMs + KGs

LLMs 和 KGs 的协同作用近年来引起了越来越多的关注，它结合了 LLMs 和 KGs 的优点，以相互提高在各种下游应用中的性能。例如，LLMs 可用于理解自然语言，而 KGs 被视为一个知识库，提供事实知识。LLMs 和 KGs 的统一可能会产生一个强大的模型，用于知识表示和推理。在本节中，我们将从两个角度讨论最新的 Synergized LLMs + KGs：1）**协同知识表示**和 2）**协同推理**。代表性工作在表 4 中进行了总结：

| 任务                                | 方法      | 年份   |
|-------------------------------------|-------------|--------|
|Synergized Knowledge representation|JointGT <br>KEPLER <br>DRAGON <br> HKLM| 2021 <br>2021<br>2022<br>2023|
|Synergized Reasoning  | LARK<br> Siyuan et al. <br>KSL<br>StructGPT<br>Think-on-graph| 2023<br>2023<br>2023<br>2023<br>2023|

*表4：总结了协同 KGs 和 LLMs 的方法。*

### 6.1 协同知识表示

文本语料库和知识图谱都包含了大量的知识。然而，文本语料库中的知识通常是隐式的、非结构化的，而知识图谱中的知识则是显式的、结构化的。协同知识表示旨在设计一种能够**有效表示来自 LLM 和 KG 知识的协同模型**，可以更好地理解两个来源的知识，对许多下游任务都具有价值。

为了共同表示知识，研究人员通过引入额外的 KG 融合模块（KG fusion modules）提出了协同模型，这些模块与 LLM 一起进行联合训练。如图 23 所示，ERNIE 提出了一个文本-知识双编码器架构，其中 T-encoder 首先对输入句子进行编码，然后 K-encoder 处理知识图谱，并将其与 T-encoder 的文本表示融合。BERT-MK 采用了类似的双编码器架构，但在 LLM 预训练期间，在知识编码器组件中引入了邻近实体的额外信息。然而，KG 中的一些邻近实体可能与输入文本无关，导致额外的冗余和噪声。CokeBERT 关注这个问题，并提出了一个基于 GNN 的模块，使用输入文本过滤掉不相关的 KG 实体。JAKET 提出在大语言模型的中间融合实体信息。

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-18.png" width="600" alt=""/>

*图23：通过额外的 KG 融合模块实现的协同知识表示。*

KEPLER 提出了一个统一的模型，用于知识嵌入和预训练的语言表示。在 KEPLER 中，他们使用 LLM 对文本实体描述进行编码作为它们的嵌入，然后联合优化知识嵌入和语言建模目标。JointGT 提出了一个图-文联合表示学习模型，提出了三个预训练任务来对齐图和文本的表示。DRAGON 提出了一个自监督方法，从文本和 KG 预训练一个联合语言-知识基础模型。它以文本片段和相关 KG 子图为输入，双向融合来自两种模态的信息。然后，DRAGON 利用两个自监督推理任务，即掩码语言建模和 KG 链接预测来优化模型参数。HKLM 引入了一个统一的 LLM，它结合了 KG 来学习特定领域知识的表示。

### 6.2 协同推理

为了更好地利用文本语料库和知识图谱推理中的知识，协同推理旨在设计一个能够有效结合 LLMs 和 KGs 进行推理的模型。

#### LLM-KG 融合推理

LLM-KG 融合推理利用两个独立的 LLM 和 KG 编码器来处理文本和相关 KG 输入。这两个编码器同等重要，共同融合两个来源的知识进行推理。为了改善文本和知识之间的交互，KagNet 提出首先对输入的 KG 进行编码，然后增强输入文本表示。相比之下，MHGRN 使用输入文本的最终 LLM 输出来指导 KG 上的推理过程。然而，它们都只设计了文本和 KG 之间的单向交互。为了解决这个问题，QA-GNN 提出使用基于 GNN 的模型通过消息传递来共同推理输入上下文和 KG 信息。JointLK 提出了一个框架，通过 LM-to-KG 和 KG-to-LLM 双向注意力机制实现文本输入中任何 token 和任何 KG 实体之间的细粒度交互。如图 24 所示，计算所有文本 token 和 KG 实体之间的点积分数，分别计算双向注意力分数。此外，在每个 JointLK 层，还基于注意力分数动态修剪 KG，以便后续层可以专注于更重要的子 KG 结构。

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-19.png" width="600" alt=""/>

*图24：LLM-KG 融合推理的框架*

尽管有效，但在 JointLK 中，输入文本和 KG 之间的融合过程仍然使用最终 LLM 输出作为输入文本表示。GreaseLM 在 LLM 的每一层，在输入文本 token 和 KG 实体之间之间设计了深度且丰富交互。其架构和融合方法与第 6.1 节讨论的 ERNIE 大致相似，不同之处在于 GreaseLM 不使用 text-only T-encoder 来处理输入文本。

#### LLMs 作为代理推理

LLMs 也可以被当作代理与 KGs 交互进行推理，如图 25 所示。KD-CoT 迭代地从 KGs 中检索事实并生成可靠的推理过程，引导 LLMs 生成答案。KSL 教导 LLMs 在 KGs 上搜索以检索相关事实，然后生成答案。StructGPT 设计了几个 API 接口，允许 LLMs 访问结构化数据并通过遍历 KGs 进行推理。Think-on-graph 提供了一个灵活的即插即用框架，其中 LLM 代理在 KGs 上迭代执行**束搜索**（beam searches）以发现推理路径并生成答案。为了增强代理能力，AgentTuning 提出了几个指令微调数据集来指导 LLM 代理在 KGs 上进行推理。

<img src="https://masutangu-1259119800.cos.ap-shanghai.myqcloud.com/paper-note-unify-llm-and-kg/illustration-20.png" width="600" alt=""/>

*图25：使用 LLMs 作为在知识图谱上进行推理的代理。*

#### 比较和讨论

LLM-KG 融合推理结合了 LLM 编码器和 KG 编码器以统一方式表示知识，然后采用协同推理模块共同推理结果。这个框架允许使用不同的编码器和推理模块，它们被端到端训练以有效利用 LLMs 和 KGs 的知识和推理能力。然而，这些额外的模块可能会引入额外的参数和计算成本，同时缺乏可解释性。LLMs 作为 KG 推理代理提供了一个灵活的框架，在不需要额外训练成本的情况下进行 KG 推理，可以推广到不同的 LLMs 和 KGs。同时，推理过程是可解释的，可用于解释结果。然而，为 LLM 代理定义动作和策略也具有挑战性。LLMs 和 KGs 的协同仍然是一个正在进行的研究主题，未来可能会有更强大的框架出现。

## 7 未来方向与里程碑

在本节中，我们将讨论统一 KGs 和 LLMs 研究领域的未来方向和几个重要里程碑。

### 7.1 KGs 用于 LLMs 的幻觉检测

现有的方法通过在少量文档上训练神经分类器来检测幻觉，但这些方法既不够健壮也不够强大，无法处理不断增长的 LLMs。最近，研究人员尝试使用 KGs 作为外部来源来验证 LLMs。此外，进一步的研究将 LLMs 和 KGs 结合起来，实现了一个通用的事实核查模型，可以跨领域检测虚假信息或幻觉。

### 7.2 利用 KGs 编辑 LLMs 中的知识

LLMs 能够存储大量的现实世界知识，但无法快速更新内部知识以适应现实世界的变化。已经有一些研究努力提出在不重新训练整个 LLMs 的情况下编辑 LLMs 中的知识，但这些解决方案仍然存在性能不佳或计算开销大的问题。现有研究还表明，编辑单个事实会对其他相关知识产生连锁反应。因此，有必要开发一种更高效有效的编辑 LLMs 中知识的方法。最近，研究人员尝试利用 KGs 来高效地编辑 LLMs 中的知识。

### 7.3 KGs 对黑盒 LLMs 的知识注入

尽管预训练和知识编辑可以更新 LLMs 以跟上最新的知识，但它们仍然需要访问 LLMs 的内部结构和参数。然而，许多最先进的大型 LLMs 只为用户和开发者提供 API 访问，因此无法按照传统知识图谱注入方法，通过添加额外的知识融合模块来改变 LLM 结构。将各种类型的知识转换为不同的文本提示似乎是一个可行的解决方案。然而，尚不清楚这些提示是否能够很好地推广到新的 LLMs 上。此外，基于提示的方法受限于 LLMs 输入 token 的长度。因此，如何为黑盒 LLMs 实现有效的知识注入仍是我们需要探索的一个开放性问题。

### 7.4 多模态 LLMs 用于知识图谱

当前的知识图谱通常依赖于文本和图结构来处理与知识图谱相关的应用。然而，现实世界的知识图谱往往是由来自多种模态的数据构建的。因此，有效地利用来自多种模态的表示将是未来知识图谱研究的一个重要挑战。

### 7.5 LLMs 用于理解 KG 结构

传统的 LLMs 在纯文本数据上进行训练，并不是为了理解像知识图谱这样的结构化数据。因此，LLMs 可能无法完全把握或理解 KG 结构所传达的信息。一种简单的方法是将结构化数据线性化为 LLMs 可以理解的句子。然而，知识图谱的规模使得无法将整个 KG 线性化为输入。此外，线性化过程可能会丢失 KG 中的一些潜在信息。因此，有必要开发可以直接理解 KG 结构并对其进行推理的 LLMs。

### 7.6 协同 LLMs 和 KGs 进行双向推理

LLMs 和 KGs 是两种互补的技术，它们可以相互协同。然而，LLMs 和 KGs 的协同作用尚未被现有研究者充分探索。理想的 LLMs 和 KGs 协同作用将涉及利用两种技术的优势来克服它们各自的局限性。LLMs 在生成类似人类的文本和理解自然语言方面表现出色，而 KGs 是捕捉并以结构化方式表示知识的数据库。通过结合它们的能力，我们可以创建一个强大的系统，该系统既受益于 LLMs 的上下文理解，也受益于 KGs 的结构化知识表示。

为了更好地统一 LLMs 和 KGs，需要融合许多先进技术，如多模态学习、图神经网络和持续学习（continuous learning）。
