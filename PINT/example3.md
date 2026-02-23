# Dynamic Per-flow Aggregation

## Sketches & Latency

Values change across packets (e.g. latency varies per packet per switch)

Goal: estimate __median/tail latency__ across a flow without storing every value

---

## Mechanism

<v-clicks>

1. **Distributed sampling**: switches use reservoir sampling via global hash $g(p_j, i) \leq r_i$
   - Each hop writes with $\Pr = 1/i$, later hops less likely to overwrite
2. **End-host aggregation**: collected samples fed into a __quantile sketch__ (KLL)
   - Bounded memory: $\tilde{O}(k \varepsilon_a^{-1})$ per-flow storage
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

- KLL sketch for quantile estimation: given $O(k\varepsilon^{-2})$ packets, produces $(\phi \pm \varepsilon)$-quantile
- Sliding-window sketches to track only recent measurements
- Sketch error converges as more packets arrive, stable after ~400 packets

</v-clicks>
