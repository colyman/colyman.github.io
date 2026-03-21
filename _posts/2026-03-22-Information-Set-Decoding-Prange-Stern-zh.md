---
layout: article
title: "Information Set Decoding：Prange 与 Stern 算法"
date: 2026-03-22
key: Information-Set-Decoding-Prange-Stern-2026-zh
lang: zh
tags: [Code-Based-Cryptography, Information-Set-Decoding, McEliece, Stern-Algorithm, Post-Quantum-Cryptography]
hidden: true
---

**tl;dr:** Information Set Decoding（ISD）是破解 McEliece、Niederreiter 等编码密码体制的核心算法。本文详解 Prange ISD（1962）和 Stern 算法（1989）的数学原理，推导 Prange 复杂度 \\(O(2^{n\\cdot H(r)})\\) 与 Stern 复杂度 \\(O(2^{n/2})\\)，并给出 SageMath 实现代码。

---

## 大纲

### 内容概述
1. **背景与动机** — SDP 问题定义，NP-完全性，McEliece/Niederreiter 密码体制
2. **ISD 框架** — 信息集概念，置换形式，核心思想；与随机码的区别
3. **Prange ISD（1962）** — 置换、高斯消元、复杂度 \\(O(2^{n\\cdot H(r)})\\) 推导
4. **Stern 算法（1989）** — 生日悖论、分区策略、碰撞概率推导（\\(t=n/4\\) 参数）
5. **复杂度对比** — Prange vs Stern vs 暴力枚举（表格 + 分析）
6. **代码实现** — SageMath 实现 Prange 和 Stern 算法
7. **总结与展望**

### 参考文献
- Prange. "Use of codes that center on weight enumerators for secure coding" (1962)
- Stern. "A method for finding codewords of small weight" (1989)
- [Wikipedia: Information-set decoding](https://en.wikipedia.org/wiki/Information-set_decoding)
- [Tanglee's Blog](https://blog.tanglee.top)

### 前置知识
- 线性代数（有限域、矩阵秩、高斯消元）
- 概率论基础（二项分布、生日悖论）
- 编码理论基本概念（GRS码、Hamming权重）
- 密码学背景：McEliece 公钥密码体制

---

## 1. 背景：Syndrome Decoding Problem

### 1.1 问题定义

设 \\(\mathbb{F}_2\\) 为二元域。给定校验矩阵

$$
H \in \mathbb{F}_2^{(n-k) \times n}, \quad \mathrm{rank}(H) = n-k,
$$

目标伴随向量 \\(s \in \mathbb{F}_2^{n-k}\\) 和整数权重 \\(t\\)，**Syndrome Decoding Problem（SDP）** 要求找到向量 \\(e \in \mathbb{F}_2^n\\) 使得

$$
He = s \quad \text{且} \quad \mathrm{wt}(e) = t.
$$

向量 \\(e\\) 称为**错误向量**（error vector）或**错误图样**（error pattern）。SDP 是 McEliece 和 Niederreiter 密码体制的计算核心——破解这两个方案本质上就是求解 SDP。

> **注记（Remark）：** 参数 \\(R = k/n\\) 为码率，\\(r = t/n\\) 为相对权重。McEliece 在 128-bit 安全级别下的典型参数为 \\(n = 2960, k = 2288, t = 128\\)。这与 Grover 搜索的类比：暴力枚举复杂度 \\(O(2^t)\\) 对比 ISD 的 \\(O(2^{n/2})\\)，差距来自ISD利用了码的代数结构。
{: .remark}

### 1.2 NP-完全性

SDP 在 \\(\mathbb{F}_2\\) 上是 NP-完全的——由 Berlekamp、McEliece 和 van Tilborg 于 1978 年证明。判定版本（"是否存在权重 \\(\leq t\\) 的解？"）是编码理论中少数几个自然的 NP-完全问题之一。

> **注记（Note）：** NP-完全性是对一般 \\(H\\) 而言。对于具有特殊结构（如 McEliece 使用的 Goppa 码）的矩阵，SDP 的计算难度不明确——这正是所有针对 McEliece 变体的攻击的切入点。
{: .note}

### 1.3 密码学背景

**McEliece 密码体制**（1978）：
- 公钥：\\(G_{\text{pub}} = SGP\\)，其中 \\(S\\) 为随机可逆矩阵，\\(G\\) 为隐藏 Goppa 码的生成矩阵，\\(P\\) 为随机置换矩阵。
- 加密：\\(c = mG_{\text{pub}} + e\\)，其中 \\(e\\) 为随机权重 \\(t\\) 的错误向量。
- 解密：持有私钥 \\((S, G, P)\\) 的人剥离掩码，用 Goppa 解码恢复 \\(m\\)。

攻击者只能看到 \\(G_{\text{pub}}\\) 和 \\(c\\)。要恢复 \\(m\\)，须在 \\(H_{\text{pub}}\\)（\\(G_{\text{pub}}\\) 的校验矩阵形式）上求解 SDP。

**Niederreiter 密码体制**（1986）：
- 公钥：隐藏 Goppa 码的校验矩阵 \\(H\\)。
- 加密：\\(s = He^T\\)（明文为权重 \\(t\\) 的错误向量 \\(e\\)）。
- 问题直接就是 SDP，无需矩阵乘法。

两种体制均归约为 SDP。对一般（看起来随机的）码求 SDP，唯一已知算法均为 ISD 变体。

---

## 2. Information Set Decoding：核心框架

### 2.1 信息集

设 \\(C\\) 为参数 \\([n, k]\\) 的线性码，校验矩阵 \\(H \in \mathbb{F}_2^{(n-k) \times n}\\)。**信息集** \\(I \subseteq \\{1, 2, \\ldots, n\\}\\) 是 \\(k\\) 个列索引的集合，使得由 \\(I\\) 索引的 \\(H\\) 的 \\(k\\) 列线性无关（即在适当行化后形成一个可逆的 \\((n-k) \\times k\\) 子矩阵）。

换句话说，对 \\(H\\) 的列做置换后得到

$$
H \cdot P = [H_1 \mid H_2],
$$

其中 \\(H_1 \in \mathbb{F}_2^{(n-k) \times k}\\) 可逆，\\(H_2 \in \mathbb{F}_2^{(n-k) \times (n-k)}\\)。对应 \\(H_1\\) 的列索引集合即为一个信息集。

随机选取 \\(k\\) 个列作为信息集的成功概率为

$$
P_{\text{info}} = \prod_{i=0}^{k-1}\left(1 - \frac{i}{n}\right) = \frac{\binom{n-k}{0}\binom{n-k}{1}\cdots\binom{n-k}{k-1}}{\binom{n}{k}} \approx 1 - \frac{k^2}{2n},
$$

对大 \\(n\\) 几乎必然成功。

### 2.2 核心洞察：置换形式

设错误向量 \\(e\\) 权重为 \\(t\\)，满足 \\(He = s\\)。对 \\(H\\) 的列做置换，将信息集 \\(I\\) 置于前部：

$$
H' = [H_1 \mid H_2], \quad e' = (e_I, e_{\bar{I}}), \quad s' = s.
$$

方程分块为

$$
H_1 \cdot e_I + H_2 \cdot e_{\bar{I}} = s.
$$

由于 \\(H_1\\) 可逆，改写为

$$
e_I = H_1^{-1}(s + H_2 \cdot e_{\bar{I}}).
$$

**Prange 的核心观察（1962）：** 若 \\(e\\) 的 \\(t\\) 个非零位置恰好全部落在 \\(n-k\\) 个非信息集列（即 \\(H_2\\) 对应的列）中，则 \\(e_I = 0\\)，\\(e_{\bar{I}} = x\\) 满足 \\(H_2x = s\\)。我们称这种情况为**置换成功**。

若置换成功，算法直接通过高斯消元解线性方程组 \\(H_2x = s\\) 即可找到 \\(e\\)，复杂度 \\(O((n-k)^3/\\omega) = O(n^3)\\)。

### 2.3 与随机码的关系

对于真正随机的（均匀分布的）矩阵 \\(H\\) 和随机权重 \\(t\\) 的 \\(e\\)，伴随向量 \\(s = He\\) 在 \\(\mathbb{F}_2^{n-k}\\) 中均匀分布。这意味着在所有 \\(\binom{n}{t}\\) 个权重 \\(t\\) 向量中，恰有一个（若存在）满足 \\(He = s\\)。挑战在于：给定 \\(H\\) 和 \\(s\\)，如何在不枚举全部可能性的情况下找到那个唯一解。

---

## 3. Prange ISD（1962）

### 3.1 算法描述

**输入：** 校验矩阵 \\(H \in \mathbb{F}_2^{(n-k) \\times n}\\)，目标伴随向量 \\(s \in \mathbb{F}_2^{n-k}\\)，目标权重 \\(t\\)。

**输出：** 满足 \\(\mathrm{wt}(e) = t\\) 且 \\(He = s\\) 的错误向量 \\(e \in \mathbb{F}_2^n\\)。

```
1. 随机选取 {1,...,n} 的一个置换 P。
2. 对 H 的列施加置换 P，得 H' = H·P。
3. 对 H' 做高斯消元，化为置换形式 [H_1 | H_2]，
   其中 H_1 为 (n-k)×(n-k) 可逆矩阵。
   记录实现此形式的列置换。
4. 对伴随向量施加相同置换：s' = s·P。
5. 设 e' = (0_{n-k}, x)，其中 x ∈ 𝔽₂^{n-k} 为候选错误
   （落在"非信息集"位置的错误部分）。
   检验：H_2·x = s'？若成立，输出 e = e'·P^{-1}。
6. 若检验失败，返回第 1 步。
```

**为何有效：** 若 \\(e\\) 是解，施加一个将 \\(e\\) 的支撑集（support）完全放入 \\(n-k\\) 个非信息集列的置换后，有 \\(e_I = 0\\) 且 \\(e_{\bar{I}} = x\\) 满足 \\(H_2x = s\\)。对任意随机置换，\\(e\\) 的 \\(t\\) 个非零位置恰好全部落在 \\(n-k\\) 个非信息集列的概率为

$$
P_{\text{perm}} = \frac{\binom{n-k}{t}}{\binom{n}{t}} \approx 2^{-t \\cdot H_2},
$$

其中 \\(H_2 = H\\left(\\frac{t}{n-k}\\right)\\) 为二元熵函数在 \\(r' = t/(n-k)\\) 处的值。当此事件发生时，算法通过高斯消元直接解出 \\(x\\)，耗时 \\(O(n^3)\\)。

### 3.2 复杂度分析

设 \\(r = t/n\\) 为相对权重，\\(R = k/n\\) 为码率，则 \\(n-k = n(1-R)\\)。算法大约需要 \\(1/P_{\\text{perm}}\\) 次迭代：

$$
\#\text{iterations} = \frac{\binom{n}{t}}{\binom{n-k}{t}}.
$$

利用 Stirling 近似和熵的近似 \\(\binom{n}{t} \\approx 2^{n \\cdot H(r)}\\)：

$$
\#\text{iterations} = \frac{2^{n \\cdot H(r)}}{2^{n(1-R) \\cdot H\\left(\\frac{r}{1-R}\\right)}} = 2^{n \\cdot \\left[H(r) - (1-R)H\\left(\\frac{r}{1-R}\\right)\\right]} = 2^{n \\cdot H(R, r)},
$$

其中 \\(H(R, r)\\) 为**信息集解码指数**。利用熵函数的恒等式可简化为

$$
H(R, r) = H(r) - (1-R)H\\left(\\frac{r}{1-R}\\right).
$$

在对称情况 \\(r = 1-R\\)（即 \\(t = n-k\\)，常用于 Niederreiter）中，\\(H\\left(\\frac{1}{2}\\right) = 1\\)，得到 \\(H(R, 1-R) = 1 - R\\)。这就是 **Prange 指数**：时间复杂度为

$$
T_{\text{Prange}} = O\\left(2^{n(1-R)}\\right), \quad \text{空间} = O(n^2).
$$

> **例子：** McEliece 参数 \\(n=2960, R \\approx 0.77\\)：
> - \\(T = 2^{2960 \\times 0.23} \\approx 2^{681}\\)，对应 NIST Level 1 安全级别。
> - 若用 Grover 算法量子攻击，复杂度降为 \\(2^{340.5}\\)——但这仍远超量子计算机的可行范围。
{: .remark}

---

## 4. Stern 算法（1989）

### 4.1 生日悖论的引入

Prange 算法的本质是问："是否存在一个置换，使得错误支撑完全落在 \\(n-k\\) 个非信息集位置？"这要求 \\(t\\) 个位置全部落入特定集合，概率 \\(\\binom{n-k}{t}/\\binom{n}{t}\\)。

Stern 的关键洞察（1989）是在非信息集位置内部应用**生日悖论**：不要求整个支撑落在特定集合，而是对 \\(n-k\\) 个位置做分区，在其中一个分区中寻找**碰撞**。

### 4.2 算法描述

设置换后的校验矩阵为 \\(H' = [H_1 \\mid H_2]\\)，\\(H_1\\) 可逆。将 \\(H_2\\) 的列分成两半，每半大小为 \\(p = (n-k)/2\\)：

$$
H_2 = [H_{2,A} \mid H_{2,B}], \quad e_{\bar{I}} = (u, v) \in \mathbb{F}_2^p \\times \\mathbb{F}_2^p,
$$

其中 \\(u\\) 和 \\(v\\) 分别是两半中的错误部分。伴随向量方程变为

$$
H_2 \\cdot e_{\bar{I}} = H_{2,A} u + H_{2,B} v = s' + H_1 \\cdot e_I.
$$

**Stern 的技巧：** 设 \\(e_I = 0\\)，并要求 \\(\\mathrm{wt}(u) = \\mathrm{wt}(v) = p/2\\)。定义：

- **左侧** \\(L = H_{2,A} u\\)
- **右侧** \\(R = s' + H_{2,B} v\\)

需要找到 \\(L = R\\)。

**生日碰撞策略：** 预计算所有 \\(u\\)（权重 \\(p/2\\)）对应的 \\(L\\) 值并存入哈希表。对每个 \\(v\\)（权重 \\(p/2\\)），计算 \\(R\\)，检查是否与哈希表中的 \\(L\\) 碰撞。

### 4.3 概率推导

设 \\(p = n-k\\)。对随机 \\(u \\in \\mathbb{F}_2^p\\)，恰有权重 \\(p/2\\) 的概率为

$$
P_1 = \\frac{\\binom{p}{p/2}}{2^p} \\approx \\frac{1}{\\sqrt{\\pi p/2}} = \\sqrt{\\frac{2}{\\pi p}}.
$$

（因为 \\(\\binom{p}{p/2} \\approx 2^p / \\sqrt{\\pi p/2}\\)。）同理对 \\(v\\)：\\(P_2 = P_1\\)。

**标准 Stern 分析：** 设信息集 \\(I\\) 大小为 \\(k\\)，补集 \\(\\bar{I}\\)（大小 \\(n-k\\)）分成两集合 \\(A\\) 和 \\(B\\)，各大小 \\(p = (n-k)/2\\)。错误向量 \\(e\\) 权重为 \\(t\\)，在 \\(A\\) 和 \\(B\\) 中各有权重 \\(t_A = t_B = t/2\\)，在 \\(I\\) 中权重 \\(t_I = 0\\)。

伴随向量方程：\\(H_A u + H_B v = s\\)。

对随机 \\(u\\)（权重 \\(p/2\\)），\\(H_A u\\) 在 \\(\mathbb{F}_2^{n-k}\\) 中均匀分布。因此，对随机 \\(v\\)（权重 \\(p/2\\)），\\(H_B v + s\\) 也是均匀分布，与 \\(H_A u\\) 碰撞的概率为 \\(1/2^{n-k}\\)。

需要尝试的 \\((u, v)\\) 对数量（成功期望）的倒数为碰撞概率：

$$
T_{\\text{Stern}} \\approx \\frac{2^{n-k}}{\\binom{p}{p/2}^2 / 2^p} = 2^{n-k} \\cdot \\frac{2^p}{\\binom{p}{p/2}^2}.
$$

将 \\(\\binom{p}{p/2} \\approx 2^p \\sqrt{2/(\\pi p)}\\) 代入：

$$
T_{\\text{Stern}} \\approx 2^{n-k} \\cdot \\frac{2^p}{2^{2p} \\cdot \\frac{2}{\\pi p}} = 2^{n-k-p} \\cdot \\frac{\\pi p}{2} = 2^{(n-k)/2} \\cdot O(n).
$$

即

$$
T_{\\text{Stern}} = O\\left(2^{\\frac{n(1-R)}{2}} \\cdot n\\right) \\approx O\\left(2^{\\frac{n(1-R)}{2}}\\right).
$$

**这就是 Stern 的核心结果：** 复杂度为 Prange 的平方根——由生日悖论驱动，复杂度指数减半。

> **例子：** \\(R = 0.5\\)（码率 1/2）：
> - Prange：\\(O(2^{n/2})\\)
> - Stern：\\(O(2^{n/4})\\)
>
> McEliece 参数 \\(n=2960, R=0.77\\)：\\(n-k = 672\\)
> - Prange：\\(O(2^{672})\\)
> - Stern：\\(O(2^{336})\\)
>
> 这仍然远不可行——现代攻击将 Stern 与 Wagner 广义生日算法、更精细的分区以及大量工程优化结合。
{: .remark}

### 4.4 最优参数

Stern 论文中的最优参数为 \\(t = n/4\\)（每半权重），分区大小 \\(p = n/2\\)。每对 \\((u, v)\\) 的成功概率为

$$
P_{\\text{pair}} = \\frac{\\binom{n/2}{n/4}^2}{2^n} \\approx \\frac{1}{\\sqrt{\\pi n/4} \\cdot \\sqrt{\\pi n/4}} = \\frac{4}{\\pi n}.
$$

期望尝试次数 \\(1/P_{\\text{pair}} \\approx \\pi n / 4 = O(n)\\)。结合规模为 \\(2^{n/2}\\) 的哈希表，总体复杂度为 \\(O(2^{n/2})\\)。

---

## 5. 复杂度对比

设 \\(n\\) 为码长，\\(k\\) 为维数，\\(R = k/n\\)，\\(n-k\\) 为冗余度。

| 算法 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| 暴力枚举 | \\(O\\left(\\binom{n}{t}\\right) \\approx O\\left(2^{n \\cdot H(r)}\\right)\\) | \\(O(1)\\) |
| Prange ISD | \\(O\\left(2^{n(1-R)}\\right)\\) | \\(O(n^2)\\) |
| Stern ISD | \\(O\\left(2^{\\frac{n(1-R)}{2}}\\right)\\) | \\(O\\left(2^{\\frac{n(1-R)}{2}}\\right)\\) |

**解读：** Stern 的空间复杂度随时间复杂度缩放（大小约为 \\(2^{n(1-R)/2}\\) 的哈希表），而 Prange 仅需存储置换后的矩阵（\\(O(n^2)\\) 比特）。复杂度指数的改进因子为 2——这与 Grover 搜索相对于暴力搜索的平方根差距完全相同，根源都在于生日悖论的结构。

```
指数（以 2 为底）
      │
  0.5 │                         ┼ Stern (斜率 = (1-R)/2)
      │                        ╱
  0.375│                      ╱
      │                    ╱
  0.25 │                  ┼ Prange (斜率 = 1-R)
      │                ╱
      │              ╱
 0.125 │            ╱
      │          ╱
      │        ┼ Brute Force (斜率 = H(r))
      │
      └────────────────────────────────→ n
```

---

## 6. 代码实现

### 6.1 Prange ISD 的 SageMath 实现

```python
# Prange Information Set Decoding
# 在 GF(2) 上求解：给定 H, s，找到权重 t 的 e 使得 H @ e = s

def prange_isd(H, s, t, max_attempts=2^20):
    """
    Prange ISD 算法。
    
    参数:
        H: GF(2) 上的校验矩阵，形状 (n-k, n)
        s: 目标伴随向量，形状 (n-k,)
        t: 目标权重
        max_attempts: 最大迭代次数
        
    返回:
        e: 权重 t 的错误向量，觅到则返回 None
    """
    n = H.ncols()
    nk = H.nrows()
    
    for _ in range(max_attempts):
        # Step 1: 随机置换
        perm = Permutations(n).random_element()
        P = permutation_matrix(ZZ, perm)
        Hp = H * P.T
        
        # Step 2: 高斯消元，化为置换形式 [H_1 | H_2]
        # 寻找 (n-k) 个线性无关的列
        try:
            # 取前 nk 列作为候选信息集
            H_left = Hp[:, :nk]
            if H_left.rank() != nk:
                continue
            H_right = Hp[:, nk:]
        except:
            continue
        
        # Step 3: 对伴随向量施加相同置换
        s_perm = s * P.T
        
        # Step 4: 设 e' = (0, x)，解 H_right @ x = s_perm
        try:
            x = H_right.solve_left(s_perm)
            # x 恰好权重 t？
            if x.hamming_weight() == t:
                e_perm = vector(GF(2), [0]*nk + list(x))
                e = e_perm * P.T
                return e
        except ValueError:
            # 无解，继续迭代
            pass
    
    return None


# 示例：小型实例验证
if __name__ == "__main__":
    n, k, t = 16, 8, 2
    nk = n - k
    
    # 随机生成校验矩阵和错误向量
    set_random_seed(42)
    H = random_matrix(GF(2), nk, n)
    
    # 随机错误向量
    e_true = vector(GF(2), n)
    support = Subsets(range(n), t).random_element()
    for idx in support:
        e_true[idx] = GF(2)(1)
    
    s = H * e_true
    
    # 运行 Prange ISD
    e_recovered = prange_isd(H, s, t, max_attempts=10000)
    if e_recovered is not None:
        print("Found:", e_recovered)
        print("Match:", e_recovered == e_true)
    else:
        print("Not found within max_attempts")
```

### 6.2 Stern ISD 的 SageMath 实现

```python
# Stern Information Set Decoding
# 生日悖论加速，划分为两半

def stern_isd(H, s, t, max_perms=2^15):
    """
    Stern ISD 算法。
    
    参数:
        H: GF(2) 上的校验矩阵，形状 (n-k, n)
        s: 目标伴随向量，形状 (n-k,)
        t: 每半的目标权重（总权重 = 2t）
        max_perms: 最大随机置换次数
        
    返回:
        e: 错误向量（总权重 2t），觅到则返回 None
    """
    n = H.ncols()
    nk = H.nrows()
    p = nk  # 非信息集大小
    
    for _ in range(max_perms):
        # Step 1: 随机置换
        perm = Permutations(n).random_element()
        P = permutation_matrix(ZZ, perm)
        Hp = H * P.T
        
        # Step 2: 置换形式 [H_1 | H_2]
        try:
            H1 = Hp[:, :nk]
            if H1.rank() != nk:
                continue
            H2 = Hp[:, nk:]
        except:
            continue
        
        # Step 3: 伴随向量置换
        s_perm = s * P.T
        
        # Step 4: H2 划分为 [H_A | H_B]，各 p/2 列
        if nk % 2 != 0:
            continue
        half = nk // 2
        HA = H2[:, :half]
        HB = H2[:, half:]
        
        # Step 5: 枚举 u（权重 t），构建哈希表 L = HA @ u
        hash_table = {}
        for u_bits in Subsets(range(half), t):
            u = vector(GF(2), half)
            for idx in u_bits:
                u[idx] = GF(2)(1)
            L = HA * u
            # 按 L 的前 l 位（ 及 H wt(u) ）分组
            # 简化版本：用 L 本身做 key，存 u 的列表
            key = tuple(L)
            if key not in hash_table:
                hash_table[key] = []
            hash_table[key].append(u)
        
        # Step 6: 枚举 v（权重 t），检查碰撞
        for v_bits in Subsets(range(half), t):
            v = vector(GF(2), half)
            for idx in v_bits:
                v[idx] = GF(2)(1)
            R = s_perm + HB * v
            key = tuple(R)
            if key in hash_table:
                # 找到碰撞！需要进一步过滤
                for u in hash_table[key]:
                    # 验证：H_A @ u + H_B @ v = s_perm
                    if HA * u + HB * v == s_perm:
                        # 构建完整的 e_perm = (0, u, v)
                        e_perm = vector(GF(2), [0]*nk + list(u) + list(v))
                        e = e_perm * P.T
                        return e
    
    return None


# 示例：小型实例验证
if __name__ == "__main__":
    n, k, t = 20, 10, 2  # t = 每半权重，总权重 = 4
    nk = n - k
    
    set_random_seed(123)
    H = random_matrix(GF(2), nk, n)
    
    # 构造错误向量：权重 2t = 4，落在非信息集区域
    e_true = vector(GF(2), n)
    # 信息集 0,1,...,nk-1，之外为非信息集
    # 权重 t 在前半，t 在后半
    non_info = list(range(nk, n))
    half = nk // 2
    import random
    random.seed(42)
    part1 = random.sample(non_info[:half], t)
    part2 = random.sample(non_info[half:], t)
    for idx in part1 + part2:
        e_true[idx] = GF(2)(1)
    
    s = H * e_true
    
    e_recovered = stern_isd(H, s, t, max_perms=5000)
    if e_recovered is not None:
        print("Found:", e_recovered)
        print("True: ", e_true)
        print("Match:", e_recovered == e_true)
    else:
        print("Not found within max_perms")
```

> **注记（Note）：** 上述实现为教学用途。实际 ISD 实现（如 in the `cryptolib2` library）包含大量工程优化：坐标压缩、分区随机化、Wagner 算法的多路推广、以及 SIMD 并行化。在 \\(n > 3000\\) 的真实 McEliece 参数下，即使 Stern 算法也需要数年 CPU 时间才能完成一次完整的密钥搜索。
{: .note}

---

## 7. 总结与展望

### 7.1 核心 Takeaways

1. **SDP 是 NP-完全问题**，是 McEliece/Niederreiter 密码体制的安全基础。
2. **Prange ISD** 利用随机置换和信息集结构，复杂度 \\(O(2^{n(1-R)})\\)，优于纯暴力枚举的 \\(O(2^{n \\cdot H(r)})\\)。
3. **Stern 算法** 在非信息集内部引入生日悖论（碰撞搜索），将复杂度指数再减半：\\(O(2^{n(1-R)/2})\\)。
4. 这个平方根差距与 Grover 搜索相对于暴力搜索的差距完全同构，根源都是生日悖论。
5. 实际攻击远超纯 Stern——现代攻击将 Stern 与 **Wagner 广义生日算法**、**信息集增强**、**分阶段碰撞**等技术结合，进一步逼近理论极限。

### 7.2 开放问题

- **ISD 下界**：是否存在比生日碰撞更好的 ISD 策略？目前最好的下界来自 Coding Theory 的 Gilmore-Van Tilborg bound，与最好攻击之间存在指数级差距——这正是 McEliece 密码体制仍然安全的根本原因。
- **结构化攻击**：对于使用 Goppa 码或 QC-LDPC 码的 McEliece 变体，是否存在比通用 ISD 更高效的专用攻击？
- **量子攻击**：Grover 搜索可以将 ISD 的复杂度平方根化，但无法做得更好。量子计算对 McEliece 的威胁有限，这也是其被选为 NIST 后量子标准候选的重要原因之一。

### 7.3 参考文献

- Prange, E. "Use of codes that center on weight enumerators for secure coding." (1962)
- Stern, J. "A method for finding codewords of small weight." (1989)
- Berlekamp, E., McEliece, R., van Tilborg, H. "On the inherent intractability of certain coding problems." IEEE Trans. IT (1978)
- [Wikipedia: Information-set decoding](https://en.wikipedia.org/wiki/Information-set_decoding)
- [Tanglee's Blog](https://blog.tanglee.top)

---

*本文由 AI 辅助写作，仅供学习参考。*
