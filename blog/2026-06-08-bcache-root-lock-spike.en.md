# A Deep Dive into a 1-Second Linux bcache Write Stall — From User-Visible Symptom to root btree Lock Contention

## TL;DR

A production Ceph cluster running on bcache + HDD was hitting reproducible **1-3 second write latency spikes**, but only after a period of idleness followed by a burst. After 7 layers of evidence (including **4 falsified hypotheses**), the root cause was localized to **`rw_semaphore` contention on the `cache_set->root` btree node** inside the bcache kernel module, specifically at `__bch_btree_map_nodes+0x10c`. The `rwsem` FIFO fairness rule **amplifies** an internal 100-300 ms lock wait into a second-scale user-visible spike. This post documents the full path, the 4 wrong guesses, and the final BPF + disassembly double-proof.

**Keywords**: Linux kernel, bcache, Ceph, rwsem, bpftrace/BCC, root btree contention

---

## 1. The scene: 1-second stall, but only under specific conditions

### 1.1 Architecture

- 3-node Ceph cluster, 14 OSDs per node
- Each OSD backend: `bluestore` over `bcache` (writeback mode)
- bcache cache: NVMe (~1 TB) × 4 per node, aggregated into 3 cache_sets
- bcache backing: HDD (~12 TB) × 14 per node
- Clients use cephfs (via a juicefs gateway)

### 1.2 Symptom

```
juicefs stats:
  commit_lat normally 1-10 ms
  → occasional jumps to 50-1000 ms (spike)
  → sustained for 1-3 seconds
  → self-heals, returns to 1-10 ms
```

**Critical trigger**: After the system has been idle for a while, the client issues a burst `dd` of a large file. The first 2 seconds **always** trigger the spike. **Continuous IO, by contrast, does not stall**.

This "idle-then-burst-only" pattern turned out to be the key clue.

---

## 2. Round 1: rule out the usual Ceph latency suspects

For any Ceph latency problem, start by ruling out the standard 5:

| # | Cause | How to rule out |
|---|---|---|
| 1 | `scrub` / `deep-scrub` | `ceph -s` shows no active scrub PGs during spike |
| 2 | `recovery` / `backfill` | `ceph -s` shows healthy cluster during spike |
| 3 | HDD slowness | `iostat -xm 1`: HDD `%util=0` during spike, fully idle |
| 4 | rocksdb compact | OSD `perf dump`: `kv_sync_lat / commit_lat < 5%` |
| 5 | **bcache design** | OSD `perf dump`: **`state_aio_wait_lat / commit_lat = 96%+`** ← hit |

Cause #5 hits, narrowing the focus to the BLOCK layer (NVMe→bcache→HDD).

### 2.1 First weird observation

`iostat` during the spike shows:
- `bcache0` etc.: `%util=100%`, but `aqu-sz=0`, `wMB/s=0`, `w_await=0`
- NVMe device: `%util=0`
- HDD device: `%util=0`

**bcache is "busy" but no IO is being submitted.** Classic signature of "bio entered bcache but got stuck on some software-layer wait inside it".

---

## 3. Round 2: capture the kernel stacks of D-state tasks

I wrote a simple high-frequency `/proc/<tid>/stack` sampler that fires during spikes:

```
==== dstate_140000_w57_n81.log (81 stacks captured) ====
57  comm=tp_osd_tp       top=cached_dev_make_request   ← Ceph OSD writer
3   comm=bcache_writebac top=bch_writeback_thread       ← bcache writeback kthread
1   comm=kworker         top=__bch_btree_map_nodes      ← btree op
others: node_exporter etc. (sysfs readers — noise)
```

**70 % of D-state tasks are `tp_osd_tp` with stack top at `cached_dev_make_request+0x3e0`**.

### 3.1 Disassembly pins down +0x3e0

```bash
KO=/lib/modules/$(uname -r)/kernel/drivers/md/bcache/bcache.ko
objdump -d $KO | grep -A 50 '<cached_dev_make_request>'
```

The line at offset `+0x3e0`:

```asm
14bb4: bl  <down_read>              ; call down_read
14bb8: ldr ...                       ; +0x3e0 (down_read return point)
```

→ **`+0x3e0` is the return point of the `down_read(&dc->writeback_lock)` call**.

Writers are stuck in bcache's `writeback_lock` (per-cached_dev rw_semaphore).

### 3.2 Hypothesis 1: direct contention on writeback_lock

The most obvious guess: writers queue on `writeback_lock`. But who's on the writer side?

In bcache, only `bch_writeback_thread` (the background writeback kthread) calls `down_write(&dc->writeback_lock)`, doing `refill_dirty`.

**Hypothesized mechanism**: the rwsem FIFO fairness rule. Once a writer is queued, subsequent readers must also queue (to prevent writer starvation). This amplifies the writer's tens-of-millisecond hold window into a user-visible 1-second-plus spike.

This hypothesis is **partially correct** (later confirmed as an *amplifier*), but it's not the root cause, because: **what is the original source of long lock hold time on the writer side?**

---

## 4. Round 3: measure lock latency precisely with BPF

`/proc/stack` sampling has limits:
- 100 ms sampling interval misses short contentions
- Only sees "a snapshot while stalled", can't quantify "how long was it stalled"

Upgrade to BPF. The environment is OpenEuler 20.03 LTS, kernel 4.19.90 — `bpftrace` isn't packaged (would need to build from source), but `bcc-tools` is. Note: in OpenEuler the Python module is named `bpfcc` (not `bcc`), so any BCC Python script needs an import compatibility layer.

### 4.1 funclatency: histogram of rwsem slow-path duration

```bash
sudo /usr/share/bcc/tools/funclatency -m 'rwsem_down_read_failed'
```

Result over 60 seconds:

```
msecs               : count    distribution
   0 -> 1           : 1410    |****************************************|
   2 -> 3           : 221     |******                                  |
   4 -> 7           : 433     |************                            |
   8 -> 15          : 545     |***************                         |
  16 -> 31          : 88      |**                                      |
  32 -> 63          : 5       |                                        |
  64 -> 127         : 260     |*******                                 |
 128 -> 255         : 109     |***                                     |
 256 -> 511         : 0       |                                        |
 512 -> 1023        : 3       |                                        |
1024 -> 2047        : 42      |*                                       |  ← 1-2 s
2048 -> 4095        : 11      |                                        |  ← 2-4 s
```

**Bimodal distribution**:
- 0-16 ms: normal or minor contention (~2,600 events)
- **1-4 s**: 53 extreme spike events

53 lock waits over 1 second, perfectly matching the user-reported "1-2 second stalls".

### 4.2 funcslower: capture full stacks

```bash
sudo /usr/share/bcc/tools/funcslower -m 50 -K 'rwsem_down_read_failed'
```

Captured 194 spike events in 60 seconds (clustered in a 3-second burst):

```
==== [11:41:08] event ====
  comm     : tp_osd_tp
  pid/tid  : 980986/981104
  latency  : 1482 ms
  lock     : 0xffff80464a160d18
  kstack   :
    rwsem_down_read_failed
    cached_dev_make_request+0x3e0     ← matches /proc/stack
    generic_make_request+0x17c
    submit_bio+0x5c
    blkdev_direct_IO+0x3b0
    aio_write+0xec
    __arm64_sys_io_submit+0xb4
    el0_svc+0x8

==== [11:41:08] event ====
  comm     : kworker/9:65
  pid/tid  : 256329/256329
  latency  : 307 ms
  lock     : 0xffff8052b466a860   ← ★ different lock address!
  kstack   :
    rwsem_down_read_failed
    __bch_btree_map_nodes+0x10c   ← ★ different function
    bch_btree_insert+0xe8
    bch_data_insert_keys+0xd0
    process_one_work+0x1b4
    worker_thread+0x54
```

**Key finding**: lock contention shows up in **two distinct locations**:
1. `tp_osd_tp` at `cached_dev_make_request+0x3e0` (writeback_lock)
2. `kworker` at `__bch_btree_map_nodes+0x10c` (a different rwsem)

The lock addresses fall into two groups:
- 14 addresses ending in `d18` — looks per-device, likely the 14 cached_dev writeback_locks
- **2 addresses ending in `860`** — only 2 distinct addresses but they account for 67 % of all events

### 4.3 Hypothesis 2: btree cache miss

Why does the lock wait so long? First intuitive guess: **btree node cache miss**. bcache holds an LRU cache of btree nodes (each 256 KB) in memory (`btree_cache_size`).

```
Total btree nodes:           ~34,000 per cache_set × 3 cache_sets = ~100k nodes
Memory to hold all in cache: 34,000 × 256 KB = 8.6 GB per cache_set
Actual btree_cache_size:     8.0 GB per cache_set
                             ↑
                       5 % short, so LRU eviction is unavoidable
```

When the system is idle, btree nodes get evicted. On the next burst, the first wave of accesses hits the missing nodes → synchronous 256 KB reads from NVMe → lock held for hundreds of milliseconds → spike.

**A clean theory**. Time to prove it.

I wrote a BPF tracer that hooks `bch_btree_node_read` (only called on cache miss):

```python
# btree_miss_tracker.py — track key bcache functions
# - bch_btree_node_get   every access (hit + miss)
# - bch_btree_node_read  only on cache miss
```

Ran for 60 seconds under heavy load:

```
ts          get/s    miss/s   alloc/s   hit_rate
15:09:12   13396.3      0.0       0.4    100.00%
15:09:17   13223.2      0.0       6.4    100.00%
15:09:22   13902.5      0.0       6.4    100.00%

Total btree accesses (get):    282,904
Total cache misses (read):           0
Hit rate:                       100.00%
```

**282,904 btree accesses, zero cache misses.**

**Hypothesis 2 falsified** ❌. Every btree node was in memory the whole time, no NVMe reads needed.

An important scientific moment: a measured falsification of one's own theory. Disappointing but it brings the truth closer.

---

## 5. Round 4: the root lock contention hypothesis

If btree is fully cached, what is the lock waiting on? Look at the data again:

- 67 % of lock events in the 60-second window cluster on **2 lock addresses**
- The btree has 100k+ nodes total, one rwsem per node
- Only 2 hot ones = **these 2 are globally shared**

In bcache, the only "globally shared btree node locks" can be **the root btree node lock of each cache_set**. With 3 cache_sets, there should be 3 root locks, but only 2 hot — possibly the third wasn't busy during this particular spike.

### 5.1 Read the bcache source

```c
// btree.h — the bcache_btree_root macro (initially overlooked)
#define bcache_btree_root(fn, c, op, ...) ({                          \
    int _r = -EINTR;                                                  \
    do {                                                              \
        struct btree *_b = (c)->root;        /* ← take c->root */    \
        bool _w = insert_lock(op, _b);                                \
        rw_lock(_w, _b, _b->level);          /* ← ★ lock root */     \
        if (_b == (c)->root && _w == insert_lock(op, _b)) {           \
            _r = bch_btree_ ## fn(_b, op, ##__VA_ARGS__);             \
        }                                                              \
        rw_unlock(_w, _b);                                            \
        ...                                                            \
    } while (_r == -EINTR);                                           \
    _r;                                                                \
})
```

`__bch_btree_map_nodes` (the function we saw in spike stacks) calls `bcache_btree_root`, whose **first step is to take the lock on root**. Every btree write (including `bch_btree_insert`) starts here.

70 concurrent writers → all must pass through the root lock → queue depth spikes → spike.

### 5.2 Disassemble `__bch_btree_map_nodes+0x10c`

```bash
objdump -d $KO | grep -A 200 '<__bch_btree_map_nodes>:'
```

Localize `+0x10c`:

```asm
b5b0: <__bch_btree_map_nodes>:                  ; function start
b5e8: add  x22, x1, #0x8, lsl #12               ; x22 = c + 0x8000
b600: ldr  x19, [x22, #20072]                    ; x19 = *(c + 0xCE68) = c->root
b608: add  x25, x19, #0x60                       ; x25 = &root->lock
b614: mov  x0, x25                                ; x0 = &root->lock
b618: b.gt b6b8                                   ; branch on read vs write decision
b61c: bl   <down_write>                          ; +0x6c: down_write
...
b6b8: bl   <down_read>                           ; +0x108: down_read
b6bc: ldr  x0, [x22, #20072]                     ; +0x10c (down_read return point)
```

**`+0x10c` is the return point of `down_read(&root->lock)`**. Seeing this offset in the stack means: the thread is currently inside `down_read`, suspended via `schedule()`.

`x19` is `c->root` (the root btree node of the cache_set), `x25` is `&root->lock`.

### 5.3 Double proof via BPF

Two independent BPF verification scripts:

**Method 1: Proof by exclusion**

Trace every return value of `bch_btree_node_get` (used to access **child nodes**), and compute `&b->lock` for each. If the hot address `0xffff8052b466a860` is *not* in this set, then it's not a child lock → it must be root.

**Method 2: Read c->root directly**

Hook `__bch_btree_map_nodes` on entry. The 2nd argument is `cache_set *c`. Use `bpf_probe_read_kernel(&root, c + 0xCE68)` to follow `c->root` directly, then compute `&root->lock = root + 0x60`, and see whether it matches the hot addresses.

```
Method 1: proof by exclusion
  10,240 child node lock addresses observed via bch_btree_node_get

  Comparison:
    0xffff8052b466a860  ← ✓ not in child set → must be root
    0xffffa057687eb860  ← ✓ not in child set → must be root

Method 2: read c->root directly
  3 root lock addresses observed:
    0xffff804880f93c60  accessed 34,289 times  (cache_set A, no spike)
    0xffffa057687eb860  accessed 26,148 times  (cache_set B) ★ MATCH
    0xffff8052b466a860  accessed 23,959 times  (cache_set C) ★ MATCH

✓ Double-proven: both hot addresses are ROOT btree node locks.
```

**Two independent methods, different paths, same conclusion.** Textbook kernel-debug proof.

### 5.4 An interesting observation: the busiest cache_set wasn't the one stalling

`cache_set A`'s root was accessed 34,289 times — more than B or C — **yet it never appeared in spike contention**. That tells us:

> A spike isn't driven by "absolute access volume". It's driven by "instantaneous arrival rate plus a confluence of triggers."

cache_set A's accesses were likely spread out over time; B and C absorbed the burst.

---

## 6. But wait — why is it a bursty spike, not sustained slowness?

If it were purely root lock contention, **continuous load should also be slow**. But the observation is:

```
continuous load    → no stall
idle then burst    → 1-3 s spike
```

Root lock contention alone doesn't explain that **time-pattern**. There must be additional factors.

### 6.1 The real multi-factor model

```
"bursty spike" = 4 latent factors ignited simultaneously

  Factor A (intrinsic):
    bcache's centralized btree design → one root per cache_set → high-concurrency
    convergence point

  Factor B (idle-triggered):
    After idleness, writeback_thread enters sleep (dirty=0, nothing to do)
    → On burst, it wakes up and immediately wants down_write(writeback_lock)
    → ★ triggers rwsem FIFO fairness
    → subsequent readers (tp_osd_tp) all queue behind it

  Factor C (bursty arrival):
    dd submits 8 GB in one shot
    → ceph triplicates → N nodes × M threads = ~70 concurrent writers
    → they all hit root lock at the same instant
    → ★ queue depth jumps from 0 to 70

  Factor D (amplifier):
    The FIFO fairness rule on cached_dev->writeback_lock
    Amplifies the root lock's internal 100-300 ms wait
    into a 1-3 s wait visible on cached_dev_make_request
    → user-visible

  → all 4 simultaneously → spike

Under continuous load:
  B doesn't hold (writeback_thread is continuously active, never sleeps deeply)
  C doesn't hold (arrivals are time-distributed, steady-state queue is short)
  → only A + weak D remain, no user-visible stall
```

### 6.2 This explains every counterintuitive observation

| Counterintuitive | Explanation |
|---|---|
| Continuous IO doesn't stall; idle-then-burst does | B+C only hold during the idle→burst transition |
| Busiest cache_set didn't stall | Its accesses are spread out; factor C doesn't apply |
| btree cache miss = 0 | Nodes are in memory; the lock wait isn't on IO |
| Child node locks are always fast | No one converges on a child node |
| Lock waits 1-3 s but each op is microseconds | rwsem FIFO + writeback_lock 2-stage amplification |

---

## 7. The full causal chain

```
[T = -∞ ~ 0]  System is idle, all 4 latent factors in position:
  · writeback_thread sleeping
  · root lock idle (no writer queued)
  · closure queue empty
  · cached_dev->writeback_lock fully idle

[T = 0]       Client dd burst of 8 GB

[T = 0+1ms]   Ceph triplicates, 70 tp_osd_tp call io_submit concurrently
  → 70 cached_dev_make_request almost simultaneously
  → 70 down_read(writeback_lock) all succeed (rwsem allows concurrent readers)
  → 70 closures enqueued onto bcache_writeback_wq

[T = 0+5ms]   70 kworkers concurrently run bch_data_insert chain
  → each invokes __bch_btree_map_nodes
  → first thing: down_read(root->lock)
  → ★ 70 readers contend for the same root lock
  → one wins and starts the btree insert; the other 69 queue

[T = 0+50ms]  Some closure triggers a btree node split, needs down_write(root)
  → root lock transitions to write mode
  → rwsem FIFO triggers: new readers must queue too

[T = 0+100ms] Concurrently, dirty rises sharply, writeback_thread wakes up
  → it calls down_write(writeback_lock)
  → finds too many holders, queues
  → ★ writeback_lock FIFO triggers
  → incoming tp_osd_tp down_read(writeback_lock) also queue

[T = 0+100ms ~ 0+2s]  Double-FIFO blockage
  → kworker stuck on root->lock
  → tp_osd_tp stuck on writeback_lock
  → cached_dev_make_request returns 1-2 s late, visible to user

[T = 0+1.5s ~ 0+3s]   Locks released layer by layer
  → btree insert finishes, root lock released
  → kworker completes closure, releases writeback_lock read
  → writeback_thread acquires write lock, refills, releases
  → queued tp_osd_tp swarm in and acquire reads
  → user-visible recovery

[T > 0+3s]    Steady state
  → writeback_thread continuously busy, never sleeps deeply
  → btree stays hot
  → closure queue stable
  → ★ no further spikes

[T > 0+N s, business stops]
  → System goes idle again, all 4 factors re-arm
  → Next burst → spike again
```

---

## 8. Mitigation comparison

I tested 4 sysfs-tunable scenarios via an automated A/B test harness with multiple phases:

| Option | Factor it attacks | Spike effect | Side effect |
|---|---|---|---|
| `cache_mode=writethrough` | Bypass the bcache write path entirely | ✅ spike gone | Steady-state write latency jumps from ~5 ms to 50-150 ms (HDD passthrough) |
| `writeback_running=0` | Kill writeback_thread (factor B) | ✅ spike gone | Dirty data never flushes, cache fills up within hours |
| `wb_aggressive` (raise writeback rate) | Keep writeback_thread perpetually busy | ✅ spike heavily reduced | Under sustained load, HDD stays busy, average latency +80% |
| Default (baseline) | — | ❌ persistent spike | — |

**No silver bullet.** Each option has a tradeoff.

### 8.1 wb_aggressive in detail

```bash
# Adjust all three together (per cached_dev)
echo 1     > /sys/block/bcacheN/bcache/writeback_percent              # default 10
echo 40960 > /sys/block/bcacheN/bcache/writeback_rate_minimum         # default 8 (sectors/sec)
echo 1     > /sys/block/bcacheN/bcache/writeback_rate_update_seconds  # default 5
```

**Why it works**: writeback_thread sustains 20+ MB/s instead of the default 4 KB/s, never goes into deep sleep, eliminates factor B.

**Suits**: intermittent burst workloads (typical OLTP, VM IO, test environments running dd).
**Doesn't suit**: sustained high-throughput writes (more SSD write amplification, constantly busy HDD).

### 8.2 Long-term option

Increase `bucket_size` from 512 KB to 2-4 MB. Cost: requires reprovisioning bcache devices (offline data migration). Benefits:

- btree node count drops 4-8×, the whole btree fits in memory permanently
- Fundamentally reduces root contention (fewer splits, shallower tree)
- But this is a hardware/deployment-time decision

---

## 9. Reflection on methodology

### 9.1 The tool chain

| Phase | Tool | Outcome |
|---|---|---|
| Initial triage | `iostat`, `ceph -s`, OSD perf dump | Rule out the standard 5 |
| Stack sampling | High-rate read of `/proc/<tid>/stack` | Find D-state tasks and function offsets |
| Disassembly | `objdump -d bcache.ko` | Translate `+0x3e0` `+0x10c` to specific function calls |
| Lock latency | `bcc-tools funclatency` | Histogram proving 1-2 s lock waits |
| Slow-path stacks | `bcc-tools funcslower -K` | Lock wait + stack as a combined event |
| Custom BPF | Python + BCC API | Falsify hypotheses (cache miss = 0) and verify directly (root lock) |
| A/B validation | Automated sysfs A/B harness | Quantify each fix |

### 9.2 The 4 falsified hypotheses (the heart of the scientific method)

```
Hypothesis 1: writeback_lock direct contention
  → partially right, but only the visible part (amplifier)

Hypothesis 2: btree cache miss
  → falsified (miss = 0)

Hypothesis 3: generic btree node lock contention (any node, anywhere)
  → falsified (child nodes consistently 1-2 μs)

Hypothesis 4: root btree node lock contention (the real root cause)
  → double-proven
```

**Good kernel debugging isn't about guessing right on the first try.** It's about falsifying, refining, falsifying again, until only one possibility remains — and then independently verifying it.

### 9.3 Heuristics for similar problems

1. **Knock out the obvious first** — the 5 standard Ceph causes account for ~80 % of cases.
2. **Break commit_lat down** — the share that's `aio_wait` is a key discriminator.
3. **Don't trust hunches, write scripts to verify them** — hypothesis 2 (cache miss) was popped by hard BPF data.
4. **Disassembly is the final arbiter** — an offset like `+0x3e0` can only be translated by reading the actual machine code.
5. **Independent multi-path verification** — when proof by exclusion and direct readout give the same answer, the conclusion is rock-solid.
6. **Pay attention to time patterns** — "stalls only on idle-then-burst" rules out static contention hypotheses and forces you to look for trigger conditions.
7. **Accept that there's no silver bullet** — every fix has a tradeoff; pick based on actual business profile.

---

## 10. Suggestions for bcache upstream

This case is really an **inherent design limitation** of bcache:

1. **Centralized btree**: one btree per cache_set, one root → unavoidable root contention under concurrency.
2. **rwsem FIFO fairness**: doesn't compose well with bcache's "closure-based async chains + locks held across thread boundaries" pattern.
3. **Default writeback rate too low**: `writeback_rate_minimum=8` (= 4 KB/s) is effectively "off". Relies on the PID controller ramping up, but the PID is slow to react.

Possible kernel improvements:
- Replace root lock with `seqlock` or RCU (readers don't block behind queued writers).
- Avoid holding locks across closure boundaries (restructure release timing).
- Saner default writeback rate (several MB/s instead of KB/s).

That said, bcache has largely been superseded by `bcachefs` (the next-generation rewrite), so upstream momentum is limited. Pragmatic short-term remediation is the sysfs tuning above.

---

## 11. Summary

```
Symptom:  1-3 s business stalls, only triggered by idle-then-burst
Root cause:
  rwsem queueing on bcache's per-cache_set root btree node under
  concurrent writers + 2-stage amplification via writeback_lock's FIFO
  fairness

Evidence:
  · BPF funclatency: 53 lock-waits between 1-4 s in 1 minute
  · BPF funcslower: 194 spike events with complete stacks containing
    cached_dev_make_request+0x3e0
  · Disassembly: the offsets correspond to
    down_read(writeback_lock) and down_read(root->lock)
  · Exclusion + direct readout of c->root: double-proves hot locks are root
  · Falsified hypotheses: btree cache miss = 0; child node locks fast

Fixes:
  · Intermittent workloads: wb_aggressive (raise writeback rate)
  · Emergency: writethrough (heavy cost but stable)
  · Long-term: bucket_size upgrade, or rethink cache architecture
```

The investigation went from a fuzzy "business is slow" to a precise function offset, a specific lock object, and a specific kernel code line, via 7 layers of evidence and 4 hypothesis-falsifications. **This is the joy of kernel debugging** — you think it's a network issue, it turns out to be the disk; you think it's the disk, it turns out to be a lock; you think it's a lock, it turns out to be a specific inline macro + rwsem FIFO + bursty arrival rates layered together.

Hope this case is useful to anyone debugging similar things.

---

## Appendix A: key BPF snippets

### A.1 Proof by exclusion (is the hot lock root?)

```python
# Trace returns of bch_btree_node_get
# Compute &b->lock = b + 0x60 (offset taken from disassembly)
# If the hot address isn't in this set, it must be root.

int kretprobe_get(struct pt_regs *ctx) {
    u64 b = PT_REGS_RC(ctx);
    if (!b) return 0;
    u64 lock_addr = b + 0x60;
    child_locks.update(&lock_addr, ...);
    return 0;
}
```

### A.2 Direct readout of c->root

```python
# Hook __bch_btree_map_nodes(op, c, ...)
# c is arg1, read c->root (offset 0xCE68)

int kprobe_map_nodes(struct pt_regs *ctx) {
    void *c = (void *)PT_REGS_PARM2(ctx);
    void *root;
    bpf_probe_read_kernel(&root, sizeof(root), (char *)c + 0xCE68);
    u64 root_lock = (u64)root + 0x60;
    root_locks.update(&root_lock, ...);
    return 0;
}
```

## Appendix B: key disassembly snippets

```asm
000000000000b5b0 <__bch_btree_map_nodes>:
    b5b0: stp x29, x30, [sp, #-112]!
    ...
    b5d0: mov x21, x1                  ; x21 = cache_set (arg1)
    b5e8: add x22, x1, #0x8, lsl #12   ; x22 = c + 0x8000
    b600: ldr x19, [x22, #20072]       ; x19 = c->root (offset 0xCE68 from c)
    b608: add x25, x19, #0x60          ; x25 = &root->lock
    b60c: ldrb w1, [x19, #194]         ; b->level
    b610: cmp w1, w0                   ; level vs op->lock
    b614: mov x0, x25                  ; x0 = &root->lock
    b618: b.gt b6b8                    ; if level > requested, jump to read path
    b61c: bl <down_write>              ; +0x6c: write lock
    ...
    b6b8: bl <down_read>               ; +0x108: read lock
    b6bc: ldr x0, [x22, #20072]        ; +0x10c (down_read return point) ★
```

## Appendix C: full causal-chain timeline

```
T=0    Business burst (dd 8 GB)
T=0+1  70 tp_osd_tp enter cached_dev_make_request
T=0+1  70 down_read(writeback_lock) succeed (concurrent readers)
T=0+1  70 closures enqueue onto bcache_writeback_wq
T=0+5  70 kworkers run __bch_btree_map_nodes
T=0+5  70 down_read(root->lock) — first succeeds, the other 69 queue
T=0+50 First closure finishes, releases root
T=0+50 Second closure acquires root (possibly down_write for a split)
T=0+100 Dirty surges, writeback_thread wakes up
T=0+100 writeback_thread down_write(writeback_lock) — queues
T=0+100 ★ rwsem FIFO triggers, new down_read also queues
T=0+1500 ~ T=0+3000  double-FIFO blockage period
        - kworkers stuck on root->lock
        - tp_osd_tp stuck on writeback_lock
        - user observes 1-3 s spike
T=0+3000 Lock chain releases, queues drain, spike passes
T>0+5s   Steady state (writeback continuously active), no more spikes
```

---

*Based on a real production case, desensitized. All kernel addresses / PIDs / process names are real data with no sensitive content. bcache source references are against kernel 4.19.*
