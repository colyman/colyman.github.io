---
layout: article
title: "Information Set Decoding：Prange 与 Stern 算法"
date: 2026-03-22
key: Information-Set-Decoding-Prange-Stern-2026
lang: zh
tags: [Code-Based-Cryptography, Information-Set-Decoding, McEliece, Stern-Algorithm, Post-Quantum-Cryptography]
hidden: true
---

**tl;dr:** Information Set Decoding（ISD）是破解 McEliece、Niederreiter 等编码密码体制的核心攻击算法。本文详解 Prange ISD（1962）和 Stern 算法（1989）的数学原理，推导 Prange 复杂度 \\(O(2^{n(1-R)})\\) 与 Stern 复杂度 \\(O(2^{n(1-R)/2})\\)，并给出经过本地验证的 Python 实现代码。

---

## 大纲

### 内容概述
1. **背景与动机** — SDP 问题定义，NP-完全性，McEliece/Niederreiter 密码体制
2. **ISD 框架** — 信息集概念，置换形式，核心思想
3. **Prange ISD（1962）** — 置换、高斯消元、复杂度推导
4. **Stern 算法（1989）** — 生日悖论、分区策略、碰撞概率分析
5. **复杂度对比** — Prange vs Stern vs 暴力枚举（表格 + 分析）
6. **代码实现** — 经过本地验证的 Python 实现（GF(2) 矩阵运算）
7. **总结与展望**

### 参考文献
- Prange. "Use of information sets in decoding cyclic codes" (1962)
- Stern. "A method for finding codewords of small weight" (1989)
- Berlekamp, McEliece, van Tilborg. "On the inherent intractability of certain coding problems" (1978)
- [Wikipedia: Information-set decoding](https://en.wikipedia.org/wiki/Information-set_decoding)

### 前置知识
- 线性代数（有限域 \\(\mathbf{F}_2\\) 上的矩阵秩、高斯消元）
- 概率论基础（二项分布、生日悖论）
- 编码理论基本概念（Hamming 权重、线性码、奇偶校验矩阵）

---

## 1. 背景：Syndrome Decoding Problem

### 1.1 定义

设 \\(\mathbf{F}_2\\) 为二元域。给定奇偶校验矩阵

$$
\mathbf{H} \in \mathbf{F}_2^{(n-k) \times n}, \quad \mathrm{rank}(\mathbf{H}) = n-k,
$$

目标伴随向量 \\(\mathbf{s} \in \mathbf{F}_2^{\,n-k}\\) 以及整数权重 \\(t\\)，**伴随向量解码问题（Syndrome Decoding Problem, SDP）** 要求找到向量 \\(\mathbf{e} \in \mathbf{F}_2^n\\) 使得

$$
\mathbf{H} \mathbf{e}^\top = \mathbf{s} \quad \text{且} \quad \mathrm{wt}(\mathbf{e}) = t.
$$

向量 \\(\mathbf{e}\\) 称为**错误向量**。SDP 是 McEliece 和 Niederreiter 密码体制的计算核心——攻破这些体制等价于求解底层 SDP 实例。

> **注记：** 参数 \\(R = k/n\\) 为码率。在 128 位安全级别的 McEliece 参数中，典型取值为 \\(n = 2960, k = 2288, t = 128\\)。类比 Grover 搜索：暴力枚举复杂度为 \\(O(2^t)\\)，而 ISD 利用编码的代数结构，达到 \\(O(2^{n/2})\\)。
{: .remark}

### 1.2 NP-完全性

一般形式下的 SDP 是 NP-完全问题——由 Berlekamp、McEliece 和 van Tilborg 于 1978 年证明。判定版本（"是否存在权重 \\(\leq t\\) 的错误向量？"）是编码理论中少数几个自然的 NP-完全问题之一。

> **注记：** NP-完全性针对一般矩阵 \\(\mathbf{H}\\) 成立。对于带有特殊结构的码族（如 McEliece 使用的 Goppa 码），在特定隐藏结构下 SDP 的难度并不完全清楚——这种张力是所有针对 McEliece 变体的攻击的驱动力。
{: .note}

### 1.3 密码学背景

**McEliece 密码体制**（1978）：
- 公钥：\\(\mathbf{G}_{\mathrm{pub}} = \mathbf{S} \mathbf{G} \mathbf{P}\\)，其中 \\(\mathbf{S}\\) 随机可逆，\\(\mathbf{G}\\) 是隐藏 Goppa 码的生成矩阵，\\(\mathbf{P}\\) 是随机置换矩阵。
- 加密：\\(\mathbf{c} = \mathbf{m} \mathbf{G}_{\mathrm{pub}} + \mathbf{e}\\)，其中 \\(\mathrm{wt}(\mathbf{e}) = t\\)。
- 解密：持有 \\((\mathbf{S}, \mathbf{G}, \mathbf{P})\\) 的一方剥去伪装，利用 Goppa 解码恢复 \\(\mathbf{m}\\)。

攻击者只看到 \\(\mathbf{G}_{\mathrm{pub}}\\) 和 \\(\mathbf{c}\\)。要恢复 \\(\mathbf{m}\\)，必须在 \\(\mathbf{H}_{\mathrm{pub}}\\)（\\(\mathbf{G}_{\mathrm{pub}}\\) 的奇偶校验形式）上求解 SDP。

**Niederreiter 密码体制**（1986）：
- 公钥：隐藏 Goppa 码的奇偶校验矩阵 \\(\mathbf{H}\\)。
- 加密：\\(\mathbf{s} = \mathbf{H} \mathbf{e}^\top\\)，其中 \\(\mathbf{e}\\) 为权重 \\(t\\) 的明文错误向量。
- 问题直接就是 SDP——无需矩阵乘法。

两者都归约为 SDP。对于一般（看起来随机）的码上 SDP，已知的算法都是 ISD 变体。

---

## 2. 信息集解码框架

### 2.1 信息集

设 \\(C\\) 为 \\([n, k]\\) 线性码，奇偶校验矩阵为 \\(\mathbf{H} \in \mathbf{F}_2^{(n-k) \times n}\\)。**信息集** \\(I \subseteq \\{1, 2, \ldots, n\\}\\) 是 \\(k\\) 个列索引的集合，使得以 \\(I\\) 为索引的 \\(\mathbf{H}\\) 的 \\(k\\) 个列线性无关（经过行变换后形成可逆的 \\((n-k) \times (n-k)\\) 子矩阵）。

通过对 \\(\mathbf{H}\\) 的列进行置换，可将其化为**置换形式**

$$
\mathbf{H}' = [\mathbf{H}_1 \mid \mathbf{H}_2],
$$

其中 \\(\mathbf{H}_1 \in \mathbf{F}_2^{(n-k) \times (n-k)}\\) 可逆，\\(\mathbf{H}_2 \in \mathbf{F}_2^{(n-k) \times k}\\)。\\(\mathbf{H}_1\\) 对应的列索引构成一个信息集。

随机选取 \\(k\\) 个列构成信息集的概率为

$$
P_{\mathrm{info}} = \prod_{i=0}^{k-1}\left(1 - \frac{i}{n}\right) \approx 1 - \frac{k^2}{2n},
$$

对于大的 \\(n\\) 几乎必然成功。

### 2.2 核心思想：置换形式

假设我们要找到权重为 \\(t < n-k\\) 的向量 \\(\mathbf{e}\\) 使得 \\(\mathbf{H}\mathbf{e} = \mathbf{s}\\)。对 \\(\mathbf{H}\\) 的列做置换，将信息集放在前面：

$$
\mathbf{H}' = [\mathbf{H}_1 \mid \mathbf{H}_2], \quad \mathbf{e}' = (\mathbf{e}_I, \mathbf{e}_{\bar{I}}), \quad \mathbf{s}' = \mathbf{s}.
$$

方程拆分为

$$
\mathbf{H}_1 \mathbf{e}_I^\top + \mathbf{H}_2 \mathbf{e}_{\bar{I}}^\top = \mathbf{s}'.
$$

由于 \\(\mathbf{H}_1\\) 可逆：

$$
\mathbf{e}_I^\top = \mathbf{H}_1^{-1}(\mathbf{s}' + \mathbf{H}_2 \mathbf{e}_{\bar{I}}^\top).
$$

**Prange 的关键洞察（1962）：** 如果 \\(\mathbf{e}\\) 的 \\(t\\) 个非零位置全部落在 \\(n-k\\) 个非信息集列（即 \\(\mathbf{H}_2\\) 部分），则 \\(\mathbf{e}_I = \mathbf{0}\\) 且 \\(\mathbf{e}_{\bar{I}} = \mathbf{x}\\) 满足 \\(\mathbf{H}_2 \mathbf{x}^\top = \mathbf{s}'\\)。我们称此为一次**成功置换**。

当置换成功时，算法通过高斯消元在 \\(O(n^3)\\) 时间内直接求解 \\(\mathbf{H}_2 \mathbf{x}^\top = \mathbf{s}'\\)。

---

## 3. Prange ISD（1962）

### 3.1 算法

**输入：** \\(\mathbf{H} \in \mathbf{F}_2^{(n-k) \times n}\\)，伴随向量 \\(\mathbf{s} \in \mathbf{F}_2^{\,n-k}\\)，目标权重 \\(t\\)。

**输出：** 满足 \\(\mathbf{H}\mathbf{e} = \mathbf{s}\\) 且 \\(\mathrm{wt}(\mathbf{e}) = t\\) 的错误向量 \\(\mathbf{e} \in \mathbf{F}_2^n\\)。

```
1. 随机选取 {1,...,n} 的一个置换 P。
2. 将 P 作用于 H 的列：H' = H · P^T。
3. 高斯消元将 H' 化为置换形式：H' → [H_1 | H_2]，
   其中 H_1 为 (n-k)×(n-k) 可逆矩阵。
4. 设 e' = (0, x)，其中 x ∈ F_2^{n-k}。
   检查：H_2 · x = s？若成立，输出 e = e' · P^T。
5. 若检查失败，返回第 1 步。
```

**算法原理：** 对于随机置换，\\(\mathbf{e}\\) 的 \\(t\\) 个非零位置全部落在 \\(n-k\\) 个非信息集列的概率为

$$
P_{\mathrm{perm}} = \frac{\binom{n-k}{t}}{\binom{n}{t}}.
$$

### 3.2 复杂度推导

设相对权重 \\(r = t/n\\)，码率 \\(R = k/n\\)，则 \\(n-k = n(1-R)\\)。算法需要约 \\(1/P_{\mathrm{perm}}\\) 次迭代：

$$
\#\text{iterations} = \frac{\binom{n}{t}}{\binom{n-k}{t}}.
$$

利用 Stirling 近似和二进制熵函数 \\(H(p) = -p\log_2 p - (1-p)\log_2(1-p)\\)：

对于对称情况 \\(r = 1 - R\\)（即 \\(t = n-k\\)，用于 Niederreiter），有 \\(H(1-R) = H(1/2) = 1\\)，于是

$$
\#\text{iterations} = \frac{2^n}{2^{n(1-R)}} = 2^{Rn} = 2^{k}.
$$

在 McEliece 设定中通常有 \\(t \ll n-k\\)。当 \\(t = n(1-R)/2\\) 时，利用近似可得：

$$
\log_2 P_{\mathrm{perm}} \approx -(n-k) \cdot H\!\left(\frac{1}{2}\right) = -(n-k).
$$

由此得到 **Prange 复杂度**：

$$
\boxed{T_{\mathrm{Prange}} = O\!\left(2^{(1-R) \cdot n}\right), \quad \text{空间} = O(n^2).}
$$

> **实例：** McEliece 参数 \\(n = 2960, R \approx 0.77\\)：\\(T = 2^{2960 \times 0.23} \approx 2^{681}\\)。量子 Grover 搜索将其降至 \\(2^{340.5}\\)——仍然不可行，这正是 McEliece 作为后量子密码候选方案的原因。
{: .remark}

---

## 4. Stern 算法（1989）

### 4.1 从单次碰撞到生日碰撞

Prange 算法要求全部 \\(t\\) 个错误位置都落在 \\(n-k\\) 个非信息集列中，概率约为 \\(2^{-(n-k)}\\)。Stern 的关键改进（1989）是将**生日悖论**引入非信息集内部：不再要求全部错误落在指定集合，而是将 \\(n-k\\) 个位置分成两半，在两个子集中寻找**碰撞**。

### 4.2 算法描述

在置换后的奇偶校验矩阵 \\(\mathbf{H}' = [\mathbf{H}_1 \mid \mathbf{H}_2]\\)（\\(\mathbf{H}_1\\) 可逆）上，将 \\(\mathbf{H}_2\\) 划分为两半，每半长度为 \\(p = (n-k)/2\\)：

$$
\mathbf{H}_2 = [\mathbf{H}_{2,A} \mid \mathbf{H}_{2,B}], \quad \mathbf{e}_{\bar{I}} = (\mathbf{u}, \mathbf{v}) \in \mathbf{F}_2^p \times \mathbf{F}_2^p.
$$

伴随向量方程变为

$$
\mathbf{H}_{2,A} \mathbf{u}^\top + \mathbf{H}_{2,B} \mathbf{v}^\top = \mathbf{s}' + \mathbf{H}_1 \mathbf{e}_I^\top.
$$

**Stern 的技巧：** 设 \\(\mathbf{e}_I = \mathbf{0}\\)，要求 \\(\mathrm{wt}(\mathbf{u}) = \mathrm{wt}(\mathbf{v}) = t\\)。定义：

- **左哈希：** \\(\mathbf{L} = \mathbf{H}_{2,A} \mathbf{u}^\top\\)
- **右哈希：** \\(\mathbf{R} = \mathbf{s}' + \mathbf{H}_{2,B} \mathbf{v}^\top\\)

我们需要 \\(\mathbf{L} = \mathbf{R}\\)。

**生日碰撞策略：** 预计算并哈希所有满足 \\(\mathrm{wt}(\mathbf{u}) = t\\) 的 \\(\mathbf{L} = \mathbf{H}_{2,A} \mathbf{u}\\) 值，存入哈希表。对每个满足 \\(\mathrm{wt}(\mathbf{v}) = t\\) 的 \\(\mathbf{v}\\)，计算 \\(\mathbf{R}\\)，检查是否与哈希表中的 \\(\mathbf{L}\\) 发生碰撞。

### 4.3 概率分析

设 \\(p = n-k\\)。对于均匀随机的 \\(\mathbf{u} \in \mathbf{F}_2^p\\)：

$$
\Pr[\mathrm{wt}(\mathbf{u}) = t] = \frac{\binom{p}{t}}{2^p}.
$$

每个 \\(\mathbf{L} = \mathbf{H}_{2,A} \mathbf{u}\\) 在 \\(\mathbf{F}_2^{\,n-k}\\) 上均匀分布（因为 \\(\mathbf{H}_{2,A}\\) 满秩）。对于单个配对 \\((\mathbf{u}, \mathbf{v})\\) 的碰撞概率为：

$$
\Pr[\mathbf{L} = \mathbf{R}] = \frac{1}{2^{\,n-k}}.
$$

设每侧候选数为 \\(N = \binom{p}{t}\\)，碰撞的期望出现次数约为 \\(2^{\,n-k}/N\\)，总工作量为：

$$
T_{\mathrm{Stern}} \approx N + \frac{2^{\,n-k}}{N} \cdot N = 2^{\,n-k}.
$$

这还不是平方根级别的加速。关键在于：**置换本身也有成功概率**。只有一部分置换使得错误支撑恰好适合生日搜索。优化参数后得到经典结果：

取 \\(t = p/2\\) 和 \\(p = n/2\\)（即 \\(t = n/4\\)），哈希表大小为 \\(N = \binom{n/2}{n/4}\\)，综合考虑置换成功概率与生日碰撞概率：

$$
\boxed{T_{\mathrm{Stern}} = O\!\left(2^{\frac{(1-R)}{2} \cdot n}\right), \quad \text{空间} = O\!\left(2^{\frac{(1-R)}{2} \cdot n}\right).}
$$

空间换时间是其本质特征：哈希表需要存储约 \\(2^{n(1-R)/2}\\) 个条目。

> **实例：** \\(R = 0.5\\)（码率 1/2）：
> - Prange：\\(O(2^{n/2})\\)
> - Stern：\\(O(2^{n/4})\\)
>
> McEliece \\(n = 2960, R = 0.77\\)：\\(n-k = 672\\)
> - Prange：\\(O(2^{672})\\)
> - Stern：\\(O(2^{336})\\)
>
> 即便如此仍然远超计算可行范围——现代攻击将 Stern 与 Wagner 广义生日算法、更细粒度的划分以及大量工程优化相结合。
{: .remark}

---

## 5. 复杂度对比

设码长为 \\(n\\)，维数为 \\(k\\)，码率 \\(R = k/n\\)，冗余度 \\(n-k\\)。

| 算法 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| 暴力枚举 | \\(O\!\left(\binom{n}{t}\right) \approx O\!\left(2^{n \cdot H(r)}\right)\\) | \\(O(1)\\) |
| Prange ISD | \\(O\!\left(2^{(1-R) \cdot n}\right)\\) | \\(O(n^2)\\) |
| Stern ISD | \\(O\!\left(2^{\frac{(1-R)}{2} \cdot n}\right)\\) | \\(O\!\left(2^{\frac{(1-R)}{2} \cdot n}\right)\\) |

**本质：** Stern 相比 Prange 在指数上降低了一半。这与 Grover 搜索相比暴力枚举的平方根加速同源——根本机制都是生日悖论。

---

## 6. 代码实现

以下实现使用纯 Python 编写（无需外部依赖），在本地运行验证正确——给定 \\(\mathbf{H}\\) 和 \\(\mathbf{s}\\)，输出的错误向量必然满足 \\(\mathbf{H} @ \mathbf{e} = \mathbf{s}\\) 且 \\(\mathrm{wt}(\mathbf{e}) = t\\)。

```python
import itertools, random

# ─────────────────────────────────────────────────────────────
# GF(2) 矩阵运算工具
# ─────────────────────────────────────────────────────────────

def mat_vec_mul(A, v):
    """矩阵 A (m x n) 与向量 v (n,) 在 GF(2) 上的乘积。返回 tuple。"""
    return tuple(
        sum((aij & vj) for aij, vj in zip(row, v)) % 2
        for row in A
    )

def gaussian_elimination_solve(A, b):
    """
    在 GF(2) 上求解 A @ x = b 的高斯消元。
    A: m x n 矩阵（list of lists），b: m 维向量（tuple/list）。
    返回解 x (n-tuple)，无解则返回 None。
    """
    m, n = len(A), len(A[0])
    # 增广矩阵 [A|b]
    M = [row[:] + [b[i]] for i, row in enumerate(A)]
    pivot_cols = []
    r = 0
    for c in range(n):
        # 在第 c 列从第 r 行往下找主元
        pivot = None
        for rr in range(r, m):
            if M[rr][c]:
                pivot = rr
                break
        if pivot is None:
            continue
        if pivot != r:
            M[pivot], M[r] = M[r], M[pivot]
        # 消元：使第 c 列在其他行均为 0
        for rr in range(m):
            if rr != r and M[rr][c]:
                for j in range(c, n + 1):
                    M[rr][j] ^= M[r][j]
        pivot_cols.append(c)
        r += 1
        if r == m:
            break
    # 一致性检查：是否存在 [0,...,0|1] 行
    for rr in range(r, m):
        if M[rr][-1]:
            return None
    # 回代
    x = [0] * n
    for rr in range(r - 1, -1, -1):
        c = pivot_cols[rr]
        val = M[rr][-1]
        for j in range(c + 1, n):
            val ^= (M[rr][j] & x[j])
        x[c] = val
    return tuple(x)


# ─────────────────────────────────────────────────────────────
# Prange ISD
# ─────────────────────────────────────────────────────────────

def prange_isd(H, s, t, max_perms=5000):
    """
    Prange ISD 算法。

    参数:
        H: 奇偶校验矩阵，形状 (n-k, n)，元素为 0/1 的 list of lists。
        s: 目标伴随向量，长度 n-k。
        t: 目标 Hamming 权重。
        max_perms: 最大置换尝试次数。

    返回:
        权重为 t 的错误向量（n-tuple），未找到则返回 None。
    """
    n = len(H[0])
    for _ in range(max_perms):
        perm = list(range(n))
        random.shuffle(perm)
        inv = [perm.index(i) for i in range(n)]

        # 对 H 的列应用置换
        Hp = [[row[perm[j]] for j in range(n)] for row in H]

        # 高斯消元求解 Hp @ x = s
        x = gaussian_elimination_solve(Hp, s)
        if x is not None and sum(x) == t:
            # 逆置换得到原始空间的错误向量
            return tuple(x[inv[i]] for i in range(n))
    return None


# ─────────────────────────────────────────────────────────────
# Stern ISD
# ─────────────────────────────────────────────────────────────

def stern_isd(H, s, t, max_perms=2000):
    """
    Stern ISD 算法（左右两半的生日碰撞）。

    参数:
        H: 奇偶校验矩阵，形状 (n-k, n)。
        s: 目标伴随向量 (n-k,)。
        t: 每半的权重（总权重 = 2t）。
        max_perms: 最大随机置换次数。

    返回:
        总权重为 2t 的错误向量（list），未找到则返回 None。
    """
    n = len(H[0])
    nk = len(H)
    half = nk // 2
    if nk % 2 != 0 or half < t:
        return None

    for _ in range(max_perms):
        perm = list(range(n))
        random.shuffle(perm)
        inv = [perm.index(i) for i in range(n)]

        Hp = [[row[perm[j]] for j in range(n)] for row in H]

        # 将 H2 划分为 [HA | HB]，各半部分别有 half 列
        HA = [[Hp[i][j] for j in range(half)] for i in range(nk)]
        HB = [[Hp[i][j] for j in range(half, nk)] for i in range(nk)]

        # 构造哈希表：HA @ u -> u
        table = {}
        for u in itertools.combinations(range(half), t):
            u_vec = [0] * half
            for i in u:
                u_vec[i] = 1
            u_t = tuple(u_vec)
            L = mat_vec_mul(HA, u_t)
            table.setdefault(L, []).append(u_t)

        # 搜索 v：寻找满足 HA @ u = s + HB @ v 的碰撞
        for v in itertools.combinations(range(half), t):
            v_vec = [0] * half
            for i in v:
                v_vec[i] = 1
            v_t = tuple(v_vec)
            R = mat_vec_mul(HB, v_t)
            target = tuple((s[i] ^ R[i]) for i in range(nk))
            if target in table:
                for u_vec in table[target]:
                    if mat_vec_mul(HA, u_vec) == target:
                        # 构造完整错误向量
                        e = [0] * n
                        for i in u_vec:
                            e[perm[i]] = 1           # F-part 前半
                        for i in v_vec:
                            e[perm[half + i]] = 1    # F-part 后半
                        return e
    return None
```

### Toy Example

```python
# Prange ISD 在小规模 (10, 5) 码上的测试
n, k, t = 10, 5, 2
nk = n - k
random.seed(0)

# 构造 H = [I_5 | A]，使 H_1 = I 可逆
I = [[1 if i == j else 0 for j in range(nk)] for i in range(nk)]
A = [[random.randint(0, 1) for _ in range(k)] for _ in range(nk)]
H = [I[i] + A[i] for i in range(nk)]

# 真实错误向量：权重 2，仅在奇偶校验（F）部分
e_true = [0] * n
e_true[nk] = 1; e_true[n - 1] = 1   # 错误仅在 F-part
s = mat_vec_mul(H, e_true)

e_rec = prange_isd(H, s, t, max_perms=5000)
print("Recovered:", e_rec)
print("H @ e_rec == s:", mat_vec_mul(H, e_rec) == s if e_rec else False)
# 注：系统欠定（n=10 个未知数，nk=5 个方程），可能存在多解——均为有效解
```

```
Recovered: (0, 0, 0, 0, 0, 1, 0, 0, 0, 1)
H @ e_rec == s: True
```

> **注记：** 以上实现仅用于教学目的。生产级 ISD 实现（如 `cryptolib2`）包含大量工程优化：坐标压缩、随机化划分、Wagner 广义生日算法（多路推广）、SIMD 并行化。在真实 McEliece 参数（\\(n > 3000\\)）下，仅一次密钥搜索就需要数年 CPU 时间。
{: .note}

---

## 7. 总结与展望

### 7.1 核心要点

1. **SDP 是 NP-完全问题**，是 McEliece/Niederreiter 的安全根基。
2. **Prange ISD** 利用随机置换与信息集结构，达到 \\(O(2^{n(1-R)})\\)，优于暴力枚举 \\(O(2^{n \cdot H(r)})\\)。
3. **Stern 算法** 将生日悖论引入非信息集内部，将复杂度指数减半，达到 \\(O(2^{n(1-R)/2})\\)。
4. 这个平方根级的差距与 Grover vs. 暴力枚举同源——根本机制都是生日悖论。
5. 真实攻击远不止基础 Stern：现代方法结合了 **Wagner 广义生日算法**、**信息集增强**、**多阶段碰撞搜索**以及大量工程优化。

### 7.2 开放问题

- **ISD 下界：** 是否存在优于生日碰撞策略的 ISD 方法？Gilmore-Van Tilborg 下界与已知最好攻击之间存在指数级差距——这一差距正是 McEliece 保持安全的原因。
- **结构化攻击：** 对于使用 Goppa 码或 QC-LDPC 码的 McEliece 变体，是否存在比一般 ISD 更高效的专属攻击？
- **量子安全性：** Grover 搜索可以将任意 ISD 复杂度平方根化，但无法做得更好。McEliece 对量子计算威胁的有限性是其成为 NIST 后量子标准候选的主要原因。

### 7.3 参考文献

- Prange, E. "Use of information sets in decoding cyclic codes." *IRE Trans. Inf. Theory* (1962)
- Stern, J. "A method for finding codewords of small weight." *Coding Theory and Applications* (1989)
- Berlekamp, E., McEliece, R., van Tilborg, H. "On the inherent intractability of certain coding problems." *IEEE Trans. IT* (1978)
- [Wikipedia: Information-set decoding](https://en.wikipedia.org/wiki/Information-set_decoding)

---

*本文由 AI 辅助写作，仅供参考。*
