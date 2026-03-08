# Linux Performance Tuning for Trading Systems

A 2-week learning guide to Linux performance optimization for ultra-low-latency trading systems, based on AWS best practices and Coinbase International Exchange's re:Invent 2023 talk.

## 🎯 What You'll Learn

This guide covers the complete stack for achieving **sub-millisecond latency** on Linux:

| Week | Topics |
|------|--------|
| **Week 1** | CPU architecture, cache, NUMA, hyperthreading, CPU pinning, isolation, real-time scheduling |
| **Week 2** | Time sync (PTP), IRQ affinity, kernel params, huge pages, NVMe tuning, AWS networking, kernel bypass |

## 🚀 Key Insights from Coinbase

- **p99 latency < 1ms** with 10 network hops in round-trip
- **80% of latency is network**, processing is single-digit microseconds
- Uses **Raft consensus** for resilience
- **Aeron messaging framework** (low-latency UDP-based)
- z1d instances for high CPU frequency + NVMe
- Cluster placement groups are **critical** for latency

## 📚 Course Structure

### Week 1: CPU & Foundations
- **Day 1**: CPU Architecture & Cache (L1/L2/L3)
- **Day 2**: NUMA (Non-Uniform Memory Access)
- **Day 3**: Hyperthreading Trade-offs
- **Day 4**: CPU Affinity & Pinning (`taskset`)
- **Day 5**: CPU Isolation (`isolcpus`)
- **Day 6**: Real-time Scheduling (`SCHED_FIFO`, `chrt`)
- **Day 7**: Time Sync & PTP

### Week 2: System & Network
- **Day 8**: Interrupts & IRQ Affinity
- **Day 9**: Kernel Parameters (`nohz_full`, `rcu_nocbs`)
- **Day 10**: Memory Optimization (Huge Pages, THP, `mlockall`)
- **Day 11**: Storage & NVMe
- **Day 12**: AWS Network Adapters (ENA, ENA Express, EFA)
- **Day 13**: Kernel Bypass (DPDK, Aeron)
- **Day 14**: Architecture Trade-offs & Review

## 💡 The One Principle

> **Eliminate interference on the hot path.**

Every optimization follows this single philosophy: keep your trading thread undisturbed.

## 🔧 Quick Reference

```bash
# The Holy Trinity of CPU isolation
isolcpus=2-7 nohz_full=2-7 rcu_nocbs=2-7

# Launch trading app with full optimization
numactl --cpunodebind=0 --membind=0 \
taskset -c 2,3 chrt -f 99 ./trading_app
```

## 📖 Files

- [`course-full.md`](course-full.md) - Complete course with all 14 days (English + Cantonese explanations)
- [`course-cantonese.md`](course-cantonese.md) - 廣東話版本 (Cantonese version for easier reading)

## 🎬 Video Reference

**Coinbase: Building an Ultra-Low-Latency Crypto Exchange on AWS**  
https://www.youtube.com/watch?v=iB78FrFWrLE

## 📚 Recommended Reading

1. Red Hat Performance Tuning Guide
2. "Systems Performance" by Brendan Gregg
3. Aeron Documentation
4. Linux Kernel Documentation (kernel.org)
5. AWS re:Invent FSI309 - Coinbase talk (2023)

## License

MIT
