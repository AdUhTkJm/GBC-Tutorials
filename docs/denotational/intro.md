# 指称语义

## 引子

我们不妨来看一个简单的语法:

$$
\begin{align*}
A &::= n \mid L \mid A + A \mid \dots \\
B &::= \mathsf{true} \mid \mathsf{false} \mid A = A \mid \neg B \mid \dots \\
C &::= \mathsf{skip} \mid L := A \mid \mathsf{if}\ B\ \mathsf{then}\ C\ \mathsf{else}\ C \mid \mathsf{while}\ B\ \mathsf{do}\ C\mid C; C
\end{align*}
$$

这里的 $n$ 是任意的自然数, 而 $L$ 则来自一个"地址"的集合. 容易发现, $A$ 中的表达式就是算术表达式, 而 $B$ 则是布尔表达式, $C$ 则是语句. (我们滥用一下符号: $A$, $B$ 和 $C$ 既指具体的表达式, 也指全体符合这个条件的表达式的集合.)

对于熟悉任何一种编程语言的人来说, 它们应该具有很明显的意义. 但是, 我们是否可以将这种"感觉上的意义"严格化呢? 实际上, 我们有不少形式化地描述语义的方式, 而**指称语义** (*denotational semantics*) 则是其中的一种.

程序其实可以看做是一个状态机. 对于我们例子中的语言来说, 这些"状态"其实就是地址中存储的内容. 假设现在程序的状态是 $s\in \mathrm{State}$, 那么 $A$ 类表达式其实就是接受 $s$ 并给出一个自然数 $m\in \mathbb{N}$ 的函数. 同样地, $B$ 类表达式则是接受 $s$ 并给出一个布尔值 $b\in \{\mathsf{true}, \mathsf{false}\}$ 的函数; $C$ 类表达式则是接受 $s$ 并给出下一个状态 $s'$ 的函数.

指称语义中提到的**指称** (*denotation*), 就是从表达式本身, 到"描述表达式效用的函数"的函数. 形式化地来讲, $A$, $B$ 和 $C$ 的指称分别是三个函数 $\mathcal{A}$, $\mathcal{B}$ 与 $\mathcal{C}$:

$$
\begin{align*}
\mathcal{A} &: A \rightarrow (\mathrm{State}\rightarrow \mathbb{N}) \\
\mathcal{B} &: B \rightarrow (\mathrm{State}\rightarrow \{\mathsf{true}, \mathsf{false}\}) \\
\mathcal{C} &: C \rightarrow (\mathrm{State}\rightarrow \mathrm{State}) \\
\end{align*}
$$

例如, 对于 $A$ 中的表达式, 我们可以这样定义:

$$
% These new commands are effective across the whole document;
% However, they won't contaminate other documents on the site.
\newcommand{\ldbrack}{[\![}
\newcommand{\rdbrack}{]\!]}
\begin{align*}
\mathcal{A}\ldbrack n\rdbrack &= \lambda s. n \\
\mathcal{A}\ldbrack A_1 + A_2\rdbrack &= \lambda s. A\ldbrack A_1\rdbrack (s) + A\ldbrack A_2\rdbrack (s) \\
\mathcal{A}\ldbrack L\rdbrack &= \lambda s. s(L)
\end{align*}
$$

这里的 $\lambda s$ 意为接受一个参数 $s$ 的匿名函数. 详情可以参考 $\lambda$-演算的相关知识 (本站尚未更新).

对于 $B$ 中的表达式, 我们同样地有:

$$
\begin{align*}
\mathcal{B}\ldbrack \mathsf{true} \rdbrack &= \lambda s. \mathsf{true} \\
\mathcal{B}\ldbrack \mathsf{false} \rdbrack &= \lambda s. \mathsf{false} \\
\mathcal{B}\ldbrack A_1 = A_2\rdbrack &= \lambda s.
\begin{cases}
\mathsf{true}, &\mathcal{A}\ldbrack A_1\rdbrack(s) = \mathcal{A}\ldbrack A_2\rdbrack(s); \\
\mathsf{false}, &\text{otherwise}
\end{cases} \\
\mathcal{B}\ldbrack \neg B \rdbrack &= \lambda s.
\begin{cases}
\mathsf{true}, &\mathcal{B}\ldbrack B \rdbrack(s) = \mathsf{false}; \\
\mathsf{false}, &\text{otherwise}
\end{cases}
\end{align*}
$$

对于 $C$, 其他的都好说, 但 while 就有些难办了. 我们知道, while 可以这样执行:

$$
\langle \mathsf{while}\ B\ \mathsf{do}\ C, s \rangle \rightarrow \langle \mathsf{if}\ B\ \mathsf{then}\ (C; \mathsf{while}\ B\ \mathsf{do}\ C)\ \mathsf{else}\ \mathsf{skip}, s \rangle
$$

我们可以这样定义 if-then-else 的语义:

$$
\mathcal{C}\ldbrack \mathsf{if}\ B\ \mathsf{then}\ C_1\ \mathsf{else}\ C_2\rdbrack = \lambda s. \mathrm{ite}(\mathcal{B}\ldbrack B\rdbrack(s), \mathcal{C}\ldbrack C_1\rdbrack(s), \mathcal{C}\ldbrack C_2\rdbrack(s))
$$

其中

$$
\mathrm{ite}(b, s, s') = 
\begin{cases}
s, &b = \mathsf{true}; \\
s', &b = \mathsf{false}.
\end{cases}
$$

接下来, 我们可以将这个定义套用到上面的 while 的执行方式上, 得到:

$$
\begin{align*}
\mathcal{C}\ldbrack \mathsf{while}\ B\ \mathsf{do}\ C \rdbrack
&= \mathcal{C}\ldbrack\mathsf{if}\ B\ \mathsf{then}\ (C; \mathsf{while}\ B\ \mathsf{do}\ C)\ \mathsf{else}\ \mathsf{skip}\rdbrack \\
&= \lambda s. \mathrm{ite}(\mathcal{B}\ldbrack B\rdbrack(s), \mathcal{C}\ldbrack C; \mathsf{while}\ B\ \mathsf{do}\ C\rdbrack(s), \mathcal{C}\ldbrack \mathsf{skip}\rdbrack(s)) \\
&= \lambda s. \mathrm{ite}(\mathcal{B}\ldbrack B\rdbrack(s), \mathcal{C}\ldbrack \mathsf{while}\ B\ \mathsf{do}\ C\rdbrack(\mathcal{C}\ldbrack C\rdbrack(s)), \mathcal{C}\ldbrack \mathsf{skip}\rdbrack(s))
\end{align*}
$$

注意到等式左端和右端同时出现了 $\mathcal{C}\ldbrack \mathsf{while}\ B\ \mathsf{do}\ C\rdbrack$: 我们只知道它是如上这个方程的不动点, 却完全不知道它具体的值是什么. 这引出了不少问题: 这个方程真的有解吗? 像这样的方程什么时候会有解? 而且, 如果它有不止一个解, 我们到底取哪个作为 $\mathcal{C}\ldbrack \mathsf{while}\ B\ \mathsf{do}\ C\rdbrack$ 的值呢? 

为了解决这些问题, 我们需要引入**偏序域理论** (*domain theory*). Wikipedia 将其称作**域理论**, 但它容易和**域论** (*field theory*) 混淆, 所以我不准备采用这个翻译. 我们要研究的**偏序域** (*domain*) 不是域 (*field*).

接下来, 我们介绍一些基础的定义. 至于它们具体的应用, 请见下一节 [Kleene 定理](./kleene.md).

## 偏序

<a name="def-1"></a>
**定义 1.** 一个集合 $D$ 上的**偏序** (*partial order*) $\sqsubseteq$ 是满足如下性质的二元关系:

- 自反性: $\forall a\in D, a\sqsubseteq a$.
- 传递性: $\forall a,b,c\in D, a\sqsubseteq b, b\sqsubseteq c \Longrightarrow a\sqsubseteq c$.
- 反对称性: $\forall a,b\in D, a\sqsubseteq b \land b\sqsubseteq a\Longrightarrow a = b$.

学过离散数学的话, 这个定义应当是非常熟悉的.

我们将这对 $(D,\sqsubseteq)$ 称作一个**偏序集** (*partially ordered set*, 或者简称 *poset*).

我们可以在从 $X$ 到 $Y$ 的偏函数集合 $(X\rightharpoonup Y)$ 上定义一个偏序:

$$
f \sqsubseteq g \Longleftrightarrow \mathrm{dom}(f)\subseteq \mathrm{dom}(g)\land \forall x\in \mathrm{dom}(f), f(x) = g(x)
$$

直观上来看, $f\sqsubseteq g$ 的意思是 $g$ 比 $f$ 更"确定": 它未定义的地方比 $f$ 更少, 但已经定义的地方和 $f$ 一样.

**定义 2.** 如果偏序集之间的函数 $f: D\to E$ 满足 $\forall d, d\in D, d\sqsubseteq_D d' \Longrightarrow f(d)\sqsubseteq_E f(d')$, 那么 $f$ 是**单调**的.

这和实数上的单调函数定义是类似的.

**定义 3.** 如果 $d\in D$ 满足 $\forall s\in D, d\sqsubseteq s$, 那么 $d$ 是 $D$ 中的**最小元** (*minimum*). 我们会把 $d$ 记作 $\bot_{D}$, 或者直接写作 $\bot$.

最小元不一定存在; 正实数集在通常的"小于"下就没有最小元. 但如果最小元存在, 那么它是唯一的: 设 $d, d'$ 是最小元, 我们知道 $d\sqsubseteq d'$ 而且 $d'\sqsubseteq d$, 而根据偏序的[反对称性](#def-1), $d' = d$.

## 不动点

**定义 4.** 对于函数 $f: D\to D$, 如果 $d\in D$ 满足 $f(d) = d$, 那么 $d$ 是 $f$ 的**不动点** (*fixed point*).

这和我们高中所学到的不动点的定义是一致的.

**定义 5.** 对于函数 $f: D\to D$, 如果 $d\in D$ 满足 $f(d)\sqsubseteq D$, 那么 $d$ 是 $f$ 的**预不动点** (*pre-fixed point*). 最小的预不动点被写作 $\mathrm{fix}(f)$; 换句话说, $\forall d\in D, f(d)\sqsubseteq d \rightarrow \mathrm{fix}(f)\subseteq d$.

显然, 与 $D$ 中的最小元一样, 最小预不动点如果存在, 必然是唯一的.

**我是 [AdUhTkJm](https://github.com/AdUhTkJm). 文中如有错漏, 请在 [Issues](https://github.com/GirlsBandCompiler/Tutorials/issues) 中指出.**
