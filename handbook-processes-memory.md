# Process Management & Memory — SRE Handbook

**Audience**: Senior SRE candidates
**Purpose**: Quick reference for interview questions and production debugging
**Format**: One-page mental models + commands

---

## 1. Process Lifecycle

### fork() → exec() Chain

```
bash (PID 1234)
  └─ calls fork()
       └─ bash clone (PID 1235)  ← identical copy, CoW memory
            └─ calls exec("/bin/ls")
                 └─ process image replaced entirely with ls binary
                      └─ ls runs, exits, returns status to bash via wait()
```

**Key points**:
- fork(): Creates child process. PID ≠ systemd unless PID 1.
- exec(): Replaces entire virtual address space. Same PID. No return on success.
- wait(): Parent collects exit status, reaps zombie.

### Copy-on-Write (CoW)

```
fork() called
  → OS marks all parent pages as shared + read-only
  → No actual RAM copied
  → Copy happens only when parent OR child writes to a page
```

**Why it matters**: fork()+exec() is cheap because exec() discards CoW pages before any write triggers a copy.

**Analogy**: LVM snapshots — same principle, different layer.

---

## 2. Process States (STAT column in ps)

| State | Meaning | Signal behavior |
|-------|---------|-----------------|
| R | Running / runnable | Normal |
| S | Interruptible sleep | Can receive signals |
| D | Uninterruptible sleep (I/O) | **SIGKILL ignored** |
| Z | Zombie | Already dead |
| T | Stopped | SIGCONT resumes |

**Critical**: D state cannot be killed. Fix the I/O first.

---

## 3. Zombie vs Orphan

| | Zombie | Orphan |
|---|---|---|
| **Cause** | Child exited, parent never called wait() | Parent exited before child |
| **In process table?** | YES (Z state) | YES (running normally) |
| **Adopted by?** | Nobody — stuck until parent reaps | init/systemd (PID 1) |
| **Fix** | Kill parent → init reaps zombie | Nothing needed — init handles |

### Fix a zombie storm

```bash
ps aux | awk '$8 == "Z"'                   # Find zombies
ps -o ppid= -p <zombie_pid>               # Find parent
kill -SIGTERM <ppid>                      # Try graceful kill
kill -9 <ppid>                            # Force → init reaps
```

**Interview trap**: "Why can't you kill a zombie?" → Already dead. Kill parent instead.

---

## 4. Process Table Full

### Symptom
```bash
-bash: fork: retry: Resource temporarily unavailable
```

### Cause
- Per-user limit hit: `ulimit -u`
- System-wide limit hit: `/proc/sys/kernel/pid_max`

### Immediate unblock
```bash
ps aux | wc -l                             # Count total processes
ps aux | awk '$8 == "Z"'                   # Find zombie storm
ps aux | awk '{print $1}' | sort | uniq -c | sort -rn | head  # Who's spawning?
ulimit -u 65536                            # Raise limit (current session)
cat /proc/sys/kernel/pid_max               # Check kernel limit
sysctl -w kernel.pid_max=131072           # Raise kernel limit
```

**Interview answer**: "I'd count processes, look for zombie storms, identify the spawning user, raise limits temporarily, then investigate what caused the exhaustion."

---

## 5. Signals

### Key Signals

| Signal | Number | Default | Use |
|--------|--------|---------|-----|
| SIGTERM | 15 | Terminate | Graceful shutdown — catchable |
| SIGKILL | 9 | Terminate | Force kill — **uncatchable, unblockable** |
| SIGHUP | 1 | Terminate | Config reload for daemons |
| SIGCHLD | 17 | Ignore | Child exited — notify parent |
| SIGSTOP | 19 | Stop | Pause — **uncatchable** |
| SIGCONT | 18 | Continue | Resume stopped process |
| SIGSEGV | 11 | Core dump | Segfault |
| SIGINT | 2 | Terminate | Ctrl+C |

### Rules
- SIGKILL and SIGSTOP **cannot** be caught, ignored, or blocked — ever.
- All other signals can be: **handled** / **ignored** / **blocked**.

### Before reaching for SIGKILL

1. Check STAT: `ps aux | grep <pid>` — if D state, SIGKILL won't work.
2. `strace -p <pid>` — see which syscall it's stuck on.
3. `lsof -p <pid>` — what files/sockets it holds.
4. `cat /proc/<pid>/wchan` — which kernel function it's sleeping in.

### Config reload without restart (SIGHUP)
```bash
kill -HUP $(cat /var/run/nginx.pid)
# or
nginx -s reload
```

---

## 6. Memory Management

### Virtual Memory Model

```
Process view:                    Reality:
┌─────────────────┐              ┌─────────────────┐
│   0xFFFFFFFF    │              │   Physical RAM   │
│   (stack)       │◀─── MMU ────▶│   Page Frame 1  │
│   (heap)        │   translates │   Page Frame 2  │
│   (data)        │              │   Swap Space    │
│   (text/code)   │              │   (disk)        │
│   0x00000000    │              └─────────────────┘
└─────────────────┘
```

Every process thinks it owns a full, private address space. MMU translates virtual → physical transparently.

### Overcommit

Linux by default: **vm.overcommit_memory = 0** (heuristic overcommit)

```
App calls malloc(8GB) on 16GB server
  → kernel says "sure" (virtual allocation succeeds)
  → no physical RAM assigned yet
  → only when app writes to those pages does kernel assign RAM (demand paging)
  → if physical RAM runs out → OOM Killer fires
```

**Why malloc succeeds but app crashes later**: Physical RAM assigned at page-touch, not malloc.

### OOM Killer

**Trigger**: Physical RAM + swap fully exhausted.

**How it works**:
- Scores each process 0–1000 (higher = more likely killed)
- Score factors: RAM used (`VmRSS`), child processes, time running, nice value

**Interface**:
```bash
cat /proc/<pid>/oom_score                     # View score (0-1000)

echo -500 > /proc/<pid>/oom_score_adj        # Protect (range: -1000 to +1000)
# -1000 = completely exempt
# +1000 = kill me first

echo -1000 > /proc/$(pidof postgres)/oom_score_adj  # Protect DB
```

**Check OOM kills**:
```bash
dmesg | grep -i "oom\|killed process"
journalctl -k | grep -i oom
```

**Interview trap**: "What if all processes are -1000?" → System panics/hangs. OOM Killer has nothing to kill.

---

## 7. Key /proc Files

```bash
cat /proc/meminfo                          # Full memory breakdown
cat /proc/<pid>/status                     # Process memory (VmRSS = actual RAM)
cat /proc/<pid>/maps                       # Virtual memory map
cat /proc/<pid>/wchan                      # Kernel function process sleeping in
cat /proc/sys/kernel/pid_max               # Max PIDs system-wide
cat /proc/<pid>/oom_score                  # OOM score
cat /proc/<pid>/oom_score_adj              # OOM adjustment
```

---

## 8. vmstat — Column Reference

```bash
vmstat 1 5                                 # 5 snapshots, 1 sec apart
```

```
procs --------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa
 2  0      0 1024000  12345 678901   0    0     0     0  234  567  5  2 93  0
```

**Columns that matter**:
- `r` = processes waiting to run (> CPU count = CPU saturated)
- `b` = processes in D state (I/O blocked)
- `si` = swap in (reading from swap → RAM) — non-zero is bad
- `so` = swap out (writing RAM → swap) — non-zero is bad
- `wa` = CPU % waiting on I/O (high = disk bottleneck)

**Thrashing**: `si` and `so` both non-zero simultaneously → system spends more time swapping than working.

---

## 9. strace — X-Ray for Processes

**Intercepts every syscall a process makes.**

```bash
strace -p <pid>                            # Attach to running process
strace -e trace=open,read,write ls /tmp   # Filter specific syscalls
strace -T -p <pid>                        # Show time per syscall
strace -c <command>                       # Count syscall frequency
strace -e trace=network -p <pid>          # Network syscalls only
```

**Output format**:
```
open("/etc/hosts", O_RDONLY) = 3
read(3, "127.0.0.1 localhost\n...", 4096) = 234
write(1, "127.0.0.1 localhost\n", 20) = 20
close(3) = 0
```

Each line: `syscall(args) = return_value`

**Production use cases**:
- Process stuck → see exact syscall it's blocked on
- App can't find file → trace every open() and error code
- Performance debugging → which syscall takes longest
- "Why is this talking to network?" → filter network

**⚠️ Warning**: 2-10x slowdown. Use on stuck/low-traffic processes only.

---

## 10. Debugging Toolkit — When to Use What

| Tool | Use When |
|------|----------|
| `ps aux` | Get process list, states, CPU/mem usage |
| `top` / `htop` | Live monitoring — **top** in prod (lightweight) |
| `strace -p` | Process stuck — see syscall it's blocked on |
| `lsof -p` | What files/sockets a process holds |
| `vmstat 1` | Memory pressure, swap activity, I/O wait |
| `dmesg` | Kernel events — OOM kills, hardware errors |
| `/proc/<pid>/` | Deep per-process internals |
| `kill -l` | List all signals |
| `ulimit -a` | Show current limits |

---

## 11. Interview One-Liners

### Processes
- **"Why can't you kill a zombie?"** — It's already dead. Kill the parent; init reaps it.
- **"Why does fork+exec not double RAM?"** — Copy-on-Write. exec discards CoW mappings before copies happen.
- **"What happens when you run a command?"** — Shell forks → child execs → parent waits.
- **"Process table full?"** — ulimit -u or kernel.pid_max hit. Count, find zombies, raise limits.

### Signals
- **"kill -9 vs -15?"** — -9 uncatchable force kill (no cleanup). -15 graceful (catchable).
- **"What does SIGHUP do?"** — Daemons reload config without restart (nginx -s reload).
- **"Why avoid kill -9 on DB?"** — No signal handler runs → no buffer flush, no lock release → corruption.
- **"D state?"** — Uninterruptible sleep (I/O wait). SIGKILL won't work.
- **"Before SIGKILL?"** — Check STAT; if D, fix I/O. If S, use strace/lsof to see what it's waiting on.

### Memory
- **"malloc succeeds but app crashes later?"** — Overcommit. Physical RAM assigned at page-touch, not malloc.
- **"What is OOM Killer?"** — Scores processes 0-1000, kills highest when RAM+swap exhausted.
- **"How to protect a process?"** — Write -1000 to `/proc/<pid>/oom_score_adj`.
- **"si+so both non-zero?"** — Thrashing. System swapping in/out simultaneously → collapse.
- **"fork+exec memory cost?"** — CoW pages discarded by exec → essentially just page table overhead.

---

## 12. Common Interview Scenarios

### Scenario 1: Zombie Storm
**Symptom**: Hundreds of zombies, parent process still running.

**Fix**:
```bash
ps aux | awk '$8 == "Z"' | awk '{print $3}' | sort | uniq -c | sort -rn  # Find parent with most zombies
kill -TERM <ppid>   # Kill parent
# init (PID 1) will reap all zombies
```

### Scenario 2: Process Table Full
**Symptom**: `-bash: fork: retry: Resource temporarily unavailable`

**Diagnosis**:
```bash
ps aux | wc -l                              # How many?
ps aux | awk '$8 == "Z"'                   # Zombie storm?
ps aux | awk '{print $1}' | sort | uniq -c | sort -rn  # Which user spawning?
ulimit -u                                   # Current limit
cat /proc/sys/kernel/pid_max               # System max
```

**Fix**: Kill runaway processes OR raise limits (`ulimit -u 65536`).

### Scenario 3: D State Process Wont Die
**Symptom**: Process stuck, `kill -9` does nothing.

**Diagnosis**:
```bash
ps aux | grep <pid>  # Check STAT column — D state?
strace -p <pid>      # What syscall?
cat /proc/<pid>/wchan # Which kernel function?
lsof -p <pid>        # What files?
dmesg | tail         # I/O errors?
```

**Root causes**: Hung NFS mount, dead disk, kernel driver bug.

**Fix**: Fix underlying I/O. Unmount dead NFS, restart disk, reboot if needed.

### Scenario 4: OOM Killer Fired
**Symptom**: Process disappeared, no explanation.

**Diagnosis**:
```bash
dmesg | grep -i "killed process"
cat /proc/<pid>/oom_score
cat /proc/<pid>/oom_score_adj
```

**Prevention**: Set `oom_score_adj` on critical processes. Monitor memory usage. Add swap/RAM.

---

## 13. Production Best Practices

### Configuring limits
```bash
# In /etc/security/limits.conf or systemd service file
<username> soft nproc 65536
<username> hard nproc 65536
```

### Protecting critical processes
```bash
# Prevent OOM killer from touching your database
echo -1000 > /proc/$(pidof postgres)/oom_score_adj
```

### Monitoring thresholds
- RAM usage > 80% → alert
- Zombie count > 10 → alert
- Process count > 80% ulimit → alert
- OOM events in dmesg → critical alert

---

**Revision tip**: This handbook is a checklist. In mock interviews, mentally walk through each section. Can you explain every command, every state, every signal?
