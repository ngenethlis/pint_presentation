# Dynamic Per-flow Aggregation : Latency Estimation

Tracking metrics that change packet-by-packet

Unlike static switch IDs, metrics like **queueing latency** vary for every single packet. 

<v-click>

**The Goal**: Estimate median and tail latency for an entire flow, without logging every hop of every packet.

</v-click>

<v-click>

**Why focus on Tail Latency?**
- The **P99** (99th-percentile) latency drives SLA (Service Level Agreements) violations.
- A single congested switch can ruin the performance of an entire flow. We need to find the culprit.

</v-click>

<v-click>

**How is this different from Congestion Control?**
- **Congestion Control:** Needs the single worst bottleneck *per packet*. Fits in one digest.
- **Tail Latency:** Requires building a statistical *distribution* across thousands of packets. You can't fit a distribution into a single header!

</v-click>

---

## Mechanism

A three-step pipeline using bounded overhead

<v-clicks>

1. **In-Network Sampling (Reservoir Sampling)**
   - Switch at hop $i$ writes its latency into the digest with probability $\frac{1}{i}$; later hops write less often.
   - Despite unequal write probabilities, every hop survives as the final sample with probability exactly $\frac{1}{k}$: the telescoping product $\frac{1}{i} \cdot \frac{i}{k} = \frac{1}{k}$ cancels out the bias.

2. **End-Host Aggregation**
   - The receiver extracts the winning latency sample from each arriving packet.
   - These continuous samples are fed directly into a **quantile sketch**.

3. **The Result**
   - Accurate P99 estimates achieved with strictly bounded header space (1 value per packet).
   - Highly memory-efficient for the receiver.

</v-clicks>

---

## Connection to Sketches

<v-click>

PINT feeds the extracted samples into **KLL (Karnin-Lang-Liberty) sketches** at the end-host.

</v-click>

<v-clicks>

- **Mergeable Quantile Tracking**
  - Estimates any target quantile $\phi$ (e.g., $\phi = 0.99$ for P99 latency).
  - Guarantees an error bound of $\pm \varepsilon$ using only $O(\varepsilon^{-1} \log \frac{1}{\delta})$ space.
  - *Answers:* "What is the current P99 latency of this flow?"

- **Sliding Window Adaptation**
  - Network conditions are highly volatile.
  - The sketch only tracks the last $W$ packets, ensuring estimates reflect *current* bottlenecks rather than stale historical data.

- **Fast Convergence**
  - The sketch stabilizes rapidly; accurate latency distributions emerge after receiving just ~400 packets.

</v-clicks>

---

### Reservoir Sampling: Worked Example ($k = 5$)

Path: $S_1 \to S_2 \to S_3 \to S_4 \to S_5$ with latencies 2ms, 8ms, 3ms, 1ms, 12ms

<P99Trace />
