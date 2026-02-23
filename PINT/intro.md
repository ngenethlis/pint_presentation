# What is network telemetry

Collect & analyze measurements to understand what is happening in the network for operation and management

<v-clicks>

- Switch ID, latency, queue depth, port utilization
- Used for: congestion control, path tracing, fault localization, load balancing

</v-clicks>

[SIGCOMM'20 Network Telemetry](https://www.youtube.com/watch?v=VzbV0ceRAVU)


---

# In-Band Network Telemetry

Switches add metadata
`ID, latency, queue depth` to every packet as it traverses the network

<v-click>

## See the issue?

</v-click>

<v-click>

Overhead grows __linearly__ with hop count

</v-click>

<v-click>

5 hops, 3 INT values per hop $\rightarrow$ 48B overhead (4.8% of 1KB packet)

</v-click>

<v-click>

FCT increases by up to __25%__, goodput degrades by __20%__

Processing latency: 48B INT header adds ~76ns per hop on 10G interfaces

</v-click>


---

# PINT

Do we need __all__ the data? __all__ the time? __perfectly__?

<v-click>

Bound the overhead, spread information across multiple packets using
__Probabilistic Encoding__

</v-click>

<v-clicks>

- User defines a __bit budget__ (as low as 1 bit per packet)
- Approximate telemetry across a __collection__ of packets
- Visibility comparable to full INT, fraction of the cost

</v-clicks>


---

# Coding & Aggregation

Switches are stateless, how do we coordinate which one writes?

<v-click>

__Global hashing!__

</v-click>

<v-click>

All switches know a hash function $g(\text{packet\_ID})$

Independently decide whether to write and what to write based on hash output

</v-click>

<v-click>

Three aggregation modes:

</v-click>

<v-clicks>

1. **Per-packet**: find bottleneck (e.g. `max(utilization)`) $\rightarrow$ congestion control
2. **Static per-flow**: recover switch IDs via distributed coding $\rightarrow$ path tracing
3. **Dynamic per-flow**: reservoir sampling + sketches $\rightarrow$ tail latency estimation

</v-clicks>

---

# Global Hashing in Action

Baseline path tracing: switch $i$ writes if $g(\text{pkt}, i) < \tfrac{1}{i}$ (reservoir sampling)

<PacketTrace />
