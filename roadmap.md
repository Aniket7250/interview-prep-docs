# Linux SRE Interview Preparation — PERRIO Roadmap

**Total budget**: 30 hours across 6 days (4 learning + 2 revision)
**Start date**: 2025-04-01
**Method**: PERRIO (Prime, Encode, Retrieve, Interleave, Over-learn, + revision)

## Topic Clusters

| # | Cluster | Status | Day(s) | Hours | Notes |
|---|---------|--------|--------|-------|-------|
| 1 | Filesystem, Storage, LVM, RAID | ✅ Done (Modules 1-2, RAID pending) | Carry-over | - | LVM/FS strong, RAID scheduled Day 4 |
| 2 | Process Management & Signals | ✅ Done | Day 1 | 6 | Completed: fork/exec, signals, memory, OOM |
| 3 | Memory Management | ✅ Done | Day 1 | 6 | Included in Day 1 |
| 4 | IPC, Syscalls, Kernel Interaction | 🔲 Next | Day 2 | 6 | Baseline assessment in progress |
| 5 | Networking Fundamentals | 🔲 Pending | Day 3 | 6 | TCP/IP, DNS, DHCP, ARP, subnets |
| 6 | Debugging Tools (lsof, strace, vmstat, dmesg) | 🔲 Partial | Day 4 | 1 | Covered some in Day 1, will complete |

## Day-by-Day Log

### Day 1 — Processes, Signals & Memory (2025-04-01)
- **Hours**: 6
- **Status**: ✅ Complete
- **Baseline score**: 79/100
- **Retrieval score**: 74/100
- **Interleaving score**: 7.5/10
- **Deliverables**:
  - `handbook-processes-memory.md`
  - Baseline + retrieval answers saved in `perrio-context.md`
- **Gaps identified**:
  - Shell does fork(), not systemd (low)
  - Zombie = parent never called wait() (medium)
  - Process table full → ulimit/pid_max (high)
  - strace mechanics (high)
  - vmstat columns (medium)
  - oom_score_adj exact syntax (medium)

### Day 2 — IPC, Syscalls & Kernel Interaction
- **Status**: 🟡 In Progress — Baseline assessment pending
- **Hours planned**: 6
- **Focus**: Pipes, sockets, shared memory, semaphores, syscall mechanics, context switches, strace/ltrace

### Day 3 — Networking Fundamentals
- **Status**: ⏳ Pending
- **Hours planned**: 6

### Day 4 — RAID + Integration + Gap Fill
- **Status**: ⏳ Pending
- **Hours planned**: 5
- **Tasks**:
  - Complete RAID (Module 3)
  - Hard/soft links deep-dive
  - Filesystem + process integration
  - Debugging tools comprehensive (lsof, strace, netstat/ss, dmesg, top/htop)

### Day 5 — Revision Day 1: Weak Spots + Scenario Drills
- **Status**: ⏳ Pending
- **Hours planned**: 3.5
- **Tasks**:
  - PERRIO Retrieval pass — all clusters
  - Overlearning scenarios (production incidents)
  - Cheat sheet review

### Day 6 — Revision Day 2: Mock Interview Simulation
- **Status**: ⏳ Pending
- **Hours planned**: 3.5
- **Tasks**:
  - Full mock interview (45 min)
  - Targeted re-encoding on fumbled topics
  - Final reference pass

## Hours Budget

| Day | Planned | Used | Remaining |
|-----|---------|------|-----------|
| 1 | 6 | 6 | 0 |
| 2 | 6 | 0 | 6 |
| 3 | 6 | 0 | 6 |
| 4 | 5 | 0 | 5 |
| 5 | 3.5 | 0 | 3.5 |
| 6 | 3.5 | 0 | 3.5 |
| **Total** | **30** | **6** | **24** |

## Resources Used

- Claude (PERRIO sessions) — Primary
- TLPI (The Linux Programming Interface) — Skim-only validation
- YouTube: Low Level Learning, NetworkChuck, Hussein Nasser
- man pages — Always open alongside PERRIO sessions

## Handbooks Created

- [x] `handbook-processes-memory.md` — Process lifecycle, signals, memory, debugging
- [ ] `handbook-ipc-syscalls.md` — Pending Day 2
- [ ] `handbook-networking.md` — Pending Day 3
- [ ] `handbook-filesystem-raid.md` — Pending Day 4

## Notes

- Use "PERRIO session: [topic]" prompt to start each session
- Keep cheat sheets concise — one-pager style
- Focus on interview one-liners and production scenarios
