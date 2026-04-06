# Batch 2 — Process Management
### Staff+ Level | Self-Interrogation + Spot Grading on Operational Questions
---

## Q1. What happens when a process is created in Linux?

### 🏭 Production Context
Every container start, every request handler spawn, every shell command — all go through this path. At scale, a pod churn event (rolling deploy, node drain) can trigger thousands of process creations per second. Each creation is not free: `fork()` + `exec()` costs ~1–2ms under load, causing latency spikes in services that spawn child processes per request.

### ⚙️ Inner Workings

```
Shell/runtime calls fork() → clone() syscall
  → kernel allocates new task_struct
  → copies parent's memory map (CoW — no physical copy yet)
  → copies file descriptor table
  → copies signal handlers
  → assigns new PID (from pid namespace)
  → child inherits parent's CPU affinity, cgroups, namespaces
  → child returns 0 from fork(); parent returns child PID
  → child calls exec() → replaces memory image
```

Kernel path:
- `fork()` → `clone()` syscall → `copy_process()` in `kernel/fork.c`
- `copy_process()` allocates `task_struct`, `mm_struct`, `files_struct`
- `_do_fork()` → `copy_mm()` (CoW), `copy_files()`, `copy_sighand()`
- PID assigned via `alloc_pid()` from the PID namespace
- New task added to run queue by scheduler

### 🔧 Tooling

```bash
strace -e trace=clone,execve bash -c "ls"
# Shows exact clone() flags and execve() call

# Watch process creation rate
perf stat -e task:task_newtask sleep 5

# Process creation latency
bpftrace -e 'tracepoint:sched:sched_process_fork { @[comm] = count(); }'
```

### 🔍 Follow-up (Level 1)
**Q: What's the difference between `fork()` and `clone()` at the kernel level?**

**A:**
`fork()` is a libc wrapper that calls `clone()` with specific flags:
```c
clone(SIGCHLD, 0)   // fork: child is separate process
```
`clone()` is the real syscall. Flags control what is shared vs copied:
- `CLONE_VM` → share memory (threads)
- `CLONE_FILES` → share FD table (threads)
- `CLONE_NEWPID` → new PID namespace (containers)
- `CLONE_NEWNET` → new network namespace (containers)

`pthread_create()` calls `clone(CLONE_VM | CLONE_FILES | CLONE_SIGHAND ...)` — full sharing = thread.
`fork()` calls `clone()` with none of those — full isolation = process.

### 🔥 Follow-up (Level 2 — Edge Case)
**Q: A service forks thousands of children per second. What breaks in the kernel?**

**A:**
- **PID namespace exhaustion** — default `pid_max` = 32768; fork-bombing consumes all PIDs
  ```bash
  cat /proc/sys/kernel/pid_max   # Check limit
  sysctl -w kernel.pid_max=4194304  # Increase (max ~4M)
  ```
- **task_struct slab cache pressure** — each process = `task_struct` alloc; under extreme fork rate, slab allocator thrashes
- **Scheduler overhead** — O(1) adds tasks to run queue; massive fork rate increases scheduler lock contention
- **Memory pressure** — each fork duplicates page table entries (even with CoW); with large parent process, page table copy alone consumes GBs of kernel memory

### 💡 Real-World Insight
PHP-FPM in process-per-request mode: each request spawned a full PHP process (fork+exec), hitting 200 req/s = 200 forks/s. Under load, `fork()` latency climbed from 2ms to 50ms because parent process had 4GB of mapped memory — page table duplication dominated. Fix: switch to PHP-FPM worker pool (pre-forked) or use OPcache to share bytecode pages.

---

## Q2. What is the difference between fork() and exec()?

### 🏭 Production Context
The fork+exec pattern is the foundation of every shell command, every container entrypoint, every CGI process. Understanding this split matters when diagnosing: why does a child process inherit parent FDs (pre-exec)? Why do env vars not propagate after exec? Why does memory usage spike during fork but drop after exec?

### ⚙️ Inner Workings

**fork():**
- Creates an exact copy of the calling process
- Child gets a new PID but same memory image (CoW), same FDs, same signal handlers
- Returns twice: child gets 0, parent gets child PID
- Kernel: `copy_process()` → duplicates `mm_struct` (CoW), `files_struct`, `sighand_struct`

**exec():**
- Replaces the current process image with a new program
- Does NOT create a new PID — same process, same PID, same FDs (unless FD_CLOEXEC set)
- Kernel: `do_execve()` → loads ELF binary, sets up new stack/heap, maps new segments
- Old memory (`mm_struct`) is released; new one built from ELF headers
- Signal handlers reset to default (handlers are code pointers to old image, now gone)
- CoW pages from fork that weren't modified are discarded — exec makes the fork's CoW irrelevant

**Combined (shell model):**
```
shell fork() → child clone of shell
child exec("/bin/ls") → child's memory replaced with ls binary
parent wait() → blocks until child exits
child exits → parent resumes
```

### 🔍 Follow-up (Level 1)
**Q: After fork(), the child inherits all parent FDs. What problem does this cause in production and how do you fix it?**

**A:**
Classic FD leak: parent has open sockets, files, pipes. Child inherits ALL of them. If child then calls exec() (launching a subprocess), the subprocess inherits them too — including listening sockets, DB connections, log file handles.

Result: 
- Parent closes a socket; the FD is still open in child → port stays "in use"
- DB connection held by child → pool thinks it's in use; parent pool depletes
- Log file held by child → log rotation fails (file can't be deleted, held open)

Fix: set `FD_CLOEXEC` on FDs that shouldn't be inherited:
```c
fcntl(fd, F_SETFD, FD_CLOEXEC);  // Close on exec
// Or at open time:
open(path, O_RDONLY | O_CLOEXEC);
socket(AF_INET, SOCK_STREAM | SOCK_CLOEXEC, 0);
```
In production: every socket/file opened by a server should have `O_CLOEXEC` unless deliberately inherited.

### 🔥 Follow-up (Level 2)
**Q: exec() keeps the same PID. What happens to the process's memory, thread group, and signal state?**

**A:**
- **Memory:** all mappings (`mmap` regions, stack, heap, code) are unmapped; new ELF loaded; stack rebuilt; heap starts fresh at brk
- **Threads:** if multi-threaded process calls exec(), all other threads are killed immediately — exec() in a multi-threaded process is single-threaded after
- **Signals:** pending signals are discarded; signal handlers reset to SIG_DFL; signal mask preserved
- **FDs:** FDs without `FD_CLOEXEC` are preserved; those with it are closed
- **Capabilities:** if binary has setuid bit, capabilities are re-evaluated on exec

### 💡 Real-World Insight
Python's `subprocess.Popen` without `close_fds=True` (Python 2 default) inherited all parent FDs into child processes. In a long-running API server with hundreds of connections, each subprocess spawned held open sockets → ulimit exhaustion after hours of traffic. Python 3 defaults `close_fds=True`. Lesson: always audit FD inheritance in subprocess-heavy services.

---

## Q3. What is copy-on-write (CoW)?

### 🏭 Production Context
CoW is why `fork()` is fast even for a 10GB process — no physical memory is copied at fork time. In production, CoW matters for:
- Redis fork() during BGSAVE — momentarily doubles virtual memory; if writes are heavy, CoW triggers massive physical page copying → memory pressure → OOM
- Container image layers — overlayFS uses CoW; writing to a container layer triggers CoW copy of underlying layer pages

### ⚙️ Inner Workings

At `fork()`, kernel marks all parent pages as read-only and shared with child. Both parent and child point to same physical pages.

When either writes to a page:
1. CPU generates page fault (write to read-only page)
2. Kernel's page fault handler: `do_wp_page()` in `mm/memory.c`
3. Allocates new physical page
4. Copies content of shared page to new page
5. Updates PTE (Page Table Entry) of writing process to point to new page
6. Re-marks new page as writable
7. Re-marks shared page as writable for the other process (if only 2 references remain → 1, it's not shared anymore)

Memory anatomy after fork():
```
Parent PTEs: [page_A readonly] → physical frame 0x1000
Child PTEs:  [page_A readonly] → physical frame 0x1000  (shared)

Parent writes to page_A:
  → page fault → copy → 
Parent PTEs: [page_A writable] → physical frame 0x2000  (new)
Child PTEs:  [page_A writable] → physical frame 0x1000  (original, now writable)
```

### 🔍 Follow-up (Level 1)
**Q: Redis does BGSAVE (fork()) on a 20GB dataset. Memory usage spikes to 30GB. Why?**

**A:**
During BGSAVE, Redis forks. At fork time: 20GB pages are shared (CoW) — physical memory is still 20GB.

But Redis is actively serving writes. Every write to a key modifies a page. That page triggers CoW — a new physical copy is made for either parent or child.

If 50% of the dataset is written during the BGSAVE window:
- 10GB of pages get CoW-copied → 10GB additional physical RAM consumed
- Total physical = 20GB original + 10GB CoW = 30GB

At extreme write rates (write-heavy workloads), the CoW rate can exceed available RAM → kernel starts swapping → BGSAVE takes longer → more writes during BGSAVE → more CoW. Runaway.

Fix:
```bash
# Check CoW rate during BGSAVE:
cat /proc/<redis-pid>/status | grep VmRSS   # Physical memory used
vmstat 1 | awk '{print $7}'                 # Swap usage

# Use jemalloc's background thread to amortize allocation
# CONFIG SET lazyfree-lazy-expire yes   # Avoid fork-time cleanup
```
Better: use Redis replication instead of fork-based persistence, or schedule BGSAVE during low-write periods.

### 🔥 Follow-up (Level 2)
**Q: exec() follows fork(). What happens to all the CoW pages that were never written?**

**A:**
exec() completely discards the child's address space — it calls `mmput()` to release all VMAs. The CoW pages that were never written (still shared with parent) simply have their reference count decremented. No copy was ever made, no physical page was duplicated. The fork() CoW overhead for exec() is nearly zero in terms of physical memory — the page table entries are set up and torn down, but physical pages are never touched.

This is why `fork()` + `exec()` is efficient: fork copies page table (fast), exec discards it (fast). Only the pages written between fork and exec are ever physically duplicated — and in a normal shell command, that's essentially none.

### 💡 Real-World Insight
A Go service used `os/exec` to call shell scripts — internally: fork() of the Go binary (which had 2GB heap). Each fork duplicated 2GB of page table entries (not physical memory, but PTEs themselves take RAM). With 50 concurrent subprocess calls, page table overhead caused 200MB of kernel memory pressure, triggering the OOM killer. Fix: use a dedicated small process as a subprocess launcher (not the main service binary), or use a long-lived subprocess pool instead of fork-per-request.

---

## Q4. What is a zombie process?

### 🏭 Production Context
Zombie processes don't consume CPU or memory (the process is dead) — but they DO consume a PID and a slot in the process table. In production, zombie accumulation causes:
- PID namespace exhaustion → `fork()` fails → service can't spawn workers
- Process table entries fill up → `ps` shows thousands of `<defunct>` entries
- Typically signals a parent process that isn't calling `wait()`

### ⚙️ Inner Workings

When a process exits:
1. Kernel releases most resources: memory, FDs, threads
2. BUT: kernel keeps a minimal `task_struct` entry — exit status code, PID, resource usage stats
3. This minimal entry waits for the parent to call `wait()` / `waitpid()` to collect it
4. Until `wait()` is called → process is in `Z` (zombie) state
5. After `wait()` → entry fully removed from process table

The zombie exists because POSIX guarantees: parent must be able to retrieve child's exit code. If kernel deleted everything immediately, the exit code would be lost.

```bash
ps aux | grep Z     # Z in STAT column = zombie
# Example:
# USER  PID  STAT  COMMAND
# www   1234  Z     [python] <defunct>
```

### 🔍 Follow-up (Level 1)
**Q: The parent is a long-running daemon that creates children but never calls `wait()`. How do you clean up zombies without killing the parent?**

**A:**
Option 1: Send SIGCHLD to parent — this is the signal the kernel sends to parent when child exits; a well-written daemon calls `wait()` in the SIGCHLD handler. Sending it manually may trigger the handler:
```bash
kill -SIGCHLD <parent-pid>
```
Caveat: only works if parent has a SIGCHLD handler that calls `wait()`.

Option 2: Kill the parent — when parent dies, zombies are re-parented to PID 1 (init/systemd). Init always calls `wait()` → zombies immediately reaped.

Option 3: If parent is yours — fix the code. Use `waitpid(-1, WNOHANG)` in SIGCHLD handler to reap all children non-blockingly:
```c
void sigchld_handler(int sig) {
    while (waitpid(-1, NULL, WNOHANG) > 0);
}
```

Option 4: Use `double fork` pattern — fork twice; grandchild does work; child exits immediately; grandchild is reparented to init. Parent only needs to wait for short-lived middle child.

### 🔥 Follow-up (Level 2)
**Q: In a Kubernetes pod, PID 1 is your application (not init/systemd). What happens to zombie processes?**

**A:**
PID 1 in a container has a special responsibility: it must reap all orphaned processes. If PID 1 is your app and it doesn't call `wait()`, zombies from child processes (health check scripts, sidecar helper scripts) accumulate permanently — nothing reaps them.

This is a very common Kubernetes footgun. Docker/containerd do NOT inject an init process by default.

Fix options:
1. **Use `tini` as PID 1:**
   ```dockerfile
   ENTRYPOINT ["/usr/bin/tini", "--", "your-app"]
   ```
   `tini` is a minimal init that reaps orphaned zombie processes.

2. **Kubernetes `shareProcessNamespace`** — pods can share PID namespace; with it enabled and an init container acting as PID 1, zombies are reaped properly.

3. **Docker/Kubernetes `--init` flag:**
   ```yaml
   # Pod spec
   spec:
     shareProcessNamespace: true
   ```
   Or in Docker: `docker run --init`

4. **Your app explicitly reaps children:** register SIGCHLD handler + `waitpid()`.

### 💡 Real-World Insight
A Node.js service in Kubernetes used shell scripts for health checks (`exec` probe). Each script ran as a child of PID 1 (Node.js). Node.js doesn't reap children. After 3 days of probes every 10 seconds = 25,920 zombies. PID namespace hit limit (32768). `fork()` failed → no new health check processes → Kubernetes marked pod unhealthy → pod killed and restarted → problem repeated. Fix: added `tini` as entrypoint. Lesson: always use an init process as PID 1 in containers.

---

## Q5. What is an orphan process?

### 🏭 Production Context
Orphan processes are the opposite problem from zombies — they're fully alive processes whose parent has died. The kernel reparents them to PID 1. In production, orphan processes are dangerous because:
- They continue consuming CPU/memory with no supervision
- No parent to deliver signals to (e.g., SIGTERM during shutdown)
- In containers, orphans reparented to app PID 1 which may not reap them → become zombies

### ⚙️ Inner Workings

When a parent exits before its children:
1. Kernel scans for children of dying process
2. Re-parents them to the subreaper (if set via `prctl(PR_SET_CHILD_SUBREAPER)`) or PID 1
3. Sends SIGHUP to the orphaned process group (if the parent was a session leader)
4. Orphan continues running normally — it just has a new parent (PID 1)

Difference from zombie:
- Zombie: dead child, parent alive but not calling wait()
- Orphan: live child, parent dead

```bash
# Find orphaned processes (parent PID = 1)
ps -eo pid,ppid,stat,cmd | awk '$2 == 1 && $1 != 1'
```

### 🔍 Follow-up (Level 1)
**Q: Why does SIGHUP get sent to the process group when the parent (session leader) exits?**

**A:**
In Unix session model:
- A **session** is a group of processes led by a **session leader** (usually the shell or terminal)
- When session leader exits, the terminal is closed → SIGHUP sent to foreground process group
- Default action for SIGHUP: terminate

This is why background processes (`command &`) die when you close a terminal — they're in the session's process group, get SIGHUP.

Escape mechanisms:
- `nohup command` — sets SIGHUP to SIG_IGN before exec; child ignores SIGHUP
- `setsid command` — starts command in a new session; no longer in terminal's session → no SIGHUP
- `disown` — shell removes job from its job table; no longer tracks it for SIGHUP

In production: daemons should call `setsid()` during startup to detach from controlling terminal.

### 🔥 Follow-up (Level 2)
**Q: A Kubernetes job spawns subprocesses and exits. The subprocesses become orphans. They outlive the job. What does Kubernetes do?**

**A:**
Kubernetes only tracks the container's main process (PID 1 in the container). When the job completes (main process exits), Kubernetes marks the job Done and sends SIGTERM to PID 1. But subprocesses that were spawned and became orphaned:
- If `shareProcessNamespace: false` (default): orphans are in the container's PID namespace; when container's cgroup is destroyed, all processes in it are killed via `SIGKILL`
- The cgroup kill is the safety net — all processes in a cgroup die when the cgroup is removed

BUT: this only works when the container fully terminates. If the container is in a weird state, cgroup cleanup might be delayed.

Real problem: if orphans consume resources before cgroup cleanup — they can:
1. Exhaust CPU quota (cgroup CPU accounting includes all PIDs in the group)
2. Hold open files/sockets the job was supposed to release
3. Write to volumes that other jobs are now using

Fix: use `prctl(PR_SET_CHILD_SUBREAPER)` in your entrypoint, or use tini/dumb-init which properly reap and then exit, triggering cgroup cleanup.

### 💡 Real-World Insight
A Spark job launched Python child processes for data transformation. Job completed, but Python subprocesses (orphaned, reparented to container's PID 1 = JVM) continued running for 10 minutes, consuming node CPU. Kubernetes marked the job complete but the node appeared CPU-saturated for 10 minutes after. Monitoring showed "job done" while node showed active CPU — alerting misfired. Fix: explicit process group management in the launcher — `os.killpg(os.getpgrp(), signal.SIGTERM)` on job completion.

---

## Q6. How do you clean zombie processes in production?

### 🏭 Production Context
Zombie cleanup is a live incident skill — you have `fork()` failures, service degraded, and you need to clean state without a full restart if possible.

### 🔧 Operational Steps

```bash
# Step 1: Identify zombies and their parents
ps aux | awk '$8 == "Z" {print $0}'
# STAT = Z → zombie
# PPID column → parent

ps -eo pid,ppid,stat,cmd | awk '$3 ~ /Z/'

# Step 2: Identify parent
ps -p <parent-pid> -o pid,ppid,cmd

# Step 3: Attempt SIGCHLD to parent (non-destructive)
kill -SIGCHLD <parent-pid>
# Wait 5s, check if zombies cleared
sleep 5 && ps aux | awk '$8 == "Z"'

# Step 4: If SIGCHLD didn't work and you can restart the parent
systemctl restart <service>   # Restarts parent; zombies reparented → reaped by init

# Step 5: If you can't restart — kill parent (with care)
# Zombies will be reparented to init and reaped
kill -SIGTERM <parent-pid>
# Watch: do new zombies appear? Parent's replacement must handle children properly

# Step 6: Prevent recurrence
# Check parent's SIGCHLD handler:
strace -e signal -p <parent-pid>   # Does it handle SIGCHLD?
```

### 🔍 Follow-up (Level 1)
**Q: You have 10,000 zombies. PID table is full. `fork()` fails. How do you recover without rebooting?**

**A:**
```bash
# 1. Identify the parent of most zombies
ps -eo ppid= | sort | uniq -c | sort -rn | head
# The PPID with most children = the zombie factory

# 2. Send SIGCHLD flood
kill -SIGCHLD <parent-pid>
# If parent has wait() in handler, this drains queue

# 3. If parent is unresponsive — check if it's in D state
cat /proc/<parent-pid>/status | grep State
# D state = can't be killed; must resolve I/O blocking

# 4. Kill parent (last resort — service disruption)
kill -9 <parent-pid>
# All 10k zombies reparented to init → immediately reaped

# 5. Verify PID space freed
cat /proc/sys/kernel/pid_max
ls /proc | wc -l   # Approximate PID usage
```

### 💡 Real-World Insight
A uWSGI instance in `--processes 50` mode had a bug where worker processes exited before master called `wait()` during graceful reload. During a deploy under load, 500 workers accumulated as zombies. PID table hit 32768 limit. All `fork()` calls failed → master couldn't spawn replacements → service fully down. Fix: set `vacuum = true` in uWSGI config, reduce pid_max ceiling monitoring, and add alerting on zombie count > 100.

---

## Q7. What happens when the process table is full?

### 🏭 Production Context
Process table full = `EAGAIN` from `fork()`. Every service that tries to spawn a worker, accept a connection, or run a health check gets an error. It's an instant cascading failure.

### ⚙️ Inner Workings

The "process table" is really:
1. **PID namespace limit** — `pid_max` (default 32768, max ~4M)
2. **Per-user process limit** — `ulimit -u` (`RLIMIT_NPROC`) — limits PIDs per UID
3. **System-wide PID count** — total PIDs across all namespaces

When `fork()` / `clone()` is called:
- Kernel calls `alloc_pid()` → checks `pid_max`
- Kernel checks `RLIMIT_NPROC` for the calling user
- If either limit hit → returns `-EAGAIN` → libc throws `errno = EAGAIN`
- In shell: `bash: fork: retry: Resource temporarily unavailable`

```bash
# Check current PID count
cat /proc/sys/kernel/pid_max        # Max PIDs
ls /proc | grep -E '^[0-9]+$' | wc -l  # Current PID count (approximate)

# Per-user limit
ulimit -u                           # Current user's process limit
cat /proc/<pid>/limits | grep processes

# System-wide file:
sysctl kernel.pid_max
```

### 🔍 Follow-up (Level 1)
**Q: `pid_max` is 32768 but you only see 5000 processes. Yet fork() still fails. Why?**

**A:**
The per-user `RLIMIT_NPROC` limit. Even if system has 27,000 free PIDs, if the user running the service has `ulimit -u 4096` and already has 4096 processes (threads count too!), `fork()` fails with `EAGAIN`.

Key trap: **threads count against `RLIMIT_NPROC`**. A Java service with 200 threads uses 200 of the user's process slots. A service that uses thread pools heavily can exhaust `RLIMIT_NPROC` with a single process.

```bash
# Count threads for a user
ps -u <username> -L | wc -l   # -L shows threads

# Check limit
cat /proc/<pid>/limits | grep "Max processes"

# Fix:
ulimit -u unlimited   # In service's launch script
# Or in systemd:
# TasksMax=infinity   # in [Service] section
```

### 💡 Real-World Insight
A Tomcat server running under `tomcat` user had `ulimit -u 1024` (OS default for service accounts). Tomcat used 50 threads + 50 connector threads + JVM internal threads = ~200 threads per instance. With 5 Tomcat instances under the same UID: 1000 threads → at 1024 limit. During a traffic spike, a 6th instance failed to start, and existing instances couldn't spawn request threads. Error in logs: `java.lang.OutOfMemoryError: unable to create new native thread`. Fix: set `LimitNPROC=infinity` in systemd unit.

---

## Q8. How do you debug "no more processes" / fork failure?

### 🏭 Production Context
This surfaces as: containers failing to start, shell commands producing "Resource temporarily unavailable", application logs showing thread creation failures.

### 🔧 Debug Sequence

```bash
# Step 1: Confirm it's a fork failure
# Application logs: "Resource temporarily unavailable" / EAGAIN
# System logs:
dmesg | grep -i "fork\|process\|pid"
journalctl -k | grep "fork"

# Step 2: Check PID space
cat /proc/sys/kernel/pid_max
ls /proc | grep -cE '^[0-9]+'          # Rough PID count
ps aux | wc -l                          # Process count

# Step 3: Check per-user limit
# Find the failing process's UID
ps -p <pid> -o uid,pid,cmd
# Check that UID's limits
cat /proc/<pid>/limits | grep "Max processes"
# Count that UID's threads
ps -u <uid> -L --no-headers | wc -l

# Step 4: Check threads specifically
ps -eLf | awk '$1 == "<username>"' | wc -l   # All threads for user

# Step 5: Check cgroup task limit (containers!)
cat /sys/fs/cgroup/pids/<container-cgroup>/pids.current
cat /sys/fs/cgroup/pids/<container-cgroup>/pids.max
# If current ≈ max → cgroup PID limit hit

# Step 6: Immediate mitigation
sysctl -w kernel.pid_max=4194304        # Raise system limit (max)
# For per-user:
prlimit --pid <parent-pid> --nproc=unlimited   # Fix running process

# For containers (Kubernetes):
# In pod spec:
# resources:
#   limits:
#     pids: 1000   # Check if set too low
```

### 🔍 Follow-up (Level 1)
**Q: Container has PID limit set to 200 via cgroup pids controller. App spawns 201 processes. What error does it get and what does the kernel do?**

**A:**
- `fork()` / `clone()` returns `EAGAIN` to the application
- Kernel does NOT kill any existing processes — it simply refuses the new one
- cgroup `pids.events` counter increments (`max` field)
  ```bash
  cat /sys/fs/cgroup/pids/<cgroup>/pids.events
  # max 42   ← 42 times limit was hit
  ```
- In Kubernetes, this surfaces as `TasksMax` limit; default in some distributions is 512 or 4096 per pod
- Mitigation: in pod spec, set `spec.containers[].resources` or adjust `--cgroup-driver` settings

### 💡 Real-World Insight
GKE clusters with default `pids.max=1000` per pod caused Spark executor pods to fail during shuffle phases — Spark's task executor spawned threads per partition, and with 200 partitions × several JVM threads each = hit 1000 limit. Pods would silently fail to create executor threads, causing task timeouts. The error was surfaced in JVM logs, not Kubernetes events. Fix: set `spec.template.spec.containers[0].resources` with appropriate `pids` limit, or configure cluster-wide `--pod-pids-limit` on kubelet.

---

## Q9. How does a process make a system call?

### 🏭 Production Context
Every read, write, socket operation, memory allocation is a syscall. At scale, syscall overhead is measurable: a service making 1 million syscalls/second pays ~1µs per call = 1 second of CPU per second just in syscall overhead. Understanding this path is essential for optimizing high-performance services (using `io_uring`, batching, avoiding unnecessary syscalls).

### ⚙️ Inner Workings

**x86-64 path (modern Linux):**
```
1. Application calls library function (e.g., read())
2. glibc/libc wrapper puts syscall number in RAX register
   (read = syscall #0 on x86-64)
3. Arguments go into: RDI, RSI, RDX, R10, R8, R9
4. CPU executes SYSCALL instruction
   → Saves user-space registers (RSP, RIP, RFLAGS) in kernel stack
   → Switches from Ring 3 (user) to Ring 0 (kernel)
   → Jumps to kernel's syscall entry point (entry_SYSCALL_64 in entry_64.S)
5. Kernel validates arguments (address space, permissions)
6. Dispatches via syscall table (sys_call_table[RAX])
7. Executes kernel function (sys_read)
8. Returns value in RAX
9. SYSRET instruction → back to Ring 3
10. glibc sets errno if RAX is negative (error)
```

Older mechanism (32-bit): `int 0x80` (software interrupt) — slower than SYSCALL.

VDSO optimization: some syscalls (gettimeofday, clock_gettime) are mapped into userspace — no ring switch needed. Kernel maps a read-only page into every process's address space with the code to read kernel time directly.

### 🔍 Follow-up (Level 1)
**Q: `strace` shows a service making 500,000 `gettimeofday()` calls/second. Why doesn't this kill performance?**

**A:**
`gettimeofday()` is implemented via VDSO (Virtual Dynamic Shared Object). The kernel maps a shared read-only memory page into every process. This page contains the timekeeping code and a memory-mapped copy of the kernel's time variables (updated by the kernel via hrtimer).

When process calls `gettimeofday()`:
- glibc calls VDSO version — no `SYSCALL` instruction
- Reads time directly from mapped kernel memory
- Pure userspace read — no ring switch, no kernel context switch
- Cost: ~20ns (memory read) vs ~1µs (real syscall)

`strace` actually lies here — it intercepts at the glibc wrapper level. Some implementations call through to VDSO; strace may or may not see it depending on ptrace attachment method.

```bash
perf stat -e syscalls:sys_enter_gettimeofday -p <pid>
# If count is 0 → VDSO is being used (no actual syscall)
```

### 🔥 Follow-up (Level 2)
**Q: What is `io_uring` and how does it reduce syscall overhead for I/O-heavy services?**

**A:**
Traditional I/O: each `read()`/`write()` = 1 syscall = 1 ring switch. For a service doing 500k I/O operations/second: 500k syscalls/second = significant overhead.

`io_uring` (Linux 5.1+): submit/complete ring buffers shared between kernel and userspace:
- **SQE (Submission Queue Entry):** userspace writes I/O request directly to shared ring buffer — no syscall
- **CQE (Completion Queue Entry):** kernel writes result to shared ring buffer — userspace polls — no syscall
- One `io_uring_enter()` syscall can submit AND reap hundreds of I/O operations

Result: amortize syscall cost across many operations. At 1M I/O ops/sec, instead of 1M syscalls: potentially 1,000 syscalls (one per batch of 1000 ops).

Used by: `io_uring`-aware network servers (Tokio in Rust, Nginx in development), databases (ScyllaDB, RocksDB experimental).

```bash
# Check if process uses io_uring
ls /proc/<pid>/fd | xargs -I{} readlink /proc/<pid>/fd/{} 2>/dev/null | grep io_uring
```

### 💡 Real-World Insight
A Python service used `os.stat()` in a hot loop to check file modification times for cache invalidation — 10,000 stat() calls/second, each being a real syscall. Under load, `%sy` (kernel time) hit 30% of CPU. Fix: use `inotify` (one syscall to set up, kernel pushes events) instead of polling. `inotifywait` or Python's `watchdog` library. Syscall count: 10,000/s → 10/s.

---

## Q10. What happens during user space to kernel space transition?

### 🏭 Production Context
Every context switch between user and kernel space has cost: register save/restore, TLB flush considerations (with KPTI), CPU pipeline stall. Under Spectre/Meltdown mitigations (KPTI — Kernel Page Table Isolation), this cost increased significantly — benchmarks showed 5–30% overhead on syscall-heavy workloads post-Meltdown patch.

### ⚙️ Inner Workings

**The transition (x86-64 SYSCALL path):**

```
User Space (Ring 3):
  - Process has its own page table (user portion)
  - CPL (Current Privilege Level) = 3
  - Cannot access kernel memory addresses

SYSCALL instruction executes:
  1. CPU saves RIP (return address) to RCX
  2. CPU saves RFLAGS to R11
  3. CPU switches to Ring 0 (CPL = 0)
  4. RSP loaded from MSR_LSTAR (kernel stack pointer)
  5. With KPTI: page tables switched from user PGD to kernel PGD
     (this is the Meltdown fix — user and kernel have separate page tables)
  6. Kernel saves all user registers to kernel stack (pt_regs struct)
  7. Kernel validates and dispatches syscall

Return (SYSRET):
  1. Kernel restores user registers from pt_regs
  2. With KPTI: switch back to user page tables
  3. CPU switches to Ring 3
  4. RIP restored from RCX → continues userspace code
```

**Cost breakdown:**
- Register save/restore: ~10ns
- KPTI page table switch (TLB flush): ~50–100ns on older hardware
- CPU pipeline flush: pipeline must drain before ring switch
- L1/L2 cache warming: kernel code and data not in user's cache

### 🔍 Follow-up (Level 1)
**Q: What is KPTI and why did it double the cost of syscalls on some workloads?**

**A:**
Meltdown (CVE-2017-5754) allowed userspace code to speculatively read kernel memory using CPU speculation. Fix: KPTI (Kernel Page Table Isolation) — maintain separate page tables for user and kernel space.

Before KPTI: user and kernel shared the same page table; kernel pages marked non-executable in user mode but still mapped. Syscall: no page table switch → fast.

After KPTI: on every syscall entry, CPU must:
1. Load new CR3 register (kernel page table)
2. TLB is flushed (old translations invalidated)
3. On return: reload user CR3, TLB flush again

TLB flush = all cached virtual→physical translations invalidated. Next userspace memory accesses = TLB misses = extra memory lookups. For syscall-heavy workloads (databases, file servers): 10–30% throughput regression.

Mitigation: PCID (Process-Context Identifiers) — CPUs that support PCID can tag TLB entries with process ID, avoiding full flush. Linux 4.14+ uses PCID with KPTI, recovering ~half the overhead.

```bash
# Check KPTI status
dmesg | grep -i kpti
cat /sys/devices/system/cpu/vulnerabilities/meltdown
# "Mitigation: PTI" = KPTI active
```

### 🔥 Follow-up (Level 2)
**Q: A service shows high `%sy` (kernel time) but low syscall count in `strace`. How?**

**A:**
High kernel CPU time without proportional syscall count means the kernel is doing expensive work per syscall. Possibilities:

1. **Large I/O operations** — single `write(fd, buf, 1GB)` = 1 syscall but kernel spends seconds copying data from userspace to kernel buffers (`copy_from_user`)

2. **Page fault handling** — not a syscall, but runs in kernel context. Triggered by memory access, counted in `%sy`. Lots of page faults = high kernel time, low strace syscall count

3. **Soft IRQ processing attributed to process** — if the process triggers network I/O, softirq processing runs in the context of whatever process is scheduled → charged to that process's `%sy`

4. **Futex contention** — `futex` syscall count may be low, but each blocks in kernel waiting for lock → each takes milliseconds

```bash
# Find expensive syscalls (by time, not count)
strace -T -p <pid>   # -T shows time per syscall
perf record -e syscalls:sys_exit_read -ag -- sleep 5
perf report   # Time in read() syscall

# Page fault accounting
perf stat -e major-faults,minor-faults -p <pid>
```

### 💡 Real-World Insight
Post-Meltdown (Jan 2018), a high-frequency trading firm saw latency increase from 50µs to 70µs on their Linux-based order matching engine. The engine made ~500k syscalls/second (ring buffer polling via `epoll_wait`). KPTI added ~40ns per syscall × 500k = 20ms of added CPU time per second, which manifested as increased scheduling jitter. Fix: migrate to io_uring for I/O, switch polling to kernel bypass (DPDK for network), and verify PCID is enabled. The firm also pinned processes to isolated CPUs to minimize cross-CPU TLB shootdowns.

---

# 📋 Batch 2 Summary

| Q | Topic | Graded? |
|---|---|---|
| Q1 | Process creation | — |
| Q2 | fork() vs exec() | — |
| Q3 | Copy-on-write | — |
| Q4 | Zombie process | — |
| Q5 | Orphan process | — |
| Q6 | Clean zombies in production | Operational ✓ |
| Q7 | Process table full | Operational ✓ |
| Q8 | Debug fork failures | Operational ✓ |
| Q9 | Syscall mechanics | — |
| Q10 | User→Kernel transition | — |

**Next: Batch 3 — Memory Management**
Topics: Virtual memory, RSS/VIRT, demand paging, page faults, swap, thrashing, heap/stack, anonymous vs file-backed pages