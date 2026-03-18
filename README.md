# Linux Kernel eBPF VFS Redactor
**Dynamic In-Flight File Redaction via Virtual File System Interception and Custom eBPF Subsystems**

This repository contains a core Linux Kernel patch (developed against 6.x) that implements a transparent, on-the-fly file content redaction mechanism. Instead of relying on user-space proxies, this system operates directly within the kernel's Virtual File System (VFS), utilizing a custom-engineered eBPF program type to evaluate and mutate buffers before they reach user space.

## Architectural Highlights
This is not a standard BPF tracing script. It involves deep modifications to the kernel source tree, custom syscalls, and extension of the eBPF verifier.

All the modifications can be found in `solution.patch` file.

* **Custom eBPF Program Type (`BPF_PROG_TYPE_REDACTOR`):** Engineered a native eBPF program type and attach mechanism (`BPF_REDACTOR`) deeply integrated with the kernel verifier (`redactor_verifier_ops`). This enforces strict memory boundaries and access rights for BPF programs interacting with the custom `redactor_ctx`.
* **Deep VFS Interception:** Injected hooks directly into `fs/open.c` (`do_sys_openat2`) for BPF-driven redaction policy evaluation, and `fs/read_write.c` (`vfs_read`, `do_iter_readv_writev`) for payload mutation.
* **Vectorized I/O (iov_iter) Mastery:** The redaction engine correctly handles complex, multi-buffer vectorized reads. By utilizing `iov_iter_save_state` and `iov_iter_restore`, the system iterates over dynamically sized buffer chunks, applying BPF mutations across fragmented memory segments without corrupting the original iterator.
* **Lockless Concurrency (SMP):** Tracks global redaction statistics directly within the `struct file` object using memory-barrier-enforced atomic operations (`atomic_long_read_acquire`, `atomic_long_add`), ensuring zero-overhead SMP scalability without spinlock contention.
* **Custom Syscalls:** Introduced low-level system calls (`sys_count_redactions`, `sys_reset_redactions`) mapped directly into the x86 architecture tables to query the lockless atomic counters from user-space.

## Technical Debt & Known Limitations
This patch was developed as an advanced academic research project into kernel internals. Consequently, certain trade-offs and architectural violations exist:

* **Kernel-to-User Memory Unsafe Access:** The custom BPF helpers (`bpf_copy_to_buffer`, `bpf_copy_from_buffer`) directly invoke `memmove` on `__user` memory pointers. While functional in controlled test environments, this bypasses standard kernel safety primitives (`copy_to_user`, `copy_from_user`). In a production kernel, this would violate Supervisor Mode Access Prevention (SMAP) and trigger a Kernel Panic during a page fault on swapped-out user memory.
* **Hardcoupled VFS Hooks:** The interception logic is hardcoded directly into the VFS core (`fs/read_write.c`). A production-grade implementation would aim to avoid modifying the core kernel tree.

## Tech Stack
* **Domain:** Linux Kernel Space, eBPF, Virtual File System (VFS).
* **Language:** C (GNU C11, Kernel Coding Standard).
* **Key Mechanics:** BPF Verifier, Vectorized I/O (`iov_iter`), Atomic SMP primitives, System Call implementation.

## Other details
The implementation behaves correctly when run against the evaluation tests within the `z2-tst-public` directory.

The patch was generated using helper scripts (located in the `scripts` directory) that facilitated kernel tree navigation and patch creation.
