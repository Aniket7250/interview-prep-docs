# Batch 5 — Performance Debugging & Kernel Internals
### Staff+ Level | Self-Interrogation + Spot Grading on Operational Questions
---

## Q35. How do you debug high CPU usage?

### 📊 Grading Applied (Operational)

### 🏭 Production Context
High CPU isn't a single failure mode — it's a symptom with 10+ root causes that look identical in `top`. The danger is acting on the wrong signal: vertical-scaling for a lock-contention problem, or optimizing userspace code when the real issue is steal time. At scale, a single CPU-saturated node in a Kubernetes cluster causes all its pods to miss health checks → eviction storm → cascades to neighboring nodes.

### ⚙️ Inner Workings — CPU Time Taxonomy

```
top/vmstat CPU columns:
  %us — userspace computation (your application code)
  %sy — kernel syscalls, context switches, softirq handler time
  %ni — nicely-renice'd process CPU
  %id — idle (available CPU)
  %wa — I/O wait (CPU idle but waiting for I/O to complete)
  %hi — hardware interrupt handling time
  %si — software interrupt (softirq): network RX/TX, timers
  %st — steal time (hypervisor took this CPU away from the VM)
```

Each category points to a different root cause.

### 🔧 Debug Sequence

```bash
# ── Phase 1: Establish which CPU time category is high ──
vmstat 1 10
# Watch: us, sy, wa, st columns
# High us → application computation
# High sy → excessive syscalls or kernel operations
# High wa → I/O bottleneck (not true CPU saturation)
# High st → hypervisor noisy neighbor (cloud issue, not your code)
# High si → softirq: likely network packet storm or timer floods

mpstat -P ALL 1 5
# Is saturation on ALL CPUs (global load) or specific CPUs (IRQ pinning, single thread)?

# ── Phase 2: Find the consuming process ──
top -b -n1 | head -30         # Snapshot sorted by CPU
ps aux --sort=-%cpu | head -20
pidstat 1 5                    # CPU per process per second (includes history)

# ── Phase 3: Inside the process — what is it doing? ──
perf top -p <pid>             # Live: which functions/symbols consuming cycles?
# Look for: userspace function names (app code) vs kernel symbols

# CPU flame graph (the gold standard):
perf record -F 99 -p <pid> -g -- sleep 30
perf script | stackcollapse-perf.pl | flamegraph.pl > /tmp/flame.svg
# Wide bars = where CPU time is spent

# ── Phase 4: Syscall profile ──
perf stat -p <pid> sleep 10
# Instructions per cycle (IPC):
#   IPC < 0.5 → memory-bound (stalls waiting for RAM), not truly CPU-bound
#   IPC > 2.0 → compute-bound (efficient CPU usage)
#   IPC ~1.0 → mixed

strace -c -p <pid>            # Syscall count and time breakdown

# ── Phase 5: Thread-level ──
top -H -p <pid>               # Show individual threads
pidstat -t -p <pid> 1 5      # Per-thread CPU

# ── Phase 6: Hardware-level (for serious optimization) ──
perf stat -e cache-misses,cache-references,branch-misses,cycles,instructions -p <pid>
# High cache-miss rate → working set exceeds L2/L3 cache → data locality issue
# High branch-misses → unpredictable branch patterns → compiler optimization target

# ── Phase 7: Steal time investigation (cloud) ──
vmstat 1 | awk '{print $16}'   # st column
# > 5% steal → hypervisor issue; escalate to cloud provider or migrate instance

# ── Phase 8: Softirq investigation ──
watch -n1 'cat /proc/softirqs | head -5'
# NET_RX growing fast → packet storm, overwhelmed NIC
cat /proc/interrupts | grep eth0
# All interrupts on CPU 0? → set_irq_affinity or enable RSS
```

### 🔍 Follow-up (Level 1)
**Q: `perf top` shows `[kernel.kallsyms]` symbols dominating CPU. The application team says their code is fast. What's happening?**

**A:**
Kernel symbols in `perf top` = the CPU is spending time in kernel code, not userspace. This happens when:

1. **Excessive syscall rate:** `strace -c -p <pid>` — count syscalls/second. If millions of `epoll_wait`, `futex`, `read`, `write` per second → each syscall costs kernel time

2. **Page fault storm:** kernel handling thousands of page faults → `handle_mm_fault()`, `do_wp_page()` prominent in perf top
   ```bash
   perf stat -e major-faults,minor-faults -p <pid>
   ```

3. **High lock contention in kernel:** `mutex_lock`, `rwsem_down_write` in perf top → kernel-level lock contention (e.g., dcache lock from filesystem metadata operations)

4. **Network stack processing:** `napi_poll`, `__netif_receive_skb` → high packet rate overwhelming kernel network stack. Consider DPDK or kernel bypass.

5. **Scheduler overhead:** `__schedule`, `pick_next_task_fair` → too many threads being scheduled. Reduce thread count or use work-stealing scheduler.

```bash
# Identify kernel subsystem:
perf top --call-graph dwarf -p <pid>
# Drill into [kernel.kallsyms] → see full call chain → identify subsystem
```

### 🔥 Follow-up (Level 2)
**Q: CPU is at 100% on all cores. Flame graph shows uniform distribution across all functions. No single hotspot. What does this tell you?**

**A:**
Uniform distribution with no hotspot = the system is doing proportional work across many code paths. This is NOT a code optimization problem — it's a **capacity** problem.

Interpretations:
1. **Legitimate load spike:** more traffic → more of everything. No single bottleneck. Fix: horizontal scale.
2. **Thundering herd:** many identical processes woke simultaneously, each doing small work. Cumulative = 100% CPU. Fix: stagger wake-ups, add jitter.
3. **GC pressure (managed languages):** garbage collector is running on all cores simultaneously alongside application threads. Flame graph shows ~20% GC, 80% app evenly distributed.
   ```bash
   # JVM GC:
   jstat -gcutil <pid> 1000    # GC time percentage
   # Go GC:
   GODEBUG=gctrace=1 ./app 2>&1 | grep "gc "
   ```
4. **Healthy system at capacity:** sometimes 100% CPU with uniform flame graph just means "you need more CPUs." Use `perf stat` IPC > 2 to confirm compute-bound (not memory-bound) before scaling.

---

### 📊 Grading

| Dimension | Score |
|---|---|
| Production Realism | 9 |
| Technical Depth | 9 |
| Debugging Practicality | 10 |
| Failure Awareness | 8 |
| Clarity & Signal |  9 |

**Total: 45/50**

**🔍 Critical Feedback:**
- Missing CPU frequency throttling (`turbostat`, `cpupower`) — thermal/power cap looks identical to steal time
- Should mention `tperf` or `bpf_prog_test_run` for eBPF-based profiling in production without perf overhead
- The steal time section is good but doesn't mention that AWS Graviton instances have better steal time visibility via CloudWatch

---

## Q36. How do you identify which thread is consuming CPU?

### 🏭 Production Context
"The process is using 400% CPU" (on 4-core machine) — which of 200 threads is responsible? Thread-level attribution is critical for Java services, Go runtimes, and C++ thread pools.

### 🔧 Thread-Level Debug

```bash
# Method 1: top with thread view
top -H -p <pid>
# -H flag shows individual threads (LWPs) sorted by CPU
# Note the PID of the high-CPU thread (it's the thread ID / LWP)

# Method 2: pidstat per thread
pidstat -t -p <pid> 1 5
# %CPU per thread, with thread name

# Method 3: ps with thread list
ps -p <pid> -L -o pid,lwp,pcpu,comm | sort -k3 -rn | head -10
# lwp = lightweight process (thread) ID
# Compare lwp to /proc/<pid>/task/<lwp>/

# Method 4: Convert thread ID to stack trace
# Get thread LWP ID from top -H:
TID=<thread_lwp_id>
cat /proc/<pid>/task/<TID>/wchan    # What kernel function sleeping in?
cat /proc/<pid>/task/<TID>/status   # State, signals

# Method 5: perf record per thread
perf record -F 99 -t <TID> -g -- sleep 10
perf report

# Method 6: Language-specific thread identification
# Java: correlate with jstack
JAVA_TID_HEX=$(printf '%x' <TID>)
jstack <java_pid> | grep -A30 "nid=0x${JAVA_TID_HEX}"
# Shows thread name, state, and stack trace

# Go: goroutine dump
kill -SIGQUIT <go_pid>   # Go prints goroutine dump to stderr
# Or: pprof endpoint
curl http://localhost:6060/debug/pprof/goroutine?debug=2

# Python: py-spy (sampling profiler, no code changes needed)
py-spy top --pid <pid>   # Live thread view
py-spy dump --pid <pid>  # Stack traces for all threads
```

### 🔍 Follow-up (Level 1)
**Q: Java thread shows 95% CPU in `top -H`. `jstack` says it's in RUNNABLE state at `sun.misc.Unsafe.park()`. Is it actually running?**

**A:**
Misleading signal. `RUNNABLE` in Java ≠ actually executing. Java's `RUNNABLE` state maps to multiple Linux states:
- Thread actually on CPU (Linux RUNNING)
- Thread waiting to be scheduled (Linux RUNNABLE — in run queue)
- Thread in a native `park()` (which on Linux is `futex_wait`) — Linux SLEEPING but Java considers it "runnable" because it can be unparked

`sun.misc.Unsafe.park()` is the underlying implementation of `LockSupport.park()` — used by Java concurrency primitives (locks, `CompletableFuture.get()`, `BlockingQueue.take()`). Thread is actually BLOCKED waiting for a signal.

Why high CPU then? park/unpark loop — thread wakes, finds condition not met, parks again. Fast cycle = busy-wait pattern disguised as parking:
```bash
# Confirm with strace:
strace -p <TID> -e trace=futex 2>&1 | head -20
# futex calls cycling rapidly = busy-wait via park/unpark
```

Real RUNNING threads show:
```bash
cat /proc/<pid>/task/<TID>/stat | awk '{print $3}'
# R = actually running on CPU
# S = sleeping (even if Java says RUNNABLE)
```

---

## Q37. How do you debug high memory usage?

### 📊 Grading Applied (Operational)

### 🏭 Production Context
"Memory is high" is one of the most common and most misdiagnosed production issues. High RSS ≠ memory leak. High VIRT ≠ actual problem. The correct question: is memory growing over time, or is it stable at a high level?

### 🔧 Debug Sequence

```bash
# ── Phase 1: Establish baseline — is it growing? ──
watch -n30 'cat /proc/<pid>/status | grep VmRSS'
# Monotonically growing: leak or unbounded cache
# Stable high: legitimate working set or fragmentation

# ── Phase 2: Memory breakdown ──
cat /proc/<pid>/smaps_rollup
# Rss, Pss, Shared_Clean, Shared_Dirty, Private_Clean, Private_Dirty, Swap, Anonymous
# High Anonymous = heap/stack (your allocations)
# High Shared_Clean = shared libraries (not a leak, not your RAM)

smem -p -k -P <process_name>
# Shows USS (unique = only yours) vs RSS (includes shared)
# Growing USS over time = definitive leak

# ── Phase 3: Map breakdown ──
pmap -x <pid> | sort -k3 -rn | head -20   # Sort by RSS, show top regions
cat /proc/<pid>/maps   # All VMA regions with permissions

# ── Phase 4: Heap analysis ──
# What's using the heap?

# C/C++ with glibc:
gdb -p <pid>
(gdb) call malloc_stats()    # Print heap stats to stderr of target
# Or: malloc_info() via gdb for XML report

# For production without gdb (heap profiling):
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libmemchecked.so ./app
# Or valgrind (not production-safe, but staging):
valgrind --leak-check=full --track-origins=yes ./app

# ── Phase 5: Language-specific ──
# Java:
jmap -histo:live <pid> | head -30    # Live objects by type
# If OutOfMemory risk:
jmap -dump:format=b,file=/tmp/heap.hprof <pid>
# Analyze with Eclipse MAT → "Dominator tree" shows what holds most memory

# Go:
curl http://localhost:6060/debug/pprof/heap > heap.pprof
go tool pprof heap.pprof
(pprof) top20       # Top 20 allocation sites by size
(pprof) web         # Open flame graph in browser

# Python:
# tracemalloc (stdlib):
import tracemalloc
tracemalloc.start()
# ... run code ...
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')
# memory_profiler:
@profile  # decorator
def my_func(): ...

# ── Phase 6: Kernel memory (not process) ──
cat /proc/meminfo | grep -E "Slab|KernelStack|PageTables"
slabtop -o   # Kernel slab cache usage
# Growing slab cache → kernel-level leak (often from driver or filesystem)
```

### 🔍 Follow-up (Level 1)
**Q: RSS is stable but the OOM killer keeps firing for other processes on the same node. How does one stable-RSS process cause OOM for others?**

**A:**
The stable-RSS process may be consuming:
1. **Huge amounts of page cache** — if the process mmap()s large files without `MADV_DONTNEED`, the kernel's page cache fills with its file pages. Other processes can't allocate memory → OOM.
   ```bash
   cat /proc/<pid>/smaps_rollup | grep -E "Shared_Clean|Shared_Dirty"
   # Large shared_clean = file-backed pages in page cache
   ```

2. **Kernel memory (Slab)** — a process that opens millions of small files or sockets causes kernel slab allocations (dentry cache, inode cache) that don't show in RSS but consume system RAM
   ```bash
   slabtop -o | head -10   # Is dentry/inode slab growing?
   ```

3. **Page table memory** — a process with many mmap regions (Java with many memory-mapped files) has large page table overhead that isn't in RSS but IS in system memory
   ```bash
   cat /proc/<pid>/status | grep VmPTE   # Page table size
   ```

4. **Shared memory segments** — a process with large System V shared memory that no one else accounts for
   ```bash
   ipcs -m   # System V shared memory segments
   cat /proc/meminfo | grep Shmem
   ```

---

### 📊 Grading

| Dimension | Score |
|---|---|
| Production Realism | 9 |
| Technical Depth | 9 |
| Debugging Practicality | 9 |
| Failure Awareness | 8 |
| Clarity & Signal | 8 |

**Total: 43/50**

**🔍 Critical Feedback:**
- Missing `perf mem` for memory access profiling — important for NUMA-related memory issues
- Should mention `cgroup memory.stat` for container-specific memory debugging (not just process-level)
- The Slab discussion is good but doesn't mention `vm.drop_caches=3` as a diagnostic tool

---

## Q38. How do you detect a memory leak?

### 🏭 Production Context
Memory leaks in production are characterized by: gradual RSS growth, periodic OOM kills (then restart, then grow again), no single obvious culprit in heap dump. The diagnosis window matters: catch it at 1GB leak vs 10GB crash.

### 🔧 Detection Pattern

```bash
# ── Definitive signal: USS growth over time ──
# Sample USS every 5 minutes for 30 minutes
for i in $(seq 1 6); do
    echo "$(date): $(cat /proc/<pid>/smaps_rollup | grep Pss | awk '{sum+=$2} END{print sum}') kB PSS"
    sleep 300
done
# Consistently growing PSS = leak

# ── Automated leak detection ──
# Create a baseline:
cat /proc/<pid>/smaps_rollup | grep -E "Rss|Pss|Anonymous" > /tmp/mem_baseline.txt
sleep 3600  # 1 hour of traffic
cat /proc/<pid>/smaps_rollup | grep -E "Rss|Pss|Anonymous" > /tmp/mem_t1.txt
diff /tmp/mem_baseline.txt /tmp/mem_t1.txt
# Anonymous growing? = heap leak
# Shared growing? = shared library or page cache issue

# ── Java: heap leak fingerprint ──
# GC runs but heap doesn't recover → objects not being collected → leak
jstat -gcutil <pid> 5000 12   # GC stats every 5s for 1 minute
# Old gen (O) growing despite full GC → strong retention chain keeping objects alive
# jmap dump + Eclipse MAT → "Unreachable objects" report

# ── Go: heap profiling ──
# Enable pprof in code:
import _ "net/http/pprof"
# Capture base:
curl http://localhost:6060/debug/pprof/heap > heap1.pprof
sleep 1800
curl http://localhost:6060/debug/pprof/heap > heap2.pprof
# Diff:
go tool pprof -base heap1.pprof heap2.pprof
(pprof) top    # What GREW between snapshots?

# ── C/C++: AddressSanitizer (non-production, staging) ──
gcc -fsanitize=address,leak -g myapp.c -o myapp
./myapp
# LeakSanitizer output on exit: leaked bytes + allocation stack traces

# ── valgrind massif (heap profiler) ──
valgrind --tool=massif ./myapp
ms_print massif.out.<pid> | head -50
```

### 🔍 Follow-up (Level 1)
**Q: Java heap dump shows no obvious leak — all objects are reachable and make sense. But RSS keeps growing. What else could be leaking?**

**A:**
Java heap is not the only memory Java uses. Off-heap (native memory) can leak:
1. **Direct ByteBuffers** (`ByteBuffer.allocateDirect()`) — allocated outside JVM heap, managed by JVM's native memory system. If not freed (GC collects the ByteBuffer object but deallocation of native memory is triggered via `Cleaner` — which may be delayed or prevented by a GC quirk)
   ```bash
   # JVM native memory tracking:
   java -XX:NativeMemoryTracking=detail -jar app.jar
   jcmd <pid> VM.native_memory
   # Shows: Java Heap, Class, Thread, Code, GC, Internal, Other
   # "Other" or "Internal" growing → native memory leak
   ```

2. **JNI (native code)** — Java calling C/C++ via JNI; if native code mallocs without freeing → RSS grows, heap dump shows nothing

3. **Metaspace** — class metadata. If application generates classes dynamically (Groovy scripts, bytecode manipulation) and classloader isn't GC'd → Metaspace grows
   ```bash
   jstat -gcmetacapacity <pid>   # Metaspace usage
   ```

4. **Thread stacks** — each Java thread has a native stack (1MB default). Thread leak → RSS growth. `jstack <pid> | grep "java.lang.Thread.State" | wc -l` — thread count growing?

5. **JIT compiled code cache** — code cache can grow if many methods are JIT compiled. Usually bounded but worth checking.

---

## Q39. How do you debug high disk I/O?

### 📊 Grading Applied (Operational)

### 🏭 Production Context
High disk I/O manifests as: high I/O wait (CPU `%wa`), elevated read/write latency, service timeouts that look like CPU or network but are actually waiting for disk. At scale: a single "noisy" process saturating disk causes ALL processes on the node to experience I/O latency — not just the guilty one.

### 🔧 Debug Sequence

```bash
# ── Phase 1: Confirm it's I/O (not CPU) ──
vmstat 1 5 | awk 'NR>1 {print "wa:", $16, "bi:", $9, "bo:", $10}'
# wa (I/O wait) > 20% = significant I/O
# bi = blocks read per second, bo = blocks written per second

iostat -x 1 5
# %util: device utilization (100% = saturated)
# await: average wait time ms (>10ms for HDD, >1ms for SSD = concerning)
# r/s, w/s: read/write operations per second
# rkB/s, wkB/s: throughput

# ── Phase 2: Which device is saturated? ──
iostat -x -d 1
# Multiple devices shown; find which has highest %util

# ── Phase 3: Which process is doing the I/O? ──
iotop -oP           # Show only processes with active I/O
# -o: only active, -P: show process (not thread) level
# Sort by read/write rate

pidstat -d 1 5      # I/O stats per process per second
# Column: kB_rd/s, kB_wr/s, kB_ccwr/s (cancelled writes — CoW)

# ── Phase 4: What is the process reading/writing? ──
lsof -p <pid> | grep REG    # Open regular files
strace -e trace=read,write,pread64,pwrite64 -p <pid> 2>&1 | head -20
# Shows which FDs are being read/written and how many bytes

# ── Phase 5: eBPF — file-level I/O attribution ──
# biotop: block I/O top (eBPF-based)
biotop 1    # From bcc tools — shows process + file path for each I/O op

# bpftrace: trace I/O by file:
bpftrace -e '
tracepoint:block:block_rq_issue {
    @[comm, args->rwbs] = sum(args->bytes);
}
interval:s:5 { print(@); clear(@); }
'

# ── Phase 6: I/O patterns ──
# Random vs sequential matters enormously for HDD
blktrace -d /dev/sda -o - | blkparse -i -   # Raw block trace
# Or simpler:
iowatcher -d /dev/sda -t 60 -o trace.html   # Visual I/O timeline

# ── Phase 7: Filesystem-level stats ──
# Cache effectiveness:
cat /proc/meminfo | grep -E "Dirty|Writeback|Cached"
# High Dirty pages → lots of pending writes (writeback lag)
# High Writeback → currently flushing to disk

# Dirty page tuning:
cat /proc/sys/vm/dirty_ratio           # % RAM that can be dirty before sync write blocks
cat /proc/sys/vm/dirty_background_ratio # % RAM at which background writeback starts
```

### 🔍 Follow-up (Level 1)
**Q: `iostat` shows 100% utilization on the disk. But the I/O rate (kB/s) is low. How does this happen?**

**A:**
100% utilization with low throughput = **IOPS-limited by latency**, not throughput-limited.

`%util` in iostat = percentage of time the device had at least one request pending. A device can be 100% "utilized" with just 1 request every 10ms (100 IOPS) — because each request takes the full 10ms (HDD seek time). Even though throughput is low (100 IOPS × 4KB = 400KB/s), the device is busy 100% of the time.

This pattern = **random small I/O on a spinning disk:**
```bash
iostat -x 1 | awk '/sda/ {print "IOPS:", $4+$5, "latency:", $10, "util:", $14}'
# Low kB/s but high await + 100% util = random I/O latency problem
```

Solutions:
1. Switch to SSD (random I/O: HDD ~100 IOPS, SSD ~100,000 IOPS)
2. Add I/O scheduler tuning: `echo mq-deadline > /sys/block/sda/queue/scheduler` (better for databases than `cfq`)
3. Application-level: coalesce small writes into larger sequential writes
4. Add caching layer (Redis/Memcached) to reduce disk hits

### 🔥 Follow-up (Level 2)
**Q: A database is doing `fsync()` after every write. You see 100% disk util during write spikes. `fsync` ensures durability — can you reduce its frequency without losing data guarantees?**

**A:**
`fsync()` = flush kernel write buffers + ensure storage device writes to durable media. It's expensive because it forces serialization: next write can't start until current fsync confirms.

Approaches without losing durability:
1. **Group commit:** batch multiple transactions' fsyncs together. PostgreSQL does this by default. Instead of N fsyncs for N transactions, 1 fsync confirms N transactions. Reduces fsync rate by N× with no data loss.
   ```postgresql
   -- PostgreSQL: synchronous_commit = local (fsync on primary, async on replica)
   -- Durability preserved on primary
   ```

2. **Battery-backed write cache (BBWC) / nvRAM:** storage controller with battery-backed cache can acknowledge fsync immediately (data is safely in persistent cache). `fsync()` returns fast; controller writes to disk independently.

3. **Journaling filesystem (ext4, xfs):** journal commits can batch multiple file fsyncs. `data=ordered` (default) is safe without full data journaling overhead.

4. **io_uring with IOSQE_IO_LINK:** chain operations so fsync only applies to the final operation in a group.

5. **PostgreSQL-specific:** `synchronous_commit = off` — transaction confirmation before fsync. Risk: up to 3×`wal_writer_delay` (default 200ms) of transactions lost on crash. Accept for non-critical data.

---

### 📊 Grading

| Dimension | Score |
|---|---|
| Production Realism | 9 |
| Technical Depth | 9 |
| Debugging Practicality | 10 |
| Failure Awareness | 8 |
| Clarity & Signal | 8 |

**Total: 44/50**

**🔍 Critical Feedback:**
- Missing `blktrace` + `seekwatcher` combo for I/O pattern visualization — a real production tool
- Should mention NVMe-specific tools (`nvme-cli`, `nvme smart-log`) for modern storage debugging
- The `dirty_ratio` section is correct but doesn't explain the write-back cliff: when dirty pages hit `dirty_ratio`, all writing processes BLOCK until writeback catches up — a very surprising latency spike

---

## Q40. What is IO wait?

### ⚙️ Inner Workings

I/O wait (`%wa` in vmstat/top) = percentage of time the CPU was IDLE AND there was at least one pending disk I/O request.

**Critical nuance:** I/O wait is NOT time the CPU is busy doing I/O. It's idle time that happens to coincide with pending I/O.

```
CPU states during I/O wait:
  - Process calls read() on a file not in page cache
  - Kernel submits I/O request to block device layer
  - Process enters D state (TASK_UNINTERRUPTIBLE — waiting for I/O)
  - Kernel has no other runnable tasks → CPU goes idle
  - While CPU is idle AND disk I/O is pending → this counts as %wa
```

**Misleading scenarios:**
- `%wa = 0` despite heavy I/O: if the CPU always has other work to do while I/O is pending → no "empty" cycles → wa = 0, but I/O is still slow
- `%wa = 50%` with fast SSD: the process is waiting for I/O, but the wait is only 0.1ms per request — still shows up as wa

```bash
# wa is NOT a saturation metric by itself
# Combine with:
iostat -x 1     # Device await time and utilization
iotop           # Which process is waiting?
vmstat 1 | awk '{print $9, $10, $16}'   # bi, bo, wa together
```

### 🔍 Follow-up (Level 1)
**Q: System shows `%wa = 80%` but `iostat` shows disk at 20% utilization. Contradiction?**

**A:**
Not a contradiction. I/O wait = CPU idle with at least one I/O pending. If MANY processes are all waiting for I/O from a slow disk (NFS timeout, for example), and the disk utilization is low (it's just slow to respond, not throughput-saturated):

- 20 processes all waiting for NFS reads
- NFS server is slow (network latency) — not disk throughput limited
- CPU has nothing else to do → 80% wa
- But NFS's "disk" is seen as a network device → doesn't show high utilization in block iostat

```bash
iostat -x 1    # Local block devices — won't show NFS
nfsiostat 1    # NFS-specific I/O stats (from nfs-utils package)
# Shows: ops/s, rpc backlog, RTT (round-trip time for NFS operations)
```

This is the "NFS stall" pattern: applications hang, wa is high, local disk looks fine.

---

## Q62. What happens when system load is high?

### ⚙️ Inner Workings

High load = tasks waiting to run. The "load average" measures the number of tasks in **runnable** (R) or **uninterruptible sleep** (D) state, averaged over 1/5/15 minutes.

When load is high:
1. Run queue fills with runnable tasks
2. Scheduler must make more decisions per second (increased `%sy` from scheduling overhead)
3. Context switching increases (each task gets smaller CPU time slice)
4. Latency for any individual task increases (waits longer in queue before being scheduled)
5. Memory pressure: more tasks = more active working sets = more page cache pressure
6. If D-state tasks dominate: load high but CPU idle — I/O bound

```bash
uptime
# 10:00:00 up 5 days, load average: 15.20, 12.10, 10.50
# System with 8 CPUs: load 15 = overloaded (15/8 = ~2× capacity)
# System with 16 CPUs: load 15 = comfortably within capacity

cat /proc/loadavg   # Raw load average + running/total tasks
# 15.20 12.10 10.50 3/450 12345
# "3/450" = 3 running right now / 450 total tasks
```

### 🔍 Follow-up (Level 1)
**Q: Load average is 50 on an 8-core system. But CPU shows 20% usage and I/O wait is 0%. What state are those tasks in?**

**A:**
This is the D-state trap. Load average counts D-state tasks. If 40+ tasks are in uninterruptible sleep (D) waiting for something that is NOT I/O wait (e.g., a kernel mutex, NFS, memory compaction), load is high but CPU and I/O appear idle.

```bash
# Find D-state processes:
ps aux | awk '$8 == "D" {print}'
# wchan (what are they sleeping on?):
ps -eo pid,state,wchan:40,comm | grep "^.* D "

# Common D-state wchans:
# "down_read" → rwsemaphore (kernel lock)
# "do_page_fault" → page fault waiting for memory compaction
# "nfs_wait_atomic" → NFS stall
# "jbd2_log_wait_commit" → ext4 journal commit wait
```

---

## Q63. What is load average (1, 5, 15)?

### ⚙️ Inner Workings

Load average is an **exponentially weighted moving average** of the count of runnable + uninterruptible tasks, sampled every 5 seconds.

- **1 minute:** most recent, spiky, reacts quickly to bursts
- **5 minute:** medium-term trend
- **15 minute:** long-term trend

**Reading the trend:**
```
1min > 15min: load is INCREASING (getting worse)
1min < 15min: load is DECREASING (recovering)
1min ≈ 15min: stable (sustained load or sustained idle)

Example: 20.5, 10.2, 5.1
→ Load tripled recently — something just happened
→ Investigate: recent deployment? cron job? traffic spike?

Example: 5.1, 10.2, 20.5
→ Load dropping significantly
→ System is recovering from a past incident
```

**Rule of thumb:** multiply CPU count to normalize
```bash
CPUS=$(nproc)
LOAD=$(uptime | awk '{print $NF}')
echo "Load ratio: $(echo "$LOAD / $CPUS" | bc -l)"
# > 1.0: overloaded
# 0.7-1.0: high but manageable
# < 0.7: headroom available
```

---

## Q64. Difference between CPU-bound and IO-bound processes?

### ⚙️ Inner Workings

**CPU-bound:**
- Process's bottleneck is the CPU itself
- Spends most time in R (running) state
- Adding more CPUs helps; faster disk doesn't
- Signature: high `%us` or `%sy`, low `%wa`, high `r` column in vmstat
- Examples: video encoding, cryptographic operations, ML inference, sorting large datasets in memory

**I/O-bound:**
- Process's bottleneck is waiting for I/O (disk, network)
- Spends most time in D (uninterruptible sleep) or S (interruptible sleep)
- Faster disk helps; more CPUs (for this process) doesn't
- Signature: high `%wa`, high `bi`/`bo` in vmstat, low CPU %
- Examples: database with cache miss, log file tailing, network-heavy services

**Detection:**
```bash
# CPU-bound:
perf stat -p <pid>
# IPC > 1.5, low cache-miss rate → compute limited

vmstat 1 | awk '{print "r:", $1, "b:", $2, "wa:", $16}'
# r (runnable) high, b (blocked) low, wa low → CPU-bound

# I/O-bound:
vmstat 1 | awk '{print "b:", $2, "wa:", $16, "bi:", $9}'
# b (blocked in I/O) high, wa high → I/O-bound

# Mixed:
# Some processes CPU-bound, others I/O-bound
mpstat -P ALL 1    # Check if specific CPUs saturated
```

### 🔍 Follow-up (Level 1)
**Q: A database query is both CPU-bound (sort) and I/O-bound (table scan). How do you optimize it?**

**A:**
Profile to find the dominant bottleneck:
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
-- "Buffers: shared hit=X read=Y" → hit = cache (CPU time for processing), read = disk I/O
-- "Sort Method: external merge  Disk: XkB" → sort spilled to disk → I/O bound
```

If I/O dominates: add index (reduce table scan), increase `shared_buffers` (more cache hits), partition table.
If CPU dominates: optimize query plan, reduce result set earlier (WHERE clause), use parallel query (`max_parallel_workers`).

In practice: fix I/O issues first (they're often 10×–100× more impactful than CPU optimization), then profile CPU hotspots.

---

## Q65. What is context switching?

### ⚙️ Inner Workings

Context switch = CPU switches from executing one task to another. The kernel:
1. Saves current task's register state (CPU registers, stack pointer, program counter) to its kernel stack
2. Loads the next task's saved register state
3. Switches memory mapping (update CR3 if different process — triggers TLB flush)
4. Resumes next task

**Voluntary vs Involuntary:**
- **Voluntary:** task calls `sleep()`, `wait()`, blocks on I/O → gives up CPU voluntarily. Low cost.
- **Involuntary:** scheduler preempts a running task (timer interrupt, higher-priority task) → task didn't want to stop. Higher cost (may be in the middle of a hot loop, cache warm).

**Cost:**
- Register save/restore: ~100–300ns
- TLB flush (if process switch): ~50–200ns additional
- Cache warming for new task: indirect cost (new task's working set not in L1/L2 cache)

```bash
# Context switch rate
vmstat 1 | awk 'NR>2 {print "cs:", $12}'    # cs column = context switches/sec
pidstat -w 1 5   # Per-process: cswch/s (voluntary) + nvcswch/s (involuntary)

# High involuntary context switches → scheduler preempting tasks frequently
# Cause: too many runnable tasks vs CPUs, or real-time process preempting batch tasks
```

### 🔍 Follow-up (Level 1)
**Q: A service has 500 threads but only 8 CPUs. Context switching overhead is significant. How do you reduce it without reducing parallelism?**

**A:**
500 threads on 8 CPUs = at any moment, 492 threads waiting. Scheduler must do ~500 decisions per second (each thread gets ~16ms time slice → 500/8 CPUs × 62.5 switches/second/CPU).

Approaches:
1. **Reduce thread count — use async I/O (epoll/io_uring):** 1 thread handles 1000 concurrent connections via event loop. No thread-per-connection overhead. Node.js, Nginx model.

2. **Thread pool sizing:** optimal = 1 thread per CPU for CPU-bound, higher for I/O-bound (Amdahl's law). For I/O-bound: `thread_count = CPUs / (1 - blocking_fraction)`. 80% blocking → `8 / 0.2 = 40 threads` optimal.

3. **CPU pinning (isolcpus):** dedicate CPUs to specific threads. No scheduler migration = no cross-CPU TLB invalidation
   ```bash
   taskset -cp 0-3 <pid>   # Pin process to CPUs 0-3
   ```

4. **Go goroutines:** M:N threading — 100k goroutines mapped to 8 OS threads. Go scheduler handles context switching in userspace (cheap, no kernel involvement).

---

## Q66. What is the difference between kernel space and user space?

### ⚙️ Inner Workings

**User Space (Ring 3):**
- Where application code runs
- CPU has restricted access: cannot directly access I/O ports, cannot modify page tables, cannot change CPU control registers
- Memory is virtual and protected — process A cannot read process B's memory
- Privilege level: CPL = 3 (lowest)

**Kernel Space (Ring 0):**
- Where kernel code runs
- Full hardware access: can access any physical memory, any I/O port, modify CPU control registers (CR3 for page tables, MSRs)
- All processes share the same kernel address space (mapped into upper portion of each process's virtual address space — pre-KPTI)
- Privilege level: CPL = 0 (highest)

**The boundary:**
```
User space → SYSCALL instruction → Kernel space
Kernel space → SYSRET → User space
Hardware interrupt → Kernel space (IRQ handler)
Page fault, divide-by-zero → Kernel space (exception handler)
```

**Why separation matters:** A bug in user space → SIGSEGV to that process. A bug in kernel space → kernel panic (the whole system crashes). This isolation protects system stability from buggy applications.

---

## Q67. Why is kernel space protected?

### ⚙️ Inner Workings

**Hardware enforcement:**
- CPU runs in Ring 3 (user) normally
- Ring 3 cannot execute privileged instructions (`HLT`, `LGDT`, `MOV CR3`) — CPU raises General Protection Fault if attempted
- Ring 3 cannot access kernel memory pages — page table entries for kernel space have User/Supervisor bit set to Supervisor; Ring 3 CPU generates page fault if it tries to access

**Why necessary:**
1. **Stability:** kernel manages all hardware. One process crashing the kernel kills everything. Protection = process bugs stay isolated.
2. **Security:** kernel manages access control (file permissions, network policy). If any process could modify kernel structures → security model collapses.
3. **Multiplexing:** kernel is the arbiter of shared resources. Allowing direct hardware access would cause conflicts (two processes writing to same disk sector).

**Meltdown (2017) — when this protection failed:**
- Speculative execution: CPU speculatively reads kernel memory before checking permissions
- Even though read is not committed (permission violation detected), the data is cached in L1
- An attacker could use timing side-channels to infer kernel memory contents via L1 cache timing
- KPTI was the fix: separate page tables for user/kernel so kernel pages are not even mapped during user execution

---

## Q68. What is syscall overhead?

### 🏭 Production Context
At high throughput, syscall overhead is measurable. A service making 1 million syscalls/second × 1µs overhead = 1 CPU core consumed entirely by syscall overhead, doing zero application work.

### ⚙️ Overhead Components

```
1. SYSCALL instruction: CPU mode switch (Ring 3 → Ring 0)
   ~10-20ns (register save, mode switch, MSR write)
   
2. KPTI page table switch (post-Meltdown):
   ~50-100ns (CR3 write + TLB flush if no PCID)
   
3. Argument validation (copy_from_user):
   Variable: copying user buffers into kernel is expensive for large payloads
   
4. Actual kernel work: variable (0ns for gettimeofday via VDSO, ms for fsync)

5. Return (SYSRET):
   ~10-20ns (mode switch back, page table switch back)

Total minimal syscall: ~200ns (with KPTI, no PCID)
With PCID (modern hardware): ~50-100ns
VDSO shortcuts: ~10-20ns (no mode switch)
```

### 🔧 Reduce Syscall Overhead

```bash
# 1. Batch syscalls (io_uring)
# Instead of 1000 × write() = 1000 syscalls:
# 1 × io_uring_enter() with 1000 operations queued = 1 syscall

# 2. Use sendmmsg/recvmmsg for UDP batching
# Instead of N × sendmsg() = N syscalls:
sendmmsg(fd, msgs, N, 0);   # 1 syscall sends N UDP packets

# 3. Large reads/writes (reduce count)
# Instead of 1000 × read(fd, buf, 4096):
read(fd, buf, 4096000);     # 1 syscall reads 4MB (fewer syscalls for same data)

# 4. mmap instead of read (no copy_from_user for reads):
mmap(NULL, size, PROT_READ, MAP_SHARED, fd, 0);
# Access mapped data directly — no syscall per access (page fault on first access only)

# Measure current syscall rate and overhead:
perf stat -e syscalls:sys_enter -p <pid> sleep 10
```

### 🔍 Follow-up (Level 1)
**Q: A Go service with `net/http` makes millions of syscalls per second via read/write on sockets. How does Go reduce this overhead internally?**

**A:**
Go's runtime uses several techniques:
1. **Non-blocking I/O + netpoller:** Go uses `epoll` to manage socket readiness. When a goroutine calls `conn.Read()`, Go's runtime checks if data is available (via epoll). If not, the goroutine is parked (Go scheduler) without a syscall. The netpoller (background thread) wakes goroutines when data arrives. Avoids blocking syscalls per-goroutine.

2. **Goroutine scheduling:** M:N threading — thousands of goroutines on tens of OS threads. Context switches between goroutines are userspace (no syscall). Fewer OS threads = fewer OS-level context switches.

3. **Large buffer reads:** Go's `bufio` package reads in 4KB+ chunks → reduces `read()` syscall frequency.

4. **Writev/sendmsg for responses:** Go's `http` server batches response headers + body into a single `writev()` when possible → 1 syscall instead of 2.

---

## Q69. How do you approach debugging a slow system?

### 📊 Grading Applied (Operational — Master Scenario)

### 🏭 Production Context
"The system is slow" = the most open-ended production problem. The discipline here is: resist guessing, follow the signal chain, eliminate categories systematically. A Staff SRE's mental model covers all resource types in a specific order.

### 🔧 The Systematic Debug Framework

```bash
# ── Step 1: Establish scope — what exactly is slow? ──
# Is it: all services? one service? specific operations? specific nodes?
# Slow since when? Correlated with: deploy? cron job? traffic increase?

# ── Step 2: Top-level resource check (60 seconds) ──
uptime              # Load average — are we overloaded?
vmstat 1 5          # CPU (us/sy/wa/st), memory (free/swap), I/O (bi/bo)
free -h             # Memory pressure
df -h               # Disk space
iostat -x 1 3       # Disk I/O utilization
ss -s               # Network connection state summary

# ── Step 3: Identify the bottleneck resource ──
# Decision tree:
# High %wa → I/O bottleneck → iostat, iotop
# High %us → CPU-bound → perf top, flame graph
# High %sy → excessive syscalls or kernel work → strace, perf
# High %st → steal time → cloud provider issue
# High load, low CPU → D-state (I/O or lock wait) → ps D state, wchan
# High memory usage → OOM or swap → /proc/meminfo, smem
# High network error rate → packet loss, MTU → ss -s, netstat -s, tcpdump

# ── Step 4: Per-service/process investigation ──
# Find the slow component:
top -b -n1 | head -30           # Which process?
pidstat 1 5                      # Per-process CPU/I/O trends

# ── Step 5: Application-level — trace the request ──
# For web services:
curl -w "\ntime_connect: %{time_connect}\ntime_ttfb: %{time_starttransfer}\ntime_total: %{time_total}\n" \
     -o /dev/null -s http://localhost/health
# time_connect high → TCP connection slow (network or listen backlog)
# time_ttfb high → application processing slow
# time_total - time_ttfb high → response transfer slow (network or large response)

# ── Step 6: Kernel-level tracing ──
# Trace all system calls for a short window:
perf trace -p <pid> -s 10    # syscall summary with timing

# Identify slow functions:
perf record -g -p <pid> -- sleep 10
perf report --sort=dso,sym

# ── Step 7: Off-CPU time (waiting, not running) ──
# Processes can be slow even with low CPU if they spend time waiting
bpftrace -e '
tracepoint:sched:sched_switch {
    if (args->prev_state == 1) {  // S state: sleeping
        @off_cpu[args->prev_comm] = count();
    }
}'
# What's causing processes to sleep?
```

### 🔍 Follow-up (Level 1)
**Q: Everything looks normal — CPU 20%, memory fine, I/O 10%, network clean. But P99 latency is 10x normal. What do you check next?**

**A:**
Normal resource metrics with high tail latency = the latency is hiding in non-resource dimensions:

1. **Lock contention:** threads waiting on mutexes → CPU looks idle (threads sleeping), but requests queue
   ```bash
   pidstat -w 1   # nvcswch/s (involuntary) high → contention
   perf record -e 'lock:lock_acquire' -g -p <pid>
   ```

2. **GC pauses (managed runtimes):** GC stop-the-world → all threads paused → requests that arrived during GC experience tail latency spike
   ```bash
   # JVM: enable GC logging
   -Xlog:gc*:file=/tmp/gc.log:time,uptime:filecount=5,filesize=50m
   grep -i "pause" /tmp/gc.log | tail -20
   ```

3. **Network retransmissions:** TCP retransmit on the return path → request appears slow but only occasionally
   ```bash
   netstat -s | grep -i retransmit
   ss -ti | grep retrans   # Per-connection retransmit count
   ```

4. **Noisy scheduler (scheduling jitter):** task preempted right before responding → 10ms scheduler jitter becomes P99 spike
   ```bash
   # Check scheduler latency:
   bpftrace -e 'tracepoint:sched:sched_wakeup { @[comm] = hist(nsecs - args->__timestamp); }'
   ```

5. **DNS lookup on hot path:** external service DNS not cached → 50ms lookup on every request
   ```bash
   strace -e trace=socket,connect,sendto,recvfrom -p <pid> 2>&1 | grep -v "EAGAIN"
   # Look for socket(AF_INET) on DNS port 53 in hot path
   ```

### 🔥 Follow-up (Level 2)
**Q: The slow system is a Kubernetes node. Other nodes are fast. How does your investigation change?**

**A:**
Node-specific slowness in K8s means: either the node's hardware/OS is impacted, or the pods on that node are causing each other interference.

```bash
# ── K8s node investigation ──

# Step 1: Node conditions
kubectl describe node <nodename>
# Look for: MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable
# Also: events section — OOM kills, kubelet errors

# Step 2: Pod resource usage on this node
kubectl top pods --all-namespaces | grep <nodename>
kubectl describe node <nodename> | grep -A50 "Allocated resources"
# CPU/Memory: Requests vs Limits vs Actual (top)

# Step 3: noisy neighbor identification
# Which pod is using the most CPU?
kubectl top pods --all-namespaces --sort-by=cpu | head -10
# Which pod caused OOM kills recently?
kubectl get events --all-namespaces | grep OOMKilling

# Step 4: Node-level kernel investigation
# SSH to node and run standard debug:
vmstat 1 5; iostat -x 1 3; perf top

# Step 5: kubelet health
systemctl status kubelet
journalctl -u kubelet --since "1 hour ago" | grep -E "error|warn|OOM|evict"
# kubelet evictions? → memory/disk pressure causing pod evictions → service disruption

# Step 6: Network CNI health
kubectl get pods -n kube-system | grep -E "flannel|calico|cilium"
# CNI pod on this node Crashing? → network issues affecting all pods on node
```

---

### 📊 Grading

| Dimension | Score |
|---|---|
| Production Realism | 9 |
| Technical Depth | 9 |
| Debugging Practicality | 10 |
| Failure Awareness | 9 |
| Clarity & Signal | 9 |

**Total: 46/50**

**🔍 Critical Feedback:**
- Missing `dmesg` and `journalctl -k` as first-pass checks — hardware errors (MCE, NIC errors) show up there and explain mysterious slowness
- Should mention USE method (Utilization, Saturation, Errors) as a formal framework attribution
- The K8s investigation is strong but doesn't cover `kubectl exec` into a debug pod on the slow node for application-level tracing

---

## Q70. How do you perform root cause analysis (RCA)?

### 🏭 Production Context
RCA is the post-incident discipline that separates reactive firefighting from systems improvement. Without structured RCA, the same incident repeats. At Staff+ level, the quality of your RCA document is a proxy for your engineering judgment.

### 🔧 The RCA Framework

**Phase 1: Timeline reconstruction**
```
Who noticed? What was the first signal?
  → Alert at 14:23 / User complaint at 14:20 / Monitoring gap?

What changed before the incident?
  → git log --since="1 hour before incident"
  → kubectl rollout history
  → terraform plan logs
  → config changes, cron jobs, external events (upstream provider issues)

What was the sequence of failure?
  → Correlated log timestamps across services
  → Metrics: which metric degraded first? (leading indicator vs lagging)
```

**Phase 2: The 5 Whys (technical)**
```
Symptom: Users see 502 errors
Why? → API servers returning errors
Why? → DB connections timing out
Why? → DB connection pool exhausted
Why? → Connection leak: exception path didn't close connections
Why? → Code change 3 hours before incident added early-return without try-finally

Root cause: Missing resource cleanup in exception path
```

**Phase 3: Contributing factors (not just root cause)**
```
Why wasn't this caught?
  → No test for exception paths
  → Connection leak monitoring had 1-hour lag (too slow to catch)
  → Code review didn't check resource lifecycle

Why did it escalate?
  → Pool size was at minimum (cost cutting)
  → No circuit breaker between API and DB
  → Alert threshold was 10 minutes — too slow for user impact
```

**Phase 4: Action items (SMART)**
```
Immediate (< 1 week):
  [ ] Fix the code (PR + tests) — Owner: @dev, Due: 2024-01-15
  [ ] Add connection leak metric (active vs max pool) — @infra
  [ ] Lower alert threshold to 2 minutes — @sre

Short-term (< 1 month):
  [ ] Add circuit breaker (Hystrix/resilience4j) — @arch
  [ ] Increase minimum pool size — @infra
  [ ] Add integration test for resource cleanup in exception paths — @dev

Long-term (< 1 quarter):
  [ ] Connection pool leak detection in staging (load test) — @sre
  [ ] Static analysis rule for resource cleanup patterns — @platform
```

### 🔍 Follow-up (Level 1)
**Q: The RCA identifies a code bug as root cause. A Staff engineer says "that's a proximate cause, not root cause." What's the distinction?**

**A:**
**Proximate cause:** the immediate trigger — the specific thing that broke ("the exception path didn't close the connection").

**Root cause:** the systemic failure that allowed the proximate cause to cause an incident:
- Why was the bug in production? → Insufficient code review
- Why wasn't it caught in staging? → No connection leak test
- Why did it page customers? → No circuit breaker, alert too slow, pool too small

A Staff engineer's RCA reaches the systemic level: process failures, missing safety nets, cultural issues. Fixing just the bug without addressing the system guarantees a future variant of the same incident.

**The "5 Whys" test:** If you can ask "Why?" to your stated root cause and get a meaningful answer → you haven't found the real root cause yet.

### 💡 Real-World Insight
A payments company had 4 incidents in 6 months, each RCA listing "code bug" as root cause. Each was fixed individually. A Staff engineer rewrote the 5th RCA to identify the actual root cause: no automated resource lifecycle testing, no staging load test before production deploy, and deployment approval process that prioritized velocity over safety. Result: the systemic fix (automated leak detection in CI + mandatory 30-minute staging soak) prevented the entire class of incidents. The individual bugs were symptoms; the process was the disease.

---

# 📋 Batch 5 Summary

| Q | Topic | Graded? |
|---|---|---|
| Q35 | High CPU Debug | Operational ✓ (45/50) |
| Q36 | Thread-level CPU | — |
| Q37 | High Memory Debug | Operational ✓ (43/50) |
| Q38 | Memory Leak Detection | — |
| Q39 | High Disk I/O Debug | Operational ✓ (44/50) |
| Q40 | IO Wait | — |
| Q62 | High System Load | — |
| Q63 | Load Average Interpretation | — |
| Q64 | CPU-bound vs IO-bound | — |
| Q65 | Context Switching | — |
| Q66 | Kernel vs User Space | — |
| Q67 | Why Kernel Space Protected | — |
| Q68 | Syscall Overhead | — |
| Q69 | Slow System Debug | Operational ✓ (46/50) |
| Q70 | RCA Methodology | Operational ✓ |

**Next: Batch 6 — Networking Fundamentals**
Topics: TCP/UDP, 3-way handshake, DNS, ARP, routing, NAT, firewalls, MTU, packet drops, curl google.com