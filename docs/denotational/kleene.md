# Kleene 定理

在[上一篇文章](./intro.md)中, 我们介绍了 $\mathcal{C}[\![\mathsf{while}\ B\ \mathsf{do}\ C]\!]$ 会产生的不动点方程, 并定义了偏序集和不动点.

我们会继续定义一些概念, 并最终证明预不动点存在的充分条件.

## 最小上界

**定义 1.** 偏序集 $D$ 中, 一个数列 $\{d_i\}_{i\in \mathbb{N}}$ 如果满足

$$
d_0\sqsubseteq d_1\sqsubseteq d_2\sqsubseteq \dots
$$

那么它就称为 $D$ 的一条**链** (*chain*).

在讨论指称语义的时候, 我们只考虑可数无穷个元素构成的数列.

**定义 2.** 给定链 $\{d_i\}\subseteq D$, 如果存在 $d\in D$ 使得

$$
\forall i\in \mathbb{N}, d_i\sqsubseteq d
$$

那么 $d$ 是 $\{d_i\}$ 的**上界** (*upper bound*). 这条链最小的上界称作**最小上界** (*least upper bound*, LUB), 我们有几种符号来记录它:

$$
\bigsqcup_{n=0}^{+\infty} d_i = \bigsqcup_{n\geq 0} d_i = \bigsqcup \{d_n\mid n\geq 0\}
$$

这里上界的定义和数学分析中实数集上界的定义是一致的. 而在数学分析中, **最小上界**一般被叫做**上确界** (*supremum*). 我们会混用这两个名字.

注意, 上界不一定在链中, 例如在通常的"小于"下, 实数列 $\{1 - 2^{-n}\}$ 的最小上界 1 就不在这个数列中. 上界也不一定存在, 例如整数列 $\{n\}$ 就没有上界. 但如果上界存在, 那么最小上界显然是存在且唯一的.

在实数中, 存在上界则必然存在上确界, 但是在一般的偏序集中不一定如此. 假设我们有元素 $\{0, 1, \dots, \omega_{1}, \omega_{2}\}$, 其中我们规定所有的自然数 $i$ 都满足 $i\sqsubseteq \omega_1$ 而 $i\sqsubseteq \omega_2$, 但 $\omega_1$ 和 $\omega_2$ 之间没有序关系. 那么数列 $\{0, 1, 2, \dots\}$ 有两个上界: $\omega_1$ 和 $\omega_2$, 但是它没有最小上界, 因为它的两个上界之间无法比较.

下面, 我们列举一些最小上界所具有的性质.

**性质 1.** 如果链 $\{d_i\}, \{e_i\}\subseteq D$ 而且对任意的 $i\in\mathbb{N}$ 都有 $d_i\sqsubseteq e_i$, 那么只要它们的上确界都存在, 那么

$$
\bigsqcup_{i=0}^{+\infty} d_i\sqsubseteq \bigsqcup_{i=0}^{+\infty} e_i
$$

**证明.** 设 $\{e_i\}$ 的上确界是 $e$. 那么 $d_i \sqsubseteq e_i \sqsubseteq e$, 所以 $e$ 是 $\{d_i\}$ 的上界. 考虑到 $\bigsqcup d_i$ 是 $\{d_i\}$ 的最小上界, 我们知道 $\bigsqcup d_i\sqsubseteq e$. 这就完成了证明.

**性质 2.** 对于链 $\{d_i\}$, 如果其上确界存在, 那么对任意有限的 $N\in\mathbb{N}$ 都有

$$
\bigsqcup_{i=0}^{+\infty} d_i = \bigsqcup_{i=N}^{+\infty} d_i
$$

**证明.** 显然等式左边 ($\{d_i\}$ 的上确界) 是 $\{d_{i+N}\}$ 的上界, 所以它不小于 $\{d_{i+N}\}$ 的上确界. 同理, 等式右边 $\{d_{i+N}\}$ 的上确界也是 $\{d_i\}$ 的上界, 不小于 $\{d_i\}$ 的上确界. 这说明它们相等.

<a name="prop-3"></a>
**性质 3.** 如果 $\exists n\in \mathbb{N}, \forall i\geq n, d_i = d_n$, 那么 $\{d_i\}$ 最终是常数. 这时, $\{d_i\}$ 的上确界是 $d_n$.

证明是显然的, 这里就不叙述了.

## 完备偏序集

**定义 3.** 如果一个偏序集 $(D, \sqsubseteq)$ 的每一条链都有上确界, 那么 $D$ 是**完备偏序集** (*chain complete poset*, CPO).

值得注意的是, 如果 $D$ 是有限的, 那么这个偏序集必然是一个 CPO. 这是因为它的每一条链最终都是常数, 而根据上面的[性质 3](#prop-3), 我们知道每条链都有上确界.

此外, 自然数集 $\mathbb{N}$ 就不是 CPO, 因为数列 $\{0, 1, 2, \dots\}$ 没有上确界.

**定义 4.** 一个**偏序域** (*domain*) 是一个有最小元的 CPO.

我们来看看几个特别的偏序域.

如果我们给自然数集加上一个元素 $\bot$, 然后规定对所有 $i\in \mathbb{N}$ 都满足 $\bot\sqsubseteq i$, 而且自然数之间不可比较, 那么这就构成了一个偏序域. 这是因为每条链都是有限的, 不重复的元素最多只有 $\bot$ 和另一个自然数, 共计两个, 所以上确界存在; 而且整个集合存在最小元 $\bot$. 我们会将它记作 $\mathbb{N}_\bot$. 
