## What does a program loader do?

1. **Input**: The `exec` system call takes the name of an executable file as its input. This file should already exist on the computer's disk.
2. **Checking Validity**: It first checks whether the specified file is a valid program that follows a specific format (XEXE format). This format ensures the file is a proper executable for the operating system.
3. **Terminating the Current Process**: If the file is valid, `exec` does two important things. First, it ends the currently running process (the program that called `exec`). This means that the current program will stop running and will not continue after `exec` is executed.
4. **Releasing Resources**: Before starting the new program, `exec` releases the resources used by the current process. This includes freeing up memory used by the program, such as heap and stack memory. It also makes sure that the memory areas associated with the current process are marked as empty.
5. **Preparing for the New Process**: Now, `exec` prepares to run the new program. It reclaims some memory previously used by the current process to set up the environment for the new one.
6. **Loading the New Program**: `exec` loads the new program from the disk into memory. This involves reading the program's code from the disk and putting it into memory so that the computer can execute it.
7. **Updating Page Table**: It updates a data structure called the page table to keep track of which parts of memory are used by the new program.
8. **Setting the Entry Point**: `exec` sets the starting point of the new program (called the Instruction Pointer or IP) to the beginning of its code.
9. **Starting Execution**: Finally, `exec` starts the execution of the new program in user mode. This means the new program begins running as if it were the one originally started by the user.

In simple terms, the `exec` system call allows you to switch from running one program to another. It makes sure the old program is properly terminated, frees up its resources, loads the new program, and starts it. This way, your computer can smoothly switch between different programs you want to run.

- eXpOS maintains a data structure called [memory free list](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#memory-free-list) in page 57 of the [memory](https://exposnitc.github.io/expos-docs/os-implementation/). Each Page can be shared by zero or more processes. There are 128 entries in the memory free list corresponding to each page of memory. For each page, the corresponding entry in the list stores the number of processes sharing the page. The constant [MEMORY_FREE_LIST](https://exposnitc.github.io/expos-docs/support-tools/constants/) gives the starting address of the memory free list.

# Module 1: Process Manager

This module contains functions that manage the different aspects related to processes.

|Function Number|Function Name|Arguments|Return Value|
|---|---|---|---|
|GET_PCB_ENTRY = 1|Get Pcb Entry|NIL|Index of Free PCB.|
|FREE_USER_AREA_PAGE = 2|Free User Area Page|PID|NIL|
|EXIT_PROCESS = 3|Exit Process|PID|NIL|
|FREE_PAGE_TABLE = 4|Free Page Table|PID|NIL|
|KILL_ALL = 5|Kill All|PID|NIL|

![[Pasted image 20230904001045.png]]
# Module 2: Memory Manager

This module handles allocation and deallocation of memory pages. The memory free list entry denotes the number of processes using(sharing) the memory page. Unused pages are therefore indicated by 0 in the corresponding entry in memory free list.

|Function Number|Function Name|Arguments|Return Value|
|---|---|---|---|
|GET_FREE_PAGE = 1|Get Free Page|NIL|Free Page number|
|RELEASE_PAGE = 2|Release Page|Page Number|NIL|
|GET_FREE_BLOCK = 3|Get Free Block|NIL|Free Block Number or -1|
|RELEASE_BLOCK = 4|Release Block|Block Number, PID|NIL|
|GET_CODE_PAGE = 5|Get Code Page|Block Number|Page Number|
|GET_SWAP_BLOCK = 6|Get Swap Block|NIL|Block Number|

## Implementation

### **1. Exit Process (function number = 3,[process manager module](https://exposnitc.github.io/expos-docs/modules/module-01/))[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-17/#1-exit-process-function-number-3process-manager-module)**

The first function invoked in the exec system call is the **Exit Process**function. Exit Process function takes PID of a process as an argument (In this stage, PID of the current process is passed). Exit process deallocates all the pages of the invoked process. It deallocates the pages present in the page table by invoking another function called **Free Page Table** present in the same module. Further, The Exit Process deallocates the user area page by invoking **Free User Area Page** in the same module. The state of the process (corresponding to the given PID) is set to TERMINATED. This is not the final Exit process function. There will be minor modifications to this function in the later stages.

### **2. Free Page Table (function number = 4, [process manager module](https://exposnitc.github.io/expos-docs/modules/module-01/))[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-17/#2-free-page-table-function-number-4-process-manager-module)**

The Free Page Table function takes PID of a process as an argument (In this stage, PID of the current process is passed). In the function Free Page Table , for every valid entry in the page table of the process, the corresponding page is freed by invoking the **Release Page** function present in the [memory manager module](https://exposnitc.github.io/expos-docs/modules/module-02/). `Since the library pages are shared by all the processes, do not invoke the Release Page function for the library pages.` Free Page Table function invalidates all the page table entries of the process with given PID. The part of the Free Page Table function involving updates to the Disk Map Table will be taken care in subsequent stages.

### **3. Free User Area Page (function number = 2,[process manager module](https://exposnitc.github.io/expos-docs/modules/module-01/))[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-17/#3-free-user-area-page-function-number-2process-manager-module)**

The function **Free User Area Page** takes PID of a process (In this stage, PID of the current process is passed) as an argument. The user area page number of the process is obtained from the process table entry corresponding to the PID. This user area page is freed by invoking the **Release Page** function from the memory manager module. `However, since we are using Free User Area Page to release the user area page of the current process one needs to be careful here. The user area page holds the kernel stack of the current process. Hence, releasing this page means that the page holding the return address for the call to Free User Area Page function itself has been released! Neverthless the return address and the saved context of the calling process will not be lost. This is because Release Page function is non blocking and hence the page will never be allocated to another process before control transfers back to the caller.` (Free User Area Page function also releases the resourses like files and semaphores acquired by the process. We will look into it in later stages.)

### **4. Release Page (function number = 2,[memory manager module](https://exposnitc.github.io/expos-docs/modules/module-02/))[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-17/#4-release-page-function-number-2memory-manager-module)**

The Release Page function takes the page number to be released as an argument. The Release Page function decrements the value in the memory free list corresponding to the page number given as an argument. Note that we don't tamper with the content of the page as the page may be shared by other processes. The [system status table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#system-status-table) keeps track of number of free memory pages available to use in the MEM_FREE_COUNT field. When the memory free list entry of the page becomes zero, no process is currently using the page. In this case, increment the value of MEM_FREE_COUNT in the system status table indicating that the page has become free for fresh allocation. Finally, Release page function must check whether there are processes in WAIT_MEM state (these are processes blocked, waiting for memory to become available). If so, these processes have to be set to READY state as memory has become available now. At present, the OS does not set any process to WAIT_MEM state and this check is superfluous. However, in later stages, the OS will set processes to WAIT_MEM state when memory is temporarily unavailable, and hence when memory is released by some process, the waiting processes have to be set to READY state.

## Overview

![[Pasted image 20230904001100.png]]

