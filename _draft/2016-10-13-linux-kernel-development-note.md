---
layout: post
date: 2016-07-07T15:02:11+08:00
title: Linux 内核设计与实现 
category: 读书笔记
---

# Chapter1. Introduction to the Linux Kernel

A handful of characteristics of Unix are at the core of its strength:

* First, Unix is simple: Whereas some operating systems implement thousands of system calls and have unclear design goals, Unix systems implement only hundreds of system calls and have a straightforward, even basic, design.
* Second, in Unix, everything is a file.2 This simplifies the manipulation of data and devices into a set of core system calls: open(),read(), write(), lseek(), and close().
* Third, the Unix kernel and related system utilities are written in C—a property that gives Unix its amazing portability to diverse hardware architectures and accessibility to a wide range of developers. 
* Fourth, Unix has fast process creation time and the unique fork() system call. 
* Finally, Unix provides simple yet robust interprocess communication (IPC) primitives that, when coupled with the fast process creation time, enable the creation of simple programs that do one thing and do it well. These single-purpose programs can be strung together to accomplish tasks of increasing complexity. **Unix systems thus exhibit clean layering, with a strong separation between policy and mechanism.** （提供机制，而不是策略。）

Technically speaking, and in this book, the operating system is considered the parts of the system responsible for basic use and administration. This includes the kernel and device drivers, boot loader, command shell or other user interface, and basic file and system utilities. 

Typical components of a kernel are interrupt handlers to service interrupt requests, a scheduler to share processor time among multiple processes, a memory management system to manage process address spaces, and system services such as networking and interprocess communication.

On modern systems with protected memory management units, the kernel typically resides in an elevated system state compared to normal user applications. This includes a protected memory space and full access to the hardware. This system state and memory space is collectively referred to as **kernel-space**. Conversely, user applications execute in **user-space**. When executing kernel code, the system is in kernel-space executing in kernel mode. When running a regular process, the system is in user-space executing in user mode.

Applications running on the system communicate with the kernel via system calls. An application typically calls functions in a library—for example, the C library—that in turn **rely on the system call interface to instruct the kernel to carry out tasks on the application’s behalf**. When an application executes a system call, we say that the kernel is executing **on behalf of the application**. Furthermore, the application is said to be executing a system call in **kernel-space**, and the kernel is running in **process context**.

Nearly all architectures, including all systems that Linux supports, provide the concept of interrupts. When hardware wants to communicate with the system, it issues an interrupt that literally interrupts the processor, which in turn interrupts the kernel. A number identifies interrupts and the kernel uses this number to execute a specific interrupt handler to process and respond to the interrupt.

To provide synchronization, the kernel can disable interrupts—either all interrupts or just one specific interrupt number. In many operating systems, including Linux, **the interrupt handlers do not run in a process context**. Instead, they run in a special **interrupt context** that is not associated with any process.

In Linux, we can generalize that each processor is doing exactly one of three things at any given moment:

* In user-space, executing user code in a process
* In kernel-space, in process context, executing on behalf of a specific process
* In kernel-space, in interrupt context, not associated with a process, handling an interrupt

This list is inclusive. Even corner cases fit into one of these three activities: For example, when idle, it turns out that the kernel is executing an idle process in process context in the kernel.

With few exceptions, a Unix kernel is typically a monolithic static binary. That is, it exists as a single, large, executable image that runs in a single address space. Unix systems typically require a system with a paged memory-management unit (MMU); this hardware enables the system to enforce memory protection and to provide a unique virtual address space to each process.

We can divide kernels into two main schools of design: the monolithic kernel and the microkernel. 

Monolithic kernels are the simpler design of the two. Monolithic kernels are implemented entirely as a single process running in a single address space. Consequently, such kernels typically exist on disk as sin- gle static binaries. All kernel services exist and execute in the large kernel address space. Communication within the kernel is trivial because everything runs in kernel mode in the same address space: The kernel can invoke functions directly, as a user-space application might. 

Microkernels, on the other hand, are not implemented as a single large process. Instead, the functionality of the kernel is broken down into separate processes, usually called servers. Therefore, direct function invocation as in monolithic kernels is not possible. Instead, microkernels communicate via message passing: An interprocess communication (IPC) mechanism is built into the system, and the various servers communicate with and invoke “services” from each other by sending messages over the IPC mechanism.

Because the IPC mechanism involves quite a bit more overhead than a trivial function call, however, and because a context switch from kernel-space to user-space or vice versa is often involved, message passing includes a latency and throughput hit not seen on mono- lithic kernels with simple function invocation. Consequently, all practical microkernel-based systems now place most or all the servers in kernel-space, to remove the overhead of fre- quent context switches and potentially enable direct function invocation. 

Linux is a monolithic kernel; that is, the Linux kernel executes in a single address space entirely in kernel mode. Linux, however, borrows much of the good from microkernels: Linux boasts a modular design, the capability to preempt itself (called kernel preemption), support for kernel threads, and the capability to dynamically load separate binaries (kernel modules) into the kernel image. Conversely, Linux has none of the performance-sapping features that curse microkernel design: Everything runs in kernel mode, with direct function invocation— not message passing—the modus of communication.

A handful of notable differences exist between the Linux kernel and classic Unix systems:

* Linux supports the dynamic loading of kernel modules. Although the Linux kernel is monolithic, it can dynamically load and unload kernel code on demand.
* Linux has symmetrical multiprocessor (SMP) support.
* The Linux kernel is preemptive. Unlike traditional Unix variants, the Linux kernel can preempt a task even as it executes in the kernel.
* Linux takes an interesting approach to thread support: It does not differentiate between threads and normal processes.To the kernel, all processes are the same—some just happen to share resources.
* Linux provides an object-oriented device model with device classes, hot-pluggable events, and a user-space device filesystem.
* Linux ignores some common Unix features that the kernel developers consider poorly designed, such as STREAMS, or standards that are impossible to cleanly implement.
* Linux is free in every sense of the word.

# Chapter2. Getting Started with the Kernel

The kernel source tree is divided into a number of directories. The directories in the root of the source tree, along with their descriptions, are listed as follow:

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

Configuration options that control the build process are either Booleans or tristates.A Boolean option is either yes or no. Kernel features, such as CONFIG_PREEMPT, are usually Booleans.A tristate option is one of yes, no, or module.The module setting represents a con- figuration option that is set but is to be compiled as a module (that is, a separate dynamically loadable object).

The build process also creates the file System.map in the root of the kernel source tree. It contains a symbol lookup table, mapping kernel symbols to their start addresses.This is used during debugging to translate memory addresses to function and variable names.

The Linux kernel has several unique attributes as compared to a normal user-space application:

* The kernel has access to neither the C library nor the standard C headers. n The kernel is coded in GNU C.
* The kernel lacks the memory protection afforded to user-space.
* The kernel cannot easily execute floating-point operations.
* The kernel has a small per-process fixed-size stack.
* Because the kernel has asynchronous interrupts, is preemptive, and supports SMP, synchronization and concurrency are major concerns within the kernel. 
* Portability is important.

* **No libc or Standard Headers**
  Unlike a user-space application, the kernel is not linked against the standard C library—or any other library, for that matter. The full C library—or even a decent subset of it-is too large and too inefficient for the kernel.

  Like any self-respecting Unix kernel, the Linux kernel is programmed in C. Perhaps sur- prisingly, the kernel is not programmed in strict ANSI C. Instead, where applicable, the kernel developers make use of various language extensions available in gcc:

* **Inline Functions**
  Both C99 and GNU C support inline functions. An inline function is, as its name suggests, inserted inline into each function call site. An inline function is declared when the keywords static and inline are used as part of the function definition. 

* **Inline Assembly**
  The gcc C compiler enables the embedding of assembly instructions in otherwise normal C functions. The asm() compiler directive is used to inline assembly code. For example, this inline assembly directive executes the x86 processor’s rdtsc instruction, which returns the value of the timestamp (tsc) register:

  ```c
  unsigned int low, high;
  asm volatile("rdtsc" : "=a" (low), "=d" (high));
  /* low and high now contain the lower and upper 32-bits of the 64-bit tsc */
  ```

* **Branch Annotation**
  The gcc C compiler has a built-in directive that optimizes conditional branches as either very likely taken or very unlikely taken. The compiler uses the directive to appropriately optimize the branch. The kernel wraps the directive in easy-to-use macros, likely() and unlikely().

* **No Memory Protection**
  When a user-space application attempts an illegal memory access, the kernel can trap the error, send the SIGSEGV signal, and kill the process. If the kernel attempts an illegal memory access, however, the results are less controlled. Memory violations in the kernel result in an oops, which is a major kernel error. It should go without saying that you must not illegally access memory, such as dereferenc- ing a NULL pointer—but within the kernel, the stakes are much higher!

  Additionally, kernel memory is not pageable. Therefore, every byte of memory you consume is one less byte of available physical memory. 

* **No (Easy) Use of Floating Point**
  When a user-space process uses floating-point instructions, the kernel manages the transi- tion from integer to floating point mode.What the kernel has to do when using floating-point instructions varies by architecture, but the kernel normally catches a trap and then initiates the transition from integer to floating point mode.

  Unlike user-space, the kernel does not have the luxury of seamless support for floating point because it cannot easily trap itself. 

  Using a floating point inside the kernel requires manually saving and restoring the floating point registers, among other possible chores. The short answer is: **Don’t do it!**

* **Small, Fixed-Size Stack**
  User-space can get away with statically allocating many variables on the stack, including huge structures and thousand-element arrays. This behavior is legal because user-space has a large stack that can dynamically grow.

  The kernel stack is neither large nor dynamic; it is small and fixed in size.The exact size of the kernel’s stack varies by architecture. Historically, the kernel stack is two pages, which generally implies that it is 8KB on 32-bit architectures and 16KB on 64-bit architectures—this size is fixed and absolute. Each process receives its own stack.

* **Synchronization and Concurrency**
  The kernel is susceptible to race conditions. Unlike a single-threaded user-space applica- tion, a number of properties of the kernel allow for concurrent access of shared resources and thus require synchronization to prevent races. Specifically:

  * Linux is a preemptive multitasking operating system. Processes are scheduled and rescheduled at the whim of the kernel’s process scheduler.The kernel must synchronize between these tasks.
  * Linux supports symmetrical multiprocessing (SMP). Therefore, without proper protection, kernel code executing simultaneously on two or more processors can concurrently access the same resource.
  * Interrupts occur asynchronously with respect to the currently executing code. Therefore, without proper protection, an interrupt can occur in the midst of accessing a resource, and the interrupt handler can then access the same resource.
  * The Linux kernel is preemptive. Therefore, without protection, kernel code can be preempted in favor of different code that then accesses the same resource.

Typical solutions to race conditions include spinlocks and semaphores.

* **Importance of Portability**
  A handful of rules—such as remain endian neutral, be 64-bit clean, do not assume the word or page size, and so on—go a long way.

# Chapter3. Process Management

## The Process
A process is a program (object code stored on some media) in the midst of execution. Processes are, however, more than just the executing program code (often called the text section in Unix).They also include a set of resources such as open files and pending signals, internal kernel data, processor state, a memory address space with one or more memory mappings, one or more threads of execution, and a data section containing global variables.

Threads of execution, often shortened to threads, are the objects of activity within the process. **Each thread includes a unique program counter, process stack, and set of processor registers.** The kernel schedules individual threads, not processes. To Linux, a thread is just a special kind of process.

On modern operating systems, processes provide two virtualizations: a virtualized processor and virtual memory. The virtual processor gives the process the illusion that it alone monopolizes the system, despite possibly sharing the processor among hundreds of other processes. .Virtual memory lets the process allocate and manage memory as if it alone owned all the memory in the system. Interestingly, note that **threads share the virtual memory abstraction, whereas each receives its own virtualized processor.**

A process begins its life when, not surprisingly, it is created. In Linux, this occurs by means of the fork() system call. In contemporary Linux kernels, fork() is actually implemented via the clone() system call.

## Process Descriptor and the Task Structure
The kernel stores the list of processes in a circular doubly linked list called the **task list**. Each element in the task list is a process descriptor of the type ```struct task_struct```, which is defined in <linux/sched.h>. The process descriptor contains the data that describes the executing program—open files, the process’s address space, pending signals, the process’s state, and much more.

### Allocating the Process Descriptor
The ```task_struct``` structure is allocated via the slab allocator to provide object reuse and cache coloring. With the process descriptor now dynamically created via the slab allocator, a new structure, struct thread_info, was created that again lives at the bottom of the stack (for stacks that grow down) and at the top of the stack (for stacks that grow up).

The thread_info structure is defined on x86 in <asm/thread_info.h> as

```c
struct thread_info { 
    struct task_struct *task;
    struct exec_domain *exec_domain;
    __u32 flags;
    __u32 status;
    __u32 cpu; 
    int preempt_count;
    mm_segment_t addr_limit; 
    struct restart_block restart_block; 
    void *sysenter_return; 
    int uaccess_err;
}
```

Each task’s thread\_info structure is allocated at the end of its stack.The task element of the structure is a pointer to the task’s actual task_struct.

### Storing the Process Descriptor
The system identifies processes by a unique process identification value or PID. The PID is a numerical value represented by the opaque type4 pid_t, which is typically an int.

Inside the kernel, tasks are typically referenced directly by a pointer to their task_struct structure. In fact, most kernel code that deals with processes works directly with struct task_struct. Consequently, it is useful to be able to quickly look up the process descriptor of the currently executing task, which is done via the current macro. This macro must be independently implemented by each architecture. Some architectures save a pointer to the task_struct structure of the currently running process in a register, enabling for efficient access. Other architectures, such as x86 (which has few registers to waste), make use of the fact that struct thread_info is stored on the kernel stack to calculate the location of thread\_info and subsequently the task\_struct.

On x86, current is calculated by masking out the 13 least-significant bits of the stack pointer to obtain the thread\_info structure.This is done by the current\_thread\_info() function.The assembly is shown here:

```
movl $-8192, %eax 
andl %esp, %eax
```

This assumes that the stack size is 8KB.When 4KB stacks are enabled, 4096 is used in lieu of 8192.

### Process State
The state field of the process descriptor describes the current condition of the process. Each process on the system is in exactly one of five different states.This value is represented by one of five flags:

* TASK_RUNNING
  The process is runnable; it is either currently running or on a runqueue waiting to run. This is the only possible state for a process executing in user-space; it can also apply to a process in kernel-space that is actively running.

* TASK_INTERRUPTIBLE
  The process is sleeping (that is, it is blocked), waiting for some condition to exist.When this condition exists, the kernel sets the process’s state to TASK_RUNNING. The process also awakes prematurely and becomes runnable if it receives a signal.

* TASK_UNINTERRUPTIBLE
  This state is identical to TASK_INTERRUPTIBLE except that it does not wake up and become runnable if it receives a signal. This is used in situations where the process must wait without interruption or when the event is expected to occur quite quickly.

* __TASK_TRACED
  The process is being traced by another process, such as a debugger, via ptrace.

* __TASK_STOPPED
  Process execution has stopped; the task is not running nor is it eligible to run. This occurs if the task receives the SIGSTOP, SIGTSTP, SIGTTIN, or SIGTTOU signal or if it receives any signal while it is being debugged.

### Manipulating the Current Process State

Kernel code often needs to change a process’s state.The preferred mechanism is using

```c
set_task_state(task, state); /* set task ‘task’ to state ‘state’ */
```

This function sets the given task to the given state. If applicable, it also provides a memory barrier (内存屏障) to force ordering on other processors. (This is only needed on SMP sys- tems.) Otherwise, it is equivalent to:

```c
task->state = state;
```

### Process Context
Normal program execution occurs in user-space. When a program executes a system call or triggers an exception, it enters kernel-space. At this point, the kernel is said to be “executing on behalf of the process” and is in process context. When in process context, the current macro is valid.

System calls and exception handlers are well-defined interfaces into the kernel. A process can begin executing in kernel-space only through one of these interfaces—all access to the kernel is through these interfaces.

Other than process context there is interrupt context. In interrupt context, the system is not running on behalf of a process but is executing an interrupt handler. No process is tied to interrupt handlers.

### The Process Family Tree
All processes are descendants of the init process, whose PID is one. The kernel starts init in the last step of the boot process. The init process, in turn, reads the system initscripts and executes more programs, eventually completing the boot process.

Every process on the system has exactly one parent. Likewise, every process has zero or more children. Processes that are all direct children of the same parent are called siblings. The relationship between processes is stored in the process descriptor. 

Each task\_struct has a pointer to the parent’s task\_struct, named parent, and a list of children, named children. The init task’s process descriptor is statically allocated as init\_task.

It is desirable simply to iterate over all processes in the system. This is easy because the task list is a circular, doubly linked list.

## Process Creation
Most operating systems implement a spawn mechanism to create a new process in a new address space, read in an executable, and begin executing it. Unix takes the unusual approach of separating these steps into two distinct functions: fork()and exec().

The first, fork(), creates a child process that is a copy of the current task. It differs from the parent only in its PID (which is unique), its PPID (parent’s PID, which is set to the original process), and certain resources and statistics, such as pending signals, which are not inherited. The second function,exec(), loads a new executable into the address space and begins executing it.

### Copy-on-Write
In Linux, fork() is implemented through the use of copy-on-write pages. Copy-on-write (or COW) is a technique to delay or altogether prevent copying of the data. The data, however, is marked in such a way that if it is written to, a duplicate is made and each process receives a unique copy.

The only overhead incurred by fork() is the duplication of the parent’s page tables and the creation of a unique process descriptor for the child.

### Forking
Linux implements fork() via the clone() system call. This call takes a series of flags that specify which resources, if any, the parent and child process should share. The fork(), vfork(), and \_\_clone() library calls all invoke the clone() system call with the requisite flags.The clone() system call, in turn, calls do\_fork().

The bulk of the work in forking is handled by do\_fork(), which is defined in kernel/fork.c.This function calls copy\_process() and then starts the process running. The interesting work is done by copy\_process():

* It calls dup_task_struct(), which creates a new kernel stack, thread_info struc- ture, and task_struct for the new process.The new values are identical to those of the current task.At this point,the child and parent process descriptors are identical.
* It then checks that the new child will not exceed the resource limits on the num- ber of processes for the current user.
* The child needs to differentiate itself from its parent.Various members of the process descriptor are cleared or set to initial values. Members of the process descriptor not inherited are primarily statistically information.The bulk of the val- ues in task_struct remain unchanged.
* The child’s state is set to TASK_UNINTERRUPTIBLE to ensure that it does not yet run.
* copy\_process() calls copy\_flags() to update the flags member of the task\_struct.The PF\_SUPERPRIV flag, which denotes whether a task used superuser privileges, is cleared.The PF\_FORKNOEXEC flag, which denotes a process that has not called exec(), is set.
* It calls alloc\_pid() to assign an available PID to the new task.
* Depending on the flags passed to clone(), copy_process() either duplicates or shares open files, filesystem information, signal handlers, process address space, and namespace.These resources are typically shared between threads in a given process; otherwise they are unique and thus copied here.
* Finally, copy_process() cleans up and returns to the caller a pointer to the new child.

Back in do\_fork(), if copy\_process() returns successfully, the new child is woken up and run. Deliberately, the kernel runs the child process first. In the common case of the child simply calling exec() immediately, this eliminates any copy-on-write overhead that would occur if the parent ran first and began writing to the address space.

### vfork()
The vfork()system call has the same effect as fork(), except that the page table entries of the parent process are not copied. Instead, the child executes as the sole thread in the parent’s address space, and the parent is blocked until the child either calls exec() or exits. The child is not allowed to write to the address space.

Today, with copy-on-write and child-runs-first semantics, the only benefit to vfork() is not copying the parent page tables entries. If Linux one day gains copy-on-write page table entries, there will no longer be any benefit.

The vfork() system call is implemented via a special flag to the clone() system call:

* In copy_process(), the task_struct member vfork_done is set to NULL.
* In do_fork(), if the special flag was given, vfork_done is pointed at a specific address.
* After the child is first run, the parent—instead of returning—waits for the child to signal it through the vfork_done pointer.
* In the mm_release() function, which is used when a task exits a memory address space, vfork_done is checked to see whether it is NULL. If it is not, the parent is sig- naled.
* Back in do_fork(), the parent wakes up and returns.

## The Linux Implementation of Threads

Linux has a unique implementation of threads. To the Linux kernel, there is no concept of a thread. Linux implements all threads as standard processes. The Linux kernel does not provide any special scheduling semantics or data structures to represent threads. Instead, a thread is merely a process that shares certain resources with other processes. Each thread has a unique ```task_struct``` and appears to the kernel as a normal process—threads just happen to share resources, such as an address space, with other processes.

To these other operating systems, threads are an abstraction to provide a lighter, quicker execution unit than the heavy process.To Linux, threads are simply a manner of sharing resources between processes.

### Creating Threads
Threads are created the same as normal tasks, with the exception that the clone() system call is passed flags corresponding to the specific resources to be shared:

```c
clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND, 0);
```

The previous code results in behavior identical to a normal fork(), except that the address space, filesystem resources, file descriptors, and signal handlers are shared. 

In contrast, a normal fork() can be implemented as 

```c
clone(SIGCHLD, 0);
```

And vfork() is implemented as 

```c
clone(CLONE_VFORK | CLONE_VM | SIGCHLD, 0);
```

Following table lists the clone flags, which are defined in <linux/sched.h>, and their effect:

|        Flag        |                       Meaning                         |
|--------------------|-------------------------------------------------------|
|   CLONE_FILES      |         Parent and child share open files             |
|    CLONE_FS        |      Parent and child share filesystem information    |
|  CLONE_IDLETASK    |      Set PID to zero (used only by the idle tasks)    |
|   CLONE_NEWNS      |          Create a new namespace for the child         |
|   CLONE_PARENT     |      Child is to have same parent as its parent       |
|   CLONE_PTRACE     |              Continue tracing child                   |
|   CLONE_SETTID     |          Write the TID back to user-space             |
|   CLONE_SETTLS     |          Create a new TLS for the child               |
|  CLONE_SIGHAND     |Parent and child share signal handlers and blocked signals|
|  CLONE_SYSVSEM     |  Parent and child share System V SEM\_UNDO semantics  |
|   CLONE_THREAD     |     Parent and child are in the same thread group     |
|   CLONE_VFORK      |vfork() was used and the parent will sleep until the child wakes it|
| CLONE\_UNTRACED    |Do not let the tracing process force CLONE_PTRACE on the child |
|   CLONE_STOP       |          Start process in the TASK_STOPPED state       |
|  CLONE_SETTLS      |  Create a new TLS (thread-local storage) for the child |
|CLONE_CHILD_CLEARTID|              Clear the TID in the child                |
|CLONE_CHILD_SETTID  |              Set the TID in the child                  |
|CLONE_PARENT_SETTID |              Set the TID in the parent                 |
|    CLONE_VM        |          Parent and child share address space          |


### Kernel Threads
It is often useful for the kernel to perform some operations in the background. The kernel accomplishes this via **kernel threads**—standard processes that exist solely in kernel-space. The significant difference between kernel threads and normal processes is that kernel threads do not have an address space. (Their mm pointer, which points at their address space, is NULL.) They operate only in kernel-space and do not context switch into user-space. Kernel threads, however, are schedulable and preemptable, the same as normal processes.

Linux delegates several tasks to kernel threads, most notably the flush tasks and the ksoftirqd task. Kernel threads are created on system boot by other kernel threads. Indeed, a kernel thread can be created only by another kernel thread. The kernel handles this automatically by forking all new kernel threads off of thekthreadd kernel process. The interface, declared in <linux/kthread.h>, for spawning a new kernel thread from an existing one is

```c
struct task_struct *kthread_create(int (*threadfn)(void *data), 
                                   void *data,
                                   const char namefmt[], 
                                   ...)
```

The new task is created via the clone() system call by the kthread kernel process. The new process will run the threadfn function, which is passed the data argument. The process will be named namefmt, which takes printf-style formatting arguments in the variable argument list. The process is created in an unrunnable state; it will not start running until explicitly woken up via wake_up_process(). A process can be created and made runnable with a single function,kthread_run():

```c
struct task_struct *kthread_run(int (*threadfn)(void *data), 
                                void *data,
                                const char namefmt[],
                                ...)
```

This routine, implemented as a macro, simply calls both kthread\_create() and wake\_up\_process().

When started, a kernel thread continues to exist until it calls do\_exit() or another part of the kernel calls kthread\_stop(), passing in the address of the task\_struct structure returned by kthread\_create().

## Process Termination
Generally, process destruction is self-induced. It occurs when the process calls the exit() system call, either explicitly when it is ready to terminate or implicitly on return from the main subroutine of any program. (That is, the C compiler places a call to exit() after main() returns.) A process can also terminate involuntarily. This occurs when the process receives a signal or exception it cannot handle or ignore. Regardless of how a process terminates, the bulk of the work is handled by do_exit(), defined in kernel/exit.c, which completes a number of chores:

* It sets the PF\_EXITING flag in the flags member of the task\_struct.
* It calls del\_timer\_sync() to remove any kernel timers. Upon return, it is guaranteed that no timer is queued and that no timer handler is running.
* If BSD process accounting is enabled, do\_exit() calls acct\_update\_integrals() to write out accounting information.
* It calls exit\_mm() to release the mm_struct held by this process. If no other process is using this address space—that it, if the address space is not shared—the kernel then destroys it.
* It calls exit\_sem(). If the process is queued waiting for an IPC semaphore, it is dequeued here.
* It then calls exit\_files() and exit\_fs() to decrement the usage count of objects related to file descriptors and filesystem data, respectively. If either usage counts reach zero, the object is no longer in use by any process, and it is destroyed.
* It sets the task’s exit code, stored in the exit\_code member of the task\_struct, to the code provided by exit() or whatever kernel mechanism forced the termina- tion.The exit code is stored here for optional retrieval by the parent.
* It calls exit\_notify() to send signals to the task’s parent, reparents any of the task’s children to another thread in their thread group or the init process（替其子进程重新找一个父进程）, and sets the task’s exit state, stored in exit_state in the task\_struct structure, to EXIT\_ZOMBIE.
* do\_exit() calls schedule() to switch to a new process. **Because the process is now not schedulable, this is the last code the task will ever execute. do\_exit() never returns.**

At this point, all objects associated with the task are freed.The task is not runnable (and no longer has an address space in which to run) and is in the EXIT_ZOMBIE exit state.The only memory it occupies is its kernel stack, the thread_info structure, and the task_struct structure. The task exists solely to provide information to its parent.After the parent retrieves the information, or notifies the kernel that it is uninterested, the remaining memory held by the process is freed and returned to the system for use.

### Removing the Process Descriptor
When it is time to finally deallocate the process descriptor, release_task() is invoked. It does the following:

* It calls \_\_exit\_signal(), which calls \_\_unhash\_process(), which in turns calls detach\_pid() to remove the process from the pidhash and remove the process from the task list.
* \_\_exit_signal() releases any remaining resources used by the now dead process and finalizes statistics and bookkeeping.
* If the task was the last member of a thread group, and the leader is a zombie, then release\_task() notifies the zombie leader’s parent.
* release\_task() calls put\_task\_struct() to free the pages containing the process’s kernel stack and thread\_info structure and deallocate the slab cache containing the task\_struct.

At this point, the process descriptor and all resources belonging solely to the process have been freed.

### The Dilemma of the Parentless Task
If a parent exits before its children, some mechanism must exist to reparent any child tasks to a new process, or else parentless terminated processes would forever remain zombies, wasting system memory. The solution is to reparent a task’s children on exit to either another process in the current thread group or, if that fails, the init process. do\_exit() calls exit\_notify(), which calls forget\_original\_parent(), which, in turn, calls find\_new\_reaper() to perform the reparenting:

```c
static struct task_struct *find_new_reaper(struct task_struct *father) {
    struct pid_namespace *pid_ns = task_active_pid_ns(father); 
    struct task_struct *thread;
    thread = father; 
    
    while_each_thread(father, thread) {
        if (thread->flags & PF_EXITING) 
            continue;

        if (unlikely(pid_ns->child_reaper == father))
            pid_ns->child_reaper = thread; 
            
        return thread;
    }

    if (unlikely(pid_ns->child_reaper == father)) { 
        write_unlock_irq(&tasklist_lock);
        if (unlikely(pid_ns == &init_pid_ns))
            panic(“Attempted to kill init!”);

        zap_pid_ns_processes(pid_ns); 
        write_lock_irq(&tasklist_lock); 
        /*
         * We can not clear ->child_reaper or leave it alone.
         * There may by stealth EXIT_DEAD tasks on ->children,
         * forget_original_parent() must move them somewhere. 
         */
        pid_ns->child_reaper = init_pid_ns.child_reaper; 
    }
    return pid_ns->child_reaper;
}
```

This code attempts to find and return another task in the process’s thread group. If another task is not in the thread group, it finds and returns the init process. Now that a suitable new parent for the children is found, each child needs to be located and repar- ented to reaper:

```c
reaper = find_new_reaper(father); 
list_for_each_entry_safe(p, n, &father->children, sibling) {
    p->real_parent = reaper; 
    if (p->parent == father) { 
        BUG_ON(p->ptrace);
        p->parent = p->real_parent; 
    }
    reparent_thread(p, father);
}
```

ptrace_exit_finish() is then called to do the same reparenting but to a list of ptraced children:

```c
void exit_ptrace(struct task_struct *tracer) {
    struct task_struct *p, *n; 
    LIST_HEAD(ptrace_dead);
    write_lock_irq(&tasklist_lock);
    list_for_each_entry_safe(p, n, &tracer->ptraced, ptrace_entry) {
        if (__ptrace_detach(tracer, p)) 
            list_add(&p->ptrace_entry, &ptrace_dead);
    } 
    write_unlock_irq(&tasklist_lock);
    BUG_ON(!list_empty(&tracer->ptraced));
    list_for_each_entry_safe(p, n, &ptrace_dead, ptrace_entry) { 
        list_del_init(&p->ptrace_entry);
        release_task(p);
    } 
} 
```

When a task is ptraced, it is temporarily reparented to the debugging process.When the task’s parent exits, however, it must be reparented along with its other siblings. In previous kernels, this resulted in a loop over every process in the system looking for children.The solution is simply to keep a separate list of a process’s children being ptraced—reducing the search for one’s children from every process to just two relatively small lists.

# Chapter4. Process Scheduling

## Multitasking
A multitasking operating system is one that can simultaneously interleave execution of more than one process. On single processor machines, this gives the illusion of multiple processes running concurrently. On multiprocessor machines, such functionality enables processes to actually run concurrently, in parallel, on different processors.

Multitasking operating systems come in two flavors: **cooperative multitasking** and **preemptive multitasking**. 

In preemptive multitasking, the scheduler decides when a process is to cease running and a new process is to begin running. The act of involuntarily suspending a running process is called preemption. The time a process runs before it is preempted is usually predetermined, and it is called the timeslice of the process. On many modern operating systems, the timeslice is dynamically calculated as a function of process behavior and configurable system policy.

Conversely, in cooperative multitasking, a process does not stop running until it voluntary decides to do so. The act of a process voluntarily suspending itself is called yielding.

## Linux’s Process Scheduler
The O(1) scheduler performed admirably and scaled effortlessly as Linux supported large “iron” with tens if not hundreds of processors. Over time, however, it became evident that the O(1) scheduler had several pathological failures related to scheduling latency-sensitive applications.

The Rotating Staircase Deadline scheduler, which introduced the concept of fair scheduling, borrowed from queuing theory, to Linux’s process scheduler.This concept was the inspiration for the O(1) scheduler’s eventual replacement in kernel version 2.6.23, the Completely Fair Scheduler, or CFS.

## Policy
Policy is the behavior of the scheduler that determines what runs when. A scheduler’s policy often determines the overall feel of a system and is responsible for optimally utilizing processor time.

### I/O-Bound Versus Processor-Bound Processes
Processes can be classified as either I/O-bound or processor-bound. The former is characterized as a process that spends much of its time submitting and waiting on I/O requests. Conversely, processor-bound processes spend much of their time executing code. They tend to run until they are preempted because they do not block on I/O requests very often. A scheduler policy for processor-bound processes, therefore, tends to run such processes less frequently but for longer durations.

The scheduling policy in a system must attempt to satisfy two conflicting goals: fast process response time (low latency) and maximal system utilization (high throughput).

### Process Priority
A common type of scheduling algorithm is priority-based scheduling. The general idea, which isn’t exactly implemented on Linux, is that processes with a higher priority run before those with a lower priority, whereas processes with the same priority are scheduled round-robin.

The Linux kernel implements two separate priority ranges. The first is the nice value, a number from –20 to +19 with a default of 0. Larger nice values correspond to a lower priority—you are being “nice” to the other processes on the system. The second range is the real-time priority. The values are configurable, but by default range from 0 to 99, inclusive. Opposite from nice values, higher real-time priority values correspond to a greater priority. **All real-time processes are at a higher priority than normal processes; that is, the real-time priority and nice value are in disjoint value spaces.**

### Timeslice
The timeslice2 is the numeric value that represents how long a task can run until it is preempted. I/O-bound processes do not need longer timeslices (although they do like to run often), whereas processor-bound processes crave long timeslices (to keep their caches hot).

Linux’s CFS scheduler, however, does not directly assign timeslices to processes. Instead, in a novel approach, CFS assigns processes a proportion of the processor. On Linux, therefore, the amount of processor time that a process receives is a function of the load of the system. This assigned proportion is further affected by each process’s nice value. The nice value acts as a weight, changing the proportion of the processor time each process receives. 

As mentioned, the Linux operating system is preemptive.When a process enters the runnable state, it becomes eligible to run. In most operating systems, whether the process runs immediately, preempting the currently running process, is a function of the process’s priority and available timeslice. In Linux, under the new CFS scheduler, the decision is a function of how much of a proportion of the processor the newly runnable processor has consumed. If it has consumed a smaller proportion of the processor than the currently executing process, it runs immediately, preempting the current process. If not, it is scheduled to run at a later time.

### The Scheduling Policy in Action
The crucial concept is what happens when the text editor wakes up. Our primary goal is to ensure it runs immediately upon user input. In this case, when the editor wakes up, CFS notes that it is allotted 50% of the processor but has used considerably less. Specifically, CFS determines that the text editor has run for less time than the video encoder. Attempting to give all processes a fair share of the processor, it then preempts the video encoder and enables the text editor to run.

## The Linux Scheduling Algorithm

### Scheduler Classes
The Linux scheduler is modular, enabling different algorithms to schedule different types of processes. This modularity is called scheduler classes. Scheduler classes enable different, pluggable algorithms to coexist, scheduling their own types of processes. Each scheduler class has a priority. The base scheduler code, which is defined in kernel/sched.c, iterates over each scheduler class in order of priority. The highest priority scheduler class that has a runnable process wins, selecting who runs next.

The Completely Fair Scheduler (CFS) is the registered scheduler class for normal processes, called SCHED\_NORMAL in Linux (and SCHED\_OTHER in POSIX). CFS is defined in kernel/sched_fair.c.

### Process Scheduling in Unix Systems
On Unix, the priority is exported to user-space in the form of nice values. This sounds simple, but in practice it leads to several pathological problems:

* First, mapping nice values onto timeslices requires a decision about what absolute timeslice to allot each nice value.
* A second problem concerns relative nice values and again the nice value to timeslice mapping. Because nice values are most commonly used in relative terms (as the system call accepts an increment, not an absolute value), this behavior means that “nicing down a process by one” has wildly different effects depending on the starting nice value.
* Third, if performing a nice value to timeslice mapping, we need the ability to assign an absolute timeslice. This absolute value must be measured in terms the kernel can measure. In most operating systems, this means the timeslice must be some integer multiple of the timer tick. This introduces several problems:

  * First, the minimum timeslice has a floor of the period of the timer tick, which might be as high as 10 milliseconds or as low as 1 millisecond. 
  * Second, the system timer limits the difference between two timeslices; successive nice values might map to timeslices as much as 10 milliseconds or as little as 1 millisecond apart. 
  * Finally, timeslices change with different timer ticks.

* The fourth and final problem concerns handling process wake up in a priority-based scheduler that wants to optimize for interactive tasks. In such a system, you might want to give freshly woken-up tasks a priority boost by allowing them to run immediately, even if their timeslice was expired. Although this improves interactive performance in many, if not most, situations, it also opens the door to pathological cases where certain sleep/wake up use cases can game the scheduler into providing one process an unfair amount of processor time, at the expense of the rest of the system.

The approach taken by CFS is a radical (for process schedulers) rethinking of timeslice allotment: Do away with timeslices completely and assign each process a proportion of the processor. CFS thus yields constant fairness but a variable switching rate.

### Fair Scheduling
CFS is based on a simple concept: Model process scheduling as if the system had an ideal, perfectly multitasking processor. In such a system, each process would receive 1/n of the processor’s time, where n is the number of runnable processes, and we’d schedule them for infinitely small durations, so that in any measurable period we’d have run all n processes for the same amount of time. 

CFS will run each process for some amount of time, round-robin, selecting next the process that has run the least. Rather than assign each process a timeslice, CFS calculates how long a process should run as a function of the total number of runnable processes. Instead of using the nice value to calculate a timeslice, CFS uses the nice value to weight the proportion of processor a process is to receive.

To calculate the actual timeslice, CFS sets a target for its approximation of the “infinitely small” scheduling duration in perfect multitasking. This target is called the targeted latency. Smaller targets yield better interactivity and a closer approximation to perfect multitasking, at the expense of higher switching costs and thus worse overall throughput. CFS imposes a floor on the timeslice assigned to each process. This floor is called the minimum granularity. By default it is 1 millisecond.

Put generally, the proportion of processor time that any process receives is determined only by the relative difference in niceness between it and the other runnable processes. CFS is called a fair scheduler because it gives each process a fair share—a proportion—of the processor’s time.

## The Linux Scheduling Implementation

### Time Accounting
All process schedulers must account for the time that a process runs. Most Unix systems do so, as discussed earlier, by assigning each process a timeslice. On each tick of the system clock, the timeslice is decremented by the tick period.When the timeslice reaches zero, the process is preempted in favor of another runnable process with a nonzero timeslice.

#### The Scheduler Entity Structure
CFS does not have the notion of a timeslice, but it must still keep account for the time that each process runs, because it needs to ensure that each process runs only for its fair share of the processor. CFS uses the scheduler entity structure, struct sched_entity, defined in <linux/sched.h>, to keep track of process accounting:

```
struct sched_entity { 
    struct load\_weight load;
    struct rb\_node run\_node;
    struct list\_head group\_node;
    unsigned int on_rq;
    u64 exec_start;
    u64 sum_exec_runtime;
    u64 vruntime;
    u64 prev_sum_exec_runtime;
    u64 last_wakeup;
    u64 avg_overlap; 
    u64 nr_migrations; 
    u64 start_runtime; 
    u64 avg_wakeup;
    /* many stat variables elided, enabled only if CONFIG_SCHEDSTATS is set */
}
```
The scheduler entity structure is embedded in the process descriptor,struct task_stuct, as a member variable named se.

#### The Virtual Runtime
The vruntime variable stores the virtual runtime of a process, which is the actual runtime (the amount of time spent running) normalized (or weighted) by the number of runnable processes. The virtual runtime’s units are nanoseconds and therefore vruntime is decoupled from the timer tick.

The function update\_curr(), defined in kernel/sched\_fair.c, manages this accounting:

```
static void update_curr(struct cfs_rq *cfs_rq) {
    struct sched_entity *curr = cfs_rq->curr; 
    u64 now = rq_of(cfs_rq)->clock;
    unsigned long delta_exec;
    if (unlikely(!curr)) 
        return;
    /*
     * Get the amount of time the current task was running
     * since the last time we changed load (this cannot
     * overflow on 32 bits): 
     */
    delta_exec = (unsigned long)(now - curr->exec_start); 
    if (!delta_exec)
        return;
    __update_curr(cfs_rq, curr, delta_exec); 
    curr->exec_start = now;

    if (entity_is_task(curr)) {
        struct task_struct *curtask = task_of(curr);
        trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime); 
        cpuacct_charge(curtask, delta_exec); 
        account_group_exec_runtime(curtask, delta_exec);
    } 
}
```

update\_curr() calculates the execution time of the current process and stores that value in delta\_exec. It then passes that runtime to \_\_update_curr(), which weights the time by the number of runnable processes. The current process’s vruntime is then incremented by the weighted value:

```
/*
* Update the current task’s runtime statistics. Skip current tasks that * are not in our scheduling class.
*/
static inline void
__update_curr(struct cfs_rq *cfs_rq, struct sched_entity *curr,
  unsigned long delta_exec)
{
    unsigned long delta_exec_weighted;
    schedstat_set(curr->exec_max, max((u64)delta_exec, curr->exec_max));    
    curr->sum_exec_runtime += delta_exec; 
    schedstat_add(cfs_rq, exec_clock, delta_exec); 
    delta_exec_weighted = calc_delta_fair(delta_exec, curr);
    curr->vruntime += delta_exec_weighted; 
    update_min_vruntime(cfs_rq);
}
```

update\_curr() is invoked periodically by the system timer and also whenever a process becomes runnable or blocks, becoming unrunnable. In this manner, vruntime is an accurate measure of the runtime of a given process and an indicator of what process should run next.

### Process Selection
In the last section, we discussed how vruntime on an ideal, perfectly multitasking processor would be identical among all runnable processes. In reality, we cannot perfectly multitask, so CFS attempts to balance a process’s virtual runtime with a simple rule: When CFS is deciding what process to run next, it picks the process with the smallest vruntime. This is, in fact, the core of CFS’s scheduling algorithm: **Pick the task with the smallest vruntime.**

CFS uses a red-black tree to manage the list of runnable processes and efficiently find the process with the smallest vruntime.

#### Picking the Next Task
CFS’s process selection algorithm is thus summed up as “run the process represented by the leftmost node in the rbtree.”The function that performs this selection is \_\_pick\_next\_entity(), defined in kernel/sched\_fair.c:

```
static struct sched_entity *__pick_next_entity(struct cfs_rq *cfs_rq) {
    struct rb_node *left = cfs_rq->rb_leftmost;
    if (!left)
        return NULL;
    return rb_entry(left, struct sched_entity, run_node);
}
```

Although it is efficient to walk the tree to find the leftmost node—O(height of tree), which is O(log N) for N nodes if the tree is balanced—it is even easier to cache the leftmost node. The return value from this function is the process that CFS next runs. If the function returns NULL, there is no leftmost node, and thus no nodes in the tree. In that case, there are no runnable processes, and CFS schedules the idle task.

#### Adding Processes to the Tree
Now let’s look at how CFS adds processes to the rbtree and caches the leftmost node. This would occur when a process becomes runnable (wakes up) or is first created via fork(). Adding processes to the tree is performed by enqueue_entity():

```c
static void
enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags) 
{
    /*
     * Update the normalized vruntime before updating min_vruntime
     * through callig update_curr(). */

    if (!(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_MIGRATE)) 
        se->vruntime += cfs_rq->min_vruntime;
    /*
     * Update run-time statistics of the ‘current’. 
     */
    update_curr(cfs_rq); 
    account_entity_enqueue(cfs_rq, se);
    if (flags & ENQUEUE_WAKEUP) { 
        place_entity(cfs_rq, se, 0); 
        enqueue_sleeper(cfs_rq, se);
    }

    update_stats_enqueue(cfs_rq, se); 
    check_spread(cfs_rq, se);
    if (se != cfs_rq->curr)
        __enqueue_entity(cfs_rq, se);
}
```

This function updates the runtime and other statistics and then invokes \_\_enqueue_entity() to perform the actual heavy lifting of inserting the entry into the red-black tree:

```
/*
 * Enqueue an entity into the rb-tree: 
 */
static void __enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se) 
{
    struct rb_node **link = &cfs_rq->tasks_timeline.rb_node; 
    struct rb_node *parent = NULL;
    struct sched_entity *entry;
    s64 key = entity_key(cfs_rq, se);
    int leftmost = 1;
    /*
     * Find the right place in the rbtree: 
     */
    while (*link) {
        parent = *link;
        entry = rb_entry(parent, struct sched_entity, run_node);
        /*
        * We dont care about collisions. Nodes with * the same key stay together.
        */
        if (key < entity_key(cfs_rq, entry)) { 
            link = &parent->rb_left;
        } else {
            link = &parent->rb_right;
            leftmost = 0;
        } 
    }
    /*
    * Maintain a cache of leftmost tree entries (it is frequently * used):
    */
    if (leftmost)
        cfs_rq->rb_leftmost = &se->run_node;

    rb_link_node(&se->run_node, parent, link); 
    rb_insert_color(&se->run_node, &cfs_rq->tasks_timeline);
}
```

The loop terminates when we compare ourselves to a node that has no child in the direction we move; link is then NULL and the loop terminates.When out of the loop, the function calls rb\_link\_node() on the parent node, making the inserted process the new child.The function rb\_insert\_color() updates the self-balancing properties of the tree.

#### Removing Processes from the Tree
Finally, let’s look at how CFS removes processes from the red-black tree.This happens when a process blocks (becomes unrunnable) or terminates (ceases to exist):

```c
static void
dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int sleep) {
    /*
     * Update run-time statistics of the ‘current’. 
     */
    update_curr(cfs_rq);

    update_stats_dequeue(cfs_rq, se); 
    clear_buddies(cfs_rq, se);
    
    if (se != cfs_rq->curr) 
        __dequeue_entity(cfs_rq, se);

    account_entity_dequeue(cfs_rq, se); 
    update_min_vruntime(cfs_rq);

    /*
     * Normalize the entity after updating the min_vruntime because the
     * update can refer to the ->curr item and we need to reflect this
     * movement in our normalized position. 
     */
    if (!sleep)
        se->vruntime -= cfs_rq->min_vruntime;
}
```

As with adding a process to the red-black tree, the real work is performed by a helper function, \_\_dequeue\_entity():

```c
static void __dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se) {
    if (cfs_rq->rb_leftmost == &se->run_node) { 
        struct rb_node *next_node;

        next_node = rb_next(&se->run_node); 
        cfs_rq->rb_leftmost = next_node;
    }
    rb_erase(&se->run_node, &cfs_rq->tasks_timeline);
}

```

### The Scheduler Entry Point
The main entry point into the process schedule is the function schedule(), defined in kernel/sched.c. This is the function that the rest of the kernel uses to invoke the process scheduler, deciding which process to run and then running it. schedule() is generic with respect to scheduler classes. That is, it finds the highest priority scheduler class with a runnable process and asks it what to run next. The only important part of the function is its invocation of pick\_next\_task(), also defined in kernel/sched.c. The pick\_next\_task() function goes through each scheduler class, starting with the highest priority, and selects the highest priority process in the highest priority class:

```
/*
* Pick up the highest-prio task: */
static inline struct task_struct * 
pick\_next\_task(struct rq *rq)
{
    const struct sched_class *class; 
    struct task_struct *p;
    /*
     * Optimization: we know that if all tasks are in
     * the fair class we can call that function directly: 
     */
    if (likely(rq->nr_running == rq->cfs.nr_running)) { 
        p = fair_sched_class.pick_next_task(rq); 
        if (likely(p))
            return p;
    } 

    class = sched_class_highest;
    for (;;) {
        p = class->pick_next_task(rq); 
        if (p)
            return p;
        /*
        * Will never be NULL as the idle class always 
        * returns a non-NULL p:
        */
        class = class->next;
    }
}
```

Note the optimization at the beginning of the function. Because CFS is the scheduler class for normal processes, and most systems run mostly normal processes, there is a small hack to quickly select the next CFS-provided process if the number of runnable processes is equal to the number of CFS runnable processes. CFS’s implementation of pick\_next\_task() calls pick\_next\_entity(), which in turn calls the \_\_pick\_next\_entity() function that we discussed in the previous section.


### Sleeping and Waking Up
A common reason to sleep is file I/O—for example, the task issued a read() request on a file, which needs to be read in from disk. As another example, the task could be waiting for keyboard input. Whatever the case, the kernel behavior is the same: **The task marks itself as sleeping, puts itself on a wait queue, removes itself from the red-black tree of runnable, and calls schedule() to select a new process to execute**. Waking back up is the inverse: **The task is set as runnable, removed from the wait queue, and added back to the red-black tree.**


#### Wait Queues
Sleeping is handled via wait queues.A wait queue is a simple list of processes waiting for an event to occur. Wait queues are represented in the kernel by wake\_queue\_head\_t. Wait queues are created statically via DECLARE\_WAITQUEUE() or dynamically via init\_waitqueue\_head(). 

Some simple interfaces for sleeping used to be in wide use. These interfaces, however, have races: It is possible to go to sleep after the condition becomes true. In that case, the task might sleep indefinitely. Therefore, the recommended method for sleeping in the kernel is a bit more complicated:

```c
/* ‘q’ is the wait queue we wish to sleep on */ 
DEFINE_WAIT(wait);
add_wait_queue(q, &wait);
while (!condition) { /* condition is the event that we are waiting for */
    prepare_to_wait(&q, &wait, TASK_INTERRUPTIBLE); 
    if (signal_pending(current))
        /* handle signal */ 
    schedule();
}
finish_wait(&q, &wait);
```

The task performs the following steps to add itself to a wait queue:

* Creates a wait queue entry via the macro DEFINE_WAIT().
* Addsitselftoawaitqueueviaadd_wait_queue().Thiswaitqueueawakensthe process when the condition for which it is waiting occurs. Of course, there needs to be code elsewhere that calls wake_up() on the queue when the event actually does occur.
* Callsprepare_to_wait()tochangetheprocessstatetoeither TASK_INTERRUPTIBLE or TASK_UNINTERRUPTIBLE.This function also adds the task back to the wait queue if necessary, which is needed on subsequent iterations of the loop.
* If the state is set to TASK_INTERRUPTIBLE, a signal wakes the process up.This is called a spurious wake up (a wake-up not caused by the occurrence of the event). So check and handle signals.
* When the task awakens, it again checks whether the condition is true. If it is, it exits the loop. Otherwise, it again calls schedule() and repeats.
* Now that the condition is true, the task sets itself to TASK_RUNNING and removes itself from the wait queue via finish_wait().

The function inotify\_read() in fs/notify/inotify/inotify\_user.c, which handles reading from the inotify file descriptor, is a straightforward example of using wait queues:

```c
static ssize_t inotify_read(struct file *file, char __user *buf, 
                            size_t count, loff_t *pos)
{
    struct fsnotify_group *group; 
    struct fsnotify_event *kevent; 
    char __user *start;
    int ret;
    DEFINE_WAIT(wait);

    start = buf;
    group = file->private_data;

    while (1) { 
        prepare_to_wait(&group->notification_waitq, 
                        &wait, TAS
                        K_INTERRUPTIBLE);
        mutex_lock(&group->notification_mutex); 
        kevent = get_one_event(group, count); 
        mutex_unlock(&group->notification_mutex);

        if (kevent) {
            ret = PTR_ERR(kevent);
            if (IS_ERR(kevent)) 
                break;
            ret = copy_event_to_user(group, kevent, buf); 
            fsnotify_put_event(kevent);
            if (ret < 0)
                break; 
            buf += ret;
            count -= ret; 
            continue;
        }

    ret = -EAGAIN;
    if (file->f_flags & O_NONBLOCK)
        break; 
    ret = -EINTR;
    if (signal_pending(current)) 
        break;
    if (start != buf) 
        break;
    schedule();
    }
    finish_wait(&group->notification_waitq, &wait);

    if (start != buf && ret != -EFAULT) 
        ret = buf - start;
    return ret;
}
```

#### Waking Up
Waking is handled via wake\_up(), which wakes up all the tasks waiting on the given wait queue. It calls try\_to\_wake\_up(), which sets the task’s state to TASK\_RUNNING, calls enqueue\_task() to add the task to the red-black tree, and sets need\_resched if the awakened task’s priority is higher than the priority of the current task. The code that causes the event to occur typically calls wake\_up() itself.

<img src="/assets/images/linux-kernel-development-note/illustration-1.png" width="800" />

## Preemption and Context Switching
Context switching, the switching from one runnable task to another, is handled by the context_switch()function defined in kernel/sched.c. It is called by schedule() when a new process has been selected to run. It does two basic jobs:
* Calls switch\_mm(), which is declared in <asm/mmu_context.h>, to switch the virtual memory mapping from the previous process’s to that of the new process.
* Calls switch\_to(), declared in <asm/system.h>, to switch the processor state from the previous process’s to the current’s. This involves saving and restoring stack information and the processor registers and any other architecture-specific state that must be managed and restored on a per-process basis.

The kernel provides the need\_resched flag to signify whether a reschedule should be performed. This flag is set by scheduler\_tick() when a process should be preempted, and by try\_to\_wake\_up() when a process that has a higher priority than the currently running process is awakened. The kernel checks the flag, sees that it is set, and calls schedule() to switch to a new process.

Upon returning to user-space or returning from an interrupt, the need\_resched flag is checked. If it is set, the kernel invokes the scheduler before continuing.

The flag is per-process, and not simply global, because it is faster to access a value in the process descriptor (because of the speed of current and high probability of it being cache hot) than a global variable.

### User Preemption
User preemption occurs when the kernel is about to return to user-space, need\_resched is set, and therefore, the scheduler is invoked. If the kernel is returning to user-space, it knows it is in a safe quiescent state. In other words, if it is safe to continue executing the current task, it is also safe to pick a new task to execute. Consequently, whenever the kernel is preparing to return to user-space either on return from an interrupt or after a system call, the value of need\_resched is checked.


In short, user preemption can occur:

* When returning to user-space from a system call
* When returning to user-space from an interrupt handler

### Kernel Preemption（抢占时机没理解）
In nonpreemptive kernels, kernel code runs until completion. That is, the scheduler cannot reschedule a task while it is in the kernel—kernel code is scheduled cooperatively, not preemptively. In the 2.6 kernel, however, the Linux kernel became preemptive: It is now possible to preempt a task at any point, so long as the kernel is in a state in which it is safe to reschedule.

So when is it safe to reschedule? **The kernel can preempt a task running in the kernel so long as it does not hold a lock.** That is, locks are used as markers of regions of nonpreemptibility. Because the kernel is SMP-safe, if a lock is not held, the current code is reentrant and capable of being preempted.

The first change in supporting kernel preemption was the addition of a preemption counter, preempt\_count, to each process’s thread\_info.This counter begins at zero and increments once for each lock that is acquired and decrements once for each lock that is released. When the counter is zero, the kernel is preemptible.

Kernel preemption can also occur explicitly, when a task in the kernel blocks or explicitly calls schedule(). This form of kernel preemption has always been supported because no additional logic is required to ensure that the kernel is in a state that is safe to preempt. It is assumed that the code that explicitly calls schedule() knows it is safe to reschedule.

Kernel preemption can occur:

* When an interrupt handler exits, before returning to kernel-space
* When kernel code becomes preemptible again
* If a task in the kernel explicitly calls schedule()
* If a task in the kernel blocks (which results in a call to schedule())

## Real-Time Scheduling Policies
Linux provides two real-time scheduling policies, SCHED\_FIFO and SCHED\_RR. The normal, not real-time scheduling policy is SCHED\_NORMAL. Via the scheduling classes framework, these real-time policies are managed not by the Completely Fair Scheduler, but by a special real-time scheduler, defined in kernel/sched_rt.c.

SCHED\_FIFO implements a simple first-in, first-out scheduling algorithm without timeslices. A runnable SCHED\_FIFO task is always scheduled over any SCHED\_NORMAL tasks. When a SCHED\_FIFO task becomes runnable, it continues to run until it blocks or explicitly yields the processor; it has no timeslice and can run indefinitely. Only a higher priority SCHED\_FIFO or SCHED\_RR task can preempt a SCHED\_FIFO task. Two or more SCHED\_FIFO tasks at the same priority run round-robin, but again only yielding the processor when they explicitly choose to do so. 

SCHED\_RR is identical to SCHED\_FIFO except that each process can run only until it exhausts a predetermined timeslice. That is, SCHED\_RR is SCHED\_FIFO with timeslices—it is a real-time, round-robin scheduling algorithm. **The timeslice is used to allow only rescheduling of same-priority processes.** As with SCHED\_FIFO, a higher-priority process always immediately preempts a lower-priority one, and a lower-priority process can never preempt a SCHED\_RR task, even if its timeslice is exhausted.

Both real-time scheduling policies implement static priorities.The kernel does not calculate dynamic priority values for real-time tasks. This ensures that a real-time process at a given priority always preempts a process at a lower priority.

The real-time scheduling policies in Linux provide soft real-time behavior. Soft real-time refers to the notion that the kernel tries to schedule applications within timing deadlines, but the kernel does not promise to always achieve these goals.

## Scheduler-Related System Calls
Linux provides a family of system calls for the management of scheduler parameters. These system calls allow manipulation of process priority, scheduling policy, and processor affinity, as well as provide an explicit mechanism to yield the processor to other tasks.

|       System Call       |               Description               |
|-------------------------|-----------------------------------------|
|          nice()         |     Sets a process’s nice value         |
|   sched_setscheduler()  |     Sets a process’s scheduling policy  |
|   sched_getscheduler()  |     Gets a process’s scheduling policy  |
|     sched_setparam()    |     Sets a process’s real-time priority |
|     sched_getparam()    |     Gets a process’s real-time priority |
| sched_get_priority_max()|     Gets the maximum real-time priority |
| sched_get_priority_min()|     Gets the minimum real-time priority |
| sched_rr_get_interval() |     Gets a process’s timeslice value    |
|    sched_setaffinity()  |     Sets a process’s processor affinity |
|    sched_getaffinity()  |     Gets a process’s processor affinity |
|       sched_yield()     |     Temporarily yields the processor    |

### Scheduling Policy and Priority-Related System Calls
The sched\_setscheduler() and sched\_getscheduler() system calls set and get a given process’s scheduling policy and real-time priority, respectively. The important work is merely to read or write the policy and rt\_priority values in the process’s task\_struct.

The sched\_setparam() and sched\_getparam() system calls set and get a process’s real-time priority. These calls merely encode rt\_priority in a special sched\_param structure.

For normal tasks, the nice() function increments the given process’s static priority by the given amount. Only root can provide a negative value, thereby lowering the nice value and increasing the priority. The nice() function calls the kernel’s set\_user\_nice() function, which sets the static\_prio and prio values in the task’s task\_struct as appropriate.

### Processor Affinity System Calls
The Linux scheduler enforces hard processor affinity（处理器绑定）. The scheduler also enables a user to say, "This task must remain on this subset of the available processors no matter what." This hard affinity is stored as a bitmask in the task’s task\_struct as cpus\_allowed. The bitmask contains one bit per possible processor on the system. By default, all bits are set and, therefore, a process is potentially runnable on any processor. The user, however, via sched_setaffinity(), can provide a different bitmask of any combination of one or more bits.

The kernel enforces hard affinity in a simple manner. First, when a process is initially created, it inherits its parent’s affinity mask. Because the parent is running on an allowed processor, the child thus runs on an allowed processor. Second, when a processor’s affinity is changed, the kernel uses the migration threads to push the task onto a legal processor. Finally, the load balancer pulls tasks to only an allowed processor.Therefore, a process only ever runs on a processor whose bit is set in the cpus_allowed field of its process descriptor.

### Yielding Processor Time
Linux provides the sched_yield() system call as a mechanism for a process to explicitly yield the processor to other waiting processes. It works by removing the process from the active array (where it currently is, because it is running) and inserting it into the expired array. This has the effect of not only preempting the process and putting it at the end of its priority list, but also putting it on the expired list—guaranteeing it will not run for a while. Because real-time tasks never expire, they are a special case. Therefore, they are merely moved to the end of their priority list.

Kernel code, as a convenience, can call yield(), which ensures that the task’s state is TASK_RUNNING and then call sched_yield(). User-space applications use the sched_yield()system call.

# Chapter5. System Calls

## Communicating with the Kernel
System calls provide a layer between the hardware and user-space processes. This layer serves three primary purposes:

* First, it provides an abstracted hardware interface for user-space.
* Second, system calls ensure system security and stability. With the kernel acting as a middleman between system resources and user-space, the kernel can arbitrate access based on permissions, users, and other criteria. 
* Finally, a single common layer between user-space and the rest of the system allows for the virtualized system provided to processes. If applications were free to access system resources without the kernel’s knowledge, it would be nearly impossible to implement multitasking and virtual memory, and certainly impossible to do so with stability and security. 

In Linux, **system calls are the only means user-space has of interfacing with the kernel; they are the only legal entry point into the kernel other than exceptions and traps.**

## APIs, POSIX, and the C Library
An API defines a set of programming interfaces used by applications. Those interfaces can be implemented as a system call, implemented through multiple system calls, or implemented without the use of system calls at all.

The system call interface in Linux, as with most Unix systems, is provided in part by the C library. The C library implements the main API on Unix systems, including the standard C library and the system call interface.

A meme related to interfaces in Unix is **"Provide mechanism, not policy."** In other words, Unix system calls exist to provide a specific function in an abstract sense. The manner in which the function is used is not any of the kernel’s business.

## Syscalls
System calls (often called syscalls in Linux) are typically accessed via function calls defined in the C library. They can define zero, one, or more arguments (inputs) and might result in one or more side effects. System calls also provide a return value of type long4 that signifies success or error. The C library, when a system call returns an error, writes a special error code into the global errno variable.

SYSCALL_DEFINE0 is simply a macro that defines a system call with no parameters (hence the 0).The expanded code looks like this:

```
asmlinkage long sys_getpid(void)
```

Let’s look at how system calls are defined:

* First, note the asmlinkage modifier on the function definition. This is a directive to tell the compiler to look only on the stack for this function’s arguments. This is a required modifier for all system calls. 
* Second, the function returns a long. For compatibility between 32- and 64-bit systems, system calls defined to return an int in user-space return a long in the kernel.
* Third, note that the getpid() system call is defined as sys\_getpid() in the kernel. This is the naming convention taken with all system calls in Linux: System call bar() is implemented in the kernel as function sys\_bar().

### System Call Numbers
In Linux, each system call is assigned a syscall number. This is a unique number that is used to reference a specific system call. When a user-space process executes a system call, the syscall number identifies which syscall was executed; the process does not refer to the syscall by name.

The syscall number is important; when assigned, it cannot change, or compiled applications will break. Likewise, if a system call is removed, its system call number cannot be recycled, or previously compiled code would aim to invoke one system call but would in reality invoke another.

The kernel keeps a list of all registered system calls in the system call table, stored in sys\_call\_table. This table is architecture; on x86-64 it is defined in arch/i386/kernel/syscall\_64.c. This table assigns each valid syscall to a unique syscall number.

### System Call Performance
System calls in Linux are faster than in many other operating systems. This is partly because of Linux’s fast context switch times; entering and exiting the kernel is a streamlined and simple affair. The other factor is the simplicity of the system call handler and the individual system calls themselves.

## System Call Handler
It is not possible for user-space applications to execute kernel code directly. They cannot simply make a function call to a method existing in kernel-space because the kernel exists in a protected memory space.

Instead, user-space applications must somehow signal to the kernel that they want to execute a system call and have the system switch to kernel mode, where the system call can be executed in kernel-space by the kernel on behalf of the application.

**The mechanism to signal the kernel is a software interrupt**: Incur an exception, and the system will switch to kernel mode and execute the exception handler. The exception handler, in this case, is actually the system call handler. The defined software interrupt on x86 is interrupt number 128, which is incurred via the int $0x80 instruction. It triggers a switch to kernel mode and the execution of exception vector 128, which is the system call handler. The system call handler is the aptly named function system_call(). It is architecture-dependent.

### Denoting the Correct System Call
On x86, the syscall number is fed to the kernel via the eax register. Before causing the trap into the kernel, user-space sticks in eax the number corresponding to the desired system call. The system call handler then reads the value from eax. Other architectures do something similar.

The system\_call() function checks the validity of the given system call number by comparing it to NR\_syscalls. If it is larger than or equal to NR\_syscalls, the function returns -ENOSYS. Otherwise, the specified system call is invoked:

```
call *sys_call_table(, %eax, 8)
```

Because each element in the system call table is 64 bits (8 bytes), the kernel multiplies the given system call number by four to arrive at its location in the system call table. On x86-32, the code is similar, with the 8 replaced by 4.

### Parameter Passing
In addition to the system call number, most syscalls require that one or more parameters be passed to them. The easiest way to do this is via the same means that the syscall number is passed: The parameters are stored in registers. On x86-32, the registers ebx, ecx, edx, esi, and edi contain, in order, the first five arguments. In the unlikely case of six or more arguments, a single register is used to hold a pointer to user-space where all the parameters are stored. The return value is sent to user-space also via register. On x86, it is written into the eax register.

## System Call Implementation

### Implementing System Calls
The first step in implementing a system call is defining its purpose. The syscall should have exactly one purpose. Multiplexing syscalls (a single system call that does wildly different things depending on a flag argument) is discouraged in Linux.

Many system calls provide a flag argument to address forward compatibility. The flag is not used to multiplex different behavior across a single system call—as mentioned, that is not acceptable—but to enable new functionality and options without breaking backward compatibility or needing to add a new system call.

### Verifying the Parameters
System calls must carefully verify all their parameters to ensure that they are valid and legal.

Before following a pointer into user-space, the system must ensure that:

* The pointer points to a region of memory in user-space. Processes must not be able to trick the kernel into reading data in kernel-space on their behalf.
* The pointer points to a region of memory in the process’s address space. The process must not be able to trick the kernel into reading someone else’s data.
* If reading, the memory is marked readable. If writing, the memory is marked writable. If executing, the memory is marked executable.The process must not be able to bypass memory access restrictions.

The kernel provides two methods for performing the requisite checks and the desired copy to and from user-space:

* For writing into user-space, the method copy_to_user() is provided. It takes three parameters. The first is the destination memory address in the process’s address space. The second is the source pointer in kernel-space. Finally, the third argument is the size in bytes of the data to copy.
* For reading from user-space, the method copy_from_user() is analogous to copy_to_user(). The function reads from the second parameter into the first parameter the number of bytes specified in the third parameter.

A call to capable() with a valid capabilities flag returns nonzero if the caller holds the specified capability and zero otherwise. 

## System Call Context
As discussed in Chapter 3, the kernel is in process context during the execution of a system call. The current pointer points to the current task, which is the process that issued the syscall. In process context, the kernel is capable of sleeping (for example, if the system call blocks on a call or explicitly calls schedule()) and is fully preemptible. These two points are important:

* First, the capability to sleep means that system calls can make use of the majority of the kernel’s functionality. The capability to sleep greatly simplifies kernel programming.（没理解）
* The fact that process context is preemptible implies that, like user-space, the current task may be preempted by another task. Because the new task may then execute the same system call, care must be exercised to ensure that system calls are reentrant. 

When the system call returns, control continues in system_call(), which ultimately switches to user-space and continues the execution of the user process.

## Final Steps in Binding a System Call
After the system call is written, it is trivial to register it as an official system call:

* Add an entry to the end of the system call table. This needs to be done for each architecture that supports the system call (which, for most calls, is all the architectures). The position of the syscall in the table, starting at zero, is its system call number. 
* For each supported architecture, define the syscallnumber in <asm/unistd.h>.
* Compile the syscall into the kernel image (as opposed to compiling as a module). This can be as simple as putting the system call in a relevant file in kernel/, such as sys.c, which is home to miscellaneous system calls.

## Accessing the System Call from User-Space
Linux provides a set of macros for wrapping access to system calls. It sets up the register contents and issues the trap instructions.These macros are named _syscalln(), where n is between 0 and 6.The number corresponds to the number of parameters passed into the syscall because the macro needs to know how many parameters to expect and, consequently, push into registers. 

For example, consider the system call open(), defined as

```
long open(const char *filename, int flags, int mode)
```

The syscall macro to use this system call without explicit library support would be

```
#define __NR_open 5
_syscall3(long, open, const char *, filename, int, flags, int, mode)
```

For each macro, there are 2 + 2 × n parameters.The first parameter corresponds to the return type of the syscall.The second is the name of the system call. Next follows the type and name for each parameter in order of the system call.The \_\_NR\_open define is in <asm/unistd.h>; it is the system call number.The _syscall3 macro expands into a C function with inline assembly; the assembly performs the steps discussed in the previous section to push the system call number and parameters into the correct registers and issue the software interrupt to trap into the kernel. Placing this macro in an application is all that is required to use the open() system call.

## Why Not to Implement a System Call
The previous sections have shown that it is easy to implement a new system call, but that in no way should encourage you to do so. Indeed, you should exercise caution and restraint in adding new syscalls. 

The pros of implementing a new interface as a syscall are as follows: 

* System calls are simple to implement and easy to use.
* System call performance on Linux is fast.

The cons:

* You need a syscall number, which needs to be officially assigned to you.
* After the system call is in a stable series kernel, it is written in stone. The interface cannot change without breaking user-space applications.
* Each architecture needs to separately register the system call and support it.
* System calls are not easily used from scripts and cannot be accessed directly from the filesystem.
* Because you need an assigned syscall number, it is hard to maintain and use a system call outside of the master kernel tree.
* For simple exchanges of information, a system call is overkill.

The alternatives:

* Implement a device node and read() and write() to it. Use ioctl() to manipulate specific settings or retrieve specific information.
* Certain interfaces, such as semaphores, can be represented as file descriptors and manipulated as such.
* Add the information as a file to the appropriate location in sysfs.

# Chapter6. Kernel Data Structures

## Linked Lists

### Singly and Doubly Linked Lists
The simplest data structure representing such a linked list might look similar to the following:

```
/* an element in a linked list */ 
struct list_element {
    void *data;  /* the payload */
    struct list_element *next;  /* pointer to the next element */
};
```

In some linked lists, each element also contains a pointer to the previous element. These lists are called doubly linked lists because they are linked both forward and backward. A data structure representing a doubly linked list would look similar to this:

```
/* an element in a linked list */ 
struct list_element {
    void *data;  /* the payload */
    struct list_element *next;  /* pointer to the next element */
    struct list_element *prev;  /* pointer to the previous element */
};
```

### Circular Linked Lists
In some linked lists, the last element does not point to a special value. Instead, it points back to the first value.This linked list is called a circular linked list because the list is cyclic. 

### The Linux Kernel’s Implementation
For example, assume we had a fox structure to describe that member of the Canidae family:

```c
struct fox { 
    unsigned long tail_length; /* length in centimeters of tail */  
    unsigned long weight; /* weight in kilograms */
    bool    is_fantastic; /* is this fox fantastic? */
};
```

The common pattern for storing this structure in a linked list is to embed the list pointer in the structure. For example:

```c
struct fox { 
    unsigned long tail_length; /* length in centimeters of tail */    
    unsigned long weight; /* weight in kilograms */
    bool is_fantastic; /* is this fox fantastic? */
    struct fox *next; /* next fox in linked list */
    struct fox *prev; /* previous fox in linked list */
};
```


The Linux kernel approach is different. Instead of turning the structure into a linked list, the Linux approach is to **embed a linked list node in the structure!**

#### The Linked List Structure

The linked-list code is declared in the header file <linux/list.h> and the data struc- ture is simple:

```c
struct list_head {
    struct list_head *next
    struct list_head *prev;
};
```

The utility is in how the list_head structure is used:

```c
struct fox { 
    unsigned long tail_length; /* length in centimeters of tail */ 
    unsigned long weight; /* weight in kilograms */
    bool is_fantastic; /* is this fox fantastic? */
    struct list_head list; /* list of all fox structures */
}
```

Using the macro container_of(), we can easily find the parent structure containing any given member variable.This is because in C, the offset of a given variable into a structure is fixed by the ABI at compile time.

```c
#define container_of(ptr, type, member) ({ \
 const typeof( ((type *)0)->member ) *__mptr = (ptr); \
  (type *)( (char *)__mptr - offsetof(type,member) );})
```

Using container_of(), we can define a simple function to return the parent structure containing any list_head:

```c
#define list_entry(ptr, type, member) \
 container_of(ptr, type, member)
```

#### Defining a Linked List
As shown, a list_head by itself is worthless; it is normally embedded inside your own structure. The list needs to be initialized before it can be used. Because most of the elements are created dynamically (probably why you need a linked list), the most common way of initializing the linked list is at runtime:

```c
struct fox *red_fox;
red_fox = kmalloc(sizeof(*red_fox), GFP_KERNEL); 
red_fox->tail_length = 40;
red_fox->weight = 6;
red_fox->is_fantastic = false; 
INIT_LIST_HEAD(&red_fox->list);
```

If the structure is statically created at compile time, and you have a direct reference to it, you can simply do this:

```c
struct fox red_fox = {
    .tail_length = 40,
    .weight = 6,
    .list = LIST_HEAD_INIT(red_fox.list),
};
```

#### List Heads
You will generally want a special pointer that refers to your linked list, without being a list node itself. Interestingly, this special node is in fact a normal list_head:

```c
static LIST_HEAD(fox_list);
```

This defines and initializes a list_head named fox_list. The majority of the linked list routines accept one or two parameters: the head node or the head node plus an actual list node.

### Manipulating Linked Lists
The kernel provides a family of functions to manipulate linked lists. They all take pointers to one or more list_head structures.The functions are implemented as inline functions in generic C and can be found in <linux/list.h>.

Interestingly, all these functions are O(1). This means they execute in constant time, regardless of the size of the list or any other inputs. 

#### Adding a Node to a Linked List
To add a node to a linked list:

```c
list_add(struct list_head *new, struct list_head *head)
```

This function adds the new node to the given list immediately after the head node. Because the list is circular and generally has no concept of first or last nodes, you can pass any element for head. 

Returning to our fox example, assume we had a new struct fox that we wanted to add to the fox_list list. We’d do this:

```c
list_add(&f->list, &fox_list);
```

To add a node to the end of a linked list:

```c
list_add_tail(struct list_head *new, struct list_head *head)
```

#### Deleting a Node from a Linked List
To delete a node from a linked list, use list_del():

```c
list_del(struct list_head *entry)
```

The implementation is instructive:

```c
static inline void __list_del(struct list_head *prev, struct list_head *next) {
    next->prev = prev; 
    prev->next = next;
}

static inline void list_del(struct list_head *entry) {
    __list_del(entry->prev, entry->next);
}
```

To delete a node from a linked list and reinitialize it, the kernel provides list_del_init():

```c
list_del_init(struct list_head *entry)
```

#### Moving and Splicing Linked List Nodes
To move a node from one list to another:

```c
list_move(struct list_head *list, struct list_head *head)
```

This function removes the list entry from its linked list and adds it to the given list after the head element.

To move a node from one list to the end of another:

```c
list_move_tail(struct list_head *list, struct list_head *head)
```

To check whether a list is empty:

```c
list_empty(struct list_head *head)
```

To splice two unconnected lists together:

```c
list_splice(struct list_head *list, struct list_head *head)
```

This function splices together two lists by inserting the list pointed to by list to the given list after the element head.

To splice two unconnected lists together and reinitialize the old list:

```c
list_splice_init(struct list_head *list, struct list_head *head)
```

This function works the same as list_splice(), except that the emptied list pointed to by list is reinitialized.

### Traversing Linked Lists

#### The Basic Approach
The most basic way to iterate over a list is with the list\_for\_each() macro. The macro takes two parameters, both list_head structures. The first is a pointer used to point to the current entry; it is a temporary variable that you must provide.The second is the list_head acting as the head node of the list you want to traverse. On each iteration of the loop, the first parameter points to the next entry in the list, until each entry has been visited. Usage is as follows:

```c
struct list_head *p;

list_for_each(p, fox_list) {
    /* p points to an entry in the list */
}
```

A pointer to the list structure is usually no good; what we need is a pointer to the structure that contains the list_head. We can use the macro list_entry(), which we discussed earlier, to retrieve the structure that contains a given list_head. For example:

```c
struct list_head *p; 
struct fox *f;
list_for_each(p, &fox_list) {
    /* f points to the structure in which the list is embedded */ 
    f = list_entry(p, struct fox, list);
}
```

#### The Usable Approach
Consequently, most kernel code uses the list\_for\_each\_entry() macro to iterate over a linked list. This macro handles the work performed by list\_entry(), making list iteration simple:

```c
list_for_each_entry(pos, head, member)
```

#### Iterating Through a List Backward
The macro list\_for\_each\_entry\_reverse() works just like list\_for\_each\_entry(), except that it moves through the list in reverse.

#### Iterating While Removing
The standard list iteration methods are not appropriate if you are removing entries from the list as you iterate. The standard methods rely on the fact that the list entries are not changing out from under them.

This is a common pattern in loops, and programmers solve it by storing the next (or previous) pointer in a temporary variable prior to a potential removal operation. The Linux kernel provides a routine to handle this situation for you:

```c
list_for_each_entry_safe(pos, next, head, member)
```

The next pointer is used by the list_for_each_entry_safe() macro to store the next entry in the list, making it safe to remove the current entry.

## Queues

### kfifo
Linux’s kfifo works like most other queue abstractions, providing two primary operations: enqueue (unfortunately named in) and dequeue (out). The kfifo object maintains two off- sets into the queue: an in offset and an out offset. The in offset is the location in the queue to which the next enqueue will occur. The out offset is the location in the queue from which the next dequeue will occur.

### Creating a Queue
To use a kfifo, you must first define and initialize it. As with most kernel objects, you can do this dynamically or statically. The most common method is dynamic:

```c
int kfifo_alloc(struct kfifo *fifo, unsigned int size, gfp_t gfp_mask);
```

This function creates and initializes a kfifo with a queue of size bytes. The kernel uses the gfp mask gfp_mask to allocate the queue.

If you want to allocate the buffer yourself, you can:

```c
void kfifo_init(struct kfifo *fifo, void *buffer, unsigned int size);
```

This function creates and initializes a kfifo that will use the size bytes of memory pointed at by buffer for its queue. With both kfifo_alloc() and kfifo_init(), size must be a power of two.

Statically declaring a kfifo is simpler, but less common:

```c
DECLARE_KFIFO(name, size); 
INIT_KFIFO(name);
```

### Enqueuing Data
When your kfifo is created and initialized, enqueuing data into the queue is performed via the kfifo_in() function:

```c
unsigned int kfifo_in(struct kfifo *fifo, const void *from, unsigned int len);
```


This function copies the len bytes starting at from into the queue represented by fifo. On success it returns the number of bytes enqueued. If less than len bytes are free in the queue, the function copies only up to the amount of available bytes. Thus the return value can be less than len or even zero, if nothing was copied.

### Dequeuing Data
When you add data to a queue with kfifo_in(), you can remove it with kfifo_out(): 

```c
unsigned int kfifo_out(struct kfifo *fifo, void *to, unsigned int len);
```

This function copies at most len bytes from the queue pointed at by fifo to the buffer pointed at by to. On success the function returns the number of bytes copied. If less than len bytes are in the queue, the function copies less than requested.

If you want to “peek” at data within the queue without removing it, you can use kfifo_out_peek():

```c
unsigned int kfifo_out_peek(struct kfifo *fifo, void *to, unsigned int len, unsigned offset);
```

The parameter offset specifies an index into the queue; specify zero to read from the head of the queue, as kfifo_out() does.

### Obtaining the Size of a Queue
To obtain the total size in bytes of the buffer used to store a kfifo’s queue, call kfifo_size():

```c
static inline unsigned int kfifo_size(struct kfifo *fifo);
```

In another example of horrible kernel naming, use kfifo_len() to obtain the number of bytes enqueued in a kfifo:

```c
static inline unsigned int kfifo_len(struct kfifo *fifo);
```

To find out the number of bytes available to write into a kfifo, call kfifo_avail(): 

```c
static inline unsigned int kfifo_avail(struct kfifo *fifo);
```

### Resetting and Destroying the Queue
To reset a kfifo, jettisoning all the contents of the queue, call kfifo_reset(): 

```c
static inline void kfifo_reset(struct kfifo *fifo);
```

To destroy a kfifo allocated with kfifo_alloc(), call kfifo_free(): 

```c
void kfifo_free(struct kfifo *fifo);
```

## Maps
Although a hash table is a type of map, not all maps are implemented via hashes. Instead of a hash table, maps can also use a self-balancing binary search tree to store their data.Although a hash offers better average-case asymptotic complexity, a binary search tree has better worst-case behavior. A binary search tree also enables order preservation, enabling users to efficiently iterate over the entire collection in a sorted order. Finally, a binary search tree does not require a hash function; instead, any key type is suitable so long as it can define the <= operator.

The Linux kernel provides a simple and efficient map data structure, but it is not a general-purpose map. Instead, it is designed for one specific use case: mapping a unique identification number (UID) to a pointer. Linux’s implementation also piggybacks an allocate operation on top of the add operation. This allocate operation not only adds a UID/value pair to the map but also generates the UID.

The idr data structure is used for mapping user-space UIDs, such as inotify watch descriptors or POSIX timer IDs, to their associated kernel data structure, such as the inotify_watch or k_itimer structures, respectively. This map is called idr.

### Initializing an idr
Setting up an idr is easy. First you statically define or dynamically allocate an idr structure.Then you call idr_init():

```c
void idr_init(struct idr *idp);
```

### Allocating a New UID
Once you have an idr set up, you can allocate a new UID, which is a two-step process:

* First you tell the idr that you want to allocate a new UID, allowing it to resize the backing tree as necessary.
* Then, with a second call, you actually request the new UID. 

This complication exists to allow you to perform the initial resizing, which may require a memory allocation, without a lock.（没理解）

The first function, to resize the backing tree, is idr_pre_get(): 

```c
int idr_pre_get(struct idr *idp, gfp_t gfp_mask);
```

This function will, if needed to fulfill a new UID allocation, resize the idr pointed at by idp. If a resize is needed, the memory allocation will use the gfp flags gfp_mask. You do not need to synchronize concurrent access to this call. Inverted from nearly every other function in the kernel, **idr_pre_get() returns one on success and zero on error—be careful!**

The second function, to actually obtain a new UID and add it to the idr, is idr_get_new():

```c
int idr_get_new(struct idr *idp, void *ptr, int *id);
```

This function uses the idr pointed at by idp to allocate a new UID and associate it with the pointer ptr. On success, the function returns zero and stores the new UID in id. On error, it returns a nonzero error code: -EAGAIN if you need to (again) call idr_pre_get() and -ENOSPC if the idr is full.

The function idr_get_new_above() enables the caller to specify a minimum UID value to return:

```c
int idr_get_new_above(struct idr *idp, void *ptr, int starting_id, int *id);
```

This works the same as idr_get_new(), except that the new UID is guaranteed to be equal to or greater than starting_id. Using this variant of the function allows idr users to ensure that a UID is never reused, allowing the value to be unique not only among currently allocated IDs but across the entirety of a system’s uptime.（保证id不会回收后被重用，即 id 在整个系统正常运行时间都是唯一的）

### Looking Up a UID

```c
void *idr_find(struct idr *idp, int id);
```

A successful call to this function returns the pointer associated with the UID id in the idr pointed at by idp. 

### Removing a UID
To remove a UID from an idr, use idr_remove(): 

```c
void idr_remove(struct idr *idp, int id);
```

A successful call to idr_remove() removes the UID id from the idr pointed at by idp. Unfortunately, idr_remove() has no way to signify error (for example if id is not in idp).

### Destroying an idr
Destroying an idr is a simple affair, accomplished with the idr_destroy() function: 

```c
void idr_destroy(struct idr *idp);
```

A successful call to idr_destroy() deallocates only unused memory associated with the idr pointed at by idp. It does not free any memory currently in use by allocated UIDs. 

To force the removal of all UIDs, you can call idr_remove_all():

```c
void idr_remove_all(struct idr *idp);
```

You would call idr_remove_all() on the idr pointed at by idp before calling idr_destroy(), ensuring that all idr memory was freed.

## Binary Trees
A tree is a data structure that provides a hierarchical tree-like structure of data. Mathematically, it is an acyclic, connected, directed graph in which each vertex (called a node) has zero or more outgoing edges and zero or one incoming edges. A binary tree is a tree in which nodes have at most two outgoing edges—that is, a tree in which nodes have zero, one, or two children.

### Binary Search Trees
A binary search tree (often abbreviated BST) is a binary tree with a specific ordering imposed on its nodes. The ordering is often defined via the following induction:
* The left subtree of the root contains only nodes with values less than the root.
* The right subtree of the root contains only nodes with values greater than the root. 
* All subtrees are also binary search trees.

### Self-Balancing Binary Search Trees
The depth of a node is measured by how many parent nodes it is from the root. Nodes at the “bottom” of the tree—those with no children—are called leaves. The height of a tree is the depth of the deepest node in the tree. **A balanced binary search tree is a binary search tree in which the depth of all leaves differs by at most one. A self-balancing binary search tree is a binary search tree that attempts, as part of its normal operations,to remain (semi) balanced.**

### Red-Black Trees
A red-black tree is a type of self-balancing binary search tree. Linux’s primary binary tree data structure is the red-black tree. Red-black trees have a special color attribute, which is either red or black. Red-black trees remain semi-balanced by enforcing that the following six properties remain true:

* All nodes are either red or black.
* Leaf nodes are black.
* Leaf nodes do not contain data.
* All non-leaf nodes have two children.
* If a node is red, both of its children are black.
* The path from a node to one of its leaves contains the same number of black nodes as the shortest path to any of its other leaves.

Taken together, these properties ensure that the deepest leaf has a depth of no more than double that of the shallowest leaf. Consequently, the tree is always semi-balanced.

### rbtrees
The Linux implementation of red-black trees is called rbtrees. They are defined in lib/rbtree.c and declared in <linux/rbtree.h>.

The root of an rbtree is represented by the rb_root structure. To create a new tree, we allocate a new rb_root and initialize it to the special value RB_ROOT:

```c
struct rb_root root = RB_ROOT;
```

The rbtree implementation does not provide search and insert routines. Users of rbtrees are expected to define their own. This is because C does not make generic programming easy, and the Linux kernel developers believed the most efficient way to implement search and insert was to require each user to do so manually, using provided rbtree helper functions but their own comparison operators.

The following function implements a search of Linux’s page cache for a chunk of a file (represented by an inode and offset pair). Each inode has its own rbtree, keyed off of page offsets into file. This function thus searches the given inode’s rbtree for a matching offset value:

```c
struct page * rb_search_page_cache(struct inode *inode, unsigned long offset)
{
    struct rb_node *n = inode->i_rb_page_cache.rb_node;
    while (n) {
        struct page *page = rb_entry(n, struct page, rb_page_cache);
        if (offset < page->offset) 
            n = n->rb_left;
        else if (offset > page->offset) 
            n = n->rb_right;
        else
            return page;
    }
    return NULL;
}
```

Insert is even more complicated because it implements both search and insertion logic:

```c
struct page * rb_insert_page_cache(struct inode *inode, 
                                   unsigned long offset, 
                                   struct rb_node *node)
{
    struct rb_node **p = &inode->i_rb_page_cache.rb_node; 
    struct rb_node *parent = NULL;
    struct page *page;
    while (*p) {
        parent = *p;
        page = rb_entry(parent, struct page, rb_page_cache);

        if (offset < page->offset) 
            p = &(*p)->rb_left;
        else if (offset > page->offset) 
            p = &(*p)->rb_right;
        else
            return page;
    }
    rb_link_node(node, parent, p); 
    rb_insert_color(node, &inode->i_rb_page_cache);
    return NULL;
}
```

Unlike with search, however, the function is hoping not to find a matching offset but, instead, reach the leaf node that is the correct insertion point for the new offset. When the insertion point is found,rb_link_node() is called to insert the new node at the given spot. rb_insert_color() is then called to perform the complicated rebalancing dance.

## What Data Structure to Use, When
If your primary access method is iterating over all your data, use a linked list. Intuitively, no data structure can provide better than linear complexity when visiting every element.
If your code follows the producer/consumer pattern, use a queue, particularly if you want (or can cope with) a fixed-size buffer. Queues make adding and removing items simple and efficient, and they provide first-in, first-out (FIFO) semantics, which is what most producer/consumer use cases demand. On the other hand, if you need to store an unknown, potentially large number of items, a linked list may make more sense, because you can dynamically add any number of items to the list.
If you need to map a UID to an object, use a map. Maps make such mappings easy and efficient, and they also maintain and allocate the UID for you.
If you need to store a large amount of data and look it up efficiently, consider a red-black tree. If you are not performing many time-critical look-up operations, a red-black tree probably isn’t your best bet. In that case, favor a linked list.

# Chapter7. Interrupts and Interrupt Handlers

How can the processor work with hardware without impacting the machine’s overall performance? One answer to this question is polling. Periodically, the kernel can check the status of the hardware in the system and respond accordingly. Polling incurs overhead, however, because it must occur repeatedly regardless of whether the hardware is active or ready. A better solution is to provide a mechanism for the hardware to signal to the kernel when attention is needed. This mechanism is called an interrupt. 

## Interrupts
Interrupts enable hardware to signal to the processor. For example, as you type, the keyboard controller (the hardware device that manages the keyboard) issues an electrical signal to the processor to alert the operating system to newly available key presses. These electrical signals are interrupts. The processor receives the interrupt and signals the operating system to enable the operating system to respond to the new data. Hardware devices generate interrupts asynchronously with respect to the processor clock—they can occur at any time. Consequently, the kernel can be interrupted at any time to process interrupts.

An interrupt is physically produced by electronic signals originating from hardware devices and directed into input pins on an interrupt controller, a simple chip that multiplexes multiple interrupt lines into a single line to the processor. Upon receiving an interrupt, the interrupt controller sends a signal to the processor.The processor detects this signal and interrupts its current execution to handle the interrupt. The processor can then notify the operating system that an interrupt has occurred, and the operating system can handle the interrupt appropriately.

Different devices can be associated with different interrupts by means of a unique value associated with each interrupt. These interrupt values are often called interrupt request (IRQ) lines. Each IRQ line is assigned a numeric value.

Interrupts associated with devices on the PCI bus, for example, generally are dynamically assigned. Other non-PC architectures have similar dynamic assignments for interrupt values. The important notion is that a specific interrupt is associated with a specific device, and the kernel knows this.

In OS texts, exceptions are often discussed at the same time as interrupts. **Unlike interrupts, exceptions occur synchronously with respect to the processor clock. Indeed, they are often called synchronous interrupts.** Exceptions are produced by the processor while executing instructions either in response to a programming error (for example, divide by zero) or abnormal conditions that must be handled by the kernel. Because many processor architectures handle exceptions in a similar manner to interrupts, the kernel infrastructure for handling the two is similar.

You are already familiar with one exception: In the previous chapter, you saw how system calls on the x86 architecture are implemented by the issuance of a software interrupt, which traps into the kernel and causes execution of a special system call handler. Interrupts work in a similar way, you will see, except hardware—not software—issues interrupts.


## Interrupt Handlers
The function the kernel runs in response to a specific interrupt is called an interrupt handler or interrupt service routine (ISR). Each device that generates interrupts has an associated interrupt handler. The interrupt handler for a device is part of the device’s driver—the kernel code that manages the device.

In Linux, interrupt handlers are normal C functions.They match a specific prototype, which enables the kernel to pass the handler information in a standard way, but otherwise they are ordinary functions. What differentiates interrupt handlers from other kernel functions is that the kernel invokes them in response to interrupts and that they run in a special context called interrupt context. This special context is occasionally called atomic context because, as we shall see, **code executing in this context is unable to block.**

At the very least, an interrupt handler’s job is to acknowledge the interrupt’s receipt to the hardware: Hey, hardware, I hear ya; now get back to work!

## Top Halves Versus Bottom Halves
These two goals—that an interrupt handler execute quickly and perform a large amount of work—clearly conflict with one another. Because of these competing goals, the processing of interrupts is split into two parts, or halves. **The interrupt handler is the top half.** The top half is run immediately upon receipt of the interrupt and performs only the work that is time-critical, such as acknowledging receipt of the interrupt or resetting the hardware. The bottom half runs in the future, at a more convenient time, with all interrupts enabled. 

When network cards receive packets from the network, they need to alert the kernel of their availability. They want and need to do this immediately, to optimize network throughput and latency and avoid timeouts.Thus, they immediately issue an interrupt: Hey, kernel, I have some fresh packets here! The kernel responds by executing the network card’s registered interrupt.

The interrupt runs, acknowledges the hardware, copies the new networking packets into main memory, and readies the network card for more packets.These jobs are the important, time-critical, and hardware-specific work. The kernel generally needs to quickly copy the networking packet into main memory because the network data buffer on the networking card is fixed and miniscule in size, particularly compared to main memory. Delays in copying the packets can result in a buffer overrun, with incoming packets overwhelming the networking card’s buffer and thus packets being dropped. After the networking data is safely in the main memory, the interrupt’s job is done, and it can return control of the system to whatever code was interrupted when the interrupt was generated. The rest of the processing and handling of the packets occurs later, in the bottom half. 

## Registering an Interrupt Handler
Drivers can register an interrupt handler and enable a given interrupt line for handling with the function request_irq(), which is declared in <linux/interrupt.h>:

```c
typedef irqreturn_t (*irq_handler_t)(int, void *);

/* request_irq: allocate a given interrupt line */ 
int request_irq(unsigned int irq,
                irq_handler_t handler, 
                unsigned long flags, 
                const char *name, 
                void *dev)
```

The first parameter, irq, specifies the interrupt number to allocate. The second parameter, handler, is a function pointer to the actual interrupt handler that services this interrupt.

### Interrupt Handler Flags
The third parameter, flags, can be either zero or a bit mask of one or more of the flags defined in <linux/interrupt.h>. Among these flags, the most important are:

* IRQF_DISABLED
  When set, this flag instructs the kernel to disable all interrupts when executing this interrupt handler. When unset, interrupt handlers run with all interrupts except their own enabled. Most interrupt handlers do not set this flag, as disabling all interrupts is bad form.

* IRQF_SAMPLE_RANDOM
  This flag specifies that interrupts generated by this device should contribute to the kernel entropy pool. The kernel entropy pool provides truly random numbers derived from various random events. If this flag is specified, the timing of interrupts from this device are fed to the pool as entropy. Do not set this if your device issues interrupts at a predictable rate (for example, the system timer) or can be influenced by external attackers (for example, a networking device).

* IRQF_TIMER
  This flag specifies that this handler processes interrupts for the system timer.

* IRQF_SHARED
  This flag specifies that the interrupt line can be shared among multiple interrupt handlers. Each handler registered on a given line must specify this flag; otherwise, only one handler can exist per line.

The fourth parameter, name, is an ASCII text representation of the device associated with the interrupt. These text names are used by /proc/irq and /proc/interrupts for communication with the user, which is discussed shortly.

The fifth parameter, dev, is used for shared interrupt lines. When an interrupt handler is freed, dev provides a unique cookie to enable the removal of only the desired interrupt handler from the interrupt line. Without this parameter, it would be impossible for the kernel to know which handler to remove on a given interrupt line. A common practice is to pass the driver’s device structure: This pointer is unique and might be useful to have within the handlers.

On success, request_irq() returns zero. A nonzero value indicates an error, in which case the specified interrupt handler was not registered. A common error is -EBUSY, which denotes that the given interrupt line is already in use.

Note that **request_irq() can sleep and therefore cannot be called from interrupt context or other situations where code cannot block.** It is a common mistake to call request_irq() when it is unsafe to sleep. This is partly because of why request_irq() can block: It is indeed unclear. On registration, an entry corresponding to the interrupt is created in /proc/irq. The function proc_mkdir() creates new procfs entries. This function calls proc_create() to set up the new procfs entries, which in turn calls kmalloc() to allocate memory. As you will see in Chapter 12,“Memory Management,”kmalloc() can sleep. So there you go!


### Freeing an Interrupt Handler
When your driver unloads, you need to unregister your interrupt handler and potentially disable the interrupt line.To do this, call

```c
void free_irq(unsigned int irq, void *dev)
```

If the specified interrupt line is not shared, this function removes the handler and disables the line. If the interrupt line is shared, the handler identified via dev is removed, but the interrupt line is disabled only when the last handler is removed. With shared interrupt lines, a unique cookie is required to differentiate between the multiple handlers that can exist on a single line and enable free_irq() to remove only the correct handler. A call to free_irq() must be made from process context.

## Writing an Interrupt Handler
The following is a declaration of an interrupt handler:

```c
static irqreturn_t intr_handler(int irq, void *dev)
```

Note that this declaration matches the prototype of the handler argument given to request_irq(). The first parameter, irq, is the numeric value of the interrupt line the handler is servicing. This value is passed into the handler, but it is not used very often, except in printing log messages.

The second parameter, dev, is a generic pointer to the same dev that was given to request_irq() when the interrupt handler was registered. 

The return value of an interrupt handler is the special type irqreturn_t. An interrupt handler can return two special values, IRQ_NONE or IRQ_HANDLED. The former is returned when the interrupt handler detects an interrupt for which its device was not the originator. The latter is returned if the interrupt handler was correctly invoked, and its device did indeed cause the interrupt. Alternatively, IRQ_RETVAL(val) may be used. If val is nonzero, this macro returns IRQ_HANDLED. Otherwise, the macro returns IRQ_NONE.

Note the curious return type, irqreturn_t, which is simply an int. This value provides backward compatibility with earlier kernels, which did not have this feature; before 2.6, interrupt handlers returned void.

Interrupt handlers in Linux need not be reentrant. When a given interrupt handler is executing, the corresponding interrupt line is masked out on all processors, preventing another interrupt on the same line from being received. Normally all other interrupts are enabled, so other interrupts are serviced, but the current line is always disabled. Consequently, the same interrupt handler is never invoked concurrently to service a nested interrupt. This greatly simplifies writing your interrupt handler.

### Shared Handlers
A shared handler is registered and executed much like a nonshared handler. Following are three main differences:
* The IRQF_SHARED flag must be set in the flags argument to request_irq().
* The dev argument must be unique to each registered handler. A pointer to any per-device structure is sufficient; a common choice is the device structure as it is both unique and potentially useful to the handler.You cannot pass NULL for a shared handler!
* The interrupt handler must be capable of distinguishing whether its device actually generated an interrupt. This requires both hardware support and associated logic in the interrupt handler. If the hardware did not offer this capability, there would be no way for the interrupt handler to know whether its associated device or some other device sharing the line caused the interrupt.

When request_irq() is called with IRQF_SHARED specified, the call succeeds only if the interrupt line is currently not registered, or if all registered handlers on the line also specified IRQF_SHARED. Shared handlers, however, can mix usage of IRQF_DISABLED.

When the kernel receives an interrupt, it invokes sequentially each registered handler on the line. Therefore, it is important that the handler be capable of distinguishing whether it generated a given interrupt. The handler must quickly exit if its associated device did not generate the interrupt. This requires the hardware device to have a status register (or similar mechanism) that the handler can check.

### A Real-Life Interrupt Handler
Let’s look at a real interrupt handler, from the real-time clock (RTC) driver, found in drivers/char/rtc.c. It is a device, separate from the system timer, which sets the system clock, provides an alarm, or supplies a periodic timer. On most architectures, the system clock is set by writing the desired time into a specific register or I/O range. Any alarm or periodic timer functionality is normally implemented via interrupt.

When the RTC driver loads, the function rtc_init() is invoked to initialize the driver. One of its duties is to register the interrupt handler:

```c
/* register rtc_interrupt on rtc_irq */
if (request_irq(rtc_irq, rtc_interrupt, IRQF_SHARED, "rtc", (void *)&rtc_port)) {
    printk(KERN_ERR "rtc: cannot register IRQ %d\n", rtc_irq); 
    return -EIO;
}
```

Finally, the handler itself:

```c
static irqreturn_t rtc_interrupt(int irq, void *dev) {
    /*
    * Can be an alarm interrupt, update complete interrupt,
    * or a periodic interrupt. We store the status in the
    * low byte and the number of interrupts received since
    * the last read in the remainder of rtc_irq_data. 
    */
    spin_lock(&rtc_lock);
    rtc_irq_data += 0x100;
    rtc_irq_data &= ~0xff;
    rtc_irq_data |= (CMOS_READ(RTC_INTR_FLAGS) & 0xF0);

    if (rtc_status & RTC_TIMER_ON)
        mod_timer(&rtc_irq_timer, jiffies + HZ/rtc_freq + 2*HZ/100);
    spin_unlock(&rtc_lock);

    /*
    * Now do the rest of the actions 
    */
    spin_lock(&rtc_task_lock); 
    if (rtc_callback)
        rtc_callback->func(rtc_callback->private_data); 
    spin_unlock(&rtc_task_lock);

    wake_up_interruptible(&rtc_wait); 
    kill_fasync(&rtc_async_queue, SIGIO, POLL_IN); 
    return IRQ_HANDLED;
}
```

Finally, this function returns IRQ_HANDLED to signify that it properly handled this device. Because the interrupt handler does not support sharing, and there is no mecha- nism for the RTC to detect a spurious interrupt, this handler always returns IRQ_HANDLED.

## Interrupt Context
When executing an interrupt handler, the kernel is in interrupt context. Recall that process context is the mode of operation the kernel is in while it is executing on behalf of a process. In process context, the current macro points to the associated task. Furthermore, because a process is coupled to the kernel in process context, process context can sleep or otherwise invoke the scheduler.

Interrupt context, on the other hand, is not associated with a process. The current macro is not relevant (although it points to the interrupted process).Without a backing process, interrupt context cannot sleep—how would it ever reschedule? Therefore, **you cannot call certain functions from interrupt context.** If a function sleeps, you cannot use it from your interrupt handler—this limits the functions that one can call from an interrupt handler.

Interrupt context is time-critical because the interrupt handler interrupts other code. Code should be quick and simple. Always keep in mind that your interrupt handler has interrupted other code. As much as possible, **work should be pushed out from the interrupt handler and performed in a bottom half, which runs at a more convenient time.**

The setup of an interrupt handler’s stacks is a configuration option. Historically, interrupt handlers did not receive their own stacks. Instead, they would share the stack of the process that they interrupted. The kernel stack is two pages in size; typically, that is 8KB on 32-bit architectures and 16KB on 64-bit architectures.

Early in the 2.6 kernel process, an option was added to reduce the stack size from two pages down to one, providing only a 4KB stack on 32-bit systems. **This reduced memory pressure because every process on the system previously needed two pages of contiguous, nonswappable kernel memory.** To cope with the reduced stack size, interrupt handlers were given their own stack, one stack per processor, one page in size.This stack is referred to as the interrupt stack.

## Implementing Interrupt Handlers
Perhaps not surprising, the implementation of the interrupt handling system in Linux is architecture-dependent.The implementation depends on the processor, the type of interrupt controller used, and the design of the architecture and machine.

Following is a diagram of the path an interrupt takes through hardware and the kernel.

<img src="/assets/images/linux-kernel-development-note/illustration-2.png" width="800" />

A device issues an interrupt by sending an electric signal over its bus to the interrupt controller. If the interrupt line is enabled (they can be masked out), the interrupt controller sends the interrupt to the processor. In most architectures, this is accomplished by an electrical signal sent over a special pin to the processor. Unless interrupts are disabled in the processor (which can also happen), the processor immediately stops what it is doing, disables the interrupt system, and jumps to a predefined location in memory and executes the code located there.This predefined point is set up by the kernel and is the entry point for interrupt handlers.

For each interrupt line, the processor jumps to a unique location in memory and executes the code located there. In this manner, the kernel knows the IRQ number of the incoming interrupt. The initial entry point simply saves this value and stores the current register values (which belong to the interrupted task) on the stack; then the kernel calls do_IRQ(). From here onward, most of the interrupt handling code is written in C; however, it is still architecture-dependent.

The do_IRQ() function is declared as 

```c
unsigned int do_IRQ(struct pt_regs regs)
```

Because the C calling convention places function arguments at the top of the stack, the pt_regs structure contains the initial register values that were previously saved in the assembly entry routine. Because the interrupt value was also saved, do_IRQ() can extract it. After the interrupt line is calculated, do_IRQ() acknowledges the receipt of the interrupt and disables interrupt delivery on the line. 

Next, do_IRQ() ensures that a valid handler is registered on the line and that it is enabled and not currently executing. If so, it calls handle_IRQ_event(), defined in kernel/irq/handler.c, to run the installed interrupt handlers for the line.

```c
/**
* handle_IRQ_event - irq action chain handler
* @irq: the interrupt number
* @action: the interrupt action chain for this irq 
*
* Handles the action chain of an irq event 
*/
irqreturn_t handle_IRQ_event(unsigned int irq, struct irqaction *action) {
    irqreturn_t ret, retval = IRQ_NONE; 
    unsigned int status = 0;

    if (!(action->flags & IRQF_DISABLED)) 
        local_irq_enable_in_hardirq();

    do {
        trace_irq_handler_entry(irq, action);
        ret = action->handler(irq, action->dev_id); 
        trace_irq_handler_exit(irq, action, ret);
        
        switch (ret) {
            case IRQ_WAKE_THREAD:
            /*
            * Set result to handled so the spurious check
            * does not trigger. 
            */
            ret = IRQ_HANDLED;
            /*
            * Catch drivers which return WAKE_THREAD but
            * did not set up a thread function 
            */
            if (unlikely(!action->thread_fn)) {
                warn_no_thread(irq, action); 
                break;
            }
            /*
            * Wake up the handler thread for this
            * action. In case the thread crashed and was
            * killed we just pretend that we handled the
            * interrupt. The hardirq handler above has
            * disabled the device interrupt, so no irq
            * storm is lurking. 
            */
            if (likely(!test_bit(IRQTF_DIED, 
                                 &action->thread_flags))) {
                set_bit(IRQTF_RUNTHREAD, &action->thread_flags); 
                wake_up_process(action->thread);
            }

            case IRQ_HANDLED:
                status |= action->flags; 
                break;
            default: 
                break;
        }

        retval |= ret;
        action = action->next; 
    } while(action);

    /* Fall through to add to randomness */ 
    if (status & IRQF_SAMPLE_RANDOM) 
        add_interrupt_randomness(irq);
    local_irq_disable(); 
    
    return retval;
}
```

First, because the processor disabled interrupts, they are turned back on unless IRQF_DISABLED was specified during the handler’s registration. （因为 CPU 在收到中断后就立即禁止了中断，这里检查 IRQF_DISABLED 标志位，如果没有设置，则重新开启中断）Next, each potential handler is executed in a loop. If this line is not shared, the loop terminates after the first iteration. After that, add_interrupt_randomness() is called if IRQF_SAMPLE_RANDOM was specified during registration. This function uses the timing of the interrupt to generate entropy for the random number generator. Finally, interrupts are again disabled (do_IRQ() expects them still to be off) and the function returns. Back in do_IRQ(), the function cleans up and returns to the initial entry point, which then jumps to ret_from_intr().

The routine ret_from_intr() is, as with the initial entry code, written in assembly. **This routine checks whether a reschedule is pending.** If a reschedule is pending, and the kernel is returning to user-space (that is, the interrupt interrupted a user process), schedule() is called. If the kernel is returning to kernel-space (that is, the interrupt interrupted the kernel itself), schedule() is called only if the preempt_count is zero.  After schedule() returns, or if there is no work pending, the initial registers are restored and the kernel resumes whatever was interrupted.

## /proc/interrupts
Procfs is a virtual filesystem that exists only in kernel memory and is typically mounted at /proc. Reading or writing files in procfs invokes kernel functions that simulate reading or writing from a real file. A relevant example is the /proc/interrupts file, which is populated with statistics related to interrupts on the system. 

## Interrupt Control
The Linux kernel implements a family of interfaces for manipulating the state of inter- rupts on a machine. These interfaces enable you to disable the interrupt system for the current processor or mask out an interrupt line for the entire machine. These routines are all architecture-dependent and can be found in <asm/system.h> and <asm/irq.h>.

Reasons to control the interrupt system generally boil down to needing to provide synchronization. By disabling interrupts, you can guarantee that an interrupt handler will not preempt your current code. Moreover, disabling interrupts also disables kernel preemption. Neither disabling interrupt delivery nor disabling kernel preemption provides any protection from concurrent access from another processor, however. Because Linux supports multiple processors, kernel code more generally needs to obtain some sort of lock to prevent another processor from accessing shared data simultaneously. These locks are often obtained in conjunction with disabling local interrupts.

### Disabling and Enabling Interrupts
To disable interrupts locally for the current processor (and only the current processor) and then later reenable them, do the following:

```c
local_irq_disable();
/* interrupts are disabled .. */ 
local_irq_enable();
```

These functions are usually implemented as a single assembly operation. On x86, local_irq_disable() is a simple cli and local_irq_enable() is a simple sti instruction. cli and sti are the assembly calls to clear and set the allow interrupts flag, respectively.

A mechanism is needed to restore interrupts to a previous state.This is a common concern because a given code path in the kernel can be reached both with and without interrupts enabled, depending on the call chain. 

```c
unsigned long flags;
local_irq_save(flags); /* interrupts are now disabled */
/* ... */
local_irq_restore(flags); /* interrupts are restored to their previous state */
```

### Disabling a Specific Interrupt Line
In some cases, it is useful to disable only a specific interrupt line for the entire system. This is called **masking out** an interrupt line. Linux provides four interfaces for this task:

```
void disable_irq(unsigned int irq);
void disable_irq_nosync(unsigned int irq); 
void enable_irq(unsigned int irq);
void synchronize_irq(unsigned int irq);
```

The first two functions disable a given interrupt line in the interrupt controller. This disables delivery of the given interrupt to all processors in the system. Additionally, the disable_irq()function does not return until any currently executing handler completes. Thus, callers are assured not only that new interrupts will not be delivered on the given line, but also that any already executing handlers have exited.

The function synchronize_irq() waits for a specific interrupt handler to exit, if it is executing, before returning.

Calls to these functions nest. For each call to disable_irq() or disable_irq_nosync() on a given interrupt line, a corresponding call to enable_irq() is required. Only on the last call to enable_irq() is the interrupt line actually enabled.

All three of these functions can be called from interrupt or process context and do not sleep.  If calling from interrupt context, be careful! You do not want, for example, to enable an interrupt line while you are handling it.

### Status of the Interrupt System
The macro irqs_disabled(), defined in <asm/system.h>, returns nonzero if the interrupt system on the local processor is disabled. Otherwise, it returns zero.
Two macros, defined in <linux/hardirq.h>, provide an interface to check the kernel’s current context. They are

```
in_interrupt() 
in_irq()
```

The most useful is the first: It returns nonzero if the kernel is performing any type of interrupt handling. This includes either executing an interrupt handler or a bottom half handler. The macro in_irq() returns nonzero only if the kernel is specifically executing an interrupt handler. More often, you want to check whether you are in process context.That is, you want to ensure you are not in interrupt context.This is often the case because code wants to do something that can only be done from process context, such as sleep. If in_interrupt() returns zero, the kernel is in process context.


# Chapter8. Bottom Halves and Deferring Work

## Bottom Halves
Although it is not always clear how to divide the work between the top and bottom half, a couple of useful tips help:

* If the work is time sensitive, perform it in the interrupt handler.
* If the work is related to the hardware, perform it in the interrupt handler.
* If the work needs to ensure that another interrupt (particularly the same interrupt) does not interrupt it, perform it in the interrupt handler.
* For everything else, consider performing the work in the bottom half.

### A World of Bottom Halves
Multiple mechanisms are available for implementing a bottom half. These mechanisms are different interfaces and subsystems that enable you to implement bottom halves.

#### The Original “Bottom Half”
In the beginning, Linux provided only the “bottom half ” for implementing bottom halves. The infrastructure was also known as BH, which is what we will call it to avoid confusion with the generic term bottom half. The BH interface was simple, like most things in those good old days. It provided a statically created list of 32 bottom halves for the entire system. The top half could mark whether the bottom half would run by setting a bit in a 32-bit integer. Each BH was globally synchronized. No two could run at the same time, even on different processors.This was easy to use, yet inflexible; a simple approach, yet a bottleneck.

#### Task Queues
Later on, the kernel developers introduced task queues both as a method of deferring work and as a replacement for the BH mechanism. The kernel defined a family of queues. Each queue contained a linked list of functions to call. The queued functions were run at certain times, depending on which queue they were in. Drivers could register their bottom halves in the appropriate queue.

#### Softirqs and Tasklets
During the 2.3 development series, the kernel developers introduced softirqs and tasklets. With the exception of compatibility with existing drivers, softirqs and tasklets could com- pletely replace the BH interface. Softirqs are a set of statically defined bottom halves that can run simultaneously on any processor; even two of the same type can run concurrently. Tasklets, which have an awful and confusing name, are flexible, dynamically created bottom halves built on top of softirqs. Two different tasklets can run concurrently on different processors, but two of the same type of tasklet cannot run simultaneously.

For most bottom-half processing, the tasklet is sufficient. Softirqs are useful when performance is critical, such as with networking. Additionally, the task queue interface was replaced by the work queue interface. Work queues are a simple yet useful method of queuing work to later be performed in process context.

Another mechanism for deferring work is kernel timers. Unlike the mechanisms discussed in the chapter thus far, timers defer work for a specified amount of time. 

## Softirqs
The softirq code lives in the file kernel/softirq.c in the kernel source tree.

### Implementing Softirqs
Softirqs are statically allocated at compile time. Unlike tasklets, you cannot dynamically register and destroy softirqs. Softirqs are represented by the softirq_action structure, which is defined in <linux/interrupt.h>:

```c
struct softirq_action {
void (*action)(struct softirq_action *);
};
```

A 32-entry array of this structure is declared in kernel/softirq.c: 

```c
static struct softirq_action softirq_vec[NR_SOFTIRQS];
```

The number of registered softirqs is statically determined at compile time and cannot be changed dynamically. The kernel enforces a limit of 32 registered softirqs; in the current kernel, however, only nine exist.

#### The Softirq Handler
The prototype of a softirq handler, action, looks like 

```c
void softirq_handler(struct softirq_action *)
```

When the kernel runs a softirq handler, it executes this action function with a pointer to the corresponding softirq_action structure as its lone argument. For example, if my_softirq pointed to an entry in the softirq_vec array, the kernel would invoke the softirq handler function as

```c
my_softirq->action(my_softirq);
```

A softirq never preempts another softirq. The only event that can preempt a softirq is an interrupt handler. Another softirq—even the same one—can run on another processor, however.

#### Executing Softirqs
A registered softirq must be marked before it will execute. This is called raising the softirq. Usually, an interrupt handler marks its softirq for execution before returning. Then, at a suitable time, the softirq runs. Pending softirqs are checked for and executed in the following places:

* In the return from hardware interrupt code path
* In the ksoftirqd kernel thread
* In any code that explicitly checks for and executes pending softirqs, such as the networking subsystem

Regardless of the method of invocation, softirq execution occurs in __do_softirq(), which is invoked by do_softirq(). If there are pending softirqs, __do_softirq() loops over each one, invoking its handler. Let’s look at a simplified variant of the important part of __do_softirq():

```c
u32 pending;
pending = local_softirq_pending(); if (pending) {
struct softirq_action *h;
/* reset the pending bitmask */ set_softirq_pending(0);
h = softirq_vec; do {
if (pending & 1) h->action(h);
h++;
pending >>= 1; } while (pending);
}
```

### Using Softirqs
Softirqs are reserved for the most timing-critical and important bottom-half processing on the system. Currently, only two subsystems—networking and block devices—directly use softirqs.

#### Assigning an Index
You declare softirqs statically at compile time via an enum in <linux/interrupt.h>.The kernel uses this index, which starts at zero, as a relative priority. Softirqs with the lowest numerical priority execute before those with a higher numerical priority.

Creating a new softirq includes adding a new entry to this enum. When adding a new softirq, you might not want to simply add your entry to the end of the list, as you would elsewhere. Instead, you need to insert the new entry depending on the priority you want to give it. By convention, HI_SOFTIRQ is always the first and RCU_SOFTIRQ is always the last entry. A new entry likely belongs in between BLOCK_SOFTIRQ and TASKLET_SOFTIRQ.

#### Registering Your Handler
Next, the softirq handler is registered at run-time via open_softirq(), which takes two parameters: the softirq’s index and its handler function. The networking subsystem, for example, registers its softirqs like this, in net/core/dev.c:

```c
open_softirq(NET_TX_SOFTIRQ, net_tx_action); 
open_softirq(NET_RX_SOFTIRQ, net_rx_action);
```

The softirq handlers run with interrupts enabled and cannot sleep. While a handler runs, softirqs on the current processor are disabled. Another processor, however, can execute other softirqs. This means that any shared data—even global data used only within the softirq handler—needs proper locking.

The raison d’être to softirqs is scalability. If you do not need to scale to infinitely many processors, then use a tasklet. Tasklets are essentially softirqs in which multiple instances of the same handler cannot run concurrently on multiple processors.

#### Raising Your Softirq
After a handler is added to the enum list and registered via open_softirq(), it is ready to run. To mark it pending, so it is run at the next invocation of do_softirq(), call raise_softirq(). For example, the networking subsystem would call,

```c
raise_softirq(NET_TX_SOFTIRQ);
```

This raises the NET_TX_SOFTIRQ softirq. Its handler, net_tx_action(), runs the next time the kernel executes softirqs. This function disables interrupts prior to actually raising the softirq and then restores them to their previous state. If interrupts are already off, the function 
raise_softirq_irqoff() can be used as a small optimization. 

```c
/*
* interrupts must already be off! 
*/
raise_softirq_irqoff(NET_TX_SOFTIRQ);
```

Softirqs are most often raised from within interrupt handlers. In the case of interrupt handlers, the interrupt handler performs the basic hardware-related work, raises the softirq, and then exits.When processing interrupts, the kernel invokes do_softirq().The softirq then runs and picks up where the interrupt handler left off. 

## Tasklets
Tasklets are a bottom-half mechanism built on top of softirqs.

### Implementing Tasklets
Because tasklets are implemented on top of softirqs, they are softirqs. As discussed, tasklets are represented by two softirqs: HI_SOFTIRQ and TASKLET_SOFTIRQ. The only difference in these types is that the HI_SOFTIRQ-based tasklets run prior to the TASKLET_SOFTIRQ-based tasklets.

#### The Tasklet Structure
Tasklets are represented by the tasklet_struct structure. Each structure represents a unique tasklet.The structure is declared in <linux/interrupt.h>:

```c
struct tasklet_struct {
    struct tasklet_struct *next;  /* next tasklet in the list */
    unsigned long state;  /* state of the tasklet */
    atomic_t count;  /* reference counter */
    void (*func)(unsigned long);  /* tasklet handler function */
    unsigned long data;  /* argument to the tasklet function */
};
```

The state member is exactly zero, TASKLET_STATE_SCHED, or TASKLET_STATE_RUN. TASKLET_STATE_SCHED denotes a tasklet that is scheduled to run, and TASKLET_STATE_RUN denotes a tasklet that is running. As an optimization, TASKLET_STATE_RUN is used only on multiprocessor machines because a uniprocessor machine always knows whether the tasklet is running. 

The count field is used as a reference count for the tasklet. If it is nonzero, the tasklet is disabled and cannot run; if it is zero, the tasklet is enabled and can run if marked pending.

#### Scheduling Tasklets
Scheduled tasklets (the equivalent of raised softirqs) are stored in two per-processor structures: tasklet_vec (for regular tasklets) and tasklet_hi_vec (for high-priority tasklets). Both of these structures are linked lists of tasklet_struct structures. Each tasklet_struct structure in the list represents a different tasklet.

Tasklets are scheduled via the tasklet_schedule() and tasklet_hi_schedule() functions, which receive a pointer to the tasklet’s tasklet_struct as their lone argument. 

let’s look at the steps tasklet_schedule() undertakes:
* Check whether the tasklet’s state is TASKLET_STATE_SCHED. If it is, the tasklet is already scheduled to run and the function can immediately return.
* Call__tasklet_schedule().
* Save the state of the interrupt system, and then disable local interrupts.This ensures that nothing on this processor will mess with the tasklet code while tasklet_schedule() is manipulating the tasklets.
* Add the tasklet to be scheduled to the head of the tasklet_vec or tasklet_hi_vec linked list, which is unique to each processor in the system.
* Raise the TASKLET_SOFTIRQ or HI_SOFTIRQ softirq, so do_softirq() executes this tasklet in the near future.
* Restore interrupts to their previous state and return.

Because TASKLET_SOFTIRQ or HI_SOFTIRQ is now raised, do_softirq() executes the associated handlers. These handlers, tasklet_action() and tasklet_hi_action(), are the heart of tasklet processing. Let’s look at the steps these handlers perform:

* Disable local interrupt delivery (there is no need to first save their state because the code here is always called as a softirq handler and interrupts are always enabled) and retrieve the tasklet_vec or tasklet_hi_vec list for this processor.
* Clear the list for this processor by setting it equal to NULL.
* Enable local interrupt delivery.Again,there is no need to restore them to their previous state because this function knows that they were always originally enabled.
* Loop over each pending tasklet in the retrieved list.
* If this is a multiprocessing machine, check whether the tasklet is running on another processor by checking the TASKLET_STATE_RUN flag. If it is currently run- ning, do not execute it now and skip to the next pending tasklet. (Recall that only one tasklet of a given type may run concurrently.)
* If the tasklet is not currently running, set the TASKLET_STATE_RUN flag, so another processor will not run it.
* Check for a zero count value, to ensure that the tasklet is not disabled. If the tasklet is disabled, skip it and go to the next pending tasklet.
* We now know thatthetaskletisnotrunningelsewhere,ismarkedasrunningsoit will not start running elsewhere, and has a zero count value. Run the tasklet handler.
* After the tasklet runs, clear the TASKLET_STATE_RUN flag in the tasklet’s state field.
* Repeat for the next pending tasklet, until there are no more scheduled tasklets waiting to run.

### Using Tasklets

#### Declaring Your Tasklet
You can create tasklets statically or dynamically. What option you choose depends on whether you have (or want) a direct or indirect reference to the tasklet. If you are going to statically create the tasklet (and thus have a direct reference to it), use one of two macros in <linux/interrupt.h>:

```c
DECLARE_TASKLET(name, func, data) 
DECLARE_TASKLET_DISABLED(name, func, data);
```

The difference between the two macros is the initial reference count.The first macro creates the tasklet with a count of zero, and the tasklet is enabled.The second macro sets count to one, and the tasklet is disabled.

To initialize a tasklet given an indirect reference (a pointer) to a dynamically created struct tasklet_struct, t, call tasklet_init():

```c
tasklet_init(t, tasklet_handler, dev); /* dynamically as opposed to statically */
```

#### Writing Your Tasklet Handler
The tasklet handler must match the correct prototype:

```c
void tasklet_handler(unsigned long data)
```

As with softirqs, tasklets cannot sleep. This means you cannot use semaphores or other blocking functions in a tasklet. Tasklets also run with all interrupts enabled, so you must take precautions (for example, disable interrupts and obtain a lock) if your tasklet shares data with an interrupt handler. Unlike softirqs, however, two of the same tasklets never run concurrently.

#### Scheduling Your Tasklet
To schedule a tasklet for execution, tasklet_schedule() is called and passed a pointer to the relevant tasklet_struct:

```c
tasklet_schedule(&my_tasklet); /* mark my_tasklet as pending */
```

You can disable a tasklet via a call to tasklet_disable(), which disables the given tasklet. If the tasklet is currently running, the function will not return until it finishes exe- cuting.

### ksoftirqd
Softirq (and thus tasklet) processing is aided by a set of per-processor kernel threads. These kernel threads help in the processing of softirqs when the system is overwhelmed with softirqs.


# Chapter9. An Introduction to Kernel Synchronization
Years ago, before Linux supported symmetrical multiprocessing, preventing concurrent access of data was simple. Because only a single processor was supported, the only way data could be concurrently accessed was if an interrupt occurred or if kernel code explicitly rescheduled and enabled another task to run.

Symmetrical multiprocessing support was introduced in the 2.0 kernel and has been continually enhanced ever since. Multiprocessing support implies that kernel code can simultaneously run on two or more processors.

Today, a number of scenarios enable for concurrency inside the kernel, and they all require protection.

## Critical Regions and Race Conditions
Code paths that access and manipulate shared data are called critical regions (also called critical sections). It is usually unsafe for multiple threads of execution to access the same resource simultaneously. To prevent concurrent access during critical regions, the pro- grammer must ensure that code executes atomically—that is, operations complete without interruption as if the entire critical region were one indivisible instruction. Ensuring that unsafe concurrency is prevented and that race conditions do not occur is called synchronization.

## Locking
Notice that locks are advisory and voluntary. Locks are entirely a programming construct that the programmer must take advantage of. Nothing prevents you from writing code that manipulates the fictional queue without the appropriate lock. 

Locks come in various shapes and sizes—Linux alone implements a handful of different locking mechanisms. The most significant difference between the various mechanisms is the behavior when the lock is unavailable because another thread already holds it— some lock variants busy wait, whereas other locks put the current task to sleep until the lock becomes available.

Locks are implemented using atomic operations that ensure no race exists. A single instruction can verify whether the key is taken and,if not, seize it. How this is done is architecture-specific, but almost all processors implement an atomic test and set instruction that tests the value of an integer and sets it to a new value only if it is zero. A value of zero means unlocked. On the popular x86 architecture, locks are implemented using such a similar instruction called compare and exchange.

### Causes of Concurrency
In user-space, the need for synchronization stems from the fact that programs are scheduled preemptively at the will of the scheduler. 

This type of concurrency—in which two things do not actually happen at the same time but interleave with each other such that they might as well—is called pseudo-concurrency. If you have a symmetrical multiprocessing machine, two processes can actually be executed in a critical region at the exact same time.That is called true concurrency.

The kernel has similar causes of concurrency:

* Interrupts 
  An interrupt can occur asynchronously at almost any time, interrupting the currently executing code.
* Softirqs and tasklets
  The kernel can raise or schedule a softirq or tasklet at almost any time, interrupting the currently executing code.
* Kernel preemption 
  Because the kernel is preemptive, one task in the kernel can preempt another.
* Sleeping and synchronization with user-space
  A task in the kernel can sleep and thus invoke the scheduler, resulting in the running of a new process.
* Symmetrical multiprocessing
  Two or more processors can execute kernel code at exactly the same time.

Code that is safe from concurrent access from an interrupt handler is said to be interrupt-safe. Code that is safe from concurrency on symmetrical multiprocessing machines is SMP-safe. Code that is safe from concurrency with kernel preemption is preempt-safe.

### Knowing What to Protect
Obviously, any data that is local to one particular thread of execution does not need protection, because only that thread can access the data. For example, local automatic variables (and dynamically allocated data structures whose address is stored only on the stack) do not need any sort of locking because they exist solely on the stack of the executing thread. Likewise, data that is accessed by only a specific task does not require locking

A good rule of thumb is that if another thread of execution can access the data, the data needs some sort of locking; if anyone else can see it, lock it. Remember to lock data, not code.

Whenever you write kernel code, you should ask yourself these questions:

* Is the data global? Can a thread of execution other than the current one access it?
* Is the data shared between process context and interrupt context? Is it shared between two different interrupt handlers?
* If a process is preempted while accessing this data, can the newly scheduled process access the same data?
* Can the current process sleep (block) on anything? If it does, in what state does that leave any shared data?
* What prevents the data from being freed out from under me?
* What happens if this function is called again on another processor?
* Given the proceeding points, how am I going to ensure that my code is safe from concurrency?

## Deadlocks
Some kernels prevent this type of deadlock by providing recursive locks. These are locks that a single thread of execution may acquire multiple times. Linux, thankfully, does not provide recursive locks. This is widely considered a good thing. Although recursive locks might alleviate the self-deadlock problem, they very readily lead to sloppy locking semantics.

Prevention of deadlock scenarios is important. Although it is difficult to prove that code is free of deadlocks, you can write deadlock-free code.A few simple rules go a long way:

* Implement lock ordering. Nested locks must always be obtained in the same order. This prevents the deadly embrace deadlock. Document the lock ordering so others will follow it.
* Prevent starvation.Ask yourself,does this code always finish? If foo does not occur,will bar wait forever?
* Do not double acquire the same lock.
* Design for simplicity. Complexity in your locking scheme invites deadlocks. 

The order of unlock does not matter with respect to deadlock, although it is common practice to release the locks in an order inverse to that in which they were acquired.

## Contention and Scalability
The term lock contention, or simply contention, describes a lock currently in use but that another thread is trying to acquire. A lock that is highly contended often has threads waiting to acquire it. A highly contended lock can become a bottleneck in the system, quickly limiting its performance. 

Scalability is a measurement of how well a system can be expanded.

The granularity of locking is a description of the size or amount of data that a lock protects. A very coarse lock protects a large amount of data.

Generally, this scalability improvement is a good thing because it improves Linux’s performance on larger and more powerful systems. Rampant scalability “improvements” can lead to a decrease in performance on smaller SMP and UP machines, however, because smaller machines may not need such fine-grained locking but will nonetheless need to put up with the increased complexity and overhead. 



# Chapter10. Kernel Synchronization Methods

## Kernel Synchronization Methods
We start our discussion of synchronization methods with atomic operations because they are the foundation on which other synchronization methods are built. Atomic operations provide instructions that execute atomically—without interruption.

The kernel provides two sets of interfaces for atomic operations—one that operates on integers and another that operates on individual bits.

### Atomic Integer Operations
The atomic integer methods operate on a special data type,atomic_t. This special type is used, as opposed to having the functions work directly on the C int type, for several reasons:

* First, having the atomic functions accept only the atomic_t type ensures that the atomic operations are used only with these special types. Likewise, it also ensures that the data types are not passed to any nonatomic functions.
* Next, the use of atomic_t ensures the compiler does not (erroneously but cleverly) optimize access to the value—it is important the atomic operations receive the correct memory address and not an alias.
* Finally, use of atomic_t can hide any architecture-specific differences in its implementation.

Developers and their code once had to assume that an atomic_t was no larger than 24 bits in size. The SPARC port in Linux has an odd implementation of atomic operations: A lock was embedded in the lower 8 bits of the 32-bit int. Consequently, only 24 usable bits were available on SPARC machines. Although code that assumed that the full 32-bit range existed would work on other machines; it would have failed in strange and subtle ways on SPARC machines. Recently, clever hacks have allowed SPARC to provide a fully usable 32-bit atomic_t, and this limitation is no more.

The declarations needed to use the atomic integer operations are in <asm/atomic.h>.

Defining an atomic_t is done in the usual manner. Optionally, you can set it to an initial value:

```c
atomic_t v;  /* define v */
atomic_t u = ATOMIC_INIT(0);  /* define u and initialize it to zero */
```

Operations are all simple:

```
atomic_set(&v, 4); /* v = 4 (atomically) */
atomic_add(2, &v); /* v = v + 2 = 6 (atomically) */
atomic_inc(&v);    /* v = v + 1 = 7 (atomically) */
```

If you ever need to convert an atomic_t to an int, use atomic_read(): 

```c
printk(“%d\n”, atomic_read(&v)); /* will print “7” */
```

A common use of the atomic integer operations is to implement counters. Protecting a sole counter with a complex locking scheme is overkill, so instead developers use atomic_inc() and atomic_dec(), which are much lighter in weight.

Another use of the atomic integer operators is atomically performing an operation and testing the result. A common example is the atomic decrement and test:

```c
int atomic_dec_and_test(atomic_t *v)
```

This function decrements by one the given atomic value. If the result is zero, it returns true; otherwise, it returns false.

The atomic operations are typically implemented as inline functions with inline assembly.

### Atomic Bitwise Operations
In addition to atomic integer operations, the kernel also provides a family of functions that operate at the bit level. Not surprisingly, they are architecture-specific and defined in <asm/bitops.h>.

The bitwise functions operate on generic memory addresses.The arguments are a pointer and a bit number. Bit zero is the least significant bit of the given address. Consider an example:

```c
unsigned long word = 0;
set_bit(0, &word);      /* bit zero is now set (atomically) */
set_bit(1, &word);      /* bit one is now set (atomically) */
printk(“%ul\n”, word);  /* will print “3” */
clear_bit(1, &word);    /* bit one is now unset (atomically) */
change_bit(0, &word);   /* bit zero is flipped; now it is unset (atomically) */


/* atomically sets bit zero and returns the previous value (zero) */ 
if (test_and_set_bit(0, &word)) {
    /* never true ... */
}

/* the following is legal; you can mix atomic bit instructions with normal C */ 
word = 7;
```

Conveniently, nonatomic versions of all the bitwise functions are also provided. If you do not require atomicity (say, for example, because a lock already protects your data), these variants of the bitwise functions might be faster.

#### What the Heck Is a Nonatomic Bit Operation?
On first glance, the concept of a nonatomic bit operation might not make any sense. Only a single bit is involved; thus, there is no possibility of inconsistency. If one of the operations succeeds, what else could matter?

Let’s jump back to just what atomicity means. Atomicity requires that either instructions succeed in their entirety, uninterrupted, or instructions fail to execute at all. Therefore, if you issue two atomic bit operations, you expect two operations to succeed. After both operations complete, the bit needs to have the value as specified by the second operation. Moreover, however, at some point in time prior to the final operation, the bit needs to hold the value as specified by the first operation. Put more generally, real atomicity requires that all intermediate states be correctly realized.

For example, assume you issue two atomic bit operations: Initially set the bit and then clear the bit. Without atomic operations, the bit might end up cleared, but it might never have been set. The set operation could occur simultaneously with the clear operation and fail. With atomic operations, however, the set would actually occur—there would be a moment in time when a read would show the bit as set—and then the clear would execute and the bit would be zero.

This behavior can be important, especially when ordering comes into play or when dealing with hardware registers.

## Spin Locks
Simple atomic operations are clearly incapable of providing the needed protection in such a complex scenario, a more general method of synchronization is needed: locks.

The most common lock in the Linux kernel is the spin lock. A spin lock is a lock that can be held by at most one thread of execution. If a thread of execution attempts to acquire a spin lock while it is already held, which is called contended, the thread busy loops—spins—waiting for the lock to become available. If the lock is not contended, the thread can immediately acquire the lock and continue.

The fact that a contended spin lock causes threads to spin (essentially wasting processor time) while waiting for the lock to become available is salient.This behavior is the point of the spin lock. It is not wise to hold a spin lock for a long time. This is the nature of the spin lock: a lightweight single-holder lock that should be held for short durations. An alternative behavior when the lock is contended is to put the current thread to sleep and wake it up when it becomes available.Then the processor can go off and execute other code.This incurs a bit of overhead—most notably the two context switches required to switch out of and back into the blocking thread, which is certainly a lot more code than the handful of lines used to implement a spin lock. Therefore, it is wise to hold spin locks for less than the duration of two context switches. 

### Spin Lock Methods
Spin locks are architecture-dependent and implemented in assembly. The architecture-dependent code is defined in <asm/spinlock.h>. The actual usable interfaces are defined in <linux/spinlock.h>. The basic use of a spin lock is

```c
DEFINE_SPINLOCK(mr_lock);
spin_lock(&mr_lock);
/* critical region ... */ 
spin_unlock(&mr_lock);
```

On uniprocessor machines, the locks compile away and do not exist; they simply act as markers to disable and enable kernel preemption. If kernel preempt is turned off, the locks compile away entirely.（关于单cpu抢占模式下的 spin_lock 实现成禁止抢占的开关，因为在抢占模式下，进入零界区（不考虑该零界区会被中断访问）之前，如果不禁用抢占，在当前进程访问零界区的时候，可能发生抢占导致进程切换。这个时候，如果新的进程也访问零界区，那么就会造成冲突。但只要在前一个进程访问该临界区的时候，不发生抢占，该临界区就是安全的。所以，在单cpu抢占模式下，只要把 spin_lock 实现成禁止抢占的就行了）

Spin Locks Are Not Recursive! Unlike spin lock implementations in other operating systems and threading libraries, the Linux kernel’s spin locks are not recursive. This means that if you attempt to acquire a lock you already hold, you will spin, waiting for yourself to release the lock. But because you are busy spinning, you will never release the lock and you will deadlock. Be careful!

Spin locks can be used in interrupt handlers, whereas semaphores cannot be used because they sleep. If a lock is used in an interrupt handler, you must also disable local interrupts (interrupt requests on the current processor) before obtaining the lock. Otherwise, it is possible for an interrupt handler to interrupt kernel code while the lock is held and attempt to reacquire the lock. The interrupt handler spins, waiting for the lock to become available.The lock holder, however, does not run until the interrupt handler completes. This is an example of the double-acquire deadlock discussed in the previous chapter. Note that you need to disable interrupts only on the current processor. If an interrupt occurs on a different processor, and it spins on the same lock, it does not prevent the lock holder (which is on a different processor) from eventually releasing the lock.

The kernel provides an interface that conveniently disables interrupts and acquires the lock. Usage is

```c
DEFINE_SPINLOCK(mr_lock); 
unsigned long flags;

spin_lock_irqsave(&mr_lock, flags);
/* critical region ... */ 
spin_unlock_irqrestore(&mr_lock, flags);
```

The routine spin_lock_irqsave()saves the current state of interrupts, disables them locally, and then obtains the given lock. Conversely, spin_unlock_irqrestore() unlocks the given lock and returns interrupts to their previous state. Note that the flags variable is seemingly passed by value. This is because the lock routines are implemented partially as macros.

On uniprocessor systems, the previous example must still disable interrupts to prevent an interrupt handler from accessing the shared data, but the lock mechanism is compiled away. The lock and unlock also disable and enable kernel preemption, respectively.

If you always know before the fact that interrupts are initially enabled, there is no need to restore their previous state. You can unconditionally enable them on unlock. In those cases, spin_lock_irq() and spin_unlock_irq() are optimal:

```c
DEFINE_SPINLOCK(mr_lock);

spin_lock_irq(&mr_lock); 
/* critical section ... */ 
spin_unlock_irq(&mr_lock);
```

The configure option CONFIG_DEBUG_SPINLOCK enables a handful of debugging checks in the spin lock code. For example, with this option the spin lock code checks for the use of uninitialized spin locks and unlocking a lock that is not yet locked.

### Other Spin Lock Methods
You can use the method spin_lock_init() to initialize a dynamically created spin lock. The method spin_trylock() attempts to obtain the given spin lock. If the lock is contended, rather than spin and wait for the lock to be released, the function immediately returns zero. If it succeeds in obtaining the lock, it returns nonzero. Similarly, spin_is_locked() returns nonzero if the given lock is currently acquired. Otherwise, it returns zero. In neither case does spin_is_locked() actually obtain the lock.

### Spin Locks and Bottom Halves
The function spin_lock_bh() obtains the given lock and disables all bottom halves. The function spin_unlock_bh() performs the inverse.

Because a bottom half might preempt process context code, if data is shared between a bottom-half process context, you must protect the data in process context with both a lock and the disabling of bottom halves. Likewise, because an interrupt handler might preempt a bottom half, if data is shared between an interrupt handler and a bottom half, you must both obtain the appropriate lock and disable interrupts.

Recall that two tasklets of the same type do not ever run simultaneously.Thus, there is no need to protect data used only within a single type of tasklet. If the data is shared be- tween two different tasklets, however, you must obtain a normal spin lock before access- ing the data in the bottom half.

With softirqs, regardless of whether it is the same softirq type, if data is shared by softirqs, it must be protected with a lock. Recall that softirqs, even two of the same type, might run simultaneously on multiple processors in the system.A softirq never preempts another softirq running on the same processor, however, so disabling bottom halves is not needed.

## Reader-Writer Spin Locks
Reader-writer spin locks provide separate reader and writer variants of the lock. One or more readers can concurrently hold the reader lock.The writer lock, conversely, can be held by at most one writer with no concurrent readers. Reader/writer locks are sometimes called shared/exclusive or concurrent/exclusive locks because the lock is available in a shared (for readers) and an exclusive (for writers) form.

Usage is similar to spin locks.The reader-writer spin lock is initialized via

```c
DEFINE_RWLOCK(mr_rwlock);
```

Then, in the reader code path:

```c
read_lock(&mr_rwlock);
/* critical section (read only) ... */ 
read_unlock(&mr_rwlock);
```

Finally, in the writer code path:

```c
write_lock(&mr_rwlock);
/* critical section (read and write) ... */ 
write_unlock(&mr_lock);
```

Note that you cannot “upgrade” a read lock to a write lock. For example, consider this code snippet:

```c
read_lock(&mr_rwlock); 
write_lock(&mr_rwlock);
```

Executing these two functions as shown will deadlock, as the write lock spins, waiting for all readers to release the shared lock—including yourself.

It is safe for multiple readers to obtain the same lock. In fact, it is safe for the same thread to recursively obtain the same read lock.This lends itself to a useful and common optimization. If you have only readers in interrupt handlers but no writers, you can mix the use of the “interrupt disabling” locks.You can use read_lock() instead of read_lock_irqsave() for reader protection.

A final important consideration in using the Linux reader-writer spin locks is that they favor readers over writers. If the read lock is held and a writer is waiting for exclusive access, readers that attempt to acquire the lock continue to succeed.The spinning writer does not acquire the lock until all readers release the lock.

## Semaphores
Semaphores in Linux are sleeping locks.When a task attempts to acquire a semaphore that is unavailable, the semaphore places the task onto a wait queue and puts the task to sleep.The processor is then free to execute other code.When the semaphore becomes available, one of the tasks on the wait queue is awakened so that it can then acquire the semaphore.

You can draw some interesting conclusions from the sleeping behavior of semaphores:

* Because the contending tasks sleep while waiting for the lock to become available, semaphores are well suited to locks that are held for a long time.
* Conversely, semaphores are not optimal for locks that are held for short periods because the overhead of sleeping, maintaining the wait queue, and waking back up can easily outweigh the total lock hold time.
* Because a thread of execution sleeps on lock contention, semaphores must be obtained only in process context because interrupt context is not schedulable.
* You can (although you might not want to) sleep while holding a semaphore because you will not deadlock when another process acquires the same semaphore. (It will just go to sleep and eventually let you continue.)
* You cannot hold a spin lock while you acquire a semaphore, because you might have to sleep while waiting for the semaphore, and you cannot sleep while holding a spin lock.

Unlike spin locks, semaphores do not disable kernel preemption and, consequently, code holding a semaphore can be preempted. This means semaphores do not adversely affect scheduling latency.

### Counting and Binary Semaphores
Whereas spin locks permit at most one task to hold the lock at a time, the number of permissible simultaneous holders of semaphores can be set at declaration time. This value is called the usage count or simply the count. The most common value is to allow, like spin locks, only one lock holder at a time. In this case, the count is equal to one, and the semaphore is called either a binary semaphore or a mutex. Alternatively, the count can be initialized to a nonzero value greater than one. In this case, the semaphore is called a counting semaphore, and it enables at most count holders of the lock at a time.

A semaphore supports two atomic operations, P() and V(), named after the Dutch word Proberen, to test (literally, to probe), and the Dutch word Verhogen, to increment. Later systems called these methods down() and up(), respectively, and so does Linux. The down() method is used to acquire a semaphore by decrementing the count by one. If the new count is zero or greater, the lock is acquired and the task can enter the critical region. If the count is negative, the task is placed on a wait queue, and the processor moves on to something else. The up() method is used to release a semaphore upon completion of a critical region. This is called upping the semaphore. The method increments the count value; if the semaphore’s wait queue is not empty, one of the waiting tasks is awakened and allowed to acquire the semaphore.

### Creating and Initializing Semaphores
The semaphore implementation is architecture-dependent and defined in <asm/semaphore.h>. The struct semaphore type represents semaphores. Statically declared semaphores are created via the following, where name is the variable’s name and count is the usage count of the semaphore:

```c
struct semaphore name; 
sema_init(&name, count);
```

As a shortcut to create the more common mutex, use the following, where, again, name is the variable name of the binary semaphore:

```c
static DECLARE_MUTEX(name);
```

More frequently, semaphores are created dynamically, often as part of a larger structure. In this case, to initialize a dynamically created semaphore to which you have only an indirect pointer reference, just call sema_init(), where sem is a pointer and count is the usage count of the semaphore:

```c
sema_init(sem, count);
```

Similarly, to initialize a dynamically created mutex, you can use

```c
init_MUTEX(sem);
```

### Using Semaphores
The function down_interruptible() attempts to acquire the given semaphore. If the semaphore is unavailable, it places the calling process to sleep in the TASK_INTERRUPTIBLE state. If the task receives a signal while waiting for the semaphore, it is awakened and down_interruptible() returns -EINTR. Alternatively, the function down() places the task in the TASK_UNINTERRUPTIBLE state when it sleeps. You most likely do not want this because the process waiting for the semaphore does not respond to signals. Therefore, use of down_interruptible() is much more common (and correct) than down().

You can use down_trylock() to try to acquire the given semaphore without blocking. If the semaphore is already held, the function immediately returns nonzero. Otherwise, it returns zero and you successfully hold the lock.

To release a given semaphore, call up(). Consider an example:

```c
/* define and declare a semaphore, named mr_sem, with a count of one */
static DECLARE_MUTEX(mr_sem);

/* attempt to acquire the semaphore ... */ 
if (down_interruptible(&mr_sem)) {
    /* signal received, semaphore not acquired ... */
}

/* critical region ... */

/* release the given semaphore */ 
up(&mr_sem);
```

## Reader-Writer Semaphores
Reader-writer semaphores are represented by the struct rw_semaphore type, which is declared in <linux/rwsem.h>. Statically declared reader-writer semaphores are created via the following, where name is the declared name of the new semaphore:

```c
static DECLARE_RWSEM(name);
```

Reader-writer semaphores created dynamically are initialized via

```c
init_rwsem(struct rw_semaphore *sem)
```

All reader-writer semaphores are mutexes—that is, their usage count is one—although they enforce mutual exclusion only for writers, not readers. All reader-writer locks use uninterruptible sleep, so there is only one version of each down(). For example:

```c
static DECLARE_RWSEM(mr_rwsem);
/* attempt to acquire the semaphore for reading ... */
down_read(&mr_rwsem);

/* critical region (read only) ... */

/* release the semaphore */ 
up_read(&mr_rwsem);

/* ... */

/* attempt to acquire the semaphore for writing ... */ 
down_write(&mr_rwsem);

/* critical region (read and write) ... */

/* release the semaphore */ 
up_write(&mr_sem);
```

As with semaphores, implementations of down_read_trylock() and down_write_trylock() are provided. Each has one parameter: a pointer to a reader-writer semaphore.

Reader-writer semaphores have a unique method that their reader-writer spin lock cousins do not have: downgrade_write(). This function atomically converts an acquired write lock to a read lock.

## Mutexes
Seeking a simpler sleeping lock, the kernel developers introduced the mutex. The term “mutex” is a generic name to refer to any sleeping lock that enforces mutual exclusion, such as a semaphore with a usage count of one. In recent Linux kernels, the proper noun “mutex” is now also a specific type of sleeping lock that implements mutual exclusion.That is, a mutex is a mutex.

The mutex is represented by struct mutex. It behaves similar to a semaphore with a count of one, but it has a simpler interface, more efficient performance, and additional constraints on its use. To statically define a mutex, you do:

```c
DEFINE_MUTEX(name);
```

To dynamically initialize a mutex, you call

```c
mutex_init(&mutex);
```

Locking and unlocking the mutex is easy:

```c
mutex_lock(&mutex);
/* critical region ... */ 
mutex_unlock(&mutex);
```

The mutex has a stricter, narrower use case:

* Only one task can hold the mutex at a time.That is, the usage count on a mutex is always one.
* Whoever locked a mutex must unlock it.That is, you cannot lock a mutex in one context and then unlock it in another.This means that the mutex isn’t suitable for more complicated synchronizations between kernel and user-space. Most use cases, however, cleanly lock and unlock from the same context.
* Recursive locks and unlocks are not allowed.That is, you cannot recursively acquire the same mutex, and you cannot unlock an unlocked mutex.
* A process cannot exit while holding a mutex.
* A mutex cannot be acquired by an interrupt handler or bottom half, even with mutex_trylock().
* A mutex can be managed only via the official API: It must be initialized via the methods described in this section and cannot be copied, hand initialized, or reinitialized.

Perhaps the most useful aspect of the new struct mutex is that, via a special debugging mode, the kernel can programmatically check for and warn about violations of these constraints. When the kernel configuration option CONFIG_DEBUG_MUTEXES is enabled, a multitude of debugging checks ensure that these (and other) constraints are always upheld.

### Semaphores Versus Mutexes
Unless one of mutex’s additional constraints prevent you from using them, prefer the new mutex type to semaphores. When writing new code, only specific, often low-level, uses need a semaphore.

### Spin Locks Versus Mutexes
Only a spin lock can be used in interrupt context, whereas only a mutex can be held while a task sleeps.

|               Requirement               |            Recommended Lock                |
|-----------------------------------------|--------------------------------------------|
|           Low overhead locking          |         Spin lock is preferred.            |
|           Short lock hold time          |         Spin lock is preferred.            |
|           Long lock hold time           |            Mutex is preferred.             |
|   Need to lock from interrupt context   |         Spin lock is required.             |
|   Need to sleep while holding lock      |            Mutex is required.              |

## Completion Variables
Using completion variables is an easy way to synchronize between two tasks in the kernel when one task needs to signal to the other that an event has occurred. In fact, completion variables merely provide a simple solution to a problem whose answer is otherwise semaphores. For example, the vfork() system call uses completion variables to wake up the parent process when the child process execs or exits.

Completion variables are represented by the struct completion type, which is defined in <linux/completion.h>. A statically created completion variable is created and initialized via

```c
DECLARE_COMPLETION(mr_comp);
```

A dynamically created completion variable is initialized via init_completion().

On a given completion variable, the tasks that want to wait call wait_for_completion(). After the event has occurred, calling complete() signals all waiting tasks to wake up.

A common usage is to have a completion variable dynamically created as a member of a data structure. Kernel code waiting for the initialization of the data structure calls wait_for_completion(). When the initialization is complete, the waiting tasks are awakened via a call to completion().

## BKL: The Big Kernel Lock
The Big Kernel Lock (BKL) is a global spin lock that was created to ease the transition from Linux’s original SMP implementation to fine-grained locking. The BKL has some interesting properties:

* You can sleep while holding the BKL. The lock is automatically dropped when the task is unscheduled and reacquired when the task is rescheduled. Of course, this does not mean it is always safe to sleep while holding the BKL, merely that you can and you will not deadlock.
* The BKL is a recursive lock. A single process can acquire the lock multiple times and not deadlock, as it would with a spin lock.
* You can use the BKL only in process context. Unlike spin locks, you cannot acquire the BKL in interrupt context.
* New users of the BKL are forbidden. With every kernel release, fewer and fewer drivers and subsystems rely on the BKL.

The function lock_kernel() acquires the lock and the function unlock_kernel() releases the lock. A single thread of execution might acquire the lock recursively but must then call unlock_kernel() an equal number of times to release the lock. The function kernel_locked() returns nonzero if the lock is currently held; otherwise, it returns zero. These interfaces are declared in <linux/smp_lock.h>.

The BKL also disables kernel preemption while it is held. On UP(uniprocessor) kernels, the BKL code does not actually perform any physical locking.

## Sequential Locks

The sequential lock, generally shortened to seq lock, is a newer type of lock introduced in the 2.6 kernel. It provides a simple mechanism for reading and writing shared data. It works by maintaining a sequence counter.

Whenever the data in question is written to, a lock is obtained and a sequence number is incremented. Prior to and after reading the data, the sequence number is read. If the values are the same, a write did not begin in the middle of the read. Further, if the values are even, a write is not underway. (Grabbing the write lock makes the value odd, whereas releasing it makes it even because the lock starts at zero.)

To define a seq lock:

```c
seqlock_t mr_seq_lock = DEFINE_SEQLOCK(mr_seq_lock);
```

The write path is then:

```c
write_seqlock(&mr_seq_lock); 
/* write lock is obtained... */ 
write_sequnlock(&mr_seq_lock);
```

This looks like normal spin lock code.The oddness comes in with the read path, which is quite a bit different:

```c
unsigned long seq; 
do {
    seq = read_seqbegin(&mr_seq_lock);
    /* read data here ... */
} while (read_seqretry(&mr_seq_lock, seq));
```

Seq locks, however, favor writers over readers. An acquisition of the write lock always succeeds as long as there are no other writers. Furthermore, pending writers continually cause the read loop to repeat, until there are no longer any writers holding the lock.

Seq locks are ideal when your locking needs meet most or all these requirements:

* Your data has a lot of readers.
* Your data has few writers.
* Although few in number, you want to favor writers over readers and never allow readers to starve writers.
* Your data is simple, such as a simple structure or even a single integer that, for whatever reason, cannot be made atomic.

A prominent user of the seq lock is jiffies, the variable that stores a Linux machine’s uptime.  Jiffies holds a 64-bit count of the number of clock ticks since the machine booted. On machines that cannot atomi- cally read the full 64-bit jiffies_64 variable, get_jiffies_64() is implemented using seq locks:

```c
u64 get_jiffies_64(void) {
    unsigned long seq; 
    u64 ret;

    do {
        seq = read_seqbegin(&xtime_lock);
        ret = jiffies_64;
    } while (read_seqretry(&xtime_lock, seq));  // 读操作期间是否发生写

    return ret;
}
```

Updating jiffies during the timer interrupt, in turns, grabs the write variant of the seq lock:

```c
write_seqlock(&xtime_lock); 
jiffies_64 += 1; 
write_sequnlock(&xtime_lock);
```

## Preemption Disabling
This means a task can begin running in the same critical region as a task that was preempted. To prevent this, the kernel preemption code uses spin locks as markers of nonpreemptive regions. If a spin lock is held, the kernel is not preemptive.

In reality, some situations do not require a spin lock, but do need kernel preemption disabled. The most frequent of these situations is per-processor data. If no spin locks are held, the kernel is preemptive, and it would be possible for a newly scheduled task to access this same variable. To solve this, kernel preemption can be disabled via preempt_disable(). The call is nestable; you can call it any number of times. For each call, a corresponding call to preempt_enable() is required.

The preemption count stores the number of held locks and preempt_disable() calls. If the number is zero, the kernel is preemptive. If the value is one or greater, the kernel is not preemptive.This count is incredibly useful—it is a great way to do atomicity and sleep debugging. The function preempt_count() returns this value.

As a cleaner solution to per-processor data issues, you can obtain the processor number (which presumably is used to index into the per-processor data) via get_cpu(). This function disables kernel preemption prior to returning the current processor number:

```c
int cpu;
/* disable kernel preemption and set “cpu” to the current processor */ 
cpu = get_cpu();
/* manipulate per-processor data ... */

/* reenable kernel preemption, “cpu” can change and so is no longer valid */ 
put_cpu();
```

## Ordering and Barriers
When dealing with synchronization between multiple processors or with hardware devices, it is sometimes a requirement that memory-reads (loads) and memory-writes (stores) issue in the order specified in your program code. When talking with hardware, you often need to ensure that a given read occurs before another read or write. Additionally, on symmetrical multiprocessing systems, it might be important for writes to appear in the order that your code issues them (usually to ensure subsequent reads see the data in the same order). Complicating these issues is the fact that both the compiler and the processor can reorder reads and writes4 for performance reasons. Thankfully, all processors that do reorder reads or writes provide machine instructions to enforce ordering requirements. It is also possible to instruct the compiler not to reorder instructions around a given point. These instructions are called barriers.

Essentially, on some processors the following code may allow the processor to store the new value in b before it stores the new value in a:

```c
a = 1; 
b = 2;
```

Both the compiler and processor see no relation between a and b. The compiler would perform this reordering at compile time; the reordering would be static, and the resulting object code would simply set b before a. The processor, however, could perform the reordering dynamically during execution by fetching and dispatching seemingly unrelated instructions in whatever order it feels is best.

Although the previous example might be reordered, the processor would never reorder writes such as the following because there is clearly a data dependency between a and b:

```c
a = 1; 
b = a;
```

Occasionally, it is important that writes are seen by other code and the outside world in the specific order you intend.

The rmb() method provides a read memory barrier. It ensures that no loads are reordered across the rmb() call. That is, no loads prior to the call will be reordered to after the call, and no loads after the call will be reordered to before the call. The wmb() method provides a write barrier. It functions in the same manner as rmb(), but with respect to stores instead of loads—it ensures no stores are reordered across the barrier.

The mb() call provides both a read barrier and a write barrier. No loads or stores will be reordered across a call to mb(). It is provided because a single instruction (often the same instruction used by rmb()) can provide both the load and store barrier.

A variant of rmb(), read_barrier_depends(), provides a read barrier but only for loads on which subsequent loads depend. All reads prior to the barrier are guaranteed to complete before any reads after the barrier that depend on the reads prior to the barrier. Basically, it enforces a read barrier, similar to rmb(), but only for certain reads—those that depend on each other.

The macros smp_rmb(), smp_wmb(), smp_mb(), and smp_read_barrier_depends() provide a useful optimization. On SMP kernels they are defined as the usual memory barriers, whereas on UP kernels they are defined only as a compiler barrier.

The barrier() method prevents the compiler from optimizing loads or stores across the call.

# Chapter11. Timers and Time Management
A large number of kernel functions are time-driven, as opposed to event-driven.

The implementation differs between how events that occur periodically and events the kernel schedules for a fixed point in the future are handled. Events that occur periodically—say, every 10 milliseconds—are driven by the system timer. The system timer is a programmable piece of hardware that issues an interrupt at a fixed frequency. The interrupt handler for this timer—called the timer interrupt—updates the system time and performs periodic work.

Dynamic timers is the facility used to schedule events that run once after a specified time has elapsed. 

## Kernel Notion of Time
The kernel must work with the system’s hardware to comprehend and manage time. The hardware provides a system timer that the kernel uses to gauge the passing of time. This system timer works off of an electronic time source, such as a digital clock or the frequency of the processor. The system timer goes off (often called hitting or popping) at a preprogrammed frequency, called the tick rate. When the system timer goes off, it issues an interrupt that the kernel handles via a special interrupt handler.

Because the kernel knows the preprogrammed tick rate, it knows the time between any two successive timer interrupts. This period is called a tick and is equal to 1/(tick rate) seconds. This is how the kernel keeps track of both wall time and system uptime. Wall time—the actual time of day—is important to user-space applications. The kernel keeps track of it simply because the kernel controls the timer interrupt. A family of system calls provides the date and time of day to user-space. The system uptime—the relative time since the system booted—is useful to both kernel-space and user-space.

The timer interrupt is important to the management of the operating system. A large number of kernel functions live and die by the passing of time. Some of the work executed periodically by the timer interrupt includes:

* Updating the system uptime
* Updating the time of day
* On an SMP system, ensuring that the scheduler runqueues are balanced and, if not, balancing them (as discussed in Chapter 4,“Process Scheduling”)
* Running any dynamic timers that have expired
* Updating resource usage and processor time statistics


## The Tick Rate: HZ
The frequency of the system timer (the tick rate) is programmed on system boot based on a static preprocessor define, HZ. The kernel defines the value in <asm/param.h>. The tick rate has a frequency of HZ hertz and a period of 1/HZ seconds.

### The Ideal HZ Value

Increasing the tick rate means the timer interrupt runs more frequently. Consequently, the work it performs occurs more often. This has the following benefits:

* The timer interrupt has a higher resolution and, consequently, all timed events have a higher resolution.
* The accuracy of timed events improves.

### Advantages with a Larger HZ
This higher resolution and greater accuracy provides multiple advantages:

* Kernel timers execute with finer resolution and increased accuracy.
* System calls such as poll()and select() that optionally employ a timeout value execute with improved precision.
* Measurements, such as resource usage or the system uptime, are recorded with a finer resolution.
* Process preemption occurs more accurately.

### Disadvantages with a Larger HZ
A higher tick rate implies more frequent timer interrupts, which implies higher overhead, because the processor must spend more time executing the timer interrupt handler. This adds up to not just less processor time available for other work, but also a more frequent thrashing of the processor’s cache and increase in power consumption.

The Linux kernel supports an option known as a tickless operation. When a kernel is built with the CONFIG_HZ configuration option set, the system dynamically schedules the timer interrupt in accordance with pending timers.

## Jiffies
The global variable jiffies holds the number of ticks that have occurred since the system booted. On boot, the kernel initializes the variable to zero, and it is incremented by one during each timer interrupt.Thus, because there are HZ timer interrupts in a second, there are HZ jiffies in a second. The system uptime is therefore jiffies/HZ seconds.

The jiffies variable is declared in <linux/jiffies.h> as 

```c
extern unsigned long volatile jiffies;
```

### Internal Representation of Jiffies
The jiffies variable has always been an unsigned long, and therefore 32 bits in size on 32-bit architectures and 64-bits on 64-bit architectures. With a tick rate of 100, a 32-bit jiffies variable would overflow in about 497 days. With HZ increased to 1000, however, that overflow now occurs in just 49.7 days!

For performance and historical reasons—mainly compatibility with existing kernel code—the kernel developers wanted to keep jiffies an unsigned long. Some smart thinking and a little linker magic saved that day.

As you previously saw, jiffies is defined as an unsigned long: 

```c
extern unsigned long volatile jiffies;
```

A second variable is also defined in <linux/jiffies.h>: 

```c
extern u64 jiffies_64;
```

The ld(1) script used to link the main kernel image (arch/x86/kernel/vmlinux. lds.S on x86) then overlays the jiffies variable over the start of the jiffies_64 variable:

```c
jiffies = jiffies_64;
```

Thus, jiffies is the lower 32 bits of the full 64-bit jiffies_64 variable. Code can continue to access the jiffies variable exactly as before. Because most code uses jiffies simply to measure elapses in time, most code cares about only the lower 32 bits. The time management code uses the entire 64 bits, however, and thus prevents overflow of the full 64-bit value. 

Code that accesses jiffies simply reads the lower 32 bits of jiffies_64. The function get_jiffies_64() can be used to read the full 64-bit value.4 Such a need is rare; consequently, most code simply continues to read the lower 32 bits directly via the jiffies variable.
On 64-bit architectures, jiffies_64 and jiffies refer to the same thing. Code can either read jiffies or call get_jiffies_64() as both actions have the same effect.

### Jiffies Wraparound
The jiffies variable, like any C integer, experiences overflow when its value is increased beyond its maximum storage limit.

The kernel provides four macros for comparing tick counts that correctly handle wraparound in the tick count. They are in <linux/jiffies.h>. Listed here are simplified versions of the macros:

```c
#define time_after(unknown, known) ((long)(known) - (long)(unknown) < 0) 
#define time_before(unknown, known) ((long)(unknown) - (long)(known) < 0) 
#define time_after_eq(unknown, known) ((long)(unknown) - (long)(known) >= 0) 
#define time_before_eq(unknown, known) ((long)(known) - (long)(unknown) >= 0)
```

The unknown parameter is typically jiffies and the known parameter is the value against which you want to compare.

### User-Space and HZ
In kernels earlier than 2.6, changing the value of HZ resulted in user-space anomalies. This happened because values were exported to user-space in units of ticks-per-second.

To prevent such problems, the kernel needs to scale all exported jiffies values. It does this by defining USER_HZ, which is the HZ value that user-space expects. On x86, because HZ was historically 100, USER_HZ is 100.The function jiffies_to_clock_t(), defined in kernel/time.c, is then used to scale a tick count in terms of HZ to a tick count in terms of USER_HZ.

The function jiffies_64_to_clock_t() is provided to convert a 64-bit jiffies value from HZ to USER_HZ units.

## Hardware Clocks and Timers
Architectures provide two hardware devices to help with time keeping: the system timer, which we have been discussing, and the real-time clock.

## Real-Time Clock
The real-time clock (RTC) provides a nonvolatile device for storing the system time. The RTC continues to keep track of time even when the system is off by way of a small battery typically included on the system board. On boot, the kernel reads the RTC and uses it to initialize the wall time, which is stored in the xtime variable.

### System Timer
The system timer serves a much more important (and frequent) role in the kernel’s time- keeping.The idea behind the system timer, regardless of architecture, is the same—to provide a mechanism for driving an interrupt at a periodic rate.

On x86, the primary system timer is the programmable interrupt timer (PIT). The kernel programs the PIT on boot to drive the system timer interrupt (interrupt zero) at HZ frequency.

## The Timer Interrupt Handler
The timer interrupt is broken into two pieces: an architecture-dependent and an architecture-independent routine.


The architecture-dependent routine is registered as the interrupt handler for the system timer and, thus, runs when the timer interrupt hits. Most handlers perform at least the following work:

* Obtain the xtime_lock lock, which protects access to jiffies_64 and the wall time value, xtime.
* Acknowledge or reset the system timer as required.
* Periodically save the updated wall time to the real time clock.
* Call the architecture-independent timer routine,tick_periodic().

The architecture-independent routine, tick_periodic(), performs much more work: 

* Increment the jiffies_64 count by one. (This is safe, even on 32-bit architectures, because the xtime_lock lock was previously obtained.)
* Update resource usages, such as consumed system and user time, for the currently running process.
* Run any dynamic timers that have expired (discussed in the following section). 
* Execute scheduler_tick(), as discussed in Chapter 4.
* Update the wall time, which is stored in xtime.
* Calculate the infamous load average.


The routine is simple because other functions handle most of the work:

```c
static void tick_periodic(int cpu) {
    if (tick_do_timer_cpu == cpu) { 
        write_seqlock(&xtime_lock);

        /* Keep track of the next tick event */
        tick_next_period = ktime_add(tick_next_period, tick_period);

        do_timer(1); 
        write_sequnlock(&xtime_lock);
    }
    update_process_times(user_mode(get_irq_regs())); 
    profile_tick(CPU_PROFILING);
}
```

Most of the important work is enabled in do_timer() and update_process_times(). The former is responsible for actually performing the increment to jiffies_64:

```c
void do_timer(unsigned long ticks) {
    jiffies_64 += ticks; 
    update_wall_time(); 
    calc_global_load();
}
```

The function update_wall_time(), as its name suggests, updates the wall time in accordance with the elapsed ticks, whereas calc_global_load() updates the system’s load average statistics.

When do_timer() ultimately returns, update_process_times() is invoked to update various statistics that a tick has elapsed, noting via user_tick whether it occurred in user-space or kernel-space:

```c
void update_process_times(int user_tick) {
    struct task_struct *p = current; 
    int cpu = smp_processor_id();

    /* Note: this timer irq context must be accounted for as well. */ 
    account_process_tick(p, user_tick);
    run_local_timers();
    rcu_check_callbacks(cpu, user_tick);
    printk_tick(); 
    scheduler_tick(); 
    run_posix_cpu_timers(p);
}
```

The account_process_tick() function does the actual updating of the process’s times: 

```c
void account_process_tick(struct task_struct *p, int user_tick) {
    cputime_t one_jiffy_scaled = cputime_to_scaled(cputime_one_jiffy); 
    struct rq *rq = this_rq();  

    if (user_tick)
        account_user_time(p, cputime_one_jiffy, one_jiffy_scaled);
    else if ((p != rq->idle) || (irq_count() != HARDIRQ_OFFSET)) 
        account_system_time(p, HARDIRQ_OFFSET, cputime_one_jiffy,
                            one_jiffy_scaled); 
    else
        account_idle_time(cputime_one_jiffy);
}
```


You might realize that this approach implies that the kernel credits a process for running the entire previous tick in whatever mode the processor was in when the timer interrupt occurred.

Next, the run_local_timers() function marks a softirq to handle the execution of any expired timers. 

Finally, the scheduler_tick() function decrements the currently running process’s timeslice and sets need_resched if needed. On SMP machines, it also balances the per-processor runqueues as needed.

The tick_periodic() function returns to the original architecture-dependent interrupt handler, which performs any needed cleanup, releases the xtime_lock lock, and finally returns.

## The Time of Day
The current time of day (the wall time) is defined in kernel/time/timekeeping.c: 

```c
struct timespec xtime;
```
The timespec data structure is defined in <linux/time.h> as:

```c
struct timespec {
    __kernel_time_t tv_sec; /* seconds */ 
    long tv_nsec; /* nanoseconds */
};
```

The xtime.tv_sec value stores the number of seconds that have elapsed since January 1, 1970 (UTC). This date is called the epoch. The xtime.v_nsec value stores the number of nanoseconds that have elapsed in the last second.

Reading or writing the xtime variable requires the xtime_lock lock, which is not a normal spinlock but a seqlock.

The primary user-space interface for retrieving the wall time is gettimeofday(), which is implemented as sys_gettimeofday() in kernel/time.c:

```c
asmlinkage long sys_gettimeofday(struct timeval *tv, struct timezone *tz) {
    if (likely(tv)) {
        struct timeval ktv;
        do_gettimeofday(&ktv);
        if (copy_to_user(tv, &ktv, sizeof(ktv)))
            return -EFAULT;
    }
    
    if (unlikely(tz)) {
        if (copy_to_user(tz, &sys_tz, sizeof(sys_tz))) 
            return -EFAULT;
    }
    return 0;
}
```

The settimeofday() system call sets the wall time to the specified value. It requires the CAP_SYS_TIME capability.

## Timers
Timers—sometimes called dynamic timers or kernel timers—are essential for managing the flow of time in kernel code. Kernel code often needs to delay execution of some function until a later time. 

### Using Timers
Timers are represented by struct timer_list, which is defined in <linux/timer.h>:

```c
struct timer_list {
    struct list_head entry;  /* entry in linked list of timers */
    unsigned long expires;  /* expiration value, in jiffies */
    void (*function)(unsigned long);  /* the timer handler function */
    unsigned long data;  /* lone argument to the handler */
    struct tvec_t_base_s *base;  /* internal timer field, do not touch */
};
```

The kernel provides a family of timer-related interfaces to make timer management easy. Everything is declared in <linux/timer.h>. Most of the actual implementation is in kernel/timer.c.

The first step in creating a timer is defining it:

```c
struct timer_list my_timer;
```

Next, the timer’s internal values must be initialized.This is done via a helper function and must be done prior to calling any timer management functions on the timer:

```c
init_timer(&my_timer);
```

Now you fill out the remaining values as required:

```c
my_timer.expires = jiffies + delay;  /* timer expires in delay ticks */
my_timer.data = 0;  /* zero is passed to the timer handler */
my_timer.function = my_function;  /* function to run when timer expires */
```

The my_timer.expires value specifies the timeout value in absolute ticks. When the current jiffies count is equal to or greater than my_timer.expires, the handler function my_timer.function is run with the lone argument of my_timer.data. 

Finally, you activate the timer:

```c
add_timer(&my_timer);
```

Sometimes you might need to modify the expiration of an already active timer. The kernel implements a function, mod_timer(), which changes the expiration of a given timer:

```c
mod_timer(&my_timer, jiffies + new_delay); /* new expiration */
```

The mod_timer() function can operate on timers that are initialized but not active, too. If the timer is inactive, mod_timer() activates it. The function returns zero if the timer were inactive and one if the timer were active. 

If you need to deactivate a timer prior to its expiration, use the del_timer() function: 

```c
del_timer(&my_timer);
```

The function works on both active and inactive timers. If the timer is already inactive, the function returns zero; otherwise, the function returns one. Note that you do not need to call this for timers that have expired because they are automatically deactivated.

To deactivate the timer and wait until a potentially executing handler for the timer exits, use del_timer_sync():

```c
del_timer_sync(&my_timer);
```

Unlike del_timer(), del_timer_sync() cannot be used from interrupt context.

### Timer Race Conditions
Because timers run asynchronously with respect to the currently executing code, several potential race conditions exist. First, never do the following as a substitute for a mere mod_timer(), because this is unsafe on multiprocessing machines:

```
del_timer(my_timer)
my_timer->expires = jiffies + new_delay; 
add_timer(my_timer);
```

Second, in almost all cases, you should use del_timer_sync() over del_timer(). Otherwise, you cannot assume the timer is not currently running, and that is why you made the call in the first place! Imagine if, after deleting the timer, the code went on to free or otherwise manipulate resources used by the timer handler.Therefore, the synchronous version is preferred.

Finally, you must make sure to protect any shared data used in the timer handler function. The kernel runs the function asynchronously with respect to other code.

### Timer Implementation
The kernel executes timers in bottom-half context, as softirqs, after the timer interrupt completes. The timer interrupt handler runs update_process_times(), which calls run_local_timers():

```c
void run_local_timers(void) {
    hrtimer_run_queues(); 
    raise_softirq(TIMER_SOFTIRQ);  /* raise the timer softirq */
    softlockup_tick();
}
```

The TIMER_SOFTIRQ softirq is handled by run_timer_softirq(). This function runs all the expired timers (if any) on the current processor.

Timers are stored in a linked list. However, it would be unwieldy for the kernel to either constantly traverse the entire list looking for expired timers, or keep the list sorted by expiration value; the insertion and deletion of timers would then become expensive. Instead, the kernel partitions timers into five groups based on their expiration value. Timers move down through the groups as their expiration time draws closer.

## Delaying Execution
Often, kernel code (especially drivers) needs a way to delay execution for some time without using timers or a bottom-half mechanism. This is usually to enable hardware time to complete a given task. The time is typically quite short.

The kernel provides a number of solutions, depending on the semantics of the delay. The solutions have different characteristics. Some hog the processor while delaying— effectively preventing—the accomplishment of any real work. Other solutions do not hog the processor but offer no guarantee that your code will resume in exactly the required time.

### Busy Looping
The simplest solution to implement (although rarely the optimal solution) is busy waiting or busy looping. This technique works only when the time you want to delay is some integer multiple of the tick rate or precision is not important.

A better solution would be to reschedule your process to allow the processor to accomplish other work while your code waits:

```c
unsigned long delay = jiffies + 5*HZ;
while (time_before(jiffies, delay)) 
    cond_resched();
```

The call to cond_resched()schedules a new process, but only if need_resched is set. In other words, this solution conditionally invokes the scheduler only if there is some more important task to run. Note that because this approach invokes the scheduler, you cannot make use of it from an interrupt handler—only from process context. Further- more, delaying execution in any manner, if at all possible, should not occur while a lock is held or interrupts are disabled.

C aficionados might wonder what guarantee is given that the previous loops even work. The C compiler is usually free to perform a given load only once. Normally, no as- surance is given that the jiffies variable in the loop’s conditional statement is even reloaded on each loop iteration. The kernel requires, however, that jiffies be reread on each iteration, as the value is incremented elsewhere: in the timer interrupt. Indeed, this is why the variable is marked volatile in <linux/jiffies.h>. The volatile keyword instructs the compiler to reload the variable on each access from main memory and
never alias the variable’s value in a register, guaranteeing that the previous loop completes as expected.

### Small Delays
Sometimes, kernel code (again, usually drivers) requires short (smaller than a clock tick) and rather precise delays.This is often to synchronize with hardware, which again usually lists some minimum time for an activity to complete—often less than a millisecond.

The kernel provides three functions for microsecond, nanosecond, and millisecond delays, defined in <linux/delay.h> and <asm/delay.h>, which do not use jiffies:

```c
void udelay(unsigned long usecs) 
void ndelay(unsigned long nsecs) 
void mdelay(unsigned long msecs)
```

The udelay() function is implemented as a loop that knows how many iterations can be executed in a given period of time.The mdelay() function is then implemented in terms of udelay(). Because the kernel knows how many loops the processor can com- plete in a second (see the sidebar on BogoMIPS), the udelay() function simply scales that value to the correct number of loop iterations for the given delay.

The BogoMIPS value is the number of busy loop iterations the processor can perform in a given period. In effect, BogoMIPS are a measurement of how fast a processor can do nothing! This value is stored in the loops_per_jiffy variable and is readable from /proc/cpuinfo. The delay loop functions use the loops_per_jiffy value to figure out (fairly precisely) how many busy loop iterations they need to execute to provide the requisite delay.

The kernel computes loops_per_jiffy on boot via calibrate_delay() in init/main.c.

The udelay() function should be called only for small delays because larger delays on fast machines might result in overflow. As a rule, do not use udelay() for delays more than one millisecond in duration. For longer durations, mdelay() works fine. 

Remember that it is rude to busy loop with locks held or interrupts disabled because system response and performance will be adversely affected. If you require precise delays, however, these calls are your best bet. Typical uses of these busy waiting functions delay for a small amount of time, usually in the microsecond range.

### schedule_timeout()
A more optimal method of delaying execution is to use schedule_timeout().This call puts your task to sleep until at least the specified time has elapsed. There is no guarantee that the sleep duration will be exactly the specified time—only that the duration is at least as long as specified. When the specified time has elapsed, the kernel wakes the task up and places it back on the runqueue. Usage is easy:

```c
/* set task’s state to interruptible sleep */ 
set_current_state(TASK_INTERRUPTIBLE);

/* take a nap and wake up in “s” seconds */ 
schedule_timeout(s * HZ);
```

If the code does not want to process signals, you can use TASK_UNINTERRUPTIBLE instead. The task must be in one of these two states before schedule_timeout() is called or else the task will not go to sleep.

Note that because schedule_timeout() invokes the scheduler, code that calls it must be capable of sleeping. 

#### schedule_timeout() Implementation
The schedule_timeout() function is fairly straightforward. Indeed, it is a simple application of kernel timers, so let’s take a look at it:

```c
signed long schedule_timeout(signed long timeout) {
    timer_t timer; 
    unsigned long expire;
    switch (timeout)
    {
        case MAX_SCHEDULE_TIMEOUT:
            schedule();
            goto out; 
        default:
            if (timeout < 0) {
                printk(KERN_ERR "schedule_timeout: wrong timeout " 
                            "value %lx from %p\n", timeout, 
                            __builtin_return_address(0));
                current->state = TASK_RUNNING; 
                goto out;
            }
    }
    expire = timeout + jiffies; 
    init_timer(&timer);
    timer.expires = expire;
    timer.data = (unsigned long) current; 
    timer.function = process_timeout;

    add_timer(&timer); 
    schedule(); 
    del_timer_sync(&timer);

    timeout = expire - jiffies;

out:
    return timeout < 0 ? 0 : timeout;
}
```

The function creates a timer with the original name timer and sets it to expire in timeout clock ticks in the future. It sets the timer to execute the process_timeout() function when the timer expires. It then enables the timer and calls schedule(). Because the task is supposedly marked TASK_INTERRUPTIBLE or TASK_UNINTERRUPTIBLE, the scheduler does not run the task, but instead picks a new one.

When the timer expires, it runs process_timeout(): 

```c
void process_timeout(unsigned long data) {
    wake_up_process((task_t *) data);
}

```

This function puts the task in the TASK_RUNNING state and places it back on the runqueue.

When the task reschedules, it returns to where it left off in schedule_timeout() (right after the call to schedule()). In case the task was awakened prematurely (if a signal was received), the timer is destroyed.The function then returns the time slept.

The code in the switch() statement is for special cases and is not part of the general usage of the function. The MAX_SCHEDULE_TIMEOUT check enables a task to sleep indefinitely. In that case, no timer is set (because there is no bound on the sleep duration), and the scheduler is immediately invoked. If you do this, you must have another method of waking your task up!

#### Sleeping on a Wait Queue, with a Timeout
Sometimes it is desirable to wait for a specific event or wait for a specified time to elapse—whichever comes first. In those cases, code might simply call schedule_timeout() instead of schedule() after placing itself on a wait queue. The task wakes up when the desired event occurs or the specified time elapses.The code needs to check why it woke up—it might be because of the event occurring, the time elapsing, or a received signal—and continue as appropriate.

# Chapter12. Memory Management
## Pages
The kernel treats physical pages as the basic unit of memory management. Although the processor’s smallest addressable unit is a byte or a word, the memory management unit (MMU, the hardware that manages memory and performs virtual to physical address translations) typically deals in pages.

Each architecture defines its own page size. Most 32-bit architectures have 4KB pages, whereas most 64-bit architectures have 8KB pages.

The kernel represents every physical page on the system with a struct page structure. This structure is defined in <linux/mm_types.h>:

```c
struct page {
    unsigned long       flags;
    atomic_t            _count;
    atomic_t            _mapcount;
    unsigned long       private;
    struct address_space *mapping;
    pgoff_t             index;
    struct list_head    lru;
    void                *virtual;
};
```

The flags field stores the status of the page. Such flags include whether the page is dirty or whether it is locked in memory. Bit flags represent the various values, so at least 32 different flags are simultaneously available. The flag values are defined in <linux/page-flags.h>.
     
The _count field stores the usage count of the page—that is, how many references there are to this page. When this count reaches negative one, no one is using the page, and it becomes available for use in a new allocation. Kernel code should not check this field directly but instead use the function page_count(), which takes a page structure as its sole parameter. 

A page may be used by the page cache (in which case the mapping field points to the address_space object associated with this page), as private data (pointed at by private), or as a mapping in a process’s page table.

The virtual field is the page’s virtual address. Normally, this is simply the address of the page in virtual memory. Some memory (called high memory) is not permanently mapped in the kernel’s address space. In that case, this field is NULL, and the page must be dynamically mapped if needed.

The important point to understand is that the page structure is associated with physical pages, not virtual pages. The data structure’s goal is to describe physical memory, not the data contained therein.

## Zones
Because of hardware limitations, the kernel cannot treat all pages as identical. Some pages, because of their physical address in memory, cannot be used for certain tasks. Because of this limitation, the kernel divides pages into different zones. The kernel uses the zones to group pages of similar properties. 

* Some hardware devices can perform DMA (direct memory access) to only certain memory addresses.
* Some architectures can physically addressing larger amounts of memory than they can virtually address. Consequently, some memory is not permanently mapped into the kernel address space.

Because of these constraints, Linux has four primary memory zones:

* ZONE_DMA—This zone contains pages that can undergo DMA.
* ZONE_DMA32—Like ZOME_DMA, this zone contains pages that can undergo DMA. Unlike ZONE_DMA, these pages are accessible only by 32-bit devices. On some architectures, this zone is a larger subset of memory.
* ZONE_NORMAL—This zone contains normal, regularly mapped, pages.
* ZONE_HIGHMEM—This zone contains “high memory,” which are pages not permanently mapped into the kernel’s address space.

The memory contained in ZONE_HIGHMEM is called high memory. The rest of the system’s memory is called low memory.

Allocations cannot cross zone boundaries.

Each zone is represented by struct zone, which is defined in <linux/mmzone.h>:

```c
struct zone { 
    unsigned long           watermark[NR_WMARK];
    unsigned long           lowmem_reserve[MAX_NR_ZONES];
    struct per_cpu_pageset  pageset[NR_CPUS];
    spinlock_t              lock;

    struct free_area        free_area[MAX_ORDER]
    spinlock_t              lru_lock;
    struct zone_lru {
        struct list_head    list;
        unsigned long       nr_saved_scan;
    } lru[NR_LRU_LISTS];
    struct zone_reclaim_stat reclaim_stat;
    unsigned long            pages_scanned;
    unsigned long            flags;
    atomic_long_t            vm_stat[NR_VM_ZONE_STAT_ITEMS]; 
    int                      prev_priority;
    unsigned int             inactive_ratio;
    wait_queue_head_t        *wait_table;
    unsigned long            wait_table_hash_nr_entries;
    unsigned long            wait_table_bits;
    struct pglist_data       *zone_pgdat;
    unsigned long            zone_start_pfn;
    unsigned long            spanned_pages;
    unsigned long            present_pages;
    const char               *name;
};
```
  
The lock field is a spin lock that protects the structure from concurrent access. Note that it protects just the structure and not all the pages that reside in the zone.

The watermark array holds the minimum, low, and high watermarks for this zone. The kernel uses watermarks to set benchmarks for suitable per-zone memory consumption.

The name field is, unsurprisingly, a NULL-terminated string representing the name of this zone.The kernel initializes this value during boot in mm/page_alloc.c, and the three zones are given the names DMA, Normal, and HighMem.

## Getting Pages
The kernel provides one low-level mechanism for requesting memory, along with several interfaces to access it. All these interfaces allocate memory with page-sized granularity and are declared in <linux/gfp.h>. The core function is

```c
struct page * alloc_pages(gfp_t gfp_mask, unsigned int order)
```

This allocates 2order (that is,1 << order) contiguous physical pages and returns a pointer to the first page’s page structure; on error it returns NULL. You can convert a given page to its logical address with the function:

```c
void * page_address(struct page *page)
```

This returns a pointer to the logical address where the given physical page currently resides. If you have no need for the actual struct page, you can call

```c
unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order)
```

If you need only one page, two functions are implemented as wrappers to save you a bit of typing:

```c
struct page * alloc_page(gfp_t gfp_mask) 
unsigned long __get_free_page(gfp_t gfp_mask)
```

### Getting Zeroed Pages
If you need the returned page filled with zeros, use the function:

```c
unsigned long get_zeroed_page(unsigned int gfp_mask)
```

This function works the same as __get_free_page(), except that the allocated page is then zero-filled—every bit of every byte is unset.

### Freeing Pages
A family of functions enables you to free allocated pages when you no longer need them:

```c
void __free_pages(struct page *page, unsigned int order) 
void free_pages(unsigned long addr, unsigned int order) 
void free_page(unsigned long addr)
```

For more general byte-sized allocations, the kernel provides kmalloc().

## kmalloc()
The kmalloc() function’s operation is similar to that of user-space’s familiar malloc() routine, with the exception of the additional flags parameter. The kmalloc() function is a simple interface for obtaining kernel memory in byte-sized chunks.

The function is declared in <linux/slab.h>: 

```c
void * kmalloc(size_t size, gfp_t flags)
```

The function returns a pointer to a region of memory that is at least size bytes in length. The region of memory allocated is physically contiguous. On error, it returns NULL. 

### gfp_mask Flags
Flags are represented by the gfp_t type, which is defined in <linux/types.h> as an unsigned int. The flags are broken up into three categories: action modifiers, zone modifiers, and types. Action modifiers specify how the kernel is supposed to allocate the requested mem- ory. In certain situations, only certain methods can be employed to allocate memory. Zone modifiers specify from where to allocate memory. Type flags specify a combination of action and zone modifiers as needed by a certain type of memory allocation. Type flags simplify the specification of multiple modifiers; instead of providing a combination of action and zone modifiers, you can specify just one type flag.

#### Action Modifiers
All the flags, the action modifiers included, are declared in <linux/gfp.h>.

|        Flag       |                Description                  |
|-------------------|---------------------------------------------|
|    __GFP_WAIT     |           The allocator can sleep.          |
|    __GFP_HIGH     |  The allocator can access emergency pools.  |
|     __GFP_IO      |      The allocator can start disk I/O.      |
|     __GFP_FS      |   The allocator can start filesystem I/O.   |

These allocations can be specified together. For example:

```c
ptr = kmalloc(size, __GFP_WAIT | __GFP_IO | __GFP_FS);
```

This call instructs the page allocator (ultimately alloc_pages()) that the allocation can block, perform I/O, and perform filesystem operations, if needed. This enables the kernel great freedom in how it can find the free memory to satisfy the allocation.

#### Zone Modifiers

|       Flag      |              Description             |
|-----------------|--------------------------------------|
|    __GFP_DMA    |     Allocates only from ZONE_DMA     |
|   __GFP_DMA32   |     Allocates only from ZONE_DMA32   |
|  __GFP_HIGHMEM  |Allocates from ZONE_HIGHMEM or ZONE_NORMAL |

Specifying one of these three flags modifies the zone from which the kernel attempts to satisfy the allocation.

The __GFP_DMA flag forces the kernel to satisfy the request from ZONE_DMA. This flag says, I absolutely must have memory into which I can perform DMA. Conversely, the __GFP_HIGHMEM flag instructs the allocator to satisfy the request from either ZONE_NORMAL or (preferentially) ZONE_HIGHMEM. This flag says, I can use high memory, so I can be a doll and hand you back some of that, but normal memory works, too. If neither flag is specified, the kernel fulfills the allocation from either ZONE_DMA or ZONE_NORMAL, with a strong preference to satisfy the allocation from ZONE_NORMAL.

You cannot specify __GFP_HIGHMEM to either __get_free_pages() or kmalloc(). Because these both return a logical address, and not a page structure, it is possible that these functions would allocate memory not currently mapped in the kernel’s virtual address space and, thus, does not have a logical address.

#### Type Flags
The type flags specify the required action and zone modifiers to fulfill a particular type of transaction.

|       Flag      |                                      Description                                        |
|-----------------|-----------------------------------------------------------------------------------------|
|   GFP_ATOMIC    | The allocation is high priority and must not sleep. This is the flag to use in interrupt handlers, in bottom halves, while holding a spin-
lock, and in other situations where you cannot sleep.                                                       |
|   GFP_NOWAIT    | Like GFP_ATOMIC, except that the call will not fallback on emergency memory pools. This increases the liklihood of the memory
allocation failing.                                                                                         |
|    GFP_NOIO     | This allocation can block, but must not initiate disk I/O. This is the flag to use in block I/O code when you cannot cause more disk
I/O, which might lead to some unpleasant recursion.                                                         |
|    GFP_NOFS     | This allocation can block and can initiate disk I/O, if it must, but it will not initiate a filesystem operation. This is the flag to use in
filesystem code when you cannot start another filesystem operation.                                         |
|    GFP_KERNEL   | This is a normal allocation and might block. This is the flag to use in process context code when it is safe to sleep. The kernel will do
whatever it has to do to obtain the memory requested by the caller. This flag should be your default choice.|
|    GFP_USER     | This is a normal allocation and might block. This flag is used to allocate memory for user-space processes. |
|  GFP_HIGHUSER   | This is an allocation from ZONE_HIGHMEM and might block. This flag is used to allocate memory for user-space processes. |
|     GFP_DMA     | This is an allocation from ZONE_DMA. Device drivers that need DMA-able memory use this flag, usually in combination with one of
the preceding flags.                                                                                        |

GFP_ATOMIC specifies a memory allocation that cannot sleep, the allocation is restrictive in the memory it can obtain for the caller. If no sufficiently sized contiguous chunk of memory is available, the kernel is not likely to free memory because it cannot put the caller to sleep. Conversely, the GFP_KERNEL allocation can put the caller to sleep to swap inactive pages to disk, flush dirty pages to disk, and so on.

A GFP_NOIO allocation does not initiate any disk I/O whatsoever to fulfill the request. On the other hand, GFP_NOFS might initiate disk I/O, but does not initiate filesystem I/O. They are needed for certain low-level block I/O or filesystem code, respectively. Imagine if a common path in the filesystem code allocated memory without the GFP_NOFS flag. The allocation could result in more filesystem operations, which would then beget other allocations and, thus, more filesystem operations! Code such as this that invokes the allocator must ensure that the allocator also does not execute it, or else the allocation can create a deadlock.

The GFP_DMA flag is used to specify that the allocator must satisfy the request from ZONE_DMA.This flag is used by device drivers, which need DMA-able memory for their devices.

### kfree()
The counterpart to kmalloc() is kfree(), which is declared in <linux/slab.h>: 

```c
void kfree(const void *ptr)
```

Note that calling kfree(NULL) is explicitly checked for and safe.

## vmalloc()
The vmalloc() function works in a similar fashion to kmalloc(), except it allocates memory that is only virtually contiguous and not necessarily physically contiguous. This is how a user-space allocation function works: The pages returned by malloc() are contiguous within the virtual address space of the processor, but there is no guarantee that they are actually contiguous in physical RAM. The kmalloc() function guarantees that the pages are physically contiguous (and virtually contiguous). The vmalloc() function ensures only that the pages are contiguous within the virtual address space. It does this by allocating potentially noncontiguous chunks of physical memory and “fixing up” the page tables to map the memory into a contiguous chunk of the logical address space.

For the most part, only hardware devices require physically contiguous memory allocations. On many architectures, hardware devices live on the other side of the memory management unit and, thus, do not understand virtual addresses. Consequently, any regions of memory that hardware devices work with must exist as a physically contiguous block and not merely a virtually contiguous one.

Despite the fact that physically contiguous memory is required in only certain cases, most kernel code uses kmalloc() and not vmalloc() to obtain memory. Primarily, this is for performance. The vmalloc() function, to make nonphysically contiguous pages contiguous in the virtual address space, must specifically set up the page table entries. Worse, pages obtained via vmalloc() must be mapped by their individual pages (because they are not physically contiguous), which results in much greater TLB thrashing than you see when directly mapped memory is used. Because of these concerns, vmalloc() is used only when absolutely necessary—typically, to obtain large regions of memory. For example, when modules are dynamically inserted into the kernel, they are loaded into memory created via vmalloc().

The TLB (translation lookaside buffer) is a hardware cache used by most architectures to cache the mapping of virtual addresses to physical addresses. This greatly improves the performance of the system, because most memory access is done via virtual addressing.

The vmalloc() function is declared in <linux/vmalloc.h> and defined in mm/vmalloc.c. Usage is identical to user-space’s malloc():

```c
void * vmalloc(unsigned long size)
```

To free an allocation obtained via vmalloc(), use 

```c
void vfree(const void *addr)
```

The function can also sleep and, thus, cannot be called from interrupt context. It has no return value.


## Slab Layer
To facilitate frequent allocations and deallocations of data, programmers often introduce free lists. A free list contains a block of available, already allocated, data structures.

One of the main problems with free lists in the kernel is that there exists no global control.When available memory is low, there is no way for the kernel to communicate to every free list that it should shrink the sizes of its cache to free up memory. The kernel has no understanding of the random free lists at all. To remedy this, and to consolidate code, the Linux kernel provides the slab layer (also called the slab allocator). The slab layer acts as a generic data structure-caching layer.

The slab layer attempts to leverage several basic tenets:

* Frequently used data structures tend to be allocated and freed often, so cache them.
* Frequent allocation and deallocation can result in memory fragmentation (the inability to find large contiguous chunks of available memory).To prevent this, the cached free lists are arranged contiguously. Because freed data structures return to the free list, there is no resulting fragmentation.
* The free list provides improved performance during frequent allocation and deallocation because a freed object can be immediately returned to the next allocation.
* If the allocator is aware of concepts such as object size, page size, and total cache size, it can make more intelligent decisions.
* If part of the cache is made per-processor (separate and unique to each processor on the system), allocations and frees can be performed without an SMP lock.
* If the allocator is NUMA-aware, it can fulfill allocations from the same memory node as the requestor.
* Stored objects can be colored to prevent multiple objects from mapping to the same cache lines.


### Design of the Slab Layer
The slab layer divides different objects into groups called caches, each of which stores a different type of object. There is one cache per object type. For example, one cache is for process descriptors (a free list of task_struct structures), whereas another cache is for inode objects (struct inode). Interestingly, the kmalloc() interface is built on top of the slab layer, using a family of general purpose caches.

The caches are then divided into slabs. The slabs are composed of one or more physically contiguous pages. Typically, slabs are composed of only a single page. Each cache may consist of multiple slabs. Each slab contains some number of objects, which are the data structures being cached. Each slab is in one of three states: full, partial, or empty. A full slab has no free objects. An empty slab has no allocated objects. A partial slab has some allocated objects and some free objects. When some part of the kernel requests a new object, the request is satisfied from a partial slab, if one exists. Otherwise, the request is satisfied from an empty slab. If there exists no empty slab, one is created. Obviously, a full slab can never satisfy a request because it does not have any free objects. This strategy reduces fragmentation.

Let’s look at the inode structure as an example, which is the in-memory representation of a disk inode. These structures are frequently created and destroyed, so it makes sense to manage them via the slab allocator. Thus, struct inode is allocated from the inode_cachep cache. This cache is made up of one or more slabs—probably a lot of slabs because there are a lot of objects. Each slab contains as many struct inode objects as possible. When the kernel requests a new inode structure, the kernel returns a pointer to an already allocated, but unused structure from a partial slab or, if there is no partial slab, an empty slab. When the kernel is done using the inode object, the slab allocator marks the object as free.

Each cache is represented by a kmem_cache structure. This structure contains three lists—slabs_full, slabs_partial, and slabs_empty—stored inside a kmem_list3 structure, which is defined in mm/slab.c. These lists contain all the slabs associated with the cache. A slab descriptor, struct slab, represents each slab:

```c
struct slab {
    struct list_head list; /* full, partial, or empty list */
    unsigned long colouroff; /* offset for the slab coloring */
    void *s_mem; /* first object in the slab */
    unsigned int inuse; /* allocated objects in the slab */
    kmem_bufctl_t free; /* first free object, if any */
};
```

Slab descriptors are allocated either outside the slab in a general cache or inside the slab itself, at the beginning. The descriptor is stored inside the slab if the total size of the slab is sufficiently small, or if internal slack space is sufficient to hold the descriptor.

The slab allocator creates new slabs by interfacing with the low-level kernel page allocator via __get_free_pages():

```c
static void *kmem_getpages(struct kmem_cache *cachep, gfp_t flags, int nodeid) {
    struct page *page; 
    void *addr;
    int i;
    flags |= cachep->gfpflags; 
    if (likely(nodeid == -1)) {
        addr = (void*)__get_free_pages(flags, cachep->gfporder); 
        if (!addr)
            return NULL;
        page = virt_to_page(addr);
    } else {
        page = alloc_pages_node(nodeid, flags, cachep->gfporder); 
        if (!page)
            return NULL;
        addr = page_address(page);
    }

    i = (1 << cachep->gfporder);
    if (cachep->flags & SLAB_RECLAIM_ACCOUNT)
        atomic_add(i, &slab_reclaim_pages); 
    add_page_state(nr_slab, i);
    while (i––) { 
        SetPageSlab(page);
        page++;
    } 
    return addr;
}
```

The previous function is a bit more complicated than one might expect because code that makes the allocator NUMA-aware. When nodeid is not negative one, the allocator attempts to fulfill the allocation from the same memory node that requested the allocation. This provides better performance on NUMA systems, in which accessing memory outside your node results in a performance penalty.
 
Memory is then freed by kmem_freepages(), which calls free_pages() on the given cache’s pages. In turn, the slab layer invokes the page allocation function only when there does not exist any partial or empty slabs in a given cache. The freeing function is called only when available memory grows low and the system is attempting to free memory, or when a cache is explicitly destroyed.

## Slab Allocator Interface
A new cache is created via

```c
struct kmem_cache * kmem_cache_create(const char *name, 
                                      size_t size,
                                      size_t align, 
                                      unsigned long flags, 
                                      void (*ctor)(void *));
```

The first parameter is a string storing the name of the cache. The second parameter is the size of each element in the cache. The third parameter is the offset of the first object within a slab. This is done to ensure a particular alignment within the page. The flags parameter specifies optional settings controlling the cache’s behavior. It can be zero, specifying no special behavior, or one or more of the following flags OR’ed together:

* SLAB_HWCACHE_ALIGN（没理解）
  This flag instructs the slab layer to align each object within a slab to a cache line.This prevents “false sharing” (two or more objects mapping to the same cache line despite existing at different addresses in memory). This improves performance but comes at a cost of increased memory footprint because the stricter alignment results in more wasted slack space. 
* SLAB_POISON
  This flag causes the slab layer to fill the slab with a known value (a5a5a5a5). This is called poisoning and is useful for catching access to uninitialized memory.

* SLAB_RED_ZONE
  This flag causes the slab layer to insert “red zones” around the allocated memory to help detect buffer overruns.

* SLAB_CACHE_DMA
  This flag instructs the slab layer to allocate each slab in DMA-able memory. This is needed if the allocated object is used for DMA and must reside in ZONE_DMA. 

The final parameter, ctor, is a constructor for the cache.The constructor is called whenever new pages are added to the cache. In practice, caches in the Linux kernel do not often utilize a constructor. In fact, there once was a deconstructor parameter, too, but it was removed because no kernel code used it. You can pass NULL for this parameter.

On success, kmem_cache_create() returns a pointer to the created cache. Otherwise, it returns NULL. This function must not be called from interrupt context because it can sleep.

To destroy a cache, call

```c
int kmem_cache_destroy(struct kmem_cache *cachep)
```

The caller of this function must ensure two conditions are true prior to invoking this function:

* All slabs in the cache are empty. Indeed, if an object in one of the slabs were still allocated and in use, how could the cache be destroyed?
* No one accesses the cache during (and obviously after) a call to kmem_cache_destroy(). The caller must ensure this synchronization.

#### Allocating from the Cache
After a cache is created, an object is obtained from the cache via

```c
void * kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)
```

This function returns a pointer to an object from the given cache cachep. If no free objects are in any slabs in the cache, and the slab layer must obtain new pages via kmem_getpages(), the value of flags is passed to __get_free_pages().

To later free an object and return it to its originating slab, use the function:

```c
void kmem_cache_free(struct kmem_cache *cachep, void *objp) 
```

This marks the object objp in cachep as free.

#### Example of Using the Slab Allocator

Let’s look at a real-life example that uses the task_struct structure (the process descriptor).This code, in slightly more complicated form, is in kernel/fork.c.

First, the kernel has a global variable that stores a pointer to the task_struct cache: 

```c
struct kmem_cache *task_struct_cachep;
```

During kernel initialization, in fork_init(), defined in kernel/fork.c, the cache is created:

```c
task_struct_cachep = kmem_cache_create("task_struct", 
                                       sizeof(struct task_struct),
                                       ARCH_MIN_TASKALIGN, 
                                       SLAB_PANIC | SLAB_NOTRACK, 
                                       NULL);
```

This creates a cache named task_struct, which stores objects of type struct task_struct. The objects are created with an offset of ARCH_MIN_TASKALIGN bytes within the slab. This preprocessor definition is an architecture-specific value. It is usually defined as L1_CACHE_BYTES—the size in bytes of the L1 cache.（没理解）

Note that the return value is not checked for NULL, which denotes failure, because the SLAB_PANIC flag was given. If the allocation fails, the slab allocator calls panic(). If you do not provide this flag, you must check the return! The SLAB_PANIC flag is used here because this is a requisite cache for system operation. 

Each time a process calls fork(), a new process descriptor must be created. This is done in dup_task_struct(), which is called from do_fork():

```c
struct task_struct *tsk;
tsk = kmem_cache_alloc(task_struct_cachep, GFP_KERNEL); 
if (!tsk)
    return NULL;
```

After a task dies, if it has no children waiting on it, its process descriptor is freed and returned to the task_struct_cachep slab cache. This is done in free_task_struct() (in which tsk is the exiting task):

```c
kmem_cache_free(task_struct_cachep, tsk);
```

## Statically Allocating on the Stack
The size of the per-process kernel stacks depends on both the architecture and a compile-time option. Historically, the kernel stack has been two pages per process.This is usually 8KB for 32-bit architectures and 16KB for 64-bit architectures because they usually have 4KB and 8KB pages, respectively.

### Single-Page Kernel Stacks
Early in the 2.6 kernel series, however, an option was introduced to move to single-page kernel stacks.When enabled, each process is given only a single page—4KB on 32-bit architectures and 8KB on 64-bit architectures. This was done for two reasons:

* First, it results in a page with less memory consumption per process. 
* Second and most important is that as uptime increases, it becomes increasingly hard to find two physically contiguous unallocated pages. Physical memory becomes fragmented, and the resulting VM pressure from allocating a single new process is expensive. 

When the stack moved to only a single page, interrupt handlers no longer fit. To rectify this problem, the kernel developers implemented a new feature: interrupt stacks. Interrupt stacks provide a single per-processor stack used for interrupt handlers.

### Playing Fair on the Stack
In any given function, you must keep stack usage to a minimum. There is no hard and fast rule, but you should keep the sum of all local (that is, automatic) variables in a particular function to a maximum of a couple hundred bytes.

Stack overflows occur silently and will undoubtedly result in problems. Because the kernel does not make any effort to manage the stack, when the stack overflows, the excess data simply spills into whatever exists at the tail end of the stack. The first thing to eat it is the thread_info structure. 

## High Memory Mappings
By definition, pages in high memory might not be permanently mapped into the kernel’s address space. Thus, pages obtained via alloc_pages() with the __GFP_HIGHMEM flag might not have a logical address.

On the x86 architecture, all physical memory beyond the 896MB mark is high mem- ory and is not permanently or automatically mapped into the kernel’s address space, despite x86 processors being capable of physically addressing up to 4GB of physical RAM.

### Permanent Mappings
To map a given page structure into the kernel’s address space, use this function, declared in <linux/highmem.h>:

```c
void *kmap(struct page *page)
```

This function works on either high or low memory. If the page structure belongs to a page in low memory, the page’s virtual address is simply returned. If the page resides in high memory, a permanent mapping is created and the address is returned.The function may sleep, so kmap() works only in process context.

Because the number of permanent mappings are limited, high memory should be unmapped when no longer needed. This is done via the following function, which unmaps the given page:

```c
void kunmap(struct page *page)
```

### Temporary Mappings
For times when a mapping must be created but the current context cannot sleep, the kernel provides temporary mappings (which are also called atomic mappings). These are a set of reserved mappings that can hold a temporary mapping. The kernel can atomically map a high memory page into one of these reserved mappings. Consequently, a temporary mapping can be used in places that cannot sleep, such as interrupt handlers, because obtaining the mapping never blocks.

Setting up a temporary mapping is done via

```c
void *kmap_atomic(struct page *page, enum km_type type)
```

This function does not block and thus can be used in interrupt context and other places that cannot reschedule. It also disables kernel preemption, which is needed because the mappings are unique to each processor. (And a reschedule might change which task is running on which processor.)

The mapping is undone via

```c
void kunmap_atomic(void *kvaddr, enum km_type type)
```

This function also does not block. In many architectures it does not do anything at all except enable kernel preemption, because a temporary mapping is valid only until the next temporary mapping.

## Per-CPU Allocations
Modern SMP-capable operating systems use per-CPU data—data that is unique to a given processor—extensively.Typically, per-CPU data is stored in an array. Each item in the array corresponds to a possible processor on the system.

You declare the data as

```c
unsigned long my_percpu[NR_CPUS];
```

Then you access it as

```c
int cpu;
cpu = get_cpu(); /* get current processor and disable kernel preemption */ 
my_percpu[cpu]++; /* ... or whatever */
printk(“my_percpu on cpu=%d is %lu\n”, cpu, my_percpu[cpu]);
put_cpu(); /* enable kernel preemption */
```

Kernel preemption is the only concern with per-CPU data. Kernel preemption poses two problems, listed here:

* If your code is preempted and reschedules on another processor, the cpu variable is no longer valid because it points to the wrong processor. (In general, code cannot sleep after obtaining the current processor.)
* If another task preempts your code, it can concurrently access my_percpu on the same processor, which is a race condition.

## The New percpu Interface
The 2.6 kernel introduced a new interface, known as percpu, for creating and manipulating per-CPU data. This interface generalizes the previous example. Creation and manipulation of per-CPU data is simplified with this new approach.

### Per-CPU Data at Compile-Time

Defining a per-CPU variable at compile time is quite easy:

```c
DEFINE_PER_CPU(type, name);
```

This creates an instance of a variable of type type, named name, for each processor on the system. If you need a declaration of the variable elsewhere, to avoid compile warnings, the following macro is your friend:

```c
DECLARE_PER_CPU(type, name);
```

You can manipulate the variables with the get_cpu_var() and put_cpu_var() routines. A call to get_cpu_var() returns an lvalue for the given variable on the current processor. It also disables preemption, which put_cpu_var() correspondingly enables.

```c
get_cpu_var(name)++; /* increment name on this processor */ 
put_cpu_var(name); /* done; enable kernel preemption */
```

You can obtain the value of another processor’s per-CPU data, too: 

```c
per_cpu(name, cpu)++; /* increment name on the given processor */
```

You need to be careful with this approach because per_cpu() neither disables kernel preemption nor provides any sort of locking mechanism. The lockless nature of per-CPU data exists only if the current processor is the only manipulator of the data. 

These compile-time per-CPU examples do not work for modules because the linker actually creates them in a unique executable section (for the curious, .data.percpu). If you need to access per-CPU data from modules, or if you need to create such data dynamically, there is hope.（没理解）

### Per-CPU Data at Runtime
The kernel implements a dynamic allocator, similar to kmalloc(), for creating per-CPU data. This routine creates an instance of the requested memory for each processor on the systems. The prototypes are in <linux/percpu.h>:

```c
void *alloc_percpu(type); /* a macro */
void *__alloc_percpu(size_t size, size_t align);
void free_percpu(const void *);
```

The alloc_percpu() macro allocates one instance of an object of the given type for every processor on the system. It is a wrapper around __alloc_percpu(), which takes the actual number of bytes to allocate as a parameter and the number of bytes on which to align the allocation.

A call to alloc_percpu() or __alloc_percpu() returns a pointer, which is used to indirectly reference the dynamically created per-CPU data. The kernel provides two macros to make this easy:

```c
get_cpu_var(ptr); /* return a void pointer to this processor’s copy of ptr */ 
put_cpu_var(ptr); /* done; enable kernel preemption */
```

The get_cpu_var() macro returns a pointer to the specific instance of the current processor’s data. It also disables kernel preemption, which a call to put_cpu_var() then enables.

## Reasons for Using Per-CPU Data
There are several benefits to using per-CPU data:
* The first is the reduction in locking requirements.  
* Second, per-CPU data greatly reduces cache invalidation. This occurs as processors try to keep their caches in sync. If one processor manipulates data held in another processor’s cache, that processor must flush or otherwise update its cache. Constant cache invalidation is called thrashing the cache and wreaks havoc on system performance. The percpu interface cache-aligns all data to ensure that accessing one processor’s data does not bring in another processor’s data on the same cache line.（不太理解）

# Chapter13. The Virtual Filesystem
The Virtual Filesystem (sometimes called the Virtual File Switch or more commonly simply the VFS) is the subsystem of the kernel that implements the file and filesystem-related interfaces provided to user-space programs. All filesystems rely on theVFS to enable them not only to coexist, but also to interoperate. This enables programs to use standard Unix system calls to read and write to different filesystems, even on different media.

## Common Filesystem Interface
The VFS is the glue that enables system calls such as open(), read(), and write() to work regardless of the filesystem or underlying physical medium. More so, the system calls work between these different filesystems and media—we can use standard system calls to copy or move files from one filesystem to another.


## Filesystem Abstraction Layer
The abstraction layer works by defining the basic conceptual interfaces and data structures that all filesystems support.

Following figure shows the flow from user-space’s write() call through the data arriving on the physical media. On one side of the system call is the genericVFS interface, providing the frontend to user-space; on the other side of the system call is the filesystem-specific backend, dealing with the implementation details.

<img src="/assets/images/linux-kernel-development-note/illustration-3.png" width="800" />

## Unix Filesystems
Historically, Unix has provided four basic filesystem-related abstractions: files, directory entries, inodes, and mount points.

A filesystem is a hierarchical storage of data adhering to a specific structure. Typical operations performed on filesystems are creation, deletion, and mounting. In Unix, filesystems are mounted at a specific mount point in a global hierarchy known as a namespace. This enables all mounted filesystems to appear as entries in a single tree.

The Unix concept of the file is in stark contrast to record-oriented filesystems, such as OpenVMS’s Files-11. Record-oriented filesystems provide a richer, more structured representation of files than Unix’s simple byte-stream abstraction, at the cost of simplicity and flexibility.

Directories may be nested to form paths. Each component of a path is called a directory entry. A path example is /home/wolfman/butter—the root directory /, the directories home and wolfman, and the file butter are all directory entries, called dentries. In Unix, directories are actually normal files that simply list the files contained therein. Because a directory is a file to theVFS,the same operations performed on files can be performed on directories.

Unix systems separate the concept of a file from any associated information about it, such as access permissions, size, owner, creation time, and so on.This information is sometimes called file metadata (that is, data about the file’s data) and is stored in a separate data structure from the file, called the inode.

All this information is tied together with the filesystem’s own control information, which is stored in the superblock. The superblock is a data structure containing information about the filesystem as a whole. Sometimes the collective data is referred to as filesystem metadata. Filesystem metadata includes information about both the individual files and the filesystem as a whole.

Traditionally, Unix filesystems implement these notions as part of their physical ondisk layout. The Unix file concepts are physically mapped on to the storage medium.

## VFS Objects and Their Data Structures
The VFS is object-oriented. A family of data structures represents the common file model. The structures contain both data and pointers to filesystem-implemented functions that operate on the data.

The four primary object types of theVFS are:

* The superblock object, which represents a specific mounted filesystem.
* The inode object, which represents a specific file.
* The dentry object, which represents a directory entry, which is a single component of a path.
* The file object, which represents an open file as associated with a process.

An operations object is contained within each of these primary objects. These objects describe the methods that the kernel invokes against the primary objects:

* The super_operations object, which contains the methods that the kernel can invoke on a specific filesystem, such as write_inode() and sync_fs()
* The inode_operations object, which contains the methods that the kernel can invoke on a specific file, such as create() and link()
* The dentry_operations object, which contains the methods that the kernel can invoke on a specific directory entry, such as d_compare() and d_delete()
* The file_operations object, which contains the methods that a process can invoke on an open file, such as read() and write()

The operations objects are implemented as a structure of pointers to functions that operate on the parent object. For many methods, the objects can inherit a generic function if basic functionality is sufficient. Otherwise, the specific instance of the particular filesystem fills in the pointers with its own filesystem-specific methods.

Each registered filesystem is represented by a file_system_type structure. This object describes the filesystem and its capabilities. Fur- thermore, each mount point is represented by the vfsmount structure. This structure contains information about the mount point, such as its location and mount flags.

Two per-process structures describe the filesystem and files associated with a process. They are, respectively, the fs_struct structure and the file structure.

## The Superblock Object
The superblock object is implemented by each filesystem and is used to store information describing that specific filesystem. This object usually corresponds to the filesystem superblock or the filesystem control block, which is stored in a special sector on disk.

The superblock object is represented by struct super_block and defined in <linux/fs.h>. The code for creating, managing, and destroying superblock objects lives in fs/super.c. A superblock object is created and initialized via the alloc_super() func- tion.When mounted, a filesystem invokes this function, reads its superblock off of the disk, and fills in its superblock object.

## Superblock Operations
The most important item in the superblock object is s_op, which is a pointer to the superblock operations table. The superblock operations table is represented by struct super_operations and is defined in <linux/fs.h>.

## The Inode Object
The inode object represents all the information needed by the kernel to manipulate a file or directory. For Unix-style filesystems, this information is simply read from the on-disk inode. If a filesystem does not have inodes, however, the filesystem must obtain the information from wherever it is stored on the disk. Filesystems without inodes generally store file-specific information as part of the file; unlike Unix-style filesystems, they do not separate file data from its control information. Some modern filesystems do neither and store file metadata as part of an on-disk database. Whatever the case, the inode object is constructed in memory in whatever manner is applicable to the filesystem.

The inode object is represented by struct inode and is defined in <linux/fs.h>. An inode represents each file on a filesystem, but the inode object is constructed in memory only as files are accessed. This includes special files, such as device files or pipes. Consequently, some of the entries in struct inode are related to these special files. For example, the i_pipe entry points to a named pipe data structure, i_bdev points to a block device structure, and i_cdev points to a character device structure.These three pointers are stored in a union because a given inode can represent only one of these (or none of them) at a time.

## Inode Operations
The inode_operations member describes the filesystem’s implemented functions that theVFS can invoke on an inode. As with the superblock, inode operations are invoked via

```c
i->i_op->truncate(i)
```

## The Dentry Object
The VFS often needs to perform directory-specific operations, such as path name lookup. Path name lookup involves translating each component of a path, ensuring it is valid, and following it to the next component. To facilitate this, the VFS employs the concept of a directory entry (dentry). A dentry is a specific component in a path. 

Dentries might also include mount points. In the path /mnt/cdrom/foo, the components /,mnt,cdrom, and foo are all dentry objects. The VFS constructs dentry objects on-the-fly, as needed, when performing directory operations.

Dentry objects are represented by struct dentry and defined in <linux/dcache.h>.

Unlike the previous two objects, the dentry object does not correspond to any sort of on-disk data structure. The VFS creates it on-the-fly from a string representation of a path name. Because the dentry object is not physically stored on the disk, no flag in struct dentry specifies whether the object is modified.

### Dentry State
A valid dentry object can be in one of three states: used, unused, or negative:

* A used dentry corresponds to a valid inode (d_inode points to an associated inode) and indicates that there are one or more users of the object (d_count is positive). A used dentry is in use by the VFS and points to valid data and, thus, cannot be discarded.
* An unused dentry corresponds to a valid inode (d_inode points to an inode), but the VFS is not currently using the dentry object (d_count is zero).  If it is necessary to reclaim memory, however, the dentry can be discarded because it is not in active use.
* A negative dentry is not associated with a valid inode (d_inode is NULL) because either the inode was deleted or the path name was never correct to begin with. The dentry is kept around, however, so that future lookups are resolved quickly. Because even this failed lookup is expensive, caching the “negative” results are worthwhile.

### The Dentry Cache
After the VFS layer goes through the trouble of resolving each element in a path name into a dentry object and arriving at the end of the path, it would be quite wasteful to throw away all that work. Instead, the kernel caches dentry objects in the dentry cache or, simply, the dcache.

The dentry cache consists of three parts:
* Lists of "used" dentries linked off their associated inode via the i_dentry field of the inode object. Because a given inode can have multiple links, there might be multiple dentry objects; consequently, a list is used.（没理解）
* A doubly linked "least recently used" list of unused and negative dentry objects. The list is inserted at the head. When the kernel must remove entries to reclaim memory, the entries are removed from the tail; those are the oldest and presumably have the least chance of being used in the near future.
* A hash table and hashing function used to quickly resolve a given path into the associated dentry object.

The actual hash value is determined by d_hash(). This enables filesystems to provide a unique hashing function.

Hash table lookup is performed via d_lookup(). If a matching dentry object is found in the dcache, it is returned. On failure, NULL is returned.

The dcache also provides the front end to an inode cache, the icache. Inode objects that are associated with dentry objects are not freed because the dentry maintains a positive usage count over the inode. As long as the dentry is cached, the corresponding inodes are cached, too. 

## Dentry Operations
The dentry_operations structure specifies the methods that the VFS invokes on directory entries on a given filesystem. The dentry_operations structure is defined in <linux/dcache.h>

## The File Object
The file object is used to represent a file opened by a process.

The file object is the in-memory representation of an open file. The object (but not the physical file) is created in response to the open() system call and destroyed in response to the close() system call. All these file-related calls are actually methods defined in the file operations table. Because multiple processes can open and manipulate a file at the same time, there can be multiple file objects in existence for the same file. The object points back to the dentry (which in turn points back to the inode) that actually represents the open file. The inode and dentry objects, of course, are unique.

The file object is represented by struct file and is defined in <linux/fs.h>.

Similar to the dentry object, the file object does not actually correspond to any on-disk data.Therefore, no flag in the object represents whether the object is dirty and needs to be written back to disk.The file object does point to its associated dentry object via the f_dentry pointer.

## Data Structures Associated with Filesystems
In addition to the fundamentalVFS objects, the kernel uses other standard data structures to manage data related to filesystems. The first object is used to describe a specific variant of a filesystem. The second data structure describes a mounted instance of a filesystem.

Because Linux supports so many different filesystems, the kernel must have a special structure for describing the capabilities and behavior of each filesystem. The file_system_type structure, defined in <linux/fs.h>, accomplishes this.

Things get more interesting when the filesystem is actually mounted, at which point the vfsmount structure is created.This structure represents a specific instance of a filesystem-in other words, a mount point. The vfsmount structure is defined in <linux/mount.h>.

The complicated part of maintaining the list of all mount points is the relation between the filesystem and all the other mount points. The various linked lists in vfsmount keep track of this information.

The vfsmount structure also stores the flags, if any, specified on mount in the mnt_flags field.

|       Flag     |                         Description                              |
|   MNT_NOSUID   | Forbids setuid and setgid flags on binaries on this filesystem   |
|   MNT_NODEV    | Forbids access to device files on this filesystem                |
|   MNT_NOEXEC   | Forbids execution of binaries on this filesystem                 |

## Data Structures Associated with a Process
Each process on the system has its own list of open files, root filesystem, current working directory, mount points, and so on. Three data structures tie together theVFS layer and the processes on the system: files_struct, fs_struct, and namespace.

The files_struct is defined in <linux/fdtable.h>.This table’s address is pointed to by the files entry in the processor descriptor. All per-process information about open files and file descriptors is contained therein:

```c
struct files_struct { 
    atomic_t  count;  /* usage count */
    struct fdtable  *fdt;  /* pointer to other fd table */ 
    struct fdtable  fdtab;  /* base fd table */
    spinlock_t file_lock;  /* per-file lock */
    int next_fd;  /* cache of next available fd */
    struct embedded_fd_set close_on_exec_init; /* list of close-on-exec fds */ 
    struct embedded_fd_set open_fds_init /* list of open fds */ 
    struct file *fd_array[NR_OPEN_DEFAULT]; /* base files array */
};
```

The array fd_array points to the list of open file objects. Because NR_OPEN_DEFAULT is equal to BITS_PER_LONG, which is 64 on a 64-bit architecture; this includes room for 64 file objects. If a process opens more than 64 file objects, the kernel allocates a new array and points the fdt pointer at it. 

The second process-related structure is fs_struct, which contains filesystem information related to a process and is pointed at by the fs field in the process descriptor. The structure is defined in <linux/fs_struct.h>:

```c
struct fs_struct { 
    int users;  /* user count */
    rwlock_t lock;  /* per-structure lock */
    int umask;  /* umask */
    int in_exec;  /* currently executing a file */
    struct path root;  /* root directory */
    struct path pwd;  /* current working directory */
}
```

This structure holds the current working directory (pwd) and root directory of the current process.

The third and final structure is the namespace structure, which is defined in <linux/mnt_namespace.h> and pointed at by the mnt_namespace field in the process descriptor. Per-process namespaces were added to the 2.4 Linux kernel. They enable each process to have a unique view of the mounted filesystems on the system:

```c
struct mnt_namespace {
    atomic_t count; /* usage count */ 
    struct vfsmount *root; /* root directory */
    struct list_head list; /* list of mount points */
    wait_queue_head_t poll; /* polling waitqueue */
    int event; /* event count */
};
```

The list member specifies a doubly linked list of the mounted filesystems that make up the namespace.

These data structures are linked from each process descriptor. For most processes, the process descriptor points to unique files_struct and fs_struct structures. For processes created with the clone flag CLONE_FILES or CLONE_FS, however, these structures are shared.

The namespace structure works the other way around. By default, all processes share the same namespace. Only when the CLONE_NEWNS flag is specified during clone() is the process given a unique copy of the namespace structure. Because most processes do not provide this flag, all the processes inherit their parents’ namespaces.

# Chapter14. The Block I/O Layer
The most common block device is a hard disk, but many other block devices exist, such as floppy drives, Blu-ray readers, and flash memory. Notice how these are all devices on which you mount a filesystem—filesystems are the lingua franca of block devices.

The other basic type of device is a character device. Character devices, or char devices, are accessed as a stream of sequential data, one byte after another. Example character devices are serial ports and keyboards. If the hardware device is accessed as a stream of data, it is implemented as a character device. On the other hand, if the device is accessed randomly (nonsequentially), it is a block device.

The difference comes down to whether the device accesses data randomly—in other words, whether the device can seek to one position from another.

Managing block devices in the kernel requires more care, preparation, and work than managing character devices.

## Anatomy of a Block Device
The smallest addressable unit on a block device is a sector. Sectors come in various powers of two, but 512 bytes is the most common size. The sector size is a physical property of the device, and the sector is the fundamental unit of all block devices—the device cannot address or operate on a unit smaller than the sector.

Software has different goals and therefore imposes its own smallest logically addressable unit, which is the block. The block is an abstraction of the filesystem—filesystems can be accessed only in multiples of a block. Although the physical device is addressable at the sector level, the kernel performs all disk operations in terms of blocks. Because the device’s smallest addressable unit is the sector, the block size can be no smaller than the sector and must be a multiple of a sector. Furthermore, the kernel (as with hardware and the sector) needs the block to be a power of two. The kernel also requires that a block be no larger than the page size. Therefore, block sizes are a power-of-two multiple of the sector size and are not greater than the page size. Common block sizes are 512 bytes, 1 kilobyte, and 4 kilobytes.

## Buffers and Buffer Heads
When a block is stored in memory—say, after a read or pending a write—it is stored in a buffer. Each buffer is associated with exactly one block. The buffer serves as the object that represents a disk block in memory. Each buffer is associated with a descriptor. The descriptor is called a buffer head and is of type struct buffer_head. The buffer_head structure holds all the information that the kernel needs to manipulate buffers and is defined in <linux/buffer_head.h>.

The b_state field specifies the state of this particular buffer. The legal flags are stored in the bh_state_bits enumeration, which is defined in <linux/buffer_head.h>.

The b_count field is the buffer’s usage count. The value is incremented and decremented by two inline functions, both of which are defined in <linux/buffer_head.h>:

```c
static inline void get_bh(struct buffer_head *bh) {
    atomic_inc(&bh->b_count);
}

static inline void put_bh(struct buffer_head *bh) {
    atomic_dec(&bh->b_count);
}
```

Before manipulating a buffer head, you must increment its reference count via get_bh() to ensure that the buffer head is not deallocated out from under you. When finished with the buffer head, decrement the reference count via put_bh().

The physical block on disk to which a given buffer corresponds is the b_blocknr-the logical block on the block device described by b_bdev.

The physical page in memory to which a given buffer corresponds is the page pointed to by b_page. More specifically, b_data is a pointer directly to the block (that exists somewhere in b_page), which is b_size bytes in length.

The purpose of a buffer head is to describe this mapping between the on-disk block and the physical in-memory buffer.

## The bio Structure
The basic container for block I/O within the kernel is the bio structure, which is defined in <linux/bio.h>. This structure represents block I/O operations that are in flight (active) as a list of segments. A segment is a chunk of a buffer that is contiguous in memory. Thus, individual buffers need not be contiguous in memory. By allowing the buffers to be described in chunks, the bio structure provides the capability for the kernel to perform block I/O operations of even a single buffer from multiple locations in memory. Vector I/O such as this is called scatter-gather I/O.

The primary purpose of a bio structure is to represent an in-flight block I/O operation. To this end, the majority of the fields in the structure are housekeeping related. The most important fields are bi_io_vec, bi_vcnt, and bi_idx. Figure shows the relationship between the bio structure and its friends:

<img src="/assets/images/linux-kernel-development-note/illustration-4.png" width="800" />

### I/O vectors
The bi_io_vec field points to an array of bio_vec structures. These structures are used as lists of individual segments in this specific block I/O operation. Each bio_vec is treated as a vector of the form <page, offset, len>, which describes a specific segment: the physical page on which it lies, the location of the block as an offset into the page, and the length of the block starting from the given offset. The bio_vec structure is defined in <linux/bio.h>:

```c
struct bio_vec {
    /* pointer to the physical page on which this buffer resides */ 
    struct page *bv_page;  
    
    /* the length in bytes of this buffer */ 
    unsigned int bv_len;

    /* the byte offset within the page where the buffer resides */ 
    unsigned int bv_offset;
};
```

In each given block I/O operation, there are bi_vcnt vectors in the bio_vec array starting with bi_io_vec. As the block I/O operation is carried out, the bi_idx field is used to point to the current index into the array.

In summary, each block I/O request is represented by a bio structure. Each request is composed of one or more blocks, which are stored in an array of bio_vec structures. These structures act as vectors and describe each segment’s location in a physical page in memory. The first segment in the I/O operation is pointed to by b_io_vec. Each additional segment follows after the first, for a total of bi_vcnt segments in the list. As the block I/O layer submits segments in the request, the bi_idx field is updated to point to the current segment.

The bi_idx field is used to point to the current bio_vec in the list, which helps the block I/O layer keep track of partially completed block I/O operations.A more impor- tant usage, however, is to allow the splitting of bio structures.

The bio structure maintains a usage count in the bi_cnt field. When this field reaches zero, the structure is destroyed and the backing memory is freed.The following two functions manage the usage counters for you.

```c
void bio_get(struct bio *bio) 
void bio_put(struct bio *bio)
```

### The Old Versus the New
The bio structure represents an I/O operation, which may include one or more pages in memory. On the other hand, the buffer_head structure represents a single buffer, which describes a single block on the disk. Because buffer heads are tied to a single disk block in a single page, buffer heads result in the unnecessary dividing of requests into block-sized chunks, only to later reassemble them. 

Switching from struct buffer_head to struct bio provided other benefits, as well: 

* The bio structure can easily represent high memory, because struct bio deals with only physical pages and not direct pointers.
* The bio structure can represent both normal page I/O and direct I/O (I/O operations that do not go through the page cache.
* The bio structure makes it easy to perform scatter-gather (vectored) block I/O operations, with the data involved in the operation originating from multiple physical pages.
* The bio structure is much more lightweight than a buffer head because it contains only the minimum information needed to represent a block I/O operation and not unnecessary information related to the buffer itself.

The concept of buffer heads is still required, however; buffer heads function as descriptors, mapping disk blocks to pages. The bio structure does not contain any information about the state of a buffer—it is simply an array of vectors describing one or more segments of data for a single block I/O operation, plus related information. 

## Request Queues
Block devices maintain request queues to store their pending block I/O requests. The request queue is represented by the request_queue structure and is defined in <linux/blkdev.h>. The request queue contains a doubly linked list of requests and associated control information. Requests are added to the queue by higher-level code in the kernel, such as filesystems. Each item in the queue’s request list is a single request, of type struct request. Individual requests on the queue are represented by struct request, which is also defined in <linux/blkdev.h>. Each request can be composed of more than one bio structure because individual requests can operate on multiple consecutive disk blocks.

## I/O Schedulers
Simply sending out requests to the block devices in the order that the kernel issues them, as soon as it issues them, results in poor performance. One of the slowest operations in a modern computer is disk seeks. 

Therefore, the kernel does not issue block I/O requests to the disk in the order they are received or as soon as they are received. Instead, it performs operations called merging and sorting to greatly improve the performance of the system as a whole. The subsystem of the kernel that performs these operations is called the I/O scheduler.

### The Job of an I/O Scheduler
An I/O scheduler works by managing a block device’s request queue. It decides the order of requests in the queue and at what time each request is dispatched to the block device.

I/O schedulers perform two primary actions to minimize seeks: merging and sorting. Merging is the coalescing of two or more requests into one. The entire request queue is kept sorted, sectorwise, so that all seeking activity along the queue moves (as much as possible) sequentially over the sectors of the hard disk. The goal is not just to minimize each individual seek but to minimize all seeking by keeping the disk head moving in a straight line.

### The Linus Elevator
The Linus Elevator performs both merging and sorting. When a request is added to the queue, it is first checked against every other pending request to see whether it is a possible candidate for merging. The Linus Elevator I/O scheduler performs both front and back merging.

If the merge attempt fails, a possible insertion point in the queue (a location in the queue where the new request fits sectorwise between the existing requests) is then sought. If one is found, the new request is inserted there. If a suitable location is not found, the request is added to the tail of the queue. Additionally, if an existing request is found in the queue that is older than a predefined threshold, the new request is added to the tail of the queue even if it can be insertion sorted elsewhere. 

In summary, when a request is added to the queue, four operations are possible. In order, they are:

* If a request to an adjacent on-disk sector is in the queue, the existing request and the new request merge into a single request.
* If a request in the queue is sufficiently old, the new request is inserted at the tail of the queue to prevent starvation of the other, older, requests.
* If a suitable location sector-wise is in the queue, the new request is inserted there. This keeps the queue sorted by physical location on disk.
* Finally, if no such suitable insertion point exists, the request is inserted at the tail of the queue.

The Linus elevator is implemented in block/elevator.c.

### The Deadline I/O Scheduler
The Deadline I/O scheduler sought to prevent the starvation caused by the Linus Elevator. The general issue of request starvation introduces a specific instance of the problem known as writes starving reads. Recognizing that the asynchrony and interdependency of read requests results in a much stronger bearing of read latency on the performance of the system, the Deadline I/O scheduler implements several features to ensure that request starvation in general, and read starvation in specific, is minimized.

In the Deadline I/O scheduler, each request is associated with an expiration time. By default, the expiration time is 500 milliseconds in the future for read requests and 5 seconds in the future for write requests. The Deadline I/O scheduler operates similarly to the Linus Elevator in that it maintains a request queue sorted by physical location on disk. It calls this queue the sorted queue. When a new request is submitted to the sorted queue, the Deadline I/O scheduler performs merging and insertion like the Linus Elevator. The Deadline I/O scheduler also, however, inserts the request into a second queue that depends on the type of request. Read requests are sorted into a special read FIFO queue, and write requests are inserted into a special write FIFO queue. Under normal operation, the Deadline I/O scheduler pulls requests from the head of the sorted queue into the dispatch queue. The dispatch queue is then fed to the disk drive.

If the request at the head of either the write FIFO queue or the read FIFO queue expire, the Deadline I/O scheduler then begins servicing requests from the FIFO queue. Because read requests are given a substantially smaller expiration value than write requests, the Deadline I/O scheduler also works to ensure that write requests do not starve read requests.

The Deadline I/O scheduler lives in block/deadline-iosched.c.

### The Anticipatory I/O Scheduler
Although the Deadline I/O scheduler does a great job minimizing read latency, it does so at the expense of global throughput. 

First, the Anticipatory I/O scheduler starts with the Deadline I/O scheduler as its base. The Anticipatory I/O scheduler implements three queues (plus the dispatch queue) and expirations for each request, just like the Deadline I/O scheduler. The major change is the addition of an anticipation heuristic.

The Anticipatory I/O scheduler attempts to minimize the seek storm that accompanies read requests issued during other disk I/O activity. When a read request is issued, it is handled as usual, within its usual expiration period.After the request is submitted, however, the Anticipatory I/O scheduler does not immediately seek back and return to handling other requests. Instead, it does absolutely nothing for a few milliseconds. In those few milliseconds, there is a good chance that the application will submit another read request.Any requests issued to an adjacent area of the disk are immediately handled. After the waiting period elapses, the Anticipatory I/O scheduler seeks back to where it left off and continues handling the previous requests.

It is important to note that the few milliseconds spent in anticipation for more requests are well worth it if they minimize even a modest percentage of the back-and-forth seeking that results from the servicing of read requests during other heavy requests. The key to reaping maximum benefit from the Anticipatory I/O scheduler is correctly anticipating the actions of applications and filesystems. This is done via a set of statistics and associated heuristics. The Anticipatory I/O scheduler keeps track of per-process statistics pertaining to block I/O habits in hopes of correctly anticipating the actions of applications.

The Anticipatory I/O scheduler lives in the file block/as-iosched.c in the kernel source tree.

### The Complete Fair Queuing I/O Scheduler
The Complete Fair Queuing (CFQ) I/O scheduler is an I/O scheduler designed for specialized workloads, but that in practice actually provides good performance across multiple workloads. The CFQ I/O scheduler assigns incoming I/O requests to specific queues based on the process originating the I/O request. Within each queue, requests are coalesced with adjacent requests and insertion sorted.

The CFQ I/O scheduler then services the queues round robin, plucking a configurable number of requests (by default, four) from each queue before continuing on to the next. This provides fairness at a per-process level, assuring that each process receives a fair slice of the disk’s bandwidth.

The Complete Fair Queuing I/O scheduler lives in block/cfq-iosched.c. It is recommended for desktop workloads, although it performs reasonably well in nearly all workloads without any pathological corner cases. 

### The Noop I/O Scheduler
A fourth and final I/O scheduler is the Noop I/O scheduler, so named because it is basically a noop—it does not do much.

The Noop I/O scheduler does perform merging, however, as its lone chore.

The Noop I/O scheduler’s lack of hard work is with reason. It is intended for block devices that are truly random-access, such as flash memory cards.

The Noop I/O scheduler lives in block/noop-iosched.c. It is intended only for random-access devices.

# Chapter15. The Process Address Space
In addition to managing its own memory, the kernel also has to manage the memory of user-space processes. This memory is called the process address space, which is the representation of memory given to each user-space process on the system.

## Address Spaces
The process address space consists of the virtual memory addressable by a process and the addresses within the virtual memory that the process is allowed to use. Each process is given a flat 32- or 64-bit address space, with the size depending on the architecture. The term flat denotes that the address space exists in a single range.

Normally, this flat address space is unique to each process. A memory address in one process’s address space is completely unrelated to that same memory address in another process’s address space. Alternatively, processes can elect to share their address space with other processes. We know these processes as threads.

Although a process can address up to 4GB of memory (with a 32-bit address space), it doesn’t have permission to access all of it. These intervals of legal addresses are called memory areas. The process, through the kernel, can dynamically add and remove memory areas to its address space.

The process can access a memory address only in a valid memory area. Memory areas have associated permissions, such as readable, writable, and executable, that the associated process must respect. If a process accesses a memory address not in a valid memory area, or if it accesses a valid area in an invalid manner, the kernel kills the process with the dreaded "Segmentation Fault" message.

Memory areas can contain all sorts of goodies, such as

* A memory map of the executable file’s code, called the text section.
* A memory map of the executable file’s initialized global variables, called the data section.
* A memory map of the zero page (a page consisting of all zeros, used for purposes such as this) containing uninitialized global variables, called the bss section.1
* A memory map of the zero page used for the process’s user-space stack. (Do not confuse this with the process’s kernel stack, which is separate and maintained and used by the kernel.)
* An additional text, data, and bss section for each shared library, such as the C library and dynamic linker, loaded into the process’s address space.
* Any memory mapped files.
* Any shared memory segments.
* Any anonymous memory mappings, such as those associated with malloc().

All valid addresses in the process address space exist in exactly one area; memory areas do not overlap. As you can see, there is a separate memory area for each different chunk of memory in a running process.

### The Memory Descriptor
The kernel represents a process’s address space with a data structure called the memory descriptor. The memory descriptor is represented by struct mm_struct and defined in <linux/mm_types.h>.

The mm_users field is the number of processes using this address space. For example, if two threads share this address space,mm_users is equal to two. The mm_count field is the primary reference count for the mm_struct. All mm_users equate to one increment of mm_count.

The mmap and mm_rb fields are different data structures that contain the same thing: all the memory areas in this address space. The former stores them in a linked list, whereas the latter stores them in a red-black tree. The mmap data structure, as a linked list, allows for simple and efficient traversing of all elements. On the other hand, the mm_rb data structure, as a red-black tree, is more suitable to searching for a given element.

All of the mm_struct structures are strung together in a doubly linked list via the mmlist field. The initial element in the list is the init_mm memory descriptor, which describes the address space of the init process. The list is protected from concurrent access via the mmlist_lock, which is defined in kernel/fork.c.

#### Allocating a Memory Descriptor
The memory descriptor associated with a given task is stored in the mm field of the task’s process descriptor. Thus, current->mm is the current process’s memory descriptor. The copy_mm() function copies a parent’s memory descriptor to its child during fork(). The mm_struct structure is allocated from the mm_cachep slab cache via the allocate_mm() macro in kernel/fork.c.

Processes may elect to share their address spaces with their children by means of the CLONE_VM flag to clone(). The process is then called a thread. In the case that CLONE_VM is specified, allocate_mm() is not called, and the process’s mm field is set to point to the memory descriptor of its parent via this logic in copy_mm():

```c
if (clone_flags & CLONE_VM) {
    /*
     * current is the parent process and
     * tsk is the child process during a fork()
     */
    atomic_inc(&current->mm->mm_users); 
    tsk->mm = current->mm;
}
```

#### Destroying a Memory Descriptor
When the process associated with a specific address space exits, the exit_mm(), defined in kernel/exit.c, function is invoked. This function performs some housekeeping and updates some statistics. It then calls mmput(), which decrements the memory descriptor’s mm_users user counter. If the user count reaches zero, mmdrop() is called to decrement the mm_count usage counter. If that counter is finally zero, the free_mm() macro is invoked to return the mm_struct to the mm_cachep slab cache via kmem_cache_free().

#### The mm_struct and Kernel Threads
Kernel threads do not have a process address space and therefore do not have an associated memory descriptor. Thus, the mm field of a kernel thread’s process descriptor is NULL. This is the definition of a kernel thread—processes that have no user context.

To provide kernel threads the needed data, without wasting memory on a memory descriptor and page tables, or wasting processor cycles to switch to a new address space whenever a kernel thread begins running, kernel threads use the memory descriptor of whatever task ran previously.

Whenever a process is scheduled, the process address space referenced by the process’s mm field is loaded. The active_mm field in the process descriptor is then updated to refer to the new address space. Kernel threads do not have an address space and mm is NULL. Therefore, when a kernel thread is scheduled, the kernel notices that mm is NULL and keeps the previous process’s address space loaded. The kernel then updates the active_mm field of the kernel thread’s process descriptor to refer to the previous process’s memory descriptor. The kernel thread can then use the previous process’s page tables as needed. Because kernel threads do not access user-space memory, they make use of only the information in the address space pertaining to kernel memory, which is the same for all processes.

## Virtual Memory Areas
The memory area structure, vm_area_struct, represents memory areas. It is defined in <linux/mm_types.h>. In the Linux kernel, memory areas are often called virtual memory areas (abbreviated VMAs).

The vm_area_struct structure describes a single memory area over a contiguous interval in a given address space. The kernel treats each memory area as a unique memory object. Each memory area possesses certain properties, such as permissions and a set of associated operations. 

Recall that each memory descriptor is associated with a unique interval in the process’s address space. The vm_start field is the initial (lowest) address in the interval, and the vm_end field is the first byte after the final (highest) address in the interval.

The vm_mm field points to this VMA’s associated mm_struct. Note that each VMA is unique to the mm_struct with which it is associated.

### VMA Flags
The vm_flags field contains bit flags, defined in <linux/mm.h>, that specify the behavior of and provide information about the pages contained in the memory area. Unlike permissions associated with a specific physical page, the VMA flags specify behavior for which the kernel is responsible, not the hardware. 

The VM_READ, VM_WRITE, and VM_EXEC flags specify the usual read, write, and execute permissions for the pages in this particular memory area. For example, the object code for a process might be mapped with VM_READ and VM_EXEC but not VM_WRITE. On the other hand, the data section from an executable object would be mapped VM_READ and VM_WRITE, but VM_EXEC would make little sense. The VM_SHARED flag specifies whether the memory area contains a mapping that is shared among multiple processes. 

The VM_IO flag specifies that this memory area is a mapping of a device’s I/O space. The VM_RESERVED flag specifies that the memory region must not be swapped out. 

The VM_SEQ_READ flag provides a hint to the kernel that the application is performing sequential (that is, linear and contiguous) reads in this mapping. The kernel can then opt to increase the read-ahead performed on the backing file. The VM_RAND_READ flag specifies the exact opposite: that the application is performing relatively random (that is, not sequential) reads in this mapping. These flags are set via the madvise() system call with the MADV_SEQUENTIAL and MADV_RANDOM flags, respectively. 

### VMA Operations
The vm_ops field in the vm_area_struct structure points to the table of operations associated with a given memory area,which the kernel can invoke to manipulate the VMA.

The operations table is represented by struct vm_operations_struct and is defined in <linux/mm.h>:

```c
struct vm_operations_struct {
    void (*open) (struct vm_area_struct *);
    void (*close) (struct vm_area_struct *);
    int (*fault) (struct vm_area_struct *, struct vm_fault *);
    int (*page_mkwrite) (struct vm_area_struct *vma, struct vm_fault *vmf); 
    int (*access) (struct vm_area_struct *, unsigned long, void *, int, int);
};
```

### Lists and Trees of Memory Areas
The first field, mmap, links together all the memory area objects in a singly linked list. Each vm_area_struct structure is linked into the list via its vm_next field. The areas are sorted by ascending address.

The second field, mm_rb, links together all the memory area objects in a red-black tree. The root of the red-black tree is mm_rb, and each vm_area_struct structure in this address space is linked to the tree via its vm_rb field.

The linked list is used when every node needs to be traversed. The red-black tree is used when locating a specific memory area in the address space.

### Memory Areas in Real Life

```
00e80000 (1212 KB)   r-xp (03:01 208530)   /lib/tls/libc-2.5.1.so
00faf000 (12 KB)     rw-p (03:01 208530)   /lib/tls/libc-2.5.1.so
00fb2000 (8 KB)      rw-p (00:00 0)        
08048000 (4 KB)      r-xp (03:03 439029)   /home/rlove/src/example
08049000 (4 KB)      rw-p (03:03 439029)   /home/rlove/src/example
40000000 (84 KB)     r-xp (03:01 80276)    /lib/ld-2.5.1.so
40015000 (4 KB)      rw-p (03:01 80276)    /lib/ld-2.5.1.so
4001e000 (4 KB)      rw-p (00:00 0)
bfffe000 (8 KB)      rwxp (00:00 0)        [stack]
mapped: 1340 KB      writable/private: 40 KB     shared: 0 KB
```
   
If a memory region is shared or nonwritable, the kernel keeps only one copy of the backing file in memory.

Note the memory areas without a mapped file on device 00:00 and inode zero.This is the zero page, which is a mapping that consists of all zeros. By mapping the zero page over a writable memory area, the area is in effect “initialized” to all zeros.This is impor- tant in that it provides a zeroed memory area, which is expected by the bss. Because the mapping is not shared, as soon as the process writes to this data, a copy is made (à la copy- on-write) and the value updated from zero.

## Manipulating Memory Areas
The kernel often has to perform operations on a memory area, such as whether a given address exists in a given VMA. A handful of helper functions are defined to assist these jobs. These functions are all declared in <linux/mm.h>.

### find_vma()
The kernel provides a function, find_vma(), for searching for the VMA in which a given memory address resides. It is defined in mm/mmap.c:

```c
struct vm_area_struct * find_vma(struct mm_struct *mm, unsigned long addr);
```

This function searches the given address space for the first memory area whose vm_end field is greater than addr. Note that because the returned VMA may start at an address greater than addr, the given address does not necessarily lie inside the returned VMA. The result of the find_vma() function is cached in the mmap_cache field of the memory descriptor.

```c
struct vm_area_struct * find_vma(struct mm_struct *mm, unsigned long addr) {
    struct vm_area_struct *vma = NULL;
    if (mm) {
        vma = mm->mmap_cache;
        if (!(vma && vma->vm_end > addr && vma->vm_start <= addr)) { 
            struct rb_node *rb_node;

            rb_node = mm->mm_rb.rb_node; 
            vma = NULL;
            while (rb_node) {
                struct vm_area_struct * vma_tmp;
                vma_tmp = rb_entry(rb_node, 
                                struct vm_area_struct, vm_rb);
                if (vma_tmp->vm_end > addr) { 
                    vma = vma_tmp;
                    if (vma_tmp->vm_start <= addr) 
                        break;
                    rb_node = rb_node->rb_left; 
                } else
                    rb_node = rb_node->rb_right;
            }
            if (vma)
                mm->mmap_cache = vma;
        }
    }   
    return vma;
}
```

### find_vma_prev()
The find_vma_prev() function works the same as find_vma(), but it also returns the lastVMA before addr. The function is also defined in mm/mmap.c and declared in <linux/mm.h>:

```c
struct vm_area_struct * find_vma_prev(struct mm_struct *mm, unsigned long addr, struct vm_area_struct **pprev)
```

### find_vma_intersection()
The find_vma_intersection() function returns the first VMA that overlaps a given address interval. The function is defined in <linux/mm.h> because it is inline:

```c
static inline struct vm_area_struct * 
find_vma_intersection(struct mm_struct *mm, 
                      unsigned long start_addr, 
                      unsigned long end_addr)
{
    struct vm_area_struct *vma;

    vma = find_vma(mm, start_addr);
    if (vma && end_addr <= vma->vm_start)
        vma = NULL; 
    return vma;
}
```

## mmap() and do_mmap(): Creating an Address Interval
The do_mmap() function is used by the kernel to create a new linear address interval. Saying that this function creates a new VMA is not technically correct, because if the created address interval is adjacent to an existing address interval, and if they share the same permissions, the two intervals are merged into one. If this is not possible, a new VMA is created. 

The do_mmap() function is declared in <linux/mm.h>:

```c
unsigned long do_mmap(struct file *file, unsigned long addr, 
                      unsigned long len, unsigned long prot,
                      unsigned long flag, unsigned long offset)
```

This function maps the file specified by file at offset offset for length len. The file parameter can be NULL and offset can be zero, in which case the mapping will not be backed by a file. In that case, this is called an anonymous mapping. If a file and offset are provided, the mapping is called a file-backed mapping.

The addr function optionally specifies the initial address from which to start the search for a free interval. The prot parameter specifies the access permissions for pages in the memory area.

|    Flag    |   Effect on the Pages in the New Interval    |
|------------|----------------------------------------------|
| PROT_READ  |           Corresponds to VM_READ             |
| PROT_WRITE |           Corresponds to VM_WRITE            |
| PROT_EXEC  |           Corresponds to VM_EXEC             |
| PROT_NONE  |           Cannot access page                 |

If any of the parameters are invalid, do_mmap() returns a negative value. Otherwise, a suitable interval in virtual memory is located. If possible, the interval is merged with an adjacent memory area. Otherwise, a new vm_area_struct structure is allocated from the vm_area_cachep slab cache, and the new memory area is added to the address space’s linked list and red-black tree of memory areas via the vma_link() function. Next, the total_vm field in the memory descriptor is updated. Finally, the function returns the initial address of the newly created address interval.

The do_mmap() functionality is exported to user-space via the mmap() system call.The mmap() system call is defined as

```c
void * mmap2(void *start, size_t length,
             int prot, int flags, 
             int fd, off_t pgoff)
```

## munmap() and do_munmap(): Removing an Address Interval
The do_munmap() function removes an address interval from a specified process address space. The function is declared in <linux/mm.h>:

```c
int do_munmap(struct mm_struct *mm, unsigned long start, size_t len)
```

The munmap()system call is exported to user-space as a means to enable processes to remove address intervals from their address space; it is the complement of the mmap() system call:

```c
int munmap(void *start, size_t length)
```

## Page Tables
Although applications operate on virtual memory mapped to physical addresses, processors operate directly on those physical addresses. Consequently, when an application accesses a virtual memory address, it must first be converted to a physical address before the processor can resolve the request. Performing this lookup is done via page tables. Page tables work by splitting the virtual address into chunks. Each chunk is used as an index into a table.The table points to either another table or the associated physical page.

In Linux, the page tables consist of three levels. The multiple levels enable a sparsely populated address space, even on 64-bit machines. 

The top-level page table is the page global directory (PGD), which consists of an array of pgd_t types. On most architectures, the pgd_t type is an unsigned long.The entries in the PGD point to entries in the second-level directory, the PMD.

The second-level page table is the page middle directory (PMD), which is an array of pmd_t types. The entries in the PMD point to entries in the PTE.

The final level is called simply the page table and consists of page table entries of type pte_t. Page table entries point to physical pages.

In most architectures, page table lookups are handled (at least to some degree) by hardware. 

Each process has its own page tables (threads share them, of course). The pgd field of the memory descriptor points to the process’s page global directory. Manipulating and traversing page tables requires the page_table_lock, which is located inside the associated memory descriptor.

Page table data structures are quite architecture-dependent and thus are defined in <asm/page.h>.

Most processors implement a translation lookaside buffer, or simply TLB, which acts as a hardware cache of virtual-to-physical mappings.

# Chapter16. The Page Cache and Page Writeback
The Linux kernel implements a disk cache called the page cache. The goal of this cache is to minimize disk I/O by storing data in physical memory that would otherwise require disk access.

## Approaches to Caching
We call the storage device being cached the backing store because the disk stands behind the cache as the source of the canonical version of any cached data.

### Write Caching
Generally speaking, caches can implement one of three different strategies:

* In the first strategy, called no-write, the cache simply does not cache write operations. A write operation against a piece of data stored in the cache would be written directly to disk, invalidating the cached data and requiring it to be read from disk again on any subsequent read. 
* In the second strategy, a write operation would automatically update both the in-memory cache and the on-disk file.This approach is called a write-through cache because write operations immediately go through the cache to the disk.
* The third strategy, employed by Linux, is called write-back. In a write-back cache, processes perform write operations directly into the page cache. The backing store is not immediately or directly updated. Instead, the written-to pages in the page cache are marked as dirty and are added to a dirty list. Periodically, pages in the dirty list are written back to disk in a process called writeback, bringing the on-disk copy in line with the in-memory cache.

### Cache Eviction
The final piece to caching is the process by which data is removed from the cache, either to make room for more relevant cache entries or to shrink the cache to make available more RAM for other uses. This process, and the strategy that decides what to remove, is called cache eviction. 

Linux’s cache eviction works by selecting clean (not dirty) pages and simply replacing them with something else. If insufficient clean pages are in the cache, the kernel forces a writeback to make more clean pages available.

#### Least Recently Used
One of the more successful algorithms, particularly for general-purpose page caches, is called least recently used, or LRU. An LRU eviction strategy requires keeping track of when each page is accessed (or at least sorting a list of pages by access time) and evicting the pages with the oldest timestamp. However, one particular failure of the LRU strategy is that many files are accessed once and then never again. Putting them at the top of the LRU list is thus not optimal. 

#### The Two-List Strategy
Linux, therefore, implements a modified version of LRU, called the two-list strategy. Instead of maintaining one list, the LRU list, Linux keeps two lists: the active list and the inactive list. Pages are placed on the active list only when they are accessed while already residing on the inactive list. Both lists are maintained in a pseudo-LRU manner: Items are added to the tail and removed from the head, as with a queue. The lists are kept in balance: If the active list becomes much larger than the inactive list, items from the active list’s head are moved back to the inactive list, making them available for eviction.

## The Linux Page Cache

### The address_space Object
A page in the page cache can consist of multiple noncontiguous physical disk blocks. Checking the page cache to see whether certain data has been cached is made difficult because of this noncontiguous layout of the blocks that constitute each page.

To maintain a generic page cache—one not tied to physical files or the inode structure—the Linux page cache uses a new object to manage entries in the cache and page I/O operations. That object is the address_space structure. While a single file may be represented by 10 vm_area_struct structures (if, say, five processes each mmap() it twice), the file has only one address_space structure—just as the file may have many virtual addresses but exist only once in physical memory. 

The address_space structure is defined in <linux/fs.h>:

```c
struct address_space { 
    struct inode *host;  /* owning inode */
    struct radix_tree_root page_tree;  /* radix tree of all pages */
    spinlock_t tree_lock;  /* page_tree lock */
    unsigned int i_mmap_writable;  /* VM_SHARED ma count */
    struct prio_tree_root i_mmap;  /* list of all mappings */
    struct list_head i_mmap_nonlinear;  /* VM_NONLINEAR ma list */
    spinlock_t i_mmap_lock;  /* i_mmap lock */
    atomic_t truncate_count;  /* truncate re count */
    unsigned long nrpages;  /* total number of pages */
    pgoff_t writeback_index;  /* writeback start offset */
    struct address_space_operations *a_ops;  /* operations table */
    unsigned long flags;  /* gfp_mask and error flags */
    struct backing_dev_info *backing_dev_info;  /* read-ahead information */
    spinlock_t private_lock;  /* private lock */
    struct list_head private_list;  /* private list */
    struct address_space *assoc_mapping;  /* associated buffers */
};
```

The i_mmap field is a priority search tree of all shared and private mappings in this address space. A priority search tree is a clever mix of heaps and radix trees. Recall from earlier that while a cached file is associated with one address_space structure, it can have many vm_area_struct structures—a one-to-many mapping from the physical pages to many virtual pages. The i_mmap field allows the kernel to efficiently find the mappings associated with this cached file.
 
There are a total of nrpages in the address space. The address_space is associated with some kernel object. Normally, this is an inode. If so, the host field points to the associated inode. The host field is NULL if the associated object is not an inode—for example, if the address_space is associated with the swapper.

### address_space Operations
The a_ops field points to the address space operations table, in the same manner as the VFS objects and their operations tables. The operations table is represented by struct address_space_operations and is also defined in <linux/fs.h>. These function pointers point at the functions that implement page I/O for this cached object. Each backing store describes how it interacts with the page cache via its own address_space_operations.

Let’s look at the steps involved in each, starting with a page read operation. First, the Linux kernel attempts to find the request data in the page cache. The find_get_page() method is used to perform this check; it is passed an address_space and page offset. These values search the page cache for the desired data:

```c
page = find_get_page(mapping, index);
```

Here, mapping is the given address_space and index is the desired offset into the file, in pages. If the page does not exist in the cache, find_get_page()returns NULL and a new page is allocated and added to the page cache:

```c
struct page *page; 
int error;
/* allocate the page ... */
page = page_cache_alloc_cold(mapping); 
if (!page)
    /* error allocating memory */

/* ... and then add it to the page cache */
error = add_to_page_cache_lru(page, mapping, index, GFP_KERNEL); 
if (error)
    /* error adding page to page cache */
```

Finally, the requested data can be read from disk, added to the page cache, and returned to the user:

```c
error = mapping->a_ops->readpage(file, page);
```

Write operations are a bit different. For file mappings, whenever a page is modified, the VM simply calls

```c
SetPageDirty(page);
```

The kernel later writes the page out via the writepage() method. Write operations on specific files are more complicated.The generic write path in mm/filemap.c performs the following steps:

```c
page = __grab_cache_page(mapping, index, &cached_page, &lru_pvec); 
status = a_ops->prepare_write(file, page, offset, offset+bytes); 
page_fault = filemap_copy_from_user(page, offset, buf, bytes); 
status = a_ops->commit_write(file, page, offset, offset+bytes);
```

First, the page cache is searched for the desired page. If it is not in the cache, an entry is allocated and added. Next, the kernel sets up the write request and the data is copied from user-space into a kernel buffer. Finally, the data is written to disk.


### Radix Tree
As you saw in the previous section, the page cache is searched via the address_space object plus an offset value. Each address_space has a unique radix tree stored as page_tree. A radix tree is a type of binary tree. The radix tree enables quick searching for the desired page, given only the file offset. Page cache searching functions such as find_get_page() call radix_tree_lookup(), which performs a search on the given tree for the given object.
The core radix tree code is available in generic form in lib/radix-tree.c. Users of the radix tree need to include <linux/radix-tree.h>.

## The Buffer Cache
Individual disk blocks also tie into the page cache, by way of block I/O buffers. This caching is often referred to as the buffer cache, although as implemented it is not a separate cache but is part of the page cache.

Block I/O operations manipulate a single disk block at a time. A common block I/O operation is reading and writing inodes. The kernel provides the bread() function to perform a low-level read of a single block from disk. Via buffers, disk blocks are mapped to their associated in-memory pages and cached in the page cache. Conveniently, the buffers describe the mapping of a block onto a page, which is in the page cache.

## The Flusher Threads
When data in the page cache is newer than the data on the backing store, we call that data dirty. Dirty page writeback occurs in three situations:

* When free memory shrinks below a specified threshold, the kernel writes dirty data back to disk to free memory because only clean (nondirty) memory is available for eviction. When clean, the kernel can evict the data from the cache and then shrink the cache, freeing up more memory.
* When dirty data grows older than a specific threshold, sufficiently old data is written back to disk to ensure that dirty data does not remain dirty indefinitely.
* When a user process invokes the sync() and fsync() system calls, the kernel performs writeback on demand.

In 2.6, a gang of kernel threads, the flusher threads, performs all three jobs.

First, the flusher threads need to flush dirty data back to disk when the amount of free memory in the system shrinks below a specified level. The goal of this background write-back is to regain memory consumed by dirty pages when available physical memory is low. The memory level at which this process begins is configured by the dirty_background_ratio sysctl. When free memory drops below this threshold, the kernel invokes the wakeup_flusher_threads() call to wake up one or more flusher threads and have them run the bdi_writeback_all() function to begin writeback of dirty pages. The function continues writing out data until two conditions are true:

* The specified minimum number of pages has been written out.
* The amount of free memory is above the dirty_background_ratio threshold.

For its second goal, a flusher thread periodically wakes up (unrelated to low-memory conditions) and writes out old dirty pages. On system boot, a timer is initialized to wake up a flusher thread and have it run the wb_writeback() function. This function then writes back all data that was modified longer than dirty_expire_interval milliseconds ago.

The flusher code lives in mm/page-writeback.c and mm/backing-dev.c and the writeback mechanism lives in fs/fs-writeback.c.

### Laptop Mode
Laptop mode is a special page writeback strategy intended to optimize battery life by minimizing hard disk activity and enabling hard drives to remain spun down as long as possible.

Laptop mode makes a single change to page writeback behavior. In addition to performing writeback of dirty pages when they grow too old, the flusher threads also piggy-back off any other physical disk I/O, flushing all dirty buffers to disk. 

### History: bdflush, kupdated, and pdflush
Prior to the 2.6 kernel, the job of the flusher threads was met by two other kernel threads, bdflush and kupdated.

The bdflush kernel thread performed background writeback of dirty pages when available memory was low.

Two main differences distinguish bdflush and the current flusher threads:

* The first, which is discussed in the next section, is that there was always only one bdflush daemon, whereas the number of flusher threads is a function of the number of disk spindles.
* The second difference is that bdflush was buffer-based; it wrote back dirty buffers. Conversely, the flusher threads are page-based; they write back whole pages. 

 The kupdated thread was introduced to periodically write back dirty pages. It served an identical purpose to the wb_writeback() function.

 In the 2.6 kernel, bdflush and kupdated gave way to the pdflush threads. The pdflush threads performed similar to the flusher threads of today. The main difference is that the number of pdflush threads is dynamic, by default between two and eight, depending on the I/O load of the system. The downside is that pdflush can easily trip up on congested disks, and congestion is easy to cause with modern hardware. Moving to per-spindle flushing enables the I/O to perform synchronously, simplifying the congestion logic and improving performance.

 ### Avoiding Congestion with Multiple Threads
 One of the major flaws in the bdflush solution was that bdflush consisted of one thread. This led to possible congestion during heavy page writeback where the single bdflush thread would block on a single congested device queue, whereas other device queues would sit relatively idle.

 The 2.6 kernel solves this problem by enabling multiple flusher threads to exist. Each thread individually flushes dirty pages to disk, allowing different flusher threads to concentrate on different device queues. With the current flusher threads model, available since 2.6.32, the threads are associated with a block device, so each thread grabs data from its per-block device dirty list and writes it back to its disk.

 # Chapter17. Devices and Modules
In this chapter, we discuss four kernel components related to device drivers and device management:

* Device types—Classifications used in all Unix systems to unify behavior of common devices
* Modules—The mechanism by which the Linux kernel can load and unload object code on demand
* Kernel objects—Support for adding simple object-oriented behavior and a parent/child relationship to kernel data structures
* Sysfs—A filesystem representation of the system’s device tree

## Device Types
In Linux, as with all Unix systems, devices are classified into one of three types:

* Block devices
* Character devices 
* Network devices

Often abbreviated blkdevs, block devices are addressable in device-specified chunks called blocks and generally support seeking, the random access of data. Block devices are accessed via a special file called a block device node and generally mounted as a filesystem.

Often abbreviated cdevs, character devices are generally not addressable providing access to data only as a stream, generally of characters (bytes). Character devices are accessed via a special file called a character device node. Unlike with block devices, applications interact with character devices directly through their device node.

Sometimes called Ethernet devices after the most common type of network devices, network devices provide access to a network (such as the Internet) via a physical adapter (such as your laptop’s 802.11 card) and a specific protocol (such as IP). Breaking Unix’s “everything is a file” design principle, network devices are not accessed via a device node but with a special interface called the socket API.

Not all device drivers represent physical devices. Some device drivers are virtual, providing access to kernel functionality. We call these pseudo devices; some of the most common are the kernel random number generator(accessible at /dev/random and /dev/urandom), the null device (accessible at /dev/null), the zero device (accessible at /dev/zero), the full device (accessible at /dev/full), and the memory device (accessible at /dev/mem). 


# Chapter19. Portability

## Portable Operating Systems
Linux takes the middle road toward portability. As much as practical, interfaces and core code are architecture-independent C code. Where performance is critical, however, kernel features are tuned for each architecture. 

Generally, exported kernel interfaces are architecture-independent. If any parts of the function need to be unique for each supported architecture (either for performance reasons or as a necessity), that code is implemented in separate functions and called as needed.  

## Word Size and Data Types
A word is the amount of data that a machine can process at one time. A word is an integer number of bytes—for example, one, two, four, or eight. When someone talks about the “n-bits” of a machine, they are generally talking about the machine’s word size.

The size of a processor’s general-purpose registers (GPRs) is equal to its word size. The widths of the components in a given architecture—for example, the memory bus—are usually at least as wide as the word size. Typically, at least in the architectures that Linux supports, the virtual memory address space is equal to the word size, although the physical address space is sometimes less. Consequently, the size of a pointer is equal to the word size. Additionally, the size of the C type long is equal to the word size.

Each supported architecture under Linux defines BITS_PER_LONG in <asm/types.h> to the length of the C long type, which is the system word size. 

Some rules to keep in mind:

* As dictated by the ANSI C standard, a char is always 1 byte.
* Although there is no rule that the int type be 32 bits, it is in Linux on all currently supported architectures.
* The same goes for the short type, which is 16 bits on all current architectures, although no rule explicitly decrees that.
* Never assume the size of a pointer or a long, which can be either 32 bits or 64 bits on the currently supported machines in Linux.
* Because the size of a long varies on different architectures, never assume that sizeof(int) is equal to sizeof(long).
* Likewise, do not assume that a pointer and an int are the same size.

### Opaque Types
Opaque data types do not reveal their internal format or structure. Developers declare a typedef, call it an opaque type, and hope no one typecasts it back to a standard C type. An example is the pid_t type, which stores a process identification number. If no code makes explicit use of this type’s size, it can be changed without too much hassle.

Another example of an opaque type is atomic_t. Although this type is an int, using the opaque type helps ensure that the data is used only in the special atomic operation functions.

Keep the following rules in mind when dealing with opaque types:

* Do not assume the size of the type. It might be 32-bit on some systems and 64-bit on others. Moreover, kernel developers are free to change its size over time.
* Do not convert the type back to a standard C type.
* Be size agnostic.Write your code so that the actual storage and format of the type can change.

### Special Types
Some data in the kernel, despite not being represented by an opaque type, requires a specific data type. When storing and manipulating specific data, always pay careful attention to the data type that represents the type and use it. It is a common mistake to store one of these values in another type, such as unsigned int. 

### Explicitly Sized Types
As a programmer, you need explicitly sized data in your code.This is usually to match external requirements, such as those imposed by hardware, networking, or binary files. For example, a sound card might have a 32-bit register, a networking packet might have a 16-bit field, or an executable file might have an 8-bit cookie. In these cases, the data type that represents the data needs to be exactly the right size.

The kernel defines these explicitly sized data types in <asm/types.h>, which is included by <linux/types.h>:

| Type |        Description       |
|------|--------------------------|
|  s8  |        Signed byte       |
|  u8  |        Unsigned byte     |
|  s16 |   Signed 16-bit integer  |
|  u16 |  Unsigned 16-bit integer |
|  s32 |   Signed 32-bit integer  |
|  u32 |  Unsigned 32-bit integer |
|  s64 |   Signed 64-bit integer  |
|  u64 |  Unsigned 64-bit integer |

These explicit types are merely typedefs to standard C types. On a 64-bit machine, they may look like this:

```c
typedef signed char s8; 
typedef unsigned char u8; 
typedef signed short s16; 
typedef unsigned short u16; 
typedef signed int s32; 
typedef unsigned int u32; 
typedef signed long s64; 
typedef unsigned long u64;
```

These types can be used only inside the kernel, in code that is never revealed to user-space. This is for reasons of namespace. The kernel also defines user-visible variants of these types, which are simply the same type prefixed by two underscores. 

### Signedness of Chars
On most architectures, char is signed by default and thus has a range from –128 to 127. On a few other architectures, such as ARM, char is unsigned by default and has a range from 0 to 255.

If you use char in your code, assume it can be either a signed char or an unsigned char. If you need it to be explicitly one or the other, declare it as such.

## Data Alignment
Alignment refers to a piece of data’s location in memory. A variable is naturally aligned if it exists at a memory address that is a multiple of its size. Thus, a data type with size 2^n bytes must have an address with the n least significant bits set to zero. When writing portable code, alignment issues must be avoided, and all types should be naturally aligned.

### Avoiding Alignment Issues
The compiler generally prevents alignment issues by naturally aligning all data types. In fact, alignment issues are normally not major concerns of the kernel developers—the gcc developers worry about them so other programmers need not. 

Accessing an aligned address with a recast pointer of a larger-aligned address causes an alignment issue:

```c
char wolf[] = “Like a wolf”;
char *p = &wolf[1];
unsigned long l = *(unsigned long *)p;
```

This example treats the pointer to a char as a pointer to an unsigned long, which might result in the 32- or 64-bit unsigned long value being loaded from an address that is not a multiple of 4 or 8, respectively.

### Alignment of Nonstandard Types
Nonstandard (complex) C types have the following alignment rules:

* The alignment of an array is the alignment of the base type; thus, each element is further aligned correctly.
* The alignment of a union is the alignment of the largest included type.
* The alignment of a structure is such that an array of the structure will have each element of the array properly aligned.

### Structure Padding
Structures are padded so that each element of the structure is naturally aligned. This ensures that when the processor accesses a given element in the structure, that element is aligned. For example, consider this structure on a 32-bit machine:

```c
struct animal_struct {
    char dog; /* 1 byte */ 
    unsigned long cat; /* 4 bytes */
    unsigned short pig; /* 2 bytes */
    char fox; /* 1 byte */
};
```

The structure is not laid out exactly like this in memory because the natural alignment of the structure’s members is insufficient. Instead, the compiler creates the structure such that in memory, the struct resembles the following:

```c
struct animal_struct { 
    char dog;           /* 1 byte */
    u8 __pad0[3];       /* 3 bytes */
    unsigned long cat;  /* 4 bytes */   
    unsigned short pig; /* 2 bytes */
    char fox;           /* 1 byte */
    u8 __pad1;          /* 1 byte */
};
```

The first padding pro- vides a 3-byte waste-of-space to place cat on a 4-byte boundary. The extra byte ensures the structure is a multiple of 4, and thus each member of an array of this structure is naturally aligned. Note that sizeof(animal_struct) returns 12 for either of these structures on most 32- bit machines. The C compiler automatically adds this padding to ensure proper alignment.

You can often rearrange the order of members in a structure to obviate the need for padding. This gives you properly aligned data without the need for padding and therefore a smaller structure:

```c
struct animal_struct { 
    unsigned long cat;   /* 4 bytes */   
    unsigned short pig;  /* 2 bytes */
    char dog;            /* 1 byte */
    char fox;            /* 1 byte */
};
```

## Byte Order
Byte ordering is the order of bytes within a word. Processors can number the bytes in a word such that the least significant bit is either the first (left-most) or last (right-most) value in the word. The byte ordering is called big-endian if the most significant byte is encoded first, with the remaining bytes decreasing in significance. The byte ordering is called little-endian if the least significant byte is encoded first, with the remaining bytes growing in significance.

Here is a simple code snippet to test whether a given architecture is big- or little-endian:

```c
int x = 1;
if (*(char *)&x == 1) 
    /* little endian */
else
    /* big endian */
```

Each supported architecture in Linux defines one of __BIG_ENDIAN or __LITTLE_ENDIAN in <asm/byteorder.h> in correspondence to the machine’s byte order.

This header file also includes a family of macros from include/linux/byteorder/, which help with conversions to and from the various orderings. The most commonly needed macros are

```c
u23 __cpu_to_be32(u32);  /* convert cpu’s byte order to big-endian */ 
u32 __cpu_to_le32(u32);  /* convert cpu’s byte order to little-endian */
u32 __be32_to_cpu(u32);  /* convert big-endian to cpu’s byte order */
u32 __le32_to_cpus(u32); /* convert little-endian to cpu’s byte order */
```

## Time
The measurement of time is another kernel concept that can differ between architectures or even kernel revisions. Never assume the frequency of the timer interrupt or the number of jiffies per second. Instead, always use HZ to scale your units of time correctly.
  
## Page Size
When working with pages of memory, never assume the page size. When working with pages of memory, use PAGE_SIZE as the size of a page, in bytes. The value PAGE_SHIFT is the number of bits to left-shift an address to derive its page number. 

## Processor Ordering
In your code, if you depend on data ordering, ensure that even the weakest ordered processor commits your load and stores in the right order by using the appropriate barriers, such as rmb() and wmb(). 

## SMP, Kernel Preemption, and High Memory
In addition to the previous portability rules, you need to follow these as well:

* Always assume your code will run on an SMP system and use appropriate locking. 
* Always assume your code will run with kernel preemption enabled and use appropriate locking and kernel preemption statements.
* Always assume your code will run on a system with high memory (memory not permanently mapped) and use kmap() as needed.