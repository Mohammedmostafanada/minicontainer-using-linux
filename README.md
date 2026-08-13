# Building a Linux Container From Scratch — No Docker

A hands-on lab that builds a small Linux container from scratch using Linux kernel primitives — **without Docker, Podman, or other container engines**.

The goal is not to reinvent Docker for production. The goal is to understand what happens underneath a container runtime by assembling the core mechanisms manually.

---

## Why This Project?

Containers can feel like a black box when the only interaction is:

```bash
docker run ...
```

This project opens that box.

Instead of starting with Docker, we start with Linux primitives and build the isolation layers ourselves:

```text
Namespaces
    ↓
What can the process see?

Root filesystem
    ↓
What does / look like?

Networking
    ↓
What network environment does it have?

cgroups
    ↓
How much CPU / memory can it use?
```

By the end of the lab, these individual mechanisms are combined into a small, container-like environment.

---

## What You Will Learn

By completing this lab, you will understand and practice:

- PID namespaces and process isolation
- UTS namespaces and hostname isolation
- Mount namespaces and mount propagation
- `chroot` and why it is not a container by itself
- `pivot_root` and container-style root filesystems
- `/proc` and its relationship with PID namespaces
- BusyBox-based minimal root filesystems
- Network namespaces and virtual Ethernet (`veth`) pairs
- cgroups v2 for CPU and memory limits
- How these primitives are combined into a mini-container
- Why real container runtimes need additional security and lifecycle mechanisms

---
Guide

The full hands-on walkthrough is available in [`GUIDE.md`](GUIDE.md). It is intentionally ordered from individual Linux primitives to the final integrated mini-container.

### Jump to a Topic

- [Introduction](GUIDE.md#1-introduction-a-container-is-not-a-vm)
- [PID Namespace](GUIDE.md#2-pid-namespace--process-isolation)
- [UTS Namespace](GUIDE.md#3-uts-namespace--hostname-isolation)
- [Mount Namespace](GUIDE.md#4-mount-namespace--isolating-the-mount-view)
- [`chroot`](GUIDE.md#5-chroot--the-first-attempt-at-changing-)
- [BusyBox & Rootfs](GUIDE.md#6-busybox--building-a-small-root-filesystem)
- [`pivot_root`](GUIDE.md#7-pivot_root--changing-the-root-mount)
- [Network Namespace](GUIDE.md#8-network-namespace--network-isolation)
- [`veth` Pair](GUIDE.md#9-veth-pair--connecting-the-network-namespace)
- [cgroups v2](GUIDE.md#10-cgroups-v2--resource-control)
- [Final Mini-Container](GUIDE.md#11-final-mini-container--putting-everything-together)
- [Final Verification](GUIDE.md#12-final-verification)
- [Mental Model](GUIDE.md#13-the-mental-model)
- [What We Did Not Cover](GUIDE.md#14-what-we-did-not-cover)
- [End-to-End Flow](GUIDE.md#15-end-to-end-lab-flow)
- [Quick Glossary](GUIDE.md#16-quick-glossary)
- [Final Takeaway](GUIDE.md#17-final-takeaway)
---

## Architecture

The project builds a container-like environment from several independent Linux mechanisms:

```text
                              Linux Kernel
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
         Namespaces              rootfs              cgroups
              │                    │                    │
      ┌───────┼────────┐      pivot_root         CPU / Memory
      │       │        │
     PID     UTS     Mount
      │                │
      │               /proc
      │
   Network
      │
     veth
      │
      ▼
 Host / bridge / routing / NAT
```

### Core idea

| Component | Role |
| --- | --- |
| **Namespaces** | Isolate what a process can see and access |
| **Root filesystem + `pivot_root`** | Define the filesystem environment seen as `/` |
| **Network namespace + `veth`** | Define the network environment and connectivity |
| **cgroups v2** | Control resource usage such as CPU and memory |

---

## Project Structure

The repository is organized around a concise project entry point and a deeper hands-on guide:

```text
.
├── README.md        # Project overview, architecture, prerequisites, and quick start
├── GUIDE.md         # Full technical walkthrough and experiments
└── minicontainer/   # Root filesystem used by the final lab
    └── rootfs/
        └── bin/
            └── busybox
```

> The structure may grow as the lab is extended with scripts, documentation, and troubleshooting notes.

---

## Prerequisites

This is a **Linux-only, privileged systems lab**.

You should have:

- A Linux host or VM
- `root` or `sudo` access
- `iproute2` (`ip` command)
- BusyBox
- cgroups v2
- A kernel with Linux namespaces enabled

The lab was developed and tested in a RHEL-based environment.

Check a few prerequisites:

```bash
uname -a
ip -V
stat -fc %T /sys/fs/cgroup
```

For cgroups v2, the last command should report:

```text
cgroup2fs
```

---

## Safety / Lab Warning

> **Run this in a disposable VM when possible.**
> 
> The lab uses privileged operations such as `mount`, `pivot_root`, network configuration, and cgroup administration. These are real kernel operations, not simulations.
> 
> Do not experiment on a production machine unless you fully understand the commands you are executing.

The examples are intentionally small and incremental, but cleanup is still part of the lab.

---

## Quick Start

The project is designed to be learned **step by step**, not executed as one large script.

A typical flow is:

```text
1. Understand each primitive individually
        ↓
2. Build a minimal rootfs with BusyBox
        ↓
3. Create PID / UTS / Mount / Network namespaces
        ↓
4. Prepare the mount tree
        ↓
5. Switch the root filesystem with pivot_root
        ↓
6. Mount /proc for the new PID view
        ↓
7. Connect networking with a veth pair
        ↓
8. Attach the process to a cgroup
        ↓
9. Apply CPU / memory limits
        ↓
10. Verify the final environment
```

For the complete walkthrough, see:

**[`GUIDE.md`](GUIDE.md)**

---

## Key Concepts

### PID namespace

Controls the process view of a process tree.

```text
Host PID namespace
        │
        └── process PID 75390
                 │
                 ▼
        Container PID namespace
                 │
                 └── PID 1
```

### Mount namespace

Gives processes an independent mount view.

### UTS namespace

Provides an independent hostname/domain-name view.

### Root filesystem

Defines what the process sees as `/`.

### `pivot_root`

Replaces the root mount inside the Mount Namespace and allows the old root mount to be detached.

### Network namespace + `veth`

Provide an isolated network stack and a virtual link between the container namespace and the host side.

### cgroups v2

Apply resource controls such as:

```text
memory.max
cpu.max
```

---

## What This Lab Does Not Try to Be

This repository is an **educational implementation**, not a production-grade container runtime.

Real runtimes add additional layers such as:

- User namespaces
- Linux capabilities
- Seccomp filtering
- Filesystem layering / OverlayFS
- More complete networking and bridge/NAT setup
- Device handling
- Signal and process lifecycle management
- Automated cleanup
- Image management
- Security hardening

These are intentionally kept outside the first implementation so the core primitives remain understandable.

---

## Cleanup

The lab creates temporary resources such as:

- Network namespaces and virtual Ethernet devices
- Mounts
- cgroups
- Temporary rootfs state

For temporary namespaces created with `unshare`, exiting the namespace normally removes the namespace once no processes remain in it.

A typical host-side cleanup pattern includes removing the veth pair and the lab cgroup when they are no longer needed:

```bash
sudo ip link delete veth-host
sudo rmdir /sys/fs/cgroup/mycontainer
```

Only run cleanup commands when the corresponding resources actually exist and are no longer in use.

---

## Troubleshooting

The full troubleshooting discussion lives in `GUIDE.md`, but several common problems are worth knowing up front:

### `ps` cannot open `/proc`

If `ps` reports:

```text
ps: can't open '/proc': No such file or directory
```

make sure a proc filesystem is mounted inside the container rootfs:

```bash
mount -t proc proc /proc
```

### `chroot` cannot run `/bin/bash`

The rootfs probably does not contain the requested executable and its required runtime dependencies.

A small static BusyBox rootfs avoids most of the shared-library setup needed for a simple educational shell.

### `pivot_root` fails

Check that:

- you are inside a separate Mount Namespace
- mount propagation is appropriate (for the lab, make it private)
- the new root is a real mount point
- the new root is a distinct mount from the current root
- you have the required privileges

### `veth-ns` cannot be moved

Remember that `ip link set ... netns` uses a PID visible from the namespace where the command is run. When run on the host, use the host-visible PID, not the container's internal PID `1`.

---

## Repository Roadmap

The full learning path is:

```text
PID Namespace
    ↓
UTS Namespace
    ↓
Mount Namespace
    ↓
chroot
    ↓
BusyBox + rootfs
    ↓
pivot_root
    ↓
/proc
    ↓
Network Namespace
    ↓
veth
    ↓
cgroups v2
    ↓
Final Mini-Container
```

Each stage is explained and tested independently before the final integration.

---

## Final Takeaway

A Linux container is not a single feature.

It is a carefully assembled environment built from multiple kernel mechanisms:

```text
Namespaces
    +
Root filesystem
    +
Mounts
    +
Networking
    +
cgroups
    +
Security controls
```

The container runtime's job is to coordinate those mechanisms automatically.

The goal of this project is to make that invisible machinery visible.

---

## License

Add the repository license of your choice before publishing.

---

## Author

Built as a hands-on Linux / container internals learning project.

If you find the project useful, feel free to open an issue or suggest an improvement.
