# Dynamic Per-flow Aggregation

## Sketches & Latency

Values change across packets (e.g. latency varies per packet per switch)

Goal: estimate __median/tail latency__ across a flow without storing every value

<v-click>

__Why useful?__ Tail latency drives SLA violations — a single slow switch causes pain for many flows. Knowing _which switch_ is the culprit enables targeted fixes.

(__P99__ = 99th-percentile latency: 99% of packets arrive within this time; the other 1% are "the tail")

</v-click>

<v-click>

__How is this different from congestion control?__
- CC: one max value _per packet_ (bottleneck), single hop
- Tail latency: need a _distribution_ across many packets → can't fit in one digest → need sketches

</v-click>

---

## Mechanism

<v-clicks>

1. **Distributed sampling**: switches use reservoir sampling via global hash $g(p_j, i) \leq r_i$
   - Each hop writes with $\Pr = 1/i$, later hops less likely to overwrite
2. **End-host aggregation**: collected samples fed into a __quantile sketch__ (KLL)
   - Bounded memory: $\tilde{O}(k \varepsilon_a^{-1})$ per-flow storage ($\tilde{O}$ hides polylogarithmic factors)
3. **Result**: accurate median/tail latency estimates with bounded bandwidth

</v-clicks>

---

## Connection to Sketches

<v-click>

Remember sketches from the course?

</v-click>

<v-click>

PINT uses __P4-compatible sketches__ at the end-host to aggregate samples

</v-click>

<v-clicks>

- **KLL sketch** (Karnin-Lang-Liberty): mergeable quantile sketch — given latency samples, returns the $(\phi \pm \varepsilon)$-quantile using $O(\varepsilon^{-1} \log \frac{1}{\delta})$ space ($\phi \in [0,1]$ is the target quantile; $\phi = 0.99$ gives P99). Tells you: _"P99 latency is X ms"_
- **Sliding-window sketch**: only keeps measurements from the last $W$ packets, so estimates reflect _current_ behaviour, not the full historical average
- Sketch error converges as more packets arrive; stable after ~400 samples

</v-clicks>

---

### Reservoir Sampling: Worked Example ($k = 5$)

Path: $S_1 \to S_2 \to S_3 \to S_4 \to S_5$ with latencies 2ms, 8ms, 3ms, 1ms, 12ms

<LatencyTrace />
