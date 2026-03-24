---
layout: article
title: "BJMM + May-Ozerov ISD: Optimizing Nearest Neighbor Search"
date: 2026-03-25
key: BJMM-May-Ozerov-ISD-2026
lang: en
tags: [Code-Based-Cryptography, ISD, May-Ozerov, Nearest-Neighbor, Walsh-Hadamard, BJMM, McEliece]
bilingual: true
mathjax: true
---

**tl;dr** The May-Ozerov algorithm (EUROCRYPT 2015) replaces the sort-and-match step in Stern ISD with a Walsh-Hadamard Transform (WHT) approach, reducing the nearest-neighbor problem to near-constant time per subproblem. Combined with BJMM, this yields ISD complexity around \$2^{0.053n}\$. However, a 2025 analysis proves this improvement is "galactic" — the crossover point where May-Ozerov beats Stern requires n > 1,874,400 bits, far beyond any practical cryptographic parameter. This post details HIVe (Hybrid Inbound Variant Ensemble), WHT-based NN search, and why asymptotic theory diverges from practice.

Useful references:

- [May, Ozerov — EUROCRYPT 2015](https://eprint.iacr.org/2015/437)
- [Hirose — PQCrypto 2016 (generalization to Fq)](https://eprint.iacr.org/2016/237)
- [Bouillaguet, Delaplace, Hamdad — CIC 2025 (galactic analysis)](https://cic.iacr.org/p/2/1/31)

---

## 1. Background: The Bottleneck in Stern ISD

The previous post ([BJMM Algorithm](/2026/03/24/BJMM-Algorithm-en/)) covered how BJMM (Eurocrypt 2012) pushes ISD complexity to \$2^{0.04934n}\$ via overlapping index sets and the \$1+1=0\$ trick. But both BJMM and its predecessor MMT share a common bottleneck: the **MERGE-JOIN** operation — a sorting-based matching algorithm.

Recall Stern ISD's structure:

```
1. Gaussian elimination: H → (Q | I)
2. Build lists:
   U = { Q·e₁ | e₁ ∈ F₂^{k/2}, wt(e₁) = p/2 }
   V = { e₂ ⊕ s | e₂ ∈ F₂^{n-k}, wt(e₂) = p/2 }
3. Sort-and-match: find (u,v) ∈ U×V with u⊕v matching in last ℓ bits
```

Step 3 is the bottleneck: each list has size \$2^{k/2}\$, so naive matching costs \$2^k\$. Even with sorting, the dual-pointer scan costs \$\widetilde{O}(2^{k/2})\$.

> **Core question**: Can we find near neighbors more efficiently than sort-and-match?

## 2. Walsh-Hadamard Transform (WHT)

Before diving into May-Ozerov, let's understand the Walsh-Hadamard Transform.

### 2.1 Definition

For a Boolean function \$f: \mathbf{F}_2^m \to \mathbf{F}_2\$, its Walsh transform is:

$$W_f(u) = \sum_{x \in \mathbf{F}_2^m} (-1)^{f(x) + u \cdot x}, \quad u \in \mathbf{F}_2^m$$

where \$u \cdot x = \sum_i u_i x_i \pmod{2}\$ is the dot product.

The Fast WHT (FWHT) computes all \$2^m\$ Walsh coefficients in \$O(m \cdot 2^m)\$ via butterfly operations:

```
FWHT(f):
  for stage = 1 to m:
    for block = 0 to 2^m - 1 step 2^stage:
      for i = 0 to 2^{stage-1} - 1:
        x = f[block + i]
        y = f[block + i + 2^{stage-1}]
        f[block + i] = x + y
        f[block + i + 2^{stage-1}] = x - y
```

### 2.2 WHT and Hamming Distance

The WHT has a crucial property for our problem: it efficiently computes Hamming weight spectra. For \$u, v \in \mathbf{F}_2^m\$, the Hamming distance relates to Walsh coefficients through:

$$\sum_{x \in \mathbf{F}_2^m} (-1)^{w_H(x)} \cdot (-1)^{u \cdot x} = \sum_{x \in \mathbf{F}_2^m} (-1)^{w_H(x \oplus u) \cdot v}$$

The WHT can thus efficiently find all vectors at a specific Hamming distance from a given vector — exactly what we need for the matching step.

## 3. The Nearest-Neighbor Problem (NMP)

### 3.1 Problem Definition

**Definition (Nearest-Neighbor Problem, Binary Case)**: Let \$0 < \delta < 1/2\$, \$0 < \tau < 1\$. Given two sets \$U, V \subset \mathbf{F}_2^m\$, \$|U| = |V| = 2^m\$ (random and pairwise independent), find all pairs \$(u, v) \in U \times V\$ with \$w_H(u \oplus v) = \delta m\$.

This is precisely the problem that arises in Stern ISD's matching step:
- \$U\$ contains vectors \$Qe_1\$ (weight p/2)
- \$V\$ contains vectors \$e_2 \oplus s\$ (weight p/2)
- We're looking for pairs where \$u \oplus v\$ has Hamming weight p

### 3.2 Naive vs. Sort-and-Match

| Method | Time Complexity | Notes |
|--------|---------------|-------|
| Naive | \$O(2^{2m})\$ | Enumerate all pairs |
| Sort-and-match | \$\widetilde{O}(2^m)\$ | Stern's approach |
| May-Ozerov | \$\widetilde{O}(2^{\alpha m})\$, \$\alpha < 1\$ | WHT-based |

## 4. HIVe: WHT-Driven Nearest-Neighbor Search

HIVe (Hybrid Inbound Variant Ensemble), proposed by May and Ozerov at EUROCRYPT 2015, replaces sort-and-match with a WHT-based approach. The core idea: **split the problem into exponentially many polynomial-size subproblems**, then use WHT to efficiently identify which subproblems contain solutions.

### 4.1 Core Insight: Randomization

The algorithm first randomizes both lists with a random permutation \$P\$ and random offset \$r\$:

$$\widetilde{U} = \{ \widetilde{u} = Pu \oplus r \mid u \in U \}$$

This serves two purposes:
1. The true solution (a genuine near-neighbor pair) survives randomization with constant probability
2. The randomized lists have uniform structure, enabling probabilistic filtering

### 4.2 Recursive Filtering

HIVe uses recursive filtering to build a search tree:

```
MO-NN(U, V, δ):
  1. Randomize: pick random permutation P and offset r
  2. Recursive build:
     for each coordinate subset A ⊂ [m]:
       U_A = { u ∈ U | # of 1's in A satisfies a balance condition }
       V_A = { v ∈ V | # of 1's in A satisfies a balance condition }
       if |U_A|, |V_A| are polynomial:
         recursively call MO-NN(U_A, V_A, δ)
  3. Leaf nodes: naive search
```

The "balance condition" is key: a vector is **balanced** if each symbol in \$\mathbf{F}_q\$ appears equally often in its coordinates. By filtering for balanced vectors, we exponentially reduce list sizes while keeping solutions with constant probability.

### 4.3 Tree Diagram

```
Root: all 2^m vectors
     │
     ├─ A₁ (first coordinate subset) ──┬─ Balanced U_A, V_A
     ├─ A₂ (second subset)            │  (size ~ polynomial)
     ├─ ...
     └─ A_t (t-th subset)
         │
         └─ Leaf: naive search
```

### 4.4 WHT in the Filtering Process

The key question: how do we efficiently find vectors in U that satisfy a balance condition on a coordinate subset A?

**Answer: WHT.** For the subset A, define:
$$f_{A,r}(x) = 1 \iff w_H(x_A) = r$$

The Walsh transform of \$f_{A,r}\$ reveals exactly which vectors in U have Hamming weight r in coordinates A. By setting up the WHT appropriately, we can extract all candidates in \$O(2^{|A|})\$ time instead of \$O(2^m)\$.

More concretely, May-Ozerov splits each vector into t pieces:
$$x = (x_1 \mid x_2 \mid \cdots \mid x_t), \quad |x_i| = \alpha_i m$$

At level i, they filter by the balance condition on \$A_i \subset [\alpha_i m]\$. The WHT computes the weight distribution for each piece, and candidates are extracted in time proportional to the piece size.

### 4.5 Complexity Analysis

From Hirose's survey (PQCrypto 2016), the May-Ozerov algorithm's time complexity is:

$$T = \widetilde{O}(q^{(y+\varepsilon)m})$$

where q is the field size (q=2 for binary), and y is:

$$y = (1-\delta) \cdot \left( H_q(\delta) - \frac{1}{q}\sum_{x \in \mathbf{F}_q} H_q(q h_x \delta) \right)$$

For the binary case (q=2), Hirose's Table 1 gives:
- Stern-MO complexity exponent: \$f(2; R_w) = 0.05498\$
- Original Stern ISD: \$f_S(2; R_w') = 0.05563\$

Improvement: \$\Delta = -0.00065\$ (≈ \$2^{0.00065n}\$ faster)

## 5. Application to ISD: Stern-MO and BJMM-MO

### 5.1 Stern-MO

Replace Stern's sort-and-match with HIVe:

```
Stern-MO(n, k, H, s):
  1. Gaussian elimination: H → (Q | I)
  2. Build lists:
     U = { Qe₁ | e₁ ∈ F₂^{k/2}, wt(e₁) = p/2 }
     V = { e₂ ⊕ s | e₂ ∈ F₂^{n-k}, wt(e₂) = p/2 }
  3. HIVe search: find (u,v) ∈ U×V with w_H(u⊕v) = p
```

### 5.2 BJMM-MO

BJMM's MERGE-JOIN also has a sort-and-match component. Replace it with HIVe:

```
BJMM-MO(Q, s, p):
  1. Parameters: p₁ = p/2 + ε₁, p₂ = p₁/2 + ε₂
  2. Layer 3: build base lists B_{i,1}, B_{i,2}
  3. Replace MERGE-JOIN with HIVe for matching
  4. Three-layer tree, but each matching step uses WHT
```

The original May-Ozerov paper reports BJMM-MO complexity ≈ \$2^{0.053n}\$.

### 5.3 Full Complexity Comparison

| Algorithm | Complexity Exponent (Half-Distance) | Core Technique |
|-----------|--------------------------------------|----------------|
| Lee-Brickell | \$2^{0.05752n}\$ | \$p < \omega\$ |
| Stern | \$2^{0.05563n}\$ | MITM |
| MMT | \$2^{0.05364n}\$ | Representation, disjoint sets |
| BJMM | \$2^{0.04934n}\$ | Overlapping sets |
| Stern-MO | \$2^{0.05498n}\$ | WHT-NN |
| BJMM-MO | \$2^{0.053n}\$ (approx) | BJMM + WHT-NN |

## 6. "Galactic" Analysis: Why the Improvement Doesn't Matter in Practice

In 2025, Bouillaguet, Delaplace, and Hamdad published a landmark analysis in *IACR Communications in Cryptology*. Their conclusion: May-Ozerov is a **galactic algorithm** for syndrome decoding.

### 6.1 Crossover Point Analysis

The crossover point where May-Ozerov beats Stern:

| Metric | Value |
|--------|-------|
| Minimum code length needed | n > 1,874,400 |
| Operations at crossover | > \$2^{63,489}\$ |

For all practical McEliece parameters (n < 10,000), Stern is actually faster.

### 6.2 Why Asymptotic Theory Doesn't Match Practice

1. **Hidden polynomial factors**: The ~O notation hides large polynomial factors. FWHT costs \$O(m \cdot 2^m)\$, and these factors dominate for practical m.

2. **Constant factors**: WHT requires complex butterfly operations; sort-and-match needs only simple comparisons. WHT's constant factor is vastly larger.

3. **Cache locality**: Sorted data is contiguous in memory (cache-friendly); WHT requires many random memory accesses.

```
Stern (sort-and-match):        May-Ozerov (WHT):
┌─────────────────────┐        ┌─────────────────────┐
│ Sort: O(m log m)    │        │ WHT: O(m · 2^m)     │
│ Scan: O(2^m)        │        │ Filter: poly recurs │
│ Total: ~O(2^m)      │        │ Total: ~O(2^{αm})   │
└─────────────────────┘        └─────────────────────┘
  ↑ Small constant, cache-       ↑ Large constant, random
    friendly                       memory access
```

### 6.3 What "Galactic" Means

David Johnson's "galactic algorithm" definition: an algorithm that is theoretically best but only outperforms simple methods at input sizes so large they would "fill the entire universe."

May-Ozerov perfectly fits this definition:
- Theory: exponent 0.05498 < 0.05563 ✓
- Practice: requires n > 1,874,400 ✗ (no known code even approaches this)

## 7. Conclusion

May-Ozerov (EUROCRYPT 2015) is a landmark work applying Walsh-Hadamard Transform to ISD's nearest-neighbor problem. It demonstrates the power of WHT in combinatorial optimization, reducing the complexity exponent from \$2^{0.05563n}\$ (Stern) to \$2^{0.05498n}\$.

However, the 2025 "galactic" analysis proves this improvement is theoretical:
- All practical McEliece parameters (n < 10,000) are far below the crossover point
- Hidden polynomial factors make WHT slower than simple sorting in practice

This lesson highlights a crucial distinction in cryptography: **asymptotic improvement** ≠ **practical improvement**. For a field that cares deeply about concrete security margins, we must always check whether theoretical gains materialize at realistic parameter sizes.

## References

1. A. May, I. Ozerov. *On Computing Nearest Neighbors with Applications to Decoding of Binary Linear Codes.* EUROCRYPT 2015. ([ePrint](https://eprint.iacr.org/2015/437))
2. S. Hirose. *May-Ozerov Algorithm for Nearest-Neighbor Problem over Fq and Its Application to Information Set Decoding.* PQCrypto 2016. ([ePrint](https://eprint.iacr.org/2016/237))
3. C. Bouillaguet, C. Delaplace, M. Hamdad. *The May-Ozerov Algorithm for Syndrome Decoding is "Galactic".* IACR Communications in Cryptology (2025). ([Link](https://cic.iacr.org/p/2/1/31))
4. A. Becker, A. Joux, A. May, A. Meurer. *Decoding Random Binary Linear Codes in 2^{n/20}: How 1+1=0 Improves Information Set Decoding.* EUROCRYPT 2012.
5. J. Stern. *A method for finding codewords of small weight.* Coding Theory 1989.
