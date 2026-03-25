---
layout: article
title: "Regular ISD：定制 ISD 技术攻击结构化解码"
date: 2026-03-26
key: Regular-ISD-2026
lang: zh
tags: [Code-Based-Cryptography, Information-Set-Decoding, Regular-Syndrome-Decoding, RSD, Esser-Santini, 2024]
bilingual: true
mathjax: true
---

**tl;dr** Regular Syndrome Decoding (RSD) 是标准伴随式解码 (SD) 的结构化变体，其错误向量按块划分、每块恰好含一个非零元。Esser–Santini (CRYPTO 2024) 的核心贡献是：**将高级 ISD 技术针对 RSD 的环结构进行定制**（Regular-ISD），将复杂度从 CCJ 的 $2^{0.141n}$ 降至 $2^{0.112n}$，同时揭示了 RSD 难度的 regime 依赖性——单纯用唯一解条件不足以刻画最坏情况。

<!-- more -->

## 1. 从 SD 到 RSD：结构化的代价

### 标准伴随式解码 (SD)

设 $H \in \mathbb{F}_2^{(n-k)\times n}$ 为一致校验矩阵，$s \in \mathbb{F}_2^{n-k}$ 为伴随式。**伴随式解码问题 (SD)** 要求寻找错误向量 $e \in \mathbb{F}_2^n$，满足 $\mathrm{wt}(e) = \omega$ 且 $He^\top = s$。

SD 是 McEliece 公钥密码体系的底层困难问题。其通用求解算法为 **Information Set Decoding (ISD)**，从 Prange (1962) 的 $O(n)$ 指数算法演进至今，最优复杂度为 **BJMM** 的 $2^{0.04934n}$。

### Regular SD：结构化的错误分布

然而，许多高效的密码构造（特别是签名方案）需要比随机 SD **更高效** 的困难问题。这催生了 **Regular Syndrome Decoding (RSD)** 的引入——将错误分布结构化：

**定义** (RSD): 设 $n = N \times t$，将错误向量划分为 $t$ 个大小为 $N$ 的块：

$$
e = (e_1 \ \| \ e_2 \ \| \ \cdots \ \| \ e_t), \quad e_i \in \mathbb{F}_2^N.
$$

RSD 要求每个块恰好含一个非零元：

$$
\forall i \in [1,t]: \quad \mathrm{wt}(e_i) = 1.
$$

换言之，$e$ 的 Hamming 重量恒为 $t$，但其支持集被限制在恰好 $t$ 个"对齐"的位置——每块一个。

**独立性问题**：虽然每块的非零位置是独立的（在 $N$ 个候选中选一个），但各块之间没有约束。这使得 RSD 的解空间（$N^t$ 种可能）远小于随机 SD 的解空间（$\binom{n}{t}$ 种），在某些参数下反而更难攻击。

## 2. 环结构：$\mathbb{F}_2[X]/(X^n - 1)$

RSD 的关键数学结构是多项式环：

$$
R_n := \mathbb{F}_2[X] \bmod (X^n - 1).
$$

将 $H$ 和 $s$ 编码为多项式后，伴随式方程 $He^\top = s$ 等价于环中的**乘法方程**：

$$
h(X) \cdot e(X) \equiv s(X) \pmod{X^n - 1}.
$$

其中 $e(X)$ 的形式尤为特殊——由于错误分块结构：

$$
e(X) = \sum_{j=0}^{t-1} X^{j \cdot N} \cdot u_j(X),
$$

其中每个 $u_j(X)$ 是单项式（恰好有一个非零项）。

### 环的分解与陪集

环 $R_n$ 的结构由多项式 $X^n - 1$ 的分解决定。在 $\mathbb{F}_2$ 上：

$$
X^n - 1 = \prod_{C \in \mathcal{C}_n} m_C(X),
$$

其中 $\mathcal{C}_n$ 是 $\langle 2 \rangle$ 在 $\mathbb{Z}_n$ 上作用的**2-陪集**（cyclotomic cosets），$m_C$ 是相应陪集的极小多项式。

由**中国剩余定理 (CRT)**：

$$
R_n \cong \bigoplus_{C \in \mathcal{C}_n} \mathbb{F}_2[X]/(m_C(X)) = \bigoplus_{C \in \mathcal{C}_n} \mathbb{F}_{2^{|C|}}.
$$

每个分量 $\mathbb{F}_{2^{|C|}}$ 的维数为 $|C|$（陪集大小）。方程 $h(X)\cdot e(X) = s(X)$ 在环中分解为**各分量独立**的子方程：

$$
h_C \cdot e_C = s_C \quad \text{在 } \mathbb{F}_{2^{|C|}} \text{ 中}.
$$

这正是 Esser–Santini 的核心洞察：**通过陪集分解将整体问题拆分为若干独立子问题**。

## 3. 参数 Regime：唯一解不是全部

一个自然的问题是：RSD 在什么参数下是困难的？此前的研究倾向于用**唯一解条件**来刻画。

Esser–Santini 的第一个重要贡献是对 RSD 难度进行**系统性分类**，揭示了一个反直觉事实：

> **分类仅基于唯一解是不充分的。**  
> 存在具有唯一解的多项式时间可解实例，也存在无唯一解但极其困难的实例。

具体而言，他们划分了三类参数 regime：

| Regime | 特点 | 难度 |
|--------|------|------|
| 大 $N$（多块，每块小）| 大量块，每块选择有限 | 多项式时间 |
| 中等 $N$ | 块大小适中 | 最坏情况，难 |
| 小 $N$（少块，每块大）| 少量块，选择空间大 | 接近标准 SD |

## 4. 前期攻击：代数与线性化

### Briaud–Øygarden (Eurocrypt 2023)

Briaud 和 Øygarden 提出了首个系统性**代数攻击**，将 RSD 建模为多项式方程组：

**建模**（以有限域 $\mathbb{F}_q$ 为例）：

- **线性部分** $P$：$n-k$ 个线性方程，来自 $He^\top = s$
- **正则结构** $B$：二次约束，刻画每块的唯一非零元

$$
e_{i,j_1} \cdot e_{i,j_2} = 0 \quad (j_1 \neq j_2)
$$

在 $\mathbb{F}_2$ 上还需加入域方程和块唯一性约束。求解使用 **Macaulay 矩阵**——将所有 $\leq d$ 次单项式排成列、所有多项式排成行，通过行阶梯化发现解的结构。

### CCJ 攻击 (Eurocrypt 2023)

Carozza、Couteau 和 Joux 提出了基于**线性化**的攻击框架，给出两个算法：

| 算法 | 复杂度 |
|------|--------|
| CCJ-1 | $2^{0.141n}$ |
| CCJ-2 | $2^{0.135n}$ |

其核心思想是将非线性方程组转化为线性系统（线性化），然后用标准方法求解。Esser–Santini 首次给出了 CCJ 算法的**严格渐近分析**。

## 5. 核心贡献：Regular-ISD

### 核心思想

Esser–Santini 的主要贡献是 **Regular-ISD**：将 ISD 的整套高级技术针对 RSD 问题进行定制，而非依赖线性化或纯代数方法。

**关键步骤**：

1. **环分解**：利用陪集分解 $R_n \cong \bigoplus_C \mathbb{F}_{2^{|C|}}$，将问题分解到每个分量
2. **分量 ISD**：在每个分量 $\mathbb{F}_{2^{|C|}}$ 中运行定制 ISD
3. **信息集选择**：在环结构中选择合适的"信息集"，考虑分块约束
4. **表示技术融合**：借鉴 MMT/BJMM 的表示技术，适配 RSD 结构

这与 CCJ 的线性化方法有本质区别——CCJ 试图绕过分块结构；Regular-ISD 则**主动利用**分块结构。

### 复杂度结果

**最佳 Regular-ISD 算法：$2^{0.112n}$**

| 算法 | 复杂度 | 来源 |
|------|--------|------|
| CCJ-1 | $2^{0.141n}$ | Eurocrypt 2023 |
| CCJ-2 | $2^{0.135n}$ | Eurocrypt 2023 |
| **Regular-ISD** | **$2^{0.112n}$** | **CRYPTO 2024** |

相比 CCJ-1，Regular-ISD 将指数从 $0.141$ 降至 $0.112$——相当于在相同安全级别下将 $n$ 缩小约 $21\%$。

## 6. 对实际参数的影响

对于 CCJ 等方案建议的参数，Regular-ISD 的攻击效率比之前预估高得多：

- 对特定参数集，安全级别下降了 **最高 30 bit**
- 参数需要相应增大才能维持目标安全级别

这一发现对基于 RSD 的签名方案具有直接意义：其参数建议需要重新审视。

## 7. 关键要点

1. **RSD 是 SD 的结构化变体**：错误向量分块、每块重量为 1，在环 $\mathbb{F}_2[X]/(X^n-1)$ 中有自然的多项式表示
2. **陪集分解是关键**：$X^n-1$ 的分解通过 2-陪集将问题拆分为独立分量
3. **唯一解条件不足以刻画难度**：RSD 的难度与参数 regime 强相关
4. **Regular-ISD 显著优于前期攻击**：$2^{0.112n}$ vs $2^{0.141n}$，差距来自 ISD 技术的定制化利用
5. **对密码方案的影响**：RSD 类签名方案的现有参数建议需重新评估，安全级别可能下降约 30 bit
