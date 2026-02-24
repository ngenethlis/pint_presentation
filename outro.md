# Evaluation

<v-clicks>

#### Congestion Control (HPCC)


- Similar FCT to full-overhead INT, better goodput for long flows
- Works well even when only $p = 1/16$ of packets carry the query digest
- At 70% load: __71% goodput improvement__ over standard INT

</v-clicks>


<v-clicks>

#### Path Tracing


- Kentucky Datalink ($D=59$): only __42 packets__ avg to trace full path
- Competing methods (PPM, AMS) need __1000-5000+ packets__
- 25-36x fewer packets than state-of-the-art

</v-clicks>

<v-clicks>

#### Tail / Median Latency Estimation

- P50 and P99 relative error stays __within configured $\varepsilon$__ using KLL sketch
- Sliding-window sketch (W = 400) stabilises within one window length
- Tested on CAIDA traces: both median and tail accurately recovered with __8-bit budget__

</v-clicks>

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
