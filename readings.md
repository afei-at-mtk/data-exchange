You have hit the nail on the head. This is one of the classic architectural dilemmas in modern SoC design: Priority vs. Urgency.
You are absolutely right that AxQoS and AxLat frequently conflict.
 * AxQoS dictates Priority (Importance): "I am the main CPU, my tasks are generally the most important for system responsiveness."
 * AxLat dictates Urgency (Time): "I am the Display Controller. I am usually low-priority, but my buffer is almost empty. If I don't get data in 50 clock cycles, the screen will tear."
If a memory scheduler blindly follows AxQoS, the Display Controller fails its deadline. If it blindly follows AxLat, the CPU stalls every time a background IP gets close to a deadline, tanking your Geekbench or SPEC scores.
Here is how modern NoCs and memory controllers are architected to handle this conflict correctly.
1. Dynamic Priority Escalation (Aging)
The most common way to resolve the conflict is to make the two signals interact dynamically. Instead of treating AxQoS as a static value, the NoC and Memory Controller use AxLat to mutate the QoS over time.
 * How it works: A transaction enters the NoC with a low AxQoS and a specific AxLat countdown. As the transaction sits in a buffer, the AxLat timer ticks down. Once AxLat crosses a critical threshold (the "Panic" threshold), the hardware forcibly overwrites the packet's AxQoS to the highest possible value.
 * The Result: The transaction behaves like normal, low-priority traffic until it absolutely must be served, at which point it pre-empts everything else. This protects overall system bandwidth while preventing real-time deadline failures.
2. Orthogonal Traffic Classes (Isochronous vs. Best-Effort)
To prevent these signals from fighting each other, advanced schedulers divide traffic into distinct logical classes right at the Network Interface (NI), evaluating them differently:
 * Isochronous / Real-Time Traffic (Display, Camera, Audio): For these IPs, the scheduler essentially ignores AxQoS and routes/schedules them strictly based on AxLat (Earliest Deadline First). They are guaranteed a specific bandwidth and maximum latency.
 * Latency-Sensitive / Best-Effort Traffic (CPU, GPU Compute, NPU): For these IPs, there are no hard deadlines. The scheduler ignores AxLat (or it isn't generated) and relies entirely on AxQoS to decide who goes first.
 * The Arbitration: The memory controller will always serve the Best-Effort traffic (maximizing throughput) unless an Isochronous transaction's AxLat indicates it is in danger of starving.
3. Ingress Traffic Shaping (Token Buckets)
A major risk is a high-AxQoS initiator (like a heavily threaded CPU) flooding the NoC and drowning out high-AxLat (urgent) requests from other IPs.
 * To handle this, architects implement Rate Limiters or Token Buckets at the NoC ingress ports.
 * Even if a CPU sends 100 requests with the highest AxQoS, the token bucket restricts how many can enter the NoC per clock cycle. This guarantees that internal NoC buffers always have physical room to accept and route urgent AxLat traffic from peripheral IPs.
4. QoS Banding in the Memory Controller
At the DDR controller level, mixing QoS and Latency creates thrashing (constantly opening and closing DRAM pages, which kills bandwidth). Memory controllers handle this using QoS Banding.
Instead of 16 distinct priority levels, the controller groups AxQoS into 3 or 4 broad bands (e.g., Panic, High, Medium, Low).
 * Inter-Band: The scheduler strictly enforces priority between bands. A "Panic" request will always interrupt a "Medium" request.
 * Intra-Band: Within the same band, AxQoS is ignored. The scheduler looks at AxLat to order the most urgent requests first, and then looks at the physical DRAM addresses to group requests together to maximize page hits and bandwidth.
Summary: The Golden Rule of Tuning
Handling them correctly relies heavily on software tuning. Hardware provides the mechanisms, but the firmware/OS must program the thresholds. The golden rule is: Do not over-allocate high QoS. High AxQoS should be a scarce resource reserved only for the CPU and critical control paths, while AxLat should be strictly calibrated to the actual physical size of the FIFOs (buffers) inside the IP blocks.
