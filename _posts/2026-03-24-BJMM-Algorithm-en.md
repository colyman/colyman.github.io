---
layout: article
title: "BJMM Algorithm: How 1+1=0 Improves Information Set Decoding"
date: 2026-03-24
key: BJMM-Algorithm-2026
lang: en
tags: [Code-Based-Cryptography, Information-Set-Decoding, ISD, McEliece, BJMM, 2012]
bilingual: true
mathjax: true
---

**tl;dr**: BJMM (Becker-Joux-May-Meurer, Eurocrypt 2012) improves ISD complexity to $2^{0.04934n}$ by allowing overlapping index sets and exploiting $1+1=0$ in $\mathbf{F}_2$ to split both 0-entries and 1-entries, dramatically increasing the number of representations per solution. The algorithm is a three-layer merge-join tree similar to Wagner's generalized birthday problem.

<!-- more -->

## 1. Background: Information Set Decoding

Let $C$ be a random binary linear $[n, k, d]$ code with parity-check matrix $H \in \mathbf{F}_2^{(n-k)\times n}$. Given a syndrome $s = He$ where the error vector $e \in \mathbf{F}_2^n$ has Hamming weight $\mathrm{wt}(e) = \omega = \lfloor (d-1)/2 \rfloor$ (half-distance decoding), the **syndrome decoding problem** asks us to recover $e$ from $s$.

Prange's ISD framework (1962) proceeds in two steps:

1. **Initial transformation**: Permute columns of $H$ by a random $P \in \mathbf{F}_2^{n \times n}$, then apply Gaussian elimination via an invertible $T \in \mathbf{F}_2^{(n-k)\times(n-k)}$ to get a quasi-systematic form

$$\widetilde{H} = T \cdot H \cdot P = \begin{pmatrix} Q & I_{n-k} \end{pmatrix}$$

with $Q \in \mathbf{F}_2^{(n-k)\times k}$. The permutation $P$ randomizes the positions of the $\omega$ error ones. After transformation, the error $\tilde{e} = Pe$ has the form $\tilde{e} = (\tilde{e}_1 \mid \tilde{e}_2)$ where $\tilde{e}_1 \in \mathbf{F}_2^k$ contains $p$ ones and $\tilde{e}_2 \in \mathbf{F}_2^{n-k}$ contains $\omega - p$ ones, with probability

$$P = \frac{\binom{k}{p}\binom{n-k}{\omega-p}}{\binom{n}{\omega}}.$$

2. **Search phase**: Find the truncated error $\tilde{e}_1$ such that $Q\tilde{e}_1 = \tilde{s}$ on the first $n-k$ coordinates. This is the **Submatrix Matching Problem (SMP)**.

The best ISD algorithms before BJMM:

| Algorithm | Complexity exponent |
|-----------|-------------------|
| Lee-Brickell (1988) | $2^{0.05752n}$ |
| Stern (1989) | $2^{0.05564n}$ |
| Ball-collision (2011) | $2^{0.05559n}$ |
| MMT (2011) | $2^{0.05364n}$ |
| **BJMM (2012)** | **$2^{0.04934n}$** |

## 2. Submatrix Matching Problem (SMP)

Let $Q \in \mathbf{F}_2^{\ell \times (k+\ell)}$ be the submatrix of $\widetilde{H}$ formed by the first $\ell = n-k$ rows, and let $s \in \mathbf{F}_2^\ell$ be the target syndrome. The **Submatrix Matching Problem** is:

> Find a set $I \subseteq [k+\ell]$, $|I| = p$, such that
> $$\sigma(Q_I) := \sum_{i \in I} q_i = s,$$
> where $q_i$ denotes the $i$-th column of $Q$.

The SMP itself is a syndrome decoding instance with parameters $[k+\ell, \ell, p]$ — improving the SMP directly improves ISD.

## 3. MMT's Representation Technique

The May-Meurer-Thomae (MMT) algorithm (Asiacrypt 2011) first introduced representations to ISD. They decomposed the solution set $I$ as the union of two **disjoint** index sets $I = I_1 \cup I_2$ with $|I_1| = |I_2| = p/2$ and $I_1 \cap I_2 = \emptyset$.

This gives $\binom{p}{p/2}$ representations per solution, and MMT showed how to exploit this exponentially. However, MMT **only** splits 1-entries of the error vector — each 1-bit becomes either $(1,0)$ or $(0,1)$ in the two halves.

## 4. Key Insight: $1+1=0$ in $\mathbf{F}_2$

BJMM's breakthrough is realizing that in the binary field $\mathbf{F}_2$, the equation $1 + 1 = 0$ allows us to **also split 0-entries**:

- 1-entry split: $1 = 0 + 1$ or $1 = 1 + 0$ ✓ (MMT already uses this)
- **0-entry split: $0 = 0 + 0$ or $0 = 1 + 1$ ✓** (BJMM's new insight!)

Since $\omega \ll n$ in coding theory, the error vector has far more zeros than ones. By allowing both types of split, BJMM dramatically increases the number of representations per solution.

### Overlapping Index Sets

BJMM chooses $|I_1| = |I_2| = p/2 + \varepsilon$ with $|I_1 \cap I_2| = \varepsilon$. The solution set is recovered as the **symmetric difference**:

$$I = I_1 \;\Delta\; I_2 = (I_1 \cup I_2) \setminus (I_1 \cap I_2).$$

Each element in $I_1 \cap I_2$ appears in both halves and cancels out: contributes $1+1=0$ to the column sum. This allows $k+\ell-p$ zero-positions of $e$ to appear as $(1,1)$ in the two halves and still cancel.

The number of representations per solution becomes:

$$R_1(p, \ell; \varepsilon_1) = \binom{p}{p/2} \cdot \binom{k+\ell-p}{\varepsilon_1}.$$

This is a **multiplicative gain** of $\binom{k+\ell-p}{\varepsilon_1}$ over MMT.

BJMM applies the representation technique **twice** (two layers), yielding:

$$R_2(p, \ell; \varepsilon_1, \varepsilon_2) = \binom{p}{p/2} \cdot \binom{k+\ell-p}{\varepsilon_1} \cdot \binom{p/2+\varepsilon_1}{p/4+\varepsilon_2/2} \cdot \binom{k+\ell-p/2-\varepsilon_1}{\varepsilon_2}.$$

## 5. MERGE-JOIN: The Building Block

The fundamental operation in BJMM is **MERGE-JOIN**. Given two lists $L_1, L_2$ of binary vectors, a target weight $p$, an offset $r$ for partial matching, and a target vector $t \in \mathbf{F}_2^r$, MERGE-JOIN returns:

$$L = L_1 \bowtie L_2 = \{ x + y \mid x \in L_1, y \in L_2, \mathrm{wt}(x+y) = p, (Q(x+y))[r] = t \}.$$

The key is to match vectors by their projected labels. Sort $L_1$ by $(Qx)[r]$ and sort $L_2$ by $(Qy)[r] + t$, then scan both sorted lists in linear time (Knuth's algorithm). All pairs with matching labels form collisions. Inconsistent solutions (wrong weight) and duplicates are filtered out.

Running time: $\widetilde{O}(\max\{|L_1|, |L_2|, C\})$ where $C$ is the collision count. With uniformly distributed labels, $\mathbb{E}[C] = |L_1| \cdot |L_2| \cdot 2^{-r}$.

## 6. COLUMN MATCH: Three-Layer Computation Tree

BJMM solves the SMP via a **three-layer computation tree** composed of MERGE-JOIN operations. This is analogous to Wagner's generalized birthday algorithm but adapted to the syndrome matching setting.

### Parameter Setup

- $p_1 = p/2 + \varepsilon_1$
- $p_2 = p_1/2 + \varepsilon_2 = p/4 + \varepsilon_1/2 + \varepsilon_2$
- $r_1 \approx \log_2 R_1$ — coordinates constrained at layer 1
- $r_2 \approx \log_2 R_2$ — coordinates constrained at layer 2

### Layer 3: Disjoint Base Lists

For each of the 4 lists $L^{(2)}_i$ ($i = 1, \ldots, 4$), BJMM constructs disjoint base lists $B_{i,1}, B_{i,2}$ via random partition of $[k+\ell] = P_{i,1} \cup P_{i,2}$ with $|P_{i,1}| = |P_{i,2}| = (k+\ell)/2$:

$$B_{i,1} = \{ y \in \mathbf{F}_2^{k+\ell} \mid \mathrm{wt}(y) = p_2/2,\ y_j = 0\ \forall j \in P_{i,2} \}$$
$$B_{i,2} = \{ z \in \mathbf{F}_2^{k+\ell} \mid \mathrm{wt}(z) = p_2/2,\ z_j = 0\ \forall j \in P_{i,1} \}$$

Then $L^{(2)}_i = \text{MERGE-JOIN}(B_{i,1}, B_{i,2}, r_2, p_2, t^{(2)}_i)$. Size: $S_3 = \binom{(k+\ell)/2}{p_2/2}$.

### Layer 2: Merge to Form $L^{(1)}_j$

$$\text{For } j = 1, 2:\quad L^{(1)}_j = \text{MERGE-JOIN}\!\left(L^{(2)}_{2j-1},\ L^{(2)}_{2j},\ r_1,\ p_1,\ t^{(1)}_j\right).$$

### Layer 0: Final Merge

$$L = \text{MERGE-JOIN}\!\left(L^{(1)}_1,\ L^{(1)}_2,\ \ell,\ p,\ s\right).$$

The tree structure:

```
Layer 3:  B1,1  B1,2 | B2,1  B2,2 | B3,1  B3,2 | B4,1  B4,2
          ───────────────   ───────────────   ───────────────   ─────────
Layer 2:      L(2)_1            L(2)_2            L(2)_3            L(2)_4
               ───────────────   ───────────────
Layer 1:           L(1)_1                 L(1)_2
               ────────────────────────────────
Layer 0:                    L
```

### Pseudocode

```
ALGORITHM COLUMN MATCH(Q, s, p)
Parameters: ε1, ε2, set p1 = p/2 + ε1, p2 = p1/2 + ε2

1. Choose random partitions Pi,1, Pi,2 of [k+ℓ] and create base lists Bi,1, Bi,2
2. Choose t(1)_1 ∈R Fr1, set t(1)_2 = s[r1] + t(1)_1
3. Choose t(2)_1, t(2)_3 ∈R Fr2.
   Set t(2)_2 = (t(1)_1)[r2] + t(2)_1, t(2)_4 = (t(1)_2)[r2] + t(2)_3
4. For i = 1 to 4: L(2)_i = MERGE-JOIN(Bi,1, Bi,2, r2, p2, t(2)_i)
5. For j = 1 to 2: L(1)_j = MERGE-JOIN(L(2)_(2j-1), L(2)_(2j), r1, p1, t(1)_j)
6. L = MERGE-JOIN(L(1)_1, L(1)_2, ℓ, p, s)
7. Return L
```

## 7. Complexity Analysis

### Time Complexity

The three MERGE-JOIN calls dominate the running time. Let $S_i$ be the expected list size at layer $i$ and $C_i$ be the expected collision count:

$$S_i = \binom{k+\ell}{p_i} \cdot 2^{-r_i}, \quad \mathbb{E}[C_i] = S_i^2 \cdot 2^{r_i - r_{i-1}}.$$

The time complexity is:

$$T(p, \ell; \varepsilon_1, \varepsilon_2) = \max\{ T_3, T_2, T_1 \} = \max\{ S_3, C_3, S_2, C_2, S_1, C_1 \}.$$

Accounting for the probability of success in each ISD iteration:

$$P^{-1} = \frac{\binom{n}{\omega}}{\binom{k+\ell}{p}\binom{n-k-\ell}{\omega-p}}.$$

Optimizing over $p, \ell, \varepsilon_1, \varepsilon_2$ yields:

$$T_{\text{BJMM}} = 2^{0.04934n} \quad \text{(half-distance decoding, worst case)}.$$

### Space Complexity

The dominant memory cost is storing the largest intermediate list, approximately $2^{0.0286n}$.

### McEliece-Specific Parameters

For typical McEliece parameters with rate $R = 0.7577$ and relative distance $D = 0.04$:

| Metric | Half-distance | Full decoding |
|--------|--------------|---------------|
| Time | $2^{0.0672n}$ | $2^{0.1019n}$ |
| Space | $2^{0.0586n}$ | $2^{0.0769n}$ |

## 8. Comparison with Prior Work

| Algorithm | $2^{\alpha n}$ (half-dist) | Key technique |
|-----------|--------------------------|-------------|
| Lee-Brickell | $2^{0.05752n}$ | $p < \omega$ optimization |
| Stern | $2^{0.05564n}$ | Meet-in-the-middle |
| Ball-collision | $2^{0.05559n}$ | Non-exact matching |
| MMT | $2^{0.05364n}$ | Representations, disjoint $I_1, I_2$ |
| **BJMM** | **$2^{0.04934n}$** | Overlapping sets, $1+1=0$ splits |

BJMM improves on MMT by a factor of $2^{0.0043n}$ — approximately a $\mathbf{3\times}$ speedup at $n=2000$, directly impacting McEliece security levels.

## 9. Provable Variant

The heuristic analysis relies on the uniformity of projected partial sums $(Qe)[r]$. The paper also provides **PROVABLE CM**: a variant that repeatedly invokes COLUMN MATCH with fresh random target values and aborts if any list exceeds $2^{\gamma n}$ times its expected size (for a constant $\gamma > 0$). For all but a negligible fraction of matrices $Q$, this variant succeeds with probability $> 1 - 3/e^2$ in time $\widetilde{O}(T \cdot 2^{3\gamma n})$.

## References

1. A. Becker, A. Joux, A. May, A. Meurer. *Decoding Random Binary Linear Codes in $2^{n/20}$: How $1+1=0$ Improves Information Set Decoding.* Eurocrypt 2012. ([ePrint](https://eprint.iacr.org/2012/026))
