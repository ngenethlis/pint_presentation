# What is network telemetry

Collect & analyze measurements to understand what is happening in the network for operation and management 

[SIGCOMM'20 Network Telemetry](https://www.youtube.com/watch?v=VzbV0ceRAVU)


---
# In-Band Network Telemetry

Switches add metadata 
`ID, latency, queue depth` to every packet


<!-- animate this to appear -->
## See the issue?

Overhead grows __linearly__ with hop count

5 hops, 2 values per hop -> 48 bytes overhead. 4.8% of 1KB packet.

Degrades goodput by 20%, FCT increases by 25%


---

# PINT

Do we need all the data? all the time? perfectly?

Bound the overhead, spread information accross multiple packets using 
__Probalistic Encoding__


---

# Coding & Aggregation

Switches are stateless, how do we coordinate which one writes?

Global hashing!

All switches know a hash function $g(packet_ID)$,
Independetly decide whether to write and what to write based on hash output

<!-- animation of packet going through switches and each one decides when to write -->

