# PERRIO Session Context — Day 1: Process Management & Memory

**Date**: 2025-04-01
**Topic**: Processes, Signals, Memory Management
**Total Hours**: 6
**Overall Score**: 79/100 (Baseline) → 74/100 (Retrieval) → 7.5/10 (Interleaving)

---

## Baseline Assessment — Raw Answers

### Q1. Process creation flow
> "when we run a command , so the first thing happen is that the current root process is duplicated with new id in general systemd or can use the same id when used exec then the process code is replaced with the code we want to run basically forking , once the command are there in the process, the process gets allocation of memory space , address space and kernel schedule the process, for example if process is realtime , memory is allocated, if process is low priority it is added to queue. Once the process execution starts , if process is something like write to disk , since disk is hardware and execution user can't access it, a system call is made like read write and other based on the process, then their are certain middle layers like vfs and drivers also called as modules which perform the operation and return the response to process. It is possble that process creates multiple thread to achieve parallelism it can do that by creating new thread under the same process and in this they can share the memory and address. sometime it is also possible that a child process is created as per the requirement , in this isolation is maintained for each process and the child process sends exit() signal to parent process using SIGCHLD() and then main process goes to wait() and once the main process receives the final response from the child process it completes the operation and gracefully close child and main processes. the child process can be viewed using top and /proc/pid/"

**Score**: 8.5/10
**Corrections**:
- Shell (bash/zsh) calls fork(), not systemd (unless PID 1)
- Child calls exit(), parent calls wait() (reversed in original)
- Otherwise complete mental model

### Q2. Zombie vs Orphan
> "Zombie process are process when child process performed or send exit() signal but the communication between the parrent and child process is broken due to any reason. To make sure it close we can either wait or we can send SIGCHLD command mannualy as kill can't work on it as its already exit() . WHILE orphan process are those whose parent process is completed and now exited. an. example can be like you run sleep 30 and then sleep 60 , now main process sleep is already exited and left the child process orphan in such cases init take care of orphan process and exit them gracefully"

**Score**: 8/10
**Corrections**:
- Zombie = child exited, parent never called wait() (not "broken communication")
- kill doesn't work because process is already dead
- Orphan example better: background job in shell script where script exits

### Q3. Process table full
> "this suggest that the process creation is failing. Not sure why but can be because the bash is running as pid 1"

**Score**: 6/10
**Corrections**:
- Real cause: ulimit -u hit OR kernel.pid_max exhausted
- "bash as PID 1" is not the cause
- Immediate moves: count processes, find zombies, kill spawers, raise ulimit

### Q4. kill -9 vs kill -15
> "kill -9 is force close and can. leave your process in incomplete state while kill -15 is graceful closure and can wait for process completion"

**Score**: 9/10
**Add**: SIGTERM can be caught/ignored; kill -9 bypasses all handlers

### Q5. App crashing with 14/16 GB used, no swap
> "This is happening because the meminfo doesn't have any inactive or active swap memory which is leading to consumption of 14 GB out of 16 GB on RAM leading to crashing, adding swap file and attaching it to swap file usking mkswap and other command the inactive or annonymous memory needed from your application can be stored and used from disk while leaving enough RAM for system to work efficiently"

**Score**: 7/10
**Corrections**:
- Crash-on-restart pattern = OOM Killer firing during startup
- Swap is mitigation, but diagnosis matters first

### Q6. OOM Killer
> "OOM happens when virtual memory is more than the RAM which is always and kernel overcommit the RAM based on that and application need that much of RAM or RAM more than physical memory causing this oom , now to handle this oom killer automatically keep tracks and maintain kinda table which is 10 * the percentage of memory using like 50% become 500 and similarly it will take decision to kill the process which can be killed to save the system, to avoid certain process from being killed from this , we can run oom_adj command not able to recall exact command but it can take a arg of number like 300 and it will substract the number like 300 from 500 giving the oom killer 200 which will make. the kernel skip this process from oom killer"

**Score**: 8.5/10
**Corrections**:
- Interface: `/proc/<pid>/oom_score_adj` (range -1000 to +1000)
- -1000 = fully exempt
- Otherwise conceptually solid

### Q7. Debugging tools used
> "strace - not able to recall ., lsof I. have used to identify the process that are using the deleted file.  top and htop I use to monitor memory cpu and other info when debugging other issues related to system or process, I generally prefer top over htop as it is light weight and don't have to worry about overstressing the system and keep the state in status quo position of error to avoid false debug as well also I use filters to get the desired columns like threads or ooms  , VMSTAT - i am not able. to recall,. dmesg i use to check the latest log for kernel"

**Score**: 8/10
**Gaps**: strace mechanics, vmstat basics (filled later)

---

## Retrieval Test — Raw Answers

### Q1. 500 MB CoW pages after fork+exec
> "the 500 MB was virtual memory address that was created by fork() . So, when exec is called the code is updated the virtual memory address should be updated, the 500MB was physical address mapped to parent process so if parent process is using that it will use or leave the RAM. CoW pages only get assigned as much needed by mapping virtual memory address to physical m a"

**Score**: 6/10
**Correction**: exec() discards entire virtual address space. CoW mappings torn down without copying. Physical RAM in parent untouched. That's why fork+exec is cheap.

### Q2. vmstat si and so both non-zero
> "It means the application is continuously do input and output in swap memory It is worse as the swap is not as fast as memory RAM , so it can lead to slowness more."

**Score**: 7/10
**Add**: si+so simultaneously = thrashing. System spends more time swapping than working.

### Q3. All microservices at oom_score_adj=-1000
> "OOMKIller won't have any process to kill in case of oom and can crash the system"

**Score**: 8/10
**Add**: System becomes unresponsive before panic. Kernel might still kill critical processes.

### Q4. D state process, manager says kill -9
> "No, that is not a good approach at all, the correct approach would be since its uninterruptable sleep, should use strace to check where process is stuck and then use lsof this will help me decide what its waiting on , for ths you can cat /proc/pid/wchan and then kill the process"

**Score**: 9/10
**Add**: Explain to manager why kill -9 won't work (D state ignores signals). Propose timeline.

### Q5. 32,000 processes from cron storm
> "this is very critical and can lead to server crash as no process can be created anymore, to diagnose it first. run ps aux |wc -l to get list and then check the process which are in s or d state due to any reason, then do strace and lsof on them to find the root and then perform the needed debug"

**Score**: 7/10
**Missing**: Stop cron FIRST (halt source), then kill parent, then fix code, then restart cron.

---

## Interleaving Scenarios

### Scenario A — fork() vs threads (1000 concurrent tasks)
**Answer**: "if you need to use the same memory or can work intertwined use thread and if you need isolation go for fork as it will create new child process"

**Evaluation**: 8/10
- Correct on isolation vs shared memory
- Missing: memory overhead, context switch cost, GIL (if Python), failure isolation, hybrid approaches

### Scenario B — Swap on 256GB trading server
**Answer**: "based on this we can use swap for safety net but instead of using default make the swap condition from 60 to 20"

**Evaluation**: 7/10
- Correct intuition: tune swappiness down
- Missing: Tiny swap (2GB), oom_score_adj, monitoring thresholds, what actual trading shops run

---

## Key Corrections & Interview One-Liners

### Process Management
- "Why can't you kill a zombie?" → Already dead. Kill parent; init reaps.
- "Why does fork+exec not double RAM?" → Copy-on-Write. exec discards mappings.
- "Process table full?" → `ulimit -u` or `kernel.pid_max`. Count, find zombies, raise limit.
- "Zombie?" → Child exited, parent never called wait().

### Signals
- "kill -9 vs -15?" → -9 uncatchable, no cleanup; -15 graceful, catchable.
- "SIGHUP on nginx?" → Config reload without dropping connections.
- "D state?" → Uninterruptible sleep (I/O wait). SIGKILL won't work.
- "Before SIGKILL?" → Check STAT (D vs S), strace -p, lsof -p, /proc/<pid>/wchan.

### Memory
- "malloc succeeds but app crashes later?" → Overcommit. Physical RAM assigned at page-touch, not malloc.
- "OOM Killer?" → Scores 0-1000. Protect with `/proc/<pid>/oom_score_adj`. Check `dmesg`.
- "si+so both non-zero?" → Thrashing. Swap loop. Kill memory hogs or add RAM.

### IPC
- (to be filled Day 2)

### Syscalls
- (to be filled Day 2)

---

## Handbooks Created

- ✅ `handbook-processes-memory.md` (see separate file)

## To-Do

- [ ] Create `handbook-ipc-syscalls.md` (Day 2)
- [ ] Update roadmap.md with Day 2 progress
- [ ] Create `handbook-networking.md` (Day 3)
- [ ] Create `handbook-filesystem-raid.md` (Day 4)
