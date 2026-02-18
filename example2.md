# Distributed Coding, Path Tracing

Figure out path $S_1 -> S_2 -> ... -> S_k$


## Technique 1: Baseline Coupon Collecter

Each switch writes its ID with $pr=frac{1}{k}$

## Technique 2: Distributed XOR

Each switch XOR's their ID onto packet with $pr=p$

Decoder: Solves system of equations, if we have $A$ and $A XOR B$ we can find B

---
layout: default
---

## Hybrid Aproach:

Pure XOR efficient but complex
Pure Baseline simple but slow

Solution?
- Layer 0 using Baseline: easy hops are found quickly
- Layers 1..L clean up missing hops efficiently

Paper shows __2__ XOR layers are required usually.


Result: Number of packets needed to trace a path reduced drastically, compared to standard probabilistic marking
