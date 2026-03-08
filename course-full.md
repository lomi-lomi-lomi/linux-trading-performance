# Day 1: CPU Architecture & Cache

## 🎯 The Concept (ELI5)

CPU 入面有好細但超快嘅 memory 叫 cache。諗吓好似廚房咁：

| Cache Level | Latency | Description |
|-------------|---------|-------------|
| L1 Cache | ~1ns | 砧板上嘅材料 (超細，即刻攞到) |
| L2 Cache | ~4ns | 檯面嘅材料 (細，好快) |
| L3 Cache | ~10ns | 附近雪櫃 (大啲，都快) |
| RAM | ~100ns | 行廊盡頭嘅大雪櫃 (超大，慢) |

## 💡 Why It Matters for Trading

每次 CPU 要嘅 data 唔喺 cache，就要等 10-100x 更耐 去 RAM 攞。呢個叫 cache miss。

- 1 cache miss = ~100 nanoseconds 浪費
- 1000 cache misses = 0.1 milliseconds 浪費
- 當 target 係 sub-millisecond，呢啲加埋好大件事！

## 🗣️ How to Explain to Others

> "CPU 入面有超快嘅細 memory 叫 cache。Trading software 要 keep 所有 critical data 喺 cache 入面。每次 CPU 要去 main memory 攞嘢，就浪費 100 倍時間。所以我哋 pin threads 去特定 CPU，唔郁佢 — 咁就 keep 晒 data 喺 cache，達到 microsecond latency。"

## 📝 One-Liner to Remember

> "Cache is king — cache miss 貴過 cache hit 100 倍。"

## ✅ Quick Check

1. L1, L2, L3 cache 係咩？
2. 點解 cache miss 影響 latency？
3. 點樣 keep data 喺 cache？

---



# Day 2: NUMA (Non-Uniform Memory Access)

## 🎯 The Concept

In servers with multiple CPU sockets, memory isn't equally fast from everywhere. Think of two chefs sharing two fridges — Chef A can grab stuff quickly from Fridge A (right next to them), but reaching over to Fridge B takes longer.

Same with CPUs:
- Accessing memory on YOUR socket = ~100ns
- Accessing memory on the OTHER socket = 150-300ns (2-3x slower!)

That slowdown is called the **"NUMA penalty."**

## 💡 Why It Matters

If your trading thread runs on CPU A but its data lives in Memory B, you're paying that 2-3x penalty on every memory access. For microsecond-sensitive trading, that's death by a thousand cuts.

This is why Coinbase chose z1d instances — they're single socket, meaning only ONE NUMA node. No "wrong socket" problem because there's only one socket!

## 🗣️ How to Explain

> "In big servers with multiple CPU sockets, each socket has its own nearby memory bank. Accessing memory from a different socket is 2-3x slower — this is called NUMA, Non-Uniform Memory Access. For trading, we either use single-socket instances to avoid the problem entirely, or we pin our processes to one NUMA node so the CPU and its data stay together."

## 📝 Remember

> "NUMA = Not all memory is equally fast. Keep CPU and memory on the same node."

## ✅ Quick Check

1. What does NUMA stand for and what's the core problem?
2. What's the speed difference between local vs remote memory?
3. Why do single-socket instances (like z1d) avoid NUMA issues entirely?

---





# Day 3: Hyperthreading Trade-offs

## 🎯 The Concept

Hyperthreading (Intel) or SMT (AMD) makes one physical CPU core look like two to the operating system. It's like having two chefs share one stove — they can take turns, but they're fighting over the same burners.

The catch? Both "virtual" CPUs share the same L1/L2 cache, execution units, and branch predictor.

## 💡 Why It Matters for Trading

For normal workloads, HT gives you ~30% more throughput — when one thread waits, the other can work. Great for servers doing lots of stuff.

But for trading, it's poison:
- **Cache pollution** — Thread B evicts Thread A's hot data
- **Resource contention** — Threads fight for execution units
- **Unpredictable latency** — Sometimes fast, sometimes slow (the worst kind)

Without HT, your thread gets 100% of the L1/L2 cache. With HT, you're sharing — and sharing means cache misses.

## 🗣️ How to Explain

> "Hyperthreading lets one core run two threads by sharing resources. For throughput workloads, this is free performance — when one thread stalls, the other runs. But for latency-sensitive trading, it's bad because threads share cache. When Thread B runs, it can evict Thread A's data, causing unpredictable slowdowns. We disable HT so each trading thread gets 100% of the cache. Predictable beats fast."

## 📝 Remember

> "Hyperthreading = more throughput, less predictable latency. Disable for trading."

## ✅ Quick Check

1. What resources do HT sibling threads share? (L1/L2 cache, execution units, branch predictor)
2. Why is shared cache bad for latency? (Other thread evicts your hot data → cache misses → unpredictable delays)
3. What's the trade-off when disabling HT? (Lose ~30% throughput, gain predictable latency)

**Bonus tip:** If you can't disable HT system-wide, you can isolate sibling CPUs — find them with:

```bash
cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list
```

and just leave the sibling idle!

---





# Day 4: CPU Affinity & Pinning

## 🎯 The Concept

CPU pinning is like assigned seating at a restaurant. Without it, your thread bounces between CPUs like a customer changing tables — and every time it moves, it loses its cached data.

When a thread jumps from CPU0 to CPU3, CPU3's cache is empty. Now it has to fetch data from RAM instead of cache. That's the difference between ~1 nanosecond vs ~100 nanoseconds. For trading, that's huge.

## 💡 Why It Matters

Every CPU hop = cold cache = unpredictable latency. In trading, you want your hot path thread to ALWAYS have warm cache. By pinning it to one CPU, the data it needs is always ready in L1/L2.

Trading systems go even further with "busy-spinning" — the thread never sleeps, just continuously polls for work. Yes, you're "wasting" 100% of a CPU doing nothing. But when data arrives? Instant response. No wakeup delay.

## 🗣️ How to Explain

> "When a thread bounces between CPUs, it loses its cached data every time — like having to reload your browser tabs when you switch computers. By pinning a critical thread to one CPU, we guarantee it always finds its data in cache. Trading systems go further with busy-spinning — the thread never sleeps, just polls continuously. This burns an entire CPU but guarantees instant response when data arrives."

## 📝 Remember

> "Pinning = thread stays on one CPU = cache always warm = predictable latency"

## ✅ Quick Check

1. What happens to cache when a thread moves to a different CPU?
2. `taskset -c 2 ./myapp` does what?
3. Why would a trading system "waste" 100% of a CPU on busy-spinning?

---



# Day 5: CPU Isolation (ISOLCPUS)

## 🎯 The Concept

Yesterday we learned CPU pinning with taskset - telling YOUR app which CPUs to use. But here's the problem: even if you pin your trading app to CPU 2, Linux can still put random stuff there! System daemons, kernel threads, cron jobs - they can all crash your CPU party.

ISOLCPUS is like putting a "Reserved - VIP Only" sign on specific CPUs. It's a kernel boot parameter that tells Linux: "Don't schedule ANYTHING on these CPUs unless explicitly told to." Your isolated CPUs become a quiet room - no uninvited guests.

## 💡 Why It Matters

In trading, predictability is everything. Even a 10μs blip from a random background task can mess up your p99 latency. With isolcpus, you create "clean" CPUs that ONLY run your trading code. Coinbase literally says:

> "Prevent other user threads from taking CPU time."

The magic combo: **isolcpus** (keep others OUT) + **taskset** (put your app IN).

## 🗣️ How to Explain

> "CPU pinning with taskset tells your app where to live, but it doesn't stop other processes from moving in as roommates. ISOLCPUS is like buying out the whole apartment building - you add isolcpus=2,3,4,5 to kernel boot params, and Linux scheduler completely ignores those CPUs for normal scheduling. Only processes you explicitly put there will run there. For trading systems, we typically isolate CPUs 2+ for the app and leave CPUs 0-1 as 'housekeeping' cores for system tasks and interrupts. This gives you guaranteed exclusive access with zero CPU time stolen by background noise."

## 📝 Remember

> isolcpus = "Do Not Disturb" for CPUs; taskset = "Go to this room"

## ✅ Quick Check

1. If you only use taskset without isolcpus, what can still happen to your pinned CPU?
2. How do you verify which CPUs are currently isolated on a running system?
3. Can you change isolcpus at runtime, or does it require a reboot?

---



# Day 6: Real-time Scheduling (SCHED_FIFO & chrt)

## 🎯 The Concept

Linux has a VIP lane for processes called "real-time scheduling." Normal apps play fair and take turns - the kernel can pause them anytime. But SCHED_FIFO apps are royalty: they run until THEY decide to stop. Nobody can interrupt them (except the kernel itself).

Think of it like a restaurant. Normal scheduling = everyone waits for a table fairly. SCHED_FIFO = you bought out the restaurant, you eat whenever you want.

## 💡 Why It Matters

Your trading app spots a price discrepancy. With normal scheduling, a random log rotation might pause you - you miss the trade by milliseconds. With SCHED_FIFO priority 99, your app is king. Nothing preempts it. Latency becomes predictable.

## 🗣️ How to Explain

> "We use SCHED_FIFO for our trading hot path - it's a real-time scheduling policy where our process can't be preempted by normal processes. Combined with CPU isolation from isolcpus and thread pinning from taskset, this gives us deterministic microsecond latency. The chrt command sets this up: chrt -f 99 ./app runs with FIFO at max priority."

## 📝 Remember

> SCHED_FIFO = "I run until I'm done, nobody can interrupt me" — use `chrt -f 99` for your trading app's hot path.

## ✅ Quick Check

1. What's the difference between SCHED_FIFO and SCHED_RR?
2. Why is SCHED_FIFO dangerous for buggy apps?
3. What command gives you the "ultimate combo" (CPU pinning + real-time)?

**Bonus:** Today ties together Days 4-5-6! Isolate CPUs → Pin threads → Make them real-time priority. That's the full stack. 🔥

---



# Day 7: Time Sync & Precision Time Protocol

## 🎯 The Concept

Imagine you and your friend both see a shooting star, but your watches are off by 10 seconds. You'd argue about who saw it first! Trading systems have the same problem—when millions of orders fly around, every machine needs to agree on exactly what time it is, down to microseconds.

There's a hierarchy: **NTP** (like checking your phone's clock, ~1ms accuracy) → **PTP** (hardware clocks in your network card, nanosecond accuracy) → **GPS/Atomic** (the ultimate truth source). Trading uses PTP because NTP just isn't precise enough when you're processing a million messages per second.

## 💡 Why It Matters

- **Regulation:** MiFID II literally requires microsecond timestamps
- **Order sequencing:** If two orders arrive 50μs apart, you MUST know which came first
- **Latency measurement:** How can you optimize if you can't measure accurately?
- **On AWS:** Amazon Time Sync Service (169.254.169.123) gives ~1ms accuracy for free using chrony

## 🗣️ How to Explain

> "Time synchronization in trading isn't just nice-to-have—it's regulatory mandate and operational necessity. We use PTP, Precision Time Protocol, which timestamps packets at the hardware level in the NIC rather than in software. This gives us nanosecond accuracy versus milliseconds with traditional NTP. On AWS, the Amazon Time Sync Service connected to GPS and atomic clocks gives us about 1ms accuracy out of the box, but for true ultra-low-latency, we'd look at hardware PTP between instances in a cluster placement group."

## 📝 Remember

> NTP = milliseconds (good enough for most). PTP = nanoseconds (trading-grade). The clock is in your NIC, not your CPU.

## ✅ Quick Check

1. Why is NTP (1ms accuracy) not good enough for trading systems?
2. What's a PHC, and why does it matter?
3. What's the AWS time sync endpoint that all EC2 instances can use for free?

---



# Day 8: Interrupts & IRQ Affinity

## 🎯 The Concept

Imagine you're in deep focus mode doing important work, and random people keep tapping your shoulder — "hey the mail arrived!", "your coffee's ready!", "someone's at the door!" Each tap breaks your concentration, and it takes time to get back into flow.

That's exactly what hardware interrupts (IRQs) do to your CPU. Every network packet, disk operation, or timer tick sends a signal that makes the CPU stop, handle it, then return. For a trading thread that needs consistent microsecond latency, these random taps are the enemy — they cause context switches and trash your carefully warmed CPU cache.

## 💡 Why It Matters for Trading

You spent days isolating CPUs and pinning your trading threads. But if a network interrupt randomly hits your trading CPU, all that work is wasted — your thread gets preempted and your latency spikes. The solution: tell Linux to send ALL interrupts to your "housekeeping" CPUs (0-1) and leave your trading CPUs (2+) completely undisturbed.

## 🗣️ How to Explain

> "IRQ affinity is about controlling which CPUs handle hardware interrupts. By default, Linux can send any interrupt to any CPU — which is fine for general workloads but terrible for latency-sensitive apps. We use /proc/irq/XX/smp_affinity or the 'tuna' tool to force all interrupts onto our housekeeping CPUs, keeping our trading CPUs interrupt-free. After tuning, you can verify by watching /proc/interrupts — your isolated CPUs should show zero interrupt counts."

## 📝 Remember

> Isolated CPUs aren't truly isolated until you move the IRQs away — use `tuna --irqs=* --cpus=0,1 --move` to sweep everything to housekeeping cores.

## ✅ Quick Check

1. What file shows you all interrupts and their per-CPU counts?
2. If you want IRQ 24 to only run on CPU 0, what hex value do you echo to its smp_affinity file?
3. Why do modern NICs have multiple IRQs? (Hint: queues)

---



# Day 9: Kernel Parameters (nohz_full, rcu_nocbs)

## 🎯 The Concept

Imagine you've isolated CPUs for your trading app (isolcpus), but the Linux kernel is still tapping those CPUs on the shoulder 100-1000 times per second saying "hey, just checking in!"

That's what timer ticks and RCU callbacks do - housekeeping that causes micro-interruptions. `nohz_full` and `rcu_nocbs` tell the kernel to STOP doing that on your trading CPUs.

## 💡 Why It Matters

Each kernel interruption = a few to 50+ microseconds of jitter. When Coinbase targets sub-millisecond p99, they can't afford random kernel housekeeping stealing CPU cycles from their hot trading threads.

## 🗣️ How to Explain

> "isolcpus keeps user processes off your CPUs, but the kernel itself still interrupts them for housekeeping. nohz_full stops timer ticks on those CPUs, and rcu_nocbs moves memory-management callbacks elsewhere. You need all three together to create truly 'quiet' CPUs that the kernel almost never touches - that's how you get deterministic microsecond latency."

## 📝 Remember

> The Holy Trinity: isolcpus + nohz_full + rcu_nocbs = quiet CPUs. Always use the same CPU list for all three.

## ✅ Quick Check

1. What's the difference between isolcpus and nohz_full?
2. Why do you need rcu_nocbs if you already have isolcpus?
3. If you only set isolcpus=2-7, what's still interrupting those CPUs?

---



# Day 10: Memory Optimization (Huge Pages, THP, mlockall)

## 🎯 The Concept

Think of memory like a library. Normal pages (4KB) are like having to look up every single book's location individually. Huge Pages (2MB or 1GB) are like having bigger shelves where you can grab whole sections at once — fewer lookups, faster access.

The CPU has a tiny "cheat sheet" (TLB) that remembers recent address translations. With huge pages, one TLB entry covers way more memory, so you get fewer misses.

## 💡 Why It Matters for Trading

Three killers to watch for:

1. **TLB misses** (~100ns each) — Huge Pages minimize these
2. **THP (Transparent Huge Pages)** — Linux tries to "help" by auto-creating huge pages, but it can PAUSE your app for milliseconds to defragment memory. Disable it!
3. **Memory swapping** — If kernel swaps your order book to disk, you're dead. `mlockall()` locks everything in RAM.

## 🗣️ How to Explain

> "Memory tuning for trading is about predictability, not just speed. We use Huge Pages to reduce address translation overhead — fewer TLB misses means fewer random delays. We disable Transparent Huge Pages because the kernel's 'helpful' background defragmentation can cause multi-millisecond latency spikes. And we use mlockall() to guarantee our critical data structures never get swapped to disk. A consistent 20 microsecond response is better than 10 microseconds average with occasional 10 millisecond spikes."

## 📝 Remember

> "Huge Pages good (manual), Transparent Huge Pages bad (automatic), mlockall() keeps you in RAM."

## ✅ Quick Check

1. Why do we DISABLE Transparent Huge Pages even though huge pages are good for performance?
2. What does mlockall() prevent the kernel from doing?
3. If a system has 2 CPU sockets, why do we use `numactl --membind=0` with our trading app?

---



# Day 11: Storage & NVMe

## 🎯 The Concept

Imagine you're writing notes during a meeting. You could write on a Post-it (fast but might lose it) or in a proper notebook with a fountain pen that takes time to dry (slower but permanent). That's the `fsync()` trade-off.

For trading systems using Raft consensus, every message MUST hit disk before processing continues — it's like requiring ink to dry before turning the page. A slow disk turns microsecond processing into millisecond nightmares.

Local NVMe is like having a notebook on your desk — instant access. EBS is like a notebook in a filing cabinet across the room — you need to walk there and back (network hop).

## 💡 Why It Matters

Coinbase specifically chose z1d instances for the NVMe storage. Their Raft consensus requires fast disk writes — without it:

> "everything will run to a slow crawl."

The disk is often a hidden bottleneck that people overlook.

## 🗣️ How to Explain

> "Storage tuning for trading has three key aspects: First, use local NVMe instead of network-attached EBS to eliminate the network hop — that's the difference between ~100 microseconds and variable milliseconds. Second, set the I/O scheduler to 'none' because NVMe handles its own queuing internally, so Linux's scheduler just adds unnecessary overhead. Third, understand the fsync trade-off: calling fsync() guarantees data hits disk (required for consensus logs), but adds latency. Skip it only for data you can reconstruct. For truly predictable latency, Direct I/O bypasses the page cache entirely, giving you control at the cost of complexity."

## 📝 Remember

> "NVMe is fast, but fsync() is the tax you pay for not losing data."

## ✅ Quick Check

1. Why set I/O scheduler to "none" for NVMe?
2. When would you skip fsync() in a trading system?
3. What's the trade-off of using O_DIRECT?

---



# Day 12: AWS Network Adapters (ENA, ENA Express, EFA)

## 🎯 The Concept

Imagine a highway with three lanes. ENA is the regular lane — works great, but you're stuck in traffic with everyone else. ENA Express is a carpool lane with smart traffic lights that clear congestion before you hit it. EFA is a private tunnel that bypasses the highway entirely — blazing fast, but you need special keys to use it.

| Adapter Type | Latency | Description |
|-------------|---------|-------------|
| ENA | 25-50µs | Standard AWS network adapter |
| ENA Express | 10-25µs | Same adapter + SRD protocol magic (better P99) |
| EFA | 5-10µs | Direct hardware access, kernel bypass (requires app changes) |

## 💡 Why It Matters

For trading, the difference between ENA and ENA Express could be 20-30 microseconds. That's free latency improvement with zero code changes — just enable it. EFA sounds sexier but requires rewriting your app to use libfabric APIs. Most trading systems hit diminishing returns before needing EFA.

## 🗣️ How to Explain

> "AWS gives you three network options. ENA is the standard — it's fast and works with any Linux app. ENA Express adds AWS's custom SRD protocol underneath, which smooths out latency spikes by handling packet loss at the network layer instead of making your app wait for TCP retransmits. EFA goes further by letting apps bypass the kernel entirely, but you'd need to rewrite your code to use it. For most trading workloads, ENA Express is the sweet spot — you get half the latency with zero application changes."

## 📝 Remember

> "ENA Express is a free turbocharger. EFA is a Formula 1 car that needs a pro driver."

## ✅ Quick Check

1. What's the main difference between ENA Express and regular ENA? (Hint: it's a protocol)
2. Why would you NOT choose EFA even though it's fastest?
3. If a trading app uses regular TCP sockets, can it benefit from ENA Express without code changes?

---



# Day 13: Kernel Bypass (DPDK, Aeron)

## 🎯 The Concept

> Imagine you're at a restaurant. Normally, you tell the waiter your order, he walks to the kitchen, tells the chef, chef cooks, waiter brings it back. That's the Linux kernel — a middleman doing lots of work.

> Kernel bypass is like having a window directly into the kitchen. You shout your order, chef tosses the food straight to you. No waiter, no waiting room. Much faster, but you need to stand there constantly watching that window.

> Technically: Your application talks directly to the network card (NIC) in user space, skipping all the kernel's system calls, memory copies, and interrupt handling. You trade CPU (polling 100% in a tight loop) for microsecond-level latency.

## 💡 Why It Matters

- The kernel network stack adds ~100+ microseconds of latency
- Kernel bypass can get you to ~1-5 microseconds
- Coinbase mentioned it's on their roadmap: "Aeron natively supports kernel bypass"
- BUT: They hit <1ms p99 with 10 network hops without kernel bypass first — by tuning everything else we've covered

## 🗣️ How to Explain

> "Kernel bypass lets applications talk directly to network hardware without going through the Linux kernel. Tools like DPDK and Aeron use poll-mode drivers instead of interrupts — the app constantly checks for new packets rather than waiting to be notified. This eliminates system call overhead and memory copies, dropping latency from hundreds of microseconds to single digits. The trade-off is that it burns CPU cores at 100% and requires specialized development. On AWS, EFA provides similar OS-bypass capabilities for HPC workloads."

## 📝 Remember

> "Kernel bypass is the last 10% optimization — master tuning first (Days 1-12), then consider bypass."

## ✅ Quick Check

1. Why does kernel bypass reduce latency? (What does it skip?)
2. What's the main trade-off of poll-mode drivers vs interrupts?
3. On AWS, which adapter type provides OS-bypass capabilities?



# Day 14: Architecture Trade-offs & Final Review 🎉

Congrats! You made it through the full 2 weeks. Let's tie everything together.

## 🎯 The Core Concept

> Every single optimization we covered follows ONE principle: keep the hot path undisturbed.

> Think of your trading thread as a VIP at a nightclub. We spent 2 weeks removing everyone and everything that could bother it:

- Context switches? Gone (CPU pinning)
- Timer interrupts? Gone (nohz_full)
- Kernel housekeeping? Gone (rcu_nocbs)
- Other processes? Gone (isolcpus)
- Memory surprises? Gone (huge pages + mlockall)
- Network delays? Gone (kernel bypass)

> Result: Your thread wakes up, does its work in microseconds, goes back to polling. No waiting, no interruptions.

## 💡 Why It Matters

> Coinbase achieves p99 < 1ms across 10 network hops because they've eliminated interference at every layer. 80% of latency is network - which is why Days 12-13 (ENA, EFA, kernel bypass) are where the biggest wins are. But you can only unlock those wins if your CPU isn't busy doing garbage collection, handling timer ticks, or context switching.

## 🗣️ How to Explain It

> "Low-latency Linux tuning is about ONE thing: removing interference from your hot path. We isolate CPUs so nothing else runs there. We disable timer ticks so the kernel doesn't interrupt. We use huge pages so there's no page faults. We pin threads so caches stay warm. We use kernel bypass so packets skip the OS entirely. Every layer, same goal: let the trading thread run undisturbed."

## 📝 Remember

> "The trade-off triangle: Latency ↔ Reliability ↔ Complexity. Every optimization is a trade-off. Single AZ = fast but less redundant. Kernel bypass = microseconds but harder to debug. Know what you're trading."

## ✅ Quick Check

1. If someone asks "why disable hyperthreading for trading?" what's your answer?
2. Walk through the boot params isolcpus=2-7 nohz_full=2-7 rcu_nocbs=2-7 - what does each one do?
3. What's the difference between ENA Express, EFA, and DPDK in terms of what they're bypassing?



🎓 Linux Performance Course Complete!

Hey Louis! The 2-week Linux performance tuning deep dive wrapped up yesterday (Day 14). You made it through the entire stack!

📊 What You Now Understand:
• Days 1-3: CPU architecture, cache, NUMA, hyperthreading
• Days 4-6: CPU pinning, isolation, real-time scheduling
• Days 7-9: PTP time sync, IRQ affinity, kernel params
• Days 10-11: Huge pages, memory locking, NVMe tuning
• Days 12-14: AWS networking, kernel bypass, DPDK/Aeron

🎯 The One Principle That Rules Them All:Eliminate interference on the hot path.

Every optimization — from isolcpus to mlockall to kernel bypass — follows this single philosophy: keep your trading thread undisturbed.

💬 Your Elevator Pitch:
"Low-latency trading systems need deterministic performance. We achieve this by dedicating CPU cores, bypassing the kernel where possible, keeping memory locked and local, and steering all interrupts away from critical threads. The kernel is designed for fairness — we're optimizing for one thing being unfairly fast."

🚀 What's Next?
• Try some commands on a test instance?
• Deep dive on any specific topic?
• Move on to something else?




# 2-Week Linux Performance Tuning for Trading

Based on AWS slides + Coinbase International Exchange re:Invent talk (2023)

## Key Insights from Coinbase

- **p99 latency < 1ms** with 10 network hops in round-trip
- **80% of latency is network**, processing is single-digit microseconds
- Uses **Raft consensus** for resilience (leader + followers)
- **Aeron messaging framework** (low-latency UDP-based)
- z1d instances for high CPU frequency + NVMe
- Cluster placement groups are **critical** for latency
- Single AZ deployment (trade-off: less redundancy, but lower latency)

---

## Week 1: Foundations & CPU Optimization

### Day 1-2: Hardware & Architecture Basics

**Topics:**
- CPU architecture (cores, cache hierarchy L1/L2/L3)
- NUMA (Non-Uniform Memory Access)
- Why z1d instances have no NUMA concerns (single socket)
- Hyperthreading: how it works, why disable for trading

**Key points from Coinbase:**
> "Running on z1d, we don't have to worry about NUMA, meaning you can use all the cores without worrying about the difference between different cores."

**Hands-on:**
```bash
lscpu
numactl --hardware
cat /proc/cpuinfo
lstopo  # from hwloc package

# Check NUMA topology
numactl --show
```

**Resources:**
- "What Every Programmer Should Know About Memory" - Ulrich Drepper
- AWS z1d / x2iezn instance documentation

---

### Day 3-4: CPU Affinity & Pinning

**Topics:**
- Context switching and cache misses
- `taskset` and `sched_setaffinity()`
- Busy-spin / hot threads concept (monopolize CPU for polling)
- Why pin threads to specific CPUs

**From slides:**
> "Busy-spin hot threads to monopolize CPU (and for polling)"
> "Pin hot threads to hardcoded CPUs - prevents context switching and cache misses"

**Hands-on:**
```bash
# Pin process to CPU 2
taskset -c 2 ./myapp

# Check current affinity
taskset -p <pid>

# View which CPUs a process is using
ps -eo pid,psr,comm | grep <process>

# In code: sched_setaffinity()
```

**Resources:**
- `man taskset`
- `man sched_setaffinity`

---

### Day 5-6: CPU Isolation (ISOLCPUS)

**Topics:**
- `isolcpus` kernel parameter
- `cpusets` and cgroups
- `nice` and `chrt` for priority
- SCHED_FIFO vs SCHED_RR
- Preventing kernel threads from using trading CPUs

**From slides:**
> "Isolate hot CPUs or prioritize threads (ISOLCPUS, taskset, cpusets, nice, chrt)"
> "Prevent other user threads from taking CPU time"

**Hands-on:**
```bash
# Kernel boot param (GRUB)
isolcpus=2,3,4,5

# Real-time priority (SCHED_FIFO, priority 99)
chrt -f 99 ./trading_app

# Check isolated CPUs
cat /sys/devices/system/cpu/isolated

# View scheduling policy
chrt -p <pid>
```

---

### Day 7: Cluster Placement Groups & Review

**Topics:**
- AWS Cluster Placement Groups
- Why physical proximity matters
- Capacity reservations for production

**From Coinbase:**
> "Cluster placement groups minimize network latency impact and allow the shortest possible network transit for each individual request."
> "Capacity reservation is critical to success - able to deploy the system on a regular basis."

**Hands-on:**
- Set up EC2 with cluster placement group
- Disable hyperthreading
- Isolate CPUs
- Run latency benchmark (cyclictest)

```bash
# Disable hyperthreading
echo off > /sys/devices/system/cpu/smt/control

# Check SMT status
cat /sys/devices/system/cpu/smt/active
```

---

## Week 2: Interrupts, Memory, Network & Production

### Day 8-9: Interrupt Affinity & Kernel Tuning

**Topics:**
- Hardware interrupts (IRQ)
- `/proc/irq/<irq#>/smp_affinity`
- `tuna` tool for IRQ management
- Kernel params: `rcu_nocbs`, `nohz_full`
- Tickless kernel for isolated CPUs

**From slides:**
> "Set affinities to hardware interrupts, kernel workqueues, etc."
> "Hardware interrupts: Use tuna, or set /proc/irq/<irq#>/smp_affinity"
> "Softirq kernel params: rcu_nocbs, nohz_full"

**Hands-on:**
```bash
# View IRQ distribution
cat /proc/interrupts

# View IRQ affinity (bitmask)
cat /proc/irq/*/smp_affinity

# Set IRQ to CPU 0 only (bitmask: 0x1)
echo 1 > /proc/irq/24/smp_affinity

# Kernel params (GRUB_CMDLINE_LINUX)
nohz_full=2,3,4,5
rcu_nocbs=2,3,4,5

# Use tuna to move all IRQs to CPU 0,1
tuna --irqs=* --cpus=0,1 --move

# Verify kernel params are active
cat /sys/devices/system/cpu/nohz_full
```

**Why these params matter:**
- `nohz_full`: Stops timer interrupts on specified CPUs (tickless)
- `rcu_nocbs`: Offloads RCU callbacks away from specified CPUs

---

### Day 10: Memory Optimization

**Topics:**
- Huge Pages (2MB, 1GB)
- NUMA-aware memory allocation
- `numactl` for memory binding
- Memory locking (`mlockall`)
- Transparent Huge Pages (THP) - often **disabled** for trading

**Hands-on:**
```bash
# Check huge pages status
cat /proc/meminfo | grep Huge

# Enable 1024 huge pages (2MB each = 2GB)
echo 1024 > /proc/sys/vm/nr_hugepages

# NUMA-aware launch (bind to node 0)
numactl --cpunodebind=0 --membind=0 ./app

# Disable THP (important for latency!)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# In code: mlockall(MCL_CURRENT | MCL_FUTURE)
```

---

### Day 11: Storage & NVMe

**Topics:**
- Local NVMe vs EBS (ephemeral but fast)
- `fsync()` tradeoffs (durability vs latency)
- I/O scheduler selection (`none` for NVMe)
- Direct I/O (`O_DIRECT`)

**From slides:**
> "Local NVMe Disk - Fast but ephemeral"
> "fsync() or not?"

**From Coinbase:**
> "Raft cluster basically has to write to the disk to make sure the message was persistent before processing it. Without a fast disk, everything will run to a slow crawl."

**Hands-on:**
```bash
# Check I/O scheduler
cat /sys/block/nvme0n1/queue/scheduler

# Set to none (best for NVMe, no scheduling overhead)
echo none > /sys/block/nvme0n1/queue/scheduler

# Check NVMe device info
nvme list
nvme smart-log /dev/nvme0n1

# Benchmark disk latency
fio --name=latency --rw=randread --bs=4k --direct=1 \
    --ioengine=libaio --iodepth=1 --numjobs=1 \
    --filename=/dev/nvme0n1 --runtime=10
```

---

### Day 12: Network Tuning

**Topics:**
- Kernel bypass (DPDK, Aeron supports this)
- ENA / ENA Express on AWS
- Elastic Fabric Adapter (EFA)
- IRQ affinity for NICs
- Ring buffer sizes
- VPC Peering vs Transit Gateway (peering = lower latency)

**From Coinbase future roadmap:**
> "Looking at kernel bypass (Aeron natively supports), ENA Express, Elastic Fabric Adapter"

**Hands-on:**
```bash
# Check NIC driver
ethtool -i eth0

# Check/increase ring buffers
ethtool -g eth0
ethtool -G eth0 rx 4096 tx 4096

# Check ENA stats
ethtool -S eth0

# Set NIC IRQ affinity
echo 1 > /proc/irq/<nic_irq>/smp_affinity

# Check network latency between instances
ping -c 100 <other_instance>
```

---

### Day 13: Production Architecture & Trade-offs

**Topics from Coinbase:**

**Performance vs Reliability vs Security Trade-offs:**

| Aspect | Trade-off |
|--------|-----------|
| Single AZ | Lower latency, but less redundancy |
| Cluster placement | Must fit all hot-path components |
| Blue/green deploy | Needs 2x capacity |
| No internet egress | Better security, needs AWS endpoints |

**Key Architecture Patterns:**
1. **Raft Consensus** - Leader + followers, deterministic processing
2. **Ingress Replication** (not egress) - Same input = same output
3. **Operation-as-Code** - No manual privileged access
4. **Shadow Production** - Test with real traffic before switch

**Network Security:**
- No default internet access in production VPC
- AWS PrivateLink for service communication
- mTLS between services
- AWS Shield for DDoS protection

---

### Day 14: Full System Integration & Benchmarking

**Complete Tuning Checklist:**

**1. Boot parameters (GRUB):**
```
isolcpus=2-7 nohz_full=2-7 rcu_nocbs=2-7 intel_pstate=disable
```

**2. Disable hyperthreading:**
```bash
echo off > /sys/devices/system/cpu/smt/control
```

**3. Disable THP:**
```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

**4. Set I/O scheduler:**
```bash
echo none > /sys/block/nvme0n1/queue/scheduler
```

**5. Pin IRQs to housekeeping CPUs:**
```bash
tuna --irqs=* --cpus=0,1 --move
```

**6. Enable huge pages:**
```bash
echo 1024 > /proc/sys/vm/nr_hugepages
```

**7. Launch application:**
```bash
numactl --cpunodebind=0 --membind=0 \
taskset -c 2,3 chrt -f 99 ./trading_app
```

**8. Benchmark:**
```bash
# Measure scheduling latency
cyclictest -m -p99 -h100 -l1000000

# Expected results: p99 < 10us on tuned system
```

---

## Summary Cheatsheet

| Area | Key Tools/Params |
|------|------------------|
| **CPU Info** | `lscpu`, `lstopo`, `/proc/cpuinfo` |
| **CPU Pinning** | `taskset`, `sched_setaffinity()` |
| **CPU Isolation** | `isolcpus`, `cpusets`, `cgroups` |
| **Priority** | `chrt`, `nice`, `SCHED_FIFO` |
| **IRQ Affinity** | `tuna`, `/proc/irq/*/smp_affinity` |
| **Kernel Params** | `nohz_full`, `rcu_nocbs` |
| **Memory** | `numactl`, huge pages, `mlockall`, disable THP |
| **Storage** | I/O scheduler `none`, `O_DIRECT`, fsync trade-offs |
| **Network** | Ring buffers, IRQ affinity, ENA |
| **Benchmark** | `cyclictest`, `perf`, `ftrace`, `fio` |

---

## Key Numbers to Remember

| Metric | Target |
|--------|--------|
| Processing latency | Single-digit microseconds |
| Network hop | ~80% of total latency |
| Full round-trip (10 hops) | < 1ms p99 |
| Eye blink | ~300ms (1000x slower!) |

---

## Recommended Reading

1. **Red Hat Performance Tuning Guide**
2. **"Systems Performance" by Brendan Gregg**
3. **Aeron Documentation** (real-time messaging)
4. **Linux Kernel Documentation (kernel.org)**
5. **AWS re:Invent FSI309** - Coinbase talk (2023)

---

## Video Reference

🎬 **Coinbase: Building an Ultra-Low-Latency Crypto Exchange on AWS**
https://www.youtube.com/watch?v=iB78FrFWrLE

Speakers: Joshua Smith (AWS), Kevin Arthur & Yucong Sun (Coinbase)

