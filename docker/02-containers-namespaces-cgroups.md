# Containers Under the Hood: Namespaces and cgroups

## Overview

Docker is not magic; it is an ergonomic wrapper around native Linux kernel features that have existed for over a decade. The term "container" does not actually exist in the Linux kernel source code. What we call a container is simply a standard Linux process (just like `sshd` or `java`) that has been wrapped in two critical kernel isolation primitives: **Namespaces** (what the process can *see*) and **Control Groups (cgroups)** (what the process can *use*).

For a Senior Platform Engineer or DevOps Architect in a highly regulated enterprise banking environment, understanding these fundamental building blocks is non-negotiable. If you do not understand how Namespaces segment network interfaces (preventing a compromised container from sniffing host traffic) or how cgroups trigger the OOM (Out Of Memory) Killer, you cannot effectively debug production incidents or secure a Kubernetes cluster. Interviewers for Staff/Principal roles will bypass Docker CLI questions entirely and dive straight into how the kernel achieves process isolation.

---

## Foundational Concepts

### What is a Linux Namespace?
Namespaces partition kernel resources such that one set of processes sees one set of resources, while another set of processes sees a different set. They provide the illusion to a containerized process that it is the only process running on a fully dedicated operating system.

When `runc` (the OCI runtime) starts a container, it traditionally uses the robust Linux `clone()` system call with specific namespace flags.

### What are cgroups (Control Groups)?
While Namespaces provide visibility isolation, they do not provide resource confinement. A containerized process could still consume 100% of the physical host's RAM, starving the host OS and causing a catastrophic system crash.

cgroups are a Linux kernel feature that limits, accounts for, and isolates the resource usage (CPU, memory, disk I/O, network) of a collection of processes. If Namespaces build the walls of the sandbox, cgroups define how much sand is inside.

---

## Technical Deep Dive: The 6 Core Namespaces

To fully isolate a process, Docker utilizes six standard Linux Namespaces (plus newer ones like cgroup namespaces). 

### 1. PID Namespace (Process ID)
*   **Flag**: `CLONE_NEWPID`
*   **The Problem**: In a standard Linux OS, process IDs are flat. If a container shares the host's PID space, an attacker inside the container could see the `java` and `dockerd` processes running on the host and send them `kill -9` signals.
*   **The Solution**: The PID namespace gives processes an independent set of PIDs from other namespaces. The first process created in a new PID namespace is assigned PID 1. Inside the container, your Node.js app thinks it is PID 1. On the physical host OS, it might actually be running as PID 14502. 

### 2. NET Namespace (Network)
*   **Flag**: `CLONE_NEWNET`
*   **The Problem**: If a container binds to port 80, no other process on the host could bind to port 80. Furthermore, the container could sniff (`tcpdump`) all network traffic traversing the host's physical NIC (`eth0`).
*   **The Solution**: This namespace gives the container its own distinct network stack: its own IP addresses, routing tables (`iptables`), firewall rules, `/etc/resolv.conf`, and completely hidden `veth` (virtual ethernet) interfaces. The container has no awareness of the host's `eth0` interface.

### 3. MNT Namespace (Mount)
*   **Flag**: `CLONE_NEWNS`
*   **The Problem**: A containerized process could easily traverse the directory tree up to `/`, find `/etc/shadow` on the physical host, and steal user password hashes.
*   **The Solution**: The MNT namespace creates an isolated filesystem hierarchy. Using features like `chroot` and `pivot_root`, the container is locked into a specific directory on the host's drive. When the container lists `/`, it only sees the root directory of the Docker Image, not the host.

### 4. IPC Namespace (Inter-Process Communication)
*   **Flag**: `CLONE_NEWIPC`
*   **The Problem**: Processes can communicate via shared memory segments or POSIX message queues. A malicious container could access the shared memory of another container running on the same host (e.g., scraping plaintext banking tokens from memory).
*   **The Solution**: Prevents processes in one IPC namespace from communicating with processes in another IPC namespace via SysV IPC or POSIX message queues.

### 5. UTS Namespace (UNIX Timesharing System)
*   **Flag**: `CLONE_NEWUTS`
*   **The Problem**: A process changing the system hostname would change the hostname for the entire physical server and all other containers, causing mass DNS failure.
*   **The Solution**: Allows a single physical server to have multiple hostnames and domain names. Each container can set its own hostname (e.g., inside the container `hostname` returns the container ID, but the host remains `aws-worker-node-1`).

### 6. USER Namespace (User ID)
*   **Flag**: `CLONE_NEWUSER`
*   **Overview**: Perhaps the most profound security feature. It allows a process to have privilege (root) *inside* the namespace, while remaining an unprivileged standard user *outside* the namespace (on the host). 
*   **Implementation**: A classic Docker container runs as `root` (UID 0). If an attacker escapes the container, they are `root` on the host server. With User Namespace Remapping configured in `daemon.json`, Docker maps the container's UID 0 to a high-numbered, unprivileged host UID (e.g., 100000). If the attacker escapes, they escape as a nobody user with zero host privileges.

---

## Technical Deep Dive: Cgroups (v1 vs v2)

### cgroup v1 Architecture
Historically, cgroups v1 structured resources hierarchically but allowed different resource controllers (CPU, memory, blkio) to have completely independent, unlinked hierarchies. This resulted in a tangled, chaotic mess of directories inside `/sys/fs/cgroup/` when running hundreds of containers, leading to race conditions and synchronization nightmares in the kernel.

### cgroup v2 Architecture
The modern Linux kernel standard (enabled by default in modern distributions and utilized by Docker 20.10+). cgroups v2 introduced a **Unified Hierarchy**. 
*   A process is assigned to exactly one cgroup in a single directory tree structure.
*   All resource controllers (CPU, memory, io) are applied to that single process group simultaneously. 
*   It introduces safety: you can more accurately monitor metrics, and crucially, it allows rootless containers (via Podman/Docker Rootless) to seamlessly delegate resource constraints without requiring raw host root access.

### Memory Limits and the OOM Killer
When you apply `docker run --memory="512m"`, Docker writes `512m` into the container's `memory.max` file inside the cgroup v2 unified hierarchy (`/sys/fs/cgroup/docker/<container-id>/`).
*   The kernel rigorously tracks every page of memory allocated by that process group.
*   If the application (e.g., a JVM with a memory leak) attempts to allocate byte 536,870,913 (exceeding 512MB), the kernel intervenes immediately. 
*   The kernel's Out-Of-Memory (OOM) Killer sends a completely un-catchable `SIGKILL` (signal 9) directly to the process. 
*   The container instantly dies with an `Exit Code 137` (`128 + 9 = 137`).

---

## Visual Representations

### The Anatomy of Container Isolation

```mermaid
graph TD
    classDef host fill:#bbdefb,stroke:#1976d2,stroke-width:2px;
    classDef namespace fill:#c8e6c9,stroke:#388e3c,stroke-width:2px;
    classDef cgroup fill:#ffe0b2,stroke:#f57c00,stroke-width:2px;
    classDef process fill:#f5f5f5,stroke:#9e9e9e,stroke-width:1px;

    subgraph Host OS [Physical Server / Linux Kernel]
        direction BT
        HostProcess(Host Process: dockerd\nPID: 1240)
        
        subgraph Container Sandbox [Containerized Java Application]
            direction BT
            
            subgraph Namespaces [Visibility Isolation (What it sees)]
                PIDNS[PID: Thinks it is PID 1]
                NETNS[NET: Isolated eth0 / 172.17.0.2]
                MNTNS[MNT: Chrooted to Image Filesystem]
                USERNS[USER: Thinks it is Root]
            end
            
            subgraph Cgroups [Resource Isolation (What it uses)]
                CPUCG[CPU: Capped at 2.0 cores]
                MEMCG[MEM: Capped at 2GB]
            end
            
            AppProc(App Process: java\nHost PID: 8521)
            
            PIDNS -. isolates .-> AppProc
            NETNS -. isolates .-> AppProc
            MNTNS -. isolates .-> AppProc
            USERNS -. isolates .-> AppProc
            
            CPUCG -. throttles .-> AppProc
            MEMCG -. OOM Kills .-> AppProc
        end
    end
    
    class Host OS host;
    class PIDNS,NETNS,MNTNS,USERNS namespace;
    class CPUCG,MEMCG cgroup;
    class HostProcess,AppProc process;
```

---

## Interview Questions & Model Answers

### Q1: Is a container technically a Virtual Machine? How does the kernel view a container?
**Model Answer**: A container is fundamentally opposed to a Virtual Machine. A VM utilizes a hypervisor to virtualize silicon hardware, booting an entirely new guest operating system kernel on top of it. To the host Linux kernel, a VM is merely a single massive, opaque block of executing memory and CPU cycles (like the `qemu-kvm` process). 
Conversely, the host kernel is intimately aware of containers. To the kernel, a container is simply an ordinary native Linux process (like a web server or database) running directly on the host kernel. The "container" abstraction is formed when the kernel applies Namespaces to hide the rest of the host's processes/network from it, and applies cgroups to restrict its hardware consumption.

### Q2: What happens internally when a Java container configured with `--memory=2g` attempts to consume 3GB of RAM? How does Docker stop it?
**Model Answer**: Docker itself does not stop it. Docker translates the `--memory=2g` flag into the native Linux `cgroups` API, writing the 2GB limit into the specific process control group directory mapped to that container (e.g., `memory.max` in cgroup v2). When the JVM attempts internal allocation bypassing the 2GB threshold, the Linux kernel's accounting system detects the violation. The kernel invokes the Out-Of-Memory (OOM) Killer. The kernel sends a `SIGKILL` (Signal 9) to the primary container process. The process is instantly violently terminated, resulting in Docker logging the container's death with Exit Code 137.

### Q3: An attacker exploits an RCE vulnerability in an NGINX container. The container was run natively without security hardening. The attacker opens a shell. They type `ps aux`. What do they see, and what stops them from killing the host's `dockerd` process?
**Model Answer**: Because the container was launched with an isolated **PID Namespace** (`CLONE_NEWPID`), when the attacker executes `ps aux`, they do not see the massive process tree of the host. They only see the processes localized explicitly within that namespace—meaning they see `nginx` running as PID 1, and their own malicious shell as PID 2. Because the host's `dockerd` process (perhaps Host PID 1400) does not exist within the attacker's PID namespace, the kernel outright denies the attacker's ability to signal, interact with, or even acknowledge the existence of the daemon.

### Q4: Why is running a container as `root` universally banned in financial CI/CD pipelines, and what kernel feature mitigates this if root is required?
**Model Answer**: Inside the container, user `root` has UID 0. Crucially, without explicit configuration, UID 0 inside the container maps exactly to the omnipotent UID 0 physical `root` on the host server. A container breakout vulnerability (e.g., via a mismatched kernel mount or a privileged escalation exploit) allows the attacker to drop onto the host filesystem with absolute authority, capable of exfiltrating SSL keys or formatting the drives. 
The mitigation is employing the **USER Namespace** (`userns-remap`). This explicitly partitions UIDs. We can map the container's internal UID 0 (`root`) to the host's UID 200000 (an unprivileged nobody). Thus, the application behaves dynamically as root conceptually inside its isolated namespace (e.g., it can bind to port 80), but if it escapes, the Linux kernel perceives it as UID 200000 and restricts access to the physical filesystem via standard Linux DAC permissions.

### Q5: In enterprise banking, we use Kubernetes (K8s) to schedule massive fleets of microservices. How do K8s "Pods" relate to Namespaces and cgroups?
**Model Answer**: A Kubernetes Pod is the atomic unit of scheduling, consisting of one or more tightly coupled containers. At the kernel level, a Pod is simply a shared set of Namespaces seamlessly draped across multiple distinct processes. 
Kubernetes initiates a "Pause" container for every Pod. This invisible container exists solely to initialize and hold open the Network (NET) and IPC namespaces. Subsequent application containers (like a Java web app and its sidecar Envoy proxy) are executed and artificially injected into the *exact same* NET and IPC namespaces as the Pause container. This is structurally why two containers in the same Pod can communicate blazingly fast over `localhost` (they share the exact same loopback interface and `veth` endpoint) and can access identical POSIX shared memory segments. Cgroups, however, are applied specifically and independently per-container to enforce pod-level CPU/Memory requests and limits.

---

## Command Line Forensics

How to prove containers are just processes:

1.  Run a container: `docker run -d --name my-nginx nginx`
2.  View the host processes: `ps -ef | grep nginx`
    *   *Observation*: You will see the native `nginx` master process running directly on the host, with a high PID (e.g., 24901).
3.  Inspect its specific cgroup: `cat /proc/24901/cgroup`
    *   *Observation*: The kernel confirms this process is tied to a specific Docker cgroup directory matching the container ID.
4.  Inspect its Namespaces: `ls -l /proc/24901/ns/`
    *   *Observation*: You will explicitly see filesystem links to the kernel's distinct `net`, `mnt`, `pid`, and `ipc` inodes, confirming the isolation wrapping the process.

## Key Takeaways

1.  **Containers do not exist in the kernel**. They are an orchestration of Namespaces and Cgroups around a standard Linux process.
2.  **Namespaces restrict Visibility** (PID, NET, MNT, IPC, UTS, USER).
3.  **Cgroups restrict Resources** (Memory, CPU percentages, Block I/O).
4.  User Namespace Remapping is the holy grail of mitigating container breakout attacks.

## Further Reading
*   [Linux Namespaces man page](https://man7.org/linux/man-pages/man7/namespaces.7.html)
*   [Linux cgroups man page](https://man7.org/linux/man-pages/man7/cgroups.7.html)
*   [Docker Security: User Namespaces](https://docs.docker.com/engine/security/userns-remap/)
