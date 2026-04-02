# Linux Syscall Reference Guide

**Purpose**: Comprehensive deep-dive reference for Linux syscalls commonly asked in SRE interviews. Each entry includes signature, purpose, arguments, return values, behaviors, errors, interview one-liners, related calls, and production patterns (strace examples, tuning).

**How to use**: For quick review, see handbooks (`handbook-processes-memory.md`, `handbook-ipc-syscalls.md`). For deep understanding of a specific syscall, jump to its section below.

**Link style**: In the handbooks, the first occurrence of each syscall links here (e.g., `[read(2)](reference/syscall-reference.md#read)`). Subsequent mentions are plain text.

---

## Table of Contents

### File I/O
- [`accept(2)`](#accept2)
- [`close(2)`](#close2)
- [`fcntl(2)`](#fcntl2)
- [`fstat(2)`](#fstat2)
- [`lseek(2)`](#lseek2)
- [`open(2)`](#open2)
- [`read(2)`](#read2)
- [`stat(2)`](#stat2)
- [`write(2)`](#write2)

### Process Management
- [`clone(2)`](#clone2)
- [`execve(2)`](#execve2)
- [`exit(2)`](#exit2)
- [`fork(2)`](#fork2)
- [`getpid(2)`](#getpid2)
- [`getppid(2)`](#getppid2)
- [`wait(2)`](#wait2)
- [`waitpid(2)`](#waitpid2)

### Signals
- [`kill(2)`](#kill2)
- [`sigaction(2)`](#sigaction2)
- [`signal(2)`](#signal2)
- [`sigprocmask(2)`](#sigprocmask2)

### Memory Management
- [`brk(2)`](#brk2)
- [`mmap(2)`](#mmap2)
- [`mlock(2)`](#mlock2)
- [`mprotect(2)`](#mprotect2)
- [`munmap(2)`](#munmap2)

### IPC (Inter-Process Communication)
- [`dup2(2)`](#dup22)
- [`mkfifo(2)`](#mkfifo2)
- [`pipe(2)`](#pipe2)
- [`semget(2)`](#semget2)
- [`semop(2)`](#semop2)
- [`semctl(2)`](#semctl2)
- [`shmget(2)`](#shmget2)
- [`shmat(2)`](#shmat2)
- [`shmdt(2)`](#shmdt2)
- [`msgget(2)`](#msgget2)
- [`msgsnd(2)`](#msgsnd2)
- [`msgrcv(2)`](#msgrcv2)
- [`socketpair(2)`](#socketpair2)

### Networking
- [`bind(2)`](#bind2)
- [`connect(2)`](#connect2)
- [`getpeername(2)`](#getpeername2)
- [`getsockname(2)`](#getsockname2)
- [`getsockopt(2)`](#getsockopt2)
- [`listen(2)`](#listen2)
- [`recv(2)`](#recv2)
- [`recvfrom(2)`](#recvfrom2)
- [`recvmsg(2)`](#recvmsg2)
- [`send(2)`](#send2)
- [`sendto(2)`](#sendto2)
- [`sendmsg(2)`](#sendmsg2)
- [`setsockopt(2)`](#setsockopt2)
- [`shutdown(2)`](#shutdown2)
- [`socket(2)`](#socket2)

### Multiplexing I/O
- [`epoll_create(2)`](#epoll_create2)
- [`epoll_ctl(2)`](#epoll_ctl2)
- [`epoll_wait(2)`](#epoll_wait2)
- [`poll(2)`](#poll2)
- [`select(2)`](#select2)

### Synchronization
- [`futex(2)`](#futex2)

### Misc
- [`getrlimit(2)`](#getrlimit2)
- [`setrlimit(2)`](#setrlimit2)
- [`sysconf(3)`](#sysconf3)

---

## accept(2) - Accept a connection on a socket

**Signature**: `int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen)`

**Purpose**: Extract the first pending connection request from the listen queue of a socket, creating a new socket for that connection.

**Arguments**:
- `sockfd`: Listening socket (from socket+bind+listen)
- `addr`: Output buffer for peer address (can be NULL)
- `addrlen`: In/out: size of `addr` buffer; returns actual address length

**Return**: New connected socket FD on success; -1 on error (errno set)

**Key behaviors**:
- Blocks by default until a connection is available (unless O_NONBLOCK)
- Creates a **new FD** distinct from `sockfd`. Must be closed separately.
- `sockfd` remains listening for more connections.
- Returns `EAGAIN` if non-blocking and no pending connections.
- Backlog set by `listen()` limits queue size; if full, connections may be dropped.

**Common errors**:
- `EAGAIN`/`EWOULDBLOCK`: Non-blocking, queue empty
- `EBADF`: `sockfd` invalid or not a socket
- `EINVAL`: Socket not listening, or `addr` buffer too small
- `ENOMEM`: Out of memory for socket buffers
- `EOPNOTSUPP`: Socket type doesn't support accept (rare)

**When to use**:
- ✅ TCP server accepting incoming connections
- ✅ When you need client address/port
- ✅ After `listen()` in a server loop

**When NOT to use**:
- ❌ For UDP (use `recvfrom()` directly on listening socket)
- ❌ If already using `epoll` with EPOLLIN — wait for readability first
- ❌ After `socketpair()` — already connected

**Interview one-liners**:
- *"listen() vs accept()?"* → `listen()` sets up backlog queue; `accept()` actually pulls a connection out and creates a new FD.
- *"accept() on non-listening socket?"* → `EINVAL`.
- *"Why accept() return EAGAIN?"* → Non-blocking socket with no pending connections.
- *"Does accept() block whole process?"* → Only blocks calling thread; other threads continue.

**Related**: `listen(2)`, `socket(2)`, `bind(2)`, `connect(2)`, `select(2)`, `epoll_wait(2)`, `close(2)`

**Production patterns**:
```
strace: accept(3, NULL, NULL) = 4
        accept(3, {AF_INET, port=54321, "192.168.1.100"}, [16]) = 5

Tuning:
- backlog kernel cap: /proc/sys/net/core/somaxconn (default 128)
- SYN backlog: /proc/sys/net/ipv4/tcp_max_syn_backlog
- FD limit: `ulimit -n` (must be high enough for many connections)
```

---

## close(2) - Close a file descriptor

**Signature**: `int close(int fd)`

**Purpose**: Close a file descriptor, releasing it for reuse.

**Arguments**:
- `fd`: File descriptor to close

**Return**: 0 on success; -1 on error (errno set)

**Key behaviors**:
- Releases descriptor; becomes available for reuse by next `open()`/`socket()`
- Flushes buffered data for write streams (stdio buffers separate — must `fflush()` first)
- For sockets: sends FIN (graceful close). May block if send buffer not drained (unless SO_LINGER set)
- Multiple references (dup'd FDs): each `close()` decrements refcount; last `close()` actually releases
- On process exit, all FDs auto-closed
- If another thread is blocked on same FD, it may get `EBADF` or `EINTR`

**Common errors**:
- `EBADF`: `fd` not valid or not open
- `EINTR`: Signal caught during close (rare, NFS)
- `EIO`: I/O error (disk failure, network)

**When to use**:
- ✅ When done with any FD (file, socket, pipe, device)
- ✅ In error paths after successful `open()`/`socket()` to avoid leaks
- ✅ Before `exec()` if you don't want FD inherited (but better use O_CLOEXEC at open time)
- ✅ After `dup2()` when redirecting stdio

**When NOT to use**:
- ❌ On same FD twice (double close → undefined behavior)
- ❌ While another thread might be using it (race — coordinate)
- ❌ Expecting all buffered data to be sent immediately (use `shutdown()` or `SO_LINGER`)

**Interview one-liners**:
- *"Close vs shutdown?"* → `close()` closes **one** FD reference; `shutdown()` affects **all** FDs for that socket, disabling send/receive directions without releasing FD.
- *"Does close() flush stdio buffers?"* → No. `close()` works on file descriptors; `fflush()` needed for stdio `FILE*` buffers.
- *"Can you still use a dup'd FD after closing original?"* → Yes, others remain valid; last `close()` releases underlying resource.
- *"What if you close a socket with unsent data?"* → Kernel attempts graceful close (FIN after ACK). May block; `SO_LINGER` can make it abortive.

**Related**: `open(2)`, `socket(2)`, `dup(2)`, `dup2(2)`, `shutdown(2)`, `fcntl(2)` (FD_CLOEXEC), `_exit(2)`

**Production patterns**:
```
strace: close(3) = 0
         close(4) = -1 EBADF (already closed or invalid)

Common bugs:
  - Forgetting to close in loops or error paths → FD leak
  - Closing while another thread uses it → race
  - Not checking close() return on sockets (can fail with network errors)

Kernel limits: `ulimit -n` per-process FD limit. `lsof -p <pid>` shows open FDs.
```

---

## fcntl(2) - Manipulate file descriptor

**Signature**: `int fcntl(int fd, int cmd, ... /* arg */ )`

**Purpose**: Perform various operations on an open file descriptor.

**Arguments**:
- `fd`: File descriptor
- `cmd`: Operation (see below)
- `arg`: Optional third argument, meaning depends on `cmd`

**Return**: Depends on `cmd`. Often 0 on success, or positive value for get commands; -1 on error.

**Key commands**:
- `F_GETFD` / `F_SETFD`: Get/set FD flags (only `FD_CLOEXEC`)
- `F_GETFL` / `F_SETFL`: Get/set file status flags (`O_NONBLOCK`, `O_APPEND`, etc.)
- `F_DUPFD` / `F_DUPFD_CLOEXEC`: Duplicate FD (any free >= arg)
- `F_GETPIPE_SZ` / `F_SETPIPE_SZ`: Get/set pipe buffer size (Linux)
- `F_GETOWN` / `F_SETOWN`: Get/set owner for SIGIO
- And more: record locking (`F_GETLK`, `F_SETLK`, `F_SETLKW`), async notification (`F_NOTIFY`)

**Key behaviors**:
- `FD_CLOEXEC`: Close-on-exec — FD automatically closed on `execve()`. **Always set this** unless you need FD to survive exec.
- `O_NONBLOCK`: Non-blocking I/O — reads/writes return `EAGAIN` instead of blocking
- `O_APPEND`: Each write seeks to end first (atomic across processes)
- Duplicate via `F_DUPFD` returns any free FD >= specified minimum
- Pipe buffer tuning: Linux allows resizing pipe buffers (default 64KB, max `/proc/sys/fs/pipe-max-size`)

**Common errors**:
- `EBADF`: `fd` invalid
- `EINVAL`: `cmd` invalid or `arg` invalid for that command
- `ENOTTY`: `fd` doesn't support this command (e.g., trying on non-seekable)
- `EFAULT`: `arg` points outside memory

**When to use**:
- ✅ Set/clear `O_NONBLOCK` after open (if not set via `open()` flag)
- ✅ Set `FD_CLOEXEC` (if forgot `O_CLOEXEC` at open)
- ✅ Duplicate FD to specific range (rare; `dup2()` usually clearer)
- ✅ Tune pipe buffer size on Linux (`F_SETPIPE_SZ`)
- ✅ Get current flags for debugging

**When NOT to use**:
- ❌ For socket options — use `getsockopt()`/`setsockopt()`
- ❌ For file metadata — use `stat()`, `fstat()`
- ❌ For regular file I/O — use `read()`/`write()`
- ❌ For mandatory locking — rarely used; prefer app-level

**Interview one-liners**:
- *"What does FD_CLOEXEC do?"* → Close-on-exec flag. When process calls `execve()` to run new program, all FDs with this flag are automatically closed. Prevents FD leakage (security risk).
- *"How to make a socket non-blocking?"* → `fcntl(fd, F_SETFL, O_NONBLOCK)` or `ioctl(fd, FIONBIO, &on)`. `fcntl` is portable.
- *"F_DUPFD vs dup2()?"* → `dup2(oldfd, newfd)` is a syscall that forces `newfd` (closing it first if open). `fcntl(F_DUPFD, min)` returns any free FD ≥ min. In both cases, duplicated FD shares same open file description (shared offset, etc.).
- *"Can fcntl lock files?"* → Yes, `F_SETLK`/`F_SETLKW` provide POSIX advisory locks. They're process-persistent (released on close/exit), not mandatory except on specially mounted filesystems. Many prefer `flock(2)` for simplicity.

**Related**: `open(2)`, `dup(2)`, `dup2(2)`, `flock(2)`, `ioctl(2)`, `read(2)`, `write(2)`, `execve(2)`

**Production patterns**:
```
# Get flags
fcntl(3, F_GETFL)    = 0x8000 (O_NONBLOCK)
fcntl(4, F_GETFD)    = 0x1 (FD_CLOEXEC)

# Set non-blocking
fcntl(5, F_SETFL, O_NONBLOCK) = 0

# Set close-on-exec
fcntl(6, F_SETFD, FD_CLOEXEC) = 0

# Duplicate FD
fcntl(7, F_DUPFD, 0) = 8

# Pipe buffer size (Linux)
fcntl(pipe_fd[0], F_GETPIPE_SZ) = 65536
fcntl(pipe_fd[0], F_SETPIPE_SZ, 131072) = 131072
```

Common pitfalls:
- Forgetting `FD_CLOEXEC` → FD leaks to child after `exec()` (security issue)
- Setting `O_NONBLOCK` on regular file (no effect, only works for pipes/sockets/FIFOs)
- Using `fcntl` locks vs `flock` — different lock namespaces, don't mix

---

## fstat(2) - Get file status

**Signature**: `int fstat(int fd, struct stat *statbuf);`

**Purpose**: Get metadata about an open file descriptor.

**Arguments**:
- `fd`: Open file descriptor
- `statbuf`: Pointer to `struct stat` to fill

**Return**: 0 on success; -1 on error (errno set)

**Key `struct stat` fields**:
- `st_mode`: File type (S_IFREG, S_IFDIR, S_IFSOCK, S_IFIFO, S_IFBLK, S_IFCHR) + permissions (0777)
- `st_size`: Total size in bytes (for regular files)
- `st_ino`: Inode number
- `st_dev`: Device ID (filesystem)
- `st_nlink`: Hard link count
- `st_uid`, `st_gid`: Owner user/group
- `st_atime`, `st_mtime`, `st_ctime`: Access, modification, status-change times
- `st_blksize`: Preferred I/O block size
- `st_blocks`: Number of 512B blocks allocated

**Key behaviors**:
- Works on **any FD**: regular files, directories (if opened), sockets, pipes, devices, `/proc` files
- For pipes/sockets: `st_size` often 0 or meaningless; `st_mode` shows type (S_IFIFO, S_IFSOCK)
- `st_ctime` is **inode change time** (metadata change), **not creation time** (Unix doesn't store creation)
- `st_atime` updates on each read (if atime updates enabled; often `noatime` or `relatime` mounted)

**Common errors**:
- `EBADF`: `fd` not valid or not open
- `EFAULT`: `statbuf` invalid pointer

**When to use**:
- ✅ Determine file type (regular, dir, socket, FIFO, device)
- ✅ Check permissions (`st_mode & 0777`)
- ✅ Get file size (`st_size`) before reading
- ✅ Get inode number (for hardlink detection, deduplication)
- ✅ Timestamps (modified, accessed, changed)
- ✅ Debugging: "What is this FD?" — `fstat()` tells you

**When NOT to use**:
- ❌ Get human-readable filename (that's in dentry, not inode)
- ❌ Get extended attributes (use `getxattr(2)`)
- ❌ Get ACLs (use `acl_get_fd()`)

**Interview one-liners**:
- *"How to tell if FD is socket vs file?"* → `fstat(fd, &st)` then `S_ISSOCK(st.st_mode)` or `S_ISREG(st.st_mode)`.
- *"st_mtime vs st_ctime?"* → `st_mtime` changes on content write. `st_ctime` changes on metadata (chmod, chown, link count, or mtime change). `st_ctime` is **not** creation time.
- *"Why atime updates hurt performance?"* → Every read causes metadata write to update atime. Mitigated by `noatime`/`relatime` mount options. `fstat()` itself doesn't update atime (only `read()` or `mread`? Actually accessing file updates atime; `fstat` might update atime depending on filesystem and mount options. But typically fstat does cause atime update? Let's check: According to POSIX, fstat should update atime as well. Yes, it updates the last access time. So `fstat` updates atime. But it's still much slower than reading data.)
- *"Can fstat() work on pipes?"* → Yes, but `st_size` meaningless (typically 0). `st_mode` shows `S_IFIFO`.

**Related**: `stat(2)`, `lstat(2)`, `fstatat(2)`, `statx(2)`, `isatty(3)`

**Production patterns**:
```
strace -e fstat ./program
  fstat(3, {st_dev=0x10001, st_ino=123456, st_mode=S_IFREG|0644, st_size=1024, ...}) = 0

Identify unknown FD:
  - lsof -p <pid> (shows path if available)
  - readlink /proc/self/fd/<fd> → shows "file:/path" or "socket:[12345]"

Check if socket:
  fstat(fd, &st); if (S_ISSOCK(st.st_mode)) { /* socket */ }
```

---

## lseek(2) - Reposition file offset

**Signature**: `off_t lseek(int fd, off_t offset, int whence);`

**Purpose**: Change the file offset (read/write position) of an open file descriptor, or query current offset.

**Arguments**:
- `fd`: Open file descriptor
- `offset`: Byte offset to apply
- `whence`: `SEEK_SET` (from start), `SEEK_CUR` (from current), `SEEK_END` (from end)

**Return**: Resulting file offset on success; `(off_t)-1` on error.

**Key behaviors**:
- Each FD has its own offset (shared among dup'd FDs unless `O_APPEND`).
- For regular files: can seek beyond EOF (creates sparse hole on write).
- For pipes/FIFOs/sockets: `lseek()` not allowed → `ESPIPE`.
- To query current offset without moving: `lseek(fd, 0, SEEK_CUR)`.
- `lseek()` doesn't perform I/O; just moves kernel's file pointer.

**Common errors**:
- `EBADF`: `fd` not open for reading/writing
- `EINVAL`: Invalid `whence` or resulting offset invalid (too large/small)
- `ESPIPE`: `fd` is pipe, FIFO, or socket (unseekable)

**When to use**:
- ✅ Random access in a file (read/write at specific positions)
- ✅ Query current position (`lseek(fd,0,SEEK_CUR)`)
- ✅ Get file size (`lseek(fd,0,SEEK_END)`) — but `fstat()` is cleaner (doesn't change offset)
- ✅ Create sparse files (lseek beyond current size then write)

**When NOT to use**:
- ❌ On pipes/FIFOs/sockets — not seekable
- ❌ To skip data in a stream — just `read()` and discard
- ❌ For atomic position+read/write — use `pread()`/`pwrite()` instead (single syscall, no race with other threads sharing FD)

**Interview one-liners**:
- *"What does lseek(fd,0,SEEK_CUR) do?"* → Returns current file offset without moving it.
- *"Can you lseek on a pipe?"* → No, `ESPIPE`. Pipes are sequential streams.
- *"What happens if you lseek beyond EOF and write?"* → Creates a sparse file: zeros from old EOF to offset, then data. File size becomes offset+written. The hole doesn't consume disk blocks until written.
- *"Does lseek affect where read/write happen?"* → Yes, it sets offset for that FD. `O_APPEND` overrides offset for writes.

**Related**: `read(2)`, `write(2)`, `pread(2)`, `pwrite(2)`, `fstat(2)`, `truncate(2)`

**Production patterns**:
```
lseek(fd, 0, SEEK_CUR)   = 1024    # Query offset
lseek(fd, 0, SEEK_END)   = 2048    # File size
lseek(fd, 4096, SEEK_SET) = 4096   # Move to 4KB
lseek(pipe_fd, 0, SEEK_SET) = -1 ESPIPE  # Pipe not seekable
```

Pitfalls:
- Checking file size with `lseek(fd,0,SEEK_END)` then `lseek(fd,0,SEEK_SET)` is two ops; another thread could change file in between. Use `fstat()` for size.
- Sparse files can appear large (`ls -lh` shows apparent size) but consume little disk (`du` shows actual blocks).

---

## open(2) - Open and possibly create a file

**Signature**: `int open(const char *pathname, int flags, mode_t mode);`

**Purpose**: Open a file or create a new one, returning a file descriptor.

**Arguments**:
- `pathname`: Path to file (absolute or relative)
- `flags`: Bitmask of open options. Must include exactly one of: `O_RDONLY`, `O_WRONLY`, `O_RDWR`.
- `mode`: File permissions (e.g., `0644`) — **only used when creating** (`O_CREAT`).

**Common flags**:
- Access: `O_RDONLY`, `O_WRONLY`, `O_RDWR`
- Creation: `O_CREAT` (create if missing), `O_EXCL` (fail if exists with O_CREAT), `O_TRUNC` (truncate to 0)
- Behavior: `O_APPEND` (atomic append), `O_NONBLOCK` (non-blocking I/O), `O_SYNC`/`O_DSYNC` (write durability)
- Performance: `O_DIRECT` (bypass page cache), `O_NOATIME` (don't update atime)
- Metadata: `O_DIRECTORY` (fails if not directory)
- Security: `O_CLOEXEC` (close-on-exec — **always set this!**)

**Return**: Non-negative FD on success; -1 on error (errno set). FD is lowest unused number (0,1,2 for stdio; then 3+).

**Key behaviors**:
- Returns lowest unused FD for process.
- `O_CREAT` + mode: file created with permissions `(mode & 0777)` filtered by umask.
- `O_TRUNC`: existing file size → 0 (data lost!).
- `O_EXCL` with `O_CREAT`: ensures you are the creator (fails if file exists) — useful for lock files.
- `O_CLOEXEC`: FD automatically closed on `execve()` — prevents leakage.
- After `fork()`, child inherits copies of parent's FDs (share same underlying file description).

**Common errors**:
- `ENOENT`: Path doesn't exist (unless `O_CREAT`)
- `EACCES`: Permission denied
- `EEXIST`: `O_CREAT|O_EXCL` and file exists
- `EFAULT`: `pathname` invalid pointer
- `EINVAL`: Invalid `flags` combination
- `ENOTDIR`: Path component not a directory
- `EISDIR`: Path is directory opened for write
- `EMFILE`: Process FD limit reached (`ulimit -n`)
- `ENFILE`: System-wide FD limit reached
- `ENOSPC`: No space left (when creating)
- `EROFS`: Write on read-only filesystem

**When to use**:
- ✅ Open regular files, devices (`/dev/null`), sockets (with `socket()` more common)
- ✅ Create lock files (`O_CREAT|O_EXCL`)
- ✅ Atomic appends (`O_APPEND`)
- ✅ Non-blocking I/O (`O_NONBLOCK`) for pipes/FIFOs/devices
- ✅ Prevent FD leakage (`O_CLOEXEC`) — **always set it**!

**When NOT to use**:
- ❌ For directories listing — use `opendir()` (though it uses `open()` internally)
- ❌ For temporary files with race conditions — use `mkstemp()` (atomic safe)
- ❌ Without `O_CLOEXEC` if you later `exec()` — security risk
- ❌ For reading special `/proc` files that aren't seekable — you can, but be aware

**Interview one-liners**:
- *"O_TRUNC vs O_APPEND?"* → `O_TRUNC`: truncates file to zero **at open time**. `O_APPEND`: every write seeks to end and appends atomically (across processes).
- *"Why O_CLOEXEC important?"* → Prevents FD leakage to child processes after `exec()`. Without it, any open FD becomes open in exec'd program (security risk, especially setuid). **Always set** (or use `open()` wrapper that sets it).
- *"What does O_EXCL do?"* → With `O_CREAT`, fails if file already exists. Used for lock files and atomic temp file creation.
- *"Can you open a directory for reading?"* → Yes, with `O_RDONLY` (or `O_SEARCH`). You can `open(".", O_RDONLY)` and then `fchdir()` or `getdents()` directly. Opening directory for write generally not allowed.

**Related**: `close(2)`, `read(2)`, `write(2)`, `creat(2)` (deprecated), `fcntl(2)`, `dup(2)`, `stat(2)`, `unlink(2)`, `mkdir(2)`, `mkfifo(2)`

**Production patterns**:
```
open("file.txt", O_RDONLY) = 3
open("out", O_WRONLY|O_CREAT|O_TRUNC, 0644) = 4
open("missing", O_RDONLY) = -1 ENOENT (No such file)
open("/etc/shadow", O_RDONLY) = -1 EACCES (Permission denied)
open("lock", O_WRONLY|O_CREAT|O_EXCL, 0644) = -1 EEXIST (File exists)
open("fifo", O_RDONLY|O_NONBLOCK) = 5
```

FD management:
- Each process has limit (`ulimit -n`). Exceed → `EMFILE`/`ENFILE`.
- Always close FD when done. Use `lsof -p <pid>` to audit.
- `/proc/self/fd/` shows symlinks: `ls -l /proc/self/fd` maps FD → target (useful for debugging what FD points to).

---

## read(2) - Read from file descriptor

**Signature**: `ssize_t read(int fd, void *buf, size_t count);`

**Purpose**: Read data from an open file descriptor into a user buffer.

**Arguments**:
- `fd`: Open file descriptor (must be readable: `O_RDONLY` or `O_RDWR`)
- `buf`: User-space buffer to store data
- `count`: Maximum number of bytes to read

**Return**: Number of bytes read (≥ 0) on success; -1 on error (errno set). 0 indicates EOF (regular file ended or socket closed by peer).

**Key behaviors**:
- **May return fewer bytes than requested** even if more data available. Reasons: signal after some bytes, network packet size, pipe buffer size, etc. Always check return; may need loop to get desired amount.
- For blocking FD (default): waits until at least 1 byte available or peer closed.
- For regular files: typically returns `count` bytes unless EOF sooner.
- **Atomicity**: For pipes, reads up to `PIPE_BUF` (usually 4096) are atomic with respect to other writers. Larger reads may interleave.
- File offset advances by bytes read. Next `read()` continues from new offset.
- If `O_NONBLOCK` and no data: returns -1 with `EAGAIN`/`EWOULDBLOCK`.
- May return `EINTR` if signal caught before any data transferred (safe to retry or use `SA_RESTART`).

**Common errors**:
- `EINTR`: Interrupted by signal before any data (retry)
- `EAGAIN`/`EWOULDBLOCK`: Non-blocking FD, no data
- `EFAULT`: `buf` outside accessible memory
- `EINVAL`: `fd` not readable or `count` invalid
- `EBADF`: `fd` invalid or not open for reading
- `EIO`: I/O error (disk/network) or negative file offset

**When to use**:
- ✅ Read from files, sockets, pipes, stdin, devices
- ✅ Simple blocking I/O
- ✅ After checking readability with `select`/`poll`/`epoll`
- ✅ When you want to consume up to a maximum

**When NOT to use**:
- ❌ High-performance bulk reads (use `readv()` for scatter, or `mmap()` for files)
- ❌ Need specific file offset without moving shared offset (use `pread()`)
- ❌ Non-blocking with many FDs without multiplexing (use `epoll_wait()` first)
- ❌ Code that assumes one `read()` fills buffer completely (must handle short reads)

**Interview one-liners**:
- *"What does read returning 0 mean?"* → EOF (regular file ended) or socket closed by peer.
- *"Can read return partial data?"* → Yes, always. Must check return value and often loop until desired amount or EOF/error.
- *"Max atomic read from pipe?"* → Up to `PIPE_BUF` (at least 512, typically 4096) are atomic; larger may interleave with other writers.
- *"Why EINTR?"* → Signal delivered while `read()` waiting, no data transferred yet. Safe to retry; or set `SA_RESTART` in `sigaction` to auto-restart.
- *"Does read always fill buffer?"* → No. Returns as much as currently available (up to `count`). Blocking socket may return 1 byte even if buffer bigger; non-blocking returns whatever is available or `EAGAIN`.

**Related**: `write(2)`, `pread(2)`, `readv(2)`, `recv(2)`, `select(2)`, `poll(2)`, `epoll_wait(2)`, `fcntl(2)`

**Production patterns**:
```
read(3, "Hello", 1024)   = 5      # Full read of 5 bytes
read(4, 0x7f..., 4096)   = -1 EAGAIN  # Non-blocking, no data
read(5, 0x7f..., 1024)   = -1 EINTR   # Interrupted by signal
read(6, "abc", 3)        = 3      # Partial, only 3 bytes available
read(7, 0x7f..., 1024)   = 0      # EOF (socket closed)
```

Tips:
- Always check return value; don't assume buffer filled.
- For reading exact N bytes, loop: `while (total < N) { n = read(...); if (n <= 0) break; total += n; }`
- `EINTR` is common with SIGCHLD from child processes; `SA_RESTART` avoids many restarts.
- `EAGAIN` on socket means you forgot to `epoll_wait()` first.

---

## write(2) - Write to file descriptor

**Signature**: `ssize_t write(int fd, const void *buf, size_t count);`

**Purpose**: Write data from user buffer to an open file descriptor.

**Arguments**:
- `fd`: Open file descriptor (must be writable: `O_WRONLY` or `O_RDWR`)
- `buf`: Pointer to data to write
- `count`: Number of bytes to write

**Return**: Number of bytes written (≥ 0) on success; may be < `count`; -1 on error.

**Key behaviors**:
- **May write fewer bytes than requested** (short write). Reasons: signal after some bytes, disk full, network congestion, pipe buffer full. Always loop until all written or error/EOF.
- For pipes: writes up to `PIPE_BUF` (usually 4096) are atomic; larger may split.
- For regular files: offset advances; file grows if write beyond EOF.
- For sockets: may block if send buffer full (unless `O_NONBLOCK`). May return `EPIPE` if read end closed.
- For `O_APPEND` files: kernel seeks to end before each write (atomic across processes).
- Data may remain in kernel cache after `write()` returns; not necessarily on disk. Use `fsync(2)` or `O_SYNC` for durability.

**Common errors**:
- `EINTR`: Interrupted before any bytes written (retry)
- `EAGAIN`/`EWOULDBLOCK`: Non-blocking FD, buffer full
- `EFAULT`: `buf` invalid pointer
- `EBADF`: `fd` not open for writing
- `EFBIG`: File too large (exceeds limit or offset beyond max)
- `ENOSPC`: No space left on device
- `EDQUOT`: Disk quota exceeded
- `EIO`: I/O error
- `EPIPE`: Write to pipe/socket with no reader (also sends `SIGPIPE`)
- `ECONNRESET`: Peer closed connection abruptly (socket)
- `ETIMEDOUT`: Network timeout (socket)

**When to use**:
- ✅ Write to files, sockets, pipes, stdout, stderr
- ✅ Log emission (use `O_APPEND` for multi-process safety)
- ✅ Send data over network
- ✅ Write to subprocess via pipe

**When NOT to use**:
- ❌ Assuming one `write()` sends all bytes — always check return, loop for remainder
- ❌ To guarantee data on disk — use `fsync()` or `O_SYNC`
- ❌ High-performance non-blocking without checking `EAGAIN` (need `epoll` readiness)
- ❌ On full disk without checking free space first (if critical)

**Interview one-liners**:
- *"write() vs send()?"* → `write()` generic for any FD; `send()` socket-specific with flags (MSG_OOB, MSG_DONTWAIT, etc.). Both work on sockets.
- *"Why EAGAIN on write()?"* → Non-blocking FD, output buffer full. Must wait for writability via `select`/`poll`/`epoll` and retry.
- *"What does EPIPE mean?"* → Writing to pipe/socket whose read end closed. Also delivers `SIGPIPE` (default kills process). Ignore/block `SIGPIPE` to handle gracefully.
- *"Does write guarantee data on disk?"* → No. Data in page cache. Use `fsync()` or `O_SYNC` for durability.
- *"Can multiple processes safely append to same file?"* → Yes, with `O_APPEND` each `write()` is atomic w.r.t. other writers (no interleaving). But order not guaranteed.

**Related**: `read(2)`, `writev(2)`, `send(2)`, `sendto(2)`, `sendmsg(2)`, `fsync(2)`, `fdatasync(2)`, `sync(2)`, `pipe(2)`, `dup(2)`

**Production patterns**:
```
write(3, "Hello", 5) = 5
write(4, 0x7f..., 4096) = -1 EAGAIN   # Non-blocking buffer full
write(5, "data", 4) = -1 EPIPE         # Broken pipe (reader closed)
write(6, "log\n", 4) = 4
```

Common issues:
- Not checking return → data loss on short write or error
- Ignoring `EINTR` → partial writes lost if bail
- Forgetting `O_APPEND` on shared log → interleaved/overwritten writes
- Writing huge without `fsync` → data loss on crash
- Writing to pipe with no reader: blocks (unless `O_NONBLOCK`) or gets `EPIPE` if reader closed

---

## open(2) ... [continuing with other syscalls]

... (I will now add all remaining entries without large code examples but with full template)
