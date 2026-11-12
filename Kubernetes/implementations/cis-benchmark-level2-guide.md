# Kubernetes CIS Benchmark Level 2 Hardening Guide

- **Target Compliance:** CIS Kubernetes Benchmark Level 2 (Target: ≥ 80%)
- **Distribution:** RKE2 (Rancher Kubernetes Engine 2)
- **Container Runtime:** Containerd (embedded)
- **CNI / Ingress:** Cilium (eBPF mode, Kube-Proxy Replacement, Gateway API)
- **Base OS:** Ubuntu Linux

---

## Phase 1 — OS Kernel Parameters & RKE2 CIS Profile

- ### 1. Host Kernel Preparation (Execute on Master & Worker nodes)

    Add the required CIS sysctl settings to `/etc/sysctl.d/90-kubelet.conf`:

    ```bash
    sudo tee /etc/sysctl.d/90-kubelet.conf << 'EOF'
    vm.panic_on_oom=0
    vm.overcommit_memory=1
    kernel.panic=10
    kernel.panic_on_oops=1
    kernel.keys.root_maxkeys=1000000
    kernel.keys.root_maxbytes=25000000
    fs.inotify.max_user_watches=524288
    fs.inotify.max_user_instances=8192
    EOF

    sudo sysctl --system
    ```

    > **`sysctl`** is a Linux interface used to inspect and modify kernel parameters at runtime (without rebooting). The file `/etc/sysctl.d/90-kubelet.conf` persists these settings across reboots, and `sysctl --system` immediately loads and applies all config files from disk into the running Linux kernel.
    >
    > ### Parameter Breakdown
    >
    > - **`vm.panic_on_oom=0`**
    > 
    >   Tells the kernel not to crash (panic) the entire OS when system memory runs out. Instead, the kernel’s Out-Of-Memory (OOM) killer selectively terminates the single culprit process/container, keeping the node alive.
    > 
    > - **`vm.overcommit_memory=1`**
    > 
    >   Instructs the kernel to always allow memory allocations (overcommit), assuming processes won’t use all allocated memory at once. Kubelet mandates this setting so container memory requests and limits function predictably.
    > 
    > - **`kernel.panic=10`**
    > 
    >   If an unrecoverable kernel panic does happen, the system waits 10 seconds before automatically rebooting itself (preventing the node from hanging indefinitely).
    > 
    > - **`kernel.panic_on_oops=1`**
    > 
    >   Forces a full kernel panic if an “oops” (a serious internal kernel error or fault) occurs, triggering the automatic reboot cycle rather than continuing to run in an unstable, potentially corrupted state.
    > 
    > - **`kernel.keys.root_maxkeys=1000000`**
    > 
    >   Increases the maximum number of cryptographic keys and keyrings the root user (and container runtimes) can store in kernel memory.
    > 
    > - **`kernel.keys.root_maxbytes=25000000`**
    > 
    >   Raises the total memory ceiling (bytes) allocated to store the root user’s kernel keys, preventing quota exhaustion when running hundreds of containers.
    > 
    > - **`fs.inotify.max_user_watches=524288`**
    > 
    >   Increases the number of files and directories that processes can monitor for real-time changes via inotify (crucial for Kubelet, Fluentbit, and container volume mounts).
    > 
    > - **`fs.inotify.max_user_instances=8192`**
    > 
    >   Increases the maximum number of distinct inotify listeners a single user ID can spawn, preventing “too many open files / listeners” errors under heavy container density.
    >
    > ### Kernel Parameter
    >
    > Think of the Linux kernel as the core engine of your operating system—the bridge between your hardware and your software.
    >
    > Kernel parameters (often modified via sysctl) are the adjustable tuning dials and configuration settings for that engine. They dictate how the core operating system handles memory, networking, security limits, file monitoring, and system crashes.
    > 
    > **Why do they matter for Kubernetes?**
    >
    > Containers do not have their own kernels; they share the host operating system’s kernel. Because Kubernetes runs heavy, multi-tenant, and demanding workloads, the default Linux kernel settings (which are meant for standard desktop or generic server use) will bottleneck or crash a K8s cluster.
    > 
    > Adjusting kernel parameters allows you to:
    > 
    > - **Prevent Node Crashes:** Tell the kernel how to handle low memory (`vm.*`) or hardware faults (kernel.panic).
    > - **Support High Scale:** Raise resource limits for file monitoring (`fs.inotify.*`) and security keys (`kernel.keys.*`) so containers don’t fail when handling thousands of files and connections.
    > - **Enforce Security & Compliance:** Fulfill strict CIS Benchmark rules (like `protectKernelDefaults`) to ensure the underlying OS cannot be easily compromised by rogue containers.

    > ### Kernel Panic
    >
    > A kernel panic occurs when the Linux kernel encounters an unrecoverable error and cannot safely continue operating.
    >
    > **Think of it this way:** the kernel is responsible for managing hardware, memory, processes, and system resources. If it detects a critical condition that it cannot recover from, it may stop the system to prevent further damage or unpredictable behavior.
    >
    > **Common causes of kernel panic**
    >
    > - **Hardware failures**
    >
    >   Faulty hardware can cause the kernel to encounter unexpected conditions.
    >   
    >   Examples:
    >   - **Faulty RAM**: Memory corruption can cause the kernel to read incorrect data or execute corrupted instructions.
    >   - **CPU problems**: Hardware errors or CPU malfunctions.
    >   - **Disk failures**: Critical filesystem or I/O errors, especially when the root filesystem becomes inaccessible.
    >   - **Motherboard problems**: Hardware communication failures.
    >
    > - **Kernel bugs**
    >
    >   The kernel itself can contain programming bugs.
    >   
    >   Examples:
    >   - Null pointer dereference in kernel space.
    >   - Invalid memory access.
    >   - Use-after-free errors.
    >   - Memory corruption.
    >   - Race conditions in kernel code.
    >
    >   For example, imagine a kernel function tries to access a structure through a NULL pointer:
    >
    >   ```c
    >   struct device *dev = NULL;
    >   dev->status = 1;
    >   ```
    >
    >   This can cause a kernel fault. Depending on the context and whether the kernel can recover, it may result in a kernel panic.
    >
    > - **Faulty kernel modules and drivers**
    >
    >   This is a particularly important cause in Linux systems.
    >   
    >   Kernel modules run in **kernel space**, meaning they have significant privileges and can directly interact with kernel memory.
    > 
    >   Examples:
    >   - A buggy NVIDIA driver.
    >   - An incompatible third-party kernel module.
    >   - A faulty filesystem module.
    >   - A network driver causing memory corruption.
    >
    >   A faulty module can corrupt kernel memory or cause an unrecoverable exception.
    >
    > - **Filesystem and storage problems**
    >
    >   Examples:
    >   - Corruption of critical filesystem metadata.
    >   - Failure to access the root filesystem.
    >   - Storage controller failures.
    >   - Problems with filesystem drivers.
    >
    >   One particularly interesting example is when Linux cannot mount its root filesystem during boot. You might see:
    >   ```
    >   VFS: Unable to mount root fs on unknown-block(0,0)
    >   ```
    > 
    >   This can result in a kernel panic because Linux cannot continue booting without its root filesystem.
    >
    > - **Kernel configuration and boot problems**
    >
    >   Incorrect kernel configuration or boot parameters can also cause problems.
    > 
    >   Examples:
    >   - Incorrect `root=` kernel parameter.
    >   - Missing storage drivers in the initramfs.
    >   - Incompatible kernel and initramfs versions.
    >   - Incorrect kernel configuration for the hardware.
    >
    >   Not every boot configuration problem causes a kernel panic, though. Some result in other boot failures.
    >
    > ### What happens internally during a kernel panic?
    >
    > A simplified sequence:
    >
    > ```
    > ┌─────────────────────────────────────────────────────────────────┐
    > |┌───────────────────────────────────────────────────────────────┐|
    > ||                  1. Critical error occurs                     ||
    > ||     Hardware fault, memory corruption, kernel bug, etc.       ||
    > |└───────────────────────────────────────────────────────────────┘|
    > |                               ↓                                 |
    > |┌───────────────────────────────────────────────────────────────┐|
    > ||        2. Kernel detects an unrecoverable condition           ||
    > ||        The kernel cannot safely continue execution.           ||
    > |└───────────────────────────────────────────────────────────────┘|
    > |                               ↓                                 |
    > |┌───────────────────────────────────────────────────────────────┐|
    > ||                      3. Kernel panic                          ||
    > ||                Normal kernel execution stops.                 ||
    > |└───────────────────────────────────────────────────────────────┘|
    > |                               ↓                                 |
    > |┌───────────────────────────────────────────────────────────────┐|
    > ||                 4. System stops or reboots                    ||
    > ||     Depending on kernel configuration and panic settings.     ||
    > |└───────────────────────────────────────────────────────────────┘|
    > └─────────────────────────────────────────────────────────────────┘
    > ```
    > 
    > For example, Linux can be configured to reboot automatically five seconds after a panic:
    >
    > ```
    > sysctl kernel.panic
    > ```
    > 
    > A value of 5 means reboot five seconds after a panic. A value of 0 means don't automatically reboot. You can configure it with:
    > ```
    > sysctl -w kernel.panic=5
    > ```
    > 
    > 3. How do you troubleshoot a kernel panic?
    > There are several useful sources of diagnostic information.
    >
    > | Command / tool | Purpose |
    > | --- | --- |
    > | **`journalctl -k -b -1`** | Kernel logs from the previous boot |
    > | **`dmesg`** | Kernel ring buffer messages |
    > | **`journalctl -k`** | Kernel messages from the current boot |
    > | **`kdump`** | Capture a memory dump after a kernel crash |
    > | **`crash`** | Analyze a kernel crash dump |
    > 
    > > One particularly important concept for Linux administration is **kdump**.
    > >
    > > When a kernel panics, the normal kernel might no longer be capable of recording useful diagnostic information.
    > >
    > > **`kdump`** uses a separate reserved memory area to boot a capture kernel and collect a crash dump (usually a `vmcore`). You can then analyze that dump to investigate what caused the panic.
    > 
    > ### Kernel panic vs. segmentation fault
    >
    > Don't confuse these two:
    >
    > | Characteristic | Segmentation fault | Kernel panic |
    > | --- | --- | --- |
    > | Where does it occur? | Usually user space | Kernel space |
    > | Scope | Usually one process | Entire operating system |
    > | Typical cause | Invalid memory access by a process | Unrecoverable kernel error |
    > | System continues? | Usually yes | Normal operation stops |
    > 
    > > **Important:** A segmentation fault in a normal user-space application does not normally cause a kernel panic. Linux isolates user-space processes from kernel space.