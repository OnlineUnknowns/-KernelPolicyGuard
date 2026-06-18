
<div align="center">

<img src="https://capsule-render.vercel.app/api?type=venom&height=220&text=KernelPolicyGuard&fontSize=45&color=0:000000,100:0F172A&fontColor=00FF41&stroke=00FF41&strokeWidth=1&animation=twinkling" width="100%"/>

<img src="https://readme-typing-svg.demolab.com?font=Share+Tech+Mono&size=18&duration=2500&pause=700&color=00FF41&center=true&vCenter=true&repeat=true&width=850&height=45&lines=%5BBOOT%5D+KernelPolicyGuard+Loaded;%5B*%5D+Attaching+fprobes...;%5B*%5D+Attaching+kprobes...;%5B*%5D+Loading+Policy+Engine...;%5B*%5D+Initializing+NUMA+Workqueues...;%5BREADY%5D+Production+Mode+Active." />

<br/>

[![Kernel](https://img.shields.io/badge/Kernel-5.10%2B-blue?style=for-the-badge&logo=linux&logoColor=white)](https://kernel.org)
[![License](https://img.shields.io/badge/License-GPLv2-green?style=for-the-badge)](LICENSE)
[![NUMA](https://img.shields.io/badge/NUMA-SMP-orange?style=for-the-badge&logo=intel)](#)
[![Enterprise](https://img.shields.io/badge/Enterprise-Ready-brightgreen?style=for-the-badge)](#)
[![Status](https://img.shields.io/badge/Status-Production-blue?style=for-the-badge)](#)
[![Hooks](https://img.shields.io/badge/Fprobe-Kprobe-purple?style=for-the-badge)](#)

</div>
```

<br/>

[![Kernel](https://img.shields.io/badge/Kernel-5.10%2B-blue?style=for-the-badge&logo=linux&logoColor=white)](https://kernel.org)
[![License](https://img.shields.io/badge/License-GPLv2-green?style=for-the-badge)](LICENSE)
[![Architecture](https://img.shields.io/badge/NUMA-SMP-orange?style=for-the-badge&logo=intel)](/)
[![Enterprise](https://img.shields.io/badge/Enterprise-Ready-brightgreen?style=for-the-badge&logo=checkmarx)](/)
[![Status](https://img.shields.io/badge/Status-Production-blue?style=for-the-badge)](/)
[![Hooks](https://img.shields.io/badge/fprobe-kprobe-purple?style=for-the-badge)](/)

</div>

---

## 🧠 What Is This?

**KernelPolicyGuard** is a high-performance Linux Kernel Module (LKM) that intercepts system calls at kernel speed using `fprobes` and `kprobes`, evaluates dynamic security policies, and synchronizes cluster state via a TLV-based UDP protocol.

> Built for 1000+ node clusters · 128-core NUMA servers · Zero-latency enforcement

---

## ⚡ Architecture

```
User Process  →  open() / execve() / send()
                        │
                        ▼
         ┌──────────────────────────────────┐
         │  Fprobe / Kprobe  (Atomic)       │
         │  READ_ONCE(cfg.enabled)          │
         │  queue_work_on(cpu, item)        │
         └─────────────┬────────────────────┘
                        │
                        ▼
         ┌──────────────────────────────────┐
         │  NUMA-Aware Workqueue            │
         │  WQ_UNBOUND | WQ_HIGHPRI        │
         │  DoS cap: MAX_PENDING = 1000     │
         └──────┬────────────────┬──────────┘
                │                │
                ▼                ▼
     ┌──────────────┐   ┌──────────────────┐
     │ Policy Engine│   │  Cluster UDP/TLV │
     │ 64 rules     │   │  CRC32+whitelist │
     │ ALLOW/DENY/  │   └────────┬─────────┘
     │ LOG          │            │
     └──────────────┘            ▼
                       ┌──────────────────┐
                       │  Generic Netlink │
                       │  STREAM_EVENTS   │
                       └──────────────────┘
```

---

## 🚀 Quick Start

```bash
# Build
make clean && make -j$(nproc)

# Load
sudo insmod KernelPolicyGuard.ko

# Add a deny rule
echo "add inode open gt 5 deny" > /sys/chronos/rule_ctl

# Monitor
sudo dmesg -w | grep KernelPolicyGuard
cat /sys/chronos/dropped

# Unload
sudo rmmod KernelPolicyGuard
```

---

## 📜 Policy Engine

| Field | Options |
|---|---|
| `hook_type` | `inode` · `net` · `task` · `ipc` |
| `operator` | `EQ` · `GT` · `LT` · `RANGE` |
| `action` | `ALLOW` · `DENY` · `LOG` |
| `max rules` | **64 per set** (first-match-wins) |

---

## 📡 TLV Cluster Protocol

```
┌──────────────────────────────────┐
│ uint16  total_len                │
│ uint16  magic      0xC4D5        │
│ uint8   version    1             │
├──────────────────────────────────┤
│ 0x01  TYPE_NODE_ID               │
│ 0x02  TYPE_LOAD                  │
│ 0x03  TYPE_BLACKLIST             │
│ 0xNN  UNKNOWN → silently skipped │
├──────────────────────────────────┤
│ uint32  crc32  (full blob)       │
└──────────────────────────────────┘
```

---

## 🔐 Security Hardening

| Component | Status |
|---|---|
| Locking model (seqcount) | ✅ Verified |
| Atomic context safety | ✅ Verified |
| Input sanitization (strscpy) | ✅ Verified |
| CRC32 cluster integrity | ✅ Verified |
| DoS event cap | ✅ Verified |
| Rate-limited printk | ✅ Verified |
| Clean shutdown sequence | ✅ Verified |
| Sender whitelist | ✅ Verified |

---

## 🗂️ Project Structure

```
KernelPolicyGuard/
├── chronos_main.c
├── include/
│   ├── chronos_protocol.h
│   └── uapi/chronos_abi.h
├── src/
│   ├── hooks/
│   │   ├── inode_hooks.c
│   │   ├── ipc_hooks.c
│   │   ├── net_hooks.c
│   │   └── task_hooks.c
│   ├── cluster_sync.c
│   ├── netlink.c
│   └── predictor.c
├── policy/
│   ├── store.c
│   └── store.h
├── telemetry/
│   └── audit_logger.c
└── Makefile
```

---

## 🖥️ Kernel Compatibility

| Kernel | Status |
|---|---|
| 5.10 LTS | ✅ Minimum supported |
| 5.15 LTS | ✅ Recommended |
| 6.1 LTS | ✅ Updated genl API |
| 6.6 LTS | ✅ Full fprobe support |
| 6.x / 7.x | 🔮 Future-proof by design |

---

## 📄 License

**GNU General Public License v2 (only)**

---

<div align="center">

**KernelPolicyGuard Research** · Enterprise Support · Custom eBPF · Multi-site Cluster Mesh

</div>
