# Batch 3 — Memory Management
### Staff+ Level | Self-Interrogation + Spot Grading on Operational Questions
---

## Q11. What is virtual memory vs physical memory?

### 🏭 Production Context
Every process sees a flat, private address space — but this is an abstraction. Physical RAM is shared, oversubscribed, and managed dynamically. In production, understanding this distinction is critical for:
- Diagnosing OOM kills that happen despite "plenty of free memory" showing in `top`
- Understanding why VIRT (virtual) memory of a process can be 10GB on a machine with 2GB RAM
- Container memory limits: they cap physical usage (RSS), not virtual address space

### ⚙️ Inner Workings

**Virtual Memory:**
- Each process gets its own virtual address space (0 to 2^48 on x86-64 for userspace)
- Virtual addresses don't map to physical RAM until accessed
- Kernel maintains a **page table** per process: virtual page → physical frame mapping
- Pages are 4KB by default (or 2MB/1GB with hugepages)
- Virtual address space is divided into VMAs (Virtual Memory Areas): stack, heap, code, mmap regions

**Physical Memory:**
- Actual RAM chips: finite, shared across all processes and kernel
- Managed by kernel's **buddy allocator** (large blocks) and **slab allocator** (small objects)
- A physical page frame can be:
  - Mapped to one process (anonymous private)
  - Shared across multiple processes (shared memory, CoW before write, file-backed pages)
  - Swapped out to disk (not in RAM at all)

**The translation:**
```
Virtual Address → [MMU: TLB lookup] → Physical Address
                         ↓
                   (TLB miss: walk page table)
                         ↓
                   PTE (Page Table Entry) → Physical Frame Number
```

MMU (Memory Management Unit) in hardware does the translation. TLB (Translation Lookaside Buffer) caches recent translations. TLB miss = page table walk = 4–5 memory reads (for 4-level page tables on x86-64).

### 🔍 Follow-up (Level 1)
**Q: A process has 100GB of virtual memory but only 2GB RSS. Is this a problem?**

**A:**
Not inherently. Virtual memory is cheap — it's just address space reservation, not physical RAM.
Common legitimate reasons for large VIRT:
- Memory-mapped files (`mmap`): a process can map a 100GB file; only accessed pages are loaded into RAM
- Java heap reservations: JVM reserves full `-Xmx` at startup but only commits as needed
- Shared libraries mapped into address space but partially referenced
- Guard pages, anonymous huge reservations

**When it BECOMES a problem:**
- OOM killer uses RSS for eviction decisions, not VIRT
- Virtual address space exhaustion (rare on 64-bit: 128TB userspace)
- Overcommit ratio: `vm.overcommit_ratio` limits total virtual memory vs physical. If overcommit disabled (`vm.overcommit_memory=2`), large VIRT reservations fail even with RAM available

```bash
cat /proc/sys/vm/overcommit_memory    # 0=heuristic, 1=always, 2=never
cat /proc/sys/vm/overcommit_ratio     # % of RAM+swap allowed for overcommit
```

### 🔥 Follow-up (Level 2)
**Q: On a 64-bit system, what prevents a process from mapping all 128TB of userspace and crashing the kernel?**

**A:**
Multiple limits:
1. `RLIMIT_AS` — per-process virtual address space limit
   ```bash
   ulimit -v   # Virtual address space limit
   cat /proc/<pid>/limits | grep "virtual memory"
   ```
2. `vm.max_map_count` — limits number of VMAs (memory-mapped regions) per process. Default: 65536. Each `mmap()` call creates a VMA. Exceeded → `mmap()` returns `ENOMEM`
   ```bash
   cat /proc/sys/vm/max_map_count
   # Java with many threads: each thread stack = 1 VMA; 65536 threads = limit hit
   ```
3. **Overcommit policy** (`vm.overcommit_memory=2`): total committed virtual memory across all processes cannot exceed `(RAM + swap) × overcommit_ratio`. New allocations fail with `ENOMEM` when this ceiling is hit.
4. Physical memory still needed for page tables themselves — each level of page table = physical RAM. Mapping 128TB requires significant page table memory.

### 💡 Real-World Insight
Elasticsearch with `vm.max_map_count` at default (65536) fails to start on large nodes: Elasticsearch uses memory-mapped files for Lucene indexes — each segment file = multiple mmap regions. With 1000 shards × multiple segments = tens of thousands of VMAs. Standard fix: `sysctl -w vm.max_map_count=262144` in Elasticsearch docs. Without it: `java.io.IOException: failed to increase mmapfs count`.

---

## Q12. What is RSS vs VIRT memory?

### 🏭 Production Context
This distinction determines what actually counts against your system's RAM. OOM killer, container memory limits, and `MemAvailable` in `/proc/meminfo` all care about physical memory (RSS), not virtual. Misreading VIRT as "actual usage" leads to wrong capacity decisions.

### ⚙️ Inner Workings

**VIRT (Virtual Memory Size):**
- Total virtual address space claimed by the process
- Includes: code, data, heap, stack, mmap'd files, shared libraries, reserved-but-not-committed memory
- Meaningless for RAM pressure estimation
- In `top`: VIRT column

**RSS (Resident Set Size):**
- Physical pages currently in RAM, belonging to this process
- Excludes: swapped-out pages, pages not yet faulted in (demand paging)
- INCLUDES shared pages (shared libraries) — double-counted across processes
- In `top`: RES column
- In `/proc/<pid>/status`: `VmRSS`

**PSS (Proportional Set Size):**
- RSS, but shared pages divided by number of processes sharing them
- More accurate for total system memory accounting
- `cat /proc/<pid>/smaps_rollup | grep Pss`

**USS (Unique Set Size):**
- Only pages unique to this process (not shared with anyone)
- True private memory footprint
- Useful for memory leak detection: USS growing = genuine leak

```bash
# Per-process memory breakdown
cat /proc/<pid>/status | grep -E "VmRSS|VmSize|VmSwap"

# Smaps for detailed breakdown
cat /proc/<pid>/smaps_rollup

# Using smem tool
smem -p -k | grep <process>   # Shows RSS, PSS, USS
```

### 🔍 Follow-up (Level 1)
**Q: System has 32GB RAM. Sum of all processes' RSS = 40GB. But the system isn't OOM. How?**

**A:**
Shared memory double-counting. RSS counts shared pages (e.g., libc, libpthread, OpenSSL) once per process. If 100 processes each link libc (10MB), RSS sum = 100 × 10MB = 1GB attributed to libc, but physical RAM used = just 10MB (one copy).

Reality: total physical RAM = sum of all PSS values (proportional sharing), not RSS.
```bash
# Accurate system memory accounting
smem -t -k   # Total: RSS vs PSS vs USS
# PSS total will be ≤ physical RAM; RSS total can exceed it due to sharing
```

Additionally:
- Kernel uses memory too (not reflected in any process RSS): `cat /proc/meminfo | grep Slab`
- Page cache (file caching): counted in `/proc/meminfo` as `Cached` — not in any process's RSS
- MemAvailable includes page cache that can be reclaimed

### 🔥 Follow-up (Level 2)
**Q: Container memory limit in Kubernetes is 512MB. Process RSS is 480MB. Process calls `malloc(100MB)`. Does it get OOM killed immediately?**

**A:**
Not necessarily. `malloc(100MB)` goes through glibc which calls `mmap(MAP_ANONYMOUS)` for large allocations (> 128KB default `MMAP_THRESHOLD`). This:
1. Creates a new VMA in the process's virtual address space
2. Returns a virtual address — no physical pages allocated yet (demand paging)
3. RSS does NOT increase immediately after malloc
4. RSS increases as the process actually touches (writes to) those pages

So: `malloc(100MB)` succeeds. VIRT increases by 100MB. RSS stays at 480MB.
If process writes to all 100MB: RSS would become 580MB → exceeds 512MB container limit → cgroup OOM kill.

But if process only writes to 50MB: RSS = 530MB → OOM kill.
If process never writes: RSS stays at 480MB → safe.

This is why memory-limiting containers with `malloc`-heavy languages is tricky: the limit check is on physical pages committed, not address space reserved. Java's `-Xmx` reserves virtual space but commits physical gradually.

```bash
# Container cgroup limit
cat /sys/fs/cgroup/memory/<container>/memory.limit_in_bytes
cat /sys/fs/cgroup/memory/<container>/memory.usage_in_bytes   # Current RSS
cat /sys/fs/cgroup/memory/<container>/memory.stat | grep pgfault  # Page faults
```

### 💡 Real-World Insight
A Go service was configured with Kubernetes memory limit 512MB. Go's runtime pre-allocates memory aggressively for GC. On startup: RSS 120MB. After 24 hours of traffic: RSS 490MB. Triggered OOM kill. Root cause: Go's runtime was holding freed memory in its internal heap (for GC efficiency) rather than returning to OS. RSS grew but from the app's perspective memory was "free." Fix: `GOGC=50` (more aggressive GC), `runtime.ReadMemStats` monitoring, and set `GOMEMLIMIT` (Go 1.19+) to soft-limit Go's heap below Kubernetes hard limit.

---

## Q13. What is demand paging?

### 🏭 Production Context
Demand paging is why Linux can run a 10GB process on a 4GB machine (with swap). It's also why a large process can start in milliseconds despite its binary being hundreds of MB. The downside: cold start latency spikes as pages are faulted in from disk.

### ⚙️ Inner Workings

When a process is created (exec):
- Kernel maps the binary's ELF segments into the address space (VMAs)
- No physical pages are allocated — just VMA metadata
- Page table entries are empty (not-present bit set)

When process accesses a virtual address:
1. CPU checks TLB → miss
2. CPU walks page table → PTE is not-present
3. CPU raises **page fault** exception (interrupt)
4. Kernel's `handle_mm_fault()` is invoked
5. Kernel checks: is this address in a valid VMA?
   - No → SIGSEGV (segfault)
   - Yes → **demand page it in:**
     a. Allocate physical page frame
     b. If file-backed (code, mmap'd file): read page from disk
     c. If anonymous (heap, stack): zero-fill the page
     d. Update PTE: virtual → physical mapping
     e. Return from page fault → instruction re-executed → succeeds

This means: process "has" 1GB of heap allocated but only the pages it actually touches are in RAM.

### 🔍 Follow-up (Level 1)
**Q: A service starts and immediately handles traffic. First few requests are slow. Subsequent requests are fast. How does demand paging explain this?**

**A:**
Classic cold start — demand paging working as designed but causing latency spikes:

1. Process starts: binary mapped but no pages loaded. RSS = just kernel/stack overhead.
2. First request arrives: code paths execute → page faults as code pages are loaded from disk
3. Library code referenced: libc, TLS libraries → more page faults
4. Heap allocated, written to: anonymous page faults
5. Each page fault: 1–10ms latency (disk read) for major faults, ~1µs for minor faults

After first few requests: all hot code paths are in RAM → no page faults → full speed.

Production mitigations:
- **Pre-warm / readiness probe delay:** don't send traffic until pod has handled N synthetic requests
- **mlock() / mlockall():** pin critical pages in RAM immediately (latency-sensitive services)
  ```c
  mlockall(MCL_CURRENT | MCL_FUTURE);  // Lock all current and future pages
  ```
- **Hugepages:** 2MB pages reduce TLB pressure for large in-memory datasets
- **Prefetching:** `madvise(addr, len, MADV_SEQUENTIAL)` hints kernel to read ahead

```bash
# Count page faults during startup
perf stat -e major-faults,minor-faults ./your-service
# major-faults >> 0 means disk reads during startup
```

### 💡 Real-World Insight
A Kubernetes pod with a 400MB JVM binary had a 10-second "warm-up" period where P99 latency was 3 seconds. Root cause: demand paging + NFS-backed container image layers. Each code page fault triggered a read from NFS. Fix: use local image cache (containerd's snapshot store on local SSD) and add a readiness probe with `initialDelaySeconds=15` to absorb warm-up. Also: `madvise(MADV_WILLNEED)` on JVM startup for critical library regions.

---

## Q14. What is a page fault?

### 🏭 Production Context
Page faults are not errors — they're expected kernel events. But at high rates, they indicate either:
- Cold start (expected, one-time)
- Memory pressure → pages being evicted and re-faulted (thrashing)
- Memory leak (continuously growing working set → continuous new page faults)

A service with 10,000 page faults/second while serving steady traffic is a red flag.

### ⚙️ Inner Workings

Page fault occurs when CPU accesses a virtual address and the PTE is:
1. **Not present** — page not in RAM (not yet loaded, or swapped out)
2. **Present but wrong permissions** — write to read-only page (CoW trigger)

Kernel's fault handler (`do_page_fault()` → `handle_mm_fault()`):
```
1. Find VMA for the faulting address
2. If no VMA → SIGSEGV (invalid access)
3. Determine fault type:
   a. Anonymous page (heap/stack): alloc new page, zero-fill
   b. File-backed: read page from file (disk)
   c. Swap: read page from swap device
   d. CoW: copy shared page → private copy
4. Update PTE
5. Return → faulting instruction retried
```

### 🔍 Follow-up (Level 1)
**Q: What is the difference between minor and major page faults?**

**A:**
**Minor page fault (soft fault):**
- Page IS in RAM somewhere (in page cache, or shared but not yet in process's page table)
- No disk I/O needed — just update the process's PTE to point to existing physical frame
- Cost: ~1µs (kernel work, no I/O)
- Examples: first access to a shared library (already loaded by another process), CoW copy

**Major page fault (hard fault):**
- Page is NOT in RAM — must be read from disk
- Disk I/O required: block device read → HDD: 5–10ms, SSD: 0.1–0.5ms
- Process blocks during the I/O
- Cost: 100µs–10ms
- Examples: first access to a freshly-started binary's code, swapped-out page being accessed

```bash
# Monitor page faults
ps -o pid,min_flt,maj_flt,cmd -p <pid>
# min_flt = minor faults, maj_flt = major faults (cumulative)

# Real-time rate
perf stat -e minor-faults,major-faults -p <pid> sleep 10

# Per-address fault tracing
bpftrace -e 'tracepoint:exceptions:page_fault_user { @[ustack] = count(); }'
```

### 🔥 Follow-up (Level 2)
**Q: A service shows 50,000 major page faults/second in steady state — not during startup. What does this indicate?**

**A:**
50k major faults/second in steady state = **thrashing**. The process's working set exceeds available RAM. Pages are being evicted to swap, then immediately needed again → re-read from swap → evicted again. Cycle repeats continuously.

This is catastrophic for performance: 50k × 1ms (SSD swap) = 50 seconds of I/O time per second of wall clock — physically impossible at scale; what actually happens is massive latency with throughput collapse.

Diagnosis:
```bash
vmstat 1 | awk '{print $7, $8}'   # si/so columns: swap in/out pages per second
# si > 0 in steady state = thrashing

free -h   # Check if swap is actively being used
cat /proc/meminfo | grep -E "Active|Inactive|SwapUsed"

# Which process is the thrash victim?
cat /proc/<pid>/status | grep VmSwap   # How much of this process is swapped?
```

Solution path:
1. Reduce memory footprint (tune JVM heap, reduce cache sizes)
2. Increase available RAM
3. Identify memory leak (`smem`, USS growth tracking)
4. Set `vm.swappiness=0` for latency-sensitive services (prefer OOM kill to thrashing)
5. Use `cgroup memory.low` to protect critical services from reclaim

### 💡 Real-World Insight
A Redis node with 16GB RAM had `vm.swappiness=60` (default). Under memory pressure from noisy neighbor containers, Redis pages were being swapped. Redis reported 50ms GET latency (normal: <1ms) — major page faults on every evicted key access. Key lesson: Redis (and any latency-sensitive in-memory service) MUST have `vm.swappiness=0` or better, `mlock` its entire dataset. Swapping = death for latency. Also: container memory limits protect against this scenario — Redis should have been in a guaranteed QoS pod (requests = limits).

---

## Q15. Minor vs Major Page Faults (see Q14 above — covered in depth)

*Fully addressed in Q14's Level 1 follow-up.*

---

## Q16. What is swap memory and when is it used?

### 🏭 Production Context
In production latency-sensitive systems: swap = enemy. In batch/analytics systems: swap = safety net. Understanding when the kernel reaches for swap — and how to prevent/tune it — is a core SRE skill.

### ⚙️ Inner Workings

Swap is disk space used to store physical RAM pages when RAM is under pressure. The kernel's page reclaim mechanism (`kswapd` daemon + direct reclaim) decides which pages to evict:

```
Memory pressure triggers kswapd:
  kswapd wakes when free pages < pages_low (watermark)
  
  Eviction candidates (by LRU lists):
    1. Clean file-backed pages → dropped (can re-read from file)
    2. Dirty file-backed pages → write-back to disk, then drop
    3. Anonymous pages (heap/stack) → write to swap, then free frame
```

`vm.swappiness` (0–200, default 60):
- Controls kernel's tendency to swap anonymous pages vs reclaim file-backed pages
- 0 = avoid swap, strongly prefer reclaiming file cache
- 100 = treat anonymous and file-backed pages equally
- 200 = aggressively swap anonymous pages
- NOT "0 = no swap ever" — if memory is truly exhausted, kernel will swap even at swappiness=0

```bash
# Current swap usage
free -h
cat /proc/meminfo | grep -i swap

# Per-process swap usage
cat /proc/<pid>/status | grep VmSwap

# Swap I/O rate
vmstat 1 | awk 'NR>2 {print $7 " in " $8 " out"}'   # si/so

# Swappiness
cat /proc/sys/vm/swappiness
```

### 🔍 Follow-up (Level 1)
**Q: You set `vm.swappiness=0` on a production host. Under extreme memory pressure, does the kernel STILL swap?**

**A:**
Yes. `swappiness=0` does NOT disable swapping — it's a hint, not a hard constraint.

What `swappiness=0` means:
- Kernel strongly prefers evicting file-backed pages (page cache) over anonymous (swap) pages
- Only resorts to swap when there's essentially nothing else to reclaim

Under true memory exhaustion (all file cache evicted, still pressure):
- Kernel WILL swap anonymous pages even at swappiness=0
- The alternative is OOM killer, which is kernel's last resort after all reclaim options

To truly prevent swap:
```bash
# Option 1: No swap device configured
swapoff -a   # Disable all swap
# → OOM killer triggers instead of swapping

# Option 2: Lock process memory (can't be swapped)
mlockall(MCL_CURRENT | MCL_FUTURE);   # In application code

# Option 3: cgroup memory limit (containers)
# When limit hit → OOM kill within container, not swap
```

For latency-sensitive production: prefer `swapoff -a` + careful memory sizing over swap-as-safety-net.

### 🔥 Follow-up (Level 2)
**Q: Swap is disabled (`swapoff -a`). System runs out of memory. What exactly happens?**

**A:**
```
kswapd runs → reclaims all reclaimable file cache
Still pressure → direct reclaim in application context (slow)
OOM condition: free pages < min watermark
  → OOM killer invoked (kernel/mm/oom_kill.c)
  → Selects victim via oom_score (0-1000)
    oom_score = (RSS / total_RAM × 1000) + oom_score_adj
  → Sends SIGKILL to victim
  → Frees victim's memory
  → Normal operation resumes (if enough freed)
```

Influencing OOM killer:
```bash
# Protect a critical process
echo -1000 > /proc/<pid>/oom_score_adj   # Never kill (-1000 = immune)

# Make a process the preferred victim
echo 1000 > /proc/<pid>/oom_score_adj    # Kill me first

# Check current oom_score
cat /proc/<pid>/oom_score       # Calculated score (higher = more likely victim)
cat /proc/<pid>/oom_score_adj   # Admin adjustment

# See OOM kill events
dmesg | grep -i "oom\|killed process"
journalctl -k | grep -i oom
```

In Kubernetes: containers are OOM killed by cgroup memory limit, NOT by system OOM killer. Different path, same result (SIGKILL to the container's PID 1).

### 💡 Real-World Insight
A MySQL server had swap enabled with `vm.swappiness=10`. During a slow query spike, buffer pool pages were partially swapped out — MySQL held 24GB of buffer pool, 2GB was swapped. InnoDB tried to read swapped pages: 10ms per access vs normal 0.1ms. Query latency went from 5ms to 500ms. No OOM, no kill — just severe degradation that looked like "slow queries" not "memory problem." Fix: `swapoff -a` + resize buffer pool to fit in available RAM. Key lesson: swap-induced latency looks identical to I/O bottleneck in slow query logs.

---

## Q17. What is swap thrashing?

### 🏭 Production Context
Thrashing is the endgame of memory overcommitment. The system spends more time doing I/O to move pages between RAM and swap than doing actual computation. CPU utilization stays high (I/O wait), throughput collapses, latency explodes. It's often misdiagnosed as a disk I/O problem.

### ⚙️ Inner Workings

Thrashing occurs when:
- Working set (all pages needed for normal operation) > available RAM
- Pages evicted to swap are immediately needed again
- Cycle: page needed → fault → read from swap → another page evicted → needed → fault → repeat

**Identification signals:**
```
vmstat 1:
  - si (swap in) > 0 continuously = pages being read from swap
  - so (swap out) > 0 continuously = pages being written to swap
  - b column (blocked) high = processes blocked waiting for page I/O
  - wa (io wait) high = CPU waiting for swap I/O
  
top/htop:
  - High %wa CPU
  - Low actual %us (not doing useful work)
  - Swap usage high and dynamic (changing)
```

### 🔍 Follow-up (Level 1)
**Q: How do you distinguish thrashing from a legitimate I/O-heavy workload?**

**A:**
Key differentiators:

**Legitimate I/O workload:**
- `si`/`so` may be high during data ingestion but not simultaneously
- Swap usage grows then stabilizes
- Application throughput is healthy despite I/O wait
- Page fault rate is bounded (not runaway)

**Thrashing:**
- `si` AND `so` both non-zero simultaneously — system is reading and writing swap concurrently
- Application throughput collapses while I/O wait is high
- `cat /proc/vmstat | grep pgmajfault` — major fault rate growing continuously
- `sar -B 1` shows `majflt/s` (major faults per second) climbing
- Process RSS is NOT growing (not a leak) — just oscillating as pages cycle through swap

```bash
# Thrashing fingerprint:
watch -n1 'vmstat 1 1 | tail -1 | awk "{print \"si:\"\$7\" so:\"\$8\" wa:\"\$16}"'
# si > 0 AND so > 0 AND wa > 20% → thrashing

# Page reclaim pressure:
cat /proc/vmstat | grep -E "pgsteal|pgscan|pgmajfault"
# pgscan > pgsteal by large margin → kernel struggling to reclaim
```

### 💡 Real-World Insight
An analytics cluster ran Spark jobs scheduled to overlap. Three jobs each needing 24GB RAM ran on a 32GB node simultaneously. Swap was enabled (120GB NVMe). Within 10 minutes: all three jobs' working sets couldn't fit in RAM → thrashing. `vmstat` showed `si=5000` (5000 pages/s swapped in), `so=4000`, `wa=85%`. Jobs took 8 hours instead of 30 minutes. Swap on NVMe still 100× slower than RAM for random access. Fix: enforce job isolation via Spark's `spark.executor.memory` limits + YARN/K8s resource quotas to prevent co-scheduling memory-intensive jobs.

---

## Q18. What is the difference between heap and stack?

### 🏭 Production Context
Stack overflows and heap fragmentation are real production failure modes. Understanding heap vs stack:
- Helps diagnose `java.lang.StackOverflowError`, `SIGSEGV` at stack boundary
- Explains why recursive algorithms fail in production at certain input depths
- Guides memory allocation strategy in high-performance services

### ⚙️ Inner Workings

**Stack:**
- Per-thread, fixed-size memory region (default 8MB on Linux: `ulimit -s`)
- Grows downward (toward lower addresses)
- LIFO: function call pushes frame; return pops it
- Managed automatically by compiler (RSP register tracks top)
- Contains: local variables, return addresses, saved registers, function arguments
- Physical pages allocated on demand (page fault as stack grows)
- Stack overflow: grows past guard page → SIGSEGV

**Heap:**
- Single (conceptually) dynamic memory region per process
- Grows upward (via `brk()`) or via `mmap(MAP_ANONYMOUS)`
- Managed by allocator (glibc malloc, jemalloc, tcmalloc)
- `malloc()` → finds free block or requests more from kernel
- `free()` → returns block to allocator's free list (may or may not return to kernel)
- Fragmentation: allocating and freeing blocks of varying sizes leaves gaps

```
Virtual Address Space (top → bottom):
┌─────────────────┐ High addresses (kernel at 0xFFFF...)
│     Stack       │ ← grows downward, per thread
│       ↓         │
│  [guard page]   │
│                 │
│  (unmapped)     │
│                 │
│       ↑         │
│     Heap        │ ← grows upward (brk)
│  (mmap regions) │ ← large mallocs via mmap
│  shared libs    │
│  code (.text)   │
│  data (.data)   │ Low addresses
└─────────────────┘
```

### 🔍 Follow-up (Level 1)
**Q: A service throws `StackOverflowError` in Java at exactly 10,000 recursive calls. Why exactly that number?**

**A:**
Each Java stack frame consumes space for:
- Local variables
- Operand stack for intermediate computations
- Reference to enclosing frame

Default JVM stack size = 512KB (client) to 1MB (server) per thread.
Typical frame size = 50–200 bytes depending on method complexity.
1MB / 100 bytes per frame ≈ 10,000 frames → StackOverflow.

```bash
# Adjust Java thread stack size:
java -Xss4m MyApp   # 4MB stack per thread (allows ~40,000 frames)

# Check current stack size:
java -XX:+PrintFlagsFinal -version | grep ThreadStackSize
```

But tuning stack size is the wrong fix. The right fix: convert recursion to iteration + explicit stack data structure, or use tail-call optimization (Scala/Kotlin with `@tailrec`). In production, unbounded recursion on user-controlled input = potential DoS vector (attacker can trigger StackOverflow intentionally).

### 🔥 Follow-up (Level 2)
**Q: Heap fragmentation causes `malloc()` to fail even when `free` reports plenty of space. How does this happen and how do you fix it in production?**

**A:**
Heap fragmentation: process has allocated and freed memory in patterns that leave the heap full of small gaps that can't be coalesced into large contiguous blocks.

Example:
```
Allocated: [16KB][8KB][16KB][8KB][16KB] ...
Free [8KB] chunks in between: total free = 100MB
Request: malloc(1MB) → FAILS → no contiguous 1MB block
```

Detection:
```bash
# For C/C++ with glibc:
malloc_stats()   # In gdb or via malloc_info() syscall
# Or: /proc/<pid>/maps — look for many small anonymous mappings

# For Java: heap dump analysis
jmap -histo:live <pid>   # Object count by type
jmap -dump:format=b,file=heap.hprof <pid>
# Analyze with Eclipse MAT or jhat
```

Fixes:
1. **Use a better allocator:** jemalloc and tcmalloc have better fragmentation resistance than glibc malloc
   ```bash
   LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 ./app
   ```
2. **Release memory to OS:** glibc malloc doesn't always return freed memory. Call `malloc_trim(0)` or set `MALLOC_TRIM_THRESHOLD_=0`
3. **For Java:** tune GC to compact heap (G1GC with `G1HeapRegionSize`, ZGC which is always compacting)
4. **Arena-based allocation:** allocate with known lifetimes together, free all at once (avoids fragmentation entirely)

### 💡 Real-World Insight
A C++ service processed variable-size messages (1B to 100MB). After 24 hours, `malloc()` started failing with `ENOMEM` — RSS was only 2GB on a 16GB machine. Heap was fragmented: thousands of 1-byte allocations interspersed with holes from freed 100MB chunks. Fix: switched to jemalloc + added size-class specific memory pools for small allocations. Monthly restart was eliminated. Key insight: glibc malloc is general-purpose; production services with predictable allocation patterns should use specialized allocators.

---

## Q19. When does memory go to heap vs stack?

### ⚙️ Inner Workings

**Stack allocation (automatic storage duration):**
- Local variables in a function: `int x = 5;`
- Fixed-size arrays: `char buf[256];`
- Function parameters
- Compiler determines size at compile time → adjusts RSP (stack pointer)
- Cost: 1–2 instructions (just move RSP)
- Lifetime: auto-freed when function returns

**Heap allocation (dynamic storage duration):**
- `malloc()`, `calloc()`, `new` in C++, `object` creation in Java/Python
- Required when: size unknown at compile time, lifetime outlasts function scope, large allocations (>8MB → goes to mmap not brk anyway)
- Cost: lock acquisition, free-list search, potential kernel call
- Lifetime: until explicitly freed (C/C++) or GC collected (managed languages)

**Language specifics:**
- C/C++: explicit heap (`malloc`/`new`), explicit stack (locals)
- Java: ALL objects on heap; primitive locals on stack; JVM escape analysis can optimize small objects to stack
- Go: escape analysis decides: if object "escapes" its function scope → heap; otherwise → stack. `go build -gcflags="-m"` shows escape decisions
- Python: all objects on heap (CPython); small integers and strings interned

```bash
# Go: see what escapes to heap
go build -gcflags="-m -m" ./... 2>&1 | grep "escapes to heap"

# C: check stack usage of a function (compiler)
gcc -fstack-usage -c file.c   # Generates .su file with stack usage per function
```

### 🔍 Follow-up (Level 1)
**Q: Go's escape analysis puts a variable on heap even though it's declared locally. When does this happen?**

**A:**
A variable "escapes" to heap when its address or value is reachable outside the function:
1. **Returned pointer:** `func f() *int { x := 5; return &x }` → x escapes; it must outlive f()
2. **Closure capture:** `f := func() { use(x) }` → x captured by closure → may escape
3. **Interface assignment:** converting concrete type to interface{} → often escapes (runtime needs heap storage)
4. **Size too large for stack:** arrays > ~64KB → escape to heap
5. **Variable passed to goroutine:** goroutine may outlive caller → captured vars escape

Escape has performance implications: heap allocation costs more than stack. In hot paths:
```go
// Avoid: allocates on heap
func badHot() io.Writer {
    var b bytes.Buffer
    return &b   // escapes
}

// Better: caller provides buffer
func goodHot(w io.Writer) {
    // w is caller's buffer, no escape needed
}
```

### 💡 Real-World Insight
A Go HTTP handler allocated a 4KB response buffer on every request: `buf := make([]byte, 4096)`. At 50k req/s: 50k × 4KB = 200MB/s of heap allocation. GC pressure was enormous — 30% of CPU on GC. Fix: `sync.Pool` to reuse buffers:
```go
var bufPool = sync.Pool{New: func() interface{} { return make([]byte, 4096) }}
buf := bufPool.Get().([]byte)
defer bufPool.Put(buf)
```
GC pressure dropped 80%. CPU reclaimed.

---

## Q20. What are anonymous vs file-backed pages?

### 🏭 Production Context
This distinction determines how the kernel reclaims memory under pressure and what ends up in swap. File-backed pages can simply be dropped (re-readable from disk). Anonymous pages must be swapped. Misidentifying which type dominates a process's RSS affects your memory tuning strategy.

### ⚙️ Inner Workings

**File-backed pages:**
- Mapped from files: executables, shared libraries, `mmap()`'d files
- Kernel's page cache manages them
- If modified (dirty) → must write back before evicting
- If clean → simply drop (re-read from original file on demand)
- Shared across processes reading the same file (1 physical page, N process PTEs)
- NOT counted in swap (they go back to the file, not swap)

**Anonymous pages:**
- Not backed by any file: heap, stack, `mmap(MAP_ANONYMOUS)`
- Created via `brk()` or `mmap(MAP_ANONYMOUS | MAP_PRIVATE)`
- Under memory pressure: must write to swap to reclaim (no file to re-read from)
- Private to the process (after CoW break)
- Zero-filled on first access

```bash
# See breakdown per process
cat /proc/<pid>/smaps | grep -E "^(Anonymous|File-backed|Swap|Shared_Clean|Private):" | head -30
# Or summarized:
cat /proc/<pid>/smaps_rollup

# System-wide:
cat /proc/meminfo
# Active(anon)  = anonymous pages in active LRU (frequently accessed)
# Inactive(anon) = anonymous pages in inactive LRU (swap candidates)
# Active(file)  = file-backed in active LRU
# Inactive(file) = file-backed in inactive LRU (eviction candidates)
```

### 🔍 Follow-up (Level 1)
**Q: A process uses `mmap()` to map a file. Are those pages anonymous or file-backed? What happens when the process writes to them?**

**A:**
`mmap(fd, ..., MAP_SHARED)`: file-backed, shared. Writes go directly to the page cache and are eventually written to the file (dirty page writeback). Visible to other processes mapping the same file.

`mmap(fd, ..., MAP_PRIVATE)`: file-backed initially, but on write → CoW → becomes **anonymous**. The write creates a private anonymous copy; the original file is unchanged. This is how executable code works: code pages are file-backed (from ELF), but once written (e.g., JIT), they CoW to anonymous.

Practical implication:
```bash
cat /proc/<pid>/smaps | grep -A5 "r-xp"   # Code section: file-backed
# After JIT: some regions become r-wp (writable) → anonymous
```

### 🔥 Follow-up (Level 2)
**Q: You need to reduce swap usage for a process. Its `/proc/<pid>/smaps_rollup` shows 90% of RSS is anonymous. What do you do?**

**A:**
90% anonymous RSS = heap + stack + private mmap regions. These are the swap candidates.

Strategy:
1. **Reduce heap footprint:** tune GC heap size (Java `-Xmx`), reduce in-memory caches, use more efficient data structures
2. **Identify what's in the anonymous pages:**
   ```bash
   cat /proc/<pid>/smaps | awk '/Size/ && /anon/ {print}'
   # Or use pmap with -X flag:
   pmap -x <pid> | sort -k3 -rn | head -20   # Sort by RSS
   ```
3. **mlock critical regions:** prevent them from being swapped
4. **Return memory to OS:** call `malloc_trim()` (C), `runtime.GC(); debug.FreeOSMemory()` (Go)
5. **Increase RAM or move to larger instance**
6. **Last resort:** `vm.swappiness=0` to delay swapping; `swapoff -a` to force OOM instead of degraded-via-swap

### 💡 Real-World Insight
A Node.js service showed 8GB anonymous RSS after a week of uptime. V8's heap kept growing (legitimate objects + fragmentation). Anonymous pages dominated. Under memory pressure, the kernel swapped out V8 heap pages → GC pause spiked from 10ms to 3 seconds (GC tried to scan swapped pages → major faults). Fix: set `--max-old-space-size=4096` to cap V8 heap + add weekly rolling restart (pod restart in K8s) to clear accumulated heap fragmentation. Long-term: switch to streaming architecture to reduce in-memory state.

---

# 📋 Batch 3 Summary

| Q | Topic | Graded? |
|---|---|---|
| Q11 | Virtual vs Physical Memory | — |
| Q12 | RSS vs VIRT | — |
| Q13 | Demand Paging | — |
| Q14 | Page Fault | — |
| Q15 | Minor vs Major Page Fault | (covered in Q14) |
| Q16 | Swap Memory | — |
| Q17 | Swap Thrashing | Operational ✓ |
| Q18 | Heap vs Stack | — |
| Q19 | Memory → Heap vs Stack | — |
| Q20 | Anonymous vs File-backed Pages | — |

**Next: Batch 4 — File System, IPC & VFS**
Topics: file descriptors, inodes, hard/soft links, IPC mechanisms, VFS, ls -l internals