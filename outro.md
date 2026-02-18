# Evaluation and Critiques

---
## Does it work?

#### Congestion Control HPCC

__PINT__ achieves:
- similar FCT to full-overhead INT
- better goodput for long flows since it saves bandwidth

#### Path tracing

__PINT__ outperforms other approaches (e.g PPM, AMS) by requiring 7-10x less packets

#### Latency

---

# The good

1. Works, written in P4, runs on commodity hardware
2. Flexible, user defines bit budget that works for their usecase
3. Programable querries

---


# The bad

1. Short flows: Path tracing needs __many__ packets, signle packet flows (common in datacenters) cannot be traced

2. Probablistic Nature: Never get perfect data for single specific packet, need aggregation

3. Routing challenges: If route changes mid-flow decoding becomes complex

---
# Accuracy vs Overhead isn't an absolute tradeoff

OmniMon, achieves full accuracy (0 error) while still being resource efficient,
instead of splitting data into packets, it splits the work: hosts count, switches track flows and central controller merges the data.

PINT saves more Bandwidth, Omnimon wins on precision.

PINT operates on the assumption that we __must__ sacrifice accuracy for performance, but other architectures prove you can have accuracy if you involve end-hosts


---



# Summary

__PINT__ trades immediate precision for bandwidth efficiency using Distributed Coding and Sketches.

Enables telemetry in production networks without performance penalties

Can we apply tihs to security (DDoS detection) where single packet precision is critical?

---
layout : end
---

Questions?
