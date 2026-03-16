In modern ARM-based Systems-on-Chip (SoCs), managing how various hardware IPs (CPUs, GPUs, Display Controllers, NPUs) access shared resources is critical to preventing bottlenecks. This is where Quality of Service (QoS) and latency signaling come into play.
Before diving into the performance impacts, I want to clarify the terminology, as there is an important distinction between standard ARM protocols and vendor-specific implementations.
1. What are AxQoS and AxLat?
 * AxQoS (AWQOS / ARQOS): This is a standardized feature introduced in the ARM AMBA AXI4 specification (and utilized in AXI5 and CHI). It consists of two 4-bit signals: AWQOS for write channels and ARQOS for read channels. It acts as a priority identifier, telling the interconnect and memory controllers how important a specific transaction is compared to others.
 * AxLat: To be perfectly candid, AxLat is not a standard signal defined in the official ARM AMBA specifications. However, in modern SoC design, it is a very common vendor-specific sideband signal (often used in interconnects by Arteris, Sonics, or custom ARM CoreLink implementations). It typically represents a Latency Target, Deadline, or Latency Tolerance Reporting (LTR) metric. While AxQoS dictates priority, AxLat dictates urgency (i.e., exactly how many clock cycles a transaction has before it fails to meet a real-time requirement).
Here is how these two concepts work together to impact the performance of the on-chip NoC and the memory subsystem.
2. Performance Impact on the On-Chip NoC (Network-on-Chip)
The NoC acts as the highway system of the SoC. AxQoS and AxLat dictate how traffic is routed and who gets the right of way at intersections (routers/switches).
 * Arbitration and Routing: At every switch inside the NoC, arbiters look at the AxQoS value. High-QoS traffic (like a Display Controller needing pixels to prevent screen tearing) will immediately pre-empt Best-Effort traffic (like a background DMA transfer).
 * Virtual Channel (VC) Allocation: To prevent Head-of-Line (HoL) blocking—where a low-priority packet blocks a high-priority packet stuck behind it—the NoC maps different AxQoS bands to separate Virtual Channels. This allows high-priority transactions to logically bypass congested physical queues.
 * Dynamic Priority Boosting (Aging): This is where AxLat shines. A transaction might start with a medium AxQoS, but as it sits in NoC buffers, its AxLat counter ticks down. As the transaction approaches its latency deadline, the NoC dynamically boosts its priority, forcing it through the network to meet its real-time requirements.
3. Performance Impact on the Memory Subsystem
The memory subsystem is where QoS and Latency handling become highly complex, as the system must balance latency (serving urgent requests fast) against throughput (serving as many requests as possible).
A. System Level Cache (SLC)
 * Cache Partitioning and MPAM: Modern ARM architectures utilize Memory Partitioning and Monitoring (MPAM). AxQoS can be tied to MPAM to restrict how much of the SLC a specific IP block can consume. For example, a high-QoS real-time process can be guaranteed a locked portion of the SLC.
 * Allocation / Bypassing: High AxLat urgency or specific AxQoS bands can dictate whether data is allocated into the SLC at all. Massive, low-priority streaming writes from a GPU might be forced to bypass the SLC entirely so they do not evict critical, high-priority CPU data.
B. Memory Scheduler
 * Reordering (FR-FCFS vs. EDF): Standard memory schedulers use a "First-Ready, First-Come, First-Served" (FR-FCFS) algorithm. They reorder transactions to maximize DRAM page hits, which maximizes bandwidth but ruins predictable latency.
   * With AxQoS: The scheduler groups requests by QoS bands. It will execute all high-QoS requests (even if it causes page misses and lowers total bandwidth) before looking at lower-QoS requests.
   * With AxLat: The scheduler switches to an "Earliest Deadline First" (EDF) approach. It will happily optimize for bandwidth until a transaction's AxLat signals that it is running out of time, at which point the scheduler drops everything to serve that specific read/write.
 * Read/Write Turnaround: Switching DRAM from reading to writing takes time (bus turnaround latency). The scheduler uses AxQoS to decide when to take this penalty. A high-priority read will force the scheduler to interrupt a batch of writes earlier than it normally would.
C. Memory DRAM Controller (PHY interaction)
 * Page Policy Management: The DRAM controller decides whether to leave a DRAM row open (for throughput) or close it (for latency). If the controller sees an influx of high-AxQoS, latency-sensitive traffic targeting random addresses, it may dynamically switch to a "Closed Page" policy. This ensures that the next urgent transaction doesn't have to wait for an open page to close before it can be served.
 * Refresh Deferral: DRAM requires constant refreshing, which blocks traffic. If a transaction with a critical AxLat deadline arrives, the memory controller can choose to defer opportunistic DRAM refreshes until the urgent transaction is complete.
Accurate Documentation Links
For standard AMBA signaling, refer directly to ARM's documentation:
 * AMBA AXI and ACE Protocol Specification (IHI0022H) - Section A8 details QoS signaling (ARQOS / AWQOS).
 * ARM CoreLink Interconnects - ARM's overview of how these signals are routed in practical CMN/NIC implementations.
Are you currently working with a specific vendor's NoC IP (like ARM CoreLink CMN-700 or Arteris FlexNoC), or are you developing a custom memory controller? I can tailor the implementation details to the specific architecture you are using.
