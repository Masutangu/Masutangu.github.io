---
layout: post
date: 2025-2-27T20:31:51+08:00
title: 【论文笔记】Agentic Retrieval-Augmented Generation - A Survey on Agentic RAG
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

本文是 [《Agentic Retrieval-Augmented Generation: A Survey on Agentic RAG》](https://arxiv.org/pdf/2501.09136) 的笔记。


Agentic RAG通过将自治 AI agent 嵌入 RAG 管道，利用 agent 设计模式中的**反思（reflection）、规划（planning）、工具使用（tool use）和多代理协作（multi-agent collaboration）**来动态管理检索策略，迭代地完善上下文理解，并通过明确定义的操作结果，从顺序步骤到自适应协作来调整工作流程。这种集成使 Agentic RAG 系统能够在各种应用中提供无与伦比的灵活性、可扩展性和上下文感知能力。

本调查全面探讨了 Agentic RAG，从其基础原则和 RAG 范式的演变开始。它详细介绍了 Agentic RAG 架构的分类法，重点介绍了在医疗、金融和教育等行业的关键应用，并研究了实际的实施策略。此外，它还解决了扩展这些系统的挑战，确保道德决策，并优化现实应用中的性能，同时提供了关于实现 Agentic RAG 的框架和工具的详细见解。

## Introduction

LLMs 由于依赖静态的预训练数据而面临重大限制。这种依赖通常导致信息过时、虚构的回答，以及无法适应动态的现实场景。这些挑战强调了需要能够整合实时数据并动态调整回答以保持上下文相关性和准确性的系统。

检索增强生成（RAG）作为应对这些挑战的一种有希望的解决方案出现。通过将 LLMs 的生成能力与外部检索机制相结合，RAG 系统提高了回答的相关性和及时性。这些系统从知识库、API或网络等来源实时检索信息，有效地弥合了静态训练数据与动态应用需求之间的差距。然而，传统的 RAG 工作流程仍受限于线性和静态的设计，限制了它们进行**复杂的多步推理、整合深层次的上下文理解和迭代改进回答的能力**。

代理（agents）的演进显著增强了 AI 系统的能力。现代代理是能够感知、推理和自主执行任务的智能实体。这些代理利用自主模式，如反思、规划、工具使用和多代理协作，以增强决策能力和适应性。

此外，这些代理采用自主工作流模式（参考阅读 1 和 2），如**提示链接（prompt chaining）**、**路由（routing）**、**并行化（parallelization）**、**编排器-工作者模型（orchestrator-worker model）**和**评估器-优化器（evaluator-optimizer）**来结构化和优化任务执行。通过整合这些模式，Agentic RAG 系统可以高效地管理动态工作流程并解决复杂的问题解决场景。RAG 和自主智能的融合催生了 Agentic RAG，这是一种将代理整合到 RAG 流程中的范式。Agentic RAG 实现了**动态检索策略、上下文理解和迭代改进**，实现了自适应和高效的信息处理。**与传统的 RAG 不同，Agentic RAG 利用自主代理来编排检索、筛选相关信息和改进回答，在需要精确性和适应性的场景中表现出色。**Agentic RAG 的概述如下图所示。

<img src="/assets/images/paper-note-agentic-rag/illustration-1.png" width="600" alt=""/>

本调查研究探讨了 Agentic RAG 的基本原理、分类和应用。它全面概述了 RAG 范式，如 Naïve RAG、Modular RAG 和 Graph RAG，以及它们演变为 Agentic RAG 系统的过程。主要贡献包括对 Agentic RAG 框架的详细分类、在医疗（参考阅读 3 和 4）、金融和教育（参考阅读 5）等领域的应用。

## Foundations of Retrieval-Augmented Generation

### Core Components of RAG

RAG 系统的架构集成了三个主要组件：

* **检索（Retrieval）**：负责查询外部数据源，如知识库、API 或向量数据库。先进的检索器利用密集向量搜索和基于 Transformer 的模型来提高检索精度和语义相关性
* **增强（Augmentation）**：处理检索到的数据，提取和总结与查询上下文相关的最相关信息
* **生成（Generation）**：将检索到的信息与 LLM 的预训练知识结合起来，生成连贯、与上下文相符的回答

### Evolution of RAG Paradigms

#### Naïve RAG

下图展示了 Naïve RAG 的简单检索-阅读工作流程，重点是基于关键词的检索和静态数据集。这些系统依赖于简单的基于关键词的检索技术，如 TF-IDF 和 BM25，从静态数据集中获取文档。然后，检索到的文档用于增强语言模型的生成能力。

<img src="/assets/images/paper-note-agentic-rag/illustration-2.png" width="600" alt=""/>

Naïve RAG 的特点是简单易实现，适用于涉及基于事实的查询且上下文复杂性较低的任务。然而，它存在一些限制：

* 缺乏上下文意识：由于依赖 Lexical matching 而非 semantic understanding，检索到的文档往往无法捕捉查询的语义细微差别
* 输出碎片化：缺乏高级预处理或上下文整合往往导致回答不连贯或过于通用
* 可扩展性问题：基于关键词的检索技术在处理大型数据集时存在困难，往往无法识别最相关的信息

#### Advanced RAG

Advanced RAG （参考阅读 6）系统在 Naïve RAG 的限制基础上进行了改进，通过融入语义理解和增强的检索技术。下图突出了检索中的语义增强以及 Advanced RAG 的迭代、上下文感知的流程。这些系统利用**密集检索模型（如 Dense Passage Retrieval，DPR）**和**神经排序算法**来提高检索精度。

<img src="/assets/images/paper-note-agentic-rag/illustration-3.png" width="600" alt=""/>

Advanced RAG 的核心特性包括：

* **密集向量搜索**：查询和文档以高维向量空间表示，从而实现用户查询和检索到的文档之间更好的语义对齐
* **上下文重新排序**：神经模型重新对检索到的文档进行排序，以优先考虑最相关的上下文信息
* **迭代检索**：Advanced RAG 引入了多跳检索机制，使得在复杂查询中可以跨多个文档进行推理

这些进展使得 Advanced RAG 适用于需要高精度和细致理解的应用，例如研究综述和个性化推荐。然而，仍然存在一些挑战，比如计算开销和有限的可扩展性，特别是在处理大型数据集或多步查询时。

#### Modular RAG

Modular RAG（参考阅读 6）是 RAG 范式的最新演进，强调**灵活性**和**定制性**。这些系统将检索和生成流程分解为独立、可重用的组件，从而实现领域特定的优化和任务适应性。下图展示了 Modular RAG 的架构，展示了混合检索策略、可组合的流程和外部工具集成。

<img src="/assets/images/paper-note-agentic-rag/illustration-4.png" width="600" alt=""/>

Modular RAG 的关键创新包括：

* **混合检索策略**：**将稀疏检索方法**（例如 sparse encoder-BM25）**与密集检索技术**（参考阅读 7，例如 DPR - Dense Passage Retrieval）**相结合**，以在不同类型的查询中最大化准确性
* **工具集成**：将外部 API、数据库或计算工具纳入系统，用于处理特定任务，如实时数据分析或领域特定计算
* **可组合的流程**：模块化 RAG 使得检索器、生成器和其他组件可以独立替换、增强或重新配置，从而实现对特定用例的高度适应性

例如，一个专为金融分析设计的 Modular RAG 系统可以通过 API 获取实时股票价格，利用密集检索分析历史趋势，并通过定制的语言模型生成可操作的投资见解。这种模块化和定制性使得 Modular RAG 非常适合处理复杂的、多领域的任务，既具有可扩展性又具有精确性。

#### Graph RAG

Graph RAG（参考阅读 8）通过集成基于图的数据结构扩展了传统的检索增强生成系统，如下图所示：

<img src="/assets/images/paper-note-agentic-rag/illustration-5.png" width="600" alt=""/>

这些系统利用图数据中的关系和层次结构来增强多跳推理和上下文丰富化。通过整合基于图的检索，Graph RAG 能够产生更丰富、更准确的生成输出，特别适用于需要关系理解的任务。

Graph RAG 的特点包括：

* **节点连接性**：捕捉并推理实体之间的关系
* **分层知识管理**：通过基于图的层次结构处理结构化和非结构化数据
* **上下文丰富**：通过利用基于图的路径增加关系理解

然而，Graph RAG也有一些限制：

* **有限的可扩展性**：依赖于图结构可能会限制可扩展性，特别是在处理大量数据源时
* **数据依赖性**：高质量的图数据对于有意义的输出至关重要，这限制了它在非结构化或标注不完善的数据集中的适用性
* **集成复杂性**：将图数据与非结构化检索系统集成增加了设计和实现的复杂性

Graph RAG 非常适用于医疗诊断、法律研究和其他需要对结构化关系进行推理的领域

#### Agentic RAG

Agentic RAG 引入了具有动态决策能力和工作流优化能力的自主代理，**代表了一种范式转变**。与静态系统不同，Agentic RAG 采用**迭代改进**和**自适应检索策略**来处理复杂的、实时的、多领域的查询。这种范式利用了检索和生成过程的模块化，同时引入了基于代理的自治性。

Agentic RAG 的关键特性包括：

* **自主决策**：代理根据查询的复杂性独立评估和管理检索策略
* **迭代改进**：引入反馈循环以提高检索准确性和响应相关性
* **工作流优化**：动态编排任务，实现实时应用的高效性

尽管 Agentic RAG 取得了进展，但也面临一些挑战：

* **协调复杂性**：管理代理之间的交互需要复杂的编排机制
* **计算开销**：使用多个代理增加了复杂工作流的资源需求
* **可扩展性限制**：虽然具有可扩展性，但系统的动态性可能会对高查询量的计算资源造成压力

Agentic RAG 在客户支持、金融分析和自适应学习平台等领域表现出色，其中动态适应性和上下文精确性至关重要。

### Challenges and Limitations of Traditional RAG Systems

最显著的局限性主要集中在**上下文整合**、**多步推理**以及**可扩展性**和**延迟**问题上。

#### Contextual Integration

即使 RAG 系统成功检索到相关信息，它们通常难以将其无缝地融入生成的响应中。检索流程的静态性和有限的上下文意识导致输出结果零散、不一致或过于通用。

例如：对于一个查询，比如“阿尔茨海默病研究的最新进展及其对早期治疗的影响是什么？”，可能会得到相关的研究论文和医疗指南。然而，传统的 RAG 系统往往无法将这些发现综合成一个连贯的解释，将新的治疗方法与具体的患者情况联系起来。同样，对于一个类似“干旱地区小规模农业的最佳可持续实践是什么？”的查询，传统系统可能会检索到关于一般农业方法的文件，但忽视了针对干旱环境量身定制的关键可持续实践。

#### Multi-Step Reasoning

许多实际查询需要迭代或多跳推理，即在多个步骤中检索和综合信息。传统的 RAG 系统往往无法根据中间洞察或用户反馈来优化检索，导致响应不完整或不连贯。

例如：一个复杂的查询，比如“欧洲可再生能源政策中的经验教训如何适用于发展中国家，可能产生哪些经济影响？”需要协调多种类型的信息，包括政策数据、发展地区的情境化信息和经济分析。传统的 RAG 系统通常无法将这些不同的要素连接成一个连贯的响应。

#### Scalability and Latency Issues

随着外部数据源的增加，查询和对大型数据集进行排序变得越来越需要大量的计算资源。这导致显著的延迟，削弱了系统在实时应用中提供及时响应的能力。

例如：在对时间敏感的场景中，如金融分析或实时客户支持中，由于查询多个数据库或处理大量文档集而导致的延迟可能会阻碍系统的整体效用。在高频交易中延迟检索市场趋势可能会导致错失机会。

### Agentic RAG: A Paradigm Shift

传统的 RAG 系统由于其静态工作流程和有限的适应性，往往难以处理动态的、多步推理和复杂的实际任务。这些局限性促使了智能代理的整合，从而产生了 Agentic RAG。Agentic RAG 整合了能够进行动态决策、迭代推理和自适应检索策略的自主代理，且通过优化的工作流程减少了延迟，并通过迭代地改进输出来解决了传统 RAG 系统在可扩展性和有效性方面的历史性挑战。

## Core Principles and Background of Agentic Intelligenc

智能代理是 Agentic RAG 系统的基础。从本质上讲，一个 AI 代理包括以下组成部分：

* **LLM（具有定义的角色和任务）**：作为代理的主要推理引擎和对话接口，负责解释用户查询，生成响应，并保持连贯性
* **记忆（短期和长期）**：在交互过程中捕捉上下文和相关数据。短期记忆跟踪即时对话状态，而长期记忆存储积累的知识和代理经验（参考阅读 9）
* **规划（反思和自我批评）**：通过反思、查询路由或自我批评（参考阅读 10）指导代理的迭代推理过程，确保有效地拆分复杂任务（参考阅读 11）
* **工具（向量搜索、网络搜索、API 等）**：扩展代理的能力，使其不仅限于文本生成，还能够访问外部资源、实时数据或专门的计算

<img src="/assets/images/paper-note-agentic-rag/illustration-6.png" width="600" alt=""/>

Agentic Patterns（参考阅读 12 和 13）提供了结构化方法以指导 Agentic RAG 系统中代理的行为。这些模式使代理能够动态适应、规划和协作，确保系统能够以精确性和可扩展性处理复杂的实际任务。有四个关键模式支撑着 agentic 工作流程：**反思（reflection）**、**规划（planning）**、**工具使用（tool use）**和**多代理协作（multi-agent collaboration）**。

### Reflection

**反思是 agentic 工作流程其中一个基础设计模式，使代理能够迭代地评估和改进其输出。**通过引入自我反馈机制，代理可以识别和解决错误、不一致性和改进空间，提高在代码生成、文本生成和问题回答等任务中的性能。在实际应用中，反思涉及促使代理对其输出进行正确性、风格和效率的批判，并将这些反馈纳入后续的迭代中。外部工具如单元测试或网络搜索，可以进一步增强这个过程，验证结果并突出差距。

**在多代理系统中，反思可以涉及不同的角色，例如一个代理生成输出，而另一个代理对其进行批判，促进协作改进。**例如，在法律研究中，代理可以通过重新评估检索到的案例法来迭代地改进回答，确保准确性和全面性。反思在 Self-Refine、Reflexion 和 CRITIC 等研究中已经展示了显著的性能改进。

### Planning

**规划是 agentic 工作流程中的一个关键设计模式，使代理能够自主地将复杂任务分解为更小、可管理的子任务。**这种能力对于在动态和不确定的情境中进行多跳推理和迭代问题解决至关重要。

通过利用规划，代理可以动态确定完成更大目标所需的步骤顺序。这种适应性使代理能够处理无法预定义的任务，确保决策的灵活性。虽然强大，**但与反思等确定性工作流程相比，规划可能产生较不可预测的结果**。

### Tool Use

工具使用使代理能够通过与外部工具、API 或计算资源的交互来扩展其能力。这种模式使代理能够获取信息、进行计算和操作超出其预训练知识范围的数据。通过将工具动态集成到工作流程中，代理可以适应复杂任务并提供更准确和与上下文相关的输出。

现代 agentic 工作流程在各种应用中都采用了工具使用，包括信息检索、计算推理和与外部系统的接口。这种模式的实现已经随着 GPT-4 的函数调用能力和能够管理多个工具访问的系统等进展而发展得非常显著。这些发展促进了复杂工作流程的实现，其中代理自主选择并执行与给定任务最相关的工具。

尽管工具使用显著增强了 agentic 工作流程，但在优化工具选择方面仍存在挑战，特别是在可用选项众多的情况下。

### Multi-Agent

**多代理协作（参考资料 14）是 agentic 工作流程中的一个关键设计模式，它实现了任务专业化和并行处理。**代理之间进行通信和共享中间结果，确保整体工作流程高效和连贯。通过将子任务分配给专门的代理，这种模式提高了复杂工作流程的可扩展性和适应性。多代理系统允许开发人员将复杂的任务分解为较小、可管理的子任务，并分配给不同的代理。这种方法不仅提高了任务性能，还为管理复杂交互提供了一个强大的框架。每个代理都有自己的记忆和工作流程，可以包括使用工具、反思或规划，实现动态和协作的问题解决。

尽管多代理协作提供了巨大的潜力，但与反思和工具使用等更成熟的工作流程相比，**它是一个较不可预测的设计模式**。然而，新兴的框架如 AutoGen、Crew AI 和 LangGraph 为实现有效的多代理解决方案提供了新的途径。

这些设计模式构成了 Agentic RAG 系统成功的基础。通过构建工作流程，从简单的顺序步骤到更具适应性和协作性的过程，这些模式使系统能够动态调整其检索和生成策略，以适应多样化和不断变化的现实环境的需求。利用这些模式，代理能够处理迭代的、上下文感知的任务，远远超出传统 RAG 系统的能力。

## Agentic Workflow Patterns: Adaptive Strategies for Dynamic Collaboration

Agentic 工作流模式对基于 LLM 的应用进行**结构化**，以优化性能、准确性和效率。根据任务的复杂性和处理要求可以采用不同的方法。

### Prompt Chaining: Enhancing Accuracy Through Sequential Processing

**提示链接将复杂任务分解为多个步骤，每个步骤都建立在前一个步骤的基础上。**这种结构化方法通过在继续前进之前简化每个子任务来提高准确性。然而，由于顺序处理，它可能会增加延迟。

当一个任务可以分解为固定的子任务，并且每个子任务都对最终输出有贡献时，这种工作流程最为有效。在需要逐步推理以提高准确性的场景中，它特别有用。

### Routing: Directing Inputs to Specialized Processes

**路由涉及对输入进行分类，并将其引导到适当的专门提示或处理过程。**这种方法确保不同的查询或任务被单独处理，提高了效率和响应质量。

### Parallelization: Speeding Up Processing Through Concurrent Execution

**并行化将一个任务分解为同时运行的独立的进程，从而减少延迟并提高吞吐量。**它可以分为分段（独立子任务）和投票（多个输出以提高准确性）两种类型。

### Orchestrator-Workers: Dynamic Task Delegation

利用中央协调模型动态地将任务分解为子任务，分配给专门的工作模型，并编译结果。与并行化不同，它能够适应不同的输入复杂性。

### Evaluator-Optimizer: Refining Output Through Iteration

评估器-优化器工作流程通过生成初始输出并根据评估模型的反馈进行改进，迭代地提高内容质量。

## Taxonomy of Agentic RAG Systems

Agentic RAG 系统可以根据其复杂性和设计原则划分为不同的架构框架。这些框架包括单代理架构、多代理系统和分层代理架构。每个框架都经过量身定制，以解决特定的挑战，并优化各种应用的性能。本节提供了这些架构的详细分类，突出它们的特点、优势和局限性。

### Single-Agent Agentic RAG: Router

单代理 Agentic RAG 作为一个集中的决策系统，其中一个单一代理负责管理信息的检索、路由和整合。这种架构通过将这些任务整合到一个统一的代理中简化了系统，特别适用于具有有限工具或数据源的设置。

### Multi-Agent Agentic RAG Systems

多代理 Agentic RAG 是单代理架构的模块化和可扩展演进，旨在通过利用多个专门的代理来处理复杂的工作流程和多样化的查询类型（如下图所示）。该系统不再依赖于单个代理来管理所有任务（推理、检索和响应生成），而是将责任分配给多个代理，每个代理针对特定的角色或数据源进行了优化。

<img src="/assets/images/paper-note-agentic-rag/illustration-7.png" width="600" alt=""/>

**工作流程：**

1. **查询提交**：用户查询，由协调代理或主检索代理接收。该代理充当中央协调器，根据查询的要求将查询委派给专门的检索代理
2. **专门的检索代理**：查询被分发给多个检索代理，每个代理专注于特定类型的数据源或任务。例如：
    * 代理1：处理结构化查询，如与基于 SQL 的数据库（如 PostgreSQL 或 MySQL）进行交互
    * 代理2：管理语义搜索，从 PDF、书籍或内部记录等来源检索非结构化数据
    * 代理3：专注于从网络搜索或 API 中检索实时公共信息
    * 代理4：专门处理推荐系统，根据用户行为或个人资料提供上下文感知的建议
  
3. **工具访问和数据检索**：每个代理将查询路由到其领域内适当的工具或数据源，例如：
    * 向量搜索：用于语义相关性
    * Text-to-SQL：用于结构化数据
    * 网络搜索：用于实时公共信息
    * API：用于访问外部服务或专有系统

    检索过程并行执行，可以高效处理多样化的查询类型。

4. **数据整合和 LLM 综合**：LLM 将检索到的信息综合成一份连贯且与上下文相关的响应，无缝地整合多个来源的见解
5. **输出生成**：系统生成一份全面的响应，并以可操作且简洁的格式返回给用户

### Hierarchical Agentic RAG Systems

多层次 Agentic RAG 系统 （参考资料 15）采用了结构化的、多层次的方法进行信息检索和处理，如下图所示，从而提高了效率和战略决策能力。该系统中的代理人按照层次结构进行组织，高层代理人负责监督和指导低层代理人。这种结构实现了多层次的决策，确保查询由最合适的资源处理。

<img src="/assets/images/paper-note-agentic-rag/illustration-8.png" width="600" alt=""/>

**工作流程：**

1. **查询接收**：用户提交查询，由顶层 agent 接收，其负责初始评估和委派
2. **战略决策**：顶层代理人评估查询的复杂性，根据查询的领域，决定优先考虑哪些下级 agent 或数据源
3. **委派给下级 agent**：顶层 agent 将任务分配给专门从事特定检索方法的低层 agent（例如 SQL 数据库、网络搜索或专有系统）。这些 agent 独立执行其分配的任务
4. **聚合和整合**：下级 agent 的结果由更高层的 agent 收集和整合，将信息综合成一份连贯的回复
5. **回复发送**：最终综合的答案返回给用户，确保回复既全面又与上下文相关。

### Agentic Corrective RAG

Corrective RAG 引入了自我纠正检索结果的机制，通过在工作流中嵌入智能 agent，迭代地改进上下文文档和响应，最小化错误并最大化相关性。Corrective RAG 的核心原则在于其能够动态评估检索到的文档，执行纠正操作，并优化查询以提高生成响应的质量。Corrective RAG 采用的方法如下：

* **文档相关性评估**：由相关性评估 agent 对检索到的文档进行评估。低于相关性阈值的文档会触发纠正步骤
* **查询优化和扩充**：由查询优化 agent 对查询进行优化，利用语义理解来优化检索以获得更好的结果
* **动态从外部来源检索**：当上下文不足时，外部知识检索 agent 执行网络搜索或访问其他数据源以补充检索到的文档
* **响应综合**：所有经过验证和改进的信息传递给响应综合 agent 进行最终的响应生成

### Adaptive Agentic RAG

Adaptive RAG（参考阅读 16）通过根据传入查询的复杂性动态调整查询处理策略，提高了 LLM 的灵活性和效率。与静态检索工作流不同，Adaptive RAG（参考阅读 17）使用分类器评估查询的复杂性，并确定最合适的方法，从单步检索到多步推理，甚至对于简单的查询完全绕过检索。

Adaptive RAG 的核心思想在于根据查询的复杂性动态调整检索策略：

* 简单查询：对于不需要额外检索的基于事实的问题，系统直接使用现有知识生成答案
* 中等复杂查询：对于需要较少上下文的中等复杂任务，系统执行单步检索以获取相关细节
* 复杂查询：对于需要多层次推理的复杂查询，系统采用多步检索，逐步优化中间结果以提供全面的答案

Adaptive RAG 系统建立在三个主要组件上：

* **分类器角色**：
  * 使用较小的语言模型分析查询以预测其复杂性
  * 分类器使用自动标注的数据集进行训练，这些数据集是根据过去模型结果和查询模式生成的
* **动态策略选择**：
  * 对于简单的查询，系统避免不必要的检索，直接利用语言模型生成响应
  * 对于中等复杂的查询，系统采用单步检索过程来获取相关上下文
  * 对于复杂的查询，系统激活多步检索，确保迭代优化和增强推理能力
* **LLM 集成**：
  * LLM 将检索到的信息综合成一份连贯的响应
  * LLM 和分类器之间的迭代交互使得对于复杂查询的优化得以实现

### Graph-Based Agentic RAG

#### Agent-G: Agentic Framework for Graph RAG

Agent-G（参考阅读 18）引入了一种新颖的代理架构，将图知识库与非结构化文档检索相结合。通过结合结构化和非结构化数据源，该框架提高了 RAG 系统的推理和检索准确性。它采用模块化的检索器库、动态代理交互和反馈循环，以确保高质量的输出。

#### GeAR: Graph-Enhanced Agent for Retrieval-Augmented Generation

GeAR（参考阅读 19）引入了一个 agent 框架，通过整合基于图的检索机制，增强了传统的 RAG 系统。通过利用图扩展技术和基于 agent 的架构，GeAR 解决了多跳检索场景中的挑战，提高了系统处理复杂查询的能力，如下图所示：

<img src="/assets/images/paper-note-agentic-rag/illustration-9.png" width="600" alt=""/>

### Agentic Document Workflows in Agentic RAG

Agentic Document Workflows（ADW，参考阅读 20）通过实现端到端的知识工作自动化，扩展了传统的 RAG 范式。这些工作流程协调以文档为中心的复杂过程，集成了文档解析、检索、推理和结构化输出，并与智能代理结合。ADW 系统通过维护状态、协调多步骤工作流程以及对文档应用特定领域的逻辑，**解决了智能文档处理（Intelligent Document Processing，IDP）和 RAG 的局限性**。

**工作流程：**

1. **文档解析和信息结构化**：
   * 使用企业级工具（例如 LlamaParse）对文档进行解析，提取相关的数据字段，如发票号码、日期、供应商信息、项目明细和付款条件
   * 对结构化数据进行组织，以便进行后续处理
2. **跨过程状态维护**：
   * 系统维护有关文档上下文的状态，确保在多步骤工作流程中的一致性和相关性
   * 跟踪文档在各个处理阶段的进度
3. **知识检索**：
   * 从外部知识库（例如 LlamaCloud）或向量索引中检索相关的参考信息
   * 检索实时的、领域特定的指南，以增强决策能力
4. **代理协调**：
   * 智能代理应用业务规则，进行多跳推理，并生成可操作的建议
   * 协调解析器、检索器和外部 API 等组件，实现无缝集成
5. **可操作输出生成**：
   * 输出以结构化格式呈现，针对特定的用例进行定制
   * 将建议和提取的见解综合成简明扼要的可操作报告
  
## Tools and Frameworks for Agentic RAG

关键工具和框架：

* LangChain 和 LangGraph：LangChain 提供了构建 RAG 流水线的模块化组件，无缝集成了检索器、生成器和外部工具。LangGraph 通过引入基于图的工作流程，支持循环、状态持久化和人机交互，为代理系统中的复杂协调和自我修正机制提供了补充。
* LlamaIndex：LlamaIndex 的 Agentic Document Workflows (ADW) 实现了文档处理、检索和结构化推理的端到端自动化。它引入了元代理架构，其中子代理管理较小的文档集合，通过顶级代理进行协调，执行合规性分析和上下文理解等任务。
* Hugging Face Transformers 和 Qdrant：Hugging Face 提供了用于嵌入和生成任务的预训练模型，而 Qdrant 通过自适应向量搜索能力增强了检索工作流程，允许代理通过在稀疏和密集向量方法之间动态切换来优化性能。
* CrewAI 和 AutoGen：这些框架强调多代理体系结构。CrewAI 支持分层和顺序处理、强大的记忆系统和工具集成。AG2（以前称为 AutoGen）在多代理协作方面表现出色，具有先进的代码生成、工具执行和决策支持。
* OpenAI Swarm Framework：这是一个教育框架，旨在实现人性化、轻量级的多代理协调，强调代理的自治性和结构化协作。
* Agentic RAG with Vertex AI：由 Google 开发，Vertex AI 与 Agentic RAG 无缝集成，提供了一个平台来构建、部署和扩展机器学习模型，同时利用先进的 AI 能力实现强大的、具有上下文意识的检索和决策工作流程。
* Semantic Kernel：Semantic Kernel 是微软开源的软件开发工具包，将 LLM 集成到应用程序中。它支持代理模式，实现了自主的 AI 代理，用于自然语言理解、任务自动化和决策。它已经在 ServiceNow 的 P1 事故管理等场景中被使用，以促进实时协作、自动化任务执行和无缝检索上下文信息。
* Amazon Bedrock for Agentic RAG：Amazon Bedrock 提供了一个强大的平台，用于实现 Agentic RAG 工作流程。
* IBM Watson和Agentic RAG：IB M的 watsonx.ai 支持构建 Agentic RAG 系统，通过集成外部信息和提高响应准确性，使用 Granite-3-8B-Instruct 模型来回答复杂的查询。
* Neo4j 和向量数据库：Neo4j 是一个知名的开源图数据库，擅长处理复杂的关系和语义查询。除了 Neo4j 之外，像 Weaviate、Pinecone、Milvus 和 Qdrant 这样的向量数据库提供高效的相似性搜索和检索能力，构成了高性能的 Agentic RAG 工作流程的基础。

## Benchmarks and Datasets

当前的基准测试和数据集为评估具有代理和基于图增强的 RAG 系统提供了有价值的见解。虽然有些数据集是专门为 RAG 设计的，但其他数据集则是为了在不同场景中测试检索、推理和生成能力而进行了调整。数据集对于测试 RAG 系统的检索、推理和生成组件至关重要。

基准测试在标准化RAG系统的评估中起着关键作用，提供了结构化任务和指标。以下基准测试尤为重要：

* BEIR（Benchmarking Information Retrieval）：一个多功能的基准，旨在评估嵌入模型在各种信息检索任务中的表现，涵盖了来自生物信息学、金融和问答等不同领域的 17 个数据集。
* MS MARCO（Microsoft Machine Reading Comprehension）：这个基准测试专注于段落排序和问答，广泛用于 RAG 系统中的密集检索任务。
* TREC（Text REtrieval Conference，Deep Learning Track）：提供了用于段落和文档检索的数据集，强调检索流程中 ranking 模型的质量。
* MuSiQue（Multihop Sequential Questioning）：这是一个用于跨多个文档的多跳推理的基准测试，强调从不相关的上下文中检索和综合信息的重要性。
* 2WikiMultihopQA：这是一个专为跨两个维基百科文章的多跳问答任务设计的数据集，重点是能够在多个来源之间连接知识。
* AgentG（Agentic RAG for Knowledge Fusion）：这是一个专为 Agentic RAG 任务量身定制的基准测试，评估在多个知识库之间进行动态信息合成的能力。
* HotpotQA：这是一个多跳问答基准测试，需要在相互关联的上下文中进行检索和推理，非常适合评估复杂的 RAG 工作流程。
* RAGBench：这是一个大规模的可解释性基准测试，涵盖了行业领域的 10 万个示例，使用 TRACe 评估框架来提供可操作的 RAG 指标。
* BERGEN（Benchmarking Retrieval-Augmented Generation）：这是一个用于系统性地对RAG系统进行基准测试的库，包含了标准化的实验。
* FlashRAG Toolkit：实现了 12 种 RAG 方法，并包含 32 个基准测试数据集，以支持高效和标准化的 RAG 评估。
* GNN-RAG：这个基准测试评估基于图的 RAG 系统在节点级和边级预测等任务上的表现，重点关注知识图谱问答（KGQA）中的检索质量和推理性能。

以下表格是用于评估 RAG 的下游任务和数据集：

<img src="/assets/images/paper-note-agentic-rag/illustration-10.png" width="600" alt=""/>


## 参考阅读

1. [Building effective agents](https://www.anthropic.com/research/building-effective-agents)
2. [Workflows and Agents](https://langchain-ai.github.io/langgraph/tutorials/workflows/)
3. [Revolutionizing mental health care through langchain: A journey with a large language model](https://arxiv.org/abs/2403.05568)
4. [The potential of large language models in recognizing symptoms of common illnesses](https://arxiv.org/abs/2405.06712)
5. [Encouraging responsible use of generative ai in education: A reward-based learning approach](https://arxiv.org/abs/2407.15022)
6. [Retrieval-augmented generation for large language models: A survey](https://arxiv.org/abs/2312.10997)
7. [Dense passage retrieval for open-domain question answering](https://arxiv.org/abs/2004.04906)
8. [Graph retrieval-augmented generation: A survey](https://arxiv.org/abs/2408.08921)
9. [A survey on the memory mechanism of large language model based agents](https://arxiv.org/abs/2404.13501)
10. [CRITIC: Large Language Models Can Self-Correct with Tool-Interactive Critiquing](https://arxiv.org/abs/2305.11738)
11. [Understanding the planning of llm agents: A survey](https://arxiv.org/abs/2402.02716)
12. [Enhancing ai systems with agentic workflows patterns in large language model](https://ieeexplore.ieee.org/document/10578990)
13. [How agents can improve llm performance](https://www.deeplearning.ai/the-batch/how-agents-can-improve-llm-performance/?ref=dl-staging-website.ghost.io)
14. [Large language model based multi-agents: A survey of progress and challenges](https://arxiv.org/abs/2402.01680)
15. [Agentic retrieval-augmented generation for time series analysis](https://arxiv.org/abs/2408.14484)
16. [Adaptive-rag: Learning to adapt retrieval-augmented large language models through question complexity](https://arxiv.org/abs/2403.14403)
17. [LangGraph Adaptive RAG](https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_adaptive_rag/)
18. [Agent-g: An agentic framework for graph retrieval augmented generation](https://openreview.net/forum?id=g2C947jjjQ)
19. [Graph-enhanced agent for retrieval-augmented generation](https://arxiv.org/abs/2412.18431)
20. [Introducing agentic document workflows](https://www.llamaindex.ai/blog/introducing-agentic-document-workflows)
