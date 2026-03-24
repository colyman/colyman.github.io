---
layout: article
title: "BJMM Algorithm"
date: 2026-03-24
key: BJMM-Algorithm-2026
lang: zh
tags: [Code-Based-Cryptography, Information-Set-Decoding, ISD, McEliece, BJMM, 2012]
bilingual: true
mathjax: true
---

**tl;dr**: BJMM 算法（Becker-Joux-May-Meurer，Eurocrypt 2012）将随机线性码的 ISD 复杂度从 $2^{0.05364n}$ 降至 $2^{0.04934n}$。核心创新：**允许索引集合重叠**（$I_1 \cap I_2 = \varepsilon$），利用 $\mathbf{F}_2$ 域中 $1+1=0$ 的性质同时拆分 0-位和 1-位，显著增加每个解的表示数量，从而降低有效搜索复杂度。

<!-- more -->

## 1. 背景：信息集解码（ISD）

设 $C$ 为随机二元线性 $[n, k, d]$ 码，其校验矩阵为 $H \in \mathbf{F}_2^{(n-k)\times n}$。给定综合征 $s = He$，其中错误向量 $e \in \mathbf{F}_2^n$ 的汉明权重为 $\omega = \lfloor (d-1)/2 \rfloor$（半距离解码），**综合征解码问题**要求从 $s$ 恢复 $e$。该问题是 McEliece 公钥密码系统安全性的核心假设。

Prange 的 ISD 框架（1962）包含两步：

### 1.1 初始变换

随机置换 $H$ 的列（$P \in \mathbf{F}_2^{n \times n}$），再通过高斯消元（左乘可逆矩阵 $T \in \mathbf{F}_2^{(n-k)\times(n-k)}$）得到准系统形式：

$$\widetilde{H} = T \cdot H \cdot P = \begin{pmatrix} Q & I_{n-k} \end{pmatrix},\quad Q \in \mathbf{F}_2^{(n-k)\times k}.$$

设 $\tilde{e} = Pe$，则 $\widetilde{H}\tilde{e} = \tilde{s} = Ts$。由于置换 $P$ 随机打乱了 $e$ 中 $\omega$ 个 1 的位置，$\tilde{e}$ 恰好在前 $k$ 位有 $p$ 个 1、后 $n-k$ 位有 $\omega - p$ 个 1 的概率为：

$$P = \frac{\binom{k}{p}\binom{n-k}{\omega-p}}{\binom{n}{\omega}}.$$

### 1.2 搜索阶段

找到截断错误向量 $\tilde{e}_1 \in \mathbf{F}_2^k$（前 $k$ 位），使得 $Q\tilde{e}_1 = \tilde{s}$ 在前 $n-k$ 个坐标上成立。这正是 **子矩阵匹配问题（SMP）**，本身就是一类综合征解码实例（参数 $[k+\ell, \ell, p]$）。

## 2. 子矩阵匹配问题（SMP）

给定 $Q \in \mathbf{F}_2^{\ell \times (k+\ell)}$（$\ell = n-k$）和目标向量 $s \in \mathbf{F}_2^\ell$，求集合 $I \subseteq [k+\ell]$，$|I| = p$，使得：

$$\sigma(Q_I) := \sum_{i \in I} q_i = s,$$

其中 $q_i$ 是 $Q$ 的第 $i$ 列。改善 SMP 的算法即可直接提升 ISD 的效率。

## 3. MMT 的表示技术

MMT 算法（Asiacrypt 2011）首先将表示技术引入 ISD：将解集 $I$ 分解为两个**不相交**的索引集 $I = I_1 \cup I_2$，其中 $|I_1| = |I_2| = p/2$，$I_1 \cap I_2 = \emptyset$。

这给出每个解 $\binom{p}{p/2}$ 种表示。MMT 的局限在于：**仅拆分 1-位**——每个 1 要么出现在第一个半集中 $(1,0)$，要么出现在第二个半集中 $(0,1)$。

## 4. 核心洞察：$1+1=0$ 在 $\mathbf{F}_2$ 中的应用

BJMM 的突破在于认识到在二元域 $\mathbf{F}_2$ 中，等式 $1 + 1 = 0$ 允许我们**同时拆分 0-位**：

| 原始位 | 拆分方式 | 说明 |
|--------|---------|------|
| 1 | $(0, 1)$ 或 $(1, 0)$ | ✓ MMT 已用 |
| **0** | $(0, 0)$ 或 $(1, 1)$ | **✓ BJMM 新增** |

由于编码中 $\omega \ll n$（错误权重远小于码长），错误向量中 0 的数量远多于 1。允许两种类型的拆分后，每个解的表示数量大幅增加。

### 重叠索引集合

BJMM 选择 $|I_1| = |I_2| = p/2 + \varepsilon$，且 $|I_1 \cap I_2| = \varepsilon$。通过**对称差**恢复原集合：

$$I = I_1 \;\Delta\; I_2 = (I_1 \cup I_2) \setminus (I_1 \cap I_2).$$

交集 $I_1 \cap I_2$ 中的元素同时出现在两侧，相加后贡献 $1+1=0$，即在列求和时相互抵消。这使得 $k+\ell-p$ 个 0-位可以以 $(1,1)$ 的形式出现在两侧而不改变结果。

每个解的表示数量变为：

$$R_1(p, \ell; \varepsilon_1) = \binom{p}{p/2} \cdot \binom{k+\ell-p}{\varepsilon_1}.$$

相比 MMT，增加了乘性因子 $\binom{k+\ell-p}{\varepsilon_1}$。

BJMM **应用表示技术两次**（两个层次），得到：

$$R_2(p, \ell; \varepsilon_1, \varepsilon_2) = \binom{p}{p/2} \cdot \binom{k+\ell-p}{\varepsilon_1} \cdot \binom{p/2+\varepsilon_1}{p/4+\varepsilon_2/2} \cdot \binom{k+\ell-p/2-\varepsilon_1}{\varepsilon_2}.$$

## 5. MERGE-JOIN：基础构建块

BJMM 的核心操作是 **MERGE-JOIN**。给定两个向量列表 $L_1, L_2$、目标权重 $p$、偏置坐标数 $r$ 和目标向量 $t \in \mathbf{F}_2^r$，MERGE-JOIN 返回：

$$L = L_1 \bowtie L_2 = \{ x + y \mid x \in L_1, y \in L_2, \mathrm{wt}(x+y) = p,\ (Q(x+y))[r] = t \}.$$

**匹配机制**：将 $L_1$ 按标签 $(Qx)[r]$ 排序，将 $L_2$ 按 $(Qy)[r] + t$ 排序，然后用双指针线性扫描所有标签相同的碰撞对（Knuth 算法）。碰撞对即满足 $(Q(x+y))[r] = t$ 的向量对。过滤掉权重不等于 $p$ 的不一致解和重复元素。

运行时间：$\widetilde{O}(\max\{|L_1|, |L_2|, C\})$，其中 $C$ 为碰撞数。标签均匀分布时 $\mathbb{E}[C] = |L_1| \cdot |L_2| \cdot 2^{-r}$。

## 6. COLUMN MATCH：三层计算树

BJMM 通过由 MERGE-JOIN 操作构成的三层计算树来求解 SMP，结构类似于 Wagner 的广义生日问题。

### 参数设置

- $p_1 = p/2 + \varepsilon_1$
- $p_2 = p_1/2 + \varepsilon_2 = p/4 + \varepsilon_1/2 + \varepsilon_2$
- $r_1 \approx \log_2 R_1$：第一层约束的坐标数
- $r_2 \approx \log_2 R_2$：第二层约束的坐标数

### 第 3 层：不相交基列表

对 $i = 1, \ldots, 4$，将 $[k+\ell]$ 随机划分为等大的两部分 $P_{i,1}, P_{i,2}$，构造不相交的基列表：

$$B_{i,1} = \{ y \in \mathbf{F}_2^{k+\ell} \mid \mathrm{wt}(y) = p_2/2,\ y_j = 0\ \forall j \in P_{i,2} \}$$
$$B_{i,2} = \{ z \in \mathbf{F}_2^{k+\ell} \mid \mathrm{wt}(z) = p_2/2,\ z_j = 0\ \forall j \in P_{i,1} \}$$

然后：
$$L^{(2)}_i = \text{MERGE-JOIN}(B_{i,1},\ B_{i,2},\ r_2,\ p_2,\ t^{(2)}_i).$$

基列表大小：$S_3 = \binom{(k+\ell)/2}{p_2/2}$。

### 第 2 层：合并形成 $L^{(1)}_j$

$$L^{(1)}_1 = \text{MERGE-JOIN}(L^{(2)}_1,\ L^{(2)}_2,\ r_1,\ p_1,\ t^{(1)}_1)$$
$$L^{(1)}_2 = \text{MERGE-JOIN}(L^{(2)}_3,\ L^{(2)}_4,\ r_1,\ p_1,\ t^{(1)}_2)$$

### 第 0 层（最终）：合并得到解列表

$$L = \text{MERGE-JOIN}(L^{(1)}_1,\ L^{(1)}_2,\ \ell,\ p,\ s)$$

计算树结构图示：

```
第3层:  B1,1  B1,2 | B2,1  B2,2 | B3,1  B3,2 | B4,1  B4,2   (不相交基列表)
         ───────────────   ───────────────   ───────────────   ─────────
第2层:      L(2)_1            L(2)_2            L(2)_3            L(2)_4
               ───────────────   ───────────────
第1层:           L(1)_1                  L(1)_2
               ────────────────────────────────
第0层:                      L  →  SMP 的解
```

### 伪代码

```
ALGORITHM COLUMN MATCH(Q, s, p)
参数: ε1, ε2, p1 = p/2 + ε1, p2 = p1/2 + ε2

1. 选择随机划分 Pi,1, Pi,2 和基列表 Bi,1, Bi,2
2. 选择 t(1)_1 ∈R Fr1, 设 t(1)_2 = s[r1] + t(1)_1
3. 选择 t(2)_1, t(2)_3 ∈R Fr2.
   设 t(2)_2 = (t(1)_1)[r2] + t(2)_1, t(2)_4 = (t(1)_2)[r2] + t(2)_3
4. 对 i = 1 to 4: L(2)_i = MERGE-JOIN(Bi,1, Bi,2, r2, p2, t(2)_i)
5. 对 j = 1 to 2: L(1)_j = MERGE-JOIN(L(2)_(2j-1), L(2)_(2j), r1, p1, t(1)_j)
6. L = MERGE-JOIN(L(1)_1, L(1)_2, ℓ, p, s)
7. 返回 L
```

## 7. 复杂度分析

### 时间复杂度

三层 MERGE-JOIN 的运行时间主导整体复杂度。设 $S_i$ 为第 $i$ 层期望列表大小，$C_i$ 为期望碰撞数：

$$S_i = \binom{k+\ell}{p_i} \cdot 2^{-r_i}, \quad \mathbb{E}[C_i] = S_i^2 \cdot 2^{r_i - r_{i-1}}.$$

时间复杂度为：

$$T(p, \ell; \varepsilon_1, \varepsilon_2) = \max\{ T_3, T_2, T_1 \} = \max\{ S_3, C_3, S_2, C_2, S_1, C_1 \}.$$

考虑每次 ISD 迭代的成功概率：

$$P^{-1} = \frac{\binom{n}{\omega}}{\binom{k+\ell}{p}\binom{n-k-\ell}{\omega-p}}.$$

在随机线性码上优化参数 $p, \ell, \varepsilon_1, \varepsilon_2$ 后：

$$T_{\text{BJMM}} = 2^{0.04934n} \quad \text{（半距离解码，最坏情况）}.$$

### 空间复杂度

主导内存为最大中间列表大小，约为 $2^{0.0286n}$。

### McEliece 典型参数

设码率 $R = 0.7577$，相对距离 $D = 0.04$（典型 McEliece 参数）：

| 指标 | 半距离解码 | 完全解码 |
|------|-----------|----------|
| 时间 | $2^{0.0672n}$ | $2^{0.1019n}$ |
| 空间 | $2^{0.0586n}$ | $2^{0.0769n}$ |

## 8. 与前人工作的对比

| 算法 | 复杂度指数 | 关键技术 |
|------|-----------|----------|
| Lee-Brickell | $2^{0.05752n}$ | $p < \omega$ 优化 |
| Stern | $2^{0.05564n}$ | 中途相遇（MITM） |
| Ball-collision | $2^{0.05559n}$ | 非精确匹配 |
| MMT | $2^{0.05364n}$ | 表示技术，不相交 $I_1, I_2$ |
| **BJMM** | **$2^{0.04934n}$** | 重叠集合，$1+1=0$ 拆分 |

BJMM 相比 MMT 的提升因子为 $2^{0.0043n}$，在 $n = 2000$ 时约为 $3\times$ 加速，直接影响 McEliece 各安全等级的参数选择。

## 9. 可证明版本

启发式分析依赖于投影部分和 $(Qe)[r]$ 的均匀性。论文还给出了 **PROVABLE CM**：反复使用新鲜随机目标值调用 COLUMN MATCH，并在任意列表超过期望大小 $2^{\gamma n}$ 倍时中止（常数 $\gamma > 0$）。除可忽略一部分矩阵 $Q$ 外，该变体以 $> 1 - 3/e^2$ 的概率成功，运行时间为 $\widetilde{O}(T \cdot 2^{3\gamma n})$。

## 参考文献

1. A. Becker, A. Joux, A. May, A. Meurer. *Decoding Random Binary Linear Codes in $2^{n/20}$: How $1+1=0$ Improves Information Set Decoding.* Eurocrypt 2012. ([ePrint](https://eprint.iacr.org/2012/026))
