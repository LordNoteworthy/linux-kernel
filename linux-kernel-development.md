
# Linux Kernel Developement 3rd edition by Love.R

## 3. Process Management

### The process

- a process is s is a program (object code stored on some media) in the midst of execution
- include a set of resources such as:
    - open files and pending signals..
    - internal kernel data, processor state, a memory address space with one or more memory mappings,
    - one or more threads of execution, and a data section containing global variables.
-  threads, are the objects of activity within the process. 
- each thread includes a unique program counter, process stack, and set of processor registers.
- Another name for a process is a task. The Linux kernel internally refers to processes as __tasks__.

### Process Descriptor and the Task Structure

- The kernel stores the list of processes in a circular doubly linked list called the task list.
- Each element in the task list is a process descriptor of the type struct `task_struct`.

#### Allocating the Process Descriptor

- The task_struct structure is allocated via the slab allocator to provide object reuse and cache coloring.

<p align="center"><img src="https://i.imgur.com/mQZ3v4q.png" width="300px" height="auto"></p>

- Each task’s `thread_info` structure is allocated at the end of its stack.- The task element of the structure is a pointer to the task’s actual `task_struct`.

#### Storing the Process Descriptor

- x86 (which has few registers to waste), make use of the fact that struct `thread_info` is stored on the kernel stack to calculate the location of `thread_info` and subsequently the `task_struct`.

#### Process state

- The state field of the process descriptor describes the current condition of the process:
    - __TASK_RUNNING__: either running or in run queue  waiting to run. This is the only possible state for a process executing in user-space.
    - __TASK_INTERRUPTIBLE__: blocked, waiting for some condition to exist.
    - __TASK_UNINTERRUPTIBLE__ : identical to TASK_INTERRUPTIBLE except
    that it does not wake up and become runnable if it receives a signal.
    - __TASK_TRACED__: process is being traced by another process, such as a debugger, via ptrace.
    - __TASK_STOPPED__: execution has stopped; the task is not running nor is it eligible to run. This occurs if the task receives the `SIGSTOP`, `SIGTSTP`, `SIGTTIN`, or `SIGTTOU` signal or if it receives any signal while it is being debugged.

<p align="center"><img src="https://i.imgur.com/O4S7S7r.png" width="300px" height="auto"></p>

#### Manipulating the Current Process State

- Kernel code often needs to change a process’s state.The preferred mechanism is using:
`set_task_state(task, state); /* set task ‘task’ to state ‘state’ */`

#### Process Context

- When a program executes a system call or triggers an exception, it enters kernel-space. At this point, the kernel is said to be _executing on behalf of the process_ and is in __process context__.

#### The Process Family Tree

- All processes are descendants of the init process, whose PID is one.
- The init process, in turn, reads the system `initscripts` and executes more programs, eventually completing the boot process.
- Processes that are all direct children of the same parent are called __siblings__
- The relationship between processes is stored in the process descriptor. Each `task_struct` has a pointer to the parent’s task_struct, named `parent`, and a list of children, named `children`.

### Process Creation

- Unix takes the unusual approach of process creation into two distinct functions: `fork(` )and `exec()`
- `fork` starts a new process which is a copy of the one that calls it.
-  while `exec` loads a new executable into the address space and begins executing it.
- The combination of `fork()` followed by `exec()` is similar to the single function most operating systems provide.

#### Copy-on-Write

- In Linux, `fork()` is implemented through the use of copy-on-write pages.
- In the common case that a process executes a new executable image immediately after forking, this optimization prevents the wasted copying of large amounts of data (with the address space, easily tens of megabytes). 
- This is an important optimization because the Unix philosophy encourages quick process execution.

#### Forking

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

#### vfork()

-  has the same effect as `fork()`, except that the page table entries
of the parent process are not copied.
- parent is blocked until the child either calls `exec()` or exits.

### The Linux Implementation of Threads

- threads enable concurrent programming and, on multiple processor systems, true parallelism.
- linux has a unique implementation of threads: It does not differentiate between threads and processes. To Linux, a thread is just a special kind of process.
- linux threads are simply a manner of sharing resources between processes(which are already quite lightweight).

#### Creating Threads

- Threads are created the same as normal tasks, with the exception that the `clone()` is passed flags corresponding to the specific resources to be shared: `clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND, 0);`
- In contrast, a normal fork() can be implemented as: `clone(SIGCHLD, 0);`

#### Kernel Threads

- the significant difference between kernel threads and normal processes is that kernel threads do not have an address space.
- they operate only in kernel-space and do not context switch into user-space.
- are schedulable and preemptable, the same as normal processes.
- an be created only by another kernel thread. the kernel handles this automatically by forking all new kernel threads off of the __kthreadd__ kernel process. Look at `kthread_create()` to spawn a new ker thread.
- The process is created in an unrunnable state; it will not start running
until explicitly woken up via `wake_up_process()`.

### Process Termination

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

#### Removing the Process Descriptor

- After `do_exit()` completes, the process descriptor for the terminated process still exists, but the process is a zombie and is unable to run.
- This enables the system to obtain information about a child process after it has terminated.
- After the parent has obtained information on its terminated child, or signified to the kernel that it does not care, the child’s `task_struct` is deallocated.
- when it is time to finally deallocate the process descriptor, `release_task()` is invoked. It does the following:
    - calls `__exit_signal()`, which calls `__unhash_process()`, which in turns calls `detach_pid()` to remove the process from the __pidhash__ and remove the process from the task list.
    - `__exit_signal()` releases any remaining resources used by the now dead process and finalizes statistics and bookkeeping.
    - If the task was the last member of a thread group, and the leader is a zombie, then `release_task()` notifies the zombie leader’s parent.
    - `release_task()` calls `put_task_struct()` to free the pages containing the process’s kernel stack and `thread_info` structure and deallocate the slab cache containing the `task_struct`.
- `wait()` blocks the calling process until one of its child processes exits or a signal is received. After child process terminates, parent continues its execution after wait system call instruction.

#### The Dilemma of the Parentless Task

- If a parent exits before its children, some mechanism must exist to reparent any child tasks to a new process.
- `do_exit()` calls `exit_notify()`, which calls `forget_original_parent()`, which, in turn, calls `find_new_reaper()` to perform the __reparenting__.

## 2. Process Scheduling

- The process scheduler decides which process runs, when, and for how long.

### Multitasking

- Multitasking operating systems come in two flavors: __cooperative multitasking__ and
__preemptive multitasking__.
- Linux, like all Unix variants and most modern operating systems, implements preemptive multitasking.
- The time a process runs before it is preempted is usually predetermined, and it is called the __timeslice__ of the process.

### Linux’s Process Scheduler

- from Linux’s first version in 1991 through the 2.4 kernel series:
    - the Linux scheduler was simple, almost pedestrian, in design.
    - easy to understand, but scaled poorly in light of many runnable processes or many processors.
- during the 2.5 kernel development series, the Linux kernel received a scheduler overhaul:
    - A new scheduler, commonly called the __O(1) scheduler__ because of its algorithmic behavior
    - performed admirably and scaled effortlessly as Linux supported large “iron” with tens if not hundreds of processors
    - ideal for large server workloads; which lack interactive processes; but not optimal for __interactive processes__.
- in early 2.6 kernel series, developers introduced new process schedulers aimed at improving the interactive performance of the O(1) scheduler:
    - most notable of these was the __Rotating Staircase Deadline__ scheduler, which introduced the concept of fair scheduling.
    - this concept was the inspiration for the O(1) scheduler’s eventual replacement in kernel version 2.6.23, the __Completely Fair Scheduler__ (CFS).

### Policy 

- is the behavior of the scheduler that determines what runs when.

#### I/O-Bound Versus Processor-Bound Processes

- processes can be classified as either __I/O-bound__ or __processor-bound__.
- the former is characterized as a process that spends much of its time submitting and waiting on I/O requests.
- conversely, processor-bound processes spend much of their time executing code.

#### Process Priority

- the goal of priority-based schedulin algo is to rank processes based on their worth and need for processor time.
- the linux kernel implements two separate priority ranges:
    - the first is the __nice value__, a number from __–20 to +19__ with a default of 0.
    - __larger__ nice values correspond to a __lower__ priority
    - `ps -el`:  to list processes with their nice values (NI)
- the second range is the __real-time__ priority.
    - configurable, but by default range from __0 to 99__, inclusive.
    - __higher__ real-time priority values correspond to a __greater__ priority.
    - `ps -eo state,uid,pid,ppid,rtprio,time,comm`: (RTPRIO)

#### Timeslice

- is the numeric value that represents how long a task can run until it is preempted.
- the scheduler policy must dictate a __default timeslice__, which is not a trivial exercise.
- too long a timeslice causes the system to have poor interactive performance.
- too short a timeslice causes significant amounts of processor time to be wasted on the overhead of context switching.
- additionally, the conflicting goals of I/O bound versus processor-bound processes again arise:     
    - I/O-bound processes do not need longer timeslices (although they do like to run often).
    - whereas processor-bound processes crave long timeslice.
- linux’s CFS scheduler does not directly assign timeslices to processes; instead; in a novel approach, CFS assigns processes a __proportion__ of the processor.
- the amount of processor time that a process receives is a function of the load of the system.
- this assigned proportion is further affected by each process’s nice value.
- processes with higher nice values (a lower priority) receive a deflationary weight, yielding them a smaller proportion of the processor; 
- processes with smaller nice values (a higher priority) receive an inflationary weight, netting them a larger proportion of the processor.

#### The Scheduling Policy in Action

- On most OSs, to ensure good __interactive performance__, text editor a higher priority and larger timeslice than the video encoder. Advanced OSs do this automatically, by detecting that the text editor is interactive.
- Linux achieves these goals too, but by different means. Instead of assigning the text editor a specific priority and timeslice, it guarantees the text editor a specific __proportion of the processor__. 

### The Linux Scheduling Algorithm

#### Scheduler Classes

- the Linux scheduler is __modular__, enabling different algorithms to schedule different types of processes.
- this modularity is called __scheduler classes__.
- each scheduler class has a __priority__.
- the __base scheduler code__, which is defined in `kernel/sched.c`, iterates over each scheduler class in order of priority.
- the __highest__ priority scheduler class that has a runnable process wins, selecting who runs next.

#### Process Scheduling in Unix Systems

- modern process schedulers have two common concepts: __process priority__ and __timeslice__.
- timeslice is how long a process runs; processes start with some default timeslice.
- processes with a __higher priority__ run __more frequently__ and (on many systems) receive a higher timeslice. 
- on Unix, the priority is exported to user-space in the form of nice values. This sounds simple, but in practice it
leads to several pathological problems:
    - mapping nice values onto timeslices requires a decision about what absolute timeslice to allot each nice value.
    - relative nice values and again the nice value to timeslice mapping.
    - if performing a nice value to timeslice mapping, we need the ability to assign an absolute timeslice.
    - process wake up in a priority-based scheduler that wants to optimize for interactive tasks.
- the true problem, which is that assigning __absolute timeslices__ yields a __constant switching rate__ but __variable fairness__.

#### Fair Scheduling

- CFS is based on a simple concept: _Model process scheduling as if the system had an ideal, perfectly multitasking processor_.
- in such a system, each process would receive `1/n` of the processor’s time, where `n` is the number of __runnable processes__, and we’d schedule them for infinitely small durations, so that in any measurable period we’d have run all `n` processes for the same amount of time:
    - impractical, because it is not possible on a single processor to literally run multiple processes simultaneously
    - not efficient to run processes for infinitely small durations (the overhead of context switches + the effects on caches).
- CFS is mindful of the overhead and performance hit in doing so. Instead, CFS will run each process for some amount of time __round-robin__, selecting next the process that has run the least.
- Rather than assign each process a timeslice, CFS calculates how long a process should run as a function of __the total number of runnable processes__. 
- instead of using the nice value to calculate a timeslice, CFS uses the __nice__ value to __weight__ the proportion of processor a process is to receive: Higher valued (lower priority) processes receive a fractional weight relative to the default nice value, whereas lower valued (higher priority) processes receive a larger weight.
- each process then runs for a timeslice __proportional__ to its weight __divided__ by the __total weight__ of all runnable threads.
-  note that CFS isn’t perfectly fair, because it only __approximates perfect multitasking__, but it can place a lower bound on latency of `n` for `n` runnable processes on the unfairness.

### The Linux Scheduling Implementation
