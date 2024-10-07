#

- Modern processors execute code at different levels of privileges.
- x86 offers four levels (or rings) ring 0 > ring 1 > ring 2 > ring 3
- arm-32 offeres seven execution modes, six of which are priviliged.
- arm-64 uses the notion of Exception levels (EL0 to EL3) where EL3 > EL2 > EL1 > EL1
- Modern OSs have two available CPI privleges levels - user and kenel.
- Modern OS are monolithic (a large piece of stone -_-) when a process or thread issues the system call, it is switched to kenel mode and itself executes kenel mode.
- The kernel code executes within the context of user-space process or thread - we call it process context.
- Besides process context, another way kernel code executes - Hardare Interrupts (keyboard, network card, disk and usb and other stuff)
- CPU saves the current context and immediately re-vectors the CPU to run the code of the Interrupt Handler (Interrupt Service Routine, ISR)
- This runs in privliged mode and is asynchronous way to switch to kernel mode. (Interrupt context)

### Context

- Process context - kernel pocess is entered becz a pocess or thread issued a system call or a processor exception (page faults like that), kernel code is executed, kernel data is woked upon and is typically synchronous.
- Interrupt context - Kernel space is entered becz a peripheral chip asserrted a hardware interrupt, kernel code is executed, kernel data is worked upon and it's asynchronous.

## Understanding the VAS (Virtual Addrress Space)

- A fundamental rule of virtual memory is that all potentially addressable memory is in a box: that is it's sandboxed.
- We see the box as process image or process VAS.
- The user VAS is divided into homogenous memory called segments (o mappings), Internally they are contructed via mmap() system call.
- Text segment: Where machine code is stoed, where the processor core's instruction pointer register points while a thread of proocess executes code, static/fixed size. (Text segment doesn't stat at virtual address 0x0, it;s above that, the very first virtual page - the one encapsulating the NULL address 0x0 - called null-trap page).
- Data segment: Above text segment, it stores global and static data variables into 3 data segments:
1. Initialized: global/static vaiables ae stored here.
2. Uninitialized: they are auto-initialized to 0 at runtime - it's bss also.
3. Heap: Standard C lib APIs for memory allocation and freeing get memory from here. malloc() as we have.
On modern glibc, only malloc() calls fo memoy below MMAP_THESHOLD bytes (128 kb by default) get memoy from the heap.
Any higher than that is allocated as a seperate mapping in process VAS by mmap() system call (also called anonymous (or anon) mapping).
Heap grows up towards higherr VA and last legally reference able location one the heap is referred to as the program break (sbrk(0)).

- Shared libs (text, data): All shared libs that a process dynamically links to are mapped at runtime by mmap() into process VAS somewhere between top of heap and below stack of the main() thread.
- Stack: A region of memory which uses LIFO semantics.
- It's used to implement a high-level language's function calling mechanism and holds the thead context.
- It's a dynamic segment and grows down in VAS.
- Everytime a funtion is called, a stack frame is allocated and is very CPU-dependent (also called CPUT Application Binary Interface, ABI)
- The processor's core Stack pointer refister alays points to current frame, top of stack. As it grows down, top of stack is usually lower address.
- Processes must contain atleast one thread in execution (a thread is just an execution path with in a process).
- Like one is main() function, Every thread shares everything within the process VAS except for the stack.
- Every thread has it's own private stack.
- The stacks of other threads can be allocated anywhere between lib mappings and main() is at the top of stack.
- Try procmap() utility for VAS visual.

### Process, threads and their stacks

- Threads share everything, all process resources, user VAS, open files, signal dispositions, IPC objects, pagining tables and more except the stack.
- Every thread has it's own private stack as it's the stack that holds the execution context, if not, how they could run in parallel.
- For now, Thread (not the process) is the kenel scheduleable entity (KSE) - it's what gets scheduled to un on a CPU core.
- On linux, evey thread inclunding kenel threads maps to kernel metadata structure called task structure (also called process descriptor).
- For every thead alive, kernel maintains a corresponding task structure.

### We require one stack per thread per privilege level supported by the CPU

- User sapce stack in in play when thread executes user-mode code paths.
- Kernel space stack is in play when the thread switches to kernel mode via a system call or processor exception and executes kernel code paths (in process context)
- Kernel thread are pure, can't see the userland.

#### User space Organization

- Every process has at least once thread of execution main() thread.
- Threads could be within that process other than main thread
1. One user space stack is always present for the main thread. If it's single threaded (only main), it will have just one user mode stack.
2. If a process is multi-threaded, it will have one user mode thread stack per thread alive (including main()). The stacks are allocated either at the time of calling fork (for main() or pthread_create()) for the remaining threads within the process which results in this code path being executed in process context within the kernel itself.
3. User stack space is dynamic, grow and shrink and size limit is RLIMIT_STACK (~ 8MB), we can look with prlimit.

#### Kernel space Orgranization
1. There will be one kernel mode stack for each user mode thread alive including main.
2. Kernel mode stacks are fixed in size (static) and are small (2 pages in 32 bit, 4 pages in 64 bit ~ 4kb in size) getpagesize() system call check.
3. Each kernel thread has a task structure and a kernel mode stack is allocated to it at creation time and kernel thread has link to user space, it can't see.
4. Don't overflow your kenel stack, it's fixed and quite small.
5. Every thread alive has a task structure in kenel, that's how it tracks it and all threads attributes are stored here.
- and also a seperate stack IRQ is present to read hardware interrupts.

## Viewing the User and Kernel stacks

- It's helpful in debug as the stack only holds the current execution context of the thread in hich funtion it is executing code right now, how it got here - which allows us to inder the history.
- Being able to see and interpret the thread's call stack (call chain/call trace/ backtrace) is crucial.

#### We can see throguh proc filesystem (kernel space stack of a given thread or process)

- in /proc/PID/stack (pseudofile)
```
wiki@pi:~/Linux-Kernel-Programming_2E/ch6$ sudo cat /proc/210/stack
[<0>] rescuer_thread+0x31c/0x420
[<0>] kthread+0xf4/0x108
[<0>] ret_from_fork+0x10/0x20
```

- The function call graph ordering is from bottom up.
- Each line of output is a call frame, a function in call chain.
- if a call frame is ?, kernel can't interrpret this stack frame.
- Any linux foo() system call will typically become a sys_foo() funciton within the kenel.
- [<0>] zeroed oyt each stack frame for security reasons.
- <func>+x/y ? ---> x is byte offset from beginnning of the function where the execution is currently at and y is what the kerrnel has deemed to be the length of this function.

#### Viewing the user space stack of a given thread/process

- it's not easy as in kernel space.
- We use GDB debugger
- using ustack example from book, this is the output
```
wiki@pi:~/Linux-Kernel-Programming_2E/ch6$ ./ustack $(pgrep --newest bash)

warning: could not find '.gnu_debugaltlink' file for /lib/aarch64-linux-gnu/libtinfo.so.6
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/aarch64-linux-gnu/libthread_db.so.1".
0x0000ffff8ff17a6c in __GI___wait4 (pid=-1, stat_loc=0xffffc72e0740, options=10, usage=0x0) at ../sysdeps/unix/sysv/linux/wait4.c:30

warning: 30     ../sysdeps/unix/sysv/linux/wait4.c: No such file or directory

Thread 1 (Thread 0xffff9007d020 (LWP 1450) "bash"):
#0  0x0000ffff8ff17a6c in __GI___wait4 (pid=-1, stat_loc=0xffffc72e0740, options=10, usage=0x0) at ../sysdeps/unix/sysv/linux/wait4.c:30
#1  0x0000aaaabb26f984 in ?? ()
#2  0x0000aaaabb1bcf6c in wait_for ()
#3  0x0000aaaabb19f6a8 in execute_command_internal ()
#4  0x0000aaaabb19fa84 in execute_command ()
#5  0x0000aaaabb190800 in reader_loop ()
#6  0x0000aaaabb185490 in main ()
[Inferior 1 (process 1450) detached]
```

- Read bottom-up.

### Modern ways to view both stacks (eBPF)

- Extended Berkeley Packet Filter
- Install: sudo apt-get install bpfcc-tools linux-headers-$(uname -r)
- Checkout: https://github.com/iovisor/bcc/blob/master/tools/stackcount_example.txt
```
wiki@pi:~/Linux-Kernel-Programming_2E/ch6$ stackcount-bpfcc -v
usage: stackcount-bpfcc [-h] [-p PID] [-c CPU] [-i INTERVAL] [-D DURATION] [-T] [-r] [-s] [-P] [-K] [-U] [-v] [-d] [-f] [--debug] pattern
stackcount-bpfcc: error: the following arguments are required: pattern
wiki@pi:~/Linux-Kernel-Programming_2E/ch6$ stackcount-bpfcc -h
usage: stackcount-bpfcc [-h] [-p PID] [-c CPU] [-i INTERVAL] [-D DURATION] [-T] [-r] [-s] [-P] [-K] [-U] [-v] [-d] [-f] [--debug] pattern

Count events and their stack traces

positional arguments:
  pattern               search expression for events

options:
  -h, --help            show this help message and exit
  -p PID, --pid PID     trace this PID only
  -c CPU, --cpu CPU     trace this CPU only
  -i INTERVAL, --interval INTERVAL
                        summary interval, seconds
  -D DURATION, --duration DURATION
                        total duration of trace, seconds
  -T, --timestamp       include timestamp on output
  -r, --regexp          use regular expressions. Default is "*" wildcards only.
  -s, --offset          show address offsets
  -P, --perpid          display stacks separately for each process
  -K, --kernel-stacks-only
                        kernel stack only
  -U, --user-stacks-only
                        user stack only
  -v, --verbose         show raw addresses
  -d, --delimited       insert delimiter between kernel/user stacks
  -f, --folded          output folded format
  --debug               print BPF program before starting (for debugging purposes)

examples:
    ./stackcount submit_bio         # count kernel stack traces for submit_bio
    ./stackcount -d ip_output       # include a user/kernel stack delimiter
    ./stackcount -s ip_output       # show symbol offsets
    ./stackcount -sv ip_output      # show offsets and raw addresses (verbose)
    ./stackcount 'tcp_send*'        # count stacks for funcs matching tcp_send*
    ./stackcount -r '^tcp_send.*'   # same as above, using regular expressions
    ./stackcount -Ti 5 ip_output    # output every 5 seconds, with timestamps
    ./stackcount -p 185 ip_output   # count ip_output stacks for PID 185 only
    ./stackcount -c 1 put_prev_entity   # count put_prev_entity stacks for CPU 1 only
    ./stackcount -p 185 c:malloc    # count stacks for malloc in PID 185
    ./stackcount t:sched:sched_fork # count stacks for sched_fork tracepoint
    ./stackcount -p 185 u:node:*    # count stacks for all USDT probes in node
    ./stackcount -K t:sched:sched_switch   # kernel stacks only
    ./stackcount -U t:sched:sched_switch   # user stacks only
```

- Other profiling tools, perf, flame graphs, bpftrace