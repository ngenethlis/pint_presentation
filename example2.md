# Distributed Coding: Path Tracing

Figure out path $S_1 \rightarrow S_2 \rightarrow \ldots \rightarrow S_k$

This is a __static per-flow__ aggregation: switch IDs don't change between packets

<v-click>

__Why useful?__ Traffic takes specific paths but operators rarely know which. Path tracing enables:
- **Fault localization**: which switch is causing packet loss or high latency?
- **Routing verification**: is traffic following expected paths through the network?

</v-click>

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

<BaselineTrace />

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


<XorTrace />

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


<HybridTrace />

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

$\log^* k$ is the __iterated logarithm__ — how many times you apply $\log_2$ before reaching $\leq 1$.
$\log^*(2^{65536}) = 5$ — so $k \log \log^* k$ is __essentially linear__ in $k$

</v-click>

<v-click>

vs competing approaches (PPM, AMS): __25–36x fewer packets__

For Kentucky Datalink ($D = 59$): PINT needs only __42 packets__ on average

</v-click>
