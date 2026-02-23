# Per-packet Aggregation : Congestion Control

Support High Precision Congestion Control (HPCC)

<v-click>

HPCC needs only the __bottleneck__ along path, not every single hop value

</v-click>

<v-click>

__PINT approach__:
- Computes `max(utilization)` in data plane as packet traverses network
- Each switch overwrites the digest only if its value is larger

</v-click>

<v-click>

__What the receiver does with it__: sends the 8-bit bottleneck digest back to the sender, which computes a new rate:
$$\text{rate} \leftarrow \text{BW} \times (1 - U_\text{max})$$
Same accuracy as reading all per-hop values — using __1 byte__ instead of 25+

</v-click>

---

## Compression via Multiplicative Approximation

We can only use 8 bits, how to encode a 32-bit value?

<v-click>

__Logarithmic rounding__: store $c = \lfloor \log_{1+\varepsilon}\, v \rfloor$ instead of $v$

</v-click>

<v-click>

With $\varepsilon = 0.05$: compress __32 bits $\rightarrow$ 8 bits__ with $< 5\%$ error

Switches use __lookup tables__ to compute logs — no floating point needed

</v-click>

<v-click>

Result: similar or better HPCC performance, at a fraction of the overhead

At 70% network load, PINT improves goodput by __71%__ vs full INT overhead

</v-click>

---

## Worked Example

`bit<32>` P4 register: link utilization $v = 75{,}000$ Mbps on a 100 Gbps link (75% load)

<v-click>

**Step 1 — Encode**: $\varepsilon = 0.05$, base $= 1.05$

$$c = \left\lfloor \log_{1.05}(75{,}000) \right\rfloor = \lfloor 230.07 \rfloor = \mathbf{230}$$

</v-click>

<v-click>

**Step 2 — Store**: $230$ fits in **8 bits** (range 0–255) ✓ &nbsp;—&nbsp; was 32 bits in the header

</v-click>

<v-click>

**Step 3 — Decode**: $1.05^{230} \approx \mathbf{74{,}737}$

$$\text{error} = \frac{75{,}000 - 74{,}737}{75{,}000} \approx 0.35\% \ll 5\% \checkmark$$

</v-click>

<v-click>

**Savings**: 4 bytes → 1 byte per hop in the packet header

</v-click>

---
<HpccTrace />
