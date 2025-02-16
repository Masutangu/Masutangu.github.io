---
layout: post
date: 2025-2-16T11:23:24+08:00
title: 【论文笔记】Group Relative Policy Optimization (GRPO)
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

本文是 [《DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models》](https://arxiv.org/pdf/2402.03300) 中 GRPO 部分的笔记。DeepSeek 引入了 Group Relative Policy Optimization (GRPO)，是 Proximal Policy Optimization (PPO)的一种变体，它增强了数学推理能力，同时优化了 PPO 的内存使用。

## PPO

Proximal Policy Optimization (PPO) 是一种广泛应用于 LLM 的强化学习微调阶段的 actor-critic RL 算法。它通过最大化以下替代目标（surrogate objective）来优化 LLM：

$$\mathcal{J}_{\text{PPO}}(\theta)=\mathbb{E}[q\sim P(Q),o\sim\pi_{\theta_{\text{old}}}(O|q)]\frac{1}{|o|}\sum_{t=1}^{|o|}\min\left[\frac{\pi_{\theta}(o_t|q,o_{<t})}{\pi_{\theta_{\text{old}}}(o_t|q,o_{<t})}A_t,\text{clip}\left(\frac{\pi_{\theta}(o_t|q,o_{<t})}{\pi_{\theta_{\text{old}}}(o_t|q,o_{<t})},1-\varepsilon,1+\varepsilon\right)A_t\right] \tag{1}$$


在上述公式中，$\pi_{\theta}$ 和 $\pi_{\theta_{old}}$ 分别表示当前策略模型和旧策略模型，$q$ 是从问题数据集采样的出的问题，$o$ 是从旧策略 $\pi_{\theta_{old}}$ 中采样得到的输出。$\varepsilon$ 是 PPO 中引入的裁剪相关的超参数，用于稳定训练。$A_t$ 是优势值，通过 Generalized Advantage Estimation（GAE），基于奖励 $\\{ r \geq t \\}$ 和学习到的值函数 $V_{\psi}$ 来计算。因此**在 PPO 中，需要同时训练值函数和策略模型**。为了减轻奖励模型的过度优化，标准做法是在每个 token 的奖励中添加来自参考模型的 KL 惩罚项（避免和参考模型输出的 token 分布差异过大）：

$$r_{t}=r_{\varphi}(q,o_{\leq t})-\beta\log{\frac{\pi_{\theta}(o_{t}|q,o_{\leq t})}{\pi_{r e f}(o_{t}|q,o_{\leq t})}} \tag{2}$$

其中，$r_{\varphi}$ 是奖励模型，$\pi_{r e f}$ 是参考模型，通常是初始的 SFT 模型，$\beta$ 是 KL 惩罚项的系数。

## From PPO to GRPO

**由于 PPO 中使用的值函数通常是与策略模型具有相当大小的另一个模型，这会带来相当大的内存和计算负担**。此外，在 RL 训练过程中，值函数被视为计算优势的基准，用于方差的减少。然而，在 LLM 的背景下，通常只有最后一个 token 会被奖励模型分配一个奖励分数，这可能会导致训练一个在每个 token 上都准确的价值函数变得复杂。


<img src="/assets/images/grpo/illustration-1.png" width="600" alt=""/>

*算法1：迭代式组相对策略优化*


为了解决这个问题，如上图所示，DeepSeek 提出了 Group Relative Policy Optimization (GRPO)，它不需要像 PPO 那样进行额外的值函数逼近，而是使用对同一个问题产生的多个采样输出的平均奖励作为基准。具体而言，对于每个问题 $q$，GRPO 从旧策略 $\pi_{\theta_{old}}$ 中采样一组输出 $\\{o_1, o_2,···, o_G\\}$，然后通过最大化以下目标来优化策略模型：

$$\mathcal{J}_{\text{GRPO}}(\theta)=\mathbb{E}[q\sim P(Q),\{o_i\}^G_{i=1}\sim\pi_{\theta_{\text{old}}}(O|q)] \\ \frac{1}{G}\sum^G_{i=1}\frac{1}{|o_i|}\sum_{t=1}^{|o_i|}\{\min\left[\frac{\pi_{\theta}(o_{i,t}|q,o_{i,<t})}{\pi_{\theta_{\text{old}}}(o_{i,t}|q,o_{i,<t})} \hat{A}_{i,t},\text{clip}\left(\frac{\pi_{\theta}(o_{i,t}|q,o_{i,<t})}{\pi_{\theta_{\text{old}}}(o_{i,t}|q,o_{i,<t})},1-\varepsilon,1+\varepsilon\right)\hat{A}_{i,t}\right]-\beta \mathbb{D}_{KL}[\pi_\theta||\pi_{ref}]\} \tag{3}$$

其中，$\varepsilon$ 和 $\beta$ 是超参数，而 $\hat{A}_{i,t}$ 是仅基于每个组内输出的相对奖励计算得出的优势值。GRPO 使用组内相对方式计算优势值，与奖励模型的比较性质相契合，因为奖励模型通常是在对同一问题的输出进行比较的数据集上进行训练的。

另外 GRPO 不是在奖励中添加 KL 惩罚项，而是通过直接在损失函数中添加训练策略和参考策略之间的 KL 散度来进行正则化，避免了 $\hat{A}_{i,t}$ 计算复杂化。


## 结果监督（Outcome Supervision）强化学习

<img src="/assets/images/grpo/illustration-2.png" width="600" alt=""/>

对于每个问题 $q$，从旧策略模型 $\pi_{\theta_{old}}$ 中采样一组输出 $\\{o_1, o_2,···, o_G\\}$。然后使用奖励模型对这些输出进行评分，得到相应的 $G$ 个奖励 $\mathbf{r} = \\{r_1, r_2,···, r_G\\}$。随后，通过减去组的平均值并除以组的标准差来对这些奖励进行归一化。结果监督提供了每个输出的最终归一化奖励，并将归一化奖励设置为输出中所有 token 的优势值，即 $\hat{A}_{i,t} = \widetilde{r}_i = \frac{r_i - \text{mean}(\mathbf{r})}{\text{std}(\mathbf{r})}$，然后通过最大化方程（3）来优化策略。

## 过程监督（Process Supervision）强化学习

结果监督仅在每个输出的末尾提供奖励，这可能不足以有效地监督复杂数学任务中的策略。根据 Wang 等人的研究，我们还探索了过程监督，它在每个推理步骤的末尾提供奖励。形式上，给定问题 $q$ 和 $G$ 个采样输出 $\\{o_1, o_2,···, o_G\\}$，使用过程奖励模型对输出的每个步骤进行评分，得到相应的奖励：$\mathbf{R} = \\{\\{r^{\text{index}(1)}_1, \cdots, r^{\text{index}(K_1)}_1\\}, \cdots, \\{r^{\text{index}(1)}_G, \cdots, r^{\text{index}(K_G)}_G\\}\\} $，其中 $\text{index}(j)$ 是第 $j$ 步的结束标记索引，$K_i$ 是第 $i$ 个输出中的总步骤数。

我们还通过平均值和标准差对这些奖励进行归一化，即：

$$\widetilde{r}_i^{\text{index}(j)} = \frac{r_{\text{index}(j)}^i - \text{mean}(\mathbf{R})}{\text{std}(\mathbf{R})}$$

随后，过程监督将每个标记的优势计算为后续步骤的归一化奖励之和，即：

$$\hat{A}_{i,t} = \sum_{\text{index}(j) \geq t} \widetilde{r}_{\text{index}(j)}^i$$

然后通过最大化方程 (3) 中定义的目标来优化策略。


## 迭代式强化学习

随着强化学习训练过程的进行，旧的奖励模型可能无法对当前的策略模型进行有效监督。因此，我们还探索了使用 GRPO 的迭代式强化学习。如算法 1 所示，在迭代式 GRPO 中，我们基于策略模型的采样结果生成新的训练集，然后使用回放机制对旧的奖励模型进行持续训练，其中包括 10% 的历史数据。然后，我们将参考模型设置为策略模型，并使用新的奖励模型对策略模型进行持续训练。


## 参考


* [人人都能看懂的RL-PPO理论知识](https://zhuanlan.zhihu.com/p/7461863937)