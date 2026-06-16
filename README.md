# Multi-WAN QoS: PCC Queue Trees with CAKE Smart Queue Management
### Deployed a Multi-Wan Layer 3 Per Connection Classifier, Centralized Global Queue Trees, and CAKE AQM to Eradicate Bufferbloat, Prevent per connection Bandiwdth Hogging, and Prioritize traffic based on Video, Voice, & Bulk

   > My Third Project.

---

## The Topology

<img width="1330" height="832" alt="TOPOLOGY2" src="https://github.com/user-attachments/assets/e9e3c65c-88b6-46a6-ad88-86bc2477cb42" />

---

## The Lore

Okay so here's the scenario:

Setting up QoS on a single internet link? Straightforward.

But when you introduce Active-Active Dual-WAN with PCC load balancing? Standard queuing completely falls apart.

Simple Queues can't easily track which ISP a packet is using. If the router blindly shapes traffic without knowing which link it's traveling through, it will over-subscribe a slower link or under-utilize a faster link. Bufferbloat spikes across both connections.

Here's the problem with PCC alone:

PCC is stateless. It hashes connections based on mathematical IP fields. It has no idea about bandwidth speeds, link congestion, or buffer states.

If you run unequal links, say a fast link and a slower link a blind 50/50 PCC hash will cause the slower provider's modem buffer to immediately overflow.

I needed PCC to be smart. I needed it coupled with advanced queuing.

So I engineered a centralized traffic framework that binds my existing PCC Load Balancing project with Nested Global Queue Trees and CAKE AQM:
1. Bandwidth aware PCC allocation, It ensures that once a connection is assigned to a provider interface, all corresponding packets stay stuck to that interface to prevent broken sessions.
   
2. Once the Queue Tree catches the congestion at that any of the two upstream ISP links, CAKE steps in. It isolates the different traffics and prioritizes in this order VOICE > VIDEO > BULK/DOWNLOAD, therefore crtitical traffics will be prioritized.
   
3. Dynamic Bandwidth allocation provided from the baseline of 15M download/ 15M upload ensures no one user can hogged the bandwidth all for itself.

The result? Intelligent path shaping that keeps critical applications running even when a single provider link is pushed to its absolute limits.

---

## Performance Highlights

**Centralized Global Bandwidth Pooling**  
Engineered a nested queue tree architecture leveraging `parent=global` to enforce structural download (15M) and upload (15M) constraints across the core routing engine.

**Multi-Stage Packet Marking**  
Developed a granular Mangle framework that pairs PCC connection states with application priority markers, creating a deterministic packet for queue tree parsing.

**Single-Link Bufferbloat Eradication**  
Validated CAKE AQM capabilities under maximum link saturation, maintaining a rock-solid Bufferbloat Grade A+ rating during multi-host stress tests.

**Dynamic Priority Tin Allocation**  
Leveraged CAKE's internal multi-tin profiling inside the global parent queue tree to give voice, video, and background data transfers/bulk their own optimized transit paths.

---

## The Proof: Validation Tests (Note these tests are done with the other ISP turned off to showcase the QOS)

### Test 1: Jitter Under Load (CAKE vs No CAKE)

**What I did:** Evaluated CAKE's performance during heavy link saturation on the active ISP line. Used Ookla Speedtest CLI on a Kali Linux endpoint. Ran tests first with CAKE disabled, then with CAKE enabled. Logged metrics including idle latency, loaded latency, jitter, packet loss, and throughput.

**What happened:** 
- **CAKE disabled:** Logs showed severe queue congestion. Jitter spiked!
- **CAKE enabled:** Router assumed control over the link bottleneck. Jitter dropped!


<img width="1607" height="641" alt="CAKE1" src="https://github.com/user-attachments/assets/1bffca79-6c4a-4d7d-a174-bf7086251e17" />


https://github.com/user-attachments/assets/1e5c3df8-081d-4211-a4e7-8747003a22f8


---

### Test 2: Fair Queueing and Host Isolation

**What I did:** Two independent LAN endpoints (Kali-1 and Kali-2) simultaneously initiated unthrottled, multi-threaded bandwidth tests via Fast.com. Created deliberate line-capacity conflict, one machine dominates the bandwidth!

**What happened:** 
- Single host: Consumed full 15 Mbps download boundary
- Second host started: CAKE's flow-isolation hashing engine intercepted packets. Instead of first host monopolizing the channel, CAKE mathematically split the link. Both endpoints settled with almost a fair bandwidth.

  

https://github.com/user-attachments/assets/a3d07b2b-5480-4fd1-9d13-338522561e4d



**The win:** No "bad neighbor" starvation. Every host gets its fair share.

| Endpoint Device | Single-Host Throughput | Contended Throughput | Status |
|----------------|------------------------|---------------------|--------|
| Kali-1 Station | 14.8 Mbps | ~6 Mbps | Pass |
| Kali-2 Station | 0 Mbps (idle) | ~8 Mbps | Pass |

---

### Test 3: Bufferbloat Benchmark Validation

**What I did:** Executed full-cycle downstream and upstream saturation tests using the standardized Waveform Bufferbloat engine on Kali-1 terminal web environment. Benchmarking loaded latency against idle ping states.

**What happened:** With CAKE queues active under the global parent tree, the router checked packet buffering at the interface boundary. Final scorecard returned a perfect **Bufferbloat Grade A+**.

**WITH CAKE:**

https://github.com/user-attachments/assets/9cee815a-b62b-4f72-bed4-e9a089a5f5aa

**WITHOUT CAKE:**


https://github.com/user-attachments/assets/c07275c8-bc01-44b8-bb31-949cb8dfb9df



---

### Test 4: DiffServ4 Traffic Prioritization Validation and Latency Test

**What I did:** Validated CAKE's internal 4-tin DiffServ architecture. Generated three simultaneous long-duration traffic flows from Kali Linux using ICMP streams with explicit DSCP markers: EF (DSCP 46 / Voice), AF41 (DSCP 34 / Video), and CS1 (DSCP 8 / Bulk). Used MikroTik Torch on the WAN interface to verify DSCP header visibility. Logged latency, jitter, and packet loss per class.

**What happened:** Torch confirmed all DSCP headers remained intact across the external interface. CAKE successfully mapped packet classes into designated priority tins under full saturation. CAKE strictly honors DiffServ4 service priorities during heavy contention.

- **EF (Voice Tin):** Lowest latency floor. Highest Priority
- **AF41 (Video Tin):** Middle Latency. Second Highest Priority.
- **CS1 (Bulk Tin):** Highest latency degradation. Lowest Priority.


**WITH CAKE:**

<img width="1397" height="1021" alt="CAKE2" src="https://github.com/user-attachments/assets/b5f9eea4-3a9d-44d1-964e-54d6ed3c9d03" />

**WITHOUT CAKE:**

<img width="1291" height="917" alt="CAKE3" src="https://github.com/user-attachments/assets/ed1b2ec0-c2d4-4bb9-a9ac-3f58c8f614f2" />

---

## Engineering Challenges

**Manual Prioritization Problem**

Manually prioritizing certain apps/softwares takes too much time and stress on the cpu, altho cake sure is cpu intensive, manually prioritizing traffics based on
ports are inaccurate since one session can open up multiple connections as we see on the PCC project. 

**How I solved it:** Utilized Cake, here's the thing Cake is advised by Mikrotik to use on wan links not utilizing the full 1Gbs link, otherwise use FQ Codel.
Cake unlike FQ-Codel tho treats each connection fair, like what we see on the PCC project one session can have so many connections, downloading something on the torrent can open up multiple connections with just one host, therefore your other connections will be compromised, sure per user fairness is achieved, but per user connection was not, is my point, that's why I utilized Cake instead of FQ-Codel, but for fallback and more than 1Gbps WAN link FQ-Codel  for sure.


---

## Final Thoughts

Multi-WAN traffic engineering is the absolute peak of network edge optimization.

Deploying multiple internet connections with simple, untracked queues? Letting your providers' modems handle link congestion? That's a disaster. It leads to uneven link utilization, broken voice calls, and severe bufferbloat.


## The Proposal

<img width="1462" height="835" alt="Topology" src="https://github.com/user-attachments/assets/5de0f718-cdc9-4a42-851c-1f41841b1662" />

<img width="822" height="402" alt="image" src="https://github.com/user-attachments/assets/a2e6a48f-3060-4dfa-9b75-bd23f72bf4f9" />

<img width="783" height="212" alt="image" src="https://github.com/user-attachments/assets/f0cf7627-9c4d-408e-a2e8-cd9fd0b6919f" />


**Load Balancing Core:** Per-Connection Classifier (PCC) balancing connection states while preserving session affinity

**Bandwidth Allocation Engine:** Centralized Hierarchical Global Queue Trees driven by strict packet-marking Mangle matrices

**Active Queue Management (AQM):** Multi-Instance CAKE leaf queues dynamically drawing from a centralized global parent pool

---

*Third project down. More to come.* 
