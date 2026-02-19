# Distributed Coding: Path Tracing

Figure out path $S_1 \rightarrow S_2 \rightarrow \ldots \rightarrow S_k$

This is a __static per-flow__ aggregation: switch IDs don't change between packets

<v-click>

Key idea: spread switch IDs across many packets using __distributed encoding__

</v-click>

---

## Technique 1: Baseline (Coupon Collector)

Each switch writes its ID with $\Pr = \frac{1}{k}$ (reservoir sampling)

<v-click>

Simple but slow: need $k \ln k$ packets on average to collect all $k$ switch IDs

Problem: first IDs come quickly, last ones take many more packets ("long tail")

</v-click>

---

### Baseline: Worked Example ($k = 5$)

Path: $S_1 \rightarrow S_2 \rightarrow S_3 \rightarrow S_4 \rightarrow S_5$, each writes with $\Pr = 1/5$

<v-clicks>

- $p_1$ carries $S_3$ &nbsp;&nbsp;→&nbsp;&nbsp; known: {$S_3$}
- $p_2$ carries $S_1$ &nbsp;&nbsp;→&nbsp;&nbsp; known: {$S_1, S_3$}
- $p_3$ carries $S_3$ &nbsp;&nbsp;→&nbsp;&nbsp; known: {$S_1, S_3$} &nbsp; ← duplicate!
- $p_4$ carries $S_1$ &nbsp;&nbsp;→&nbsp;&nbsp; known: {$S_1, S_3$} &nbsp; ← duplicate again!
- $p_8$ carries $S_5$ &nbsp;&nbsp;→&nbsp;&nbsp; known: {$S_1, S_2, S_3, S_5$}
- $p_{12}$ carries $S_4$ → known: {$S_1, S_2, S_3, S_4, S_5$} &nbsp; ✓ done!

</v-clicks>

<v-click>

For $k=25$: __median 89 packets__, 99th percentile 189 packets

</v-click>

---

## Technique 2: Distributed XOR

Each switch XORs its ID onto the packet digest with $\Pr = p = 1/d$

<v-click>

Receiver computes $g(p_j, 1), \ldots, g(p_j, k)$ to determine which switches XOR'd

</v-click>

<v-click>

If exactly __one__ unknown block was XOR'd, we can recover it, then back-substitute

</v-click>

---

### XOR: Worked Example ($k = 5$)

Path: $S_1 \rightarrow S_2 \rightarrow S_3 \rightarrow S_4 \rightarrow S_5$, each XORs with $\Pr = 1/5$

<v-clicks>

- $p_1$: digest = $S_2 \oplus S_4$ &nbsp;&nbsp;→&nbsp;&nbsp; 2 unknowns, can't decode yet
- $p_2$: digest = $S_4$ &nbsp;&nbsp;→&nbsp;&nbsp; 1 unknown → __decode $S_4$__ ✓
- Back-sub $p_1$: $S_2 \oplus \cancel{S_4}$ → __decode $S_2$__ ✓
- $p_3$: digest = $S_1 \oplus S_2 \oplus S_4$ → know $S_2, S_4$ → __decode $S_1$__ ✓
- Continue until all 5 switches recovered

</v-clicks>

<v-click>

Efficient: $O(k \log k)$ packets, but decoding requires solving system of XOR equations

</v-click>

---
layout: default
---

## Hybrid Approach

Pure XOR: efficient but complex &nbsp;|&nbsp; Pure Baseline: simple but slow

<v-click>

__Solution: Interleave both schemes across layers__

</v-click>

<v-clicks>

- **Layer 0** (Baseline): easy hops found quickly, with $\Pr = \tau \approx 3/4$
- **Layers 1..$\mathcal{L}$** (XOR): clean up remaining hops efficiently
- Paper shows __1-2 XOR layers__ suffice in practice

</v-clicks>

---

### Hybrid: Worked Example ($k = 5$, $\mathcal{L}=2$)

Each packet: hash decides → Baseline ($\tau = 3/4$) or XOR ($1 - \tau = 1/4$)

<v-clicks>

- $p_1$ [Baseline]: $S_2$ → known: {$S_2$}
- $p_2$ [Baseline]: $S_5$ → known: {$S_2, S_5$}
- $p_3$ [Baseline]: $S_1$ → known: {$S_1, S_2, S_5$}
- $p_4$ [XOR]: $S_3 \oplus S_4$ → 2 unknowns, wait...
- $p_5$ [Baseline]: $S_2$ → duplicate
- $p_6$ [XOR]: $S_4$ → 1 unknown → __decode $S_4$__ ✓, back-sub $p_4$ → __decode $S_3$__ ✓

</v-clicks>

<v-click>

__All 5 switches in 6 packets!__

</v-click>

---

### Comparison ($k = 25$)

| | Baseline | XOR only | Hybrid (2 layers) |
|--|----------|----------|-------------------|
| Median packets | 89 | ~80 | __41__ |
| 99th pct packets | 189 | ~150 | __68__ |
| Complexity | $k \ln k$ | $O(k \log k)$ | $k \log \log^* k$ |

<v-click>

Baseline finds most hops fast, XOR layers __clean up the stragglers__

</v-click>

<v-click>

vs competing approaches (PPM, AMS): __25–36x fewer packets__

For Kentucky Datalink ($D = 59$): PINT needs only __42 packets__ on average

</v-click>
