---
layout: article
title: "BJMM Algorithm: How 1+1=0 Improves Information Set Decoding"
date: 2026-03-24
key: BJMM-Algorithm-2026
lang: en
tags: [Code-Based-Cryptography, Information-Set-Decoding, ISD, McEliece, BJMM, 2012]
bilingual: true
---

**tl;dr**: BJMM (Eurocrypt 2012) reduces the complexity of decoding random binary linear codes from MMT's 2^{0.05364n} to 2^{0.04934n}. The key innovation is allowing index sets to overlap, leveraging that 1+1=0 in GF(2) to split both 1-positions and 0-positions.

<!-- more -->

## 1. Background: Information Set Decoding

The syndrome decoding problem is central to code-based cryptography. ISD (Prange, 1962) is the most fundamental approach.

- Lee-Brickell (1988): 2^{0.05752n}
- Stern (1989): MITM -> 2^{0.05564n}
- Ball-collision (2011): non-exact matching -> 2^{0.05559n}
- MMT (2011): representation technique -> 2^{0.05364n}

## 2. Submatrix Matching Problem (SMP)

Given Q in GF(2)^{l x (k+l)} and target s in GF(2)^l, find I subseteq [1, k+l], |I|=p, such that sum_{i in I} q_i = s.

MMT's insight: each solution has C(p, p/2) representations by writing I = I_1 cup I_2 with I_1 cap I_2 = varnothing.

## 3. The Key Insight: 1+1=0

MMT requires I_1 and I_2 to be disjoint. BJMM allows |I_1 cap I_2| = epsilon > 0, giving the solution as the symmetric difference:

    I = I_1 Delta I_2 = (I_1 cup I_2) \ (I_1 cap I_2)

This enables splitting 0-positions (which are far more numerous than 1-positions): (1,1) contributes 0 via 1+1=0.

> **Representations**: MMT has C(p, p/2); BJMM has C(p, p/2) * C(k+l-p, epsilon)

## 4. COLUMN MATCH: Three-Layer Algorithm

A three-level divide-and-conquer algorithm:

- Layer 3: 4 pairs of base lists (disjoint splits)
- Layer 2: MERGE-JOIN to weight-p_2 vectors
- Layer 1: MERGE-JOIN to weight-p_1 vectors
- Layer 0: Final MERGE-JOIN to SMP solutions

Correctness follows from |I_1 Delta I_2| = 2(|I_1| - |I_1 cap I_2|) = p.

## 5. Complexity

| Algorithm | Half-distance decoding | Space |
|-----------|------------------------|-------|
| Lee-Brickell | 2^{0.05752n} | — |
| Stern | 2^{0.05564n} | 2^{0.0135n} |
| Ball-collision | 2^{0.05559n} | 2^{0.0148n} |
| MMT | 2^{0.05364n} | 2^{0.0216n} |
| **BJMM** | 2^{0.04934n} | 2^{0.0286n} |

## 6. Conclusion

BJMM's three-way decomposition and 1+1=0 insight remain essential for understanding ISD evolution.

## References

1. Becker, Joux, May, Meurer. Eurocrypt 2012. ePrint 2012/026
2. J. Stern. Coding Theory and Applications, 1989.
3. May, Meurer, Thomae. Asiacrypt 2011.
4. Bernstein, Lange, Peters. Crypto 2011.
