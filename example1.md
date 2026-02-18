# Per-packet Aggregation : Congestion Control

Support High Precision Congestion Control

HPCC needs only the bottleneck along path, not every single hop value

__PINT__ 
- computes `max(utilization)` in data plane as packet traverses network
- Uses Multiplicative Approximation (logs) to compress the value
- Fast, switches uses lookup tables to compute logs, compressing 32bits to 8bits

<quote Similar or better performance, while minimizing overhead \>

<!-- compression example animation -->

