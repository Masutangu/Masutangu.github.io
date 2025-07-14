---
layout: post
date: 2025-7-14T13:20:20+08:00
title: 数学分析笔记二：极限
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

## 有界序列与无穷小序列

**定义**：设 $\\{x_n\\}$ 是⼀个实数序列。 如果对任意实数：$\epsilon ＞ 0$，都存在⾃然数 $N$，使得只要 $n ＞ N$ ， 就有 $\|x_n\| < \epsilon$，那么我们就称 $\\{x_n\\}$ 为⽆穷⼩序列。

**引理：设 $\{\alpha_n\}$ 和 $\{\beta_n\}$ 是实数序列，并设存在 $N_0 \in \mathbf{N}$，使得 $\|\alpha_n\| \leq \beta_n, \forall n > N_0$。如果 $\{\beta_n\}$ 是无穷小序列，那么 $\{\alpha_n\}$ 也是无穷小序列。**

**引理：如果 $\{\alpha_n\}$ 是无穷小序列，那么它也是有界序列。**

**定理 1：**
1. **两个有界序列的和与乘积都是有界序列**
2. **两个无穷小序列的和也是无穷小序列**
3. **无穷小序列和有界序列的乘积是无穷小序列**
4. **$\{\alpha_n\}$ 是无穷小序列 $\iff$ $\{\|\alpha_n\|\}$ 是无穷小序列**

**推论：**

1. **两个无穷小序列的乘积也是无穷小序列**
2. **实数与无穷小序列的乘积也是无穷小序列**
3. **有限个无穷小序列之和仍是无穷小序列，有限个无穷小序列的乘积也是无穷小序列**


## 收敛序列

**定义**：设 $\\{x_n\\}$ 是实数序列，$a$ 是实数，如果对任意实数 $\epsilon > 0$ 都存在自然数 $N$，使得只要 $n > N$，就有 $\|x_n-a\| < \epsilon$，那么我们说序列 $\\{x_n\\}$ 收敛，以 $a$ 为极限，记为 $\lim x_n=a$ 或者 $x_n \to a$

**定理 1：如果序列 $\\{x_n\\}$ 有极限，那么它的极限是唯一的。**

**定理 2（夹挤原理）：设 $\\{x_n\\}, \\{y_n\\}$ 和 $\\{z_n\\}$ 都是实数序列，满足条件 $x_n \leq y_n \leq z_n, \forall n \in \mathbf{N}$。如果 $\lim x_n = \lim z_n = a$，那么 $\\{y_n\\}$ 也是收敛序列，并且 $\lim y_n = a$。**

**定理 3：设 $\\{x_n\\}$ 是实数序列，$a$ 是实数，则以下三陈述等价：**

1. **序列 $\\{x_n\\}$ 以 $a$ 为极限&**
2. **$\\{x_n - a\\}$ 是无穷小序列**
3. **存在无穷小序列 $\\{a_n\\}$ 使得 $x_n = a + a_n, n=1,2,\cdots$**

**定理 5：**
1. **设 $\lim x_n=a$，则 $\lim \|x_n\| = \|a\|$**
2. **设 $\lim x_n=a, \lim y_n=b$，则 $\lim(x_n \pm y_n) = a \pm b$**
3. **设 $\lim x_n=a, \lim y_n=b$，则 $\lim(x_ny_n) = ab$**
4. **设 $x_n \neq 0 ~(n=1,2,\cdots), \lim x_n = a \neq 0$，则 $\lim \frac{1}{x_n} = \frac{1}{a}$**

**推论：**
1. **设 $\lim x_n = a, c \in \mathbf{R}$，则 $\lim (cx_n) = ca$**
2. **设 $x_n \neq 0 ~(n=1,2,\cdots), \lim x_n = a \neq 0，\lim y_n = b$，则 $\lim \frac{y_n}{x_n} = \frac{b}{a}$**

**定理 6：如果 $\lim x_n ＜ \lim y_n$， 那么存在 $N \in \mathbf{N}$， 使得 $n ＞ N$ 有 $x_n < y_n$。**

**定理 7：如果 $\\{x_n\\}, \\{y_n\\}$ 都是收敛序列，并且满足条件 $x_n \leq y_n, \forall n > N_0$，那么 $\lim x_n \leq \lim y_n$。**

## 收敛原理

**定理 1：递增序列 $\\{x_n\\}$ 收敛的充分必要条件是它有上界。**

**推论：递减序列 $\\{y_n\\}$ 收敛的充分必要条件是它有下界。**

注：因一个序列的收敛性和极限值都只与序列的尾部有关，因此定理 1 和推论中的单调条件可以削弱为“从某一项之后单调“。

证明：

必要性：收敛序列是有界的。

充分性：设序列 $\\{x_n\\}$ 有上界，则存在上确界 $a=\sup\\{x_n\\}$。对任意 $\epsilon > 0$，显然 $a - \epsilon < a$，存在 $x_N$ 使得 $a-\epsilon < x_N \leq a$。当 $n> N$ 时，有 $a-\epsilon < x_N \leq x_n \leq a$。这就证明了 $\lim x_n = a = \sup\\{x_n\\}$。

**定理 2（闭区间套原理）：如果实数序列 $\\{a_n\\}$ 和 $\\{b_n\\}$ 满足条件：（1）$a_{n-1} \leq a_n \leq b_n \leq b_{n-1}, \forall n > 1$ （2）$\lim (b_n - a_n) = 0$，那么（i）序列 $\\{a_n\\}$ 和 $\\{b_n\\}$ 收敛于相同的极限值：$\lim a_n = \lim b_n = c$，（ii）$c$ 是满足以下条件的唯一实数：$a_n \leq c \leq b_n, \forall n \in \mathbf{N}$**

证明：

（i）由条件（1）可得：$a_1 \leq a_{n-1} \leq a_n \leq b_n \leq b_{n-1} \leq b_1$，序列 $\\{a_n\\}$ 递增且有上界 $b_1$，序列 $\\{b_n\\}$ 递减且有下界 $a_1$，因此 $\\{a_n\\}$ 和 $\\{b_n\\}$ 都是收敛序列。由条件（2）可得：$\lim a_n - \lim b_n = \lim (b_n - a_n) = 0$，这证明了序列 $\\{a_n\\}$ 和 $\\{b_n\\}$ 的极限相等：$\lim a_n = \lim b_n = c$。

（ii）因为 $c = \sup\\{a_n\\} = \inf \\{b_n\\}$，所以显然有 $a_n\leq c_n \leq b_n, \forall n \in \mathbf{N}$。如果实数 $c'$ 也满足条件 $a_n \leq c' \leq b_n, \forall n \in \mathbf{N}$。那么上式中让 $n\to +\infty$ 取极限就得到 $c=\lim a_n \leq c' \leq \lim b_n = c$（夹挤原理）。这证明了实数 $c$ 是唯一的。


注：闭区间套原理的各条件对保证结论成立非常重要。
（1）如果一列闭区间不是一个套在另一个之中，那么这列闭区间就可能不包含公共点。
（2）如果一列闭区间一个套在另一个之中，但这列闭区间的长度不收缩于0，那么属于这列闭区间的公共点就不止一个。
（3）如果把闭区间套换成了“开区间套”（仍然要求长度收缩于0），那么仍存在 $c=\lim a_n = \lim b_n$，但 $c$ 可以不属于各开区间 $(a_n, b_n)$。

**定理 3：设序列 $\\{x_n\\}$ 收敛于 $a$，则它的任何⼦序列 $\\{x_{n_k}\\}$ 也都收敛于同⼀极限 $a$。**

**定理 4（波尔查诺-维尔斯特拉斯（Bolzano-Weierstrass）定理）：任意有界序列 $\\{x_n\\}$ 都具有收敛的子序列。** 

证明：序列 $\\{x_n\\}$ 有界，因而可设 $a \leq x_n \leq b, \forall n \in \mathbf{N}$。用中点 $\frac{a+b}{2}$ 把闭区间 $[a, b]$ 分为两个闭子区间：$[a, \frac{a+b}{2}]$ 和 $[\frac{a+b}{2}, b]$。这两个闭子区间至少有一个含有序列 $\\{x_n\\}$ 的无穷多项，我们把这一闭子区间记为 $[a_1, b_1]$，再将其对分成两个闭子区间 $[a_1, \frac{a_1+b_1}{2}]$ 和 $[\frac{a_1+b_1}{2}, b_1]$。在这两个闭子区间中，又至少有一个闭子区间含有序列 $\\{x_n\\}$ 的无穷多项，记为 $[a_2, b_2]$。用上述方式，我们取得一串闭区间：

$$
[a_1, b_1] \supset [a_2, b_2] \supset \cdots \supset [a_k, b_k] \supset \cdots
$$

其中第 $k$ 个闭区间 $[a_k, b_k]$ 的长度为 $b_k - a_k=\frac{b-1}{2^k}$。根据闭区间套原理，可以断定存在一个实数 $c$ 满足 $c \in [a_k, b_k], \forall k \in \mathbf{N}$。

接下来来证明序列 $\\{x_n\\}$ 有⼀个⼦序列收敛于 $c$。⾸先，因为 $\\{x_n\\}$ 有⽆穷多项在 $[a_1, b_1]$ 之中， 我们可以选取其中某⼀项，把它记为 $x_{n_1}$。然后， 因为 $\\{x_n\\}$ 有⽆穷多项在 $[a_2, b_2]$ 之中，可以选取其中在 $\\{x_n\\}$ 之后的某⼀项，把它记为 $\\{x_{n_2}\\}$ 。 继续这样下去得到 $\\{x_n\\}$ 的⼀个⼦序列 $\\{x_{n_k}\\}$，满⾜ $x_{n_k} \in [a_k, b_k], \forall k \in N$。因为 $\|x_{n_k} - c\| \leq b_k - a_k = \frac{b-a}{2^k}, \forall n \in \mathbf{N}$，所以 $\lim x_{n_k} = c$。


**定义**：如果序列 $\\{x_n\\}$ 满足条件：对任意 $\epsilon > 0$，存在 $N \in \mathbf{N}$，使得当 $m, n > N$ 时，就有 $\|x_m - x_n\| < \epsilon$，那么就称这序列为**基本序列**（或者**珂西序列**）。


**引理：基本序列 $\\{x_n\\}$ 是有界的。**

证明： 对于 $\epsilon > 1$，存在 $N \in \mathbf{N}$，使得当 $m, n > N$ 时，就有 $\|x_m - x_n\| < 1$。于是，对于 $n > N$，我们有 $\|x_n\| \leq \|x_n - x_{N+1}\| + \|x_{N+1}\| < 1 + \|x_{N+1}\|$。

记 $K=\max\\{\|x_1\|, \|x_2\|, \cdots, \|x_N\|, 1+\|x_{N+1}\|\\}$，则有

$$
|x_n| \leq K, \forall n \in \mathbf{N}
$$



**定理 5（收敛原理）：序列 $\\{x_n\\}$ 收敛的必要充分条件是：对任意 $\epsilon > 0$，存在 $N \in \mathbf{N}$，使得当 $m, n > N$ 时，就有 $\|x_m - x_n\| < \epsilon$。即 序列 $\\{x_n\\}$ 收敛 $\iff$ 序列 $\\{x_n\\}$ 是基本序列。**

证明：

必要性：如果序列 $\\{x_n\\}$ 收敛于 $a$， 那么这序列中序号充分⼤的两项 $x_m$ 和 $x_n$ 都接近于 $a$，对任意 $\epsilon ＞ 0$， 存在 $N \in \mathbf{N}$， 使得当 $m, n ＞ N$ 时，有

$$
|x_m - a| < \epsilon / 2, |x_n - a| < \epsilon / 2
$$

此时有

$$
\begin{aligned}
|x_m - x_n| &= |(x_m - a) - (x_n - a)| \\
&\leq |x_m-a| + |x_n + a| \\
&< \epsilon / 2 + \epsilon / 2 = \epsilon
\end{aligned}
$$

充分性：因为基本序列是有界的，引用波尔查诺-维尔斯特拉斯定理，可以断定存在序列 $\\{x_n\\}$ 的收敛子序列 $\\{x_{n_k}\\}$，设 $x_{n_k} \to a~(k \to +\infty)$。

由基本序列定义可得，对任意 $\epsilon > 0$，存在 $N \in \mathbf{N}$，使得当 $m, n > N$ 时，就有 $\|x_m - x_n\| < \epsilon/2$。又，存在 $N_1 \in \mathbf{N}$，使得 $k > N_1$ 时有 $\|a - x_{n_k}\| < \epsilon / 2$。

取 $k > \max\\{N, N_1\\}$，对任意 $n > N$ 有 $\|a-x_n\| \leq \|a - x_{n_k}\| + \|x_{n_k} - x_n\| < \epsilon / 2 + \epsilon = \epsilon$。这就证明了 $\lim x_n = a$。

## 无穷大

**定义**：

1. 设 $\\{x_n\\}$ 是实数序列，如果对任意正实数 $E$，存在自然数 $N$，使得当 $n > N$ 时，就有 $x_n > E$，那么我们就说序列 $\\{x_n\\}$ 发散于 $+\infty$，记为 $\lim x_n = + \infty$
2. 设 $\\{y_n\\}$ 是实数序列，如果对任意正实数 $E$，存在自然数 $N$，使得当 $n > N$ 时，就有 $y_n < -E$，那么我们就说序列 $\\{y_n\\}$ 发散于 $-\infty$，记为 $\lim y_n = - \infty$
3. 设 $\\{z_n\\}$ 是实数序列，如果序列 $\\{\|z_n\|\\}$ 发散于 $+\infty$，即 $\lim \|z_n\| = + \infty$，那么我们就称 $\\{z_n\\}$ 为**无穷大序列**，记为 $\lim z_n = \infty$

**定理 1：单调序列必具有（有穷或无穷的）极限。**

**定理 3：如果 $\lim x_n = + \infty$（或 $-\infty$，或 $\infty$），那么对于 $\\{x_n\\}$ 的任意子序列 $\\{x_{n_k}\\}$，也有 $\lim x_{n_k} = + \infty$（或 $-\infty$，或 $\infty$）。**

**定理 5：实数序列最多只有一个极限。**