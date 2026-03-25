---
layout: article
title: "Regular ISD: Tailoring Information Set Decoding for Structured Decoding"
date: 2026-03-26
key: Regular-ISD-2026
lang: en
tags: [Code-Based-Cryptography, Information-Set-Decoding, Regular-Syndrome-Decoding, RSD, Esser-Santini, 2024]
bilingual: true
mathjax: true
---

**tl;dr** Regular Syndrome Decoding (RSD) is a structured variant of standard Syndrome Decoding (SD), where the error vector is partitioned into blocks with exactly one nonzero per block. The core contribution of Esser–Santini (CRYPTO 2024) is **Regular-ISD**: tailoring the full machinery of advanced ISD techniques to the ring structure of RSD, reducing the complexity from CCJ's $2^{0.141n}$ to $2^{0.112n}$. They also reveal that RSD hardness is regime-dependent — uniqueness of the solution alone does not characterize the worst case.

<!-- more -->

## 1. From SD to RSD: Structured Errors

### Standard Syndrome Decoding (SD)

Given $H \in \mathbb{F}_2^{(n-k)\times n}$ and syndrome $s \in \mathbb{F}_2^{n-k}$, the **Syndrome Decoding (SD) problem** asks to find $e \in \mathbb{F}_2^n$ with $\mathrm{wt}(e) = \omega$ such that $He^\top = s$.

SD is the underlying hardness assumption of the McEliece public-key cryptosystem. The state-of-the-art generic algorithms for SD are based on **Information Set Decoding (ISD)**, with the best known complexity of **BJMM** at $2^{0.04934n}$.

### Regular SD: A Structured Error Distribution

Many efficient cryptographic constructions — especially signature schemes — require structured variants of SD. This motivates **Regular Syndrome Decoding (RSD)**, introduced by Augot–Finiasz–Sendrier (AFS 2005).

**Definition** (RSD): Let $n = N \times t$. Partition the error vector into $t$ blocks of size $N$:

$$
e = (e_1 \ \| \ e_2 \ \| \ \cdots \ \| \ e_t), \quad e_i \in \mathbb{F}_2^N.
$$

RSD requires that each block has exactly one nonzero coordinate:

$$
\forall i \in [1,t]: \quad \mathrm{wt}(e_i) = 1.
$$

The total Hamming weight of $e$ is always $t$, with one nonzero per block.

**Independence**: While the nonzero position within each block is chosen independently from $N$ candidates, the blocks themselves are unconstrained. This gives $N^t$ candidate solutions — far fewer than the $\binom{n}{t}$ of random SD, paradoxically making RSD **harder to attack** in some parameter regimes.

RSD is central to several advanced constructions: CCJ signatures, PCGs, VOLE, and correlated Oblivious Transfer.

## 2. The Ring Structure: $\mathbb{F}_2[X]/(X^n - 1)$

The key mathematical structure underlying RSD is the polynomial ring:

$$
R_n := \mathbb{F}_2[X] \bmod (X^n - 1).
$$

Encoding $H$ and $s$ as polynomials, the syndrome equation $He^\top = s$ becomes a **multiplication equation** in this ring:

$$
h(X) \cdot e(X) \equiv s(X) \pmod{X^n - 1}.
$$

The error polynomial $e(X)$ has a special form due to the block structure:

$$
e(X) = \sum_{j=0}^{t-1} X^{j \cdot N} \cdot u_j(X),
$$

where each $u_j(X)$ is a monomial (exactly one nonzero term).

### Ring Decomposition and Cyclotomic Cosets

The structure of $R_n$ is determined by the factorization of $X^n - 1$ over $\mathbb{F}_2$:

$$
X^n - 1 = \prod_{C \in \mathcal{C}_n} m_C(X),
$$

where $\mathcal{C}_n$ is the set of **2-cyclotomic cosets** of $\langle 2 \rangle$ acting on $\mathbb{Z}_n$, and $m_C$ is the minimal polynomial of the corresponding coset.

By the **Chinese Remainder Theorem (CRT)**:

$$
R_n \cong \bigoplus_{C \in \mathcal{C}_n} \mathbb{F}_2[X]/(m_C(X)) = \bigoplus_{C \in \mathcal{C}_n} \mathbb{F}_{2^{|C|}}.
$$

Each component $\mathbb{F}_{2^{|C|}}$ has dimension $|C|$ (the size of the coset). The ring equation decomposes into **independent sub-equations** per component:

$$
h_C \cdot e_C = s_C \quad \text{in } \mathbb{F}_{2^{|C|}}.
$$

This is the core insight of Esser–Santini: **coset decomposition separates the overall problem into independent subproblems**, each indexed by a cyclotomic coset.

## 3. Parameter Regimes: Uniqueness Is Not Enough

A natural question: when is RSD actually hard? Prior work leaned on the **uniqueness condition** as the primary hardness criterion.

Esser–Santini's first major contribution is a **systematic hardness classification** of RSD, revealing a counterintuitive fact:

> **Classification by uniqueness alone is insufficient.**  
> There exist polynomial-time solvable instances with unique solutions, and there exist hard instances without unique solutions.

They identify three parameter regimes:

| Regime | Characteristic | Difficulty |
|--------|---------------|------------|
| Large $N$ (many small blocks) | Many blocks, limited choices per block | Polynomial time |
| Medium $N$ | Moderate block size | Worst case, hard |
| Small $N$ (few large blocks) | Few blocks, large per-block space | Approaches standard SD |

## 4. Prior Attacks: Algebraic and Linearization

### Briaud–Øygarden (Eurocrypt 2023)

Briaud and Øygarden introduced the first systematic **algebraic attack**, modeling RSD as a polynomial system:

- **Linear part** $P$: $n-k$ linear equations from $He^\top = s$
- **Regular structure** $B$: quadratic constraints encoding per-block uniqueness

$$
e_{i,j_1} \cdot e_{i,j_2} = 0 \quad (j_1 \neq j_2)
$$

Resolution uses the **Macaulay matrix**: arrange all monomials up to degree $d$ as columns and all polynomials as rows, then row-reduce to expose the solution structure.

### CCJ Attack (Eurocrypt 2023)

Carozza, Couteau, and Joux proposed a **linearization-based** attack with two algorithms:

| Algorithm | Complexity |
|-----------|-----------|
| CCJ-1 | $2^{0.141n}$ |
| CCJ-2 | $2^{0.135n}$ |

The core idea: treat the nonlinear equations as a linear system via substitution (linearization). Esser–Santini provide the first rigorous **asymptotic analysis** confirming these worst-case complexities.

## 5. Core Contribution: Regular-ISD

### Main Idea

Esser–Santini's primary contribution is **Regular-ISD**: adapting advanced ISD techniques — rather than linearization or generic algebra — specifically to the RSD setting.

**Key steps**:

1. **Ring decomposition**: Apply coset decomposition $R_n \cong \bigoplus_C \mathbb{F}_{2^{|C|}}$, splitting the problem into components
2. **Component ISD**: Run **customized ISD** in each component $\mathbb{F}_{2^{|C|}}$
3. **Information set selection**: Choose "information sets" respecting the block constraints in the ring structure
4. **Representation technique**: Incorporate MMT/BJMM-style representation techniques, adapted to RSD's regular structure

This is fundamentally different from CCJ — CCJ tries to linearize and bypass the block structure; Regular-ISD **actively exploits** it.

### Complexity Result

**Best Regular-ISD algorithm: $2^{0.112n}$**

| Algorithm | Complexity | Source |
|-----------|-----------|--------|
| CCJ-1 | $2^{0.141n}$ | Eurocrypt 2023 |
| CCJ-2 | $2^{0.135n}$ | Eurocrypt 2023 |
| **Regular-ISD** | **$2^{0.112n}$** | **CRYPTO 2024** |

Compared to CCJ-1, Regular-ISD reduces the exponent from $0.141$ to $0.112$ — equivalent to reducing $n$ by ~21% at the same security level.

## 6. Impact on Concrete Parameters

Beyond asymptotics, Esser–Santini analyze **specific parameter sets** used in RSD-based schemes.

### Up to 30-bit Security Reduction

For parameters recommended by CCJ and related schemes, Regular-ISD is significantly more efficient than previously estimated:

- For certain parameter sets, the actual security level is **up to 30 bits lower** than previously assumed
- To maintain a target security level, $n$ must be increased accordingly

This directly affects RSD-based signature schemes (CCJ23a, CLY+24, etc.).

## 7. Key Takeaways

1. **RSD is a structured variant of SD**: error vectors are partitioned into blocks of weight 1, with a natural polynomial representation in $\mathbb{F}_2[X]/(X^n-1)$
2. **Cyclotomic coset decomposition is key**: $X^n-1$'s factorization into 2-coset minimal polynomials splits the problem into independent components via CRT
3. **Uniqueness is insufficient for hardness classification**: RSD hardness is regime-dependent, requiring systematic analysis
4. **Regular-ISD significantly outperforms prior attacks**: $2^{0.112n}$ vs $2^{0.141n}$, leveraging ISD techniques adapted to the ring structure
5. **Concrete parameters need reassessment**: existing RSD-based scheme parameters may overestimate security by up to 30 bits
