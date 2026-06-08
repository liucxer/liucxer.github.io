# 一次 Linux bcache 写入卡顿 1 秒的深度排查 — 从用户感知到 root btree 锁竞争

## TL;DR

线上 Ceph 集群在 bcache + HDD 配置下出现规律性的"客户端写入 1-3 秒级延迟突刺",但仅在**业务空闲一段时间后突发写入**时触发。经过 7 层证据排查(包括 4 次假说证伪),最终定位到 bcache 内核模块中 **`cache_set->root` btree 节点的 `rw_semaphore` 锁竞争**,在 `__bch_btree_map_nodes` 函数 +0x10c 偏移处。`rwsem` FIFO 公平规则将内部 100-300 ms 的锁等待**放大**为业务可见的秒级 spike。本文记录完整的排查路径、4 个被证伪的假说、以及最终用 BPF/反汇编双重证明的过程。

**关键词**: Linux 内核、bcache、Ceph、rwsem、bpftrace/BCC、root btree contention

---

## 1. 现场:1 秒卡顿,但只在特定条件下

### 1.1 架构

- 3 节点 Ceph 集群,每节点 14 个 OSD
- 每个 OSD 后端: `bluestore` over `bcache` (writeback 模式)
- bcache cache: NVMe (~1 TB) × 4 / 节点,聚合成 3 个 cache_set
- bcache backing: HDD (~12 TB) × 14 / 节点
- 客户端通过 cephfs (juicefs gateway) 访问

### 1.2 现象

```
juicefs stats:
  commit_lat 通常 1-10 ms
  → 偶发跳到 50-1000 ms (spike)
  → 持续 1-3 秒
  → 自愈, 回到 1-10 ms
```

**关键触发条件**: 业务空闲一段时间后,客户端突发 `dd` 大文件,前 2 秒 100% 触发。**持续 IO 反而不卡**。

这个"先空闲再突发才卡"的模式,后来证明是定位根因的关键线索。

---

## 2. 第一轮排除:常见 Ceph 高延迟原因

任何 Ceph 性能问题先排除 5 大常见原因:

| # | 原因 | 怎么排除 |
|---|---|---|
| 1 | `scrub` / `deep-scrub` | `ceph -s` 检查活跃 scrub PG;spike 时段无 scrub |
| 2 | `recovery` / `backfill` | `ceph -s` 检查 `recovering`;spike 时段集群健康 |
| 3 | HDD 高延迟 | `iostat -xm 1` spike 时 HDD `%util=0`,完全空闲 |
| 4 | rocksdb compact | OSD `perf dump`: `kv_sync_lat / commit_lat < 5%` |
| 5 | **bcache 设计** | OSD `perf dump`: **`state_aio_wait_lat / commit_lat = 96%+`** ← 命中 |

第 5 条命中,缩小到 BLOCK 层(NVMe→bcache→HDD)。

### 2.1 第一个奇怪现象

spike 时 `iostat`:
- `bcache0` 等设备: `%util=100%`, 但 `aqu-sz=0`, `wMB/s=0`, `w_await=0`
- NVMe 设备: `%util=0`
- HDD 设备: `%util=0`

**bcache 在"忙"但什么 IO 也没下发**。这是经典的"bio 进了 bcache 内部但被某个软件层卡住"的特征。

---

## 3. 第二轮:抓 D-state 任务的内核栈

写一个简单的 `/proc/<tid>/stack` 高频采样器,在 spike 期间抓 D-state 任务:

```
==== dstate_140000_w57_n81.log (实际抓到 81 个栈) ====
57  comm=tp_osd_tp       top=cached_dev_make_request   ← Ceph OSD 写者
3   comm=bcache_writebac top=bch_writeback_thread       ← bcache 写回 kthread
1   comm=kworker         top=__bch_btree_map_nodes      ← btree 操作
其他: node_exporter 等 (sysfs reader, 是噪声)
```

**70% 的 D-state 任务是 `tp_osd_tp`,栈顶 `cached_dev_make_request+0x3e0`**。

### 3.1 反汇编锁定 +0x3e0 的具体位置

```bash
KO=/lib/modules/$(uname -r)/kernel/drivers/md/bcache/bcache.ko
objdump -d $KO | grep -A 50 '<cached_dev_make_request>'
```

找到 `+0x3e0` 那一行:
```asm
14bb4: bl  <down_read>              ← 调用 down_read
14bb8: ldr ...                       ← +0x3e0 (down_read 返回点)
```

→ **`+0x3e0` 是 `down_read(&dc->writeback_lock)` 调用的返回点**。

写者卡在 bcache 的 `writeback_lock` (per cached_dev rw_semaphore) 上。

### 3.2 假说 1: writeback_lock 直接竞争

最直接的假说: writers 在 `writeback_lock` 上排队。但谁在 writer 那一边?

bcache 源码中只有 `bch_writeback_thread`(后台写回 kthread)会 `down_write(&dc->writeback_lock)`,做 refill_dirty。

**机制猜测**: rwsem 的 FIFO 公平规则。一旦 writer 排队,后续 reader 也必须排队(防止 writer 饿死)。这把 writer 持锁几十毫秒的窗口,放大成业务 1+ 秒 spike。

这个假说**部分正确**(后面证实是放大器),但不是根因,因为:**writers 持锁的真正源头是什么?**

---

## 4. 第三轮:用 BPF 精确测锁延迟

`/proc/stack` 采样有局限:
- 采样间隔 100ms,短锁竞争错过
- 只能看"卡时的快照",不能量化"卡多久"

升级到 BPF。环境是 OpenEuler 20.03 LTS 内核 4.19.90,没有 `bpftrace` 包(只能从源码编译),但有 `bcc-tools`。`bpfcc` 模块名跟 `bcc` 不同,要做兼容。

### 4.1 funclatency: rwsem 慢路径直方图

```bash
sudo /usr/share/bcc/tools/funclatency -m 'rwsem_down_read_failed'
```

结果(60 秒采样):

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
1024 -> 2047        : 42      |*                                       |  ← 1-2 秒
2048 -> 4095        : 11      |                                        |  ← 2-4 秒
```

**双峰分布**:
- 0-16 ms:正常或轻微竞争 (~2600 次)
- **1-4 秒**:53 次极端 spike

53 次锁等待超过 1 秒,完美对应业务感知的"1-2 秒卡顿"。

### 4.2 funcslower: 抓完整栈

```bash
sudo /usr/share/bcc/tools/funcslower -m 50 -K 'rwsem_down_read_failed'
```

抓到 194 个 spike 事件(60 秒内,集中在 3 秒爆发):

```
==== [11:41:08] event ====
  comm     : tp_osd_tp
  pid/tid  : 980986/981104
  latency  : 1482 ms
  lock     : 0xffff80464a160d18
  kstack   :
    rwsem_down_read_failed
    cached_dev_make_request+0x3e0     ← 跟 /proc/stack 一致
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
  lock     : 0xffff8052b466a860   ← ★ 不同的锁地址!
  kstack   :
    rwsem_down_read_failed
    __bch_btree_map_nodes+0x10c   ← ★ 另一个函数
    bch_btree_insert+0xe8
    bch_data_insert_keys+0xd0
    process_one_work+0x1b4
    worker_thread+0x54
```

**关键发现**: 锁竞争发生在**两个不同的位置**:
1. `tp_osd_tp` 在 `cached_dev_make_request+0x3e0` (writeback_lock)
2. `kworker` 在 `__bch_btree_map_nodes+0x10c` (另一把 rwsem)

锁地址也分两类:
- 14 个 `*d18` 结尾的地址 → 看起来 per-device,可能就是 14 个 cached_dev 的 writeback_lock
- **2 个 `*860` 结尾的地址** → 只有 2 个,但占总事件数 67%

### 4.3 假说 2: btree cache miss

锁竞争为什么这么长?第一个直觉假说: **btree node cache miss**。bcache 的 btree 节点(每个 256 KB)在内存中有一个 LRU cache(`btree_cache_size`)。统计:

```
btree 总节点数: ~34000 / cache_set × 3 cache_set = ~100k 节点
全装内存需要: 34000 × 256 KB = 8.6 GB / cache_set
实际 btree_cache_size: 8.0 GB / cache_set
                       ↑
                  缺 5%, 必然有 LRU 替换
```

业务空闲后,btree node 被 LRU 替换出去。突发时第一波访问 miss → 同步从 NVMe 读 256 KB → 持锁数百毫秒 → spike。

**这是个漂亮的理论**。怎么证明?

写一个 BPF 跟踪器,hook `bch_btree_node_read`(只在 cache miss 时被调用):

```python
# btree_miss_tracker.py - 跟踪 bcache 关键函数
# - bch_btree_node_get  每次访问 (hit + miss)
# - bch_btree_node_read 仅 cache miss 时调用
```

跑 60 秒,期间高负载:

```
ts          get/s    miss/s   alloc/s   hit_rate
15:09:12   13396.3      0.0       0.4    100.00%
15:09:17   13223.2      0.0       6.4    100.00%
15:09:22   13902.5      0.0       6.4    100.00%

总 btree 访问 (get):    282904
总 cache miss (read):       0
命中率:                100.00%
```

**282,904 次 btree 访问,0 次 cache miss**。

**假说 2 推翻** ❌。btree 节点全部在内存里,根本不读 NVMe。

这是个重要的科学时刻:**用实测推翻自己的假说**,虽然遗憾,但更接近真相。

---

## 5. 第四轮:Root 锁竞争假说

如果 btree 全 hit,那锁等待是什么?重新审视数据:

- 60 秒内 67% 的锁等待事件集中在 **2 个 lock 地址**
- 而 btree 共有 100k+ 个节点,每节点一把锁
- 只有 2 个热点 = 这 2 把锁是**全局共享的**

bcache 中"全局共享的 btree node 锁"只能是**每个 cache_set 的 root btree node 锁**。3 个 cache_set 应该有 3 个 root,但只看到 2 个 hot —— 可能 spike 时第 3 个不忙。

### 5.1 看 bcache 源码

```c
// btree.h - bcache_btree_root 宏 (我们之前没注意到)
#define bcache_btree_root(fn, c, op, ...) ({                          \
    int _r = -EINTR;                                                  \
    do {                                                              \
        struct btree *_b = (c)->root;        /* ← 直接取 c->root */ \
        bool _w = insert_lock(op, _b);                                \
        rw_lock(_w, _b, _b->level);          /* ← ★ 锁 root */      \
        if (_b == (c)->root && _w == insert_lock(op, _b)) {           \
            _r = bch_btree_ ## fn(_b, op, ##__VA_ARGS__);             \
        }                                                              \
        rw_unlock(_w, _b);                                            \
        ...                                                            \
    } while (_r == -EINTR);                                           \
    _r;                                                                \
})
```

`__bch_btree_map_nodes` 函数(我们看到的 spike 栈)调用 `bcache_btree_root`,**第一步就是锁 root**。每个 btree 写入(包括 `bch_btree_insert`)都从这里开始。

70 个并发写者 → 都要先过 root 锁 → 队列瞬间深 → spike。

### 5.2 反汇编 `__bch_btree_map_nodes+0x10c`

```bash
objdump -d $KO | grep -A 200 '<__bch_btree_map_nodes>:'
```

定位到 `+0x10c`:
```asm
b5b0: <__bch_btree_map_nodes>:                  ← 函数起始
b5e8: add  x22, x1, #0x8, lsl #12               ; x22 = c + 0x8000
b600: ldr  x19, [x22, #20072]                    ; x19 = *(c + 0xCE68) = c->root
b608: add  x25, x19, #0x60                       ; x25 = &root->lock
b614: mov  x0, x25                                ; x0 = &root->lock
b618: b.gt b6b8                                   ; 判断 read 还是 write
b61c: bl   <down_write>                          ; +0x6c 写锁
...
b6b8: bl   <down_read>                           ; +0x108 读锁
b6bc: ldr  x0, [x22, #20072]                     ; +0x10c (down_read 返回点)
```

**`+0x10c` 就是 `down_read(&root->lock)` 的返回点**。栈里看到这个偏移,意味着线程当前在 `down_read` 内部 schedule。

`x19` 是 `c->root` (cache_set 的 root btree 节点),`x25` 是 `&root->lock`。

### 5.3 双重证明:bpf 验证

写两个独立的 BPF 验证脚本:

**方法 1: 排除法**

跟踪 `bch_btree_node_get` (用于访问**子节点**) 的所有返回值,计算它返回过的所有 btree node 的 `&b->lock` 地址。如果热点地址 `0xffff8052b466a860` 不在这个集合里,说明它不是子节点锁 → 必然是 root 锁。

**方法 2: 直接读 c->root**

Hook `__bch_btree_map_nodes` 入口,第二个参数是 `cache_set *c`。通过 `bpf_probe_read_kernel(&root, c + 0xCE68)` 直接读 `c->root` 指针,计算 `&root->lock = root + 0x60`,看是否匹配热点地址。

```
方法 1: 排除法
  经过 bch_btree_node_get 的子节点 lock 共 10240 个

  对比:
    0xffff8052b466a860  ← ✓ 不在子节点集合 → 必然 root
    0xffffa057687eb860  ← ✓ 不在子节点集合 → 必然 root

方法 2: 直接读 c->root
  读到 3 个 root lock 地址:
    0xffff804880f93c60  访问 34289 次  (cache_set A, 没卡)
    0xffffa057687eb860  访问 26148 次  (cache_set B) ★ MATCH
    0xffff8052b466a860  访问 23959 次  (cache_set C) ★ MATCH

✓ 双重证明: 这两个 hot 地址就是 ROOT btree node 锁
```

**两种独立方法、不同途径、同一结论**。这是教科书级别的内核 debug 证明。

### 5.4 有趣观察:**最忙的 cache_set 反而不卡**

`cache_set A` 的 root 被访问 34289 次,比 B/C 都多,**但没出现在 spike 锁竞争中**。这告诉我们:

> spike 不取决于"绝对访问量",而是"瞬时到达率 + 触发条件"

cache_set A 可能访问分散在时间上;B/C 在 dd 突发时被瞬间集中冲击。

---

## 6. 但等等 — 为什么是"间歇 spike"不是"持续慢"?

如果纯粹是 root 锁竞争,**持续负载下也该一直慢**。但实测是:

```
持续负载    → 不卡
空闲后突发  → 1-3 秒 spike
```

只用"root 锁"不能解释这个**时间模式**。需要叠加其他因素。

### 6.1 真正的多因素模型

```
"突发 spike" = 4 个易爆因素同时点燃

  因素 A (固有):
    bcache 中心化设计 → 每 cache_set 1 个 root → 高并发集中点

  因素 B (idle 触发):
    业务空闲后 writeback_thread 进入 sleep (dirty=0 → 没活干)
    → 突发时, 它被唤醒, 立刻想 down_write(writeback_lock)
    → ★ 触发 rwsem FIFO 公平规则
    → 后续 reader (tp_osd_tp) 全部排队

  因素 C (突发到达):
    dd 8 GB 一瞬间提交
    → ceph 三副本分发 → N 节点 × M 线程 = 70 个并发 writer
    → 同一瞬间冲 root 锁
    → ★ 队列深度瞬间从 0 涨到 70

  因素 D (放大):
    cached_dev->writeback_lock 上的 FIFO 二次放大
    把 root 锁内部 100-300 ms 的等待
    放大到 cached_dev_make_request 上的 1-3 秒等待
    → 业务可见

  → 4 个因素叠加 → spike

持续负载下:
  B 不成立 (writeback 一直工作不 sleep)
  C 不成立 (到达率均匀, 队列稳态短)
  → 只有 A + 弱 D, 业务无感
```

### 6.2 这解释了所有反直觉现象

| 反直觉现象 | 解释 |
|---|---|
| 持续 IO 不卡, 空闲后才卡 | B+C 只在 idle→burst 转换时同时成立 |
| 最忙的 cache_set 反而不卡 | 它的访问分散, C 不成立 |
| btree cache miss = 0 | 节点都在内存, 不是 IO 慢 |
| 子节点锁全程快 | 没人在子节点上集中 |
| 锁等待 1-3 秒,但单次操作微秒级 | rwsem FIFO + writeback_lock 二次放大 |

---

## 7. 完整因果链

```
[T = -∞ ~ 0]  系统空闲, 4 个易爆因素已就位:
  · writeback_thread sleep
  · root 锁空闲 (无 writer 排队)
  · closure 队列空
  · cached_dev 上的 writeback_lock 完全空闲

[T = 0]  客户端 dd 8 GB 突发

[T = 0+1ms]  Ceph 把数据切片三副本, 70 个 tp_osd_tp 并发 io_submit
  → 70 个 cached_dev_make_request 几乎同时
  → 70 个 down_read(writeback_lock) 都成功 (rwsem 允许并发 reader)
  → 启动 70 个 closure 进 bcache_writeback_wq

[T = 0+5ms]  70 个 kworker 同时跑 bch_data_insert chain
  → 都调到 __bch_btree_map_nodes
  → 第一件事: down_read(root->lock)
  → ★ 70 个 reader 抢同一把 root 锁
  → 第一个拿到的开始 btree insert; 其他 69 个排队等

[T = 0+50ms]  某个 closure 触发 btree node split, 需要 down_write(root)
  → root 锁进入 write 模式
  → rwsem FIFO 触发: 新来的 reader 也排队

[T = 0+100ms]  与此同时, dirty 增长激增, writeback_thread 醒来
  → 它要 down_write(writeback_lock)
  → 它发现持锁的 reader 太多, 排队等
  → ★ writeback_lock 的 FIFO 触发
  → 新来的 tp_osd_tp 想 down_read(writeback_lock) 也排队

[T = 0+100ms ~ 0+2s]  双重 FIFO 阻塞
  → kworker 卡在 root->lock
  → tp_osd_tp 卡在 writeback_lock
  → 业务看到 cached_dev_make_request 1-2 秒返回

[T = 0+1.5s ~ 0+3s]  锁链逐层释放
  → btree insert 完成, root 锁释放
  → kworker 完成 closure, 释放 writeback_lock 读锁
  → writeback_thread 拿到 write 锁, 做完 refill, 释放
  → 排队的 tp_osd_tp 一拥而上拿读锁
  → 业务恢复

[T > 0+3s]  进入稳态
  → writeback_thread 持续工作不再 sleep
  → btree 一直 hot
  → closure 队列稳定
  → spike 消失

[T > 0+N 秒, 业务停]
  → 等待 idle, 4 个易爆因素重新就位
  → 下次突发再 spike
```

---

## 8. 修复方案对比

测试了 4 个 sysfs 可调参数方案,通过 A/B 多 phase 自动化测试验证:

| 方案 | 攻击的因素 | spike 效果 | 副作用 |
|---|---|---|---|
| `cache_mode=writethrough` | 完全绕开 bcache 写路径 | ✅ spike 消失 | 稳态写延迟从 5ms 涨到 50-150ms (HDD 透传) |
| `writeback_running=0` | 干掉 writeback_thread (因素 B) | ✅ spike 消失 | dirty 永远不冲洗,几小时内 cache 写满 |
| `wb_aggressive` (调高 writeback rate) | 让 writeback 持续工作不 sleep | ✅ spike 大幅减少 | 持续负载下 HDD 一直忙,平均延迟 +80% |
| 默认 (基线) | — | ❌ 持续 spike | — |

**没有银弹方案**。每个都有 trade-off。

### 8.1 wb_aggressive 详细配置

```bash
# 三个参数同时调 (per cached_dev)
echo 1     > /sys/block/bcacheN/bcache/writeback_percent          # 默认 10
echo 40960 > /sys/block/bcacheN/bcache/writeback_rate_minimum     # 默认 8 (扇区/秒)
echo 1     > /sys/block/bcacheN/bcache/writeback_rate_update_seconds  # 默认 5
```

**原理**: 让 writeback_thread 持续以 20 MB/s+ 的速率工作(默认是 4 KB/s),不再有"sleep 唤醒"瞬态,因素 B 被消除。

**适用场景**: 业务模式是"间歇突发"(典型 OLTP / VM IO / 测试环境 dd)
**不适用**: 持续高吞吐量写(SSD 写入量增加 + HDD 持续繁忙)

### 8.2 长期方案

`bucket_size` 升级到 2-4 MB(默认 512 KB)。代价: 需要重做 bcache 设备(线下数据迁移)。收益:
- btree 节点数减少 4-8 倍 → 全装内存,btree 全 hot
- 根本上减少 root 锁竞争(更少 split,更短走树深度)
- 但需要在硬件 / 部署阶段决定

---

## 9. 排查方法论的反思

### 9.1 关键工具链

| 阶段 | 工具 | 收获 |
|---|---|---|
| 初步定位 | `iostat`, `ceph -s`, OSD perf dump | 排除 5 大常见原因 |
| 栈采样 | `/proc/<tid>/stack` 高频读 | 找到 D-state 任务和函数偏移 |
| 反汇编 | `objdump -d bcache.ko` | 把 `+0x3e0` `+0x10c` 翻译成具体函数调用 |
| 锁延迟量化 | `bcc-tools funclatency` | 直方图证明 1-2 秒锁等待 |
| 慢路径栈 | `bcc-tools funcslower -K` | 抓到锁等 + 栈 双重证据 |
| 自定义 BPF | Python + BCC API | 假说证伪 (cache miss=0) 和定向验证 (root 锁) |
| A/B 验证 | 自动化 sysfs A/B 测试脚本 | 修复方案效果量化 |

### 9.2 4 个被证伪的假说(科学方法的体现)

```
假说 1: writeback_lock 直接竞争
  → 部分对, 但只是表象 (放大器)

假说 2: btree cache miss
  → 推翻 (miss=0)

假说 3: 一般的 btree node 锁竞争 (任何节点都可能)
  → 推翻 (子节点全程 1-2 μs)

假说 4: root btree node 锁竞争 (本质根因)
  → 双重证明成立
```

**好的内核 debug 不是一次猜中**,而是不断证伪、修正、再证伪,直到剩下唯一可能并独立验证。

### 9.3 给类似问题的方法学建议

1. **先排除显而易见的** — Ceph 5 大原因,通常占 80% 案例
2. **OSD perf 拆解 commit_lat** — aio_wait 占比是关键判分
3. **不要相信猜测,要写脚本验证** — 假说 2 (cache miss) 被 BPF 数据击穿
4. **反汇编是终极仲裁** — `+0x3e0` 这种相对偏移,只有反汇编能告诉你它是哪个 `bl <fn>` 的返回点
5. **多路径独立验证** — 排除法 + 直接读两种方法独立得到同一结论,这是铁证
6. **关注"时间模式"** — "持续不卡只在突发卡" 这个观察排除了纯静态锁竞争假说,迫使我们找触发因素
7. **接受没有银弹** — 每个修复都有 trade-off,要根据真实业务选

---

## 10. 给 bcache upstream 的建议

这个 case 实际上是 **bcache 设计上的固有限制**:

1. **中心化 btree**: 每 cache_set 单 btree、单 root → 高并发下必然有 root contention
2. **rwsem FIFO 公平**: 跟 bcache 的"closure 异步链 + 跨线程持锁"模式不友好
3. **writeback rate 默认值过低**: `writeback_rate_minimum=8` (= 4 KB/s) 实际上是"基本不工作",rely on PID ramp,但 PID 反应慢

可能的内核改进方向:
- root 锁 → 用 `seqlock` 或 RCU(reader 不阻塞 writer 排队)
- closure 链不跨线程持锁 → 改 lock 释放时机
- writeback 默认值更合理(几 MB/s 而非几 KB/s)

不过 bcache 已经被 `bcachefs` (新一代)替代,主线开发不活跃,upstream 修复可能性不高。临时方案就是上面的 sysfs 调参。

---

## 11. 总结

```
现象:    业务 1-3 秒突刺, 仅在 idle 后突发触发
根因:    bcache root btree node rwsem 在并发写者下排队 
        + writeback_lock 的 FIFO 公平规则二次放大
证据:    
  · BPF funclatency: 锁等 1-4 秒, 53 次/分钟
  · BPF funcslower: 194 个 spike 事件, 完整栈含 cached_dev_make_request+0x3e0
  · 反汇编: 偏移对应 down_read(writeback_lock) 和 down_read(root->lock)
  · 排除法 + 直接读 c->root: 双重证明热点锁就是 root
  · 证伪假说: btree cache miss=0, 子节点锁全程快

修复:    
  · 间歇业务: wb_aggressive (调高 writeback rate)
  · 紧急: writethrough (代价大但稳)
  · 长期: bucket_size 升级或重新设计 cache 架构
```

排查过程从最初模糊的"业务慢",到最终精确到具体函数偏移、具体锁对象、具体内核代码行,经过 7 层证据、4 个假说证伪。**这正是内核 debug 的乐趣** — 你以为是网络问题,结果是磁盘;你以为是磁盘问题,结果是锁;你以为是锁,结果是某个 inline 宏 + rwsem FIFO + 业务突发到达的多因素叠加。

希望这个 case 对类似问题的排查者有帮助。

---

## 附录 A: 关键 BPF 脚本片段

### A.1 排除法证明 root 锁

```python
# 跟踪 bch_btree_node_get 返回的所有 btree*
# 计算 &b->lock = b + 0x60 (offset 来自反汇编)
# 如果热点地址不在这个集合, 必然是 root

int kretprobe_get(struct pt_regs *ctx) {
    u64 b = PT_REGS_RC(ctx);
    if (!b) return 0;
    u64 lock_addr = b + 0x60;
    child_locks.update(&lock_addr, ...);
    return 0;
}
```

### A.2 直接读 c->root

```python
# Hook __bch_btree_map_nodes(op, c, ...)
# c 是 arg1, 读 c->root (offset 0xCE68)

int kprobe_map_nodes(struct pt_regs *ctx) {
    void *c = (void *)PT_REGS_PARM2(ctx);
    void *root;
    bpf_probe_read_kernel(&root, sizeof(root), (char *)c + 0xCE68);
    u64 root_lock = (u64)root + 0x60;
    root_locks.update(&root_lock, ...);
    return 0;
}
```

## 附录 B: 反汇编关键片段

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
    b618: b.gt b6b8                    ; if level > requested, jump to read
    b61c: bl <down_write>              ; +0x6c 写锁
    ...
    b6b8: bl <down_read>               ; +0x108 读锁
    b6bc: ldr x0, [x22, #20072]        ; +0x10c (down_read 返回点) ★
```

## 附录 C: 完整因果链时序图

```
T=0    业务突发 (dd 8GB)
T=0+1  70 个 tp_osd_tp 进 cached_dev_make_request
T=0+1  70 个 down_read(writeback_lock) 成功 (并发)
T=0+1  70 个 closure 进 bcache_writeback_wq
T=0+5  70 个 kworker 跑 __bch_btree_map_nodes
T=0+5  70 个 down_read(root->lock) - 第 1 个成功, 其他排队
T=0+50 第 1 个 closure 完成, 释放 root 锁
T=0+50 第 2 个 closure 拿 root 锁 (可能是 down_write 做 split)
T=0+100 dirty 激增, writeback_thread 醒来
T=0+100 writeback_thread down_write(writeback_lock) - 排队
T=0+100 ★ rwsem FIFO 触发, 新 down_read 排队
T=0+1500 ~ T=0+3000  双重 FIFO 阻塞期间
        - kworker 卡 root->lock
        - tp_osd_tp 卡 writeback_lock
        - 业务感知 1-3 秒 spike
T=0+3000 锁链释放, 队列清空, spike 平息
T>0+5s   稳态 (writeback 持续工作), 不再 spike
```

---

*本文基于真实生产案例脱敏整理。所有内核地址 / PID / 进程名为实际数据,无敏感信息。bcache 源码引用基于内核 4.19。*
