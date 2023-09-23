# Summary

This stage covers exception handling in eXpOS, which involves four events that generate exceptions: illegal memory access, illegal instruction, arithmetic exception, and page fault. The exception handler code used in previous stages contained only the halt instruction, which is inappropriate for handling exceptions that occur in a single process. Therefore, we implement the exception handler that takes appropriate action for each exception. The exception handler occupies page 2 and 3 in the memory and blocks 15 and 16 in the disk. There are four special registers in XSM that are used to obtain the cause of the exception and related information.

The page fault exception (EC=0) occurs when the last instruction in the currently running application tried to access/modify data from a legal address within its address space, but the page was set to invalid in the page table or fetch an instruction from a legal address within its address space, whose page table entry is invalid. In this case, the exception handler resumes the execution of the process after allocating the required page(s) for the process and attaching the page(s) to the process (by setting page table entries appropriately). The strategy followed in this stage is to start executing a process with just one page of code and two pages of stack allocated initially. When the process tries to access a page that was not loaded, an exception is generated, and the execption handler will allocate the required page.

In previous stages, exec system call allocated 2 memory pages each for the heap and the stack. It also allocated and loaded all the code pages of the process. We will modify exec to allocate memory pages for only stack (2 pages). For code blocks, only a single memory page is allocated, and the first code block is loaded into that memory page. We will write a new module function Get Code Page in the memory manager module for simultaneously allocating a memory page and loading a code block.

The Get Code Page function takes as input the block number of a single code block and loads that block into a memory page. Code pages are shared by the processes running the same program. The exception handler first switches to the kernel stack and backs up the register context as done by any other hardware interrupt routine. If the cause of the exception is other than page fault, exception handler halts the process gracefully and then invokes the scheduler to run other processes.

---

# Starting

## Demand Paging

Let us know about memory allocation strategy known as "lazy allocation" or "demand paging." This strategy is used by modern operating systems to efficiently manage memory resources and allow more concurrent processes to run compared to pre-allocating all memory for a process at the time of initialization.

Here's a breakdown of why the OS uses lazy allocation and how it improves memory utilization and system performance:

1. **Conserving Memory Resources:** In a computer system, physical RAM is a finite resource, and it is shared among multiple processes. If the OS allocated all the pages required for a process when it is initialized, it would have to reserve a potentially large amount of memory for each process, even if the process doesn't immediately need all of that memory. This approach would lead to inefficient memory utilization because many processes would have allocated but unused memory.
2. **Concurrency:** One of the primary goals of an operating system is to maximize system resource utilization, including CPU and memory. By allocating memory pages only when they are actually needed, the OS can support a larger number of concurrent processes. In a multi-user or multi-tasking environment, this is essential for efficient resource sharing.
3. **Page Fault Handling:** When a process is initialized with a limited number of pages (e.g., one page of code and two pages of stack), it can start execution quickly. If the process tries to access a page that was not initially loaded into memory (a page fault), the OS can handle this on-demand. It will allocate the required page, whether it's a code page or a data page, from secondary storage (disk) and load it into memory.
4. **Efficient Memory Utilization:** With demand paging, memory utilization is improved because pages are allocated based on the actual needs of each process. Pages that are rarely or never accessed might never be loaded into memory, saving precious RAM for more critical data and code pages.
5. **Optimizing for Working Set:** Over time, the OS can track which pages are frequently accessed by a process, forming what is called the "working set" of pages. The OS can then prioritize loading and keeping these pages in memory to minimize page faults, further improving performance.

In summary, the lazy allocation or demand paging strategy allows the operating system to allocate memory resources efficiently and support a larger number of concurrent processes by allocating pages on-demand as processes access them. This approach strikes a balance between memory utilization and performance, ensuring that processes have access to the memory they need without wasting valuable RAM on unused pages.

## Disk map table

In this system setup, each process has a special data structure called a "Per-process Disk Map Table." This table is used to keep track of where the different parts of a process's memory are stored on the computer's disk. It helps the operating system manage memory efficiently. Here's a breakdown of how it works:

1. **Structure of the Disk Map Table**: The Disk Map Table is like a small chart or table that holds information for a specific process. Each entry in this table takes up one word of data. The table has a fixed size of 10 words for each process.
2. **Allocation for Different Memory Parts**: The Disk Map Table is divided into sections to manage different parts of the process's memory:
    - One word is reserved for the "user area page." This area might store information related to the user, but its purpose can vary depending on the system.
    - Two words are reserved for the "heap." The heap is a part of memory where the process can dynamically allocate data during its execution.
    - Four words are reserved for "code" pages. These pages typically store the program's instructions.
    - Two words are reserved for "stack" pages. The stack is used for function calls and local variables in the program.
    - One word is left unused in this setup.
3. **Storing Disk Block Numbers**: When a process is running, its memory pages are stored in RAM (the computer's working memory). However, to save space and allow for efficient management, the operating system may decide to store a copy of some of these pages on the computer's disk. The Disk Map Table helps keep track of which memory pages are on the disk by storing the disk block numbers.
4. **Filling the Disk Map Table during Exec System Call**: The "exec" system call is a way to start a new program or process. During this call, the operating system prepares the process for execution. In this specific stage, the system is modifying the "exec" system call to initialize the Disk Map Table.
    - For code pages: The Disk Map Table entries corresponding to the code pages of the new process are filled with disk block numbers that point to the location of the executable file in the "inode table." The inode table is a data structure that keeps track of files and their locations on disk.
    - Remaining entries: Entries for the heap, stack, and other parts of memory are set to "invalid" (-1 in this case). This means that initially, these parts of memory are not yet allocated or stored on the disk. They will be managed and filled as needed during the execution of the program.

So, in summary, the Disk Map Table is a data structure used by the operating system to keep track of where different parts of a process's memory are located on the computer's disk. During the "exec" system call, the table is initialized with the disk block numbers for the code pages of the program being loaded, while other parts of memory remain unallocated until they are needed during program execution.

- There are four events that result in generation of an exception in XSM. These events are a) illegal memory access, b) illegal instruction, c) arithmetic exception and d) page fault.

> Exception handler mechanism gives a facility to resume the execution of the process after the corresponding exception has been taken care of. It is not always possible to resume the execution of the process, as some events which cause the exception cannot be corrected. In this case, the proper action is to halt the process gracefully. For the events 1) illegal memory access (EC=2) 2) illegal instruction (EC=1) and 3) arithmetic exception (EC=3), the exception handler just prints the cause of the exception. These cases occur because the last instruction executed (in the currently running user process) resulted in the corresponding error condition. As the OS is not reponsible for correcting these conditions (why?), the exception handler halts the process gracefully and then invokes the scheduler to run other processes.

- The **page fault exception** (EC=0) occurs when the last instruction in the currently running application tried to either -
    
    1. Access/modify data from a legal address within its address space, but the page was set to invalid in the page table
    2. fetch an instruction from a legal address within its address space, whose page table entry is invalid.
- In either case, the exception occured not because of any error from the side of the application, but because the OS had not loaded the page and set the page tables. In such case, **the exception handler resumes the execution of the process after allocating the required page(s) for the process and attaching the page(s) to the process (by setting page table entries appropriately). If the faulted page is a code page, the OS needs to load the page from the disk to the newly allocated memory.**
    
- The register EIP saves the logical IP value of the instruction which has raised the exception. The register PN stores the logical page number of the address that has caused the page fault.**Note that eXpOS is designed such that, page fault exception can only occur for heap and code pages.** Library pages are shared by all processes so they are always present in the memory. Stack pages are neccessary to run a process and are accessed more frequently. So both library and stack pages for a process should be present in the memory. Based on the value present in Exception Page Number (EPN) register, the exception handler finds out whether page fault has caused for heap or code page. When page fault has occured for heap page (EPN value 2 or 3), exception handler allocates 2 new memory pages by invoking the **GetFree Page** function in[memory manager module](https://exposnitc.github.io/expos-docs/modules/module-02/). If the page fault has occured for a code page, then the exception handler invokes the **Get Code Page** function in memory manager module. The page table of the process is updated to store the page number obtained from Get Code Page or Get Free Page functions. After handling the page fault exception, the exception handler restores the register context, switches to user stack and returns to user mode.
    

> Upon return to user mode, the instruction in the application that caused the exception must be re-executed. This indeed is the correct execution semantics as the machine had failed to execute the instruction that generated the exception. The XSM hardware sets the address of the instruction in the EIP register at the time of entering the exception.After completing the actions of the exception handler, the OS must place this address on the top of the application program's stack before returning control back to user [mode.An](http://mode.An) OS can implement **Demand Paging**, as we will be doing here, only if the underlying machine hardware supports re-execution of the instruction that caused a page fault.

![[Pasted image 20230922203353.png]]

#### Implementation of [Exception Handler](https://exposnitc.github.io/expos-docs/os-design/exe-handler/)

![[Pasted image 20230922203416.png]]

---

# Questions

```ad-question
title: Does EPN always equal to the logical page number of EIP?
No. Page fault can occur in two situations. One possibility is during insturction fetch - if the instruction pointer points to an invalid page. In this case, the missing virtual page number (EPN) corresponds to the logical page number of the EIP. The second possibility is during instruction execution when an operand fetch/memory write accesses a page that is not loaded. In this case EPN will indicate the page number of the missing page, and not the logical page number corresponding to EIP value.
```

```ad-question
title: Why does the exception handler terminate the process when the userSP value is PTLR*512-1 ?
The XSM machine doesn't push the return address into the user stack when the exception occurs, instead it stores the address in the EIP register. Hence, for the exception handler to return to the instruction which caused the exception, the EIP register value must be pushed onto the top of the user stack of the program. However, when the application's stack is full (userSP = PTLR*512-1), there is no stack space left to place the return address and the only sensible action for the OS is to terminate the process.
```

```ad-question
title: Why does the exception handler save the contents of the EIP register immediately into the kernel stack upon entry into the exception handler?
The execption handler may block for a disk read and invoke the scheduler during it's course of execution. The value of the EIP register must be stored before scheduling other processes as the current value will be overwritten by the machine if an exception occurs in another application that is scheduled in this way.
```

---
# Assignment

```ad-question
title: Assignment 1
Write an ExpL program to implement a linked list. Your program should first read an integer N, then read N intergers from console and store them in the linked list and print the linked list to the console. Run this program using shell version-I of stage 17.
```

> NOTE: FOR ASSIGNMENT 2 PUT A BREAKPOINT AND RUN IN DEBUGGER
## Implementation
1. int 9
2. mod 2
3. mod 1
4. mod 7
5. haltprog
6. link list (expl)

> File: int_9.spl
```c
// Save user stack value for later use, set up the kernel stack.

alias userSP R0;
userSP = SP;

[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16)+13]=SP;
SP=[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 11]*512-1;

// Set the MODE FLAG in the process table to system call number of exec.

[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 9] = 9;

// Get the argument (name of the file) from user stack.

alias fileName R1;
fileName = [[PTBR+2*((userSP-4)/512)]*512 + (userSP-4)%512];

// Search the memory copy of the inode table for the file, If the file is not present or file is not in XEXE format return to user mode with return value -1 indicating failure (after setting up MODE FLAG and the user stack).

// inode index value
alias i R2;
i = 0;

while(i < MAX_FILE_NUM) do
	if([INODE_TABLE + 16 * i + 1] == fileName && [INODE_TABLE + 16 * i + 0] == EXEC) then
		break;
	endif;
i = i+1;
endwhile;

if( i == MAX_FILE_NUM ) then
	[PROCESS_TABLE + 16*[SYSTEM_STATUS_TABLE+1] + 9 ] = 0;
	SP = userSP;

	[[PTBR+2*((userSP-1)/512)]*512 + (userSP-1)%512] = -1;
	ireturn;
endif;

// Call the Exit Process function in process manager module to deallocate the resources and pages of the current process.

alias exitPID R3;
exitPID = [SYSTEM_STATUS_TABLE+1];

multipush(R0,R1,R2,R3);

// EXIT_PROCESS = 3 and argument is PID
R1 = 3;
R2 = [SYSTEM_STATUS_TABLE+1];
call PROCESS_MANAGER;

multipop(R0,R1,R2,R3);

// Get the user area page number from the process table of the current process. This page has been deallocated by the Exit Process function. Reclaim the same page by incrementing the memory free list entry of user area page and decrementing the MEM_FREE_COUNT field in the system status table.

alias user_area_page R4;
user_area_page = [PROCESS_TABLE + 16 * exitPID + 11];

[MEMORY_FREE_LIST + user_area_page] = [MEMORY_FREE_LIST + user_area_page] + 1;
[SYSTEM_STATUS_TABLE + 2] = [SYSTEM_STATUS_TABLE + 2] - 1;

// Set the SP to the start of the user area page to intialize the kernel stack of the new process.

SP = user_area_page * 512 - 1;

// pre-process resource table
alias iter R6;
iter = 0;
while(iter < 16) do
	[(([PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 11] + 1) * 512 ) - 16 + iter] = -1;
	iter = iter + 1;
endwhile;

// New process uses the PID of the terminated process. Update the STATE field to RUNNING and store inode index obtained above in the inode index field in the process table.

[PROCESS_TABLE + 16 * exitPID + 4] = RUNNING;
[PROCESS_TABLE + 16 * exitPID + 7] = i;

// Allocate new pages and set the page table entries for the new process.

alias freePage R5;

// Invoke the Get Free Page function to allocate 2 stack and 2 heap pages. Also validate the corresponding entries in page table.

//Library
[PTBR+0] = 63;
[PTBR+1] = "0100";
[PTBR+2] = 64;
[PTBR+3] = "0100";

// allocating stack page 1

multipush(R0,R1,R2,R3);
R1 = GET_FREE_PAGE;
call MEMORY_MANAGER;
freePage = R0;
multipop(R0,R1,R2,R3);

[PTBR + 16] = freePage;
[PTBR + 17] = "0110";

// allocating stack page 2

multipush(R0,R1,R2,R3);
R1 = GET_FREE_PAGE;
call MEMORY_MANAGER;
freePage = R0;
multipop(R0,R1,R2,R3);

[PTBR + 18] = freePage;
[PTBR + 19] = "0110";

// Don't allocate memory pages for heap. Instead, invalidate page table entries for heap.
// allocating heap page 1

[PTBR + 4] = -1;
[PTBR + 5] = "0000";

// allocating heap page 2

[PTBR + 6] = -1;
[PTBR + 7] = "0000";

// Change the page allocation for code pages from previous stage. Invoke the Get Code Page function for the first code block and update the page table entry for this first code page.Invalidate rest of the code pages entries in the page table.

// allocation for code page 1

multipush(R0, R1, R2, R3, R4, R5);
R1 = 5;
// block number
R2 = [INODE_TABLE + (i * 16) + 8];
// pid
R3 = [SYSTEM_STATUS_TABLE+1];
call MEMORY_MANAGER;
print R0;
[PTBR + 8] = R0;
[PTBR + 9] = "0110";
multipop(R0,R1,R2,R3,R4,R5);

print "here";
// allocating rest pages of the code to be null (-1)
[PTBR + 10] = -1;
[PTBR + 11] = "0000";
[PTBR + 12] = -1;
[PTBR + 13] = "0000";
[PTBR + 14] = -1;
[PTBR + 15] = "0000";

// Initialize the disk map table of the process. The code page entries are set to the disk block numbers from inode table of the program (program given as argument to exec). Initialize rest of the entries to -1.
[DISK_MAP_TABLE + [SYSTEM_STATUS_TABLE + 1] *10 + 0] =-1;
[DISK_MAP_TABLE + [SYSTEM_STATUS_TABLE + 1] *10 + 1] =-1;
[DISK_MAP_TABLE + [SYSTEM_STATUS_TABLE + 1] *10 + 2] =-1;
[DISK_MAP_TABLE + [SYSTEM_STATUS_TABLE + 1] *10 + 3] =-1;
[DISK_MAP_TABLE + [SYSTEM_STATUS_TABLE + 1] *10 + 4] = [INODE_TABLE + (i * 16) + 8 + 0];
[DISK_MAP_TABLE + [SYSTEM_STATUS_TABLE + 1] *10 + 5] = [INODE_TABLE + (i * 16) + 8 + 1];
[DISK_MAP_TABLE + [SYSTEM_STATUS_TABLE + 1] *10 + 6] = [INODE_TABLE + (i * 16) + 8 + 2];
[DISK_MAP_TABLE + [SYSTEM_STATUS_TABLE + 1] *10 + 7] = [INODE_TABLE + (i * 16) + 8 + 3];
[DISK_MAP_TABLE + [SYSTEM_STATUS_TABLE + 1] *10 + 8] =-1;
[DISK_MAP_TABLE + [SYSTEM_STATUS_TABLE + 1] *10 + 9] =-1;


// Making the stack point to the top of the code section
[[PTBR+16]*512]=[[PTBR+8]*512+1];

// Change SP to user stack, change the MODE FLAG back to user mode and return to user mode.

SP = 8 * 512;
[PROCESS_TABLE + 16 * [SYSTEM_STATUS_TABLE+1] + 9] = 0;
ireturn;
```

> File: mod_2.spl
```c
// According to the function number value present in R1, implement different functions in module 2.


alias funcNo R1;

// If the function number corresponds to Get Free Page, follow steps below

// GET_FREE_TABLE function number is 1
if(funcNo == 1) then

	// Increment WAIT_MEM_COUNT field in the system status table.
	// Do not increment the WAIT_MEM_COUNT in busy loop

	// WAIT_MEM_COUNT field id is 3
	// The number of processes waiting for mem is increased by 1
	[SYSTEM_STATUS_TABLE + 3] = [SYSTEM_STATUS_TABLE + 3] + 1;
	
	// While memory is full (MEM_FREE_COUNT will be 0), do following.
	
	// MEM_FREE_COUNT field id is 2
	while([SYSTEM_STATUS_TABLE + 2] == 0) do
		
		// Set the state of the invoked process as WAIT_MEM.
		[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 4] = WAIT_MEM;

		// Schedule other process by invoking the context switch module. // blocking the process

		multipush(R1,R2);
		call MOD_5;
		multipop(R1,R2);

	endwhile;

	// Decrement the WAIT_MEM_COUNT field and MEM_FREE_COUNT field in the system status table. Note the sequence - increment WAIT_MEM_COUNT, waiting for the memory, decrement WAIT_MEM_COUNT.

	[SYSTEM_STATUS_TABLE + 3] = [SYSTEM_STATUS_TABLE + 3] - 1;
	[SYSTEM_STATUS_TABLE + 2] = [SYSTEM_STATUS_TABLE + 2] - 1;

	// Find a free page using memory free list and set the corresponding entry as 1. Make sure to store the obtained free page number in R0 as return value.
	alias retVal R0;
	alias i R3;
	i = 0;
	while(i < MAX_MEM_PAGE) do
		if([MEMORY_FREE_LIST + i] == 0) then
			[MEMORY_FREE_LIST + i] = 1;
			retVal = i;

			// Return to the caller.
			return;
		endif;
		i = i + 1;
	endwhile;
endif;

// If the function number corresponds to Release Page, follow steps below

// RELEASE_PAGE function number is 2
if(funcNo == 2) then
	alias pageNo R2;
	// The Page number to be released is present in R2. Decrement the corresponding entry in the memory free list.
	[MEMORY_FREE_LIST + pageNo] = [MEMORY_FREE_LIST + pageNo] - 1;
	
	// If that entry in the memory free list becomes zero, then the page is free. So increment the MEM_FREE_COUNT in the system status table.
	if([MEMORY_FREE_LIST] + pageNo == 0) then
		[SYSTEM_STATUS_TABLE + 2] = [SYSTEM_STATUS_TABLE + 2] + 1;

		// Update the STATUS to READY for all processes (with valid PID) which have STATUS as WAIT_MEM.
		alias i R3;
		i = 0;
		while(i < 16) do
			if([PROCESS_TABLE + i * 16 + 4] == WAIT_MEM) then
				[PROCESS_TABLE + i * 16 + 4] = READY;
			endif;
			i = i + 1;
		endwhile;
	endif;

	R0 = pageNo;
	// Return to the caller.
	return;
endif;

// Check the disk map tableentries ofall the processes, if the given block number is present in any entry and the corresponding page table entry is valid then return the memory page number.Also increment the memory free list entry of that page. Memory Free list entry is incrementedas page is being shared by another process.
if(funcNo == 5) then
    alias bNo R2;
    alias pid R3;
    alias i R4;
    i = 0;
    while(i < MAX_PROC_NUM)do
        alias j R7;
        alias page_entry R5;
        alias disk_entry R6;
        page_entry = [PROCESS_TABLE + i * 16 + 14 ];
        disk_entry = DISK_MAP_TABLE + i *10;
        j = 0;
        while(j < 4) do
            if([disk_entry + 4 + j] == bNo && [page_entry + 8 + j*2] != -1) then
                R0 = [page_entry + 8 + j*2];
                [MEMORY_FREE_LIST + R0] = [MEMORY_FREE_LIST + R0] + 1;
                return;
            endif;
            j = j + 1;
        endwhile;
        i = i + 1;
    endwhile;


    multipush(R1,R2,R3);
    R1 = 1;
    call MOD_2; 
    multipop(R1,R2,R3);
    multipush(R0,R1,R2,R3);
    R1 = 2;
    R4 = bNo;
    R2 = pid;
    R3 = R0;
    call MOD_4; 
    multipop(R0,R1,R2,R3);

endif;

// Decrement the count of the disk block number in the memory copy of the Disk Free List.
if(funcNo == 4) then
    alias bNo R2;
    [DISK_FREE_LIST + bNo] = [DISK_FREE_LIST + bNo] - 1;
endif;

return;
```

> File: mod_1.spl
```c
// According to the function number value present in R1, implement different functions in module 1.

alias funcNo R1;
alias currPID R2;

// If the function number corresponds to Free User Area Page, follow steps below

// FREE_USER_AREA_PAGE function number is 2
if(funcNo == 2) then

	// Obtain the user area page number from the process tableentry corresponding to the PID given as an argument.

	multipush(R1,R2);

	// Free the user area page by invoking the Release Page function.
	// RELEASE_PAGE function number is 2
	R1 = 2;
	R2 = [PROCESS_TABLE + 16 * currPID + 11];
	call MEMORY_MANAGER;

	multipop(R1,R2);

	// Return to the caller.
	return;
endif;

// If the function number corresponds to Exit Process, follow steps below

// EXIT_PROCESS function number is 3
if(funcNo == 3) then

	// Extract PID of the invoking process from the corresponding register.

	multipush(R1,R2);
	
	// Invoke the Free Page Table function with same PID to deallocate the page table entries.
	// FREE_PAGE_TABLE function number is 4
	R1 = 4;
	R2 = currPID;
	call PROCESS_MANAGER;

	multipop(R1,R2);

	// Invoke the Free user Area Page function with the same PID to free the user area page.

	multipush(R1,R2);

	// FREE_USER_AREA_PAGE function number is 2
	R1 = 2;
	R2 = currPID;
	call PROCESS_MANAGER;
	multipop(R1,R2);

	// Set the state of the process as TERMINATED and return to the caller.

	[PROCESS_TABLE + 16 * currPID + 4] = TERMINATED;
	return;
endif;

// If the function number corresponds to Free Page Table, follow steps below

// FREE_PAGE_TABLE function number is 4
if(funcNo == 4) then

	// Invalidate the page table entries for the library pages by setting page number as -1 and auxiliary data as "0000" for each entry.
	alias pageNum R3;
	// Count of lib pages is set to 2
	pageNum = 2;

	while(pageNum < 10) do
		if([PTBR + 2 * pageNum] != -1) then

			// For each valid entry in the page table, release the page by invoking the Release Page function and invalidate the entry.

			multipush(R1,R2,R3);

			// RELEASE_PAGE funtion number is 2
			R1 = 2;
			R2 = [PTBR + 2 * pageNum];
			call MEMORY_MANAGER;

			multipop(R1,R2,R3);

			// setting page number as -1 and auxiliary data as "0000" for each entry
			[PTBR + 2 * pageNum] = -1;
			[PTBR + 2 * pageNum + 1] = "0000";

			// Return to the Caller.
		endif;
		pageNum = pageNum + 1;
	endwhile;

	// Go through the heap and stack entries in the disk map table of the process with given PID. If any valid entries are found, invoke the Release Blockfunction in the memory manager module.

	alias DISK_ENTRY R4;
	alias i R5;
	DISK_ENTRY = DISK_MAP_TABLE + [SYSTEM_STATUS_TABLE + 1] * 10;
	i=2;
	while(i <4) do
		if([DISK_ENTRY + i] != -1) then
			multipush(R0,R1,R2,R3,R4,R5);
			R1 = 4;
			R2 = [DISK_ENTRY + i];
			call MOD_2;
			multipop(R0,R1,R2,R3,R4,R5);
		endif;
		i = i + 1;
	endwhile;
	
	// Invalidate all the entries of the disk map table.
	[DISK_ENTRY + 0] = -1;
	[DISK_ENTRY + 1] = -1;
	[DISK_ENTRY + 2] = -1;
	[DISK_ENTRY + 3] = -1;
	[DISK_ENTRY + 4] = -1;
	[DISK_ENTRY + 5] = -1;
	[DISK_ENTRY + 6] = -1;
	[DISK_ENTRY + 7] = -1;
	[DISK_ENTRY + 8] = -1;
	[DISK_ENTRY + 9] = -1;	

	return;
endif;
```

> File: haltprog.spl
```c
[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 9] = -1;
[PROCESS_TABLE + ( [SYSTEM_STATUS_TABLE + 1] * 16) + 13] = SP;				//Save the current value of User SP into the corresponding Process Table entry.

SP = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 11] * 512 - 1;			// Setting SP to UArea Page number * 512 - 1
backup;
multipush(EIP);
alias userSP R1;
userSP = [PROCESS_TABLE + ( [SYSTEM_STATUS_TABLE + 1] * 16) + 13];

if(EC != 0 || userSP == [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 15] * 512 - 1) then
	print "ERROR";
	if (EC == 1) then
		//print EIP;
		//print R0;
		print "ILLEGAL INSTRUCTION";
	else
		if(EC == 2) then
			print "ILLEGAL MEMORY ACCESS";
		else
			print "ARITHMETIC EXPRESSION";
		endif;
	endif;
	
	
	//invoke exit process
	//print "exit";
	multipush(R0);
	R1 = 3;
	R2 = [SYSTEM_STATUS_TABLE + 1];
	call MOD_1;
	multipop(R0);
	
	//print R0;
	//print "call scheduler";
	call MOD_5;
	//print "exit";

else
	print "page fault";
	//for heap page
	alias i R1;
	alias ptbr R2;
	ptbr = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 14];
	if(EPN == 2 || EPN == 3) then
		print "heap";
		i=0;
		while(i<2) do
			multipush(R1, R2);
			R1 = 1;
			call MOD_2;
			multipop(R1, R2);
			
			[ptbr + 2*i + 4] = R0;
			[ptbr + 2*i + 5] = "0110";
			i = i + 1;
		endwhile;
	endif;
	
	
	
	//for code page
	if ( EPN >= 4 && EPN <=7) then
		//print EPN;
		print "code";
		multipush(R0, R1, R2);
		R1 = 5;
		R2 = [DISK_MAP_TABLE + ([SYSTEM_STATUS_TABLE + 1]*10) + EPN];
		call MOD_2;
		//R3 = R0;
		multipop(R0, R1, R2);
		[ptbr + EPN*2] = R0;
		[ptbr + EPN*2 + 1] = "1100";
	endif;
endif;
	
//print "doe";
[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 9] = 0;
multipop(EIP);
restore;
	//increment SP, store EIP, to location pointed by SP
SP = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 13];
SP = SP + 1;
[[[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 14] + 2*((SP)/512)] * 512 + ((SP)%512)] = EIP;

ireturn;
```

> File: mod_7.spl
```c
//load exception handler
loadi(2, 15);
loadi(3, 16);

//load the timer interupt routine
loadi(4, 17);
loadi(5, 18);

//load disk interrupt
loadi(6, 19);
loadi(7, 20);

//load console interrupt
loadi(8,21);
loadi(9,22);

//load INT 6
loadi(14,27);
loadi(15,28);

//load int7
loadi(16,29);
loadi(17,30);

//load int 9
loadi(20,33);
loadi(21,34);

//load int10 module
loadi(22,35);
loadi(23,36);

//module 0
loadi(40, 53);
loadi(41, 54);

// load  module 1, 
loadi(42, 55);
loadi(43, 56);

// load module 2 
loadi(44, 57 );
loadi(45, 58 );

//module 4
loadi(48, 61);
loadi(49, 62);

//load inode table
loadi(59,3);
loadi(60,4);

//scheduler - mod5
loadi(50,63);
loadi(51,64);

//load disk map table
loadi(61, 2);

//load library code form disk to memory
loadi(63,13);
loadi(64,14);

//load init program
loadi(65,7);
loadi(66,8);





//---------------------------------------------------->init program<--------------------------------------------

PTBR = PAGE_TABLE_BASE + 20;
PTLR = 10;

//library - page 0 and 1
[PTBR+0] = 63;
[PTBR+1] = "0100";
[PTBR+2] = 64;
[PTBR+3] = "0100";

//heap - page 2 and 3
[PTBR+4] = 78;
[PTBR+5] = "0110";
[PTBR+6] = 79;
[PTBR+7] = "0110";

//page 4 and 5 - code
[PTBR+8] = 65;
[PTBR+9] = "0100";
[PTBR+10] = 66;
[PTBR+11] = "0100";

[PTBR+12] = -1;
[PTBR+13] = "0000";
[PTBR+14] = -1;
[PTBR+15] = "0000";


//page 8 and 9- stack
[PTBR+16] = 76;
[PTBR+17] = "0110";
[PTBR+18] = 77;
[PTBR+19] = "0110";


//PROCESS TABLE FOR INIT PID =1
[PROCESS_TABLE + 16 + 1] = 1;
[PROCESS_TABLE + 16 + 4] = CREATED;
[PROCESS_TABLE + 16 + 11] = 80;   //UAreaPgNo.
[PROCESS_TABLE + 16 + 12] = 0;    //KPTR
[PROCESS_TABLE + 16 + 13] = 8*512;    //UPTR : LOGICAL ADDRESS OF userSP
[PROCESS_TABLE + 16 + 14] = PTBR;
[PROCESS_TABLE + 16 + 15] = PTLR;


[76*512] = [65 * 512 + 1];


//initialise memory free list
alias i R1;
i = 0;
while(i<83) do
  [MEMORY_FREE_LIST + i] = 1;
  [SYSTEM_STATUS_TABLE+3] = [SYSTEM_STATUS_TABLE+3]  - 1;
  i = i + 1;
endwhile;

while(i<128) do
  [MEMORY_FREE_LIST + i] = 0;
  i = i + 1;
endwhile;

[DISK_STATUS_TABLE] = 0;
[TERMINAL_STATUS_TABLE] = 0;
[SYSTEM_STATUS_TABLE + 3] = 0;
[SYSTEM_STATUS_TABLE + 2] = 45;

alias perProcessTableEntry R1;
alias i R2;
i = 0;
while(i<16) do 
    perProcessTableEntry = (([PROCESS_TABLE + (1*16) + 11] + 1) * 512 ) - 16 + i;
    [perProcessTableEntry] = -1;
    i = i + 1;
endwhile;


//initialise disk map table
i = 0;
while(i < 10)do
    [DISK_MAP_TABLE+1*10+i] = -1;
    i = i + 1;
endwhile;
// code blocks
[DISK_MAP_TABLE+1*10+4] = 7;
[DISK_MAP_TABLE+1*10+5] = 8;


i = 2;
while(i < 16) do
    [PROCESS_TABLE + i * 16 + 4] = TERMINATED;
    i = i + 1;
endwhile;

return;
```

## Compile and Load

```baah
$> ./compile.sh stage19
```

![[Pasted image 20230923011653.png]]

> File: load19.dat
```.
load --library ../expl/library.lib
load --idle /home/kali/myexpos/expl/expl_progs/stage19/infinite.xsm
load --init /home/kali/myexpos/expl/expl_progs/stage19/name.xsm
load --exec /home/kali/myexpos/expl/expl_progs/stage19/link_l.xsm
load --int=10 /home/kali/myexpos/spl/spl_progs/stage19/int_10.xsm
load --int=7 /home/kali/myexpos/spl/spl_progs/stage19/sample_int7.xsm
load --int=6 /home/kali/myexpos/spl/spl_progs/stage19/sample_int6.xsm
load --int=9 /home/kali/myexpos/spl/spl_progs/stage19/int_9.xsm
load --int=console /home/kali/myexpos/spl/spl_progs/stage19/console.xsm
load --int=disk /home/kali/myexpos/spl/spl_progs/stage19/disk.xsm
load --module 0 /home/kali/myexpos/spl/spl_progs/stage19/mod_0.xsm
load --module 5 /home/kali/myexpos/spl/spl_progs/stage19/mod_5.xsm
load --module 4 /home/kali/myexpos/spl/spl_progs/stage19/mod_4.xsm
load --module 7 /home/kali/myexpos/spl/spl_progs/stage19/mod_7.xsm
load --module 1 /home/kali/myexpos/spl/spl_progs/stage19/mod_1.xsm
load --module 2 /home/kali/myexpos/spl/spl_progs/stage19/mod_2.xsm
load --exhandler /home/kali/myexpos/spl/spl_progs/stage19/haltprog.xsm
load --int=timer /home/kali/myexpos/spl/spl_progs/stage19/sample_timer.xsm
load --os /home/kali/myexpos/spl/spl_progs/stage19/os_startup.xsm
exit
```

## Run xsm
```bash
$> ./xfs-interface < ../test/load_files/load19.dat
$> ./xsm
Input file:
link_l.xsm
76
here
page fault
heap
5
1
2
3
4
5
1
2
3
4
5
Machine is halting.
```

![[Pasted image 20230923011824.png]]













