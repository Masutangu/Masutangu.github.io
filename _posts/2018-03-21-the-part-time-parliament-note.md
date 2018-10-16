---
layout: post
date: 2018-03-21T20:36:16+08:00
title: The Part-Time Parliament 论文笔记
tags: 读书笔记
---

# 背景
Paxos 岛兼职议会类似容错式分布式系统面对的问题：**议员对应分布式系统中的进程，议员缺席对应进程挂掉。Paxos 设计的议会协议在议员经常缺席的情况下可以保证法令的一致性。**

# The Single-Decree Synod
单一法令的神会协议的演化如下：

* 首先由几个**能保证一致性和允许进展性的约束**推导出**初级协议（preliminary protocol)**
* **preliminary protocol**的约束版本得到**基本协议（basic protocol）**，其满足一致性但不保证进展性
* 进一步约束**basic protocol**得到完整的神会协议，既满足一致性又保证进展性

接下来四小节先讲解保证一致性的约束，再依次得出 preliminary protocol、basic protocol 和完整的神会协议。

## 一致性的约束条件
神会法令通过多轮带编号的**表决（ballot）**选出。每一轮 ballot 是对单一法令的投票。每轮 ballot 牧师只能选择投票（表示赞成）或不投票（表示不赞成）。每轮 ballot 都关联一个**法定人数集（quorum）**的牧师集合。只有当 quorum 中的每一位牧师都投票，这轮 ballot 才算成功。一轮 ballot B 包括以下四个元素：

* B(dec): 被投票的法令
* B(qrm): 该轮 ballot 的 quorum
* B(vot): 投票的牧师集合
* B(bal): ballot 的编号

当且仅当 B(qrm) ⊆ B(vot) 时，这轮 ballot 才算成功。**B(bal) 值的大小和进行 ballot 的顺序无关，B(bal) 比较大的 ballot 有可能在 B(bal) 比较小的 ballot 之前发生。**

Paxos 数学家在由多轮 ballot 组成的集合 𝜷 上定义了三个约束条件，如果集合中的 ballot 都满足这些条件，则可以保证一致性会：

* B1(𝜷): 𝜷 中的每一轮 ballot 都拥有唯一的编号
* B2(𝜷): 𝜷 中任意两轮 ballot 的 quorum 至少有一个公共成员
* B3(𝜷): 𝜷 中任意一轮 ballot B，如果 B(qrm) 中任何一个牧师在 𝜷 中更早的 ballot 中投过票，则 B(dec) 与所有更早的 ballots 中最后那轮 ballot 的法令相同


下图是对 B3(𝜷) 的图解。五轮 ballot 组成集合 𝜷，A、B、Γ、∆ 和 E 表示五位牧师，方框圈起的表示投票的牧师。

<img src="/assets/images/the-part-time-parliament-note/illustration-1.png" width="800"/>

* 编号 2 是最早的一轮 ballot，因此显而易见三个条件都满足（没有比他更早的 ballot了）
* 编号 5 的 ballot，四位 quorum 都没有在更早的 ballot 中投过票，因此三个条件也满足
* 编号 14 的 ballot 中，∆ 是 quorum 中唯一一位在更早的 ballot 中投过票的，因此这一轮的法令必须和编号为 2 的 ballot 一致，都为 𝜶
* 编号 27 是一轮成功的 ballot，A、Γ 和 ∆ 为该轮的 quorum 成员。Γ 在编号 5 的 ballot 中投过票，∆ 在编号为 2 的 ballot 中投过票，根据 B3(𝜷)，这一轮的法令必须和编号 5 的法令一致（编号 5 比编号 2 更新）
* 编号 29 的 quorum 成员为 B、Γ 和 ∆。B 在编号 14 的 ballot 投过票，Γ 在编号 5 和 27 的 ballots 都投过票，∆ 在编号为 2 和 27 的ballots 也投过票，这些 ballots 中最新的编号为 27，因此这轮 ballot 的法令必须和编号 27 的法令一致

下面给出 B1(𝜷) B2(𝜷) B3(𝜷) 的数学定义：

符号 v 表示一次投票，其中 v(pst) 表示投票的牧师，v(bal) 表示投票的编号，v(dec) 表示投票的法令。集合 Votes(𝜷) 表示满足以下条件的投票 v 的集合：v(pst) ∈ B(vot)，v(bal) = B(bal)，v(dec) = B(dec)，B ∈ 𝜷。定义 p 为牧师，b 为 ballot 编号，则 MaxVote(b, p, 𝜷) 表示集合 **{ v ∈ Votes(B): (v(pst) = p) ∧ (v(bal) < b) } ∪ { null(p) }** 中 ballot 最大的投票。*注：null(p) 表示 null 投票，即 v(bal) 为-∞, v(dec) 为 BLANK 且 v(pst) = p*

对任意非空的牧师集合 Q，MaxVote(b, Q, 𝜷) 定义为 Q 中的所有牧师 p 的 MaxVote(b, p, 𝜷) 的最大值。*注：翻译得不太妥当，原文：For any nonempty set Q of priests, MaxVote(b, Q, B) was defined to equal the maximum of all votes MaxVote(b, p, B) with p in Q.*

B1(𝜷) B2(𝜷) B3(𝜷) 的数学定义如下：

* B1(𝜷) ≜ ∀B, B' ∈ 𝜷 : (B ≠ B') ⇒ (B(bal) ≠ B'(bal))
* B2(𝜷) ≜ ∀B, B' ∈ 𝜷 : B(qrm) ∩ B'(qrm) ≠ ∅
* B3(𝜷) ≜ ∀B ∈ 𝜷 : (MaxVote(B(bal) , B(qrm), 𝜷)bal ≠ −∞) ⇒ (B(dec) = MaxVote(B(bal) , B(qrm) , 𝜷)dec)

**引理：如果满足 B1(𝜷) B2(𝜷) B3(𝜷)，则 ((B(qrm) ⊆ B(vot)) ∧ (B'(bal) > B(bal))) ⇒ (B(dec) = B'(dec))**

引理证明如下：

对 𝜷 中任意 ballot B，定义 Ψ(B, 𝜷) 表示 𝜷 中所有编号大于 B(bal) 且法令不等于 B(dec) 的 ballot 的集合：**Ψ(B, 𝜷) ≜ {B' ∈ 𝜷 : (B'(bal) > B(bal) ∧ (B'(dec) ≠ B(dec))}**。如果证明 B(qrm) ⊆ B(vot) 则 Ψ(B, 𝜷) 为空，引理即成立。下面是反证法，假设存在 B 满足 B(qrm) ⊆ B(vot) 且 Ψ(B, 𝜷) 不为空，下面推导得出矛盾的结论：

1. 选择 C ∈ Ψ(B, B) 满足 C(bal) = min{ B'(bal) : B' ∈ Ψ(B, 𝜷)}
2. C(bal) > B(bal)：由 1 和 Ψ(B, 𝜷) 的定义得出
3. B(vot) ∩ C(qrm) ≠ ∅：由 B2(𝜷) 和 B(qrm) ⊆ B(vot) 得出
4. MaxVote(C(bal), C(qrm), 𝜷)bal ≥ B(bal)：由 2、3 和 MaxVote 的定义得出
5. MaxVote(C(bal), C(qrm), 𝜷) ∈ Votes(B)：由 4 知道 MaxVote(C(bal), C(qrm) , 𝜷) 不是 null 投票
6. MaxVote(C(bal), C(qrm), 𝜷)dec = C(dec)：由 5 和 B3(𝜷) 得出
7. MaxVote(C(bal), C(qrm), 𝜷)dec ≠ B(dec)：由 6、1 和 Ψ(B, 𝜷) 的定义得出
8. MaxVote(C(bal), C(qrm), 𝜷)bal > B(bal)：由 4 7 和 B1(𝜷) 的出
9. MaxVote(C(bal), C(qrm), 𝜷) ∈ Votes(Ψ(B, 𝜷))：由 7 8 和 Ψ(B, 𝜷) 的定义得出
10. MaxVote(C(bal), C(qrm), 𝜷)bal < C(bal)：由 MaxVote(C(bal), C(qrm), 𝜷) 的定义得出
11. 由 9 10 1 得出矛盾，因为 1 中定义了 C(bal) 是 Ψ(B, 𝜷) 中编号最小的，由 9 知道 MaxVote(C(bal), C(qrm), 𝜷) 属于 Votes(Ψ(B, 𝜷))，那么 MaxVote(C(bal), C(qrm), 𝜷) 必须大于等于 C(bal)，这与 10 得到的 MaxVote(C(bal), C(qrm), 𝜷)bal < C(bal) 矛盾

由引理得出，如果满足 B1(𝜷) B2(𝜷) B3(𝜷)，则任意两轮成功的 ballots 都是相同的法令，数学表示：**((B(qrm) ⊆ B(vot)) ∧ (B'(qrm) ⊆ B'(vot))) ⇒ (B'dec = Bdec)**

## The Preliminary Protocol
为了遵守 B3(𝜷)，牧师在发起表决前需要先确定 MaxVote(b, Q, 𝜷)dec，因此需要确定 Q 中的每一个 p 的 MaxVote(b, q, 𝜷)dec。Preliminary protocol 的前两步如下：

(1) 牧师 p 选择新的 ballot number b，发送一条 NextBallot(b) 消息给某些牧师

(2) 牧师 q 收到 NextBallot(b) 消息后，发送 LastVote(b, v) 消息给 p，v 为 MaxVote(b, q, 𝜷)

发送完 LastVote(b, v)，**q 承诺不再给编号在 [v(bal), b] 区间的表决投票。**

*注：这个承诺是为了解决乱序的问题， ballot 编号大的投票发生在编号小的之前时，如下图，B 和 C 投票的法令出现了不一致［参考自[Paxos理论介绍\(1\): 朴素Paxos算法理论推导与证明](https://zhuanlan.zhihu.com/p/21438357)］：*

<img src="/assets/images/the-part-time-parliament-note/illustration-2.png" width="800"/>

接下来两个步骤是：

(3) 在收到 majority 的 LastVote(b, v) 之后，p 发起一轮编号为 b，quorum 为 Q 且法令为 d（d 需要满足B3(𝜷)）的表决，给 Q 中每一个牧师发送 BeginBallot(b, d) 消息

(4) 在收到 BeginBallot(b, d) 消息后，q 决定是否投票，如果可以投票（不违背上面的承诺），则发送 Voted(b, q) 消息给 p

步骤 3 将表决 B 加入到集合 𝜷 中，其中 B(bal) = b，B(qrm) = Q，B(vot) = ∅。步骤 4 中，如果牧师 q 投票了，则将 q 加入 B(vot) 中。

协议的剩余步骤如下：

(5) 如果 p 收到了 Q 中每一个 q 的 Voted(b, q) 消息，那么 p 在律簿上记录下法令 d，并且发送 Success(d) 给每一个牧师

(6) 牧师 q 收到 Success(d) 消息后，将法令 d 记录在自己的律簿上

## The Basic Protocol 
在 preliminary protocol 中，每一个牧师都必须记录 **(i) 他发起的每一个ballot number (ii) 他投过的每一次票 (iii) 他发送过的每一个 LastVote 消息**。要记录这么多信息是非常困难的，因此 Paxos 人对 preliminary protocol 做了进一步约束，得到更实用的 basic protocol，每个牧师只需要在律簿上记录下面三个信息：

* **lastTried[p]**：p 发起的最后一个表决的编号
* **prevVote[p]**：p 投过的所有表决中编号最大的那次投票
* **nextBal[p]**：p 发过的所有 LastVote(b, v) 消息中 b 的最大值

Preliminary protocol 允许牧师并行管理任意数量的表决，basic protocol 中牧师在一个时间内只管理一个表决，编号为 lastTried[p]，忽略和之前发起的表决相关的消息。Basic protocol 中对 LastVote 做了更强的承诺：**不再对编号小于 b 的表决进行投票。**

Basic protocol 步骤如下：

(1) 牧师 p 选择新的 ballot number b，必须大于 lastTried[p]，并更新 lastTried[p] 置为 b，然后发送 NextBallot(b) 消息给某些牧师

(2) 牧师 q 收到 NextBallot(b) 消息，如果 b 大于 nextBal(q)，则更新 nextBal(q) 置为 b 并发送 LastVote(b, v) 消息给 p，v 即 prevVote[q]。如果 b 小于等于 nextBal(q) 则忽略该 NextBallot 消息。

(3) 在收到 majority 的 LastVote(b, v) 消息后，如果 b 等于 lastTried[p]，则 p 发起一轮新的表决，指定编号为 b，quorum 为 Q，法令为 d，d 需要满足 B3(𝜷)。p 发送 BeginBallot(b, d) 消息给 Q 中所有牧师。

(4) 在收到 BeginBallot(b, d) 消息后，如果 b 等于 nextBal[q]，牧师 q 投票，并将 prevVote[q] 设置为本次投票，然后发送 Voted(b, q) 消息给 p。如果 b 不等于 nextBal[q]，则忽略该 BeginBallot 消息

(5) 如果 p 收到 Q 中所有牧师的 Voted(b, q) 消息，并且 b 等于 lastTried[p]，他在律簿上记录下法令 d，并发送 Success(d) 消息给所有的牧师。

(6) 在收到 Success(d) 消息后，牧师在律簿上记录下法令 d

Basic protocol 是 preliminary protocol 的约束版本，preliminary protocol 满足一致性的条件，那么 basic protocol 也一定满足。不过和 preliminary protocol 一样，basic protocol 也没要求必须执行某个操作，因此同样没有解决进展性的问题。

## The Complete Synod Protocol

为了保证进展性，**关键在于决定牧师什么时候应该发起表决**。永远不发起表决和太频繁的发起表决都会影响进展性。完整的神会协议在 basic protocol 的基础上新增一个流程来选择唯一的牧师--总统，来发起表决。这部分细节参照论文的 2.4 节。

# The Multi-Decree Parliament

## The Protocol
Paxos 议会需要通过一系列法令而不仅仅单一一个法令。法令提议给总统，由总统赋予 ballot number 并尝试通过它。协议对每个法令使用不同实例的 Complete Synod Protocol，但这系列实例只需要一位总统来负责，并且神会协议的前两步只需要执行一次。

在神会协议中，总统在第三步之前不会选择法令和 quorum，因此总统可以为所有实例发送一条 NextBallot(b) 消息，议员回复一条 LastVote 消息，规则和单一法令的协议相同，只是把所有待表决的实例信息包含在一条 LastVote 信息里。

当总统收到 majority 的回复后，就为每个待表决的实例执行 Complete Synod Protocol 的第三步。

因此 Paxon 的议会协议如下：

* 已经知道结果的表决不再需要再走一遍 Complete Synod Protocol 的流程，因此当新选举的总统 p 在律簿上已经记录有编号小于等于 n 的法令，那他将发送 NextBallot(b, n) 代替之前的 NextBallot(b) 消息。议员收到 NextBallot 消息后，将他律簿上所有编号大于 n 的法令返回给 p，并且告诉 p 他缺失的编号小于等于 n 的法令。

## Properties of the Protocol
**在总统提议任何法令之前，必须先向 majority 学习他们已经投过票的法令。**任何一个已经通过的法令一定被至少一个多数集合中的议员投过票。这样就保证了总统在提议新法令前律簿上不会有空缺。
