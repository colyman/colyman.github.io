---
layout: article
title: "BJMM + May-Ozerov ISD：近邻搜索优化"
date: 2026-03-25
key: BJMM-May-Ozerov-ISD-2026
lang: zh
tags: [Code-Based-Cryptography, ISD, May-Ozerov, Nearest-Neighbor, Walsh-Hadamard, BJMM, McEliece]
bilingual: true
mathjax: true
---

**tl;dr** May-Ozerov 算法（EUROCRYPT 2015）用 Walsh-Hadamard 变换（WHT）替代 Stern ISD 中的排序匹配（sort-and-match），将近邻搜索问题（NMP）的求解效率提升了一个量级。结合 BJMM 可将 ISD 复杂度降至约 \$2^{0.053n}\$。然而 2025 年的研究表明，这个改进在实践中是"银河级"的（galactic）——仅在 n > 1,874,400 时才比 Stern 算法更快。本文详细介绍 HIVe（Hybrid Inbound Variant Ensemble）算法、近邻搜索的 WHT 方法，以及其在 ISD 中的应用与局限性。

Useful references:

- [May, Ozerov — EUROCRYPT 2015](https://eprint.iacr.org/2015/437)
- [Hirose — PQCrypto 2016 (generalization to Fq)](https://eprint.iacr.org/2016/237)
- [Bouillaguet, Delaplace, Hamdad — CIC 2025 (galactic analysis)](https://cic.iacr.org/p/2/1/31)

---

## 1. 背景：Stern ISD 的瓶颈

前文（[BJMM Algorithm](/2026/03/24/BJMM-Algorithm-zh/)）已经详细介绍了 BJMM 算法，它通过重叠索引集合和 \$1+1=0\$ 技巧将 ISD 复杂度降至 \$2^{0.04934n}\$。然而，无论是 BJMM 还是 MMT，其核心操作都是 **MERGE-JOIN**——一种基于排序的匹配算法。

回顾一下 Stern ISD 的结构：

```
1. 高斯消元：H → (Q | I)
2. 构造列表：
   U = { Q·e₁ | e₁ ∈ F₂^{k/2}, wt(e₁) = p/2 }
   V = { e₂ ⊕ s | e₂ ∈ F₂^{n-k}, wt(e₂) = p/2 }
3. 排序匹配：找 (u,v) ∈ U×V，使 u⊕v 在最后 ℓ 位匹配
```

第 3 步是瓶颈：列表大小为 \$2^{k/2}\$，朴素匹配需要 \$2^k\$ 次比较。即使排序后用双指针，复杂度也是 \$\widetilde{O}(2^{k/2})\$。

> **核心问题**：能否用比排序匹配更高效的方法来找到近邻对？

## 2. Walsh-Hadamard 变换（WHT）

在介绍 May-Ozerov 算法之前，先回顾 Walsh-Hadamard 变换（WHT）。

### 2.1 定义

对于布尔函数 \$f: \mathbf{F}_2^m \to \mathbf{F}_2\$，其 Walsh 变换定义为：

$$W_f(u) = \sum_{x \in \mathbf{F}_2^m} (-1)^{f(x) + u \cdot x}, \quad u \in \mathbf{F}_2^m$$

其中 \$u \cdot x = \sum_i u_i x_i \pmod{2}\$ 是点积。

WHT 的快速算法（FWT）在 \$O(m \cdot 2^m)\$ 时间内计算所有 \$2^m\$ 个 Walsh 系数，通过蝶形操作（butterfly operations）实现：

```
FWT(f):
  for stage = 1 to m:
    for block = 0 to 2^m - 1 step 2^stage:
      for i = 0 to 2^{stage-1} - 1:
        x = f[block + i]
        y = f[block + i + 2^{stage-1}]
        f[block + i] = x + y
        f[block + i + 2^{stage-1}] = x - y
```

### 2.2 WHT 与汉明距离

WHT 的一个关键应用是计算汉明距离的频谱。设 \$u, v \in \mathbf{F}_2^m\$，定义 \$d = w_H(u \oplus v)\$。则：

$$\sum_{x \in \mathbf{F}_2^m} (-1)^{w_H(x)} \cdot (-1)^{u \cdot x} = \sum_{x \in \mathbf{F}_2^m} (-1)^{w_H(x \oplus u) \cdot v}$$

通过 WHT，我们可以高效地找到与给定向量汉明距离为特定值的向量集合。

## 3. 近邻搜索问题（NMP）

### 3.1 问题定义

**定义（近邻搜索问题，Binary Case）**：设 \$0 < \delta < 1/2\$，\$0 < \tau < 1\$。给定两个集合 \$U, V \subset \mathbf{F}_2^m\$, \$|U| = |V| = 2^m\$（随机且两两独立），求所有满足 \$w_H(u \oplus v) = \delta m\$ 的配对 \$(u, v) \in U \times V\$。

这个问题恰好出现在 Stern ISD 的排序匹配步骤中：
- \$U\$ 中的向量是 \$Qe_1\$（权重 p/2）
- \$V\$ 中的向量是 \$e_2 \oplus s\$（权重 p/2）
- 我们要找 \$u \oplus v\$ 的汉明权重 = p

### 3.2 朴素算法

朴素算法的时间复杂度是 \$O(2^m \cdot 2^m) = O(2^{2m})\$，完全不可行。

排序匹配的时间复杂度是 \$\widetilde{O}(2^m)\$（先排序，再扫描匹配），这是 Stern ISD 采用的方法。

> **May-Ozerov 的目标**：用 WHT 将复杂度降至 \$2^{\alpha m}\$（其中 \$\alpha < 1\$），从而改善 ISD 的整体复杂度。

## 4. HIVe 算法：WHT 驱动的近邻搜索

HIVe（Hybrid Inbound Variant Ensemble）是 May-Ozerov 在 EUROCRYPT 2015 中提出的 WHT 基近邻搜索算法。其核心思想是将问题分解为多个多项式大小的子问题，然后在子问题上用朴素搜索。

### 4.1 核心洞察

将每个向量随机化（随机置换 \$P\$ + 随机偏移 \$r\$）：
$$\widetilde{U} = \{ \widetilde{u} = Pu \oplus r \mid u \in U \}$$

这样做的目的是打乱向量的结构，使得：
1. 未知解（真正的近邻对）在随机化后的集合中仍然以恒定概率存在
2. 子问题的规模可以通过概率分析来控制

### 4.2 递归过滤

HIVe 使用递归过滤（Recursive Filtering）来构建一棵搜索树：

```
MO-NN(U, V, δ):
  1. 随机化：选择随机置换 P 和随机偏移 r
  2. 递归构建：
     for 每个坐标子集 A ⊂ [m]:
       U_A = { u ∈ U | 坐标 A 中 1 的数量满足某条件 }
       V_A = { v ∈ V | 坐标 A 中 1 的数量满足某条件 }
       if |U_A|, |V_A| 多项式大小:
         递归调用 MO-NN(U_A, V_A, δ)
  3. 叶子节点：朴素搜索
```

### 4.3 分而治之的图示

```
根节点：全部 2^m 个向量
     │
     ├─ A₁ (第一个坐标子集) ──┬─ 满足平衡条件的 U_A, V_A
     ├─ A₂ (第二个坐标子集) ──┤  (大小 ~多项式)
     ├─ ...
     └─ A_t (第 t 个坐标子集)
         │
         └─ 叶子：朴素搜索
```

### 4.4 WHT 在过滤中的应用

关键是 **WHT 可以高效计算汉明权重的分布**。当我们需要找到"坐标子集 A 中 1 的数量为 r"的向量时：
- 对 U 中的所有向量计算 WHT
- 找到汉明权重恰好为 r 的 Walsh 系数峰值
- 对应的原始向量就是候选向量

更具体地说，May-Ozerov 使用了以下观察：

设向量 \$x \in \mathbf{F}_2^m\$，将其分成两部分：\$x = (x_1 \mid x_2)\$，其中 \$x_1, x_2 \in \mathbf{F}_2^{m/2}\$。

定义函数 \$f_{A,r}(x) = 1\$ 当且仅当 x 在坐标子集 A 中的汉明权重为 r。

$$WHT(f_{A,r})(u) = \sum_{x} (-1)^{f_{A,r}(x) + u \cdot x}$$

当 \$WHT(f_{A,r})(u) = 2^m\$ 时，说明存在唯一的 x 满足条件。

### 4.5 复杂度分析

从 Hirose 的 survey 中，May-Ozerov 算法的时间复杂度为：

$$T = \widetilde{O}(q^{(y+\varepsilon)m})$$

其中 q 是有限域大小（二进制情况下 q=2），y 是一个与 δ 相关的量：

$$y = (1-\delta) \cdot \left( H_q(\delta) - \frac{1}{q}\sum_{x \in \mathbf{F}_q} H_q(q h_x \delta) \right)$$

对于二进制情况（q=2），Hirose 的 Table 1 给出：
- Stern-MO 的复杂度指数：\$f(2; R_w) = 0.05498\$
- 原始 Stern ISD：\$f_S(2; R_w') = 0.05563\$

提升：\$\Delta = -0.00065\$（约 \$2^{0.00065n}\$ 倍）

## 5. 应用于 ISD：Stern-MO 和 BJMM-MO

### 5.1 Stern-MO

将 HIVe 嵌入 Stern ISD 的排序匹配步骤：

1. **高斯消元**：\$H \to (Q \mid I)\$
2. **构造列表**：
   $$U = \{ Qe_1 \mid e_1 \in \mathbf{F}_2^{k/2}, wt(e_1) = p/2 \}$$
   $$V = \{ e_2 \oplus s \mid e_2 \in \mathbf{F}_2^{n-k}, wt(e_2) = p/2 \}$$
3. **HIVe 搜索**：用 WHT 驱动的近邻搜索替代排序匹配
   - 在 U×V 中找汉明距离为 p 的配对

### 5.2 BJMM-MO

BJMM 的 MERGE-JOIN 操作同样包含排序匹配步骤。将其替换为 HIVe：

```
BJMM-MO(Q, s, p):
  1. 参数设置：p₁ = p/2 + ε₁, p₂ = p₁/2 + ε₂
  2. 第 3 层：构造基列表 B_{i,1}, B_{i,2}
  3. 用 HIVe 替代 MERGE-JOIN 进行匹配
  4. 三层树结构依然，但每层的匹配操作使用 WHT
```

原始 May-Ozerov 论文报告 BJMM-MO 的复杂度约为 \$2^{0.053n}\$。

### 5.3 复杂度对比

| 算法 | 复杂度指数（半距离） | 核心技术 |
|------|-------------------|---------|
| Lee-Brickell | \$2^{0.05752n}\$ | \$p < \omega\$ |
| Stern | \$2^{0.05563n}\$ | MITM |
| MMT | \$2^{0.05364n}\$ | 表示技术 |
| BJMM | \$2^{0.04934n}\$ | 重叠集合 |
| Stern-MO | \$2^{0.05498n}\$ | WHT-NN |
| BJMM-MO | \$2^{0.053n}\$（约） | BJMM + WHT-NN |

## 6. "银河级"分析：为什么改进在实践中不可用？

2025 年，Bouillaguet、Delaplace 和 Hamdad 在 *IACR Communications in Cryptology* 上发表了里程碑式的分析。他们的核心发现：

### 6.1 交叉点分析

May-Ozerov 算法相比 Stern 算法的交叉点（crossover point）：

| 指标 | 值 |
|------|-----|
| 最小码长 n | > 1,874,400 |
| 交叉点所需运算数 | > \$2^{63,489}\$ |

这意味着：
- 当 n ≤ 1,874,400 时，Stern 算法（甚至 Lee-Brickell）始终比 May-Ozerov 快
- 1,874,400 是什么概念？已知最大的 McEliece 参数是 n=6960
- 因此，**在所有实际的密码学相关参数下，May-Ozerov 都不如 Stern**

### 6.2 原因分析

为什么理论上有优势的算法在实践中反而更慢？

1. **隐藏的多项式因子**：~O 记号中隐藏了巨大的多项式因子。WHT 的复杂度是 \$O(m \cdot 2^m)\$，其中 m 是向量长度。对于 Stern ISD 中的典型参数，m ≈ n/2，这些因子主导了实际运行时间。

2. **常数因子**：WHT 需要复杂的蝶形操作，而排序匹配只需简单的比较操作。在实际规模下，WHT 的常数因子远大于排序。

3. **缓存局部性**：排序后的数据在内存中是连续的，缓存友好；WHT 需要大量的随机内存访问。

```
Stern (sort-and-match):        May-Ozerov (WHT):
┌─────────────────────┐        ┌─────────────────────┐
│ 排序: O(m log m)    │        │ WHT: O(m · 2^m)    │
│ 扫描: O(2^m)        │        │ 过滤: 多项式递归    │
│ 总计: ~O(2^m)       │        │ 总计: ~O(2^{αm})   │
└─────────────────────┘        └─────────────────────┘
  ↑ 常数小，缓存好                 ↑ 常数大，内存乱
```

### 6.3 银河算法的定义

David Johnson 提出的"银河算法"（Galactic Algorithm）定义：一个算法在理论上是最好的，但只在输入规模达到"填满整个宇宙"时才实际优于简单方法。

May-Ozerov 完美符合这个定义：
- 理论上：复杂度指数 0.05498 < 0.05563 ✓
- 实践中：需要 n > 1,874,400 ✗（宇宙中任何已知的编码都不接近这个规模）

## 7. 结论

May-Ozerov 算法是用 Walsh-Hadamard 变换优化 ISD 中近邻搜索的里程碑工作。它展示了 WHT 在组合优化中的强大能力，并将 ISD 的复杂度指数从 \$2^{0.05563n}\$（Stern）降至 \$2^{0.05498n}\$。

然而，2025 年的研究表明，这个改进在实践中是"银河级"的：
- 所有实际的 McEliece 参数（n < 10000）都远未达到交叉点
- 隐藏的多项式因子和常数因子使得 WHT 方法在实践中不如简单排序

这个教训提醒我们：**渐进复杂度分析和渐近复杂度分析**（即忽略多项式因子）是两回事。在密码学这样注重实际安全边界的领域，我们需要同时关注两者的分析。

## 参考文献

1. A. May, I. Ozerov. *On Computing Nearest Neighbors with Applications to Decoding of Binary Linear Codes.* EUROCRYPT 2015. ([ePrint](https://eprint.iacr.org/2015/437))
2. S. Hirose. *May-Ozerov Algorithm for Nearest-Neighbor Problem over Fq and Its Application to Information Set Decoding.* PQCrypto 2016. ([ePrint](https://eprint.iacr.org/2016/237))
3. C. Bouillaguet, C. Delaplace, M. Hamdad. *The May-Ozerov Algorithm for Syndrome Decoding is "Galactic".* IACR Communications in Cryptology (2025). ([Link](https://cic.iacr.org/p/2/1/31))
4. A. Becker, A. Joux, A. May, A. Meurer. *Decoding Random Binary Linear Codes in 2^{n/20}: How 1+1=0 Improves Information Set Decoding.* EUROCRYPT 2012.
5. J. Stern. *A method for finding codewords of small weight.* Coding Theory 1989.
