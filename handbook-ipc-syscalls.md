# IPC & Syscalls — SRE Handbook

**Audience**: Senior SRE candidates
**Purpose**: Debug production issues at the kernel communication layer
**Format**: Mental models, commands, interview one-liners

---

## 1. IPC Mechanisms — Quick Reference

### Comparison Table

| Mechanism | Speed | Direction | Buffer | Addressing | Coordination | Use Case |
|-----------|-------|-----------|--------|------------|--------------|----------|
| Pipe | Fast | 1-way | 64KB | Inherited FD | Implicit | `ls \| grep` |
| FIFO | Fast | 1-way | 64KB | Pathname | Implicit | Producer/consumer, file semantics |
| Unix socket | Fast | 2-way | Socket queue | Pathname | Implicit | Docker daemon, local APIs |
| Shared memory | **Fastest** | 2-way | Full segment | Key/ID | **Explicit lock** | HFT, real-time video |
| Semaphore | N/A | N/A | N/A | Key/ID | — | Synchronization only |
| Message queue | Fast | 2-way | Configurable | Key/ID | Implicit | Event queues with priorities |
| TCP socket | Slower | 2-way | Socket buffers | IP:Port | Implicit | Anything, anywhere |

---

### 1.1 Pipes (Anonymous)

- **Buffer**: ~64KB (kernel `PIPE_DEFAULT_BUFFERSIZE`)
- **Direction**: Unidirectional
- **Who**: Related processes (parent-child, siblings via `dup2()`)
- **Creation**: `pipe(fd[2])` → fd[0]=read, fd[1]=write
- **Behavior**: Writer blocks when buffer full. Reader blocks when buffer empty.
- **Flow control**: Built-in — kernel prevents buffer overflow

**Shell example**: `ls | grep foo`
Shell creates pipe, connects stdout of `ls` to pipe write end, stdin of `grep` to pipe read end.

**When to use**:
- ✅ Parent-child communication
- ✅ Simple pipelines
- ✅ Data fits in 64KB burst

**When NOT to use**:
- ❌ Unrelated processes (no common ancestor to inherit FDs)
- ❌ Large sustained streams (buffer may starve)
- ❌ Need bidirectional (need two pipes)

**Interview**: "What happens if writer writes 100KB to empty pipe?" → First 64KB succeed, writer blocks until reader drains ~64KB, then continues.

---

### 1.2 FIFOs (Named Pipes)

- **Filesystem object**: `mkfifo /tmp/myfifo` (type `p` in `ls -l`)
- **Open blocking**: Reader `O_RDONLY` blocks until writer opens. Writer `O_WRONLY` blocks until reader opens.
- **Buffer**: Same 64KB
- **Who**: Any process that can pathname

**Use case**:
```bash
# Terminal 1
cat /tmp/myfifo

# Terminal 2
echo "message" > /tmp/myfifo
```

**When to use**:
- ✅ Simple producer/consumer without common ancestor
- ✅ Legacy systems (FIFOs predate Unix sockets)
- ✅ Want file-like semantics (permissions, ls visibility)

**When NOT to use**:
- ❌ High performance (Unix sockets faster)
- ❌ Bidirectional (need two FIFOs)
- ❌ Message boundaries (byte stream)

---

### 1.3 Unix Domain Sockets (`AF_UNIX`)

- **Address**: Pathname (`/var/run/docker.sock`) or abstract (`@name`)
- **Type**: Stream (SOCK_STREAM) or datagram (SOCK_DGRAM)
- **Performance**: 2-3x faster than TCP on localhost (no IP stack)
- **Features**: Can pass file descriptors between processes

**Example**: Docker daemon
```bash
curl --unix-socket /var/run/docker.sock http://localhost/v1.41/containers/json
```

**Commands**:

**`ss -xap`** — List all Unix domain sockets with detailed process information
- `-x` : Show Unix sockets (AF_UNIX, AF_UNIX_DGRAM)
- `-a` : Show both listening and established sockets
- `-p` : Show the process using each socket (requires root for all processes)
- **Output columns**: NetID (kernel internal ID), State (LISTEN/CONN), Recv-Q, Send-Q, Local Address (pathname), Peer Address, Process info
- **Example output**: `u_str  LISTEN  0  128  @/tmp/.X11-unix/X0  *  users:(("Xorg",pid=1234,fd=22))`
- **What it tells you**: Which processes have Unix sockets open, what pathnames they're bound to, and the socket state

**`lsof -U`** — List all open Unix sockets across all processes
- `-U` : Filter to only Unix domain sockets (AF_UNIX)
- **Common variants**:
  - `lsof -U | grep docker` : Find Docker socket specifically
  - `lsof -U -p <pid>` : Show Unix sockets for a specific process
- **What it tells you**: Process name, PID, FD number, socket type (stream/dgram), device (inode), node (pathname)
- **Example**: `Xorg  1234  cwd    DIR  0,3  4096  2  /tmp/.X11-unix`
- **Why use it**: More human-readable than `ss`, shows file descriptor numbers and exact pathnames

**When to use**:
- ✅ Local daemon APIs (Docker, systemd, journald, D-Bus)
- ✅ High-performance local IPC
- ✅ When you need socket semantics (connect/disconnect, FD passing)

**When NOT to use**:
- ❌ Need network transparency (use TCP)
- ❌ Need firewall rules (Unix sockets bypass netfilter)
- ❌ Cross-platform (Windows named pipes different)

**Interview**: "Unix socket vs TCP localhost?" → Unix socket faster (no IP/TCP), but TCP lets you move to network without changing code.

---

### 1.4 Shared Memory

**Fastest IPC** — memory mapped into both processes' address spaces. No copying after setup.

**Types**:
- **System V**: `shmget()`, `shmat()`, `shmdt()`
- **POSIX**: `shm_open()`, `ftruncate()`, `mmap()`

**Synchronization**: **Required** — use semaphores or mutexes. Without them → race conditions, data corruption.

**Pattern**:
```c
// Creator
int shmid = shmget(key, size, IPC_CREAT | 0666);
void *ptr = shmat(shmid, NULL, 0);
// Use semaphore to protect access

// Attacher
int shmid = shmget(key, size, 0);
void *ptr = shmat(shmid, NULL, 0);
// Wait on same semaphore, then read/write
```

**Commands**:

**`ipcs -m`** — List all System V shared memory segments
- `-m` : Show only shared memory (use `-q` for message queues, `-s` for semaphores, `-a` for all)
- **Key output columns**:
  - `key` : The numeric key used to create/access the segment (in hex, e.g., `0x12345678`)
  - `shmid` : Shared memory ID (kernel handle, used with `ipcrm`)
  - `owner` : User who created it
  - `perms` : Permissions (like file permissions, e.g., `666`)
  - `bytes` : Size of segment in bytes
  - `nattch` : Number of current attachments (processes using it)
  - `status` : Additional info (usually blank)
- **Example**: `0x00000000 12345678 user  666  4096  2`
- **Interpretation**: Segment with key=0 (often default), shmid=12345678, 4KB size, 2 processes attached
- **Why use it**: Diagnose SHM leaks — segments with nattch=0 but old ctime likely orphaned

**`ipcs -m -i <shmid>`** — Get detailed information about a specific shared memory segment
- `-i <shmid>` : Show info for this specific segment ID
- **Additional details shown**: Creation time (ctime), last attach/detach time, creator PID, last operation PID
- **Use case**: Investigate a suspicious segment from `ipcs -m` output to see who created it and when
- **Example**: `ipcs -m -i 12345`

**`ipcrm -m <shmid>`** — Remove a shared memory segment
- `-m <shmid>` : Remove the segment with this ID
- **What happens**: Segment marked for deletion. **Actually removed when last process detaches** (calls `shmdt()`).
- **If forced**: All processes get `SIGSEGV` if they try to access after removal
- **Caution**: Don't remove segments that are in active use! Check `nattch` first.
- **Example**: `ipcrm -m 12345`

**`ls /dev/shm`** — List POSIX shared memory objects (tmpfs backing store)
- `/dev/shm` is a tmpfs (memory-based filesystem) where POSIX shm objects appear as files
- **Why different from System V**: POSIX shm uses filesystem namespace (`/dev/shm/myshm`) vs System V numeric keys
- **Show sizes**: `ls -lh /dev/shm`
- **Remove POSIX shm**: `rm /dev/shm/myshm` **only if no process has it open**, or `shm_unlink()` in code
- **Normal output**: `myshm` (POSIX shm), `sem.mysem` (POSIX semaphore)

**When to use**:
- ✅ Extreme performance (HFT, real-time systems)
- ✅ Large data shared frequently
- ✅ Trusted processes (same team, no security boundary)

**When NOT to use**:
- ❌ Untrusted processes (no isolation)
- ❌ Complex coordination (synchronization is hard)
- ❌ Need message boundaries (just a blob)

**Risks**: Corruption if no sync, memory leaks if not detached/removed, security (any process with key can attach).

---

### 1.5 Semaphores (Synchronization)

**Counting lock** — coordinate access to shared resources.

**Types**:
- **System V**: `semget()`, `semop()`
- **POSIX**: `sem_open()`, `sem_wait()`, `sem_post()`

**Operations**:
- `wait()` / `P()` / `down()`: decrement, block if zero
- `post()` / `V()` / `up()`: increment, wake waiters

**Used with shared memory**:
```c
sem_wait(sem);           // Enter critical section
shm->data = value;
sem_post(sem);           // Exit critical section
```

**Commands**:
```bash
ipcs -s                   # List semaphores
ipcrm -s <semid>          # Remove
```

**Common issues**: Deadlock (circular wait), priority inversion (fix with priority inheritance).

---

### 1.6 Message Queues

**Discrete messages** with priorities (not byte stream).

**Types**:
- **System V**: `msgget()`, `msgsnd()`, `msgrcv()`
- **POSIX**: `mq_open()`, `mq_send()`, `mq_receive()`

**Features**:
- Message boundaries preserved
- Priorities (msgtyp)
- Persistent (survive process death)
- Blocking/non-blocking options

**Use case**: Event-driven architecture, producer-consumer with priority.

**Commands**:
```bash
ipcs -q                   # List message queues
ipcrm -q <msqid>          # Remove
```

**When to use**:
- ✅ Need message boundaries
- ✅ Priorities matter
- ✅ Decoupling (async)
- ✅ Queue persistence

**When NOT to use**:
- ❌ Very large messages (>8KB typical limit)
- ❌ Highest performance (shared memory faster)
- ❌ Simple streaming (pipe is simpler)

---

## 2. Syscall Mechanics

### 2.1 What Happens on a Syscall

```
Your code: read(fd, buf, 1024)
  │
  ├─► Place args in registers (x86-64: rdi=fd, rsi=buf, rdx=1024)
  ├─► Execute `syscall` instruction
  │      (CPU switches from ring 3 to ring 0)
  │
  ▼
Kernel entry:
  • Save user registers
  • Read syscall number from rax (read=0)
  • Validate: fd valid? buf in user space? size OK?
  • Look up fd → file struct → f_op->read()
  • VFS dispatch: pipe_read() / sock_read() / file_read()
  • If data ready: copy to user buffer
  • If not ready: block process (TASK_INTERRUPTIBLE)
  • Set rax = return value (bytes read or -errno)
  │
  ▼
Return to user:
  • Restore registers
  • `sysret` instruction (ring 0 → ring 3)
  │
  ▼
Your code continues with return value
```

**Context switches**:
- Non-blocking: 2 (user→kernel, kernel→user)
- Blocking: 2 + scheduler switches (process sleeps, wakes later)

**Cost**: ~100-300 ns overhead per syscall (mode switch, TLB, cache).

---

### 2.2 Common Syscalls (x86-64 Linux)

| Number | Name | Purpose |
|--------|------|---------|
| 0 | read | Read from file descriptor |
| 1 | write | Write to file descriptor |
| 2 | open | Open file |
| 3 | close | Close file descriptor |
| 22 | pipe | Create pipe |
| 41 | socket | Create socket |
| 42 | connect | Connect socket |
| 43 | accept | Accept connection |
| 49 | bind | Bind socket to address |
| 50 | listen | Listen for connections |
| 57 | fork | Create process |
| 59 | execve | Execute new program |
| 60 | clone | Create thread/process |
| 202 | futex | Fast userspace mutex |
| 228 | epoll_wait | I/O multiplexing |

See `/usr/include/asm/unistd_64.h` for full list.

---

### 2.3 Context Switch Cost

**What**: Switching CPU from one process/thread to another.

**Costs**:
- **Thread→thread** (same process): ~200-500 ns (no address space change)
- **Process→process**: ~1000-2000 ns (full MMU switch, TLB flush)

**Measurement**:
```bash
perf stat -e context-switches -p <pid> sleep 5
vmstat 1 5          # cs column = context switches/sec
pidstat -w 1 5      # cswch/s (voluntary), nvcswch/s (involuntary)
```

**Thresholds**:
- < 1K/sec: normal
- 1K-10K/sec: busy but okay
- > 50K/sec: investigate (scheduler overhead)
- > 100K/sec: critical

---

## 3. Network Stack Deep Dive (localhost TCP)

### 3.1 Data Path for write() → read() on localhost

```
Process A                                     Process B
  write(sock_fd, "hello", 5)
    │ sys_write() (CS1: user→kernel)
    ▼
  sock_write() → tcp_sendmsg()
    • Add TCP header (seq, ack, flags)
    • Add IP header (127.0.0.1 → 127.0.0.1)
    • Copy to sk_buff (kernel buffer)
    ▼
  ip_queue_xmit() → dev_queue_xmit(lo)
    • Route to loopback device
    ▼
  loopback_xmit()
    • No hardware — copy directly to **receive queue** of dst socket
    • Call tcp_rcv() on same kernel thread
    ▼
  tcp_rcv() on B's socket
    • Reassemble TCP segment
    • If B blocked in read(), wake B
    │
    │ (Scheduler: switch to B)  (CS2: context switch)
    ▼
  B: read(sock_fd, buf, 1024)
    │ sys_read() (CS3: user→kernel)
    ▼
  sock_read() → tcp_recv()
    • Copy "hello" from kernel socket buffer to user buf
    • Return 5
    │
    ▼
  (CS4: kernel→user)
  B continues

  A: write() returns (CS5: kernel→user if not blocked earlier)
```

**Total context switches**: 4-5 (2 per syscall + scheduler switch between A and B).

**Why TCP slower than Unix socket**:
- More layers (IP, TCP headers, checksums)
- More data copies (kernel→kernel via loopback)
- Routing lookup (even if localhost)
- But same number of context switches!

---

### 3.2 TCP Three-Way Handshake (localhost)

**Client**:
```
socket()           → Create socket (AF_INET, SOCK_STREAM)
connect()          → Kernel sends SYN, waits for SYN-ACK (blocks ~RTT)
                   ← Kernel receives SYN-ACK, sends ACK
                   → connect() returns 0
```

**Server**:
```
socket() → bind() → listen()
accept()           → Blocks until SYN arrives
                   ← SYN received, send SYN-ACK
                   ← ACK received, connection established
                   → accept() returns new socket fd
```

**strace on client**:
```
socket(AF_INET, SOCK_STREAM, 0) = 3
connect(3, {AF_INET, 8080}, 16) = 0  ← blocks ~RTT during handshake
write(3, "GET /", 4) = 4
read(3, "HTTP/1.1 200", 20) = 20
close(3) = 0
```

---

### 3.3 Socket States & Recv-Q

`ss -ltn` shows:
- `Recv-Q`: how many bytes/connections waiting to be **accepted** (in accept queue)
- `Send-Q`: how many bytes waiting to be **sent** (in send buffer)

**LISTEN socket Recv-Q > 0**: Connections completed handshake, waiting for `accept()`. If at max backlog, new SYNs may be dropped.

**ESTABLISHED socket Recv-Q > 0**: Data received, not yet read by application. Growing = app not reading fast enough.

---

## 4. Debugging Toolkit

### 4.1 Syscall Tracing

```bash
strace -p <pid>                         # All syscalls
strace -c -p <pid>                      # Summary (count, time)
strace -T -p <pid>                      # Show time per syscall
strace -e trace=open,read,write -p <pid>  # Filter
strace -e trace=network -p <pid>        # Network syscalls only
strace -f -p <pid>                      # Follow forks
strace -o /tmp/trace.log -p <pid>       # Save to file
```

**Common patterns**:
- `read(...) = -1 EAGAIN` → Non-blocking fd, no data
- `write(...) = -1 EAGAIN` → Send buffer full
- `futex(addr, FUTEX_WAIT, ...)` → Waiting on mutex
- `accept(...) = -1 EAGAIN` → Non-blocking listen, no connections
- `... <... ETIMEDOUT>` → Network timeout
- `... <... ECONNREFUSED>` → Nothing listening

---

### 4.2 Library Tracing

```bash
ltrace -p <pid>                         # libc calls (malloc, printf, etc.)
ltrace -e malloc,free -p <pid>          # Only memory allocs
```

Useful when syscalls are fine but app-level code is slow.

---

### 4.3 IPC Inspection

```bash
# Unix sockets
ss -xap
lsof -U

# Pipes (anonymous)
lsof -p <pid> | grep pipe

# Named pipes (FIFOs)
ls -l /tmp | grep '^p'
fuser /tmp/myfifo

# Shared memory
ipcs -m
cat /dev/shm

# Message queues
ipcs -q

# Semaphores
ipcs -s

# Remove
ipcrm -m <shmid>
ipcrm -q <msqid>
ipcrm -s <semid>
```

---

### 4.4 Socket Debugging

```bash
ss -tlnp              # Listening TCP sockets with process
ss -tnp | awk '{print $7}' | cut -d',' -f2 | sort | uniq -c  # Connections per process
netstat -tnp | grep :8080

# Buffer sizes
sysctl net.core.rmem_default
sysctl net.core.wmem_default

# Backlog (SYN queue)
sysctl net.ipv4.tcp_max_syn_backlog
```

---

### 4.5 Process State

```bash
ps aux | grep <pid>              # STAT column: R/S/D/Z/T
cat /proc/<pid>/status
cat /proc/<pid>/wchan            # Kernel function sleeping in

# Stack trace (if stuck)
gdb -p <pid> -ex "thread apply all bt" -ex "quit"
```

---

### 4.6 Performance Counters

```bash
perf stat -e context-switches,syscalls:sys_enter -p <pid> sleep 5
perf top -e context-switches     # Live top-like view
perf record -g -p <pid>          # Record stack traces
perf report                      # View profile
```

---

## 5. Interview One-Liners

### IPC
- **"Pipes vs sockets?"** → Pipes: parent-child, 64KB buffer, unidirectional. Sockets: any processes, pathname/IP, bidirectional.
- **"Fastest IPC?"** → Shared memory (no copy), but needs explicit locking.
- **"How to protect shared memory?"** → Use semaphores (sem_wait before access, sem_post after).
- **"Unix socket vs TCP localhost?"** → Unix socket 2-3x faster (no IP stack), but TCP portable to network.
- **"What is a FIFO?"** → Named pipe (`mkfifo`). Has filesystem entry, blocks on open until both ends connected.
- **"Message queue vs pipe?"** → MQ: message boundaries, priorities, async, unrelated processes. Pipe: byte stream, simpler.

### Syscalls
- **"What happens on read()?"** → User→kernel switch, VFS dispatch, copy from kernel buffer to user, kernel→user switch. 2 context switches if non-blocking.
- **"How expensive is a syscall?"** → ~100-300 ns overhead. 10M/sec = 1-3 sec pure overhead.
- **"Why are context switches expensive?"** → Save/restore registers, TLB flush, cache pollution.
- **"How many switches for localhost TCP write+read?"** → ~4-5 (2 per syscall + scheduler switch).
- **"What is futex?"** → Fast userspace mutex. Lock fast path in userspace; slow path blocks via kernel futex syscall.
- **"How to see current syscall of blocked process?"** → `cat /proc/<pid>/syscall`

---

## 6. Common Production Scenarios

### Scenario 1: Non-blocking socket send fails with EAGAIN

**Symptom**: `send()/sendto()` returns -1, errno=EAGAIN (" Resource temporarily unavailable")

**Cause**: Send buffer full, socket is non-blocking.

**Fix**:
```c
// Use select/poll/epoll to wait for writable
struct pollfd pfd = { .fd = sock, .events = POLLOUT };
poll(&pfd, 1, -1);  // Wait indefinitely
// Now retry send() (may still send partial, handle that)
```

**Or**: Increase socket buffer size:
```bash
sysctl -w net.core.wmem_default=262144
```

---

### Scenario 2: Accept queue full (Recv-Q at backlog limit)

**Symptom**: `ss -ltn` shows Recv-Q = 128 (or your backlog size). Connections being dropped.

**Cause**: Application not calling `accept()` fast enough. Worker pool exhausted or accept loop slow.

**Fix**:
- Increase backlog: `listen(fd, 1024)` (but kernel caps it via `somaxconn`)
- Increase kernel limit: `sysctl -w net.ipv4.tcp_max_syn_backlog=2048`
- Add more workers (processes/threads) to call `accept()`
- Use `accept4()` with `SOCK_NONBLOCK` to avoid blocking on accept

---

### Scenario 3: Process stuck in futex(FUTEX_WAIT)

**Symptom**: `strace -p <pid>` shows repeated `futex(addr, FUTEX_WAIT, expected, NULL)`.

**Cause**: Thread waiting on mutex that never unlocked (deadlock or crashed holder).

**Diagnosis**:
```bash
# Get futex address from strace
# Find which thread holds the mutex:
gdb -p <pid>
(gdb) p/x *(int *)0xADDR              # Check futex word value
(gdb) thread apply all bt             # Look for thread in critical section
(gdb) info threads                    # Which thread is in pthread_mutex_lock?

# Check if another process holds it (cross-process mutex)
# Usually same-process for pthread mutexes.
```

**Fix**: Code fix — ensure every lock has matching unlock, even on error paths. Use RAII (C++), `defer` (Go), `try/finally` (Java/Python).

---

### Scenario 4: Shared memory leak (many old segments)

**Symptom**: `ipcs -m` shows dozens of segments from your app, some days old.

**Cause**: Processes attach to SHM but never detach on exit, or creator never calls `IPC_RMID` / `shm_unlink()`.

**Fix**:
```bash
# Remove all old segments (>24h)
ipcs -m | awk '$6 < $(date -d "24 hours ago" +%s) {print $2}' | xargs -r ipcrm -m

# In code:
// POSIX: unlink immediately after creation (removes name, segment lives until all detach)
shm_unlink("/myshm");
// Or: creator calls shmctl(shmid, IPC_RMID, NULL) after last attach
```

**Prevention**: Set up monitoring alert on SHM segment count > threshold.

---

### Scenario 5: Pipe writer blocks indefinitely

**Symptom**: Process writing to pipe blocks, reader seems alive but not reading.

**Cause**: Reader blocked on something else (deadlock), or reader died without closing pipe.

**Diagnosis**:
```bash
strace -p <reader_pid>    # Is it stuck? What syscall?
ps aux | grep <reader>    # STAT column — D? Z?
cat /proc/<reader_pid>/wchan
lsof -p <reader_pid>      # Is pipe FD still open?
```

**Fix**: Kill writer if reader is dead/unstoppable. Fix deadlock in reader code.

---

## 7. Key /proc Files

```bash
/proc/meminfo              # Memory stats
/proc/<pid>/status         # Process stats (VmRSS, VmSize,SigBlk, etc.)
/proc/<pid>/maps           # Memory mappings
/proc/<pid>/wchan          # Kernel function process sleeping in
/proc/<pid>/syscall        # Current syscall (if in syscall)
/proc/<pid>/fd/            # Open file descriptors (symlinks)
/proc/<pid>/cwd           # Current working directory
/proc/<pid>/cmdline       # Full command line
/proc/<pid>/environ       # Environment variables
/proc/sys/kernel/pid_max  # Maximum PID
```

---

## 8. Quick Command Reference

| Task | Command |
|------|---------|
| Trace all syscalls | `strace -p <pid>` |
| Trace network only | `strace -e trace=network -p <pid>` |
| Count syscalls | `strace -c <cmd>` |
| See socket buffers | `ss -tnp` |
| Unix sockets | `ss -xap` |
| Shared memory list | `ipcs -m` |
| Message queues | `ipcs -q` |
| Semaphores | `ipcs -s` |
| Remove SHM | `ipcrm -m <shmid>` |
| Process state | `ps aux \| grep <pid>` |
| Wait channel | `cat /proc/<pid>/wchan` |
| Context switches | `perf stat -e context-switches -p <pid>` |
| Syscalls/sec | `vmstat 1` (cs column) |
| Open files per process | `lsof -p <pid>` |
| Which process owns socket? | `lsof -i :8080` |

---

**Revision tip**: Walk through each scenario in this handbook and explain it out loud. Can you read strace output and diagnose the problem? Can you choose the right IPC for a given use case? Can you trace a syscall from user code to kernel and back?
