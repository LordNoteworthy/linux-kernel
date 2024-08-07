# Chapter 3. Process Management

## The process

- a process is s is a program (object code stored on some media) in the midst of execution
- include a set of resources such as:
    - open files and pending signals..
    - internal kernel data, processor state, a memory address space with one or more memory mappings,
    - one or more threads of execution, and a data section containing global variables.
-  threads, are the objects of activity within the process.
- each thread includes a unique program counter, process stack, and set of processor registers.
- Another name for a process is a task. The Linux kernel internally refers to processes as __tasks__.

## Process Descriptor and the Task Structure

- The kernel stores the list of processes in a circular doubly linked list called the task list.
- Each element in the task list is a process descriptor of the type struct `task_struct`.

### Allocating the Process Descriptor

- The task_struct structure is allocated via the slab allocator to provide object reuse and cache coloring.

<p align="center"><img src="https://i.imgur.com/mQZ3v4q.png" width="450px" height="auto"></p>

- Each task’s `thread_info` structure is allocated at the end of its stack.- The task element of the structure is a pointer to the task’s actual `task_struct`.

### Storing the Process Descriptor

- x86 (which has few registers to waste), make use of the fact that struct `thread_info` is stored on the kernel stack to calculate the location of `thread_info` and subsequently the `task_struct`.

### Process state

- The state field of the process descriptor describes the current condition of the process:
    - __TASK_RUNNING__: either running or in run queue  waiting to run. This is the only possible state for a process executing in user-space.
    - __TASK_INTERRUPTIBLE__: blocked, waiting for some condition to exist.
    - __TASK_UNINTERRUPTIBLE__ : identical to TASK_INTERRUPTIBLE except
    that it does not wake up and become runnable if it receives a signal.
    - __TASK_TRACED__: process is being traced by another process, such as a debugger, via ptrace.
    - __TASK_STOPPED__: execution has stopped; the task is not running nor is it eligible to run. This occurs if the task receives the `SIGSTOP`, `SIGTSTP`, `SIGTTIN`, or `SIGTTOU` signal or if it receives any signal while it is being debugged.

<p align="center"><img src="https://i.imgur.com/O4S7S7r.png" width="450px" height="auto"></p>

### Manipulating the Current Process State

- Kernel code often needs to change a process’s state.The preferred mechanism is using:
`set_task_state(task, state); /* set task ‘task’ to state ‘state’ */`

### Process Context

- When a program executes a system call or triggers an exception, it enters kernel-space. At this point, the kernel is said to be _executing on behalf of the process_ and is in __process context__.

### The Process Family Tree

- All processes are descendants of the init process, whose PID is one.
- The init process, in turn, reads the system `initscripts` and executes more programs, eventually completing the boot process.
- Processes that are all direct children of the same parent are called __siblings__
- The relationship between processes is stored in the process descriptor. Each `task_struct` has a pointer to the parent’s task_struct, named `parent`, and a list of children, named `children`.

## Process Creation

- Unix takes the unusual approach of process creation into two distinct functions: `fork(` )and `exec()`
- `fork` starts a new process which is a copy of the one that calls it.
-  while `exec` loads a new executable into the address space and begins executing it.
- The combination of `fork()` followed by `exec()` is similar to the single function most operating systems provide.

### Copy-on-Write

- In Linux, `fork()` is implemented through the use of copy-on-write pages.
- In the common case that a process executes a new executable image immediately after forking, this optimization prevents the wasted copying of large amounts of data (with the address space, easily tens of megabytes).
- This is an important optimization because the Unix philosophy encourages quick process execution.

### Forking

- Linux implements `fork()` via the `clone()` system call.
- The bulk of the work in forking is handled by `do_fork()`, which is defined in `kernel/fork.c`. This function calls `copy_process()` and then starts the process running.
- The interesting work is done by `copy_process()`:
    - calls `dup_task_struct()` which creates a new kernel stack, `thread_info` structure, and `task_struct` for the new process.
    - checks that the new child will not exceed the resource limits on the number of processes for the current user.
    - Various members of the process descriptor are cleared or set to initial values.
    - child’s state is set to __TASK_UNINTERRUPTIBLE__
    - calls `copy_flags()` to update the flags member of the `task_struct`: clear `PF_SUPERPRIV` and set `PF_FORKNOEXEC`.
    - `alloc_pid()` to assign an available PID to the new task.
    - Depending on the flags passed to clone(), copy_process() either duplicates or shares open files, filesystem information, signal handlers, process address space, and namespace.
    - `copy_process()` cleans up and returns to the caller a pointer to the new child.

### vfork()

-  has the same effect as `fork()`, except that the page table entries
of the parent process are not copied.
- parent is blocked until the child either calls `exec()` or exits.

## The Linux Implementation of Threads

- threads enable concurrent programming and, on multiple processor systems, true parallelism.
- linux has a unique implementation of threads: It does not differentiate between threads and processes. To Linux, a thread is just a special kind of process.
- linux threads are simply a manner of sharing resources between processes(which are already quite lightweight).

### Creating Threads

- Threads are created the same as normal tasks, with the exception that the `clone()` is passed flags corresponding to the specific resources to be shared: `clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND, 0);`
- In contrast, a normal fork() can be implemented as: `clone(SIGCHLD, 0);`

### Kernel Threads

- the significant difference between kernel threads and normal processes is that kernel threads do not have an address space.
- they operate only in kernel-space and do not context switch into user-space.
- are schedulable and preemptable, the same as normal processes.
- an be created only by another kernel thread. the kernel handles this automatically by forking all new kernel threads off of the __kthreadd__ kernel process. Look at `kthread_create()` to spawn a new ker thread.
- The process is created in an unrunnable state; it will not start running
until explicitly woken up via `wake_up_process()`.

## Process Termination

- occurs when the process calls the `exit()` explicitly or returned form `main()`.
- involuntarily when it receives a signal or exception it cannot handle or ignore.
- Regardless of how a process terminates, the bulk of the work is handled by `do_exit()`:
    - sets the `PF_EXITING` flag in the flags member of the `task_struct`.
    - calls `del_timer_sync()` to remove any kernel timers.
    - If BSD process accounting is enabled, do_exit() calls `acct_update_integrals()` to write out accounting information.
    - It calls `exit_mm()` to release the `mm_struct` held by this process.
    - calls `exit_sem()`. If the process is queued waiting for an IPC semaphore, it is dequeued here.
    - calls `exit_files()` and `exit_fs()` to decrement the usage count of objects related to file descriptors and filesystem data, respectively.
    - sets the task’s exit code, stored in the `exit_code` member of the `task_struct`, to the code provided by `exit()` or whatever kernel mechanism forced the termination.
    - calls `exit_notify()` to send signals to the task’s parent, reparents any of the task’s children to another thread in their thread group or the init process, and sets the task’s exit state, stored in `exit_state` in the `task_struct` structure, to __EXIT_ZOMBIE__.
    - calls `schedule()` to switch to a new process.

### Removing the Process Descriptor

- After `do_exit()` completes, the process descriptor for the terminated process still exists, but the process is a zombie and is unable to run.
- This enables the system to obtain information about a child process after it has terminated.
- After the parent has obtained information on its terminated child, or signified to the kernel that it does not care, the child’s `task_struct` is deallocated.
- when it is time to finally deallocate the process descriptor, `release_task()` is invoked. It does the following:
    - calls `__exit_signal()`, which calls `__unhash_process()`, which in turns calls `detach_pid()` to remove the process from the __pidhash__ and remove the process from the task list.
    - `__exit_signal()` releases any remaining resources used by the now dead process and finalizes statistics and bookkeeping.
    - If the task was the last member of a thread group, and the leader is a zombie, then `release_task()` notifies the zombie leader’s parent.
    - `release_task()` calls `put_task_struct()` to free the pages containing the process’s kernel stack and `thread_info` structure and deallocate the slab cache containing the `task_struct`.
- `wait()` blocks the calling process until one of its child processes exits or a signal is received. After child process terminates, parent continues its execution after wait system call instruction.

### The Dilemma of the Parentless Task

- If a parent exits before its children, some mechanism must exist to reparent any child tasks to a new process.
- `do_exit()` calls `exit_notify()`, which calls `forget_original_parent()`, which, in turn, calls `find_new_reaper()` to perform the __reparenting__.
