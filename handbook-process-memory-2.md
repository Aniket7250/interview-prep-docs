# Process Management & Memory — SRE Engineering Handbook
> Module: Day 1 — Process Management & Memory  
> Status: Encoding + Reference + Retrieval + Interleaving Complete  
> Retrieval Score: 74/100  
> Next: Overlearning (pending answer)

---

## 1. Mental Models & Analogies

### The Restaurant Kitchen Analogy
| Kitchen | Linux |
|---------|-------|
| Head chef | Kernel scheduler |
| Kitchen | RAM + CPU |
| Each cook | A process |
| Cook cloning themselves | `fork()` |
| Clone switching to a different recipe | `exec()` |
| Cook finished but waiting for chef to acknowledge | Zombie process |
| Cook whose trainer left | Orphan process |
| Kitchen running out of stations | Process table full |
| Chef removing a cook hogging all ingredients | OOM Killer |

### CoW = LVM Snapshot — Same Idea, Different Layer
`fork()` marks all parent pages as shared + read-only (no actual RAM copy).  
Only when either process **writes** to a page does the kernel copy that specific page.  
Same Copy-on-Write mechanism as LVM snapshots — just at the memory layer instead of storage.

---

## 2. Process Lifecycle

### fork() → exec() Chain
```
Shell (bash, PID 1234)
  └─ calls fork()
       └─ Child clone (PID 1235) — identical copy, CoW pages set up
            └─ calls exec("/bin/ls")
                 └─ entire virtual address space replaced with ls binary
                      └─ ls runs → exit() → parent calls wait() → child reaped
```

**Key facts:**
- `fork()` is called by the **shell**, not systemd
- `exec()` keeps the same PID but replaces: code, stack, heap, data segments
- CoW pages set up during `fork()` are **discarded by exec() before any copy is triggered** — fork+exec is cheap
- Parent calls `wait()` to collect exit status and remove child from process table

### Process States (STAT column in `ps aux`)
| State | Meaning | Signal behaviour |
|-------|---------|-----------------|
| R | Running / runnable | Normal |
| S | Interruptible sleep | Can receive signals |
| D | Uninterruptible sleep — blocked on I/O | **SIGKILL won't work** |
| Z | Zombie — exited, parent hasn't called wait() | Already dead, can't be killed |
| T | Stopped | SIGSTOP received |

---

## 3. Zombie vs Orphan

| | Zombie | Orphan |
|---|---|---|
| Cause | Child exited, parent never called `wait()` | Parent exited before child |
| In process table? | YES — Z state | YES — running normally |
| Adopted by? | Nobody — stuck until parent reaps | init/systemd (PID 1) |
| Can you kill it? | No — already dead | Not needed |
| Fix | Kill parent → init reaps zombie | Nothing — init handles it |

### Fixing a Zombie Storm
```bash
# Find zombies
ps aux | awk '$8 == "Z"'

# Find parent of a zombie
ps -o ppid= -p <zombie_pid>

# Kill parent gracefully — init will reap the zombies
kill -SIGTERM <ppid>

# Force kill parent if needed
kill -9 <ppid>
```

**Interview trap:** "Why can't you kill a zombie?" — It's already dead. You kill the **parent** that isn't calling `wait()`.  
SIGCHLD is already sent by kernel automatically on child exit — problem is parent is ignoring it (no `wait()` handler).

---

## 4. Process Table Full

### Symptom
```
-bash: fork: retry: Resource temporarily unavailable
```

### Root Cause
- `ulimit -u` (per-user process limit) exhausted, **OR**
- `kernel.pid_max` system-wide limit hit (default: 32768 or 4194304)

### Immediate Unblock — In Order
```bash
# 1. Count total processes
ps aux | wc -l

# 2. Find zombie storm (most common culprit)
ps aux | awk '$8 == "Z"'

# 3. Find who's spawning most processes
ps aux | awk '{print $1}' | sort | uniq -c | sort -rn | head

# 4. Raise per-user limit (current session only)
ulimit -u 65536

# 5. Check and raise kernel-wide limit
cat /proc/sys/kernel/pid_max
sysctl -w kernel.pid_max=131072
```

**Senior SRE move:** Stop the **source** first (e.g. `systemctl stop crond` if cron is spawning) before cleaning up — otherwise you're bailing a sinking boat without plugging the hole.

---

## 5. Signals

### Key Signals — Must Know Cold
| Signal | Number | Default Action | Real-world Use |
|--------|--------|---------------|----------------|
| SIGTERM | 15 | Terminate | Graceful shutdown request |
| SIGKILL | 9 | Terminate (force) | Unblockable force kill |
| SIGHUP | 1 | Terminate | **Config reload for daemons** |
| SIGCHLD | 17 | Ignore | Child exited — notify parent |
| SIGSTOP | 19 | Stop | Pause process — unblockable |
| SIGCONT | 18 | Continue | Resume stopped process |
| SIGSEGV | 11 | Core dump | Segmentation fault |
| SIGINT | 2 | Terminate | Ctrl+C |

### The Golden Rules
- **SIGKILL and SIGSTOP** cannot be caught, ignored, or blocked — ever
- All other signals can be: handled / ignored / blocked by the process
- A process in **D state** cannot receive any signals — not even SIGKILL

### Config Reload Without Restart (SIGHUP)
```bash
# nginx reload — zero connection drop
kill -HUP $(cat /var/run/nginx.pid)
nginx -s reload

# Most daemons: sshd, rsyslog, haproxy support SIGHUP reload
```

### Before Reaching for SIGKILL
```bash
# Step 1: Check process state
ps aux | grep <pid>      # D state = SIGKILL won't work

# Step 2: What syscall is it stuck on?
strace -p <pid>

# Step 3: What files/sockets does it hold?
lsof -p <pid>

# Step 4: Which kernel function is it sleeping in?
cat /proc/<pid>/wchan
```

**Why avoid kill -9:** No signal handler runs → no buffer flush, no DB connection close, no lock release → potential data corruption, stale locks blocking restart.

---

## 6. Memory Management

### Virtual Memory Model
```
Process view (virtual):          Physical reality:
┌─────────────────┐              ┌─────────────────┐
│   stack         │              │  Physical RAM    │
│   heap          │◀── MMU ─────▶│  Page Frame 1   │
│   data          │  translates  │  Page Frame 2   │
│   text/code     │              │  Swap (disk)    │
└─────────────────┘              └─────────────────┘
```

- Every process has a **private** virtual address space
- MMU translates virtual → physical transparently
- Two processes can share the same virtual address — they map to different physical pages

### Overcommit — Why OOM Happens
```
App calls malloc(8GB) on a 16GB server
  → kernel says "yes" (heuristic overcommit, vm.overcommit_memory=0)
  → NO physical RAM assigned yet
  → physical RAM assigned only when app touches pages (demand paging)
  → if physical RAM exhausted at touch time → OOM Killer fires
```

**This is why:** app crashes on restart (not on malloc) — startup touches pages, RAM exhausted, OOM fires.

### OOM Killer — Scoring & Control
```bash
# View process OOM score (0–1000, higher = more likely killed)
cat /proc/<pid>/oom_score

# Adjust score (-1000 to +1000)
echo -1000 > /proc/$(pidof postgres)/oom_score_adj   # fully exempt
echo 1000  > /proc/<pid>/oom_score_adj                # kill me first
echo -500  > /proc/<pid>/oom_score_adj                # protect but not exempt

# Check OOM kill events
dmesg | grep -i "oom\|killed process"
```

**Interview trap:** "Set all processes to -1000?" — OOM Killer has nothing to kill → system panic or hard hang. Protect **selectively**.

### Swap Tuning by Workload
| Workload | swappiness | Rationale |
|----------|-----------|-----------|
| General production | 20–30 | Reduce swap use, keep safety net |
| Latency-sensitive | 1 | Only swap to avoid OOM, never routine |
| Trading / HFT | 0 + mlockall() | Pin pages in RAM, no paging ever |
| Desktop | 60 (default) | Default kernel value |

```bash
# Temporary
sysctl -w vm.swappiness=1

# Permanent
echo "vm.swappiness=1" >> /etc/sysctl.conf
sysctl -p

# Pin process memory in RAM (application-level, used by HFT/DBs)
# Application calls mlockall(MCL_CURRENT | MCL_FUTURE) in code
```

**si + so both non-zero = thrashing** — kernel swapping pages in for one process while swapping out for another. System spending more time on swap management than real work. Immediate action needed.

---

## 7. fork() vs Threads — Decision Framework

| Dimension | Threads | Fork (Child Processes) |
|-----------|---------|----------------------|
| Memory | Shared — efficient, race condition risk | Isolated — safe, CoW overhead |
| Crash impact | One thread crash = whole process dies | Child crash = parent survives |
| Communication | Direct via shared memory | IPC needed (pipes, sockets) |
| Creation cost | ~10µs | ~100µs |
| Use case | CPU-bound parallel work, shared data | Untrusted code, worker isolation |

**The trap:** 1,000 threads = context switch thrash. Real answer is **thread pool capped at CPU count** or an **event loop model** (nginx/Node).

---

## 8. Key /proc Files
```bash
cat /proc/meminfo              # full memory breakdown
cat /proc/<pid>/status         # process memory — VmRSS = actual RAM used
cat /proc/<pid>/maps           # virtual memory map of the process
cat /proc/<pid>/wchan          # kernel function process is sleeping in
cat /proc/sys/kernel/pid_max   # max PIDs system-wide
```

---

## 9. vmstat — Column Reference
```bash
vmstat 1 5    # 5 snapshots, 1 second apart
```

| Column | Meaning | Warning Sign |
|--------|---------|-------------|
| r | Processes waiting to run | Consistently > CPU count = CPU saturated |
| b | Processes in D state (I/O blocked) | Any non-zero warrants investigation |
| si | Swap in (disk → RAM) | Non-zero = active swapping |
| so | Swap out (RAM → disk) | Non-zero = memory pressure |
| wa | CPU % waiting on I/O | >20% = disk bottleneck |

**si + so simultaneously non-zero = thrashing**

---

## 10. strace — X-Ray for Processes
```bash
strace -p <pid>                          # attach to running process
strace -p <pid> -e trace=open,read,write # filter to specific syscalls
strace -T -p <pid>                       # show time spent per syscall
strace -c <command>                      # count syscall frequency summary
strace -e trace=network -p <pid>         # network syscalls only

# Output format: syscall(args) = return_value
open("/etc/hosts", O_RDONLY) = 3
read(3, "127.0.0.1 localhost\n", 4096)  = 234
```

**Warning:** 2–10x performance overhead. Use on stuck or low-traffic processes only. Never on high-traffic production without understanding the impact.

---

## 11. Debugging Toolkit — When to Use What

| Tool | Use When |
|------|----------|
| `ps aux` | Get process list, states, CPU/mem |
| `top` | Live monitoring — preferred in prod (lightweight, preserves error state) |
| `htop` | Interactive debugging in non-prod |
| `strace -p` | Process stuck — see exact syscall blocked on |
| `lsof -p` | What files/sockets a process holds |
| `vmstat 1` | Memory pressure, swap activity, I/O wait |
| `dmesg` | Kernel events — OOM kills, hardware errors |
| `/proc/<pid>/` | Deep per-process internals |

**Production tip (Aniket's own insight):** Prefer `top` over `htop` in production — lightweight, doesn't stress the system, preserves the error state for accurate debugging.

---

## 12. Interview One-Liners

| Question | Answer |
|----------|--------|
| Why can't you kill a zombie? | It's already dead — kill the parent, init reaps it |
| Why avoid kill -9? | No signal handler → no buffer flush, no lock release, data corruption risk |
| What is D state? | Uninterruptible sleep — waiting on kernel I/O — SIGKILL won't work |
| What does SIGHUP do to nginx? | Reloads config without dropping connections |
| Why does malloc succeed but app crashes later? | Overcommit — physical RAM assigned at page touch, not at malloc |
| How does fork() not double RAM? | Copy-on-Write — pages shared until written |
| fork+exec is cheap — why? | CoW pages set up by fork are discarded by exec before any copy triggers |
| si + so both non-zero? | Thrashing — fix immediately, add RAM or kill memory hogs |