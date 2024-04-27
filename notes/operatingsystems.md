# Operating Systems
## Week 1
Concepts that will be repeatedly mentioned in the future

### Generic Computer Architecture

- CPU: processor for computation
    - Registers: fastest storage in computer
    - ALU: arithmetic logic unit: does arithmetic computations
    - PC/stack pointers
    - Instruction register + instruction decode
- I/O devices: terminal, disks, video board, printer, etc.
- Memory: RAM containing data and programs for CPU

### Architectural Features Motivated by OS Services

- Protection: kernel, protected instructions, base/limit registers
- Interrupts: interrupt vectors
- System calls: Trap instructions and trap vectors
- I/O: interrupts and memory mapping
- Scheduling, error recovery, accounting: timer
- synchronization: atomic instructions
- Virtual memory: translation look-aside buffers

### Protection

- kernel mode vs user mode: to protect the system from aberrations, certain instructions restricted for use
    - users can’t address I/O directly
    - users can’t manipulate state of memory
    - can’t set mode bits that determine user/kernel mode ;)
    - disable/enable interrupts
    - halt machine
- Only kernel mode can do those things

### System calls

- privileged instructions
- causes trap, which vectors (jumps) to trap handler in kernel
- uses parameter to jump to appropriate handler
- handler saves caller’s state before executing
- think of system calls as API for the kernel
- examples: fork, waitpid, execve, open, close, read, write

### Memory protection

- protect user programs from each other
- protect the OS from user programs
- base and limit registers loaded by OS before running program; these are the bounds of allocated memory for program
- when running program, check each user reference to make sure it falls in between base and limit registers

Programs shouldn’t be able to access memory of other programs

### Memory Hierarchy

- higher = small, fast, more costly, lower latency
- lower = large, slow, less expensive, higher latency
- registers: 1 cycle, L1: 2 cycles, L2: 7 cycles, RAM: 100 cycles, DIsk: 40,000,000 cycles, Network: 200 million+ cycles
- Note: cycle is one iteration of CPU; for example, 1 GHz CPU is a billion cycles per second
- L1 and L2 are within each core; often there is a L3 cache on the CPU as well
- when you want to load data:
    - look in L1; if isn’t there, go to lower level
    - as seen above, getting data takes longer and longer, so important to minimize cache misses
- Why are disks so slow? Physical form of hard storage, distance from CPU

### Process Layout in Memory

When a program is running, it becomes a process. The OS creates a region of memory for that process that can't be touched by other programs.

Components:
- Stack
- Gap
- Data
- Text

- After running a program, the assembly code is stored at addresses x0000 and up.
- Stack stores function parameters, function calls, serves as temporary storage; grows downward from 0xFFF...
- Data: variables stored by the program

To implement this, there are three special registers: stack pointer, frame pointer, program counter

What if you run multiple programs at once? Current state of registers is saved; context switch

### Caches
The cache policy decides what units of memory are stored in the caches.
Commonly, entire lines are moved into cache; this is because of spatial locality.
What is spatial locality? Very often we are accessing units of memory close by (like consecutive elements of array)

### Traps

- Special conditions detected by architecture
  - ex: page fault, write to read only, overflow, system call
- After detecting a trap, hardware
  - saves state of process
  - transfers control to the handling process for trap
    - CPU indexes the memory-mapped trap vector with the trap number
    - jumps to address given in the vector,
    - executes at that address.
    - After, resumes execution of process
- Traps are performance optimization; naive solution would be to insert extra instructions where special condition could arise

### I/O

- Every I/O device has processor to run autonomously
- CPU issues commands to I/O devices
- When I/O device completes command, it issues interrupt
- CPU stops and responds to interrupt (with traps)
- Two methods: synchronous, asynchronous
  - Synchronous: while hardware handling command, CPU is stuck waiting
  - Asynchronous: CPU can keep on running cycles while hardware handles request
- Memory mapped I/O:
  - enables direct access (vs moving I/O code and data into memory)

## Week 2
When you run an exe file, OS creates process, or a running program
- OS timeshares CPU across multiple processes: virtualizes CPU
  - Number of runnning processes -> number of physical cores
  - programs can respond even while CPU is doing other tasks
- OS has scheduler that picks a process to execute

### What is a process?
- Unique identifier (PID)
- memory image: code and data (static), stack and heap (dynamic)
- CPU context: registers, program counter, current operands, stack pointer
- File descriptors: pointers to open files and devices: STDOUT, STDIN, STDERR
  - often, STDOUT is your screen

### State of a process
- Running: currently executing
- Ready: waiting to be scheduled
- Blocked: suspended, not ready to run
  - Could be waiting for some event
  - Disk will issue an interrupt when data is ready
- New: being created, yet to run
- Dead: terminated

Process state transitions:
- From running to ready: descheduled
- from running to blocked: input/output initiates
- from blocked to running: input/output done

### OS data structures
OS maintains a data structure (e.g. a linked list) of all active processes
information about process stored in process control block (PCB)
- Process identifier
- Process state
- pointers to related processes
- CPU context of process (saved while suspended)
- pointers to memory locations
- pointers to open files

In Linux, you can see process information in directory `proc/<pid>`

### Process APIs
- API: Application Programming Interface
  - functions available to write user programs
- API provided by OS is set of system calls

Should we rewrite programs for each OS?
- POSIX API: standard set of system calls that OS must implement
  - Portable Operating System Interface
  - Programs written to POSIX API can run on any POSIX compliant OS
  - most OSes are POSIX compliant
  - ensures proram portability
- Program language libraries: hide the details of invoking system calls
  - `printf()` function in C library calls the write system call to `write()` to screen
  - User programs usually don't need to worry about system calls

Process related system calls (UNIX):
- `fork()` creates new child process
  - All processes created by forking from a parent
  - New process created by copy of parent's memory image
  - new process added to process list, scheduled
  - parent and child modify memory independently
  - On success, the PID of child process returned to parent
  - On failure, no child created, returns -1 to parent
  - Which runs first?
    - Determined by scheduling policy
    - Upon creation of child, it is in ready state
    - up to OS to decide which one to run first
  - Process termination scenarios
  - `wait()`: blocks parent process until child process finishes
  - What happens during exec?
    - Both parent and child running same code; not very useful!
    - Process can run  `exec()` to load other executable to its memory image
      - child  can run different program from parent
  - in basic OS, init process created after initialization of hardware
    - init spawns a shell like `bash`
    - shell reads user command, `fork`s a child, `exec`s the command, `wait`s, then reads next command
    - common commands like `ls` are executables that are `exec`'ed by shell

Example:
```
int main() {
  pid = fork();
  if (pid < 0) printf("Error"); // child process failed to be created
  else if (pid == 0) printf("this is child process"); // in the child process, fork() returns 0
  else printf("this is parent process); // in the parent process, fork() will return PID of child process
}
```

### Process Execution Mechanism
How are processes executed by CPU?

### Function call vs System call
- Function call translates to jump instruction
- new stack frame pushed to stack, stack pointer updated
- old value of PC saved
- in user memory space

How is system call different?
- CPU has multiple privilege levels
- Kernel does not trust user stack; uses separate kernel stack
- kernel does not trust user provided addresses to jump to
  - sets up interrupt descriptor table (IDT) at boot time
  - IDT has addresses of kernel functions for system calls and other events

### Mechanism of system call
Trap instructions
- usually hidden from user
- Execution:
  - move CPU to higher privilege level
  - switch to kernel stack
  - save context on kernel stack (so know where to return)
  - look up address in IDT and jump to trap handler
- IDT has many addresses, which to use?
  - System calls/interrupts store number in CPU register before calling trap, to identify which entry to use

Return from trap
- when OS done handling syscall or interrupt, calls special instruction: return from trap
  - restore context of CPU 
  - change privilege back to user mode
- User process unaware of being suspended

Possible for CPU to not return to same process
- sometimes impossible to return: process has exited, segfault, blocking system call
- sometimes OS does not want to return back: runtime is too long, must timeshare CPU
- OS performs context switch to switch to another process

### OS Scheduler
How the OS decides what processes to run at any time

Two types of schedulers: non preemptive, preemptive
- non preemptive schedules switch only if blocked or terminated process
- preemptive schedulers can switch even when process is ready to continue

Example of context switch:
- todo

### Scheduling Policies
What are we trying to optimize?
- maximize utilization: fraction of time that CPU is used
- minimize average turnaround time: time between arrival and completion of process
- minimize average response time: time between arrival and first scheduling
- fairness: all processes must be treated equally
- minimize overhead: run process long enough to amortize cost of context switch

Policies:
- FIFO: first in first out (queue)
  -  issue: convoy effect, high turnaround time
- SJF: shortest job first
  - optimal when all processes arrive together
  - non preemptive; short jobs can still be stuck behind long ones if they arrive later
- STCF: shortest time to completion first
  - preemptive scheduler
  - preempts running task if time left is more than than that of new arrival
- Round Robin
  - every process executes for fixed quantum slice
  - slices large enough to amortize cost of context switch
  - preemptive
  - good in response time and fairness
  - bad turnaround time
- Real schedulres are more complex
  - Multi level feedback queue:
    - many queues with different priority
    - process from highest priority queue scheduled first
    - within same priority, any algorithm like RR
    - priority of process decays with age; job in top queue can get switched to lower queue

## Week 4
### Virtual Memory
Why virtualize memory?
- real memory is messy
- multiple active processes timeshare CPU
  - memory of many processes must be in memory
- Hide complexity from user

Virtual address space:
- every process assumes it has access to memory from 0 to MAX
- program code, heap (grows positively), stack (grows negatively)
- CPU issues loads and stores to virtual addresses

How to translate between real and virtual memory addresses?

### MMU
Memory management unit
- OS divides virtual address space into fixed size pages
- pages mapped to physical frames
- page table stores mappings from virtual to physical
- MMU has access to page table and uses it to translate
- Context switch: CPU gives MMU pointer to new page table

### Design Goals
- Transparency: hide details from user
- Efficiency: minimize overhead and wastage in memory and time
- isolation, projection: user processes should not be able to access outside address space

Memory Allocation
- Malloc (C library)
- heap: libc uses brk/sbrk system call
- can also allocate page sized memory using mmap()

### Mechanism of Address Translation
Base and bound registers
- place entire memory image in one chunk
- physical address = virtual address + base

Segmentation

Paging

typical size of page table

### Demand Paging
Not neccessary for pages of active processes to always be in main memory;
OS uses part of disk (swap space) to store pages not in active use

Page fault
- Present bit: indicates if page in memory or not
- MMU reads present bit; if page present, directly access, if not, page fault

Page fault handling
- CPU to kernel mode
- OS fetches disk address of page, issues read to disk
- OS context switches to other process
- when read complete, OS updates page table, marks it as ready
- when process scheduled again, OS restarts instruction that caused page fault

Summary
- CPU issues load instruction to virtual address for code or data
  - check CPU cache first; go to main memory in case of cache miss
  - caches return raw data (no address associated)
- MMU looks up translation lookaside buffer for virtual address
  - if TLB hit, obtain physical address, fetch memory location and return to CPU
  - if TLB miss, MMU accesses memory, walks page table, obtains page table entry
    - if present bit in PTE, access memory
    - if not present but valid, page fault
    - if invalid, trap to OS for illegal access

How does TLB work?
- cache of frequently requested pages by the process
- each entry is unique to that process
  - same virtual addresses for two different processes map to different physical addresses
- must be cleared out whenever switching to new process
- What does this imply?
  - frequent context switches reduce TLB hit rate

More in page faulting
- when servicing page fault, what if OS finds there is no free page to swap in faulting page?
- Inefficient to swap out existing and then swap in faulting page
- OS proactively swap out pages to keep list of free pages
- Page replacement policy

Page replacement policy
- optimal: replace page not needed for longest time in future (but OS doesn't know that)
- FIFO
  - not good because popular pages get swapped in and out over and over
- LRU: not frequently used in past will be swapped out
  - works well due to locality of references
- 

## Week 5
Demand paging comtinued

### Pre-paging
OS guesses in advance which page to move back to memory
- Very hard to do because of dynamic branching (CPU doesn't know beforehand whether to jump or not)

What happens when page removed from memory?
- if page contained code, simply remove; you can reload it from disk
- if page contained data, save the data so it can be reloaded if referred to again
- for pages containing stack and heap data, must copy it over

At any given time, page of virtual memory can exist in one or more of
- file system
- physical memory
- swap space

Locality of reference:
- Temporal locality: items tend to be referenced again (while loop)
- Spatial locality: memory close together tends to be referenced (e.g. iterating through array)

This is why demand paging can work!

Let p be probability of page fault
- access time (1-p) x ma + p x page fault time
- if memory access is 200 ns and page fault takes 25 ms
- assuming p is very small, we have ma + p x page fault time

Transparent page faults
- Suppose we have instruction mov a, (r10)+;
- if a isn't in memory, page fault triggered and r10 incremented
- but when instruction restarted, r10 is wrongly incremented again! this is a **side effect**
- other examples: block transfer instructions where source and destination overlap

### LRU Policy
Works well because of locality of references

All implementations and approximations require hardware assistance

Possible implementations:
- timestamp for each page with time of last access; throw out LRU page
  - problems: OS must record timestamp for each memory access, and need to look through all pages to find LRU
  - O(N)
- Keep list of pages; front is most recently used, end is least recently used
  - on page access, move page to front of list, doubly linked list
  - still too expensive; OS must modify multiple pointers on each memory access
  - O(1) though!

These implementations are perfect; we can try to find approximation that is less expensive

### Second Chance Policy
Essentially, pages can have one of two age values: 0 or 1
- when a page is retrieved, set age to 0
- during page fault, iterate through pages:
  - if page has age 0, set age to 1
  - if page has age 1, swap it out and stop iterating
- Benefits: low space overhead (1 bit!), less time overhead (worst case is still the same)
- Have to implement some structure to hold the pages (linked list)

### Enhanced Second Chance
- hardware keeps a *modify* bit (in addition to reference bit)
- 1: page has been modified
- 0: page is same as copy on disk
- If page is unmodified, no need to rewrite it on disk
- (r,m)
  - 0,0: replace!
  - 0,1: not as good for replacement
  - 1,0: likely used again soon, OS won't need to write it though
  - 1,1: will be used again soon and must be written out
- page algorithm:
  - (0,0): replace the page
  - (0,1): initiate I/O to write out page, lock page in memory until I/O completes, and then continue
  - pages with reference bit 1 have it set to zero
  - hand goes completely around once: wasn't any (0,0) page
  - on second pass, some pages might now be (0,0)
  - there must exist (0,0) page on third pass

### Inter Process Communication
Shared memory:
- processes can both access same region of memory via shmget() system call

Signals
- certain set of signals supported by OS
  - for example: ctrl C sends SIGINT signal to running process
- signal handler: every process as default code to execute for each signal
- some handlers can be overridden to do other things
- Can't send very much data

Sockets
- sockets used for two processes on same or different machines to communicate
  - TCP/UDP sockets across machines
  - Unix sockets in local machine
- Communicating with sockets:
  - processes open sockets and connect them to each other
  - Messages written into socket can be read from other process
  - OS transfers data across socket buffers

Message Queues
- mailbox abstraction
- process can open mailbox at specified location
- processes can send/receive messages from mailbox

Blocking vs non-blocking communication
- some IPC actions can block
  - reading from socket with no data, or empty message queue
  - writing to full socket/message queue
- system calls have versions that block or return with error code

### Multiprogramming and Thrashing
...

## Week 7?
### Threads and Concurrency
So far we have looked at single threaded programs

What's a thread?
- Another copy of a process that executes independently
- shares the same address space
  - compare to fork(): child processes have a new memory image
- each thread has separate PC, can run over different part of program
- separate stacks for independent function calls

Process vs threads
- Parent P forks child C
  - P and C share no memory
  - need IPC to communicate
  - extra copies of data and code in memory
- Parent P executes two threads T1, T2
  - T1 and T2 share parts of address space
  - global variables used for communication
  - smaller memory footprint
- threads are like separate processes, but on same address space

Parallelism: single process can effectively utilize multiple CPU cores
- If you have cores C1, C2, and process has threads T1, T2, both threads executing at same time on separate cores

Concurrency: running multiple threads/processes at same time, even on single CPU core, by interleaving execution

Question: if there is no parallelism in system, is concurrency still useful?
- even without parallelism, concurrency of threads ensures effective use of CPU when one thread blocks

Scheduling Threads
- OS schedules ready threads, much like processes
- Context of thread (PC, registers) saved from thread control block (TCB)
  - Each PCB has one or more TCBs
- Threads scheduled independently by kernel are kernel threads
- some libraries provide user-level threads
  - user program sees 3 threads, there are fewer kernel threads
  - user level threads have low overhead 

### Race Conditions

## Week 8?
### Locks
Consider update of variable shared between threads
- Special lock variable `lock_t mutex`
- `lock()` and `unlock()` functions surround critical section
  - Said section must be executed by first thread that reaches it
  - Other threads are blocked from proceeding until lock is released
  - Pthreads library

Goals of lock implementation
- mutual exclusion
- fairness: all threads should eventually get lock
- low overhead: acquiring releasing and waiting for lock shouldn't consume many resources

Condition Variable:
- Queue that thread can put itself into when waiting on some condition
- Another thread that makes condition true