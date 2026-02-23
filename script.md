# Presentation Script — PINT (SIGCOMM 2020)

Bullet points per slide. Italics = things to say out loud. Plain = reminders.

---

## Slide: What is Network Telemetry

- _"Network telemetry means collecting measurements from inside the network — switch IDs, queue depths, port utilisation, latency — so operators can actually see what's happening."_
- _"It powers congestion control, fault localisation, path tracing, load balancing. Without it you're flying blind."_

---

## Slide: In-Band Network Telemetry (INT)

- _"The standard approach is called INT — every switch stamps its metadata directly into every packet as it passes through."_
- _"The problem is obvious once you see the numbers: five hops, three values per hop — that's 48 bytes of overhead on a 1 KB packet. Nearly 5%."_
- _"In practice that translates to a 25% increase in flow completion time and 20% goodput degradation. That's a big price to pay for visibility."_

---

## Slide: PINT — The Key Idea

- _"PINT asks: do we actually need all the data, all the time, from every packet, perfectly? The answer is usually no."_
- _"Instead, PINT lets the operator set a bit budget — as low as one bit per packet — and spreads telemetry information probabilistically across many packets."_
- _"Even one bit is useful. You can encode a threshold signal: 'is utilisation above 80%?' Or contribute one sample to a path trace."_
- _"It's implemented entirely in P4 — a programming language for network ASICs — so it runs at line rate inside the switch with zero CPU involvement."_

---

## Slide: PINT Architecture

- _"PINT has three layers. The data plane lives entirely inside each switch — no extra hardware, no CPU. Each switch evaluates a hash on the packet ID and its own switch ID. If the hash falls below a threshold, it writes into the digest field in the packet header. Otherwise it passes the digest unchanged."_
- _"The digest field is the only overhead — 8 to 16 bits added to the packet header. It accumulates information as the packet traverses the path."_
- _"At the other end, the receiver collects these digests over many packets and runs the decoding — Gaussian elimination for path tracing, sketch updates for latency."_
- _"The query engine sits above all of this. The operator defines what they want to measure and how many bits to spend. That compiles to P4 and gets deployed to the switches. Crucially, switches need no per-flow state and no coordination with each other — each decision is made independently using only the hash."_

---

## Slide: Coding & Aggregation — Global Hashing

- _"The central mechanism is global hashing. Every switch knows the same hash function g(packet_ID, switch_ID). Each switch independently evaluates it and decides whether to write. No coordination, no shared state."_
- _"This gives you three aggregation modes depending on what you're measuring."_
- _"Per-packet: compute a running max of utilisation — that's congestion control."_
- _"Static per-flow: switch IDs don't change, so you can recover the full path — that's path tracing."_
- _"Dynamic per-flow: latency changes every packet, so you need sketches — that's tail latency."_

---

## Slide: Global Hashing in Action (PacketTrace animation)

- _"This animation shows reservoir sampling. Switch at hop i writes its value if the hash is below 1/i — so the first hop always writes, the second writes with probability half, and so on."_
- _"The key property: whichever hop's value survives to the receiver, it was equally likely to be any of the k hops. That's the uniform coverage guarantee."_
- Step through the animation — let it speak.

---

## Slide: Per-packet Aggregation — HPCC

- _"For congestion control, HPCC only needs the bottleneck — the single worst link along the path. It doesn't need every hop's value."_
- _"PINT exploits this: each switch checks if its utilisation is larger than what's already in the digest. If yes, overwrite. If no, leave it."_
- _"The receiver gets the maximum in one byte instead of 25 or more. Same accuracy, drastically less overhead."_

---

## Slide: Why Multiplicative, Not Additive?

- _"We have eight bits — how do you encode a 32-bit utilisation value in that?"_
- _"An additive error bound doesn't work. Link speeds span six orders of magnitude — 1 Mbps to 100 Gbps. A fixed epsilon is either too coarse for fast links or unnecessarily tight for slow ones."_
- _"Multiplicative error gives the same relative precision everywhere. Store the floor of log base (1+ε) of v, decode as (1+ε)^c."_
- _"The formal guarantee: the relative error is at most ε/(1+ε), which is strictly less than ε. At ε=0.05 that's less than 5% relative error with 8 bits."_
- _"One practical constraint: P4 switches have no floating point. The log is precomputed into a lookup table, evaluated at line rate."_

---

## Slide: HPCC Worked Example (HpccTrace animation)

- _"Let's trace a packet through five switches. Each switch computes its encoded utilisation — that's the floor of log_{1.05} applied to the raw value."_
- _"If the encoded value is larger than what's in the digest, overwrite. Otherwise pass it on."_
- _"At the receiver you decode: raise 1.05 to the stored exponent. The error is within the 5% bound."_
- Step through — highlight the encode/compare/write decision at each hop.

---

## Slide: Distributed Coding — Path Tracing

- _"Path tracing is a different problem. Switch IDs are static — they don't change between packets for the same flow. We want to recover all k switch IDs."_
- _"This matters for fault localisation — which switch is dropping packets? — and routing verification — is traffic following the intended path?"_
- _"Key idea: distribute the switch IDs across many packets. No single packet needs to carry everything."_

---

## Slide: Technique 1 — Baseline (Coupon Collector)

- _"The simplest scheme: each switch writes its ID with probability 1/k. Think of it as collecting coupons — you need one of each."_
- _"The expected number of packets is k times the k-th harmonic number. H_k is just 1 + 1/2 + 1/3 + ... + 1/k — it grows like ln k, so you need about k ln k packets in total."_
- _"For the tail: the probability a specific switch hasn't been written after t packets is (1 - 1/k)^t. By the Chernoff inequality — using 1-x ≤ e^{-x} — that's at most e^{-t/k}."_
- _"Union bound over all k switches: probability any switch is missing is at most k·e^{-t/k}. Set t = c·k·ln k and that becomes k^{1-c}, which goes to zero for any c greater than 1."_
- _"The problem: the last few switches dominate. The k-th coupon alone takes k packets in expectation. That's the long tail."_

---

## Slide: Baseline Worked Example (BaselineTrace animation)

- _"Watch the progress bar at the top — switches light up green as they're recovered."_
- _"Notice duplicates accumulating. That's the coupon collector in action."_
- Step through — point out the NEW vs DUPLICATE distinction and the final packet count.

---

## Slide: Technique 2 — Distributed XOR

- _"The XOR scheme is smarter. Instead of writing one switch ID, each switch XORs its ID into the digest with some probability."_
- _"Each packet becomes a linear equation over F_2 — the two-element field where addition is XOR and 1+1=0. The digest is the XOR of whichever switches participated."_
- _"The receiver knows which switches participated — it can evaluate the same hash — so it has the equation system. Gaussian elimination over F_2 solves it."_
- _"The power: equations share information. If you know S3 XOR S4 from one packet and S4 from another, you get S3 for free. Baseline requires each switch to be written independently."_

---

## Slide: XOR Worked Example (XorTrace animation)

- _"Watch the equation system build up on the right. An equation with one unknown can be solved immediately, then substituted back into the others — peeling off unknowns one by one."_
- Step through — emphasise the back-substitution steps.

---

## Slide: Hybrid Approach

- _"Pure baseline is simple but needs k ln k packets. Pure XOR is more efficient but every packet contributes only one equation — can be slow to start."_
- _"Hybrid interleaves both. A Baseline layer at probability τ = 3/4 catches the easy isolated hops quickly. XOR layers clean up the stragglers."_
- _"In practice 1 to 2 XOR layers are enough. The worked example recovers all 5 switches in 6 packets versus about 12 expected for pure baseline."_

---

## Slide: Comparison Table (k = 25)

- _"For k=25, pure Baseline needs 89 packets median, 189 at P99. Hybrid needs 41 median and 68 at P99 — about half."_
- _"The complexity column: Baseline is k ln k, XOR is O(k log k), Hybrid is k log log* k — essentially linear for any realistic k."_
- _"Against competing approaches like PPM and AMS: 25 to 36 times fewer packets."_

---

## Slide: log* — Iterated Logarithm

- _"log* k asks: how many times must you apply log_2 before the value drops to 1 or below?"_
- _"For k = 2^65536 — a number vastly larger than atoms in the observable universe — log* is 5. For any network you'll ever encounter, log* k is at most 5."_
- _"So k·log·log* k is effectively k times a tiny constant. That's why the paper calls it essentially linear."_

---

## Slide: Theorem 3 — Hybrid Complexity

- _"The theorem says: after k·log·log* k · (1+o(1)) packets, the hybrid scheme recovers all k switch IDs with high probability."_
- _"The proof goes like this. Baseline at rate τ = 3/4: after O(k) packets, each switch has appeared independently with probability τ. By the Chernoff bound, the number of stragglers concentrates tightly around (1-τ)·k — a constant fraction."_
- _"Each XOR layer then halves the number of unknowns on a log scale: m unknowns → O(m log m) equations needed → solvable, leaving log m unknowns."_
- _"Stack log* k layers: k → log k → log log k → ... → O(1). That's where the log log* k comes from. In practice L=2 is already near-linear."_

---

## Slide: Dynamic Per-flow — Tail Latency

- _"Third use case: latency. Unlike switch IDs, latency changes every packet — you can't use a static encoding."_
- _"Goal: estimate the median and P99 latency per flow without storing every measurement."_
- _"Why tail latency? P99 drives SLA violations. A single slow switch causes the tail for many flows. Knowing which switch is the culprit lets you fix the right thing."_

---

## Slide: Mechanism

- _"Step one: reservoir sampling. Switch at hop i writes its latency with probability 1/i. The result is that any hop's value is equally likely to survive — an unbiased sample."_
- _"Step two: the receiver feeds these samples into a KLL quantile sketch — a mergeable structure that estimates the φ-quantile using O(ε^{-1} log 1/δ) space. The Õ hides log factors."_
- _"A sliding window of width W ensures estimates reflect current behaviour — old samples age out."_
- _"P99 means φ = 0.99: 99% of packets are below this latency. The 1% above it is the tail you're tracking."_

---

## Slide: Connection to Sketches

- _"KLL sketch: you've seen quantile sketches in the course. This one is mergeable — individual switch estimates can be combined at the controller."_
- _"Sliding window: only the last W packets contribute. For a busy flow W=400 samples gives stable P50 and P99 estimates."_
- _"Error converges as samples accumulate — the sketch guarantee is within ε of the true quantile with probability 1-δ."_

---

## Slide: P99Trace Animation

- _"Each step is one packet arriving. The surviving hop's latency is added to the sample buffer shown at the top."_
- _"The bar chart on the right shows how many times each switch has won the reservoir sampling. S5 at 12ms accumulates disproportionately — it's the high-latency hop."_
- _"P99 stabilises at 12ms from the second sample onwards and correctly identifies S5 as the bottleneck."_
- Step through — point out P99 banner appearing on the right panel.

---

## Slide: Evaluation — Congestion Control

- _"At 70% load, PINT with HPCC gives 71% goodput improvement over standard INT. Flow completion time is comparable to full-overhead INT — you get the same control quality for a fraction of the cost."_
- _"Even when only 1 in 16 packets carries the query, the accuracy holds."_

---

## Slide: Evaluation — Path Tracing

- _"On the Kentucky Datalink topology with D=59, PINT needs 42 packets on average to recover the full path. PPM and AMS need 1000 to 5000 packets. That's a 25 to 36 times reduction."_

---

## Slide: Evaluation — Tail/Median Latency

- _"P50 and P99 estimates stay within the configured ε after a window of about 400 samples. Tested on CAIDA traces with an 8-bit budget."_

---

## Slide: Combined Experiment

- _"Running all three queries concurrently with a 16-bit global budget — 8 bits each — the interference is minimal."_
- _"HPCC short flows slow by 6.6%, long flows unaffected. Path tracing packet count up 0.5%. Latency error up 0.7%. Each packet carries two of the three queries."_

---

## Slide: The Good

- _"Practical: deployed in P4 on commodity programmable switches. This is not a simulation — it runs on real hardware."_
- _"Flexible: the operator picks the bit budget. Different queries can have different allocations."_
- _"Composable: multiple queries share the same packet header space without interfering."_

---

## Slide: The Bad

- _"Short flows: path tracing fundamentally needs many packets. A single-packet flow — common in datacentres — gives you nothing."_
- _"Probabilistic: you never get certainty about one specific packet. You get statistics over a flow."_
- _"Route changes: if the path changes mid-flow, XOR equations from the two routes reference different switch sets. The system becomes inconsistent and decoding fails. Flowlet-level tracking can mitigate this."_
- _"Manual execution plan: bit budget allocation currently requires manual configuration per use case."_
- _"Security: volume-based DDoS could in principle be detected from aggregate flow statistics. But low-rate or evasive attacks depend on spotting individual malicious packets — exactly what probabilistic sampling cannot guarantee."_

---

## Slide: Accuracy vs Overhead — OmniMon

- _"PINT assumes you must sacrifice accuracy for performance. OmniMon (also SIGCOMM 2020) questions that assumption."_
- _"OmniMon splits the work rather than the data: end-hosts count, switches track flow state, a controller merges everything. Zero error."_
- _"PINT wins on bandwidth. OmniMon wins on precision. They occupy different points in the tradeoff space — neither dominates."_

---

## Slide: Summary

- _"PINT encodes telemetry probabilistically — global hashing coordinates switches without shared state, distributed coding recovers path information, quantile sketches estimate latency distributions."_
- _"16 bits per packet is enough for three concurrent queries with near-full INT visibility and no measurable performance penalty."_
- _"The core insight: you rarely need perfect data about every packet. Statistical guarantees over flows are sufficient for most network management tasks."_
