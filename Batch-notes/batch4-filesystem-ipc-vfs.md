# Batch 4 — File System, IPC & VFS
### Staff+ Level | Self-Interrogation + Spot Grading on Operational Questions
---

## Q21. What is a file descriptor?

### 🏭 Production Context
File descriptors are the universal handle for I/O in Linux. Every socket, file, pipe, device, timer, epoll instance, eventfd is a file descriptor. FD exhaustion silently cripples services without crashing them. Understanding the FD abstraction is foundational for every production I/O problem.

### ⚙️ Inner Workings

An FD is an integer — an index into the process's **open file table** (stored in `task_struct→files→fdtable`).

**Three-level abstraction:**
```
Process FD Table:
  FD 0 → file description pointer → inode (stdin)
  FD 1 → file description pointer → inode (stdout)
  FD 2 → file description pointer → inode (stderr)
  FD 3 → file description pointer → inode (socket)

File Description (struct file in kernel):
  - current offset (for files)
  - open flags (O_RDONLY, O_WRONLY, etc.)
  - reference count (how many FDs point here)
  - pointer to inode

Inode:
  - actual file metadata
  - pointer to file operations (read, write, seek, etc.)
```

**Key behaviors:**
- `dup(fd)` → creates new FD pointing to same file description (same offset, shared)
- `fork()` → child inherits all FDs; both point to same file descriptions
- `open()` → always creates new file description (separate offset, separate state)
- FD 0, 1, 2 → stdin, stdout, stderr by convention

```bash
# Inspect a process's FDs
ls -la /proc/<pid>/fd/           # Symlinks showing what each FD points to
lsof -p <pid>                    # Detailed FD listing with type and state
cat /proc/<pid>/fdinfo/<fd>      # Offset, flags, mount ID for specific FD
```

### 🔍 Follow-up (Level 1)
**Q: Two processes open the same file independently. They share the inode but NOT the file description. What difference does this make?**

**A:**
Independent `open()` calls → independent file descriptions → independent read/write offsets.

Process A reads 4KB → its offset at 4KB. Process B reads 4KB → its offset starts at 0, advances to 4KB independently. Neither affects the other.

But: `fork()` → child and parent share the SAME file description (same offset). Parent reads 4KB → offset at 4KB. Child reads next → gets bytes 4KB–8KB. This is the shared offset behavior, useful for pipe-like patterns.

Production implication:
- Log files: if parent and child both write to a log FD (post-fork), they share offset → writes are serialized correctly (atomic for small writes)
- Database: DO NOT fork after opening DB connections — child inherits connection file descriptions, both parent and child send queries on same socket → protocol corruption

```bash
# Test: open same file in two processes, watch offsets diverge
strace -e trace=lseek,read cat /proc/self/fd/0   # Shows seek position
```

### 💡 Real-World Insight
A Python web worker (gunicorn) using `preload_app=True` loaded the app (including DB connections) before forking workers. Each worker inherited the master's DB connection FD. Multiple workers sent queries on the same socket simultaneously → PostgreSQL received interleaved query streams → "unexpected message type" errors. Fix: `postfork` hook to close and re-open DB connections in each worker process.

---

## Q22. What consumes file descriptors apart from files?

### 🏭 Production Context
The surprise for most engineers: FDs are NOT just open files. In a typical microservice, only 20% of FDs are actual files. The other 80% are network sockets, epoll instances, and timers. This matters because `lsof` output filtered for regular files won't show the real FD pressure.

### ⚙️ Inner Workings — FD Types

Every `open()`/`socket()`/`pipe()`/`*_create()` that returns an integer is an FD.

**Complete FD consumer list:**
```
Network:
  - TCP socket (1 FD per connection: listen socket + each accepted connection)
  - UDP socket
  - Unix domain socket
  - Raw socket

Filesystem:
  - Regular files (including deleted files still open)
  - Directories (opendir() → FD)
  - Character/block devices
  - Named pipes (FIFOs)

IPC:
  - Anonymous pipes (pipe() → 2 FDs: read end + write end)
  - Eventfd (eventfd())
  - Signalfd (signalfd())

Kernel notification:
  - epoll instance (epoll_create() → 1 FD; each monitored socket does NOT consume extra FDs)
  - inotify instance (inotify_init())
  - fanotify
  - timerfd (timerfd_create())

Process:
  - memfd (memfd_create() → anonymous memory as a file)
  - pidfd (pidfd_open() → process as a file)

TLS/crypto:
  - /dev/urandom, /dev/null, /dev/zero (opened at startup usually)
```

```bash
# FD type breakdown for a process
ls -la /proc/<pid>/fd | awk '{print $NF}' | 
  sed 's|/.*||; s|\[.*\]|[socket]|' | sort | uniq -c | sort -rn
# Count: pipes, sockets, files, anon_inode, etc.

lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -rn
# TYPE column: IPv4, IPv6, FIFO, REG, DIR, unix, etc.
```

### 🔍 Follow-up (Level 1)
**Q: An epoll server handles 100,000 concurrent connections. How many FDs does it use for epoll itself vs for connections?**

**A:**
- 1 FD for the `epoll_create()` instance itself
- 100,000 FDs for 100,000 accepted sockets
- 1 FD for the listening socket

Total: 100,002 FDs. epoll does NOT create additional FDs for registered events. The `epoll_ctl(ADD)` call registers a socket to be monitored, but the FD for that socket already exists. This is epoll's key advantage over `select()` (which has FD_SETSIZE = 1024 limit).

```bash
# Configure for 100k connections:
sysctl -w net.core.somaxconn=65536         # Listen backlog
sysctl -w net.ipv4.tcp_max_syn_backlog=65536
ulimit -n 200000                            # FD limit (connections + overhead)
# Permanent (systemd):
# LimitNOFILE=200000
```

### 💡 Real-World Insight
A Go service using `net/http` had an unexpected FD leak: `http.Get()` without `resp.Body.Close()` — the response body is a socket FD. Each unclosed response = 1 FD leak. The service had a code path that returned early on non-200 response without closing the body. After 6 hours of traffic: 65,536 FD limit hit. `lsof -p <pid> | grep ESTABLISHED | wc -l` showed 65,000 established connections, all in CLOSE_WAIT. Always: `defer resp.Body.Close()` immediately after `http.Get()`.

---

## Q23. How do you debug a file descriptor leak?

### 📊 Grading Applied (Operational Question)

### 🏭 Production Context
FD leaks are slow-burn failures: service runs for hours or days, FD count grows monotonically, eventually hits `ulimit -n`, new connections/files rejected. The tricky part: the service appears healthy until the moment it isn't.

### 🔧 Debug Sequence

```bash
# ── Phase 1: Confirm leak (FD count growing over time) ──
watch -n5 'ls /proc/<pid>/fd | wc -l'
# Monotonically increasing count = leak

# ── Phase 2: Snapshot and identify FD types ──
lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -rn
# Which type is growing? (IPv4? FIFO? REG?)

lsof -p <pid> | grep CLOSE_WAIT | wc -l
# CLOSE_WAIT count → socket FD leak (client disconnected, server not closing)

ls -la /proc/<pid>/fd | grep deleted | wc -l
# FDs to deleted files → log file or temp file not closed after rotation

# ── Phase 3: Time-series snapshot comparison ──
ls -la /proc/<pid>/fd > /tmp/fds_t1.txt
sleep 300
ls -la /proc/<pid>/fd > /tmp/fds_t2.txt
diff /tmp/fds_t1.txt /tmp/fds_t2.txt
# New entries in t2 that weren't in t1 = leaked FDs

# ── Phase 4: Strace to catch open-without-close ──
strace -f -e trace=open,openat,socket,accept,close,dup -p <pid> 2>&1 | 
  awk '/open|socket|accept/{opens++} /close/{closes++} END{print "Opens:", opens, "Closes:", closes}'
# Opens >> Closes over time = definitive leak

# ── Phase 5: eBPF — track FD lifecycle ──
bpftrace -e '
tracepoint:syscalls:sys_exit_socket /retval >= 0/ {
    @sockets[pid, retval] = nsecs;
}
tracepoint:syscalls:sys_enter_close {
    delete(@sockets[pid, args->fd]);
}
interval:s:60 {
    print(@sockets); clear(@sockets);
}'
# Shows sockets open for > 60 seconds without being closed

# ── Phase 6: Language-specific tools ──
# Java: check for unclosed resources
jstack <pid> | grep -i "waiting\|blocked"
# Python: gc.get_objects() to find uncollected file objects
# Go: runtime/debug + pprof goroutine dump
```

### 🔍 Follow-up (Level 1)
**Q: You identify CLOSE_WAIT sockets as the FD leak. The application team says "we always call `close()`." Who is right?**

**A:**
Both can be right — and the bug is subtle.

CLOSE_WAIT means: the remote end sent FIN (closed their write side), our kernel acknowledged it, but OUR application has NOT called `close()` on the socket yet.

Application calling `close()` on normal completion ≠ application calling `close()` when client disconnects first.

Common bug:
```python
while True:
    data = sock.recv(4096)
    if not data:
        break    # Client disconnected → EOF → but sock.close() never called!
    process(data)
# sock.close() is here — but unreachable if break exits the while loop without close
```

The "correct" close in the normal code path doesn't handle the early disconnect case.

```bash
# Verify: check CLOSE_WAIT socket remote addresses
ss -tnp state close-wait | awk '{print $4}'   # Remote addresses
# If these are client IPs that disconnected → app didn't close after EOF
```

### 🔥 Follow-up (Level 2)
**Q: You fix the CLOSE_WAIT leak, but FD count is still growing — now it's open regular files to deleted paths. How?**

**A:**
`/proc/<pid>/fd/X → /path/to/file (deleted)` — process has FD open to a file that was unlinked.

Common causes:
1. **Log rotation without signal:** logrotate rotates the file (renames/deletes old), but the process still writes to the old inode via its open FD. File is "deleted" from the directory but exists until FD is closed.
   ```bash
   # Fix: logrotate with postrotate to send SIGUSR1 (log reopen signal)
   # Or: use logrotate's copytruncate (copies file, truncates original — no FD needed)
   ```

2. **Temp file not cleaned up:** process creates temp file, unlinks it immediately (standard Unix pattern for self-cleaning temp files), but forgets to close the FD
   ```bash
   ls -la /proc/<pid>/fd | grep deleted | xargs -I{} ls -la {}
   # Shows size of "deleted" files still held open
   ```

3. **Crash before cleanup:** process opened files, caught SIGTERM, cleanup code had a bug → FDs not closed before exit. Next startup: same pattern.

```bash
# Find FDs to deleted files:
lsof -p <pid> | grep deleted
# Or:
find /proc/<pid>/fd -type l | xargs readlink | grep "(deleted)"
```

---

### 📊 Grading

| Dimension | Score |
|---|---|
| Production Realism | 9 |
| Technical Depth | 9 |
| Debugging Practicality | 10 |
| Failure Awareness | 9 |
| Clarity & Signal | 8 |

**Total: 45/50**

**🔍 Critical Feedback:**
- Missing `ss -tnp state close-wait` as the fastest CLOSE_WAIT diagnostic — only referenced indirectly
- Should mention `ulimit -n` vs `/proc/<pid>/limits` distinction for running processes
- The eBPF snippet is correct but would benefit from a note about kernel version requirements (bpftrace >= 0.9)

---

## Q24. What is an inode?

### 🏭 Production Context
Inodes are the backbone of the filesystem. Every file, directory, symlink = one inode. Running out of inodes causes "disk full" with space remaining — a production mystery that panics junior engineers.

### ⚙️ Inner Workings

An inode (index node) stores file metadata but NOT the filename and NOT the file data:

```
Inode contains:
  - File type (regular, directory, symlink, device, pipe)
  - Permissions (mode bits: owner/group/other rwx)
  - Owner UID, GID
  - File size
  - Timestamps (atime, mtime, ctime)
  - Link count (how many directory entries point to this inode)
  - Pointers to data blocks (direct, indirect, double-indirect — traditional ext2/3)
  - For ext4: extent tree (more efficient for large files)

Inode does NOT contain:
  - Filename (stored in directory entry)
  - File data (stored in data blocks, pointed to by inode)
```

**Inode numbers:**
```bash
ls -i /etc/passwd   # Shows inode number
stat /etc/passwd    # Full inode metadata
```

Directory entries are just mappings: `filename → inode number`. Multiple filenames can point to the same inode (hard links).

Inode table: fixed at filesystem creation time. `mkfs.ext4` calculates inode count based on total size and bytes-per-inode ratio (default: 1 inode per 16KB of disk space).

### 🔍 Follow-up (Level 1)
**Q: `ls -l` shows a file with link count 3. What does that mean?**

**A:**
Three directory entries point to this inode. The inode has 3 hard links:
```bash
mkdir test
echo "data" > test/file.txt           # link count = 1
ln test/file.txt test/file_link.txt   # link count = 2
ln test/file.txt /tmp/file_link2.txt  # link count = 3
ls -l test/file.txt
# -rw-r--r-- 3 user group ...   ← the "3"
```

When `rm test/file.txt` is called:
- Link count decremented to 2
- File data NOT deleted (inode still has 2 references)
- Only when link count reaches 0 AND no open FDs → data blocks freed

This is also why: deleting a file while another process has it open → file "disappears" from directory but data accessible via open FD until FD closed.

```bash
lsof | grep deleted   # Files deleted from directory but still open
```

### 💡 Real-World Insight
A tmpfs filesystem (/tmp) on a build server had 256K inode limit (default for tmpfs). CI build jobs unpacked thousands of npm packages — each file = 1 inode. At peak: 260K files → inode exhaustion → build jobs failed with "No space left on device" → disk showed 40GB free. Fix: remount tmpfs with more inodes: `mount -o remount,nr_inodes=2M /tmp`. Long-term: move build cache to ext4 with custom inode ratio.

---

## Q25. Why do you get "disk full" even when space is available?

### 🏭 Production Context
"No space left on device" with `df -h` showing 40% free is a classic on-call panic. Two root causes: inode exhaustion or reserved blocks. Both have different fixes.

### ⚙️ Inner Workings

**Cause 1: Inode exhaustion**
- Disk has free blocks (space) but no free inodes
- Creating a new file requires an inode slot
- Small-file-heavy workloads exhaust inodes before blocks

```bash
df -i    # Check inode usage (IUse% column)
df -h    # Check block usage
# If df -i shows 100% but df -h shows 40% → inode exhaustion
```

**Cause 2: Reserved blocks (ext4)**
- ext4 reserves 5% of blocks for root user (prevents system freeze from disk full)
- Non-root processes hit "disk full" when disk is 95% full (5% reserved)
- `tune2fs -m 1 /dev/sda1` reduces reserved to 1%

**Cause 3: Filesystem corruption**
- Orphaned inodes claiming blocks but not visible via `ls`
- `e2fsck` can reclaim them

```bash
# Diagnose
df -ih /var    # Inodes for /var
df -h /var     # Blocks for /var

# Find inode hogs (directories with most files)
find /var -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head -20

# Count files per directory (inode consumers)
for dir in /var/log /tmp /var/cache; do
    echo "$dir: $(find $dir -maxdepth 1 | wc -l) files"
done

# Check reserved blocks (ext4)
tune2fs -l /dev/sda1 | grep -i "reserved\|block count"
```

### 🔍 Follow-up (Level 1)
**Q: Inode table is full. You can't delete files to free inodes because `rm` itself needs to create/modify directory entries. How do you recover?**

**A:**
`rm` doesn't create new inodes — it modifies directory entries (unlinks them) and decrements inode link counts. `rm` does NOT require a free inode.

BUT: the real problem is you can't do anything that requires a NEW inode (create file, create directory). Recovery path:

1. **Delete files to free inodes (rm works without new inode):**
   ```bash
   find /var/log -name "*.log" -mtime +7 -delete   # Old logs
   find /tmp -maxdepth 1 -mtime +1 -delete          # Old temp files
   ```

2. **Find and delete large numbers of small files:**
   ```bash
   find /path -name "*.sess" -delete   # PHP session files (classic inode hog)
   find /path -name "*.lock" -delete
   ```

3. **If rm itself fails (command not found from FD exhaustion too):**
   ```bash
   # Use bash builtins, no external commands
   cd /var/log && for f in *.log; do unlink "$f"; done
   ```

4. **Nuclear: tar to a temp location, delete originals, untar:**
   Only if you have space on another filesystem.

### 💡 Real-World Insight
PHP application storing sessions in `/var/lib/php/sessions/` as individual files. Each user session = 1 file = 1 inode. At 100k concurrent users: 100k inodes. After a traffic spike with session leak (sessions not expiring): 1M files → inode exhaustion on the ext4 volume (which had 500K inodes). Webserver couldn't create new session files → all new requests failed with 500. Fix: enable PHP session.gc_probability, add cron to clean old sessions, AND switch to Redis-based sessions (0 inodes for sessions).

---

## Q26. How do you debug inode exhaustion?

### 🔧 Debug Sequence

```bash
# Step 1: Confirm inode exhaustion
df -i
# Filesystem      Inodes  IUsed   IFree IUse% Mounted on
# /dev/sda1      1310720 1310720     0  100%  /var   ← 100% = exhausted

# Step 2: Find which directory has the most files
# WARNING: this can be slow on large filesystems — run with ionice
ionice -c 3 find / -xdev -printf '%h\n' 2>/dev/null | sort | uniq -c | sort -rn | head -20

# Faster: check known inode hog locations first
du --inodes -d3 /var | sort -rn | head -20

# Step 3: Identify file type
ls /suspected/path | head -5   # What kind of files?
stat /suspected/path/file1     # How old?

# Step 4: Clean up
# PHP sessions:
find /var/lib/php/sessions -mtime +1 -delete

# Log files:
find /var/log -name "*.log.*" -mtime +7 -delete

# Temporary files:
find /tmp -maxdepth 2 -mtime +1 -not -name "." -delete

# Step 5: Prevent recurrence
# Add cron:
0 2 * * * find /var/lib/php/sessions -mtime +1 -delete 2>/dev/null

# Step 6: Long-term — tune inode ratio on new filesystems
mkfs.ext4 -T largefile /dev/sdb   # Fewer, larger inodes (for big-file workloads)
mkfs.ext4 -T small /dev/sdb       # More inodes (for small-file workloads)
# Or set bytes-per-inode directly:
mkfs.ext4 -i 4096 /dev/sdb        # 1 inode per 4KB (4x more inodes than default)
```

### 🔍 Follow-up (Level 1)
**Q: You can't run `find` (FD exhaustion or inode full). What alternative do you have?**

**A:**
```bash
# Pure bash — no external commands needed
ls /suspected/dir | wc -l   # Count files in dir (bash builtin ls)

# If even ls fails:
echo /suspected/dir/* | tr ' ' '\n' | wc -l   # Glob expansion in bash

# Kernel stats directly:
cat /proc/sys/fs/inode-nr   # total inodes in use system-wide
# Format: total allocated | free
```

### 💡 Real-World Insight (same as Q25 — PHP sessions pattern)

---

## Q27. What is the difference between hard link and soft link?

### ⚙️ Inner Workings

**Hard link:**
- Another directory entry pointing to the SAME inode
- Same file, different names (potentially in different directories)
- `ln source.txt hardlink.txt`
- Both filenames point to inode #12345
- Deleting source.txt → hardlink.txt still fully accessible (link count: 2→1)
- Data deleted only when ALL hard links are removed AND no open FDs
- Cannot cross filesystems (inode numbers are filesystem-local)
- Cannot hard-link directories (would create cycles in the directory tree — fsck can't handle it)

**Soft link (symbolic link):**
- A special file that contains a pathname string
- Points to a path, not an inode
- `ln -s /original/path symlink`
- Symlink has its own inode (type: symlink); content = target path string
- If target is deleted → symlink becomes dangling (broken)
- Can cross filesystems
- Can link to directories
- Adds one extra path resolution step (kernel follows the link)

```bash
# Hard link:
ln /etc/passwd /tmp/passwd_hardlink
ls -li /etc/passwd /tmp/passwd_hardlink
# SAME inode number, link count = 2

# Soft link:
ln -s /etc/passwd /tmp/passwd_symlink
ls -li /tmp/passwd_symlink
# DIFFERENT inode number, l type, -> /etc/passwd
stat /tmp/passwd_symlink   # Shows symlink's own size = len(target_path)
```

### 🔍 Follow-up (Level 1)
**Q: A deployment script creates a symlink `current → /app/release-42`. During deployment, it atomically switches to `current → /app/release-43`. Why is this atomic and what syscall makes it so?**

**A:**
`rename(oldpath, newpath)` is atomic at the filesystem level. The deployment pattern:
```bash
ln -s /app/release-43 /app/current_new   # Create new symlink
mv /app/current_new /app/current         # Atomic rename — mv calls rename()
```
`rename()` is a single syscall. From any observer's perspective, `current` either points to release-42 or release-43 — never to a non-existent target. No moment where the symlink is broken.

This is the blue-green deployment pattern at the filesystem level. Used by:
- Capistrano deployment tool
- npm/yarn package managers (atomic publish)
- Container image layers

```bash
strace mv /app/current_new /app/current
# rename("/app/current_new", "/app/current")  = 0
```

---

## Q28. Why can't hard links cross filesystems?

### ⚙️ Inner Workings

Inode numbers are only meaningful within a single filesystem. They're essentially an index into that filesystem's inode table.

If you tried to hard-link `/var/log/app.log` (on `/dev/sdb`, inode #5432) to `/home/user/app.log` (on `/dev/sda`):
- `/home/user/app.log` would need to reference inode #5432
- But inode #5432 on `/dev/sda` points to a completely different file (or doesn't exist)
- The filesystem layer has no cross-device inode namespace

The kernel enforces this: `link()` syscall returns `EXDEV` (cross-device link) if source and destination are on different filesystems.

```bash
ln /var/log/test.log /tmp/test_link
# ln: failed to create hard link '/tmp/test_link' => '/var/log/test.log': Invalid cross-device link
# errno = EXDEV
```

**Workaround:** bind mounts — mount a directory from one filesystem into another filesystem's path. Files accessed via bind mount are still on their original filesystem. But hard links between bind-mount path and original path still fail (same underlying filesystem, different mount points — actually works if same underlying device).

### 🔍 Follow-up (Level 1)
**Q: Docker images use hard links for layer deduplication in overlayFS. But overlayFS layers are on the same device — how does Docker ensure this?**

**A:**
Docker's `overlay2` storage driver requires all layers to be on the same underlying filesystem. When pulling images:
- All layers are stored in `/var/lib/docker/overlay2/`
- Same ext4/xfs filesystem → hard links work for deduplicating identical files across layers
- `l/` directory in overlay2 contains short symlinks to layer directories; hard links in lower layers point to shared inodes

If `/var/lib/docker` is on a different device than the overlay layers → Docker falls back to copying (no deduplication). This is why Docker recommends a dedicated partition for `/var/lib/docker` but it must be a single filesystem.

```bash
# Check Docker storage driver
docker info | grep "Storage Driver"
# overlay2 → efficient; vfs → no hard links, worst performance
```

---

## Q29. How does Linux resolve a symbolic link?

### ⚙️ Inner Workings

Path resolution in Linux (`namei()` in `fs/namei.c`):

For each component in the path, kernel:
1. Looks up filename in current directory → gets inode
2. Checks inode type
3. If regular file or directory → continue path traversal
4. If symlink → read symlink content (the target path)
5. If target is absolute → restart from `/`
6. If target is relative → restart from current directory
7. Continue with remaining path components

**Symlink follow limit:** `MAXSYMLINKS = 40`. If resolution requires following >40 symlinks → `ELOOP` error (prevents infinite symlink loops).

```bash
# Watch symlink resolution
strace -e trace=readlink,stat,lstat ls /usr/bin/python3
# readlink("/usr/bin/python3") = "/usr/bin/python3.10"
# Multiple steps if chained symlinks

# Find dangling symlinks:
find /usr/bin -type l | while read l; do
    [ -e "$l" ] || echo "Dangling: $l -> $(readlink $l)"
done

# Resolve full symlink chain:
readlink -f /usr/bin/python   # Follows all links to final target
```

### 🔍 Follow-up (Level 1)
**Q: What is the difference between `stat` and `lstat` for symlinks?**

**A:**
- `stat(path)` — follows symlinks; returns metadata of the TARGET file, not the symlink itself
- `lstat(path)` — does NOT follow symlinks; returns metadata of the symlink itself

```bash
stat /usr/bin/python3      # Shows stats of python3.10 (the target)
lstat /usr/bin/python3     # Shows stats of the symlink (size = len(target_path), type = l)
```

In C and tools:
- `ls -l` uses `lstat` (shows `l` in permissions column)
- `ls -lL` uses `stat` (follows symlinks, shows target stats)
- `find -L` follows symlinks
- `open()` follows symlinks; `open(O_NOFOLLOW)` doesn't

Production gotcha: `O_NOFOLLOW` is important for security. Writing to a symlink that an attacker controlled → arbitrary file write. Always use `O_NOFOLLOW` when path trust is uncertain.

---

## Q30. What are different IPC mechanisms in Linux?

### 🏭 Production Context
Choosing the wrong IPC mechanism is a performance bottleneck. At scale:
- Pipes: simple but synchronous, buffered only 64KB
- Unix sockets: full-duplex, much higher throughput
- Shared memory: zero-copy, microsecond latency, but needs synchronization
- Message queues: ordered delivery, but kernel-limited queue depth

### ⚙️ Full IPC Taxonomy

**1. Pipes (anonymous):**
```bash
ls | grep foo   # Shell creates pipe between ls and grep
# One FD read end, one FD write end
# Unidirectional, in-kernel buffer (64KB default)
# Only between parent/child (shares FD via fork)
```

**2. Named Pipes (FIFOs):**
```bash
mkfifo /tmp/myfifo   # Creates special file in filesystem
# Two unrelated processes can communicate
# Blocks until both reader and writer open
```

**3. Unix Domain Sockets:**
```bash
socket(AF_UNIX, SOCK_STREAM, 0)
# Full-duplex, bidirectional
# Much higher throughput than pipes (~1GB/s vs ~100MB/s)
# No network stack overhead
# Docker's API, MySQL's local connection, Nginx↔PHP-FPM
```

**4. System V Shared Memory / POSIX Shared Memory:**
```bash
# POSIX:
shm_open("/myshm", O_CREAT|O_RDWR, 0600)   # Creates /dev/shm/myshm
mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0)
# Zero-copy: both processes see same physical pages
# Need mutex/semaphore for synchronization
# Postgres uses shared memory for buffer pool
```

**5. Message Queues (POSIX/System V):**
```bash
# Kernel-managed queue with message boundaries
# Discrete messages (unlike stream pipes)
# mq_open(), mq_send(), mq_receive()
# Max queue depth: /proc/sys/fs/mqueue/msg_max
```

**6. Signals:**
```bash
# Asynchronous notification (limited data: signal number only)
# Signal handlers must be async-signal-safe
kill -USR1 <pid>   # Send SIGUSR1 to process
```

**7. Sockets (TCP/UDP):**
```bash
# Loopback for local IPC (127.0.0.1)
# Goes through full network stack → slower than Unix sockets
# But: can extend to remote hosts without code change
```

**8. eventfd / signalfd (modern):**
```bash
eventfd(0, EFD_NONBLOCK)   # Lightweight notification FD
# Used by container runtimes for lightweight event passing
```

**Performance hierarchy (latency, same host):**
```
Shared memory + futex: ~200ns
Unix socket: ~2µs
Pipe: ~5µs
TCP loopback: ~30µs
```

### 🔍 Follow-up (Level 1)
**Q: Nginx communicates with PHP-FPM via Unix socket OR TCP. Which is faster and why does it matter at scale?**

**A:**
Unix socket wins — and significantly:
- Unix socket: kernel copies data directly between process buffers, no network stack (no TCP header parsing, no checksum, no congestion control, no ACK round-trip)
- TCP loopback: goes through full TCP stack even though both are on same host — TCP segmentation, ACK, checksum computation

Benchmark: Unix socket ~2µs latency vs TCP loopback ~30µs. At 10,000 req/s: 10,000 × 28µs saved = 280ms/s of CPU saved on Nginx alone.

```nginx
# Nginx config:
upstream php {
    server unix:/var/run/php-fpm.sock;  # Unix socket (faster)
    # server 127.0.0.1:9000;           # TCP loopback (slower)
}
```

At scale (50k+ req/s), switching Nginx→PHP-FPM from TCP to Unix socket reduced PHP-FPM CPU by 15%.

---

## Q31. What is a signal in Linux?

### ⚙️ Inner Workings

A signal is an asynchronous notification sent to a process. It's the kernel's way of saying "something happened that needs your attention."

**Signal delivery path:**
```
Sender calls kill(pid, signum)  →  kernel sets bit in target's pending signal mask
                                    (in task_struct.signal.pending)
At next scheduling point (returning from syscall, interrupt):
  kernel checks: pending signals? → yes
  kernel invokes signal handler (in process's context):
    - If default action: kernel handles (terminate, ignore, stop)
    - If custom handler: kernel sets up sigframe on process stack
      → process resumes at handler address
      → after handler returns: sigreturn() restores original execution context
```

**Important signal categories:**
```
Termination:    SIGTERM (15) — graceful shutdown request
                SIGKILL (9)  — immediate kill (cannot be caught/ignored)
                SIGQUIT (3)  — quit with core dump
                SIGINT (2)   — Ctrl+C
                SIGHUP (1)   — terminal hangup / config reload

Errors:         SIGSEGV (11) — segmentation fault (invalid memory access)
                SIGBUS (7)   — bus error (alignment issues, device error)
                SIGFPE (8)   — floating point exception
                SIGILL (4)   — illegal instruction

Process:        SIGCHLD (17) — child status changed
                SIGSTOP (19) — stop process (cannot be caught)
                SIGCONT (18) — continue stopped process

User-defined:   SIGUSR1 (10) — application-defined (log reopen, reload)
                SIGUSR2 (12) — application-defined
```

```bash
# Send signals:
kill -TERM <pid>      # Graceful shutdown
kill -HUP <pid>       # Reload config (nginx, syslog)
kill -USR1 <pid>      # App-specific (nginx: reopen logs)

# See pending/blocked signals for a process:
cat /proc/<pid>/status | grep -E "Sig(Pnd|Blk|Ign|Cgt)"
# SigPnd: pending (not yet delivered)
# SigBlk: blocked (masked)
# SigIgn: ignored
# SigCgt: caught (has custom handler)
```

### 🔍 Follow-up (Level 1)
**Q: What makes a function "async-signal-safe" and why does it matter?**

**A:**
Signal handlers can interrupt ANY point in program execution — including inside `malloc()`, `printf()`, or any library function.

If a signal handler calls `malloc()` while the main program was already inside `malloc()` (holding its internal lock): deadlock or heap corruption.

Async-signal-safe functions: those that can be called from a signal handler without risk. They either:
- Don't use global state
- Are atomic in their effect
- Use separate re-entrant code paths

POSIX-specified async-signal-safe functions: `write()`, `read()`, `open()`, `close()`, `kill()`, `_exit()`, `sigaction()`, basic memory operations.

NOT safe: `printf()`, `malloc()`, `free()`, `pthread_mutex_lock()`, `strerror()`.

**Production pattern — self-pipe trick:**
Signal handler writes 1 byte to a pipe FD. Main event loop (epoll/select) monitors the pipe's read end. On byte received → main loop handles the signal in normal execution context where all functions are safe.

```c
// In signal handler (safe):
write(signal_pipe[1], &sig, 1);

// In event loop (normal context, all functions safe):
if (epoll_event.fd == signal_pipe[0]) {
    read(signal_pipe[0], &sig, 1);
    handle_signal(sig);  // Can call any function here
}
```

---

## Q32. What is the difference between SIGTERM and SIGKILL?

### 🏭 Production Context
This is not academic — misusing SIGKILL in production causes:
- Data corruption (in-flight transactions not committed)
- Incomplete state writes (config files half-written)
- Port not released (connection in FIN_WAIT)
- Kubernetes pod stuck terminating (grace period exceeded)

### ⚙️ Inner Workings

**SIGTERM (15):**
- Delivered to process
- Process can: catch it, run cleanup (flush buffers, close connections, complete transactions), then exit
- Process can also: ignore it (bad practice but possible)
- Default action: terminate (if no custom handler)

**SIGKILL (9):**
- NOT delivered to the process's signal handlers
- Kernel handles it directly: marks task for death, no code runs in the process
- Process CANNOT catch, block, or ignore SIGKILL
- No cleanup code runs — buffers unflushed, temp files undeleted, connections hard-closed
- Only reliable way to kill a process
- Does NOT work on D-state (uninterruptible sleep) processes

```bash
# Graceful shutdown sequence:
kill -TERM <pid>               # Request graceful shutdown
sleep 30                       # Wait for cleanup
kill -0 <pid> && kill -KILL <pid>  # Force if still alive

# Kubernetes follows this exactly:
# 1. SIGTERM sent to PID 1
# 2. Wait terminationGracePeriodSeconds (default 30s)
# 3. SIGKILL if still running
```

### 🔍 Follow-up (Level 1)
**Q: A process is sent SIGKILL but doesn't die. What state must it be in?**

**A:**
D state (TASK_UNINTERRUPTIBLE) — uninterruptible sleep. The process is inside the kernel waiting for an I/O event that cannot be interrupted. SIGKILL is queued but cannot be delivered because the process is in kernel code that doesn't check for signals.

Common causes:
- Waiting for NFS I/O (NFS server down or unresponsive)
- Waiting for a local block device that has failed
- Waiting for a hung kernel module
- `ioctl()` on a hardware device that's not responding

```bash
# Identify D-state processes
ps aux | awk '$8 == "D"'
cat /proc/<pid>/wchan   # What kernel function is it sleeping in?
# e.g., "nfs_wait_atomic_killable" → NFS hang

# Resolution:
# For NFS: fix NFS server, or force-unmount
umount -l /nfs_mount   # Lazy unmount (detaches immediately, cleans up on last close)
umount -f /nfs_mount   # Force unmount

# If hardware device: may need reboot
```

### 🔥 Follow-up (Level 2)
**Q: You SIGKILL a container's PID 1. It dies. But a subprocess (now orphaned, reparented to init) is still running in the container's cgroup. What happens?**

**A:**
When the container runtime (containerd/CRI-O) kills PID 1 and cleans up the container:
1. The container's cgroup is marked for deletion
2. Any remaining processes in the cgroup receive SIGKILL from the cgroup cleanup mechanism
3. The cgroup is destroyed once all member processes are dead

BUT: there's a race window. If the orphaned process spawned children or opened network connections, those may briefly persist. The cgroup killer is reliable but not instantaneous.

In Kubernetes: `kubectl delete pod` triggers graceful shutdown. SIGTERM → grace period → SIGKILL PID 1. Remaining cgroup members also get SIGKILL via cgroup cleanup. No process leaks past cgroup lifetime.

---

## Q33. Why can't you kill a process in D state?

*(Covered in Q32's follow-up — fully addressed there.)*

**Additional detail:**

D state vs S state:
- S (TASK_INTERRUPTIBLE): sleeping but will wake on signal → kill works
- D (TASK_UNINTERRUPTIBLE): sleeping, ignores signals → kill queued but not delivered

```bash
cat /proc/<pid>/wchan    # Kernel wait channel
# "pipe_wait" → S state (interruptible pipe read)
# "nfs_wait"  → D state (uninterruptible NFS wait)

# Monitor D-state duration:
bpftrace -e 'tracepoint:sched:sched_switch /args->prev_state == 2/ {
    @dstate[args->prev_comm] = count();
}'
# 2 = TASK_UNINTERRUPTIBLE
```

---

## Q34. How do you terminate a process safely?

### 🏭 Production Context
Safe termination = graceful shutdown = application gets a chance to flush state, complete in-flight transactions, release resources, deregister from service discovery. Getting this wrong means user-facing errors during deploys.

### 🔧 Production Termination Sequence

```bash
# ── Step 1: SIGTERM (request graceful shutdown) ──
kill -TERM <pid>

# ── Step 2: Wait with timeout ──
TIMEOUT=30
while kill -0 <pid> 2>/dev/null && [ $TIMEOUT -gt 0 ]; do
    sleep 1
    ((TIMEOUT--))
done

# ── Step 3: Force kill if still alive ──
if kill -0 <pid> 2>/dev/null; then
    echo "Process didn't exit in 30s, sending SIGKILL"
    kill -KILL <pid>
fi

# ── Step 4: Verify dead ──
wait <pid> 2>/dev/null
echo "Exit code: $?"
```

**Application-side (what good graceful shutdown looks like):**
```python
import signal, sys

def graceful_shutdown(signum, frame):
    print("SIGTERM received, draining requests...")
    server.stop_accepting()          # No new connections
    server.wait_for_requests(30)     # Wait max 30s for in-flight requests
    db_pool.close_all()              # Close DB connections cleanly
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
```

**In Kubernetes:**
```yaml
spec:
  terminationGracePeriodSeconds: 60    # Default 30 — increase for slow drains
  containers:
  - lifecycle:
      preStop:                         # Runs before SIGTERM; gives load balancer
        exec:                          # time to drain traffic from pod
          command: ["sleep", "10"]
```

### 🔍 Follow-up (Level 1)
**Q: Why does Kubernetes send SIGTERM to PID 1 and not to the actual application process (which might be PID 7)?**

**A:**
Kubernetes (via kubelet → container runtime) sends SIGTERM to the container's PID 1. The design intent: PID 1 is responsible for managing its children. If PID 1 is a proper init (tini, dumb-init), it forwards SIGTERM to children.

But if PID 1 is a shell script that launches your app (common pattern: `CMD ["/entrypoint.sh"]`): the shell is PID 1, your app is PID 2. SIGTERM kills the shell. App (PID 2) becomes an orphan, reparented to containerd's shim, and eventually SIGKILL'd without cleanup.

Fix:
```dockerfile
# Bad:
CMD ["/bin/sh", "-c", "your-app"]  # Shell is PID 1, app is child

# Good option 1: exec form (no shell)
CMD ["your-app", "--arg"]  # your-app IS PID 1

# Good option 2: exec in shell script to replace shell with app
ENTRYPOINT ["tini", "--"]
CMD ["your-app"]  # tini is PID 1, forwards signals properly

# In entrypoint.sh, end with:
exec your-app "$@"   # exec replaces shell with app → app becomes PID 1
```

---

## Q59–Q61. VFS, ls -l Internals, Syscalls in File Listing

### Q59. What happens when you run `ls -l` internally?

### 🏭 Production Context
Understanding the `ls -l` syscall chain matters when: `ls` is slow on a directory (metadata reads), NFS directories time out during `ls`, or you're writing a high-performance directory scanner.

### ⚙️ Full Syscall Chain

```bash
strace ls -l /etc/ 2>&1 | head -40
```

Trace:
```
execve("/bin/ls", ["ls", "-l", "/etc/"], ...)       # Load ls binary
brk(NULL)                                            # Set up heap
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY)      # Load dynamic linker cache
mmap(...)                                            # Map shared libraries (libc, etc.)
openat(AT_FDCWD, "/etc/", O_RDONLY|O_DIRECTORY)     # Open /etc/ directory
getdents64(fd, buf, 32768)                           # Read directory entries (batch)
  # Returns: filename + inode number for each entry
  # Multiple calls until all entries read
lstat("/etc/passwd", &stat_buf)                      # Get metadata for each file
  # lstat used (not stat) to handle symlinks correctly
  # Returns: permissions, size, owner, timestamps
getpwuid(uid)                                        # UID → username lookup
getgrgid(gid)                                        # GID → group name lookup
  # These may trigger /etc/nsswitch.conf → LDAP/NIS calls → network!
write(1, output, len)                                # Write formatted output to stdout
close(fd)                                            # Close directory FD
```

**The slow `ls` mystery:** On NFS or LDAP-backed environments, `ls -l` calls `getpwuid()` for EVERY file — each may trigger an LDAP/NIS lookup. 1000 files × 10ms LDAP latency = 10 second `ls -l`. Fix: `ls -n` (numeric UIDs) or ensure nscd (name service cache daemon) is running.

### Q60. What syscalls are involved in file listing?

**Key syscalls:**
- `openat(AT_FDCWD, dir, O_RDONLY | O_DIRECTORY)` — open directory
- `getdents64(fd, buf, count)` — read directory entries (returns filenames + inodes in batches)
- `lstat(path, &stat)` — get file metadata per entry (used instead of stat to handle symlinks)
- `fstat(fd, &stat)` — stat an already-open FD
- `readlink(path, buf)` — for symlinks, read target

**`getdents64` is the critical one:**
```c
struct linux_dirent64 {
    ino64_t        d_ino;    // inode number
    off64_t        d_off;    // offset to next entry
    unsigned short d_reclen; // length of this record
    unsigned char  d_type;   // file type (DT_REG, DT_DIR, etc.)
    char           d_name[]; // filename (null-terminated)
};
```

Older `readdir()` uses `getdents()` (one entry at a time) — `getdents64()` batches them → much faster for large directories.

### Q61. What is VFS (Virtual File System)?

### ⚙️ Inner Workings

VFS is the kernel abstraction layer that provides a uniform interface for all filesystems. Applications call `open()`, `read()`, `write()` — VFS dispatches to the correct filesystem implementation.

```
Application: open("/proc/cpuinfo", O_RDONLY)
      ↓
VFS layer:
  - Parse path → find mount point
  - Identify filesystem type (procfs)
  - Call procfs's open() implementation
      ↓
procfs: generate content dynamically (no disk)

Application: open("/home/user/file.txt", O_RDONLY)
      ↓
VFS layer:
  - Parse path → find mount point
  - Identify filesystem (ext4)
  - Call ext4's open() implementation
      ↓
ext4: read from block device
```

**VFS objects:**
- `struct super_block` — represents a mounted filesystem
- `struct inode` — represents a file (in VFS terms)
- `struct dentry` — represents a path component (cached in dcache)
- `struct file` — represents an open file (per-FD object)

**Dcache (dentry cache):**
- Caches recent path resolutions (pathname → inode mappings)
- Avoids re-walking directory tree for frequently accessed paths
- ```bash
  cat /proc/sys/fs/dentry-state   # Current dcache state
  # dentry_num, unused, age_limit, ...
  ```

VFS filesystems include: ext4, xfs, btrfs, tmpfs, procfs (/proc), sysfs (/sys), devtmpfs (/dev), overlayfs (containers), nfs, cgroup, debugfs.

### 🔍 Follow-up (Level 1)
**Q: Why does `rm -rf /proc/1234` fail even though you're root?**

**A:**
procfs is a virtual filesystem — its "files" and "directories" are kernel data structures, not real filesystem objects. The kernel's procfs implementation deliberately returns `EPERM` or similar errors for operations that don't make sense (like deleting kernel data).

VFS calls procfs's `unlink()` implementation, which returns an error because:
- procfs entries are dynamically generated from `task_struct` data
- There's no inode to unlink — no physical storage
- Deleting `/proc/1234` would mean "kill process 1234," which has a different syscall (`kill()`)

```bash
rm -rf /proc/1234
# rm: cannot remove '/proc/1234': Is a directory
# Even with -rf, kernel-backed directories reject rmdir()
```

---

# 📋 Batch 4 Summary

| Q | Topic | Graded? |
|---|---|---|
| Q21 | File Descriptor | — |
| Q22 | FD Consumers | — |
| Q23 | FD Leak Debug | Operational ✓ (45/50) |
| Q24 | Inode | — |
| Q25 | Disk Full with Space Available | — |
| Q26 | Inode Exhaustion Debug | Operational ✓ |
| Q27 | Hard Link vs Soft Link | — |
| Q28 | Hard Links Can't Cross Filesystems | — |
| Q29 | Symlink Resolution | — |
| Q30 | IPC Mechanisms | — |
| Q31 | Signals | — |
| Q32 | SIGTERM vs SIGKILL | — |
| Q33 | D-State Process | — |
| Q34 | Safe Termination | Operational ✓ |
| Q59 | ls -l Internals | — |
| Q60 | File Listing Syscalls | — |
| Q61 | VFS | — |

**Next: Batch 5 — Performance Debugging & Kernel Internals**
Topics: High CPU/Memory/IO debugging, load average, context switching, kernel/user space, syscall overhead, slow system RCA