# Evaluation

---
## Does it work?

#### Congestion Control (HPCC)

<v-clicks>

- Similar FCT to full-overhead INT, better goodput for long flows
- Works well even when only $p = 1/16$ of packets carry the query digest
- At 70% load: __71% goodput improvement__ over standard INT

</v-clicks>

#### Path Tracing

<v-clicks>

- Kentucky Datalink ($D=59$): only __42 packets__ avg to trace full path
- Competing methods (PPM, AMS) need __1000-5000+ packets__
- 25-36x fewer packets than state-of-the-art

</v-clicks>

---

## Combined Experiment

<v-click>

Run all 3 queries concurrently with 16-bit global budget (8 bits per query)

</v-click>

<v-clicks>

- HPCC: short flows 6.6% slower, long flows unaffected
- Path tracing: packet count increases only 0.5% vs running alone
- Latency estimation: relative error increases only 0.7%

</v-clicks>

<v-click>

Each packet carries digests for 2 of the 3 concurrent queries

</v-click>

---

# The Good

<v-clicks>

1. __Practical__: implemented in P4, runs on commodity programmable switches
2. __Flexible__: user defines bit budget per use case
3. __Composable__: multiple queries run concurrently within budget
4. __Connects to theory__: distributed coding, coupon collector, sketches

</v-clicks>

---


# The Bad

<v-clicks>

1. __Short flows__: path tracing needs many packets, single-packet flows (common in datacenters) cannot be traced
2. __Probabilistic nature__: never get perfect data for a single specific packet, need aggregation over flow
3. __Route changes__: if route changes mid-flow, decoding becomes complex (flowlet-level tracking needed)
4. __Manual execution plan__: query engine currently requires manual configuration of bit budget allocation

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


---



# Summary

<v-clicks>

- __PINT__ trades immediate precision for bandwidth efficiency using __Distributed Coding__ and __Sketches__
- Enables telemetry in production networks without performance penalties
- 16 bits per packet $\rightarrow$ 3 concurrent queries with near-full-INT visibility

</v-clicks>

<v-click>

Can we apply this to __security__ (DDoS detection) where single-packet precision is critical?

</v-click>

---
layout : end
---

Questions?
