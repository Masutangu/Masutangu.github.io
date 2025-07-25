---
layout: post
date: 2025-7-25T13:56:23+08:00
title: 数学分析笔记三：连续函数
tags: 
  - 读书笔记
  - 数学
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


## 连续与间断

**定义 1：设函数 $f(x)$ 在 $x_0$ 点的邻域 $U(x_0, \eta)$ 上有定义。如果对任何满足条件 $x_n \to x_0$ 的序列 $\\{x_n\\} \subset U(x_0, \eta)$，都有 $\lim f(x_n) = f(x_0)$，那么我们就说函数 $f$ 在 $x_0$ 点连续，或者说 $x_0$ 点是函数 $f$ 的连续点。**

**定义 2：设函数 $f(x)$ 在 $x_0$ 点的邻域 $U(x_0, \eta)$ 上有定义。如果对任意 $\epsilon > 0$，存在 $\delta > 0$，使得只要 $\|x - x_0\| < \delta$，就有 $\|f(x) - f(x_0)\|< \epsilon$。那么我们就说函数 $f$ 在 $x_0$ 点连续，或者说 $x_0$ 点是函数 $f$ 的连续点。**

**定理 1：设函数 $f$ 在 $x_0$ 点连续，则存在 $\delta > 0$，使得函数 $f$ 在 $U(x_0,\delta)$ 上有界。**

**定理 2：设函数 $f(x)$ 和 $g(x)$ 在 $x_0$ 点连续，则 (1) $f(x)\pm g(x)$ 在 $x_0$ 处连续；（2）$f(x)\cdot g(x)$ 在 $x_0$ 处连续；（3）$\frac{f(x)}{g(x)}$ 在使得 $g(x_0)\neq 0$ 的 $x_0$ 处连续；（4）$cg(x)$ 在 $x_0$ 点连续。**

**定理 4：设函数 $f(x)$ 和 $g(x)$ 在 $x_0$ 点连续，如果 $f(x_0) < g(x_0)$，那么存在 $\delta > 0$，使得对于 $x \in U(x_0, \delta)$ 有 $f(x) < g(x)$。**

**定理 5（复合函数的连续性）：设函数 $f(x)$ 在 $x_0$ 点连续，函数 $g(y)$ 在 $y_0=f(x_0)$ 点连续，那么复合函数 $g \circ f(x) = g(f(x))$ 在 $x_0$ 点连续。**

**定义**：设函数 $f(x)$ 在 $(x_0 - \eta, x_0]$ 上有定义，如果 $\lim_{x\to x_0-}f(x) = f(x_0)$，那我们说函数 $f$ 在 $x_0$ 点左侧连续。类似的可定义右侧连续。引入记号 $f(x_0-) = \lim_{x\to x_0-}f(x), f(x_0+) = \lim_{x\to x_0+}f(x)$。

**定理 6：设函数 $f(x)$ $U(x_0, \eta)$ 上有定义，则 $f(x)$ 在 $x_0$ 点连续点充分必要条件是它在这点左侧连续且右侧连续。**

不连续存在两种情形：
* 情形 1：函数 $f(x)$ 在 $x_0$ 点的两个单侧极限 $f(x_0-)$ 和 $f(x_0+)$ 都存在，但 $f(x_0-)\neq f(x_0+)$ 或 $f(x_0-)=f(x_0+) \neq f(x_0)$
* 情形 2：函数 $f(x)$ 在 $x_0$ 点至少有一个单侧极限不存在

**定义**：函数 $f(x)$ 在 $(x_0 - \eta, x_0]$ 上有定义，在 $x_0$ 点不连续，如果是情形 1，我们就说 $x_0$ 是函数 $f$ 的第一类间断点；如果是情形 2，就说 $x_0$ 是函数 $f$ 的第二类间断点。

例子：
* 任何 $x\in \mathbf{R}$ 都是 Dirichlet 函数的第二类间断点。
* 所有的无理点都是 Riemann 函数的连续点，所有的有理点都是 Riemann 函数的第一类间断点。

## 闭区间上连续函数的重要性质

**定义**：如果函数 $f$ 在闭区间 $[a, b]$ 上有定义，在每一点 $x\in (a, b)$ 连续，在 $a$ 点右侧连续，在 $b
$ 点左侧连续，我们就说函数在闭区间 $[a, b]$ 连续。

**引理（闭区间完备）：设 $\\{x_n\\} \subset [a, b], x_n \to x_0$，则 $x_0 \in [a, b]$。**

证明：从 $a \leq x_n \leq b, n=1,2,\cdots$，可得 $a\leq \lim_{x_n} = x_0 \leq b$。如果是开区间 $a < x_n < b, n=1,2,\cdots$，不能得出 $a < \lim_{x_n} = x_0 < b$，只能得出 $a\leq \lim_{x_n} = x_0 \leq b$，极限可能等于 $a$ 或者 $b$，但 $a$ 或者 $b$ 不在区间里。

**定理 1：设函数 $f$ 在闭区间 $[a, b]$ 连续，如果 $f(a)$ 与 $f(b)$ 异号：$f(a)f(b) < 0$，那么必定存在一点 $c \in (a,b)$，使得 $f(c) = 0$。**

**Brouwer 不动点定理的特殊情形：把 $[a, b]$ 映入到 $[a, b]$ 之中的连续函数必定有不动点。**

证明：记 $g(x) = f(x) - x$，则函数 $g(x)$ 在闭区间 $[a, b]$ 连续。由条件 $a \leq f(x) \leq b, \forall x \in [a, b]$，可得 $f(a) \geq a, f(b) \leq b$，即 $g(a) \geq 0, g(b) \leq 0$。

如果 $g(a) = 0$（或则 $g(b) = 0$），那么 $c=a$（或者 $c = b$）就满足要求：$g(c) = 0, f(c) = c$。

如果 $g(a) > 0 > g(b)$，那么根据定理 1，存在 $c\in (a, b)$，使得 $g(c) = 0, f(c) = c$。

**定理 2（介值定理）：设函数 $f$ 在闭区间 $[a, b]$ 连续。如果在这闭区间的两端点的函数值 $f(a) = \alpha$ 与 $f(b) = \beta$ 不相等，那么在这两点之间的函数 $f$ 能够取得介于 $\alpha$ 与 $\beta$ 之间的任意值 $\gamma$。即如果 $f(a) < \gamma < f(b)$（或者 $f(a) > \gamma > f(b)$），那么存在 $c \in (a, b)$，使得 $f(c) = \gamma$。**

**定理 3：设函数 $f$ 在闭区间 $[a, b]$ 连续，则 $f$ 在 $[a, b]$ 上有界。**

证明：反证法。假设 $f$ 在 $[a, b]$ 上无界。考察 $[a, b]$ 的两个闭子区间 $[a, \frac{a+b}{2}]$ 和 $[\frac{a+b}{2}, b]$。$f$ 至少在其中一个闭子区间无界。我们记这闭子区间为 $[a_1, b_1]$。然后以 $[a_1, b_1]$ 代替 $[a, b]$，重复可得 $[a_2, b_2]$，函数 $f$ 在这闭子区间上无界。由此得到一串闭区间：$[a, b] \supset [a_1, b_1] \supset \cdots \supset [a_n, b_n] \supset \cdots$，满足条件：

1. $0 < b_n - a_n = \frac{b - a}{2^n}$
2. 函数 $f$ 在 $[a_n, b_n]$ 上无界


闭区间套 $\\{[a_n, b_n]\\}$ 收缩于唯一的一点：$c=\lim a_n = \lim b_n \in [a, b]$。

因为函数 $f$ 在 $c$ 点连续，所以存在 $\eta > 0$ 使得 $f$ 在 $U(c, \eta)$ 上是有界的：

$$
|f(x)| \leq K, \forall x \in U(c, \eta)
$$

又可取 $m$ 充分大，使得

$$
|a_m - c| < \eta, |b_m -c | < \eta
$$

这是就有 $[a_m, b_m] \subset U(c, \eta)$，因而有 $\|f(x)\| \leq K, \forall x \in [a_m, b_m]$。但这与闭子区间 $[a_m, b_m]$ 的选取方式矛盾。以上证明函数 $f$ 在闭区间 $[a, b]$ 上应该是有界的。

如果函数 $f$ 在开区间 $(a, b)$ 连续，那么关于 $f$ 在 $(a, b)$ 上是否有界不能得出一般性结论。

**定理 4（最大值与最小值定理）：设函数 $f$ 在闭区间 $[a, b]$ 连续，记 $M = \sup_{x \in [a,b]} f(x), m = \inf_{x \in [a,b]}f(x)$，则存在 $x', x'' \in [a,b]$，使得 $f(x') = M, f(x'') = m$。**

证明：由定理 3 可知 $-\infty < m \leq M < +\infty$。根据上确界定义可得：对任意 $n \in \mathbf{N}$，必定存在 $x_n \in [a, b]$，使得 

$$
M - \frac{1}{n} < f(x_n) \leq M
$$

从有界序列 $\\{x_n\\} \subset [a, b]$ 之中，可以选取收敛的子序列 $\\{x_{n_k}\\}$，设 $x_{n_k} \to x' \in [a, b]$。

由函数 $f$ 在 $x'$ 点的连续性可得 $f(x_{n_k}) \to f(x')$。但我们有 

$$
M - \frac{1}{n_k} < f(x_{n_k}) \leq M
$$

在上面不等式中让 $k\to +\infty$ 取极限即得 $f(x') = \lim f(x_{n_k}) = M$。

**定义**：设 $E$ 是 $\mathbf{R}$ 的一个子集，函数 $f$ 在 $E$ 上有定义。如果对任意 $\epsilon > 0$，存在 $\delta > 0$，使得只要 $x_1, x_2 \in E, \|x_1 - x_2\| < \delta$，就有 $\|f(x_1) - f(x_2)\| < \epsilon$，那我们就说函数 $f$ 在集合 E 上是**一致连续**的。

**定理 5（一致连续性定理）：如果函数 $f$ 在闭区间 $I=[a,b]$ 连续，那么它在 $I$ 上是一致连续的**（闭区间是充分条件）。

证明：用反证法。假设函数 $f$ 在闭区间 $I$ 上连续而不一致连续，那么至少存在一个 $\epsilon > 0$，使得无论 $\delta > 0$ 多小，总有 $x', x'' \in I$，满足条件：

$$
|x' - x''| < \delta, |f(x') - f(x'')| \geq \epsilon
$$

对这样的 $\epsilon$ 和 $\delta = 1/n$（$n=1, 2, \cdots$）存在 $x'_n, x_n'' \in  \mathbf{I}$，满足：

$$
|x_n' - x_n''| < 1/n, |f(x_n') - f(x_n'')| \geq \epsilon
$$

因为 $\\{x_n'\\} \subset I$ 是有界序列，它具有收敛的子序列 $\\{x_{n_k}'\\}$：$x_{n_k}' \to x_0 \in I$。

因为

$$
\begin{aligned}
|x_0 - x_{n_k}''| &\leq |x_0 - x_{n_k}'| + |x_{n_k}' - x_{n_k}''| \\
&< |x_0 - x_{n_k}'| + 1/n_k,
\end{aligned}
$$

所以又有 $x_{n_k}'' \to x_0$。又因为函数 $f$ 在 $x_0$ 点连续，所以 $\lim f(x_{n_k}') = \lim f(x_{n_k}'') = f(x_0)$。但这与 $\|f(x_{n_k}') - f(x_{n_k}'')\| \geq \epsilon$ 相矛盾。

**定理 6：设 $E$ 是 $\mathbf{R}$ 的一个子集，函数 $f$ 在 $E$ 上有定义。则 $f$ 在 $E$ 上一致连续的充分必要条件是：对任何满足条件 $\lim(x_n - y_n) = 0$ 的序列 $\\{x_n\\}\subset E$ 和 $\\{y_n\\} \subset E$，都有 $\lim(f(x_n) - f(y_n)) = 0$**

证明：
必要性：设 $f$ 在 $E$ 上一致连续，则对任何 $\epsilon > 0$，存在 $\delta > 0$，使得只要 $x, y \in E, \|x-y\| < \delta$，就有 $\|f(x) - f(y)\| < \epsilon$。

如果 $\\{x_n\\} \subset E$ 和 $\\{y_n\\} \subset E$ 满足条件：$\lim(x_n - y_n) = 0$，那么存在 $N \in \mathbf{N}$，使得 $n > N$ 时有 $\|x_n - y_n\| < \delta$。也就有 $\|f(x_n) - f(y_n)\| < \epsilon$。因此 $\lim(f(x_n) - f(y_n)) = 0$。

充分性：反证法。假设 $f$ 在 $E$ 上不一致连续，则对某个 $\epsilon > 0$，无论 $\delta = 1/n$ 取多小，总存在 $x_n, y_n \in E$，使得 $\|x_n - y_n\| < 1/n, \|f(x_n) - f(y_n)\| \geq \epsilon$。序列 $\\{x_n\\}\subset E$ 和 $\\{y_n\\} \subset E$ 满足条件 $\lim(x_n - y_n) = 0$。但序列 $\\{f(x_n) - f(y_n)\\}$ 却不能收敛于0，与条件矛盾。

## 单调函数，反函数

**引理（区间连通性）：集合 $J \in \mathbf{R}$ 是一个区间的充分必要条件为：两个实数 $\alpha, \beta \in J$，介于 $\alpha$ 和 $\beta$ 之间的任何实数 $\gamma$ 也一定属于 $J$。**

**定理 1：如果函数 $f$ 在区间 $I$ 上连续，那么 $J=f(I) = \\{f(x) \| x\in I\\}$ 也是一个区间。**定理 1 的逆命题一般来说不成立。

证明：介值定理。

**定理 2：设函数 $f$ 在区间 $I$ 上单调。则 $f$ 在 $I$ 连续的充分必要条件为：$f(I) 也是一个区间。$**

证明：

必要性：定理 1。

充分性：设 $f$ 在 $I$ 上递增并且 $f(I)$ 是一个区间。用反证法证明。假设 $f$ 在 $x_0 \in I$ 不连续，那么至少有以下两种情况之一：$f(x_0-) < f(x_0)$ 或 $f(x_0) < f(x_0+)$。对这两种情况，我们分别用 $(\lambda, \rho)$ 表示 $(f(x_0-), f(x_0))$ 或 $(f(x_0), f(x_0+))$。于是，在开区间 $(\lambda, \rho)$ 的两侧都有集合 $f(I)$ 中的点，但由于函数 $f$ 的单调性，任何 $\gamma \in (\lambda, \rho)$ 都不在集合 $f(I)$ 之中，因而 $f(I)$ 不能是一个区间（见引理：两侧的点是区间内，则两侧之间的点也在区间内）。这一矛盾说明 $f$ 必须在 $I$ 的每一点连续。

**定义**：定义函数 $g$，对任意 $y \in J$，函数值 $g(y)$ 规定为由关系 $f(x) = y$ 所决定的唯一的 $x\in I$。这样定义的函数 $g$ 称为函数 $f$ 的反函数，记为 $g=f^{-1}$。

**定理 3：设函数 $f$ 在区间 $I$ 上严格单调并连续，则其反函数 $g=f^{-1}$ 在区间 $J=f(I)$ 上严格单调且连续。**

## 指数函数与对数函数，初等函数连续性问题小结

接下来将通过极限定义正数的无理数指数方幂。

**引理 1：设 $a\in \mathbf{R}, a > 1; p, q \in \mathbf{Q}, \|p - q\| < 1$。则有 $\|a^p - a^q\| \leq a^q(a-1)\|p-q\|$。**

证明：因为有 $\|a^p - a^q\| = a^q\|a^{p-q} - 1\|$，因此只需证明 $\|a^{p-q} - 1\| \leq (a-1)\|p - q\|$。即证明 $\|a^r - 1\| \leq (a-1)\|r\|, \forall r \in \mathbf{Q}, \|r\| < 1$。

情形 1：$r = 0$。显然成立。

情形 2：$r = m/n \in (0, 1)$。利用几何平均数与算术平均数不等式可得：

$$
\begin{aligned}
a^r &= (a^m)^{1/n} = (a^m\cdot 1^{n-m})^{1/n} \\
&\leq \frac{ma+n-m}{n} = \frac{m}{n}(a-1) + 1 \\
&= (a-1)r + 1, \\
&0 < a^r - 1 \leq (a-1)r
\end{aligned}
$$

情形 3：$r = -s \in(-1, 0)$，对这种情形，我们有：

$$
\begin{aligned}
|a^r - 1| &= |a^{-s} - 1| = 1 - a^{-s} \\
&= \frac{a^s - 1}{a^s} < a^s - 1 \\
&\leq (a-1)s = (a - 1)|r|
\end{aligned}
$$


**引理 2：设 $a \in \mathbf{R}, a > 0, x \in \mathbf{R}$，则有：**

1. **如果 $\\{p_n\\} \subset \mathbf{Q}, p_n \to x$，那么 $\\{a^{p_n}\\}$ 收敛**
2. **如果 $\\{p_n\\}, \\{q_n\\} \subset \mathbf{Q}, p_n \to x, q_n \to x$，那么 $\lim a^{p_n} = \lim a^{q_n}$**

证明：

先对 $a > 1$的情形给出证明。

(1) 收敛序列 $\\{p_n\\}$ 是有界的，可设对于 $M \in \mathbf{N}$ 有 $p_n \leq M, \forall n \in \mathbf{N}$。收敛序列 $\\{p_n\\}$ 又是基本序列，对任意的 $\epsilon \in (0, 1)$，存在 $N \in \mathbf{N}$，使得 $m, n > N$ 时有 $\|p_m - p_n\| < \epsilon$。

于是，$m, n>N$ 时有：

$$
\begin{aligned}
|a^{p_m} - a^{p_n}| &\leq a^{p_n}(a-1)|p_m - p_n| \\
&\leq a^M(a-1)\epsilon
\end{aligned}
$$

因此 $\\{a^{p_n}\\}$ 是基本序列，也就证明了 $\\{a^{p_n}\\}$ 收敛。

（2）收敛序列是有界的，可设对于 $M \in \mathbf{N}$ 有 $q_n \leq M, \forall n \in \mathbf{N}$。又因为 $\lim(p_n - q_n) = 0$，可设 $n > N$ 时，$\|p_n - q_n\| < \epsilon < 1$。于是 $n > N$ 时就有：

$$
\begin{aligned}
|a^{p_n} - a^{q_n}| &\leq a^{q_n}(a-1)|p_n - q_n| \\
&\leq a^M(a-1)\epsilon
\end{aligned}
$$

可得 $\lim(a^{p_n} - a^{q_n}) = 0$。

继续考察 $0< a \leq 1$ 的情形，如果 $a=1$，那么 $\\{a^{p_N}\\}$ 和 $\\{a^{q_n}\\}$ 都是常数序列，因此结论（1）和（2）都成立。如果 $0 < a < 1$，那么 $1/a > 1$，因为 $a^{p_n} = \frac{1}{(\frac{1}{a})^{p_n}}, a^{q_n} = \frac{1}{(\frac{1}{a})^{q_n}}$，所以结论（1）和（2）也成立。

**定义**：设 $a\in\mathbf{R}, a > 0$，$x$ 是无理数，我们定义 $a^x = \lim a^{q_n}$，这里 $\\{q_n\\}$ 是收敛于 $x$ 的任意有理数序列。

**定理 2：对于 $a \in \mathbf{R}, a > 0$ 和 $x, y \in \mathbf{R}$，我们有（1）$a^{x+y}  = a^x \cdot a^y$；（2）$a>1,x<y \implies a^x < a^y, a < 1, x < y \implies a^x > a^y$**

证明：

（2）对 $a > 1$ 的情形给出证明。设 $\\{p_n\\}, \\{q_n\\}$ 是有理数序列，$p_n \to x, q_n \to y$，因为 $x < y$，所以对充分大的 $n$ 就有 $p_n < q_n$。于是 $a^{p_n} < a^{q_n}, a^x = \lim a^{p_n} \leq \lim a^{q_n} = a^y$。

为了得到严格不等式，我们在 $x$ 与 $y$ 之间插入两个有理数 $r$ 和 $s$：$x < r < s < y, r,s \in \mathbf{Q}$。于是 $a^x \leq a^r < a^s \leq a^y$。

**引理 3：设 $a\in \mathbf{R}, a > 1; x, y \in \mathbf{R}, \|x - y\| < 1$。则有 $\|a^x - a^y\| \leq a^y(a-1)\|x-y\|$。**

证明：设 $\\{p_n\\}, \\{q_n\\}$ 是有理数序列，$p_n \to x, q_n \to y$，则对充分大的 $n$ 有 $\|p_n - q_n\| < 1$。于是有 

$$
|a^{p_n} - a^{q_n}| \leq a^{q_n}(a-1)|p_n - q_n|
$$

上式中让 $n \to +\infty$ 取极限可得：$\|a^x - a^y\| \leq a^y(a-1)\|x-y\|$。

**定理 3：设 $a \in \mathbf{R}, a > 1$。则指数函数 $a^x$ 在 $\mathbf{R} = (-\infty, +\infty)$ 上有定义，严格递增且连续。**

证明：严格递增见定理 2。这里证明连续性。设 $x_0 \in \mathbf{R}, \\{x_n\\} \subset \mathbf{R}, x_n \to x_0$。则对充分大的 $n$ 有 $\|x_n - x_0\| < 1$。于是有 $\|a^{x_n} - a^{x_0}\| \leq a^{x_0}(a - 1)\|x_n - x_0\|$。由此可得函数 $a^x$ 在 $x_0$ 点连续。

**引理 4：设 $a \in \mathbf{R}, a > 1$，则有 (1) $\lim_{x\to +\infty} a^x= +\infty$ （2）$\lim_{x \to -\infty} a^x = 0$**

证明：

（1）我们有不等式 $a^n = (1 + (a-1))^n\geq 1 + n(a-1), \forall n \in \mathbf{N}$。对任何 $E > 0$，可取 $\Delta = \frac{E}{a-1} + 1$，则当 $x > \Delta$ 时，就有 

$$
\begin{aligned}
a^x &\geq a^{[x]} \geq 1 + [x](a-1) \\
&> 1 + \frac{E}{a-1}(a-1) > E
\end{aligned}
$$

（2）$\lim_{x\to-\infty}a^x = \lim_{x\to-\infty}\frac{1}{a^{-x}} = \lim_{x\to+\infty}\frac{1}{a^x} = 0$


**定义**：多项式函数，有理分式函数，三角函数，反三角函数，指数函数，对数函数等函数在他们有定义的范围内都是连续的。这些函数称为**基本初等函数**。由基本初等函数经过有限次四则运算和复合而成的函数称为**初等函数**。

**定理 5：初等函数在其有定义的范围内都是连续的。**

## 无穷小量（无穷大量）的比较，几个重要的极限

**定义 1**：设函数 $\alpha(x)$ 在 $a$ 点的某个去心邻域 $\check{U}(a)$ 上有定义，如果 $\lim_{x\to a}\alpha(x) = 0$。那么就说 $\alpha(x)$ 是 $x \to a$时的**无穷小量**。

**定义 2**：设函数 $A(x)$ 在 $a$ 点的某个去心邻域 $\check{U}(a)$ 上有定义，如果 $\lim_{x\to a}A(x) = \infty$。那么就说 $A(x)$ 是 $x \to a$时的**无穷大量**。

**定义 3**：设函数 $\phi(x)$ 和 $\psi(x)$ 在 $a$ 点的某个去心邻域 $\check{U}(a)$ 上有定义，并设在 $\check{U}(a)$ 上 $\phi(x) \neq 0$。我们分别用记号 “$O$”，“$o$” 和 “$\sim$” 表示比值 $\frac{\psi(x)}{\phi(x)}$ 在 $a$ 点邻近的几种状况：

（1）$\psi(x) = O(\phi(x))$ 表示 $\frac{\psi(x)}{\phi(x)}$ 是 $x \to a$ 时的有界变量（即 $\frac{\psi(x)}{\phi(x)}$ 在 $a$ 点的某个去心邻域上上有界的）；

（2）$\psi(x) = o(\phi(x))$ 表示 $\frac{\psi(x)}{\phi(x)}$ 是 $x \to a$ 时的无穷小量（即 $\lim_{x\to a}\frac{\psi(x)}{\phi(x)} = 0$）；

（3）$\psi(x) = \sim\phi(x)$ 表示 $\lim_{x\to a}\frac{\psi(x)}{\phi(x)} = 1$。

记号 $\psi(x) = O(1) ~ (x \to a)$ 表示 $\psi(x)$ 在 $a$ 点的某个去心邻域上有界；而记号 $\omega(x) = o(1) ~ (x \to a)$ 表示 $\lim_{x\to a} \omega(x) = 0$。

设 $\phi(x)$ 和 $\psi(x)$ 都是无穷小量（无穷大量），如果 $\psi(x)=o(\phi(x))$，那么就说 $\psi(x)$ 是比 $\phi(x)$ 更高阶的无穷小（更低阶的无穷大）。如果 $\psi(x)=\sim\phi(x)$，那么就说 $\psi(x)$ 是与 $\phi(x)$ 等价的无穷小（等价的无穷大）。

**定理 1：设 $\phi(x)$ 和 $\psi(x)$ 在 $a$ 点的某个去心邻域 $\check{U}(a)$ 上有定义，$\phi(x) \neq 0$。则有 $\psi(x)\sim \phi(x) \implies \psi(x) = \phi(x) + o(\phi(x))$。** 

**定理 2：设 $\phi(x)$ 在 $a$ 点的某一去心邻域上有定义且不等于 $0$，则有**

（1）$o(\phi(x)) = O(\phi(x))$

（2）$O(\phi(x)) + O(\phi(x)) = O(\phi(x))$

（3）$o(\phi(x)) + o(\phi(x)) = o(\phi(x))$

（4）$o(\phi(x))O(1) = o(\phi(x)), o(1)O(\phi(x)) = o(\phi(x))$

**定理 3：对于极限过程 $x \to 0$，我们有：**

（1）$\sin x = x + o(x), \tan x = x + o(x)$

（2）$\cos x = 1 - \frac{1}{2}x^2 + o(x^2)$

（3）$e^x = 1 + x + o(x)$

（4）$\ln (1+x) = x + o(x)$

（5）$(1+x)^\mu = 1 + \mu x + o(x)$

**定理 4：如果 $x \to a$ 时，$\psi(x)\sim \phi(x)$，那么有：**

（1）$\lim_{x\to a}\psi(x)f(x) = \lim_{x \to a}\phi(x)f(x)$

（2）$\lim_{x\to a}\frac{\psi(x)f(x)}{g(x)} = \lim_{x \to a}\frac{\phi(x)f(x)}{g(x)}$

（3）$\lim_{x\to a}\frac{f(x)}{\psi(x)g(x)} = \lim_{x \to a}\frac{f(x)}{\phi(x)g(x)}$
