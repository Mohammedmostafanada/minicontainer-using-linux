# Building a Linux Container From Scratch — No Docker

> A hands-on explanation of how a simple Linux container can be built using only core Linux primitives such as `unshare`, `chroot` / `pivot_root`, `cgroups v2`, and `veth`.
> 
> The goal is to understand that a container is not "magic technology" — it is an ordinary Linux process running under a carefully constructed set of kernel isolation and resource-control mechanisms.

> **Scope:** This guide is an educational lab, not a production-grade container runtime. It focuses on the core Linux primitives and the reasoning behind them.

## How to Use This Guide

This guide is intentionally ordered from individual primitives to a final integrated mini-container. Follow the sections in order and run commands from the context indicated by each section.

The learning pattern is:

```text
Concept
  ↓
Primitive
  ↓
Small experiment
  ↓
Expected behavior
  ↓
Integration
```

### Command Context

- **Host** — the normal Linux shell outside the experimental namespaces.
- **Namespace / Container** — a shell running inside the namespaces created for the lab.
- **Second Host terminal** — another normal host shell used when the host must inspect or modify a resource owned by the container namespace.

> When a command must use the host-visible PID instead of the container-visible PID, this guide calls that out explicitly.

## Prerequisites

- A Linux system or disposable Linux VM
- `sudo` access
- `iproute2` (`ip`)
- `mount` / `umount` utilities
- BusyBox
- cgroups v2 mounted at `/sys/fs/cgroup`
- A kernel with Linux namespaces enabled

> **Safety:** This lab uses privileged operations such as `mount`, `pivot_root`, virtual networking, and cgroup administration. A disposable VM is strongly recommended.

## Repository Companion Files

The repository separates the project overview from the detailed walkthrough:

- `README.md` — project overview, architecture, prerequisites, scope, and quick orientation.
- `GUIDE.md` — the full hands-on walkthrough and the reasoning behind each step.

---
Table of Contents

- [1. Introduction: A Container Is Not a VM](#1-introduction-a-container-is-not-a-vm)
- [2. PID Namespace — Process Isolation](#2-pid-namespace--process-isolation)
- [3. UTS Namespace — Hostname Isolation](#3-uts-namespace--hostname-isolation)
- [4. Mount Namespace — Isolating the Mount View](#4-mount-namespace--isolating-the-mount-view)
- [5. chroot — The First Attempt at Changing `/`](#5-chroot--the-first-attempt-at-changing-)
- [6. BusyBox — Building a Small Root Filesystem](#6-busybox--building-a-small-root-filesystem)
- [7. pivot_root — Changing the Root Mount](#7-pivot_root--changing-the-root-mount)
- [8. Network Namespace — Network Isolation](#8-network-namespace--network-isolation)
- [9. veth Pair — Connecting the Network Namespace](#9-veth-pair--connecting-the-network-namespace)
- [10. cgroups v2 — Resource Control](#10-cgroups-v2--resource-control)
- [11. Final Mini-Container — Putting Everything Together](#11-final-mini-container--putting-everything-together)
- [12. Final Verification](#12-final-verification)
- [13. The Mental Model](#13-the-mental-model)
- [14. What We Did Not Cover](#14-what-we-did-not-cover)
- [15. End-to-End Lab Flow](#15-end-to-end-lab-flow)
- [16. Quick Glossary](#16-quick-glossary)
- [17. Final Takeaway](#17-final-takeaway)
- [18. Workflow Summary (Quick Reference)](#18-workflow-summary-quick-reference)

---

## 1. Introduction: A Container Is Not a VM

Before writing any commands, this idea should be clear:

- A **VM** runs a complete guest operating system, including its own kernel, usually under a hypervisor.
- A **container** is an ordinary process running on the host's kernel, isolated using several Linux kernel primitives.

| Component | Purpose |
| --- | --- |
| **Namespaces** | Control what the process can see: processes, hostname, mounts, network, etc. |
| **Root filesystem** | Defines the filesystem environment seen as `/` by the process. |
| **cgroups** | Control how much resources the process can consume, such as CPU and memory. |
| **Networking** | Provides an isolated network environment and connectivity. |

A useful mental model is:

```
Namespaces
    ↓
What can the process see?

Root filesystem
    ↓
What does / look like?

cgroups
    ↓
How much can the process use?

Networking
    ↓
What network environment can the process access?
```

Container runtimes such as Docker, Podman, containerd, and OCI runtimes such as runc automate and coordinate mechanisms like these instead of requiring the user to assemble them manually.
[↑ Back to Table of Contents](#table-of-contents)
---

## 2. PID Namespace — Process Isolation

### 2.1 The Idea

A PID namespace gives processes a separate PID view.

A process inside the namespace can have a different PID from the same process as seen from the host.

The first process created inside a new PID namespace becomes:

```
PID 1
```

This process has a special role similar to an init process for that namespace.

### 2.2 Creating a PID Namespace

```
unshare --pid --fork bash
```

- `--pid`: create a new PID namespace.
- `--fork`: run the requested program as a child process, which is the process that becomes the first process in the new PID namespace.

Check the PID:

```
echo $$
```

Expected:

```
1
```

### 2.3 The `ps` Problem

At first, `ps` may still show the host's processes.

Why? Because `ps` obtains process information through `/proc`.

If the new PID namespace does not have an appropriate `/proc` mount, the process may still be looking at the host's proc filesystem.

The result is that the PID namespace exists, but tools such as `ps` do not yet have the correct process view.

### 2.4 Mounting `/proc`

```
unshare --pid --fork --mount-proc bash
```

- `--mount-proc` mounts a new proc filesystem that reflects the PID namespace.
- It also creates a new mount namespace so that the `/proc` mount does not modify the host's existing mount environment.

Now:

```
ps
```

shows the processes visible inside the new PID namespace.

### 2.5 PID Mapping

The same process can have different PIDs depending on which PID namespace is looking at it.

Example:

```
Host PID  →  Container PID
  75390   →       1
```

From the host, the process appears as `75390`. From inside the PID namespace, the same process appears as `1`.

The kernel exposes this mapping through the `NSpid` field in `/proc/<pid>/status`. To read it, you need to look up that file **from a namespace that can see the target PID** — typically from the host, using the process's host-visible PID. A process running fully inside an isolated namespace generally cannot inspect PIDs outside its own namespace.

[↑ Back to Table of Contents](#table-of-contents)
---

## 3. UTS Namespace — Hostname Isolation

### 3.1 The Idea

A UTS namespace isolates:

- hostname
- domain name

This allows a container to have its own hostname without changing the host's hostname.

### 3.2 Creating a UTS Namespace

```
unshare --uts bash
```

Inside the namespace:

```
hostname container
```

Check it:

```
hostname
```

Expected:

```
container
```

The host's hostname remains unchanged because the hostname belongs to the UTS namespace of the process.

### 3.3 `hostname` vs `hostnamectl`

For this lab, use:

```
hostname container
```

not:

```
hostnamectl hostname container
```

The reason is that we are studying the runtime hostname associated with the UTS namespace. `hostnamectl` is a systemd-oriented tool that also manages the system's persistent hostname configuration. Using it in a namespace lab can introduce system-wide configuration changes that are unrelated to the concept we are studying.

[↑ Back to Table of Contents](#table-of-contents)
---

## 4. Mount Namespace — Isolating the Mount View

### 4.1 The Idea

A Mount Namespace isolates the process's mount view. It does not create a new filesystem or duplicate the underlying disk. Instead, different namespaces can have different mount trees.

```
Host Mount Namespace
    └── /mnt/test

Container Mount Namespace
    └── /mnt/test
         └── different mount
```

### 4.2 Creating a Mount Namespace

```
unshare --mount bash
```

### 4.3 Proving It Is a Different Namespace

```
readlink /proc/$$/ns/mnt
```

Compare that value with the same command on the host. A different namespace ID confirms that the process belongs to a different Mount Namespace.

### 4.4 Practical Experiment: tmpfs

Create a mount:

```
mount -t tmpfs tmpfs /mnt/test
```

The command contains:

- `-t tmpfs`: filesystem type
- `tmpfs`: filesystem source name
- `/mnt/test`: mount point

Now the namespace sees:

```
/mnt/test
    ↓
tmpfs
```

while the host may still see:

```
/mnt/test
    ↓
nothing mounted
```

The directory itself still exists because it belongs to the underlying filesystem. **The mount placed over it is what is isolated** — not the directory itself.

### 4.5 Mount Propagation

Mount events can propagate between namespaces depending on the propagation mode. Important propagation modes include:

- `shared`
- `private`
- `slave`
- `unbindable`

For our container setup, we want mount changes to stay inside the container's Mount Namespace. Therefore we use:

```
mount --make-rprivate /
```

This makes the mounts recursively private and prevents mount events from propagating between the container's mount tree and the host's mount tree.

[↑ Back to Table of Contents](#table-of-contents)
---

## 5. chroot — The First Attempt at Changing `/`

### 5.1 The Idea

`chroot` changes the root directory seen by a process. For example:

```
chroot /path/to/rootfs
```

If `/home/mohamed/rootfs` is used as the new root, the process sees it as `/`.

However, `chroot` does **not** isolate:

- processes
- network
- resources
- hostname
- mounts

So `chroot` (change filesystem root) is not equivalent to a container.

### 5.2 Why Did the First `chroot` Fail?

If the rootfs is empty and we run:

```
chroot /path/to/rootfs
```

the command may fail with:

```
/bin/bash: No such file or directory
```

The problem is not that `chroot` itself failed. The problem is that the new root filesystem does not contain the program that `chroot` is trying to execute.

A usable rootfs needs a userspace environment. Depending on the programs being executed, this can include:

- executable binaries
- dynamic loader
- shared libraries
- filesystem directories
- configuration files
- `/proc`
- other runtime filesystems

### 5.3 Static vs Dynamic Binaries

**Dynamic binary**

A dynamically linked binary depends on a dynamic loader and shared libraries:

```
binary
   ↓
dynamic loader
   ↓
shared libraries
```

Those dependencies must exist inside the rootfs.

**Static binary**

A statically linked binary contains the code it needs for execution and therefore does not require external shared libraries in the rootfs. This is why a static BusyBox binary is ideal for a small educational rootfs.

[↑ Back to Table of Contents](#table-of-contents)
---

## 6. BusyBox — Building a Small Root Filesystem

### 6.1 Why BusyBox?

BusyBox is a single binary containing many Unix utilities, for example:

```
sh, ls, ps, cat, echo, mount, cp, mkdir, ...
```

This makes it extremely useful for creating a tiny educational rootfs.

### 6.2 Rootfs Layout

Our rootfs starts like this:

```
rootfs/
└── bin/
    └── busybox
```

### 6.3 BusyBox Applets

BusyBox uses the concept of **applets**. Instead of having completely separate binaries for every command, multiple command names can point to the same BusyBox executable. For example:

```
/bin/sh  ─┐
/bin/ls  ─┼──► /bin/busybox
/bin/ps  ─┘
```

BusyBox determines which applet to execute based on the name through which it was invoked.

### 6.4 Creating the Symlinks Automatically

Instead of creating every symlink manually:

```
busybox --install -s /bin
```

This creates symlinks for the supported BusyBox applets in the specified directory. The exact applets available depend on how that BusyBox binary was built.

### 6.5 Running BusyBox Inside chroot

```
chroot rootfs /bin/sh
```

Now the shell is running with the rootfs as `/`.

### 6.6 Why `ps` Still Needs `/proc`

Even if BusyBox contains the `ps` applet, `ps` still needs access to `/proc` to obtain process information. Without a mounted `/proc`, you may see an error such as:

```
ps: can't open '/proc': No such file or directory
```

This is the same important relationship we saw earlier:

```
PID Namespace
      +
appropriate /proc mount
      ↓
correct process view
```
[↑ Back to Table of Contents](#table-of-contents)
---

## 7. pivot\_root — Changing the Root Mount

### 7.1 Why Use pivot\_root?

`chroot` changes the root directory used by a process. `pivot_root`, however, changes the root mount inside the Mount Namespace. That makes it a better fit for constructing the filesystem environment of our mini-container.

This does not mean that `pivot_root` itself is a security boundary. The isolation comes from the combination of the Mount Namespace and the rest of the container configuration.

### 7.2 The Difference

| `chroot` `pivot_root` |     |     |
| --- | --- | --- |
| Changes | The root directory used by a process | The root mount in the Mount Namespace |
| Scope | Mainly a filesystem-root operation | Mount-tree operation |
| Isolation | Does not itself isolate the process | Designed to work with mount namespace isolation |
| Typical use | Simple environments | Constructing a container-style root filesystem |

### 7.3 Requirements

`pivot_root` requires:

- a separate Mount Namespace for our experiment
- the new root must **be a mount point**
- the new root must **not be on the same filesystem/mount as the current root** — `pivot_root` moves the current root mount aside and installs a different mount in its place, so the two must be distinct mounts. This is a common source of confusing errors for beginners who try it against a plain directory that hasn't been bind-mounted.
- the relevant mount propagation must not be `shared`
- appropriate privileges, typically `CAP_SYS_ADMIN` — because changing the root mount of a namespace is a privileged operation that affects how the entire process tree under that namespace sees the filesystem, the kernel restricts it to processes with this capability (normally held by root).

### 7.4 Making the Rootfs a Mount Point

A directory can be made a mount point by bind mounting it onto itself:

```
mount --bind rootfs rootfs
```

This does not duplicate the filesystem. It simply makes `rootfs` a proper mount point, satisfying one of the requirements of `pivot_root`.

### 7.5 Performing the Pivot

```
pivot_root /rootfs /rootfs/oldroot
```

Conceptually:

**Before:**

```
/
└── old host root

/rootfs
└── container rootfs
```

**After:**

```
/
└── container rootfs

/oldroot
└── old root mount
```

The old root is now reachable through `/oldroot` within the namespace.

### 7.6 Cleaning Up the Old Root

Switch the current working directory:

```
cd /
```

Then unmount the old root:

```
umount /oldroot
```

After that, the old root mount is no longer part of the active mount tree of the container. `pivot_root` is a mount operation; removing the old root with `umount` is a separate cleanup step.

### 7.7 Rollback During the Lab

Because the experiment is running inside a separate Mount Namespace, the simplest cleanup method is normally to terminate the namespace's processes. For a temporary namespace created with `unshare`, once the namespace no longer has member processes, it is torn down automatically.

[↑ Back to Table of Contents](#table-of-contents)
---

## 8. Network Namespace — Network Isolation

### 8.1 The Idea

A Network Namespace provides an independent network environment, including:

- network interfaces
- IP addresses
- routing tables
- network stack
- sockets and ports
- network-related `/proc` and `/sys` views

### 8.2 Creating a Network Namespace

```
unshare --net bash
```

### 8.3 Loopback

A new network namespace initially has a loopback interface (`lo`) that is normally administratively down. Enable it:

```
ip link set lo up
```

You may see `state UNKNOWN` for loopback. That is normal because loopback does not have a physical carrier whose link state can be detected like an Ethernet device.

[↑ Back to Table of Contents](#table-of-contents)
---

## 9. veth Pair — Connecting the Network Namespace

### 9.1 The Idea

A veth pair behaves like a virtual Ethernet cable:

```
veth-host  <──────────────►  veth-ns
```

Packets entering one end appear at the other end.

### 9.2 Creating the Pair

```
ip link add veth-host type veth peer name veth-ns
```

Initially, both ends belong to the same Network Namespace where the command was executed.

### 9.3 Moving One End into the Container Namespace

From the host:

```
ip link set veth-ns netns <HOST_PID>
```

The PID supplied here must be a PID **visible from the host's PID namespace**. This matters because this command is run from the host shell, outside the container's namespaces — the host has no knowledge of the container's internal PID numbering (like `1`), so it must reference the process by the PID it sees, not the PID the process sees itself as.

For example:

```
Host PID       Container PID
75390      →        1
```

The host uses `75390` because `ip link set ... netns` is executed from the host.

### 9.4 Assigning IP Addresses

```
Host:      veth-host → 10.10.10.1/24
Container: veth-ns   → 10.10.10.2/24
```

The resulting topology is:

```
Host                         Container

10.10.10.1                  10.10.10.2
veth-host  <──────────────>  veth-ns
```

### 9.5 Testing Connectivity

From the container:

```
ping -c 3 10.10.10.1
```

A successful ping proves connectivity between the Network Namespace and the host.

### 9.6 Host Connectivity Is Not Internet Connectivity

The veth pair alone does not provide Internet access. Internet access would additionally require mechanisms such as:

- IP forwarding
- routing
- NAT / masquerading
- possibly a bridge or another host-side networking setup

Those are outside the scope of this basic lab.

[↑ Back to Table of Contents](#table-of-contents)
---

## 10. cgroups v2 — Resource Control

### 10.1 Namespaces vs cgroups

```
Namespaces  → What can the process see?
              (visibility / isolation)

cgroups     → How much can the process use?
              (resource control)
```

These mechanisms are independent. You can use cgroups without namespaces, and namespaces without cgroups.

### 10.2 Verify cgroups v2

```
stat -fc %T /sys/fs/cgroup
```

Expected:

```
cgroup2fs
```

### 10.3 Create a cgroup

```
mkdir /sys/fs/cgroup/mycontainer
```

Linux automatically exposes cgroup controller files inside the directory.

### 10.4 Add a Process

Processes are associated with a cgroup through `cgroup.procs`:

```
echo 39696 | sudo tee /sys/fs/cgroup/mycontainer/cgroup.procs
```

Now PID `39696` belongs to the cgroup.

### 10.5 Memory Limit

```
echo 50M | sudo tee /sys/fs/cgroup/mycontainer/memory.max
```

This sets the maximum memory limit for the cgroup.

### 10.6 CPU Limit

```
echo "50000 100000" | sudo tee /sys/fs/cgroup/mycontainer/cpu.max
```

The values mean:

```
quota  = 50000 microseconds
period = 100000 microseconds
```

So the cgroup receives at most `50000 / 100000 = 50%` of one CPU's scheduling time over each 100 ms period.

[↑ Back to Table of Contents](#table-of-contents)
---

## 11. Final Mini-Container — Putting Everything Together

After understanding every primitive independently, we can combine them.

### 11.1 Prepare the Rootfs

```
minicontainer/
└── rootfs/
    └── bin/
        └── busybox
```

### 11.2 Create the Namespaces

```
unshare --pid --uts --mount --net --fork bash
```

Now the process has: PID, UTS, Mount, and Network namespaces.

### 11.3 Make Mounts Private

```
mount --make-rprivate /
```

This prevents mount events from propagating to the host.

### 11.4 Prepare the Rootfs for pivot\_root

```
mkdir -p rootfs/proc rootfs/oldroot
mount --bind rootfs rootfs
```

The bind mount makes `rootfs` a proper mount point.

### 11.5 Perform pivot\_root

```
pivot_root rootfs rootfs/oldroot
cd /
umount /oldroot
```

Now the container root filesystem is `/`.

### 11.6 Mount /proc

```
mkdir -p /proc
mount -t proc proc /proc
```

Now the proc filesystem reflects the PID namespace instead of exposing the host's process view.

### 11.7 Configure the UTS Namespace

```
hostname minicontainer
```

The container now has its own hostname.

### 11.8 Configure Networking

Create the veth pair from the host:

```
ip link add veth-host type veth peer name veth-ns
```

Move one end into the container's Network Namespace:

```
ip link set veth-ns netns <HOST_PID>
```

Assign:

```
Host       → 10.10.10.1/24
Container  → 10.10.10.2/24
```

Test:

```
ping -c 3 10.10.10.1
```

### 11.9 Attach the Container Process to the cgroup

From the host, use the host PID:

```
echo <HOST_PID> | sudo tee /sys/fs/cgroup/mycontainer/cgroup.procs
```

### 11.10 Apply Resource Limits

```
echo 50M | sudo tee /sys/fs/cgroup/mycontainer/memory.max
echo "50000 100000" | sudo tee /sys/fs/cgroup/mycontainer/cpu.max
```

---
### 11.11 Running an Application Instead of a Shell

Throughout this lab, we used `/bin/sh` as the container's interactive entry point so we could inspect and configure the environment.

A real container does not have to start a shell. It can start any executable available inside the rootfs.

For example:

```bash
/path/to/application
```
When that application is started as the first process in the PID namespace, it becomes:

PID 1

Conceptually:
```

Container
└── PID 1
    └── application

instead of:

Container
└── PID 1
    └── /bin/sh
```

This is the basic idea behind running a container for a specific workload: the container provides the isolated environment, while the application is the process running inside it.

Note: /bin/sh is used throughout this lab for interactive learning and debugging. In a real container, the main application would normally be the process started as PID 1.

[↑ Back to Table of Contents](#table-of-contents)
---
## 12. Final Verification

### 12.1 PID Isolation

Inside the container:

```
echo $$
ps
```

Expected: `PID 1`, and only the processes belonging to the container's PID namespace should be visible.

### 12.2 UTS Isolation

```
hostname
```

Expected: `minicontainer`. The host should still have its original hostname.

### 12.3 Filesystem Isolation

```
pwd
ls /
```

Expected: `/`, and the contents should come from the container's rootfs.

### 12.4 /proc

```
mount | grep ' /proc '
```

You should see a proc filesystem mounted at `/proc`. Then `ps` should work because the container now has the proc filesystem required by `ps`.

### 12.5 Network Isolation

```
ip addr
```

You should see `lo` and `veth-ns` with `10.10.10.2/24`. Test:

```
ping -c 3 10.10.10.1
```

### 12.6 cgroup Configuration

From the host:

```
cat /sys/fs/cgroup/mycontainer/cgroup.procs
cat /sys/fs/cgroup/mycontainer/memory.max
cat /sys/fs/cgroup/mycontainer/cpu.max
```

These confirm that the container process belongs to the cgroup and that the resource limits are configured.

---

## 12.7 Cleanup Checklist

When the lab is complete, remove temporary resources from the host side. The exact cleanup depends on where execution stopped, but the common resources are:

```bash
# Remove the veth pair (deleting one end removes the pair)
sudo ip link delete veth-host

# Restore cgroup limits before removing the cgroup
echo max | sudo tee /sys/fs/cgroup/mycontainer/memory.max
echo "max 100000" | sudo tee /sys/fs/cgroup/mycontainer/cpu.max

# Remove the cgroup once it contains no processes or child cgroups
sudo rmdir /sys/fs/cgroup/mycontainer
```

> If the namespace shell is still running, exit it first. Temporary namespaces created by `unshare` disappear after their member processes terminate.

## 12.8 Common Troubleshooting Cases

### `ps` shows host processes

Check whether `/proc` was mounted for the PID namespace. A PID namespace can exist while `ps` still reads the host's proc filesystem.

### `ps: can't open '/proc'`

The new rootfs probably does not contain a `/proc` mount point yet. Create it inside the new root and mount proc there:

```bash
mkdir -p /proc
mount -t proc proc /proc
```

### `chroot` cannot run `/bin/bash`

The selected rootfs does not contain the executable and/or its runtime dependencies. For this lab, BusyBox is used to provide a small static userspace.

### `ip link set ... netns <PID>` does not move the interface

Make sure the command is being run from the host and that `<PID>` is the **host-visible PID** of a process that owns the target Network Namespace. The PID may be `1` inside the container while the host sees the same process as a different PID.

### `pivot_root` fails

Check the prerequisites: you should be inside a dedicated Mount Namespace, mount propagation should not be shared, and the new root should be a proper mount point. A self bind mount is commonly used for the lab rootfs.

---
[↑ Back to Table of Contents](#table-of-contents)
---

## 13. The Mental Model

A Linux container is not one feature. It is a combination of independent kernel mechanisms:

```
                     Linux Container
                           │
          ┌────────────────┼────────────────┐
          │                │                │
      Namespaces         rootfs           cgroups
          │                │                │
      isolation         filesystem       resource limits
          │                │                │
    ┌─────┼─────┐          │          ┌────┴────┐
    │     │     │          │          │         │
   PID   UTS   Mount     pivot_root   CPU     Memory
    │           │
    │          /proc
    │
  Network
    │
   veth
```

### Logical Relationship

| Component Role |     |
| --- | --- |
| **Namespaces** | Isolate the process's view of the system |
| **rootfs + pivot\_root** | Define the filesystem environment |
| **Network Namespace + veth** | Define the network environment and connectivity |
| **cgroups** | Control resource consumption |

The core idea is:

```
Namespaces
    ↓
What can the process see?

rootfs + pivot_root
    ↓
What does / look like?

veth + Network Namespace
    ↓
What network environment does it have?

cgroups
    ↓
How much can it use?
```
[↑ Back to Table of Contents](#table-of-contents)
---

## 14. What We Did Not Cover

This lab deliberately focuses on the core building blocks. Real container runtimes add many additional mechanisms and security layers. Important topics for the next stage include:

**User Namespaces**

User namespaces isolate UIDs, GIDs, and capabilities. They are particularly important for reducing the danger of having a process that appears to be root inside the container.

**Linux Capabilities**

Instead of giving a process unrestricted root privileges, capabilities split root's privileges into smaller permission sets, such as `CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, `CAP_CHOWN`, `CAP_KILL`, and others.

**Seccomp**

Seccomp can restrict which system calls a process is allowed to make.

**Image Layering**

Container images are usually not just one directory copied to disk. Technologies such as overlayfs allow multiple filesystem layers to be combined into a single view.

**Internet Connectivity**

A realistic container network commonly requires:

```
veth → bridge → routing / forwarding → NAT → Internet
```

**Lifecycle Management**

A real container runtime also handles process supervision, cleanup, namespace lifecycle, network cleanup, cgroup cleanup, signal handling, exit status, and logging.

---
[↑ Back to Table of Contents](#table-of-contents)
---

## 15. End-to-End Lab Flow

Use this as the condensed execution order after studying the detailed sections above.

```text
1. Prepare a minimal BusyBox rootfs
2. Create PID + UTS + Mount + Network namespaces
3. Make the mount tree private
4. Prepare rootfs + /proc + /oldroot
5. pivot_root into the container rootfs
6. Unmount the old root
7. Mount /proc inside the new root
8. Set the container hostname
9. Create a veth pair on the host
10. Move one veth endpoint into the container Network Namespace
11. Assign IP addresses and test host connectivity
12. Attach the container process to a cgroup
13. Apply CPU and memory limits
14. Verify PID, hostname, filesystem, /proc, network, and cgroup state
15. Clean up temporary resources
```
[↑ Back to Table of Contents](#table-of-contents)
---

## 16. Quick Glossary

| Term | Meaning in this lab |
| --- | --- |
| **PID Namespace** | Separate PID view for processes. |
| **UTS Namespace** | Separate hostname/domain-name view. |
| **Mount Namespace** | Separate mount tree/view. |
| **rootfs** | Filesystem tree exposed as the container's `/`. |
| **`chroot`** | Changes a process's filesystem root. |
| **`pivot_root`** | Changes the root mount inside a Mount Namespace. |
| **`/proc`** | Kernel-provided process/system interface used by tools such as `ps`. |
| **veth pair** | Two linked virtual Ethernet interfaces used to connect namespaces. |
| **cgroup** | Kernel resource-control hierarchy for processes. |

---

## 17. Final Takeaway

A container is not a single Linux feature. It is an environment assembled from several independent kernel mechanisms:

```
Namespaces + Root filesystem + Mounts + Networking + cgroups + Security controls
```

The container runtime's job is to coordinate these mechanisms automatically. Instead of manually doing:

```
unshare, mount, pivot_root, veth, ip, cgroups...
```

you eventually use something like:

```
docker run ...
```

The runtime performs and coordinates the underlying work for you.

The important lesson is not memorizing the commands. The important lesson is being able to look at a container and understand:

```
Process isolation     → PID / UTS / Network / Mount namespaces
Filesystem isolation  → rootfs + mounts + pivot_root
Resource isolation    → cgroups
Network connectivity  → veth + bridge + routing + NAT
Security              → user namespaces + capabilities + seccomp
```

That is the foundation underneath modern Linux container systems.

[↑ Back to Table of Contents](#table-of-contents)
---

## 18. Workflow Summary (Quick Reference)

```text
Create namespaces
    │
    ├── PID
    ├── UTS
    ├── Mount
    └── Network
    │
    ▼
Prepare rootfs
    │
    └── BusyBox
    │
    ▼
Make mounts private
    │
    ▼
Prepare rootfs mount
    │
    ▼
pivot_root
    │
    ├── new rootfs becomes /
    └── old root moves to /oldroot
    │
    ▼
Unmount old root
    │
    ▼
Mount /proc
    │
    ▼
Configure hostname
    │
    ▼
Configure networking
    │
    ├── veth pair
    ├── IP addresses
    └── connectivity test
    │
    ▼
Attach process to cgroup
    │
    ├── memory limit
    └── CPU limit
    │
    ▼
Verify everything
    │
    ▼
Clean up
```
