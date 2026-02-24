# Evaluation

Tested on real topologies and traffic traces (CAIDA, Kentucky Datalink)

<v-click>

**Congestion Control (HPCC)**
- Flow completion time matched full-overhead INT; goodput better for long flows
- Sampling only $\tfrac{1}{16}$ of packets is sufficient — most overhead eliminated
- **71% goodput improvement** over standard INT at 70% link load

</v-click>

<v-click>

**Path Tracing**
- Kentucky Datalink ($D = 59$ hops): **42 packets** on average to recover the full path
- Competing methods (PPM, AMS) require 1,000–5,000+ packets: a **25–36× reduction**

</v-click>

<v-click>

**Tail / Median Latency**
- P50 and P99 error within configured $\varepsilon$ across all tested flows
- Sliding-window sketch ($W = 400$ packets) stabilises within one window length
- 8-bit budget sufficient for accurate quantile recovery on CAIDA traces

</v-click>

---

# The Good

<v-clicks>

1. __Practical__: implemented in P4, runs on commodity programmable switches
2. __Flexible__: user defines bit budget per use case
3. __Composable__: multiple queries run concurrently within budget

</v-clicks>

---


# The Bad

<v-clicks>

1. __Short flows__: path tracing needs many packets, single-packet flows (common in datacenters) cannot be traced.
2. __Probabilistic nature__: never get perfect data for a single specific packet, need aggregation over flow
3. __Route changes__: if the route changes mid-flow, XOR equations from the two paths reference different switch sets, the system becomes inconsistent and decoding fails.
<!-- (Requires __flowlet-level tracking__: treat each consecutive burst on the same route as a separate sub-flow and reset the equation system on route change) -->
4. __Manual execution plan__: query engine currently requires manual configuration of bit budget allocation.
5. __Security applications__: volume-based DDoS (SYN floods, amplification) could be detectable via flow-level aggregates.  Low-rate or evasive attacks depend on spotting individual malicious packets, which probabilistic sampling misses by design.

</v-clicks>

---

# Accuracy vs Overhead: not the only tradeoff

<v-click>

__OmniMon__ achieves full accuracy (0 error) while being resource efficient

Instead of splitting data into packets, it splits the __work__: hosts count, switches track flows, central controller merges

</v-click>

<v-click>

PINT saves more __bandwidth__; OmniMon wins on __precision__

</v-click>

<v-click>

PINT assumes we __must__ sacrifice accuracy for performance, but other architectures prove you can have accuracy if you involve end-hosts

</v-click>


[OmniMon SIGCOMM'20](https://dl.acm.org/doi/abs/10.1145/3387514.3405877)

---



# Summary

<v-clicks>

- __PINT__ trades immediate precision for bandwidth efficiency using __Distributed Coding__ and __Sketches__
- Enables telemetry in production networks without performance penalties
- 16 bits per packet $\rightarrow$ 3 concurrent queries with near-full-INT visibility

</v-clicks>

---
layout : end
---

Questions?
