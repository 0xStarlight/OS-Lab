# Pager Module

# Summary

In this stage, we will learn about effectively using the limited physical memory pages of the XSM machine to run multiple concurrent processes. We will implement the Swap Out and Swap In functions of the Pager module (Module 6), along with modifications in the Timer Interrupt and Context Switch modules.

eXpOS allows running 16 processes concurrently, with 52 memory pages available for user processes. If each process requires four code pages, two heap pages, two user stack pages, and one kernel stack page, a total of 144 pages would be needed for all 16 processes. To overcome this limitation, the OS can swap out inactive processes' memory pages to disk and allocate them to new processes. The swapped out pages can be swapped back when the inactive processes need to be reactivated, creating an illusion of more memory.

eXpOS uses a memory management approach where the system regularly checks the available memory. If the available memory drops below a critical level (MEM_LOW), a swap out is initiated. The OS selects an inactive process and swaps out its pages to make more free memory available. The heap and user stack pages are swapped out to the disk. The code pages and kernel stack page are not swapped out. The swap area in the disk (256 to 511) is reserved for swapping purposes.

A swapped out process is swapped back into memory when either its TICK value exceeds MAX_TICK or the available free memory pages exceed MEM_HIGH. The TICK value is associated with each process and is incremented during the timer interrupt routine.

The timer interrupt handler checks the TICK status of swapped out processes and the memory availability status. If a swap-in or swap-out is needed, it sets the PAGING_STATUS field in the system status table accordingly and informs the scheduler. The scheduler schedules the swapper daemon (PID=15) when PAGING_STATUS is set to SWAP_IN or SWAP_OUT. The swapper daemon shares the code of the idle process and is responsible for setting up a user context for swapping operations.

During swapping, if the swapper daemon gets blocked, the idle process is run. Only the swapper daemon or the idle process is scheduled until the swap-in or swap-out is completed to avoid unpredictable conditions caused by other processes rapidly changing memory availability.

The algorithms for Swap-in and Swap-out are implemented in the Pager Module (Module 6).

---
# Starting

In this phase, the focus is on optimizing the usage of the XSM machine's limited physical memory pages to enable the execution of a higher number of concurrent processes. This involves the implementation of the Swap Out and Swap In functions within the Pager module (Module 6). Additionally, certain adjustments are made to the Timer Interrupt and Context Switch module to accommodate these changes effectively.

### 1. Memory Management
1. eXpOS permits the simultaneous execution of up to 16 processes within the system.
2. The available memory for user processes ranges from page 76 to page 127 in the memory organization.
3. If each process requires four code pages, two heap pages, two user stack pages, and one kernel stack page, the total memory requirement for 16 processes exceeds the available memory, amounting to 144 pages.
4. To overcome the memory limitation, the operating system identifies inactive processes whose memory pages can be swapped out to the disk when there is a shortage of free memory pages for a new process.
5. The freed memory pages are allocated to the new process, effectively expanding the available memory for active processes.
6. Later, when the inactive process needs to be reactivated, the OS can swap the previously swapped-out memory pages back into the main memory, thereby maintaining the illusion of more memory being available than the physical memory allows.

### 2. When will the swapping happen?
1. eXpOS employs proactive memory management to prevent complete memory exhaustion before initiating a process swap out.
2. The system regularly monitors the available memory and triggers a swap out when the available memory falls below a critical level denoted by MEM_LOW (presently set to 4).
3. When the available memory pages are less than MEM_LOW, eXpOS calls the Swap Out function of the pager module to select a less active process for swapping.
4. Swap Out moves the memory pages of the selected process to the disk blocks and releases the memory pages, excluding those of the library and the kernel stack.
5. Code pages are not copied to the disk as they already exist there, whereas the heap and user stack pages are swapped to the disk.
6. eXpOS reserves 256 blocks in the disk (blocks 256 to 511 in the disk organization) for the swapping process, which constitute the swap area.

### 3. When is the swapped out process swapped back into memory?

1. A swapped-out process is brought back into memory under two conditions: 
	1. if it remains in a swapped-out state for more than a defined period
	2. if the available memory pages exceed the MEM_HIGH threshold (set to 12 in the current design).
2. Each process has an associated TICK value that is reset upon swapping out and incremented each time the system enters the timer interrupt routine.
3. When a process's TICK value surpasses the MAX_TICK threshold, the OS initiates the swapping-in process for the corresponding process.
4. Another condition for swapping-in occurs when the available free memory pages surpass the MEM_HIGH value.
5. The OS checks for the MEM_LOW/MEM_HIGH condition during the timer interrupt handler, ensuring regular monitoring of the TICK/MEM_FREE_COUNT values.

### 4. Timer Interrupt Implementations

> Note: A swapper daemon process, also known as a swapping daemon, is a background process that manages the swapping of processes between main memory and secondary storage (such as a hard disk) in an operating system. It is responsible for moving inactive or less frequently used processes out of main memory to free up space for other processes that need to be executed. The swapper daemon ensures efficient utilization of memory resources by swapping processes in and out based on their priority and demand.

1. The timer interrupt handler will be modified to examine the TICK status of swapped-out processes and the memory availability status in the system status table.
2. If the handler determines that a swap-in/swap-out operation is necessary, it sets the PAGING_STATUS field in the system status table to SWAP_IN/SWAP_OUT, signaling the scheduler about the requirement.
3. The timer interrupt handler, when triggered from a regular process context (not the special swapper daemon process), will not directly initiate any swap-in/swap-out operations.
4. The actions performed during the timer interrupt handler's execution in the context of the swapper daemon will be described separately.

### 3. Scheduling Implementations

1. The eXpOS scheduler is modified to handle the scheduling of the swapper daemon when the PAGING_STATUS field in the system status table is set to SWAP_IN or SWAP_OUT.
2. The OS reserves PID=15 specifically for the swapper daemon process, which is a special process initiated by the kernel for handling swapping operations.
3. The swapper daemon process shares the code of the idle process and essentially functions as a duplicate idle process, running with a distinct PID.
4. The primary role of the swapper daemon is to establish a user context for the swapping operations, allowing for efficient swapping of processes between the disk and memory.
5. Introducing the swapper daemon reduces the number of concurrently running user applications to 12, as one of the process slots is now allocated to the swapper daemon.

### 4. Swap In / Swap Out

1. The timer interrupt handler, when invoked by the swapper daemon, initiates the Swap-in/Swap-out functions of the pager module after inspecting the value of PAGING_STATUS in the system status table.
2. During the swapping process, the swapper daemon might become blocked due to disk-memory transfer delays, causing the scheduler to execute the Idle process.
3. The Idle process always remains unblocked and is prioritized for scheduling whenever no other process is ready to run.
4. Once a swap-in/swap-out is triggered by the timer, the OS scheduler exclusively schedules the swapper daemon or the idle process until the swap operation is completed, preventing unpredictable conditions that could arise from rapidly changing memory availability due to other processes acquiring or releasing memory.
5. This design, although not the most efficient, is simple to implement and ensures the execution of the full quota of 16 processes concurrently, as constrained by the size of the process table in the eXpOS implementation.

## Modifications

### 1. Timer Interrupt
![[Pasted image 20231022211036.png]]

```ad-tip
title: Swapper Deamon
It is a user program which is created by the bootstrap loader. This process uses the same code of the Idle process and hence has no file in the disk associated with it. The main purpose of the swapper daemon is to serve as a user program such that the OS performs swap-in and swap-out operations in the kernel context of this process. The PID of the swapper daemon is fixed to be 15.

```ad-note
title: Algorithm
Identical to the Idle Process and runs with the code of the Idle Process.

```

#### 1. Swapping Mechanism

1. The timer interrupt handler must verify whether the current process is the swapper daemon.
2. This condition arises only when a swap operation is to be initiated.
3. When the swapper daemon is detected, the handler checks the PAGING_STATUS field of the system status table and calls the Swap_in/Swap_out function of the pager module as required, with SWAP_OUT = 1 and SWAP_IN = 2.

#### 2. Idle Process Handling 

1. When the current process is the idle process, two scenarios can arise.
2. If swapping is ongoing (check PAGING_STATUS), it suggests that the idle process was scheduled due to the swapper daemon being blocked. The timer needs to call the scheduler. If the daemon remains blocked, the scheduler will continue to run the idle process, or the daemon will be scheduled again if unblocked.
3. If swapping is not ongoing, the scenario is similar to the condition checked when the timer is entered from any process other than the paging process, which will be described subsequently.

#### 3. Decision Logic for Scheduling and Swap-in/Swap-out

1. Timer handler determines normal scheduling or swap-in/swap-out when entered from a process without ongoing scheduling.
2. Swap-in is initiated if MEM_FREE COUNT is below MEM_LOW.
3. Swap-out is initiated if memory availability is high (MEM_FREE_COUNT exceeds MEM_HIGH) or a swapped process has TICK value exceeding MAX_TICK.

#### 4. Process Tick Management and its Role in Swap-in/Swap-out Algorithms

1. The Timer interrupt is modified to increment the TICK field in the process table of each NON-TERMINATED process.
2. Upon process creation via the Fork system call, the TICK value is initialized to 0 in the process table.
3. Every time the system enters the timer interrupt handler, the TICK value of each process is incremented accordingly.
4. The TICK value of a process is reset to zero upon its swap-in or swap-out, providing an indication of its time in memory or swap.
5. The tick value of a non-swapped process signifies its duration in memory without being swapped out.
6. Conversely, the tick value of a swapped out process signifies its duration in the swapped state.
7. Swap-in and swap-out algorithms utilize the TICK value to determine the process that has been in the swapped state for the longest time for swapping operations.

#### 5. Algorithm for Implementation
```c

Switch to the Kernel Stack.     /* See kernel stack management during system calls */
Save the value of SP to the USER SP field in the Process Table entry of the process.
Set the value of SP to the beginning of User Area Page.

Backup the register context of the current process using the BACKUP instruction.


/* This code is relevant only when the Pager Module is implemented in Stage 27 */
If swapping is initiated, /* check System Status Table */
{
    /* Call Swap In/Out, if necessary */

    if the current process is the Swapper Daemon and Paging Status is SWAP_OUT,
        Call the swap_out() function in the Pager Module.

    else if the current process is the Swapper Daemon and Paging Status is SWAP_IN, 
        Call the swap_in() function in the Pager Module.

    else if the current process is Idle,                          
        /* Swapping is ongoing, but the daemon is blocked for some disk operation and idle is being run now */
        /* Skip to the end to perform context switch. */

}

else           /* Swapping is not on now.  Check whether it must be initiated */
{
    if (MEM_FREE_COUNT < MEM_LOW)       /* Check the System Status Table */
        /* Swap Out to be invoked during next Timer Interrupt */
        Set the Paging Status in System Status Table to SWAP_OUT.

    else if (there are swapped out processes)            /* Check SWAPPED_COUNT in System Status Table */
        if (Tick of any Swapped Out process > MAX_TICK or MEM_FREE_COUNT > MEM_HIGH)
            /* Swap In to be invoked during next Timer Interrupt */
            Set the Paging Status in System Status Table to SWAP_IN.

}
/* End of Stage 27 code for Swap In/Out management */


Change the state of the current process in its Process Table entry from RUNNING to READY.

Loop through the process table entires and increment the TICK field of each process.

Invoke the context switch module .

Restore the register context of the process using RESTORE instruction.

Set SP as the user SP saved in the Process Table entry of the new process.
Set the MODE_FLAG in the process table entry to 0.

ireturn
```

### 2. Pager Module (Mod 6)

Pager module is responsible for selecting processes to swap-out/swap-in and also to conduct the swap-out/swap-in operations for effective memory management.
#### 1. Swap Out (function number = 1, Pager module )

![[Pasted image 20231022214711.png]]

> Section 1
1. The Swap Out function is called exclusively from the timer interrupt handler when it is entered from the context of the Swapper Daemon.
2. The Swap Out function does not require any arguments for its execution.
3. It begins by identifying an appropriate process for swapping out from the memory to the disk.
4. Processes in the WAIT_PROCESS or WAIT_SEMAPHORE state and not actively running are considered first for swapping out.
5. In case no such process is available, the function selects the process that has remained in memory for the longest duration for swapping out.
6. The TICK field in the process table aids in detecting the process that has been in memory for the longest time, thus facilitating the selection of the process with the highest TICK value for swapping out.

> Section 2
1. The selected process for swapping out has its TICK field reset to 0, indicating the start of its time count in memory.
2. Code pages belonging to the process are released, and the corresponding page table entries for the code pages are invalidated.
3. The swapping-out process may have shared heap pages, but for simplicity, shared heap pages are not swapped out to the disk.
4. The kernel stack page, as well, is not swapped out to maintain simplicity in the implementation.
5. Non-shared heap pages and user stack pages are saved in the swap area on the disk.
6. The Get Swap Block function from the memory manager module is used to find available free blocks in the swap area for storing memory pages.
7. The Disk Store function of the device manager module is then invoked to store these memory pages into the allocated disk blocks, updating the disk map table with the corresponding page numbers.
8. Memory pages of the process are released using the Release Page function of the memory manager module, and the page table entries for the swapped out pages are invalidated.
9. The SWAP FLAG in the process table of the swapped out process is set to 1, indicating that the process has been swapped out.

**Finally, the PAGING_STATUS in the System Status Table is reset to 0. This step informs the scheduler that the swap operation is complete and normal scheduling can be resumed.**

> Algorithm
```c

Choose a process to swap out. (other than the IDLE, Shell or INIT)
    Loop through the Process Table and find a non-swapped process that is in the WAIT_PROCESS state.
    If there are no non-swapped processes in the WAIT_PROCESS state, find a non-swapped process in the WAIT_SEMAPHORE state.
    If there are no non-swapped processes in the WAIT_PROCESS and WAIT_SEMAPHORE state, 
            find process with the highest TICK which is not running, terminated, allocated or swapped.

If no such process exists, 
        set the PAGING_STATUS back to 0 and return.

Set the TICK field of the process table entry of the selected process to 0.
/* When the process goes to swap, TICK starts again */

Call the release_page() function in the Memory Manager module to deallocate the valid code pages of the process.
Invalidate the Page table entry correpsonding to the code pages.

For each heap page that is not shared and is valid { /* Shared heap pages are not swapped out. */
    Get a free swap block by calling the get_swap_block() function in the Memory Manager module.
    Store the disk block number in the Disk Map Table entry of the process curresponding to the heap page.
    Use the disk_store() function in the Device Manager module to write the heap page to the block found above
    Call the release_page() function in the Memory Manager module to deallocate the page.
    Invalidate the Page table entry correpsonding to the page.
}

Get two free swap block by calling the get_swap_block() function in the Memory Manager module.

Use the disk_store() function in the Device Manager module to write the two stack pages to the disk blocks found above.

Call the release_page() function in the Memory Manager module to deallocate the two pages.

Update the Disk Map Table entry of the process to store the disk block numbers of the stack.

Invalidate the Page table entries correpsonding to the two stack pages.

Set the SWAP_FLAG field in the process table entry of the process to 1.

In the System Status Table, increment the SWAP_COUNT and reset the PAGING_STATUS back to 0.   
/* The scheduler can now resume normal scheduling */ 

return;
```
#### 2. Swap In (function number = 2, Pager module )

![[Pasted image 20231022220106.png]]

> Section 1

1. Swap In function is invoked from the timer interrupt handler and does not take any arguments.
2. The Swap In function selects a swapped-out process to be brought back to memory. 
3. The process which has stayed the longest time in the disk and is ready to run is selected.
4. That is, the process with the highest TICK among the swapped-out READY processes is selected.

> Section 2

1. Now that a process is selected to be swapped back into the memory, the TICK field for the selected process is initialized to 0.
2. Code pages of the process are not loaded into the memory, as these pages can be loaded later when an exception occurs during the execution of the process.
3. Free memory pages for the heap and user stack are allocated using the Get Free Page function of the memory manager module, and disk blocks of the process are loaded into these memory pages using the Disk Load function of the device manager module.
4. The Page table is updated for the new heap and user stack pages.
5. The swap disk blocks used by these pages are released using the Release Block function of the memory manager module, and the Disk map table is invalidated for these pages.
6. Also, the SWAP FLAG in the process table of the swapped-in process is set to 0, indicating that the process is no longer swapped out.

**Finally, the PAGING_STATUS in the System Status Table is reset to 0. This step informs the scheduler that the swap operation is complete and normal scheduling can be resumed.**

### 3. Memory Manager Module (Mod 2)

#### 1. Get Swap Block (function number = 6, Memory Manager Module )

1. Get Swap Block function does not take any arguments.
2. The function returns a free block from the swap area (disk blocks 256 to 511 - see disk organization) of the eXpOS.
3. Get Swap Block searches for a free block from DISK_SWAP_AREA (starting of disk swap area) to DISK_SIZE - 1 (ending of the eXpOS disk).
4. If a free block is found, the block number is returned.
5. If no free block in the swap area is found, -1 is returned to the caller.

> Algorithm
```c

loop through entries in the Disk Free List from DISK_SWAP_AREA to DISK_SIZE - 1{    /* swap area */
    if ( a free entry is found ){
            Set the Disk Free List entry as 1;
            Return the corresponding block number;
    }
}
return -1;
```

### 4. Memory Manager Module (Mod 2)

Previously, the [Context Switch module](https://exposnitc.github.io/expos-docs/modules/module-05/) (scheduler module) would select a new process to schedule according to the Round Robin scheduling algorithm. The procedure for selecting a process to execute is slightly modified in this stage. If swap-in/swap-out is ongoing (that is, if the PAGING_STATUS field of the [system status table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#system-status-table) is set), the context switch module schedules the Swapper Daemon (PID = 15) **whenever it is not blocked** . If the swapper daemon is blocked (for some disk operation), then the idle process (PID = 0) must be scheduled. (The OS design disallows scheduling any process except Idle and Swapper daemon when swapping is on-going.) If the PAGING_STATUS is set to 0, swapping is not on-going and hence the next READY/CREATED process which is not swapped out is scheduled in normal Round Robin order. Finally, if no process is in READY or CREATED state, then the idle process is scheduled.

Modify Context Switch module implemented in earlier stages according to the detailed algorithm given [here](https://exposnitc.github.io/expos-docs/modules/module-05/) .

### 5. OS Startup 

Modify [OS Startup Code](https://exposnitc.github.io/expos-docs/os-design/misc/#os-startup-code) to initialize the process table and page table for the Swapper Daemon (similar to the Idle Process).

> Final Algorithm
```c

  Load IDLE process and boot module from disk to memory. See disk/memory organization.
  Set SP to (user area page number of idle) * 512 + 1 and invoke module 7. //running the boot module in the context of idle.

  // after returning from the boot module


  /* Initialize the IDLE process.*/

  Initialize the Page table for IDLE process (PID = 0)
        Initialize the Page table base register (PTBR) to PAGE_TABLE_BASE and PTLR to 10.
        Set the page table entries for library and heap to -1. Set auxiliary information for these pages to "0000". 
        // idle doesn't invoke any library function.   
        Set the first code page entry to 69 (See memory organization). Set auxiliary information for valid code pages to "0100". 
        Set remaining code page entries to -1 and auxiliary information to "0000".
        Set the first stack page entry to 70 and auxiliary information for this page to "0110".
        Set second stack page entry to -1 and auxiliary information to "0000".
   
  Initialize the process table for IDLE process.
  Store the IP value (from the header of the IDLE) on top of first user stack page [70*512] = [69*512 +1].


  /* Initialize the Swapper Daemon (not relevant before Stage 27) */

  Initialize the Page table for Swapper Daemon (PID = 15)
        /* Swapper Daemon is identical to Idle and shares the code for Idle */
        Initialize the Page table base register (PTBR) to PAGE_TABLE_BASE + 20*15 and PTLR to 10.
        Set the page table entries for library and heap to -1. Set auxiliary information for these pages to "0000". 
        // swapper doesn't invoke any library function.   
        Set the first code page entry to that of Idle (See memory organization). Set auxiliary information for valid code pages to "0100". 
        Set remaining code page entries to -1 and auxiliary information to "0000".
        Set the first stack page entry to 81 and auxiliary information for this page to "0110".
        Set second stack page entry to -1 and auxiliary information to "0000".
  
  Initialize the process table for Swapper Daemon.
        Initialize the fields of process table as -  TICK, USERID as 0, PID as 15, STATE as CREATED,
        USER AREA PAGE NUMBER as 82 (allocated from free user space), KPTR to 0, UPTR to 4096 (starting of first user stack page),
        PTBR to PAGE_TABLE_BASE + 20*15 and PTLR as 10.
    
  Store the IP value (from the header of the IDLE whose code is shared by Swapper) on top of first user stack page [81*512] = [69*512 +1].


  /* Initialize the IDLE2 (not relevant before Stage 28) */

  Initialize the Page table for IDLE2 (PID = 14)
        /* IDLE2 is identical to Idle and shares the code for Idle */
        Initialize the Page table base register (PTBR) to PAGE_TABLE_BASE + 20*14 and PTLR to 10.
        Set the page table entries for library and heap to -1. Set auxiliary information for these pages to "0000". 
        // swapper doesn't invoke any library function.   
        Set the first code page entry to that of Idle (See memory organization). Set auxiliary information for valid code pages to "0100". 
        Set remaining code page entries to -1 and auxiliary information to "0000".
        Set the first stack page entry to 83 and auxiliary information for this page to "0110".
        Set second stack page entry to -1 and auxiliary information to "0000".
   
  Initialize the process table for IDLE2.
        Initialize the fields of process table as -  TICK, USERID as 0, PID as 14, STATE as RUNNING,
        USER AREA PAGE NUMBER as 84 (allocated from free user space), KPTR to 0, UPTR to 4096 (starting of first user stack page),
        PTBR to PAGE_TABLE_BASE + 20*14 and PTLR as 10.
     


  Set the Page table base register (PTBR) to PAGE_TABLE_BASE and PTLR to 10.

  Schedule IDLE process for excecution. (Return to user mode.)

```

### 6. Boot Module (Mod 7)

Modify [Boot module](https://exposnitc.github.io/expos-docs/modules/module-07/) to add the following steps :

1. Load module 6 (Pager Module) form disk to memory. See [disk/memory organization](https://exposnitc.github.io/expos-docs/os-implementation/) .
2. Initialize the SWAPPED_COUNT field to 0 and PAGING_STATUS field to 0 in the [system status table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#system-status-table) to 0, as initially there are no swapped out processes.
3. Initialize the TICK field to 0 for all the 16 [process table](https://exposnitc.github.io/expos-docs/os-design/process-table/) entries.
4. Update the MEM_FREE_COUNT to 45 in the [system status table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#system-status-table) . (The 2 pages are allocated for the user/kernel stack for Swapper Daemon reducing the number from 47 to 45).

---

# Questions

```ad-question
title: What is the reasoning behind the order of choosing the process to swap out? Why is a process in WAIT_PROCESS swapped out ahead of a process waiting for the terminal? What about processes waiting for the disk?
```

```ad-question
title: Is it possible that there are processes waiting for memory, even when MEM_FREE_COUNT > MEM_HIGH?
```

```ad-question
title: Why are code pages not loaded back to memory when the process is swapped in?
```

```ad-question
title: Why only READY state processes are selected for swap in, even though swapped out processes can be in blocked state also?
It is not very useful to swap in a process which is in blocked state into the memory. As the process is in blocked state, even after swapping in, the process will not execute until it is made READY. Until the process is made READY, it will just occupy memory pages which could be used for some other READY/RUNNING process.
```


---

# Assignments

> Attached in the files





















































