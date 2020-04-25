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
    * The call returns in **both the parent process and the child proress**
    * In the parent process: **the child's pid**
    * In the child process: `0`
    * So we can **check the return value** to do different instructions in different processes

4. The parent and child are executing with **different memory and different registers**: changing a variable in one does not affect the other.

### 1.1.2 The `exit` system call

1. **Functions**: 
    * Cause the calling process to **stop executing**
    * Release resources such as memory and open files

2. **Params**: An integer status argument:
    * `0` -> success
    * `1` -> failure

3. **Returns**: None

### 1.1.3 The `wait` system call

1. **Functions**: Wait for a child of the process to done

2. **Params**: The address which gets the **exit status of the child** (pass a `0` address if not care about the exit status of the child)

3. **Returns**: The `pid` of an existed child of the current process

### 1.1.4 The `exec` system call

1. **Functions**: 
    * Replace the calling process's memory with a new memory image loaded from a file stored in the file system. 

2. **Params**: 
    * 1: The name of the file containing the executable
    * 2: An array of string arguments (*Most programs ignore the first argument, which is conventionally the name of the program.*)

3. The file must have a particular format (xv6 uses the **ELF format**), which specifies:
    * which part of the file holds **instructions**
    * which part is **data**
    * at which instruction to **start**

4. **Returns**: When exec succeeds, it **does not return to the calling program**; instead, the instructions loaded from the file **start executing at the entry point** declared in the ELF header. 

### 1.1.5 A simple shell

```c
...

void
panic(char *s)
{
    fprintf(2, "%s\n", s);
    exit(1);
}

int
fork1(void)
{
    int pid;

    pid = fork();
    if(pid == -1)
        panic("fork");
    return pid;
}

void
runcmd(struct cmd *cmd)
{
    int p[2];
    ...

    if(cmd == 0)
        exit(1);

    switch(cmd->type){
    default:
        panic("runcmd");

    case EXEC:
        ecmd = (struct execcmd*)cmd;
        if(ecmd->argv[0] == 0)
            exit(1);
        exec(ecmd->argv[0], ecmd->argv);
        fprintf(2, "exec %s failed\n", ecmd->argv[0]);
        break;

    ...

    }
    exit(0);
}

int
main(void)
{
    static char buf[100];
    int fd;

    // Ensure that three file descriptors are open.
    while((fd = open("console", O_RDWR)) >= 0){
        if(fd >= 3){
            close(fd);
            break;
        }
    }

    // Read and run input commands.
    while(getcmd(buf, sizeof(buf)) >= 0){
        // Call exec only in the child process
        if(fork1() == 0)
            runcmd(parsecmd(buf));
        // Wait for the child process to be done
        wait(0);
    }
    exit(0);
}

...

```

### 1.1.6 The `sbrk` system call

1. **Functions**: A process that **needs more memory at run-time** (perhaps for `malloc`) could grow its data memory by `n` (the only param) bytes.

2. **Params**: An integer `n`.

3. **Returns**: The location of the new memory.

### 1.1.7 Others

1. Xv6 does not provide a notion of users or of protecting one user from another; in Unix terms, **all xv6 processes** run as `root`.

## 1.2 I/O and File descriptors

### 1.2.1 File descriptor

1. File descriptor:
    * A small integer
    * Represent a kernel-managed object that a process may **read from** or **write to**

2. Ways to create a `fd`:
    * Open a file, directory or device
    * Create a pipe
    * Duplicate an existing `fd`

3. Features of `fd`:
    * **Every** process has **a private space** of file descriptors
    * Start from `0`

4. Default `fd`s:
    * `0`: standard input
    * `1`: standard output
    * `2`: standard error

### 1.2.2 The `read` and `write` system call

1. **Functions**: 
    * In general: Read bytes from and write bytes to open files named **by file descriptors**
    * `read`: 
        * **Fewer than** `0` bytes are read only when **an error** occurs
        * Read data from **the current file offset**
        * Advance that offset by **the number of bytes read**
        * when there are no more bytes to read, read returns `0` to **indicate the end of the file**
    * `write`: 
        * **Fewer than** `n` bytes are written only when **an error** occurs
        * Write data at **the current file offset**
        * Advance that offset by **the number of bytes written**

2. **Params**:
    * `read(fd, buf, n)` reads **at most** `n` bytes from the file descriptor `fd`, copies them into `buf`
    * `write(fd, buf, n)` writes `n bytes` from `buf` to the file descriptor `fd`

3. **Returns**: 
    * `read`: the number of bytes read (`< 0` if **an error occurs**)
    * `write`: the number of bytes written (`< n` if **an error occurs**)

4. Others: 
    * Each file descriptor that refers to a file has **an offset** associated with it. 

### 1.2.3 The `close` system call

1. **Functions**: Release a file descriptor, making it free for reuse by a future `open`, `pipe`, or `dup` system call.

2. **Params**: An integer, indicating the `fd`.

### 1.2.4 The `dup` system call

1. **Functions**: 
    * Duplicate an existing file descriptor `fd`
    * Return a new one that refers to **the same underlying I/O object**

2. **Params**: `dup(fd)`, in which `fd` is the existing file descriptor

3. **Returns**: A new file descriptor

### 1.2.5 Details about the file descriptor

1. A newly allocated file descriptor is always the **lowest-numbered unused** descriptor of the current process.

2. `fork` **copies the parent’s file descriptor table** along with its memory.

3. `exec` replaces the calling process’s memory but **preserves its file table**.

4. Why it is a good idea that fork and exec are **separate calls**:  Because if they are separate, the shell can `fork` a child, use `open`, `close`, `dup` in the child to **change the standard input and output file descriptors**, and then `exec`. 

5. When two file descriptors **share an offset** (The two conditions below must be both satisfied): 
    * They were derived from **the same original file descriptor**
    * By a sequence of `fork` and `dup` calls (These two calls **ONLY**)

6. Dup allows shells to implement commands like this:
```shell
ls existing-file non-existing-file > tmp1 2>&1
```
The `2>&1` tells the shell to give the command a file descriptor `2` that is **a duplicate of descriptor `1`**. Both the name of the existing file and the error message for the non-existing file will show up in the file `tmp1`.

## 1.3 Pipes

1. **Definition**: A pipe is a small kernel **buffer** exposed to processes as **a pair of file descriptors**, one for reading and one for writing. 

2. **Features**:
    * Writing data to one end of the pipe makes that data avaliable for reading from the other end of the pipe
    * If no data is available, a `read` on a pipe waits for one of the following conditions to happen:
        * Some data is written to the pipe
        * All file descriptors referring to the write end are all closed (In this case, `read` will return `0`)

3. **Usage**:
```c
int p[2];
char *argv[2];

argv[0] = "wc";
argv[1] = 0;

pipe(p);
if (fork() == 0) {
    close(0);
    dup(p[0]);
    close(p[0]);
    close(p[1]);
    exec("/bin/wc", argv);
} else {
    close(p[0]);
    write(p[1], "hello world\en", 12);
    close(p[1]);
}
```
* Write and read ends:
    * `p[0]` refers to the read end
    * `p[1]` refers to the write end

4. **Example in a shell**:
```sh
echo hello world | wc
```

5. **Advantages**: 
    * Automatically clean themselves up
    * Can pass arbitrarily long streams of data
    * Allow for parallel execution of pipeline stages
    * Can block reads and writes more efficiently

## References

1. xv6: a simple, Unix-like teaching operating system

2. https://www.tutorialspoint.com/operating_system/os_processes.htm