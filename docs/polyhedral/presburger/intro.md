# Presburger 算术

Presburger 算术是弱化版的整数四则运算. 它支持变量和常量的加减法, 变量乘以常数, 但不支持两个变量的乘除.

在编译器中, 我们经常会遇到**仿射变换** (*affine transformation*). 对于一些整数 $x_1, \dots, x_n$ 来说, 这是指函数

$$
f(x_1, \dots, x_n) = a_1x_1 + \dots + a_nx_n + c
$$

容易发现, 这样的运算是可以在 presburger 算术中表示的. 因此, 求解关于 presburger 算术的问题, 可以方便我们进行程序分析.

在此, 我们介绍 **FPL** (*Fast Presburger Library*) 的内部结构. 它是 MLIR 内部的 presburger 算术求解库. 这篇文章基于 Arjun 等人对 FPL 的[介绍](https://dl.acm.org/doi/pdf/10.1145/3485539), 同时补充了一些其中未详细展开的算法.

## 基本集

在 FPL 中, **基本集** (*basic set*) 是一些仿射约束的交. 举个例子:

$$
\{(x, y)\mid 2x+y\leq 3 \land x = 2y\}
$$

这就是一个基本集, 它有两个约束: $2x+y\leq 3$ 以及 $x = 2y$. 若无说明, 这里的所有变量都是整数.

有时, 基本集中除了常数, 还会有一些参数, 例如:

$$
\{x\mid p\leq x \land x\leq q\}
$$

这里的参数 $p$, $q$ 被称为**符号变量** (*symbolic variable*), 而 $x$ 则称为**普通变量** (*ordinary variable*). 不论是符号变量还是普通变量, 都必须能用 presburger 算术表示, 例如 $\{x \mid x \leq p^2\}$ 就不是一个基本集, 因为 presburger 算术不允许变量间的乘法.

基本集中的约束支持"存在"量词 $\exists$. 例如, 这是合法的基本集:

$$
\{x\mid \exists y, 6y\leq x < 6(y+1)\}
$$

容易发现, 在这里 $y = x/6$ (默认向下取整), 所以我们可以用这种特殊的方式在 presburger 算术中表示除法. 既然有了除法, 取余运算也就手到擒来了. 在 FPL 中, 为除法而引入的"存在"变量可以进行特殊的优化, 所以它将这些变量和约束中本来就具有的"存在"量词区分开来, 前者称为**除法变量** (*division variable*), 后者则称为**存在变量** (*existential variable*).

将数个基本集并起来, 就可以得到一个 **presburger 集**. 例如, 一个 presburger 集可能长这样:

$$
\{x\mid \exists y, x = 2y + 1\} \cup \{x\mid \exists y, x = 3y - 2\}
$$

FPL 能够求解的内容包括:

- 计算两个 presburger 集合的交, 并, 补与差;
- 判断 presburger 集合是否为空;
- 简化 presburger 集合 (用更少的约束描述同样的集合).

虽然 FPL 中未实现, 但 [Barvinok 算法](../maths/barvinok.md)可以计算 presburger 集合中元素的个数.

对于比较复杂的运算, 请移步本站的其他文章. 在这篇文章里, 我们只介绍最简单的两种运算: 并和交, 来熟悉一下 presburger 集合.

## 简单的运算: 并和交

并集是十分显然的. Presburger 集合本身就是许多基本集的并, 我们只需要把它们合在一起就好了.

至于交集, 我们可以运用分配律. 假设我们要计算 $S\cap T$, 其中 $S$ 是一系列基本集 $S_i$ 的并, 而 $T$ 则是基本集 $T_j$ 的并. 那么:

$$
\begin{align*}
S \cap T
&= \left(\bigcup_{i} S_i\right) \cap \left(\bigcup_{j} T_j\right) \\
&= \bigcup_{i,j} S_i\cap T_j
\end{align*}
$$

那么, 我们只需要处理基本集的交就可以了. 基本集本身就是一些约束的交, 可以直接复制——不过在这之前, 我们还需要考虑到"存在"变量. 假如 $S_i$ 有 $a$ 个存在变量 $x_m$, 而 $T_j$ 有 $b$ 个变量 $y_m$, 那么它们的交就会有 $a+b$ 个存在变量, 而且在 $S_i$ 原本的约束中 $y_m$ 的系数都为 0, $T_j$ 中 $x_m$ 的系数都为零.

这么说很抽象, 不过这个操作本身是十分直观的. 举个例子:

$$
\begin{align*}
&\{x\mid \exists y, x = 2y + 1\} \cap \{x\mid \exists y, x = 3y - 2\} \\
= &\{x\mid \exists y_0, x = 2y_0 + 1\} \cap \{x\mid \exists y_1, x = 3y_1 - 2\} \\
= &\{x\mid \exists y_0, y_1, x = 2y_0 + 0y_1 + 1 \land x = 0y_0 + 3y_1 - 2 \}
\end{align*}
$$

这就是在 FPL 内部存储约束条件的矩阵内, 加入了一些为零的行而已.

**我是 [AdUhTkJm](https://github.com/AdUhTkJm). 文中如有错漏, 请在 [Issues](https://github.com/GirlsBandCompiler/Tutorials/issues) 中指出.**
