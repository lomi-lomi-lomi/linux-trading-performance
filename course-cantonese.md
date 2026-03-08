# Linux 效能調校 - 廣東話版 🇭🇰

兩星期學識 Linux 效能調校，針對超低延遲交易系統。

---

## 📚 Day 1: CPU 架構同 Cache

### 概念 (ELI5)

CPU 入面有好細但超快嘅 memory 叫 cache。諗吓好似廚房咁：

| Cache 層級 | 速度 | 比喻 |
|-----------|------|------|
| L1 Cache | ~1ns | 砧板上嘅材料 (超細，即刻攞到) |
| L2 Cache | ~4ns | 檯面嘅材料 (細，好快) |
| L3 Cache | ~10ns | 附近雪櫃 (大啲，都快) |
| RAM | ~100ns | 行廊盡頭嘅大雪櫃 (超大，慢) |

### 點解重要？

每次 CPU 要嘅 data 唔喺 cache，就要等 **10-100x 更耐** 去 RAM 攞。呢個叫 **cache miss**。

- 1 cache miss = ~100 nanoseconds 浪費
- 1000 cache misses = 0.1 milliseconds 浪費
- 當 target 係 sub-millisecond，呢啲加埋好大件事！

### 一句記住

> "Cache is king — cache miss 貴過 cache hit 100 倍。"

---

## 📚 Day 2: NUMA (Non-Uniform Memory Access)

### 概念

Server 有幾個 CPU socket 嘅話，memory 唔係每個位都一樣快。好似兩個廚師共用兩個雪櫃 — Chef A 攞 Fridge A 好快（就喺隔離），但攞 Fridge B 就要行遠啲。

- 攞自己 socket 嘅 memory = ~100ns
- 攞另一個 socket 嘅 memory = 150-300ns (慢 2-3x!)

### 點解重要？

如果你個 trading thread 行緊 CPU A 但 data 喺 Memory B，每次 memory access 都慢 2-3x。

Coinbase 揀 z1d instances 就係因為佢係 **single socket** — 只有一個 NUMA node，冇「錯 socket」問題！

### 一句記住

> "NUMA = 唔係所有 memory 都一樣快。Keep CPU 同 memory 喺同一個 node。"

---

## 📚 Day 3: Hyperthreading

### 概念

Hyperthreading 令一個實體 CPU core 喺 OS 眼中變成兩個。好似兩個廚師共用一個爐 — 可以輪流用，但係爭緊同一個火位。

兩個「虛擬」CPU 共用同一個 L1/L2 cache、execution units、branch predictor。

### 點解 Trading 要關？

- **Cache pollution** — Thread B 會踢走 Thread A 嘅 hot data
- **Resource contention** — Threads 爭緊 execution units
- **Unpredictable latency** — 有時快有時慢 (最差嗰種)

冇 HT，你個 thread 攞晒 100% L1/L2 cache。有 HT，你要同人分享。

### 一句記住

> "Hyperthreading = 多啲 throughput，但 latency 唔 predictable。Trading 要關。"

---

## 📚 Day 4: CPU Affinity & Pinning

### 概念

CPU pinning 好似餐廳嘅固定座位。冇嘅話，你個 thread 會喺唔同 CPU 之間跳嚟跳去 — 每次跳都冇晒 cached data。

Thread 由 CPU0 跳去 CPU3，CPU3 嘅 cache 係空嘅。要由 RAM 攞 data，差 100 倍！

### 點做？

```bash
# Pin process 去 CPU 2
taskset -c 2 ./myapp

# Check 現有 affinity
taskset -p <pid>
```

### 一句記住

> "Pinning = thread 留喺一個 CPU = cache 永遠暖 = predictable latency"

---

## 📚 Day 5: CPU Isolation (ISOLCPUS)

### 概念

`taskset` 係話俾你個 app 知去邊個 CPU。但 Linux 仲可以放其他嘢落嗰度！

`isolcpus` 係 kernel boot parameter，話俾 Linux 知：「呢啲 CPU 唔好排任何嘢，除非我話要。」

### 點做？

```bash
# Kernel boot param (GRUB)
isolcpus=2,3,4,5

# Check isolated CPUs
cat /sys/devices/system/cpu/isolated
```

### 一句記住

> "isolcpus = CPU 嘅「請勿打擾」牌; taskset = 「去呢間房」"

---

## 📚 Day 6: Real-time Scheduling

### 概念

Linux 有 VIP 通道俾 process 叫「real-time scheduling」。Normal apps 要排隊輪，但 SCHED_FIFO apps 係皇帝 — 佢哋行到自己想停先停。

### 點做？

```bash
# Real-time priority (SCHED_FIFO, priority 99)
chrt -f 99 ./trading_app
```

### 一句記住

> "SCHED_FIFO = 「我行到完先停，冇人打斷到我」"

---

## 📚 Day 7-9: Interrupts 同 Kernel Parameters

### 概念

就算你 isolate 咗 CPU，Linux kernel 仲會「拍膊頭」叫你處理 interrupts 同 housekeeping。

**Holy Trinity:**
- `isolcpus` — 其他 user process 唔好入嚟
- `nohz_full` — 停止 timer ticks
- `rcu_nocbs` — 搬走 memory management callbacks

### 點做？

```bash
# Kernel boot params
isolcpus=2-7 nohz_full=2-7 rcu_nocbs=2-7

# Move all IRQs to housekeeping CPUs
tuna --irqs=* --cpus=0,1 --move
```

### 一句記住

> "Holy Trinity: isolcpus + nohz_full + rcu_nocbs = 真正安靜嘅 CPU"

---

## 📚 Day 10: Memory Optimization

### 概念

三樣嘢要注意：

1. **Huge Pages** — 減少 TLB misses (好)
2. **Transparent Huge Pages (THP)** — Linux 自動幫你整，但會 pause 你個 app (壞！要關)
3. **mlockall()** — 鎖住 memory，唔俾 swap 去 disk

### 點做？

```bash
# 開 1024 個 huge pages (2MB each = 2GB)
echo 1024 > /proc/sys/vm/nr_hugepages

# 關 THP!
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

### 一句記住

> "Huge Pages 好 (手動), Transparent Huge Pages 壞 (自動), mlockall() keep 你喺 RAM"

---

## 📚 Day 11: Storage & NVMe

### 概念

Raft consensus 每個 message 都要寫 disk 先繼續。Disk 慢 = 成個 system 慢。

- Local NVMe = 枱面嘅筆記簿 (即刻攞到)
- EBS = 走廊另一邊嘅 filing cabinet (要行過去)

### 點做？

```bash
# Set I/O scheduler to none (NVMe 自己處理)
echo none > /sys/block/nvme0n1/queue/scheduler
```

### 一句記住

> "NVMe 快，但 fsync() 係你要付嘅稅 — 換取唔會 lose data"

---

## 📚 Day 12-13: Network (ENA, EFA, Kernel Bypass)

### 概念

| 選項 | 速度 | 要改 Code? |
|-----|------|-----------|
| ENA | 25-50µs | 唔使 |
| ENA Express | 10-25µs | 唔使 |
| EFA | 5-10µs | 要 |
| DPDK/Aeron | 1-5µs | 要大改 |

### 一句記住

> "ENA Express 係免費 turbocharger。EFA/DPDK 係 F1 賽車要專業車手。"

---

## 📚 Day 14: 總結

### 一個原則

> **消除 hot path 上嘅所有干擾。**

每一個 optimization 都係同一個目的：令你個 trading thread 唔受打擾。

### 完整 Launch Command

```bash
numactl --cpunodebind=0 --membind=0 \
taskset -c 2,3 chrt -f 99 ./trading_app
```

### Trade-off Triangle

```
     Latency
       /\
      /  \
     /    \
Reliability — Complexity
```

每個 optimization 都係 trade-off。Single AZ = 快但少 redundancy。Kernel bypass = microseconds 但難 debug。知道你 trade 緊咩。

---

## 🎬 Video Reference

**Coinbase: Building an Ultra-Low-Latency Crypto Exchange on AWS**
https://www.youtube.com/watch?v=iB78FrFWrLE
