# Dynamic per-flow Aggregation.

## Sketches & Latency

Remember sketches?

Calculate median latency accross flow without storing every value

__PINT__ uses P4-compatible sketches

## Mechanism

- Switches use resevoir sampling (via hasing) to pick random value to write
- End-host aggregates these into a quantile sketch to estimate median/tail latency

We get: accurate stats with bounded memory and bandwidth
