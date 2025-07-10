---
layout: post
date: 2025-7-10T13:57:39+08:00
title: 数学分析笔记一：实数
tags: 
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


## 有尽小数在实数系中处处稠密

**定理：设 $a$ 和 $b$ 是实数，$a < b$，则存在有尽小数 $c$，满足 $a < c < b$。**

证明：如果 $a < 0 < b$，则 $c = 0$ 满足要求。因此只需证明 $0 \leq a < b$ 或 $a < b \leq 0$ 的情况。这里只证明 $0 \leq a < b$。

设 $a = a_0.a_1a_2\cdots$，$b = b_0.b_1b_2\cdots$，$a < b$，则必存在 $p \in \mathbf{Z_+}$ 使得：

$$
a_0 = b_0, a_1 = b_1, ...,a_{p-} < b_{p-1}, a_p < b_p
$$

又因为 $a$ 是规范小数，因此必然存在 $q > p$ 使得 $a_q < 9$。

取 $c=a_0.a_1a_2...a_p..a_{q-1}(a_q+1)000\cdots$，$c$ 是有尽小数，且满足 $a < c < b$。

## 确界原理

**定理：$\mathbf{R}$ 的任意非空且有上界的子集合 $\mathbf{D}$ 在 $\mathbf{R}$**

**第二种陈述：$\mathbf{R}$ 的任意非空且有下界的子集合 $\mathbf{D}$ 在 $\mathbf{R}$ 中有下确界。**

证明：
情形 1：设 0 是集合 $E$ 的一个下界，因为 $E \neq \emptyset$，因此必然存在 $x \in E$ 和 $k \in \mathbf{N}$ 使得 $k >x$。

可以看出 0 是 集合 $E$ 的下界，k 不是集合 $E$ 的下界。依次考察 $0,1,\cdots,k-1$ 这些数，可以断定，存在 $a_0 \in \\{0,1,\cdots, k-1\\}$，使得 $a_0$ 是 $E$ 的下界，而 $a_0 + 1$不是 $E$ 的下界。接下来依次考察 $a_0.0, a_0.1, \cdots, a_0.9$ 这些数，又可以断定，存在 $a_1 \in \\{0,1,\cdots, k-1\\}$，使得 $a_0.a_1$ 是 $E$ 的下界，而 $a_0.a_1 + \frac{1}{10}$不是 $E$ 的下界。依次类推，我们得到一串数字：

$$
a_0, a_0.a_1, a_0.a_1a_2, \cdots, a_0.a_1a_2\cdots a_n, \cdots
$$

这些数满足 $a_0.a_1a_2\cdots a_n$ 是 $E$ 的下界，而 $a_0.a_1a_2\cdots a_n + \frac{1}{10^n}$不是 $E$ 的下界。

我们将证明，$a_0.a_1a_2\cdots a_n a_{n+1} \cdots$ 是规范小数，且正好是 $E$ 的下确界。

假设 $a_0.a_1a_2\cdots$ 不是规范小数，则必然存在 $p \in \mathbf{Z_+}$ 使得 $a_{p+1} = a_{p+2} = \cdots = 9$。设 $p$ 是满足该条件的最小的非负整数。对任意 $\beta \in E$，设 $\beta$ 的规范小数为 $\beta_0.\beta_1\beta_2\cdots$，则必然存在 $n > p$ 使得 $\beta_n < 9$（规范小数）。又因为 $\beta \geq a_0.a_1a_2\cdots a_n$，所以必然存在 $q \in \\{0, 1, \cdots, p\\}$，使得

$$
\beta_0 = a_0, \cdots, \beta_{q-1} = a_{q-1}, \beta_q \geq a_q + 1
$$

于是有

$$
\begin{aligned}
\beta &\geq a_0.a_1\cdots a_{q-1}(a_q + 1) \\
&\geq a_0.a_1\cdots a_{p-1}(a_p + 1) \\
&=a_0.a_1\cdots a_{p-1}a_p+\frac{1}{10^p}
\end{aligned}
$$

与 $a_p$ 的选择方法矛盾。因此 $a_0.a_1a_2\cdots$ 必定是规范小数。

接下来证明 $a_0.a_1a_2\cdots$ 是下确界。任何 $\gamma \in E$ 必须满足 $\gamma \geq a_0.a_1\cdots$。否则，则必定存在 $h \in \mathbf{Z}$ 使得 $\gamma < a_0.a_1\cdots a_h$。这与 $a_0.a_1a_2\cdots$ 的选取方法矛盾。其次，对于任何 $b >  a_0.a_1\cdots$，必定存在 $k \in \mathbf{Z_+}$，使得  $b \geq a_0.a_1\cdots a_k + \frac{1}{10^k}$（和 $a_0.a_1a_2\cdots$ 的选取方法矛盾，$a_0.a_1\cdots a_k + \frac{1}{10^k}$不是 $E$ 的下界）。这样 $b$ 不可能是集合的下界。

情形 2：设 0 不是集合 $E$ 的下界，即存在 $x \in E$，使得 $x < 0$。因此 $E$ 的任何下界 $l$ 必定小于 0。
考察 $\mathbf{R}$ 的另一非空子集合 $F = \\{-l | \text{l 是 E 的下界}\\}$。可以看出 0 是 $F$ 的下界，由情形 1 可得：$F$ 在 $\mathbf{R}$ 中有下确界，即存在：

$$
c = \inf F \in \mathbf{R}
$$

我们将证明 $a=-c$ 是集合 $E$ 的下确界。考察 $\gamma \in E$，显然对任何 $-l \in E$ 都有 $r \geq l, -\gamma \leq l$。这说明  $-\gamma$ 是集合 $F$ 的一个下界。因而 $-\gamma \leq c, \gamma \geq -c = a$。这说明 $a = -c$ 是集合 $E$ 的一个下界。另一方面，对于任意 $b > a$，我们有 $-b < -a = c$，因而 $-b \notin F$，这表示，任意大于 $a$ 的实数都不是集合 $E$ 的下界。

## 实数四则运算

**定理 1：设 $a$ 和 $b$ 是实数，则存在唯一实数 $u$，使对于满足条件 $\alpha \leq a \leq \alpha', \beta \leq b \leq \beta'$ 的任意有尽小数 $\alpha, \alpha', \beta, \beta'$，都有：**

$$
\alpha + \beta \leq u \leq \alpha' + \beta'
$$

定义：我们将定理 1 中确定的唯一实数 $u$ 叫做实数 $a$ 和 $b$ 之和。

**定理 2：设 $a$ 和 $b$ 是非负实数，则存在唯一实数 $v$，使对于满足条件 $0 \leq \alpha \leq a \leq \alpha', 0 \leq \beta \leq b \leq \beta'$ 的任意有尽小数 $\alpha, \alpha', \beta, \beta'$，都有：**

$$
\alpha \beta \leq v \leq \alpha' \beta'
$$

定义：我们将定理 2 中确定的唯一实数 $v$ 叫做非负实数 $a$ 和 $b$ 的乘积。

**引理 1：设 $a$ 是任意一个实数，则对任何正的有尽小数 $\epsilon$，存在有尽小数 $\alpha$ 和 $\alpha'$，满足条件 $\alpha\leq a \leq \alpha', \alpha' - \alpha < \epsilon$。**

证明：设 $\epsilon=\epsilon_0.\epsilon_1\epsilon_2\cdots \epsilon_p$，并取第一位不为 0 的数字是 $\epsilon_{k-1}, 0 \leq k - 1 \leq p$，则有 $\frac{1}{10^k} < \epsilon$

如果 $a$ 的规范小数表示为 $a_0.a_1a_2\cdots$，则取

$$
\alpha = a_0.a_1a_2\cdots a_k, \alpha' = a_0.a_1a_2\cdots a_k + 1/10^k
$$

如果 $a$ 的规范小数表示为 $-a_0.a_1a_2\cdots$，则取

$$
\alpha = -a_0.a_1a_2\cdots a_k - 1/10^k, \alpha' = a_0.a_1a_2\cdots a_k 
$$

这两种都有 $\alpha \leq a \leq \alpha', \alpha' - \alpha < \epsilon$

**引理 2：设 $c$ 和 $c'$ 是实数，如果对任何正的有尽小数 $\epsilon$，存在有尽小数 $\gamma$ 和 $\gamma'$，满足条件 $\gamma\leq c \leq c' \leq \gamma', \gamma' - \gamma < \epsilon$。那么就必有 $c=c'$**

证明：反证法，假设 $c < c'$，那么存在有尽小数 $\eta$ 和 $\eta'$ 满足 $c < \eta < \eta' < c'$。对于 $\epsilon = \eta' - \eta > 0$，任何满足条件 $\gamma \leq c < \eta < \eta' < c' \leq \gamma'$ 的有尽小数 $\gamma$ 和 $\gamma'$ 都不能使得 $\gamma'-\gamma < \epsilon = \eta' - \eta$。因此 $c'=c$。

**引理 3：设 $\epsilon$ 是正的有尽小数，$M$ 和 $N$ 是自然数，则存在正的有尽小数 $\epsilon'$ $\epsilon''$，使得 $M\epsilon' + N \epsilon'' < \epsilon$**

证明：设 $\epsilon=\epsilon_0.\epsilon_1\cdots\epsilon_p$，并设其中第一位不等于 0 的数字是 $\epsilon_{k-}, 0 \leq k-1 \leq p$，则有：

$$
1/10^k < \epsilon
$$

我们取自然数 $m$ 和 $m$，使得 $10^m \geq M, 10^n \geq N$，然后取：

$$
\epsilon'=\frac{1}{10^{m+k+1}}, \epsilon''=\frac{1}{10^{n+k+1}}
$$

可得：

$$
\begin{aligned}
M\epsilon'+N\epsilon'' &\leq 10^m\epsilon' + 10^n\epsilon'' \\
&=\frac{1}{10^{k+1}} + \frac{1}{10^{k+1}} < \frac{1}{10^k} < \epsilon
\end{aligned}
$$

**定理 1 证明：**

存在性：实数

$$
u = \sup \{\alpha+\beta | \alpha \text{ 和 } \beta \text{ 是有尽小数}，\alpha \leq a, \beta \leq b\}
$$

$u$ 是 $\alpha+\beta$ 的上确界，因此 $\alpha+\beta \leq u$。由 $\alpha \leq \alpha', \beta \leq \beta'$ 可得 $\alpha'+\beta'$ 是 $\alpha+\beta$ 的上界，因此有 $\alpha'+\beta' \geq u$。

唯一性：对于任意正的有尽小数 $\epsilon$ 和自然数 $M=N=1$，根据引理 3，存在正的有尽小数 $\epsilon'$ 和 $\epsilon''$，使得 $\epsilon'+\epsilon'' < \epsilon$。又根据引理 1，存在有尽小数 $\alpha$，$\alpha'$ 和 $\beta$，$\beta'$，分别满足：

$$
\alpha \leq a \leq \alpha', \alpha' - \alpha < \epsilon'
$$

和

$$
\beta \leq b \leq \beta', \beta' - \beta < \epsilon''
$$

可得 $(\alpha'+\beta') - (\alpha + \beta) < \epsilon$。因为 $\epsilon$ 可以取任意正的有尽小数，根据引理 2，满足条件 $\alpha + \beta \leq u \leq \alpha' + \beta'$ 的 $u$ 是唯一的。

**定理 2 证明：**

存在性：实数

$$
u = \sup \{\alpha\beta | \alpha \text{ 和 } \beta \text{ 是有尽小数}，\alpha \leq a, \beta \leq b\}
$$

符合定理要求。

唯一性：取自然数 $M$ 和 $N$，使得 $0 \leq a < M, 0 \leq b < N$。对于任意正的有尽小数 $\epsilon$，根据引理 3，存在正的有尽小数 $\epsilon'$ 和 $\epsilon''$，使得 $M\epsilon' + N\epsilon'' < \epsilon$。根据引理 1，存在有尽小数 $\alpha$，$\alpha'$ 和 $\beta$，$\beta'$，分别满足：

$$
0 \leq \alpha \leq a \leq \alpha' < M, \alpha' - \alpha < \epsilon''
$$

和

$$
0 \leq \beta \leq b \leq \beta' < N, \beta' - \beta < \epsilon'
$$

于是有 

$$
\begin{aligned}
\alpha'\beta' - \alpha\beta &= \alpha'\beta' - \alpha'\beta + \alpha'\beta - \alpha\beta \\
&=\alpha'(\beta'-\beta) + (\alpha'-\alpha)\beta \\
&< M\epsilon' + N\epsilon'' < \epsilon
\end{aligned}
$$

因为 $\epsilon$ 可以取任意正的有尽小数，根据引理 2，满足条件的 $u$ 是唯一的。

对于给定的正的有尽⼩数 $\alpha, \beta$ 和⾃然数 $n$， 通过逐位试商，可以确定⼀个有尽⼩数 $\gamma = \gamma_0\gamma_1\cdots\gamma_n$ 满足这样的条件：

$$
\gamma \cdot \alpha \leq \beta < (\gamma + 1/10^n) \cdot \alpha
$$

并约定以下符号：

$$
(\frac{\beta}{\alpha})_n = \gamma, (\frac{\beta}{\alpha})'_n = \gamma', \gamma' = \gamma + 1/10^n
$$


**定理 3：对于任何正实数 $a$，存在唯一的正实数 $w$，使得对于满足条件 $0 < \alpha \leq a \leq \alpha'$ 的任意有尽小数 $\alpha, \alpha'$ 和任意自然数 $m, n$ 都有 $(1/\alpha')_m \leq w \leq (1/\alpha)'_n$**

定义：定理 3 中唯一确定的正实数 $w$ 叫做正实数 $a$ 的倒数，记为 $1/a$。

证明：

存在性：因为

$$
\alpha' \cdot (1/\alpha')_m  \leq 1 \leq \alpha \cdot (1/\alpha)'_n \leq \alpha' \cdot (1/\alpha)'_n
$$

所以有 $(1/\alpha')_m \leq (1/\alpha)'_n, \forall m,n \in \mathbf{N}$

由此得实数

$$
w = \sup \{(\frac{1}{\alpha'})_m | \alpha' 是有尽小数，\alpha' \geq a, m \in \mathbf{N}\}
$$

符合定理要求（$(1/\alpha)'_n$ 是 $(1/\alpha')_m$ 的上界，$w$ 是上确界）。

唯一性：首先取 $\sigma=1/10^k$ 和 $M=10^l$，使得 $0 < \sigma < a < M$

其次，设 $\epsilon$ 是任意一个正的有尽小数，根据引理 3，存在正的有尽小数 $\epsilon'$ 和 $\epsilon''$，使得：

$$
\epsilon' + 10^{2(k+l)}\epsilon'' < \epsilon
$$

可得：$\sigma^2\epsilon' + M^2\epsilon'' = \sigma^2(\epsilon'+10^{2(k+1)\epsilon''}) < \sigma^2 \epsilon$

可以选取有尽小数 $\alpha, \alpha'$ 和自然数 $n$，满足以下条件：

$$
0 < \sigma <  \alpha \leq a \leq \alpha' < M \\
\alpha' - \alpha < \sigma^2\epsilon', 1/10^{n-1} < \epsilon''
$$

使得：

$$
\begin{aligned}
\sigma^2\left\{\left(\frac{1}{\alpha}\right)'_n - \left(\frac{1}{\alpha'} \right)_n \right\} &< \alpha\alpha'\left\{\left(\frac{1}{\alpha}\right)'_n - \left(\frac{1}{\alpha'} \right)_n \right\} \\
&=\alpha\alpha'\left\{\left(\left(\frac{1}{\alpha}\right)_n + \frac{1}{10^n}\right) - \left(\left(\frac{1}{\alpha'} \right)'_n - \frac{1}{10^n}\right) \right\} \\
&=\alpha'\left\{ \alpha \left(\frac{1}{\alpha}\right)_n\right\} - \alpha\left\{ \alpha' \left(\frac{1}{\alpha'}\right)'_n\right\} + 2\alpha\alpha'\frac{1}{10^n} \\
&<\alpha' - \alpha + \alpha\alpha'\frac{1}{10^{n-1}} \\
&<\sigma^2\epsilon' + M^2\epsilon'' < \sigma^2\epsilon
\end{aligned}
$$

于是有 $\left(\frac{1}{\alpha}\right)'_n - \left(\frac{1}{\alpha'}\right)_n < \epsilon$，因为 $ \epsilon$ 可以是任意正的有尽小数，因此符合定理的 $w$ 只有一个。

## 不等式

### 绝对值不等式

$$
|a_1+a_2 + \cdots + a_n| \leq |a_1| + |a_2| + \cdots + |a_n|
$$

### 伯努里（Bernoulli）不等式

设 $x \geq 0$，则由二项式定理：

$$
(1+x)^n = 1 + nx + \frac{n(n-1)}{2}x^2 + \cdots + x^n
$$

可得：

$$
(1+x)^n \geq 1 + nx
$$

### 算数平均和几何平均不等式

设 $x_1,x_2,\cdots,x_n \geq 0$，则：

$$
\frac{x_1 + x_2 + \cdots + x_n }{n} \geq \sqrt[n]{x_1x_2\cdots x_n}
$$

### 三角函数不等式

$$
\sin x < x < \tan x
$$