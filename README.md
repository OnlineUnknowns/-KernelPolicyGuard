<p align="center">
  <img src="https://img.shields.io/badge/Kernel-5.10%2B-blue?logo=linux&logoColor=white" alt="Kernel 5.10+"/>
  <img src="https://img.shields.io/badge/License-GPLv2-green" alt="GPLv2"/>
  <img src="https://img.shields.io/badge/Architecture-NUMA%20%7C%20SMP-orange" alt="NUMA SMP"/>
  <img src="https://img.shields.io/badge/Enterprise-Ready-brightgreen" alt="Enterprise Ready"/>
  <img src="https://img.shields.io/badge/Status-Production-blue" alt="Production"/>
</p>

<h1 align="center">🛡️ KernelPolicyGuard</h1>
<p align="center">
  <strong>Dynamic LKM Security Framework · Real-Time Policy Enforcement · Cluster-Aware</strong><br/>
  <em>Formerly Codename: Chronos / Karenl</em>
</p>

---

## Overview

**KernelPolicyGuard** is a high-performance Linux Kernel Module (LKM) that provides **zero-latency security hooking** using `fprobes` and `kprobes`. It evaluates dynamic rule-based policies, offloads heavy processing to a **NUMA-aware workqueue**, and synchronizes state across clusters via a secure **TLV-based UDP protocol**.

Built for **enterprise-grade environments** — 1000+ nodes, 128-core servers — it bridges kernel-speed interception with user-space manageability via `sysfs` and **Generic Netlink**.

---

## Architecture

```
User Process
    │
    ▼
System Call (open / execve / send)
    │
    ▼
┌─────────────────────────────────────────────────┐
│           Fprobe / Kprobe (Atomic Context)       │
│   READ_ONCE(cfg.enabled) → queue_work_on(cpu)   │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────┐
│        NUMA-Aware Workqueue (Process Context)    │
│   Per-CPU dispatch · DoS cap: MAX_PENDING=1000  │
└──────────────────┬──────────────────────────────┘
                   │
          ┌────────┴────────┐
          ▼                 ▼
  ┌──────────────┐  ┌──────────────────────┐
  │ Policy Engine │  │   Cluster Sync UDP   │
  │  64-rule set  │  │   TLV Protocol v1    │
  │ ALLOW/DENY/  │  │  CRC32 + whitelist   │
  │     LOG       │  └──────────┬───────────┘
  └──────────────┘             │
                               ▼
                   ┌──────────────────────┐
                   │   Generic Netlink    │
                   │  KERNEL_POLICY_GUARD │
                   │  GET_STATS · UPDATE  │
                   │  STREAM_EVENTS mcast │
                   └──────────────────────┘
```

---

## Key Features

| Feature | Description |
|---|---|
| ⚡ Zero-Copy Hooking | `fprobe` (ftrace-based) for near-zero overhead interception |
| 🧠 NUMA-Aware Workqueue | Per-CPU dispatch via `queue_work_on()` with cache locality |
| 📜 TLV Cluster Sync | Type-Length-Value protocol over UDP — forward-compatible across versions |
| 🔐 Secure by Design | CRC32 checksums, sender whitelisting, `seqcount` lock-free reads |
| 🖥️ Dual Interface | `sysfs` for admin control + Generic Netlink for daemon integration |
| 📊 Real-Time Telemetry | Audit logging via character device with rate-limited kernel logs |
| 🔧 Dynamic Policy Engine | 64-rule sets with `EQ / GT / LT / RANGE` operators and `ALLOW / DENY / LOG` actions |

---

## Project Structure

```
KernelPolicyGuard/
├── chronos_main.c              # Core: init/exit, workqueue, sysfs, seqcount sync
├── include/
│   ├── chronos_protocol.h      # TLV headers, rule structs, Netlink command defs
│   └── uapi/
│       └── chronos_abi.h       # Public ABI for userspace tools
├── src/
│   ├── hooks/
│   │   ├── inode_hooks.c       # File/inode interception points
│   │   ├── ipc_hooks.c         # IPC event hooks
│   │   ├── net_hooks.c         # Network syscall hooks
│   │   └── task_hooks.c        # Process/task hooks
│   ├── abi_handshake.c         # Userspace ABI negotiation
│   ├── cluster_sync.c          # UDP listener + TLV parser
│   ├── netlink.c               # Generic Netlink family (KERNEL_POLICY_GUARD)
│   └── predictor.c             # Heuristic event predictor
├── policy/
│   ├── store.c                 # Rule storage & evaluation engine
│   └── store.h                 # Policy store definitions
├── telemetry/
│   └── audit_logger.c          # Char device for security event logging
└── Makefile
```

---

## Quick Start

### 1. Build

```bash
make clean && make -j$(nproc)
```

Requires: Linux kernel headers 5.10+, GCC or Clang, GNU Make.

### 2. Load the Module

```bash
sudo insmod KernelPolicyGuard.ko
```

Verify loading:

```bash
dmesg | tail -20 | grep KernelPolicyGuard
cat /sys/chronos/enabled
```

### 3. Add a Policy Rule

Rules follow the format: `add <hook_type> <hook> <operator> <value> <action>`

```bash
# Deny all 'open' calls exceeding 5 per second
echo "add inode open gt 5 deny" > /sys/chronos/rule_ctl

# Log all network sends
echo "add net send gt 0 log" > /sys/chronos/rule_ctl

# Allow process forks below threshold
echo "add task fork lt 100 allow" > /sys/chronos/rule_ctl
```

### 4. Monitor Events

```bash
# Real-time kernel log
sudo dmesg -w | grep KernelPolicyGuard

# Dropped events counter (DoS protection)
cat /sys/chronos/dropped

# Active rule count
cat /sys/chronos/rule_count
```

### 5. Unload

```bash
sudo rmmod KernelPolicyGuard
```

---

## Policy Engine

KernelPolicyGuard evaluates rules in order — **first-match-wins**.

### Rule Structure

| Field | Options |
|---|---|
| `hook_type` | `inode`, `net`, `task`, `ipc` |
| `hook` | `open`, `execve`, `send`, `fork`, `connect`, ... |
| `operator` | `EQ`, `GT`, `LT`, `RANGE` |
| `value` | u64 threshold |
| `action` | `ALLOW`, `DENY`, `LOG` |

### sysfs Interface

```bash
# Add a rule
echo "add inode open gt 100 deny" > /sys/chronos/rule_ctl

# Clear all rules
echo "clear" > /sys/chronos/rule_ctl

# View current rule count
cat /sys/chronos/rule_count
```

### Generic Netlink Commands

| Command | Description |
|---|---|
| `KPGG_CMD_GET_STATS` | Returns dropped/pending event counts |
| `KPGG_CMD_UPDATE_POLICY` | Accepts a binary rule struct for bulk update |
| `KPGG_CMD_STREAM_EVENTS` | Subscribes to the real-time multicast alert group |

---

## Cluster Sync Protocol (TLV)

Cluster peers communicate over UDP using a **Type-Length-Value** protocol designed for forward compatibility.

### Packet Layout

```
┌─────────────────────────────────────┐
│  uint16  total_len                  │
│  uint16  magic      (0xC4D5)        │
│  uint8   version    (1)             │
├─────────────────────────────────────┤
│  TLV[0]: type | len | data[]        │
│  TLV[1]: type | len | data[]        │
│  ...                                │
├─────────────────────────────────────┤
│  uint32  crc32 (over entire blob)   │
└─────────────────────────────────────┘
```

### Known TLV Types

| Type ID | Name | Description |
|---|---|---|
| `0x01` | `TYPE_NODE_ID` | Cluster node identifier |
| `0x02` | `TYPE_LOAD` | Current node load metric |
| `0x03` | `TYPE_BLACKLIST` | Blacklisted entity hash |

Unknown TLV types are **silently skipped** — enabling backward-compatible protocol upgrades.

---

## Security Hardening

| Component | Status | Detail |
|---|---|---|
| Locking model | ✅ Verified | `seqcount` for lock-free reads; mutex only in sysfs write paths |
| Atomic context safety | ✅ Verified | No `mutex_lock()` or `GFP_KERNEL` inside probe handlers |
| Input sanitization | ✅ Verified | All sysfs stores use `strscpy()` with explicit range validation |
| Cluster input | ✅ Verified | TLV parser enforces length + CRC32 checksum on every packet |
| DoS protection | ✅ Verified | Pending work capped at `MAX_PENDING_WORK = 1000` events |
| Rate limiting | ✅ Verified | All probe-path `printk` wrapped with `printk_ratelimit()` |
| Clean shutdown | ✅ Verified | `kthread_stop()` + `flush_workqueue()` + `sock_release()` in sequence |
| Sender whitelist | ✅ Verified | UDP source IPs validated against configured cluster peer list |

---

## NUMA / SMP Architecture

- Each workqueue item carries the **originating CPU ID** at enqueue time.
- Work is dispatched via `queue_work_on(cpu, wq, &item->work)` to maintain **cache locality**.
- The worker function logs its execution CPU for latency debugging.
- Workqueue flags: `WQ_UNBOUND | WQ_HIGHPRI` for priority scheduling across NUMA nodes.

---

## Kernel Version Compatibility

| Kernel | Status | Notes |
|---|---|---|
| 5.10 LTS | ✅ Supported | Minimum supported version |
| 5.15 LTS | ✅ Supported | Recommended for production |
| 6.1 LTS | ✅ Supported | Uses updated `genl` helpers |
| 6.6 LTS | ✅ Supported | Full fprobe multi-handler support |
| 6.x / 7.x | ✅ Future-proof | TLV protocol + genl design ensures compatibility |

> **Assumption:** Generic Netlink helpers used require kernel 6.1+ for the latest `genl_register_family()` API. A fallback path is included for 5.10–6.0.

---

## Contributing

This is a commercial-grade project. Internal contributors must follow kernel coding standards:

- **Atomic context restrictions**: No sleeping allocations or locks inside probe handlers.
- **READ_ONCE / WRITE_ONCE discipline**: All shared state accessed through appropriate barriers.
- **GFP flags**: Only `GFP_ATOMIC` permitted in interrupt/atomic context.
- See `CONTRIBUTING.md` for the full internal code review checklist.

---

## License

**GNU General Public License v2 (only)**

Built for Linux kernel 5.10+. All kernel module code is GPLv2-licensed per Linux kernel requirements.

---

## Enterprise Support

For enterprise deployment packages including:
- Custom eBPF integration
- Dedicated SLA and incident response
- Multi-site cluster mesh configuration
- Compliance audit documentation

Contact: **KernelPolicyGuard Research**

---

*Branded and maintained by the KernelPolicyGuard Team.*
