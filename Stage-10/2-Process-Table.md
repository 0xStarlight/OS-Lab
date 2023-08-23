# Process Table (Process Control Block)

The Process Table (PT) contains an entry for each [process](https://exposnitc.github.io/expos-docs/os-spec/processmodel/) present in the system. The entry is created when the process is created by a Fork system call. Each entry contains several fields that stores all the information pertaining to a single process. The maximum number of entries in PT (which is maximum number of processes allowed to exist at a single point of time in eXpOS) is MAX_PROC_NUM. In the current version of eXpOS, MAX_PROC_NUM = 16.

Each entry in the Process Table has a constant size, defined by the PT_ENTRY_SIZE. In this version of eXpOS, PT_ENTRY_SIZE is set to 16 words. Any entry of PT has the following fields:

|   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
|Offset|0|1|2|3|4|6|7|8|9|10|11|12|13|14|15|
|Field Name|TICK|PID|PPID|USERID|STATE|SWAP FLAG|INODE INDEX|INPUT BUFFER|MODE FLAG|USER AREA SWAP STATUS|USER AREA PAGE NUMBER|KERNEL STACK POINTER (KPTR)|USER STACK POINTER (UPTR)|PTBR|PTLR|

- **TICK** (1 word)- keeps track of how long the process was in memory/ swapped state. It has an initial value of 0 and is updated whenever the scheduler is called. TICK is reset to 0 when a process is swapped out or in.
- **PID** (1 word) - process descriptor, a number that is unique to each process. This field is set by Fork System Call. In the present version of eXpOS, the pid is set to the index of the entry in the process table.
- **PPID** (1 word) - process descriptor of the parent process. This field is set by Fork System Call. PPID of a process is set to -1 when it's parent process exits. A process whose parent has exited is called an Orphan Process.
- **USERID** (1 word) - Userid of the currently logged in user. This field is set by Fork System Call.
- [**STATE**](https://exposnitc.github.io/expos-docs/os-design/process-table/#state) (2 words) - a two tuple that describes the current state of the process. The details of the states are explained below.
- **SWAP FLAG** (1 word) - Indicates if the process is swapped (1) or not (0). The process is said to be swapped if any of its user stack pages or its kernel stack page is swapped out.
- **INODE INDEX** (1 word)- Pointer to the Inode entry of the executable file, which was loaded into the process's address space.
- **INPUT BUFFER** (1 word) - Buffer used to store the input read from the terminal. Whenever a word is read from the terminal, Terminal Interrupt Handler will store the word into this buffer.
- **MODE FLAG** (1 word) - Used to store the system call number if the process is executing inside a system call. It is set to -1 when the process is executing the exception handler. The value is set to 0 otherwise.
- **USER AREA SWAP STATUS** (1 word) - Indicates whether the user area of the process has been swapped (1) or not (0).
- **USER AREA PAGE NUMBER** (1 word) - Page number allocated for the user area of the process.
- **KERNEL STACK POINTER** (1 word) - Pointer to the top of the kernel stack of the process. The **offset of this address within the user area** is stored in this field.
- **USER STACK POINTER** (1 word) - Logical address of the top of the user stack of the process. This is used when the process is running in kernel mode and the machine's stack pointer is pointing to the top of the kernel stack.
- **PTBR** (1 word) - pointer to [PER-PROCESS PAGE TABLE](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-page-table).
- **PTLR** (1 word) - PAGE TABLE LENGTH REGISTER of the process.

Invalid entries are represented by -1.

**Note1:** In this version of eXpOS, the [Per-Process Resource Table](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-resource-table) is stored in the user area of each process. Generally, the Per-Process Resource Table is stored somewhere in memory and a pointer to the table is kept in the Process Table entry.

**Note2:** The Process Table is present in page 56 of the memory (see [Memory Organisation](https://exposnitc.github.io/expos-docs/os-implementation/)), and the SPL constant [PROCESS_TABLE](https://exposnitc.github.io/expos-docs/support-tools/constants/) points to the starting address of the table. PROCESS_TABLE + PID*16 gives the begining address of process table entry corresponding to the process with identifier PID.

#### STATE[¶](https://exposnitc.github.io/expos-docs/os-design/process-table/#state "Permanent link")

The tuple can take the following values

- (ALLOCATED,___) - The PCB Entry for the process has been allocated, but the process has not been created yet, because the Fork system call has not completed the creation of the new process.
- (CREATED,___) – The process has been created in memory and data structures set up, but has never been scheduled. The fork system call creates the child process in CREATED state.
    
- (RUNNING,___) – The process is in execution. This field is set by the scheduler when a process is scheduled.
    
- (READY,___) – The process is ready to be scheduled.
- (WAIT_PROCESS, WAIT_PID) – The process is waiting for a signal from another process whose PID is WAIT_PID. This field is set by WAIT system call.
- (WAIT_FILE, INODEINDEX) - The process is blocked for a file whose index in the inode table is INODEINDEX.
- (WAIT_DISK,___) - The process is blocked because of one of the following reasons: a) It is waiting for disk to complete a disk-memory transfer operation it had initiated OR b) It wants to execute a disk transfer, but the disk is busy, handling a disk-memory transfer request issued by some other process. The disk interrupt handler changes the state from (WAIT_DISK,___) to (READY,___), after it has completed the disk transfer.
- (WAIT_SEMAPHORE,SEMTABLEINDEX) - The process is waiting for a semaphore that was locked by some other process.
- (WAIT_MEM, ___) – The process is blocked due to unavailability of memory pages.
- (WAIT_BUFFER, BUFFERID) – The process is waiting for disk buffer of index BUFFERID to be unlocked.
- (WAIT_TERMINAL,___) – The process is waiting for one of the following: a) a Read from Terminal, which it has issued OR b)to be completed or if the terminal is blocked by some other process. The terminal interrupt handler changes the state from (WAIT_TERMINAL,___) to (READY,___), after it has completed the terminal read/write.

Process states and transitions can be viewed [here](https://exposnitc.github.io/expos-docs/os-design/state-diag/).

Note

When a process terminates, the STATE field in it's process table entry is marked TERMINATED to indicate that the process table entry is free for allocation to new processes.

#### PER-PROCESS PAGE TABLE[¶](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-page-table "Permanent link")

The Per-Process Page Table contains information regarding the physical location of the pages of a process. Each valid entry of a page table stores the physical page number corresponding to each logical (virtual) page associated with the process. The logical page number can vary from 0 to MAX_PROC_PAGES- 1 for each process. Therefore, each process has MAX_NUM_PAGES entries in the page table. The address of Page Table of the currently executing process is stored in PTBR of the machine and length of the page table is stored in PTLR. In this version of eXpOS, MAX_NUM_PAGES is set to 10, hence PTLR is always set to 10.

Associated with each page table entry, typically **auxiliary information** is also stored. This is to store information like whether the process has write permission to the page, whether the page is dirty, referenced, valid etc. The exact details are architecture dependent. The eXpOS specification expects that the hardware provides support for reference, valid and write bits. (See page table structure of XSM [here](https://exposnitc.github.io/expos-docs/arch-spec/paging-hardware/)).

|   |   |   |   |
|---|---|---|---|
|PHYSICAL PAGE NUMBER|REFERENCE BIT|VALID BIT|WRITE BIT|

- **Reference bit** - The reference bit for a page table entry is set to 0 by the OS when the page is loaded to memory and the page table initialized. When a page is accessed by a running process, the corresponding reference bit is set to 1 by the machine hardware. This bit is used by the page replacement algorithm of the OS.
- **Valid bit** - This bit is set to 1 by the OS when the physical page number field of a page table entry is valid (i.e, the page is loaded in memory). It is set to 0 if the entry is invalid. The OS expects the architecture to generate a page fault if any process attempts to access an invalid page.
- **Write bit** - This bit is set to 1 by the OS if the page can be written and is set to 0 otherwise. The OS expects the architecture to generate an exception if any process, while running in the user mode, attempts to modify a page whose write bit is not set.

If the Page Table entry is invalid, the Physical Page number is set to -1.

Note

In the XSM machine, the first three bits of the second word stores the reference bit, valid bit and the write permission bit. The fourth bit is the dirty bit which is not used by eXpOS.

For more information, see [XSM.](https://exposnitc.github.io/expos-docs/arch-spec/)

Note

In the eXpOS implementation on the XSM architecture, if a page is not loaded to the memory, but is stored in a disk block, the disk block number corresponding to the physical page number is stored in the [disk map table](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-disk-map-table) of the process. If memory access is made to a page whose page table entry is invalid, a _page fault_ occurs and the machine transfers control to the Exception Handler routine, which is responsible for loading the correct physical page.

Note

The Page Table is present in page 58 of the memory (see [Memory Organisation](https://exposnitc.github.io/expos-docs/os-implementation/)), and the SPL constant [PAGE_TABLE_BASE](https://exposnitc.github.io/expos-docs/support-tools/constants/) points to the starting address of the table. PAGE_TABLE_BASE + PID*20 gives the begining address of page table entry corresponding to the process with identifier PID.

#### PER-PROCESS DISK MAP TABLE[¶](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-disk-map-table "Permanent link")

The per-process Disk Map Table stores the disk block number corresponding to the pages of each process. The Disk Map Table has 10 entries for a single process. When the memory pages of a process are swapped out into the disk, the corresponding disk block numbers of those pages are stored in this table. It also stores block numbers of the code pages of the process.

The entry in the disk map table entry has the following format:

|   |   |   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|
|Unused|Unused|Heap 1 in disk|Heap 2 in disk|Code 1 in disk|Code 2 in disk|Code 3 in disk|Code 4 in disk|Stack Page 1 in disk|Stack Page 2 in disk|

If a memory page is not stored in a disk block, the corresponding entry must be set to -1.

Note

The Disk Map Table is present in page 58 of the memory (see [Memory Organisation](https://exposnitc.github.io/expos-docs/os-implementation/)), and the SPL constant [DISK_MAP_TABLE](https://exposnitc.github.io/expos-docs/support-tools/constants/) points to the starting address of the table. DISK_MAP_TABLE + PID*10 gives the begining address of disk map table entry corresponding to the process with identifier PID.

#### USER AREA[¶](https://exposnitc.github.io/expos-docs/os-design/process-table/#user-area "Permanent link")

Corresponding to each user process, the kernel maintains a seperate memory region (called the user area) for its own purposes. The present version of eXpOS allocates one memory page per process as the user area. A part of this space is used to store the per process resource table of the process. The rest of the memory is alloted for the kernel stack of the process.

Hence in eXpOS, each process has a kernel stack in addition to user stack. We maintain a seperate stack for the kernel operations to prevent user-level "hacks" into kernel.

![User Area](https://exposnitc.github.io/expos-docs/assets/img/user_area_new.png)

#### PER-PROCESS RESOURCE TABLE[¶](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-resource-table "Permanent link")

The Per-Process Resource Table has 8 entries and each entry is of 2 words. **The last 16 words of the User Area Page are reserved for this**. For every instance of a file opened (or a semaphore acquired) by the process, it stores the index of the Open File Table (or Semaphore Table) entry for the file (or semaphore) is stored in this table. One word is used to store the resource identifier which indicates whether the resource opened by the process is a [FILE](https://exposnitc.github.io/expos-docs/support-tools/constants/) or a [SEMAPHORE](https://exposnitc.github.io/expos-docs/support-tools/constants/). Open system call sets the values of entries in this table for a file.

The per-process resource table entry has the following format.

|   |   |
|---|---|
|Resource Identifier (1 word)|Index of Open File Table/ Semaphore Table entry (1 word)|

File descriptor, returned by Open system call, is the index of the per-process resource table entry for that open instance of the file.

A free entry is denoted by -1 in the Resource Identifier field.

#### PER-PROCESS KERNEL STACK[¶](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-kernel-stack "Permanent link")

Control is tranferred from a user program to a kernel module on the occurence of one of the following events :

1. The user program executes a system call
2. When an interrupt/exception occurs.

In either case, the kernel allocates a separate stack **for each process** (called the kernel stack of the process) which is different from the stack used by the process while executing in the user mode (called the user stack). Kernel modules use the space in the kernel stack for storing local data and do not use the user stack. This is to avoid user "hacks" into the kernel using the application's stack.

In the case of a system call, the application will store the parameters for the system call in its user stack. Upon entering the kernel module (system call), the kernel will extract these parameters from the application's stack and then change the stack pointer to its own stack before further execution. Since the application invokes the kernel module voluntarily, it is the responsibility of the application to save the contents of its registers (except the stack pointer and base pointer registers in the case of the XSM machine) before invoking the system call.

In the case of an interrupt/exception, the user process does not have control over the transfer to the kernel module (interrupt/exception handler). Hence the execution context of the user process (that is, values of the registers) must be saved by the kernel module, before the kernel module uses the machine registers for other purposes, so that the machine state can be restored after completion of the interrupt/exception handler. The kernel stack is used to store the execution context of the user process. This context is restored before the return from the kernel module. (For the implementation of eXpOS on the XSM architecture, the [backup](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/#backup) and [restore](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/#restore) instructions facilitate this).

In addition to the above, if a kernel module invokes another kernel module while executing a system call/interrupt, the parameters to the called module and the return values from the module are passed through the same kernel stack.

Here is a detailed tutorial on [kernel stack management in sys](https://exposnitc.github.io/expos-docs/os-implementation/)A separate tutorial is provided for [kernel stack managament during context switch](https://exposnitc.github.io/expos-docs/os-design/timer-stack-management/).