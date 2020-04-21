# xv6-book Chapter 1 - Operating system interfaces

1. **To-dos** of OS:
    * manage and abstract the low-level hardware
    * share the hardware among multiple programs so that they run (or appear to run) at thr same time
    * provide controlled ways for programs to interact

2. **Interface**: An operating system provides services to user programs **through an interface**. Unix provides a narrow interface whose mechanisms combine well, offering a surprising degree of generality.

3. **Some definitions**:
    * **Kernel**: a special program that provides services to running programs
    * **Process**: Each running program, called a process, has memory containing **instructions, data, and a stack**.
    * **System Call**: The system call enters the kernel; the kernel performs the service and returns. Thus a process alternates between executing in **user space** and **kernel space**.
    * **Shell**: An ordinary program (which is **a user program**) that reads commands from the user and executes them.

4. **Hardware protection mechanisms**: 
    * Ensure that each process executing in **user space** can access only its **own memory**.
    * The kernel executes with **the hardware privileges** required to implement these protections; user programs execute **without those privileges**.

## 1.1 Processes and memory

1. **Components** of an xv6 process:
    * User-space memory (instructions, data, and stack)
    * Per-process state private to the kernel (ready, running, waiting, or whatever)

2. Xv6 can **time-share** processes: it **transparently** switches the available CPUs among the set of processes waiting to execute. When a process is not executing, xv6 saves its **CPU registers**, restoring them when it next runs the process.

3. The kernel associates **a process identifier**, or `pid`, with each process.

### 1.1.1 The `fork` system call

1. **Functions**: Create a new process, called **the child process**, with **exactly the same memory contents** as the calling process, called **the parent process**.

2. **Params**: None

3. **Returns**: 
    * In the parent process: **the child's pid**
    * In the child process: `0`


## References

1. xv6: a simple, Unix-like teaching operating system

2. https://www.tutorialspoint.com/operating_system/os_processes.htm