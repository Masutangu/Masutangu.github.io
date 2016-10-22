---
layout: post
date: 2016-07-07T15:02:11+08:00
title: Linux 内核设计与实现 
category: 读书笔记
---

# Chapter1. Introduction to the Linux Kernel

* A handful of characteristics of Unix are at the core of its strength.
  * First, Unix is simple:Whereas some operating systems implement thousands of system calls and have unclear design goals, Unix systems implement only hundreds of system calls and have a straightforward, even basic, design.
  * Second, in Unix, everything is a file.2 This simpli- fies the manipulation of data and devices into a set of core system calls:open(),read(), write(), lseek(), and close().
  * Third, the Unix kernel and related system utilities are written in C—a property that gives Unix its amazing portability to diverse hardware architectures and accessibility to a wide range of developers. 
  * Fourth, Unix has fast process creation time and the unique fork() system call. 
  * Finally, Unix provides simple yet robust interprocess communication (IPC) primitives that, when coupled with the fast process creation time, enable the creation of simple programs that do one thing and do it well. These single-purpose programs can be strung together to accomplish tasks of increasing com- plexity. **Unix systems thus exhibit clean layering, with a strong separation between policy and mechanism.** （提供机制，而不是策略。）

* Techni- cally speaking, and in this book, the operating system is considered the parts of the system responsible for basic use and administration. This includes the kernel and device drivers, boot loader, command shell or other user interface, and basic file and system utilities. 

* Typical components of a kernel are interrupt handlers to service interrupt requests, a scheduler to share processor time among multiple processes, a memory management system to manage process address spaces, and system services such as networking and interprocess communication.

* On modern systems with protected memory management units, the kernel typically resides in an elevated system state compared to normal user applications. This includes a protected memory space and full access to the hardware. This system state and memory space is collectively referred to as **kernel-space**. Conversely, user applications execute in **user-space**. When executing kernel code, the system is in kernel-space executing in kernel mode. When running a regular process, the system is in user-space executing in user mode.

* Applications running on the system communicate with the kernel via system calls. An application typically calls functions in a library—for example, the C library—that in turn **rely on the system call interface to instruct the kernel to carry out tasks on the application’s behalf**. When an application executes a system call, we say that the kernel is executing **on behalf of the application**. Furthermore, the application is said to be executing a system call in **kernel-space**, and the kernel is running in **process context**.

* Nearly all architectures, including all systems that Linux supports, provide the concept of interrupts.When hardware wants to communicate with the system, it issues an interrupt that literally interrupts the processor, which in turn interrupts the kernel. A number identifies interrupts and the kernel uses this number to execute a specific interrupt handler to process and respond to the interrupt.

  To provide synchronization, the kernel can disable interrupts—either all interrupts or just one specific interrupt number. In many operating systems, including Linux, **the interrupt handlers do not run in a process context**. Instead, they run in a special **interrupt context** that is not associated with any process.

* In Linux, we can generalize that each processor is doing exactly one of three things at any given moment:
  * In user-space, executing user code in a process
  * In kernel-space, in process context, executing on behalf of a specific process
  * In kernel-space, in interrupt context, not associated with a process, handling an interrupt

  This list is inclusive. Even corner cases fit into one of these three activities: For example, when idle, it turns out that the kernel is executing an idle process in process context in the kernel.

* With few exceptions, a Unix kernel is typically a monolithic static binary. That is, it exists as a single, large, executable image that runs in a single address space. Unix systems typically require a system with a paged memory-management unit (MMU); this hardware enables the system to enforce memory protection and to provide a unique virtual address space to each process.

* We can divide kernels into two main schools of design: the monolithic kernel and the microkernel. 

  Monolithic kernels are the simpler design of the two. Monolithic kernels are implemented entirely as a single process running in a single address space. Consequently, such kernels typically exist on disk as sin- gle static binaries. All kernel services exist and execute in the large kernel address space. Communication within the kernel is trivial because everything runs in kernel mode in the same address space: The kernel can invoke functions directly, as a user-space application might. 

  Microkernels, on the other hand, are not implemented as a single large process. Instead, the functionality of the kernel is broken down into separate processes, usually called servers. Therefore, direct function invocation as in monolithic kernels is not possible. Instead, microkernels communicate via message passing: An interprocess communication (IPC) mechanism is built into the system, and the various servers communicate with and invoke “services” from each other by sending messages over the IPC mechanism.

  Because the IPC mechanism involves quite a bit more overhead than a trivial function call, however, and because a context switch from kernel-space to user-space or vice versa is often involved, message passing includes a latency and throughput hit not seen on mono- lithic kernels with simple function invocation. Consequently, all practical microkernel-based systems now place most or all the servers in kernel-space, to remove the overhead of fre- quent context switches and potentially enable direct function invocation. 

  Linux is a monolithic kernel; that is, the Linux kernel executes in a single address space entirely in kernel mode. Linux, however, borrows much of the good from microkernels: Linux boasts a modular design, the capability to preempt itself (called kernel preemption), support for kernel threads, and the capability to dynamically load separate binaries (kernel modules) into the kernel image. Conversely, Linux has none of the performance-sapping features that curse microkernel design: Everything runs in kernel mode, with direct function invocation— not message passing—the modus of communication.

* A handful of notable differences exist between the Linux kernel and classic Unix systems:
  * Linux supports the dynamic loading of kernel modules.Although the Linux kernel is monolithic, it can dynamically load and unload kernel code on demand.
  * Linux has symmetrical multiprocessor (SMP) support.
  * The Linux kernel is preemptive. Unlike traditional Unix variants, the Linux kernel can preempt a task even as it executes in the kernel.
  * Linux takes an interesting approach to thread support: It does not differentiate between threads and normal processes.To the kernel, all processes are the same—some just happen to share resources.
  * Linux provides an object-oriented device model with device classes, hot-pluggable events, and a user-space device filesystem.
  * Linux ignores some common Unix features that the kernel developers consider poorly designed, such as STREAMS, or standards that are impossible to cleanly implement.
  * Linux is free in every sense of the word.

# Chapter2. Getting Started with the Kernel

* The kernel source tree is divided into a number of directories. The directories in the root of the source tree, along with their descriptions, are listed as follow:

  | Directory       |                           Description                         |
  |-----------------|---------------------------------------------------------------|
  |     arch        |                   Architecture-specific source                |
  |    block        |                         Block I/O layer                       |
  |    crypto       |                           Crypto API                          |
  |  Documentation  |                    Kernel source documentation                |
  |    drivers      |                           Device drivers                      |
  |    firmware     |           Device firmware needed to use certain drivers       |
  |      fs         |            The VFS and the individual filesystems             |
  |    include      |                           Kernel headers                      |
  |     init        |                    Kernel boot and initialization             |
  |     ipc         |                   Interprocess communication code             |
  |     kernel      |               Core subsystems, such as the scheduler          |
  |      lib        |                           Helper routines                     |
  |      mm         |           Memory management subsystem and the VM              |
  |     net         |                       Networking subsystem                    |
  |    samples      |                   Sample, demonstrative code                  |
  |    scripts      |                 Scripts used to build the kernel              |
  |    security     |                       Linux Security Module                   |
  |     sound       |                           Sound subsystem                     |
  |      usr        |             Early user-space code (called initramfs)          |
  |     tools       |               Tools helpful for developing Linux              |
  |     virt        |                   Virtualization infrastructure               |

* Configuration options that control the build process are either Booleans or tristates.A Boolean option is either yes or no. Kernel features, such as CONFIG_PREEMPT, are usually Booleans.A tristate option is one of yes, no, or module.The module setting represents a con- figuration option that is set but is to be compiled as a module (that is, a separate dynamically loadable object).

* The build process also creates the file System.map in the root of the kernel source tree. It contains a symbol lookup table, mapping kernel symbols to their start addresses.This is used during debugging to translate memory addresses to function and variable names.

* The Linux kernel has several unique attributes as compared to a normal user-space application:

  * The kernel has access to neither the C library nor the standard C headers. n The kernel is coded in GNU C.
  * The kernel lacks the memory protection afforded to user-space.
  * The kernel cannot easily execute floating-point operations.
  * The kernel has a small per-process fixed-size stack.
  * Because the kernel has asynchronous interrupts, is preemptive, and supports SMP, synchronization and concurrency are major concerns within the kernel. 
  * Portability is important.

  * **No libc or Standard Headers**
    Unlike a user-space application, the kernel is not linked against the standard C library—or any other library, for that matter. The full C library—or even a decent subset of it-is too large and too inefficient for the kernel.

    Like any self-respecting Unix kernel, the Linux kernel is programmed in C. Perhaps sur- prisingly, the kernel is not programmed in strict ANSI C. Instead, where applicable, the kernel developers make use of various language extensions available in gcc:

    * Inline Functions
    Both C99 and GNU C support inline functions. An inline function is, as its name suggests, inserted inline into each function call site. An inline function is declared when the keywords static and inline are used as part of the function definition. 

    * Inline Assembly
    The gcc C compiler enables the embedding of assembly instructions in otherwise normal C functions. The asm() compiler directive is used to inline assembly code. For example, this inline assembly directive executes the x86 processor’s rdtsc instruction, which returns the value of the timestamp (tsc) register:

    ```
    unsigned int low, high;
    asm volatile("rdtsc" : "=a" (low), "=d" (high));
    /* low and high now contain the lower and upper 32-bits of the 64-bit tsc */
    ```

    * Branch Annotation
    The gcc C compiler has a built-in directive that optimizes conditional branches as either very likely taken or very unlikely taken. The compiler uses the directive to appropriately optimize the branch. The kernel wraps the directive in easy-to-use macros, likely() and unlikely().

  * **No Memory Protection**
  When a user-space application attempts an illegal memory access, the kernel can trap the error, send the SIGSEGV signal, and kill the process. If the kernel attempts an illegal memory access, however, the results are less controlled. Memory violations in the kernel result in an oops, which is a major kernel error. It should go without saying that you must not illegally access memory, such as dereferenc- ing a NULL pointer—but within the kernel, the stakes are much higher!

  Additionally, kernel memory is not pageable. Therefore, every byte of memory you consume is one less byte of available physical memory. 

  * **No (Easy) Use of Floating Point**
  When a user-space process uses floating-point instructions, the kernel manages the transi- tion from integer to floating point mode.What the kernel has to do when using floating-point instructions varies by architecture, but the kernel normally catches a trap and then initiates the transition from integer to floating point mode.

  Unlike user-space, the kernel does not have the luxury of seamless support for floating point because it cannot easily trap itself. 

  Using a floating point inside the kernel requires manually saving and restoring the floating point registers, among other possible chores. The short answer is: **Don’t do it!**

  * **Small, Fixed-Size Stack**
  User-space can get away with statically allocating many variables on the stack, including huge structures and thousand-element arrays.This behavior is legal because user-space has a large stack that can dynamically grow.

  The kernel stack is neither large nor dynamic; it is small and fixed in size.The exact size of the kernel’s stack varies by architecture. Historically, the kernel stack is two pages, which generally implies that it is 8KB on 32-bit architectures and 16KB on 64-bit archi- tectures—this size is fixed and absolute. Each process receives its own stack.

  * **Synchronization and Concurrency**
  The kernel is susceptible to race conditions. Unlike a single-threaded user-space applica- tion, a number of properties of the kernel allow for concurrent access of shared resources and thus require synchronization to prevent races. Specifically:
  