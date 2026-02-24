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
Same accuracy as reading all per-hop values, using __1 byte__ instead of 25+

</v-click>

---

## Why Multiplicative, Not Additive?

We can only use 8 bits. How do we encode a 32-bit utilization value?

<v-click>

__Additive__ error $|v - \hat{v}| \leq \varepsilon$: useless here : link speeds span __6 orders of magnitude__ (1 Mbps to 100 Gbps). A fixed $\varepsilon$ is either too coarse at high speeds or too tight at low speeds.

</v-click>

<v-click>

__Multiplicative__ error: decoded $\hat{v} \in \bigl(\tfrac{v}{1+\varepsilon},\ v\bigr]$ : always an underestimate, same __relative__ error at every scale

Store $c = \lfloor \log_{1+\varepsilon} v \rfloor$, decode as $\hat{v} = (1+\varepsilon)^c$

</v-click>

<v-click>

__Formal guarantee__: since $(1+\varepsilon)^c \leq v < (1+\varepsilon)^{c+1}$, we have $\hat{v} \leq v$ and:

$$\frac{|v - \hat{v}|}{v} = 1 - \frac{\hat{v}}{v} \leq 1 - \frac{1}{1+\varepsilon} = \frac{\varepsilon}{1+\varepsilon} < \varepsilon$$

</v-click>

<v-click>

With $\varepsilon = 0.05$: __32 bits $\rightarrow$ 8 bits__ with $< 5\%$ relative error

P4 switches have __no floating-point arithmetic__ : switches precompute $\lfloor \log_{1.05} v \rfloor$ into a __lookup table__ evaluated at line rate, no division or $\log$ at runtime

</v-click>

---

## HPCC Trace with $k=4$

<HpccTrace />
