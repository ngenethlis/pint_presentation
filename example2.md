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

$k$ = number of switches on the path. Each switch writes its ID independently with $\Pr = \frac{1}{k}$ — a new packet reveals a previously-unseen switch with some probability

<v-click>

__Why $k \ln k$?__ After collecting $j$ of $k$ IDs, any packet reveals a new one with probability $\frac{k-j}{k}$

$$\mathbb{E}[\text{total packets}] = \sum_{j=0}^{k-1} \frac{k}{k-j} = k \sum_{i=1}^{k} \frac{1}{i} = k H_k \approx k \ln k$$

</v-click>

<v-click>

__Tail bound__: after $t$ packets, $\Pr[\text{switch } i \text{ not yet seen}] = \left(1-\tfrac{1}{k}\right)^t \leq e^{-t/k}$

_(Chernoff: use $1-x \leq e^{-x}$, applied to each Bernoulli trial independently)_

Union bound over all $k$ switches: $\;\Pr[\text{any switch missing}] \leq k \cdot e^{-t/k}$

Set $t = c\, k \ln k$: $\;\Pr[\text{any missing}] \leq k \cdot k^{-c} = k^{1-c} \to 0$ for any constant $c > 1$

</v-click>

<v-click>

__Long tail__: the last few IDs dominate — the $k$-th ID alone takes $k$ packets in expectation

</v-click>

---

### Baseline: Worked Example ($k = 5$)

Path: $S_1 \rightarrow S_2 \rightarrow S_3 \rightarrow S_4 \rightarrow S_5$, each writes with $\Pr = 1/5$

<BaselineTrace />

---

## Technique 2: Distributed XOR

Each switch XORs its ID onto the packet digest with $\Pr = p = 1/d$

<v-click>

Each packet defines a __linear equation over $\mathbb{F}_2$__:

$$\bigoplus_{\{i\,:\,g(p_j,i) < \tfrac{1}{d}\}} s_i \;=\; \text{digest}(p_j)$$

The receiver uses the same hash $g$ to reconstruct which switches participated

</v-click>

<v-click>

Decode by __Gaussian elimination over $\mathbb{F}_2$__:
- Equation with **one unknown** $s_i$: directly solve, then back-substitute into all other equations containing $s_i$
- This may reduce further equations to one-unknown — peel off iteratively

</v-click>

<v-click>

__Advantage over Baseline__: equations share information across packets — knowing $S_3 \oplus S_4$ from one packet and $S_4$ from another gives $S_3$ for free; Baseline requires each switch to be written independently

Complexity: $O(k \log k)$ packets

</v-click>

---

### XOR: Worked Example ($k = 5$)

<XorTrace />

---
layout: default
---

## Hybrid Approach Worked Example with ($k = 5$, $\mathcal{L}=2$)

Pure Baseline: simple but slow | Pure XOR: efficient but complex 

__Interleave protocols__
Each packet: hash decides → Baseline ($\tau = 3/4$) or XOR ($1 - \tau = 1/4$)


<HybridTrace />

---

### Comparison ($k = 25$)

| | Baseline | XOR only | Hybrid (2 layers) |
|--|----------|----------|-------------------|
| Median packets | 89 | ~80 | __41__ |
| 99th pct packets | 189 | ~150 | __68__ |
| Complexity | $k \ln k$ | $O(k \log k)$ | $k \log \log^* k$ |

Baseline finds most hops fast, XOR layers __clean up the stragglers__

<v-click>

$\log^* k$ is the iterated logarithm: the number of $\log_2$ applications required until the value drops to $\leq 1$.

$$\log^*(k) = \begin{cases} 0 & k \leq 1 \\ 1 + \log^*(\log_2 k) & k > 1 \end{cases}$$

</v-click>

<v-click>

vs competing approaches (PPM, AMS): __25–36x fewer packets__

For Kentucky Datalink ($D = 59$): PINT needs only __42 packets__ on average

</v-click>

---

## Theorem 3 — Hybrid Complexity

> After $k \log \log^* k \cdot (1 + o(1))$ packets, the multilayer scheme recovers all $k$ switch IDs with high probability.

<v-click>

__Proof sketch__:
- __Baseline__, $\tau \approx 3/4$: each switch appears with prob $\tau$ per packet. After $O(k)$ packets, a constant fraction remain _(Chernoff: expected stragglers $(1-\tau)^{O(k)} \cdot k$ concentrates tightly)_. Those that haven't appeared form the "straggler" set.
- __XOR__: collects equations over the stragglers. When $m$ unknowns remain, the XOR system becomes solvable after $O(m \log m)$ new packets.
- With $\mathcal{L}$ XOR layers: unknowns shrink by a $\log$ factor each layer, giving $O(k \log^{(\mathcal{L})} k)$ total packets.

</v-click>

<v-click>

Set $\mathcal{L} = \log^* k$: the chain $k \to \log k \to \log \log k \to \cdots \to O(1)$ terminates in $\log^* k \leq 5$ steps for any real $k$, yielding the $k \log \log^* k$ bound. In practice $\mathcal{L} = 2$ already achieves near-linear performance.

</v-click>
