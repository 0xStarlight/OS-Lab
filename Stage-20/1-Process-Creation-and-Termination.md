# Summary

The fork system call creates a new process with a child-parent relationship. The child has a different PID and address space, but shares the code and heap regions with the parent. The child is allocated two new stack pages and a new user area page. The child process table is initialized with the same values as the parent except for some fields. The parent's stack contents are copied into the new stack pages allocated for the child. The contents of the per process resource table in the user area page of the parent process are copied to the child process. The Alloc() function of eXpOS library allocates memory from the heap region of a process, and memory allocated by Alloc() to store objects referenced by variables of user-defined types in an ExpL program will be allocated in the heap. The parent and child share memory allocated using Alloc(). The OS provides support for synchronization through system calls for semaphores and signal handling.

---
# Starting

In this stage, you will learn how to create a new process using the fork system call and how to terminate a process using the exit system call.
## Fork System Call[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-20/#fork-system-call "Permanent link")

The Fork system call spawns a new process. The new process and the process which invoked fork have a child-parent relationship. The child process will be allocated a different PID and a new [address space](https://exposnitc.github.io/expos-docs/abi/#virtual-address-space-model). Hence, the child process will have a different process table and page table. However, the child and the parent will share the code and heap regions of the address space. The child will be allocated two new stack pages and a new user area page.

The process table of the child is initialized with the same values of the parent except for the values of TICK, PID, PPID, USER AREA PAGE NUMBER, KERNEL STACK POINTER, INPUT BUFFER, MODE FLAG, PTBR and PTLR.The contents of the stack of the parent are copied into the new stack pages allocated for the child. The contents of the per process resource table in the [user area page](https://exposnitc.github.io/expos-docs/os-design/process-table/#user-area) of the parent process is copied to the child process. However, the contents of the parent's kernel stack are not copied to the child, and the kernel stack of the child is set to empty (that is, KPTR field in the process table entry of the child is set to 0.)

Fork system call returns to the the parent process. The parent resumes execution from the next instruction following the INT instruction invoking fork. Upon successful completion, fork returns the PID of the child process to parent process.

After completion of fork, the child process will be ready for execution and will be in the CREATED state. When the child process is scheduled (by the scheduler) to run for the first time, it will start its execution from the immediate instruction after the call to fork. The return value of fork to the child process is zero. We have already noted that the child process shares the heap and code pages with the parent process, whereas new memory pages are allocated for user stack of the child. The contents of the parent's user stack pages are copied to the user stack of the child process. **ExpL compiler allocates local variables, global variables and arrays of primitive data types (int, string) of a process in the stack. Since the parent and child processes have different memory pages for the user stack, they resume after fork with separate private copies of these variables, with the same values.** Since the stack page is not shared between the parent and the child, subsequent modifications to these variables by either parent or the child will not be visible to the other.

The [Alloc()](https://exposnitc.github.io/expos-docs/os-spec/dynamicmemoryroutines/#alloc)function of eXpOS library allocates memory from the heap region of a process. Hence memory allocated by _Alloc()_ to store objects referenced by variables of user defined types in an ExpL program will be allocated in the heap. As heap pages are shared by the parent and the child, both processes share the memory allocated using _Alloc()_. Thus,**if the parent had allocated memory using the Alloc() function and attached it to a variable of some user defined type before fork, the copies of the variables in both the parent and the child store the address of the same shared memory.** **Since the Parent and child processes can concurrently access/modify the heap pages, they need support from the OS to synchronize access to the shared heap memory.**eXpOS provides support for such synchronization through systems calls for **semaphores** and **signal handling**. These will be discussed in later stages. Even though, code and library pages are shared among parent and child processes, synchronization is not required for these pages as their access is read only.

```ad-note
There is a subtility to be noted here. It can happen that the kernel has not allocated any heap pages for the parent process at the time when it invoked Fork. This is because heap pages are allocated inside the exception handler when a process generates a page fault exeception while trying to access the heap. Thus a process could invoke fork before any heap pages were allocated to it. However, the eXpOS sharing semantics requires that the parent and the child shares their heap pages. Hence, to satisfy the heap sharing semantics, the OS needs to ensure during Fork  
that the parent is allocated its heap pages and these pages are shared with the child.
```

![[Pasted image 20230923201729.png]]

High level ExpL programs can invoke _fork_ system call using the library interface function [exposcall](https://exposnitc.github.io/expos-docs/os-spec/dynamicmemoryroutines/). Fork has system call number 8 and it is implemented in the interrupt routine 8. Fork does not take any arguments. Follow the description below to implement the fork system call.
### How to implement fork

The fork system call creates a new process with a child-parent relationship. The first step is to set the MODE FLAG to the system call number and switch to the kernel stack. To get a new PID for the child process, invoke the Get Pcb Entry function from the process manager module. If a free process table is not available, Get Pcb Entry returns -1. If PID is available, proceed with the fork system call. If the heap pages are not allocated for the parent process, allocate heap pages by invoking the Get Free Page function of the memory manager module. The next step in the fork system call is initialization of the process table for the child process. Set the MODE FLAG, KPTR and TICK fields of the child process to 0. PID of the parent is stored in the PPID field of the process table of the child. STATE of the child process is set to CREATED. Copy the entries of the per-process resource table of the parent to the child. Copy the per-process disk map table of the parent to the child. Initialize the page table of the child process. Store the value in the BP register on top of the kernel stack of child process. Set up return values in the user stacks of the parent and the child processes.
## Exit System Call[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-20/#exit-system-call "Permanent link")

At the end of every process,_exit_ system call is invoked to terminate the process. In the previous stages, we had already implemented the _exit_ system call. Now, we will change the _exit_ system call to invoke the module function Exit Process to terminate the process.

_Exit_ system call has to update/clear the OS data structures of the terminating process and detach the memory pages allocated to the process from its address space. The page table entries are invalidated. There may be other processes waiting for this process to terminate. Exit system call must wake up these processes. (Currently, processes do not wait, we will see this in the next stage.) _Exit_has to close the files and release the semaphores acquired by the process. Per-process resource table has to be invalidated. Finally,_Exit_has to set the state of the process to TERMINATED. These tasks are done by invoking the **ExitProcess** function of the [process manager module](https://exposnitc.github.io/expos-docs/modules/module-01/).

![[Pasted image 20230923201759.png]]


---

# Questions

```ad-question
title: How does eXpOS prevent a program running in user mode from writing to the library and code pages?
In the page table of every process, pages of a process have auxiliary information associated with them. Auxiliary information consists of 4 bits of which third bit is write bit. If write bit is set to 1 for a page, then the process (to which the page belongs) has permission to write to the corresponding page. When write bit is set to 0, the page has read only access. So the process can not modify the content of the page. For every process, write bit is set to 0 for library and code pages while initializing page table. When a process tries to write to a page for which write bit is 0, XSM machine raises [illegal memory access](https://exposnitc.github.io/expos-docs/tutorials/xsm-interrupts-tutorial/#exception-handling-in-xsm)exception. Refer about auxiliary information [here](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-page-table).
```

```ad-question
title: Where does ExpL allocate memory for variables of user defined data type?
Variables of user defined data type are allocated memory in stack, same as any variable of primitive data type. Every variable in ExpL is allocated one word memory in stack. Variable of primitive data type saves actual data in the word that is allocated to it, but variables of user defined data type stores the starting address of the object. Alloc() library function allocates memory for an object in heap and returns the starting address. This return address is stored in the stack for corresponding variable.
```

```ad-question
title: Upon completion of the fork system call, the parent and the child will contain the same return IP address on the top of the user stack. The value of the user stack pointer (UPTR) will also be the same for both the processes. When the fork sytem call returns to user mode (using the IRET instruction) which process executes first - parent or child? why?
Fork system call returns to the parent process. IRET sets the value of IP register to the return address at the top of the stack, pointed to by the SP register. The machine translates the logical SP to physical SP using the page table pointed to by the PTBR register, which points to the page table of the parent. Subsequent instruction fetch cycles continue to proceed by translating the value of IP using the PTBR value, which points to the page table of the parent process. The parent process continues execution till a context switch occurs.
```

```ad-question
title: What would go wrong if the Get Pcb Entry function sets the state of a newly allocated PCB entry to CREATED instead of ALLOCATED?
The fork system call which invokes the Get Pcb Entry function for the child process might block before completing process creation (if the Get Free Page function finds no free page in memory and invokes the scheduler). In such case, the scheduler must not try to run the new process as its creation is not complete.
```

```ad-question
title:  When there is a page fault for a heap page, the exception handler which you have completed in stage 19 allocates both the heap pages for the process. Suppose you modify the exception handler to allocate only one heap page corresponding to the page requested (deferring the allocation of the second page till another exception occurs), what modification would be needed to your Fork system call code?
eXpOS semantics requires that the parent and the child share the heap. Hence, if the parent did not have two heap pages already allocated before Fork, the pages must be allocated at the time of Fork and shared with the child. An alternate approach would be to share only the presently allocated heap page (if any) of the parent with the child and wait for a page fault in either of the processes to allocate the other page. However, in this case, care must be taken to ensure that the same page is shared between the parent and the child when one of them is allocated a heap page.
```

---

# Assignments

```ad-question
title: Assignment 1
Write two ExpL programs even.expl and odd.expl to print the first 100 even and odd numbers respectively. Write another ExpL program that first creates a child process using Fork. Then, the parent process shall use the exec system call to execute even.xsm and the child shall execute odd.xsm. Load this program as the init program.
```

## Modifications/Implementation
1. int_8.spl
2. int_10.spl
3. mod_1.spl
4. mod_5.spl
5. mod_7.spl
6. fork.expl
7. link_100f.expl

> File: int_8.spl
```c
// Follow the description below to implement the fork system call.

// The first action to perform in the fork system call is to set the MODE FLAG to the system call number and switch to the kernel stack.

// Fork Syscall number is 8
[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1]*16 + 9] = 8;

alias pTable R8;
pTable = PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16);

alias userSP R2;
userSP = SP;

//Save the current value of User SP into the corresponding Process Table entry.
[pTable + 13] = SP;
PTBR = [pTable + 14];

// Setting SP to UArea Page number * 512 - 1
SP = [pTable + 11] * 512 -1;
PTLR = [pTable + 15];

// To get a new PID for the child process, invoke the Get Pcb Entryfunction from the process manager module.

multipush(R2);
// GET_PCB_ENTRY = 1
R1 = 1;
// MOD 1
call PROCESS_MANAGER;
multipop(R2);

// STEP 1
// Get Pcb Entry returns the index of the new process table allocated for the child. This index is saved as the PID of the child. As there are only 16 process tables present in the memory, maximum 16 processes can run simultaneously. If a free process table is not available, Get Pcb Entry returns -1. In such case, store -1 as the return value in the stack, reset the MODE FLAG (to 0), switch to user stack and return to the user mode from the fork system call. When PID is available, proceed with the fork system call.

if(R0 == -1) then
	// No entry available in PCB Entry
	// Resetting the MODE FLAG
	[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1]*16 + 9] = 0;
	// Store -1 as the return value in the stack
	[[PTBR + 2*((userSP - 1)/512)] * 512 + ((userSP - 1)%512)] = -1;
	// Return to user mode
	SP = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16)+13];
	ireturn;
endif;

// STEP 2
// If the heap pages are not allocated for the parent process, allocate heap pages by invoking the Get Free Page function of the memory manager module and set the page table entries for the heap pages of the parent process to the pages acquired. - The child process requires new memory pages for stack (two) and user area page (one). To allocate a memory page, invoke the Get Free Pagefunction of the memory manager module.

alias i R3;
i = 0;
while(i < 2) do
	// checking if the heap pages are allocated or not
	// If not allocated we can allocate them
	if([PTBR + 4 + 2*i] == -1) then
		multipush(R0,R1,R2,R3);
		// Invoke GET_FREE_PAGE
		R1 = 1;
		call MEMORY_MANAGER;
		[PTBR + 4 + 2*i] = R0;
		[PTBR + 5 + 2*i] = "0110";
		multipop(R0,R1,R2,R3);
	endif;
	i = i+1;
endwhile;


// STEP 3
// The next step in the fork system call is initialization of the process table for the child process. 

alias childPID R1;
childPID = R0;
alias childPTBR R2;
alias childPT R3;

// initialization of the process table for the child process
childPT = PROCESS_TABLE + childPID*16;
childPTBR = [childPT + 14];

// allocate the stack page to child process
alias i R4;
i = 0;
while(i < 2) do
	multipush(R0,R1,R2,R3,R4);
	// Invoke GET_FREE_PAGE
	R1 = 1;
	call MEMORY_MANAGER;
	[childPTBR + 16 + 2*i] = R0;
	[childPTBR + 17 + 2*i] = "0110";
	multipop(R0,R1,R2,R3,R4);
	i = i+1;
endwhile;

// allocate the userArea Page
multipush(R0,R1,R2,R3);
// Invoke GET_FREE_PAGE
R1 = 1;
call MEMORY_MANAGER;
[PROCESS_TABLE + childPID*16 + 11] = R0;
multipop(R0,R1,R2,R3);

// Copy the USERID field from the process table of the parent to the child process, as the user (currently logged in) will be same for both child and parent. (We will discuss "USERID" later when we add multi-user support to eXpOS.) 
// Similarly, copy the "SWAP FLAG" and the "USER AREA SWAP STATUS" fields. (These fields will be discussed later, when we discuss swapping out processes from memory to the disk.) "INODE INDEX" for the child and the parent processes will be same, as both of them run the same program. "UPTR" field should also be copied from the parent process. As mentioned earlier, content of the user stack is same for both of them, so when both of the processes resume execution in user mode, the value of SP must be the same.

// Userid Falg
[childPT + 3] = [pTable + 3];

// Swap Flag
[childPT + 6] = [pTable + 6];

// Inode Index Flag
[childPT + 7] = [pTable + 7];

// User Area Swap Flag
[childPT + 10] = [pTable + 10];

// UPTR
[childPT + 13] = [pTable + 13];

// STEP 4
// Set the MODE FLAG, KPTR and TICK fields of the child process to 0. MODE FLAG, KPTR are set to zero as the child process starts its execution from the user mode. The TICK field keep track of how long a process has been running in memory and should be initialized to 0, when a process is created. (The TICK field will be used later to decide which process must be swapped out of memory when memory is short. The strategy will be to swap out that process which had been in memory for the longest time). PID of the parent is stored in the PPID field of the process table of the child. STATE of the child process is set to CREATED. Store the new memory page number obtained for user area page in the USER AREA PAGE NUMBER field in the process table of the child proces. PID, PTBR and PTLR fields of the child process are already initialized in the Get Pcb Entryfunction. It is not required to initialize INPUT BUFFER.

// PPID Flag
[childPT + 2] = [SYSTEM_STATUS_TABLE + 1];

// Tick Flag
[childPT + 0] = 0; 

// Mode Flag
[childPT + 9] = 0; 

// KPTR Flag
[childPT + 12] = 0; 

// State Flag
[childPT + 4] = CREATED; 

// STEP 5
// The per-process resource table has details about the open instances of the files and the semaphores currently acquired by the process. Child process shares the files and the semaphores opened by the parent process. Hence we need to copy the entries of the per-process resource table of the parent to the child. We will discuss files and semaphores in later stages. (There is a little bit more book keeping work associated with files and semaphores. Since we have not added files or semaphores so far to the OS, we will skip this work for the time being and complete the pending tasks in later stages).

// parent pre-process table
alias pppTable R4;
// child pre-process table
alias cppTable R5;
alias i R6;
i=0;
while(i<16) do
	pppTable = (([pTable + 11] + 1)* 512) + ((2*i) - 16);
	cppTable = (([childPT + 11] + 1)* 512) + ((2*i) - 16);
	[cppTable] = [pppTable];
	i=i+1;
endwhile;

// STEP 6
// Copy the per-process disk map table of the parent to the child. This will ensure that the disk block numbers of the code pages of the parent process are copied to the child.Further, if the parent has swapped out heap pages, those will be shared by the child. (This will be explained in detail in a later stage). The eXpOS design guarentees that the stack pages and the user area page of a process will not be swapped at the time when it invokes the fork system call. Hence the disk map table entries of the parent process corresponding to the stack and user area pages will be invalid, and these entries of the child too must be set to invalid.

// parent Disk Map Table
alias pDMT R4;
// child Disk Map Table
alias cDMT R5;
i=0;
while(i<10) do
	pDMT = DISK_MAP_TABLE + [SYSTEM_STATUS_TABLE + 1] * 10 + i;
	cDMT = DISK_MAP_TABLE + childPID * 10 + i;
	[cDMT] = [pDMT];
	i=i+1;
endwhile;

// STEP 7
// Initialize the page tableof the child process. As heap, code and library pages are shared by the parent process and the child process, copy these entries (page number and auxiliary information) form the page table of the parent to the child. For each page shared, increment the corresponding share count in the memory free list(why do we do need to do this?). Initialize the stack page entries in the page table with the new memory page numbers obtained earlier. Note that the auxiliary information for the stack pages is same for both parent and child (why?). Copy content of the user stack pages of the parent to the user stack pages of the child word by word.

// parent lib page
alias libraryPageP R4;
// child lib page
alias libraryPageC R5;
i=0;
while(i<2) do
	libraryPageP = PTBR + i*2;
	libraryPageC = childPTBR + i*2;
	[libraryPageC] = [libraryPageP];
	[libraryPageC + 1] = [libraryPageP + 1];
	[MEMORY_FREE_LIST + [libraryPageP]] = [MEMORY_FREE_LIST + [libraryPageP]] + 1;
	i = i + 1;
endwhile;

i=0;
while(i<2) do
	[childPTBR+4+i*2] = [PTBR+4+i*2];
	[childPTBR+4+i*2+1] = [PTBR+4+i*2+1];
	[MEMORY_FREE_LIST + [PTBR + 4+i*2]] = [MEMORY_FREE_LIST + [PTBR + 4+i*2]] + 1;
	i = i + 1;
endwhile;

alias codePageP R4;
alias codePageC R5;
i=0;
while(i<4) do
	codePageP = PTBR + (8 + (2*i));
	codePageC = childPTBR + (8 + (2*i));
	[codePageC] = [codePageP];
	[codePageC + 1] = [codePageP + 1];
	
	if([codePageP] != -1) then
		[MEMORY_FREE_LIST + [codePageP]] = [MEMORY_FREE_LIST + [codePageP]] + 1;
	endif;
	
	i = i + 1;
endwhile;

// parent stack
alias stackP R4;
// child stack
alias stackC R5;
i=0;
alias j R7;
j=0;
while(i<2) do
	j=0;
	while(j<512) do
		stackP = PTBR + (16 + (2*i));
        	stackC = childPTBR + (16 + (2*i)); 
        	[[stackC]*512+j] = [[stackP]*512+j];
        	j = j+1;
    	endwhile;
    i = i+1;
endwhile;

// STEP 8
// Store the value in the BP register on top of the kernel stack of child process. This value will be used to initialize the BP register of the child process by the scheduler when the child is scheduled for the first time.

[[childPT + 11] * 512] = BP;

// STEP 9
// Set up return values in the user stacks of the parent and the child processes. Store the PID of the child process as return value to the parent and 0 as the return values to the child. Reset the MODE FLAG of the parent process. Switch to the user stack of the parent process and return to the user mode.

[[childPT + 11] * 512] = BP;

alias cSP R5;
cSP = [childPT+13];
[[childPTBR + 2 * ((cSP-1)/ 512)] * 512 + ((cSP-1) % 512)] = 0;

alias pSP R5;
pSP = [pTable+13];
[[PTBR + 2 * ((pSP-1)/ 512)] * 512 + ((pSP-1) % 512)] = childPID;

[pTable + 9] = 0;

SP = [pTable + 13];
ireturn;

```

> File: int_10.spl
```c
[ PROCESS_TABLE+[SYSTEM_STATUS_TABLE + 1]*16+4] = TERMINATED;
//sets the MODE FLAG to 10
[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 9] = 10;   

// switch to kernel stack 
[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 13] = SP;
SP = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 11] * 512  - 1 ;

// exit function number
R1 = 3;           
R2 = [SYSTEM_STATUS_TABLE + 1]; 
call PROCESS_MANAGER; 
call SCHEDULER; 

```

>File: mod_1.spl
```c
// According to the function number value present in R1, implement different functions in module 1.

alias funcNo R1;
alias currPID R2;

// If the function number corresponds to Get Pcb Entry, follow steps below
// GET_PCB_ENTRY function number is 2
if(funcNo == 1) then
	// The Get Pcb Entry function in the process manager, finds out a free process table entry and returns the index of it to the caller. If no process table entry is free, it returns -1. A free process table entry (STATEfield is set to TERMINATED) can be found out by looping through all process table entries. 
	alias PID R0;
	alias i R3;
	alias state R4;

	PID = -1;
	i = 0;

	while(i < 16) do
		state = [PROCESS_TABLE + i * 16 + 4];
		if(state == TERMINATED) then
			// Initialize the PID to the index of the free entry. Set the STATE to ALLOCATED. Initialize PTBR to the starting address of the page table for that process (obtained using index) and PTLR to 10. Return the index to the caller.
			alias pTable R5;
			pTable = PROCESS_TABLE + i * 16;
			[pTable + 1] = i;
			[pTable + 4] = ALLOCATED;
			[pTable + 14] = PAGE_TABLE_BASE + (i * 20);
			[pTable + 15] = 10;
			PID = i;
			break;
		endif;
		i = i+1;
	endwhile;
	return;
endif;

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

> File: mod_5.spl
```c
alias currentPID R0;
currentPID = [SYSTEM_STATUS_TABLE+1];

multipush(BP);

alias process_table_entry R1;
process_table_entry = PROCESS_TABLE + currentPID * 16;

[process_table_entry + 12] = SP % 512;
[process_table_entry + 14] = PTBR;
[process_table_entry + 15] = PTLR;

alias pid_last R2;
pid_last=currentPID;

currentPID=(currentPID+1)%16;
process_table_entry = PROCESS_TABLE + currentPID * 16;

alias newPID R3;
newPID=-1;
while([process_table_entry + 4]!=READY && [process_table_entry + 4]!=CREATED) do
 currentPID=(currentPID+1)%16;
 process_table_entry = PROCESS_TABLE + currentPID * 16;
 if([process_table_entry + 4]==READY || [process_table_entry + 4]==CREATED)then
  break;
 endif;
 if(currentPID==pid_last) then
  newPID=0;
  break;
 endif;
endwhile;
if(newPID==-1)then
 newPID=currentPID;
endif;

alias new_process_table R4;
new_process_table = PROCESS_TABLE + newPID * 16;

//Set back Kernel SP, PTBR , PTLR
SP =  [new_process_table + 11] * 512 + [new_process_table + 12] ;
PTBR = [new_process_table + 14];
PTLR = [new_process_table + 15];

[SYSTEM_STATUS_TABLE + 1] = newPID;

if([new_process_table + 4] == CREATED) then
 [new_process_table + 4] = RUNNING;
 SP = [new_process_table + 13];
 //MODE=0
 [new_process_table + 9] = 0;
 // new code
 BP = [[new_process_table + 11] * 512];
 ireturn;
endif;

[new_process_table + 4] = RUNNING;

multipop(BP);

return;

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

//load int8
loadi(18,31);
loadi(19,32);

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

> File: even.expl
```c
int main()
{
decl
    int temp,num,check;
    str a;
enddecl
begin
    num=1;
    a="even";
    while ( num <= 100 ) do
    	 check = num%2;
    	 if( check == 0) then
            write(a);
	         write(num);
         endif;
         num = num + 1;
    endwhile;
    return 0;
end
}

```

> File: odd.expl
```c
int main()
{
decl
    int temp,num,check;
    str a;
enddecl
begin
    num=1;
    a = "odd";
    while ( num <= 100 ) do
    	 check = num%2;
    	 if( check != 0) then
            write(a);
	        write(num);
         endif;
         num = num + 1;
    endwhile;
    return 0;
end
}
```

> File: fork.expl
```c
int main()
{
	decl
		str name, fileName;
		int temp, pid;
	enddecl

	begin

		pid = exposcall("Fork");
	
		if (pid == 0) then
		
			fileName = "child";
		else
		
			fileName = "parent";
		endif;

		temp = exposcall("Write", -2, fileName);
		//breakpoint;
		//fileName = "Breaky";
		temp = exposcall("Read", -1, name);
		//breakpoint;
		temp = exposcall("Exec",name);
		return 0;
	end
}

// fork(parent) -> child(fork)
// 				-> even
// 			 -> parent(fork)
// 			 	 -> odd

```

## Compile and Execute

```bash
$> ./compile.sh stage20
```

![[Pasted image 20230923190117.png]]

> File: load20.dat
```c
load --library ../expl/library.lib
load --idle /home/kali/myexpos/expl/expl_progs/stage20/odd.xsm
load --init /home/kali/myexpos/expl/expl_progs/stage20/fork.xsm
load --exec /home/kali/myexpos/expl/expl_progs/stage20/even.xsm
load --int=6 /home/kali/myexpos/spl/spl_progs/stage20/int_6.xsm
load --int=7 /home/kali/myexpos/spl/spl_progs/stage20/int_7.xsm
load --int=8 /home/kali/myexpos/spl/spl_progs/stage20/int_8.xsm
load --int=9 /home/kali/myexpos/spl/spl_progs/stage20/int_9.xsm
load --int=10 /home/kali/myexpos/spl/spl_progs/stage20/int_10.xsm
load --int=console /home/kali/myexpos/spl/spl_progs/stage20/console.xsm
load --int=disk /home/kali/myexpos/spl/spl_progs/stage20/disk.xsm
load --module 0 /home/kali/myexpos/spl/spl_progs/stage20/mod_0.xsm
load --module 1 /home/kali/myexpos/spl/spl_progs/stage20/mod_1.xsm
load --module 2 /home/kali/myexpos/spl/spl_progs/stage20/mod_2.xsm
load --module 4 /home/kali/myexpos/spl/spl_progs/stage20/mod_4.xsm
load --module 5 /home/kali/myexpos/spl/spl_progs/stage20/mod_5.xsm
load --module 7 /home/kali/myexpos/spl/spl_progs/stage20/mod_7.xsm
load --exhandler /home/kali/myexpos/spl/spl_progs/stage20/haltprog.xsm
load --int=timer /home/kali/myexpos/spl/spl_progs/stage20/timer.xsm
load --os /home/kali/myexpos/spl/spl_progs/stage20/os_startup.xsm
exit
```

```bash
$> ./xfs-interface < ../test/load_files/load20.dat
$> ./xsm
odd
parent
child
1
even.xsm
odd.xsm
odd
86
here
3
even
odd
2
5
even
odd
4
7
even
odd
6
9
even
odd
8
11
even
odd
10
13
even
odd
12
15
...
```

![[Pasted image 20230923190414.png]]

---

# Assignment 2

```ad-question
title: Assignment 2
Write an ExpL program which creates linked list of the first 100 numbers. The program then forks to create a child so that the parent and the child has separate pointers to the head of the shared linked list. Now, the child prints the 1st, 3rd, 5th, 7th... etc. entries of the list whereas the parent prints the 2nd, 4th, 6th, 8th....etc. entries of the list. Eventually all numbers will be printed, but in some arbitrary order (why?). The program is given [here](https://exposnitc.github.io/expos-docs/test-programs/#test-program-3). Try to read and understand the program before running it. Run the program as the INIT program. In the next stages, we will see how to use the sychronization primitives of the OS to modify the above program so that the numbers are printed out in sequential order.
```

> File: link_100.expl
```c
type
List
{
    int data;
    List next;
}
endtype

decl
    List head;
    int x, pid, temp, length;
enddecl

int main()
{
decl
    List p, q;
    int i;
enddecl

begin
    temp = exposcall("Heapset");
    head = null;
    q = head;

    length=1;
    while (length <= 100)  do
        p = exposcall("Alloc",2);
        p.data = length;
        p.next = null;

        if (head == null) then
            head = p;
            q = p;
        else
            q.next = p;
            q = q.next;
        endif;

        length = length+1;
    endwhile;

    pid = exposcall("Fork");
    if(pid == 0) then
        p = head;
        while(p != null)  do
            x = p.data;
            temp = exposcall("Write",-2,x);
            p = p.next;
            if(p == null) then
                break;
            endif;
            p = p.next;
        endwhile;
    else
        q = head.next;
        while(q != null)  do
            x = q.data;
            temp = exposcall("Write",-2,x);
            q = q.next;
            if(q == null) then
                break;
            endif;
            q = q.next;
        endwhile;
    endif;

    return 0;
end
}
```

> File: infinite.expl
```c
int main()
{
decl
    int temp,num,check;
enddecl
begin
    num=1;
    while ( num == 1 ) do
         check = num%2;
    endwhile;
    return 0;
end
}
```

> File: load20_a2.dat
```.
load --library ../expl/library.lib
load --idle /home/kali/myexpos/expl/expl_progs/stage20/infinite.xsm
load --init /home/kali/myexpos/expl/expl_progs/stage20/link_100.xsm
load --exec /home/kali/myexpos/expl/expl_progs/stage20/odd.xsm
load --int=6 /home/kali/myexpos/spl/spl_progs/stage20/int_6.xsm
load --int=7 /home/kali/myexpos/spl/spl_progs/stage20/int_7.xsm
load --int=8 /home/kali/myexpos/spl/spl_progs/stage20/int_8.xsm
load --int=9 /home/kali/myexpos/spl/spl_progs/stage20/int_9.xsm
load --int=10 /home/kali/myexpos/spl/spl_progs/stage20/int_10.xsm
load --int=console /home/kali/myexpos/spl/spl_progs/stage20/console.xsm
load --int=disk /home/kali/myexpos/spl/spl_progs/stage20/disk.xsm
load --module 0 /home/kali/myexpos/spl/spl_progs/stage20/mod_0.xsm
load --module 1 /home/kali/myexpos/spl/spl_progs/stage20/mod_1.xsm
load --module 2 /home/kali/myexpos/spl/spl_progs/stage20/mod_2.xsm
load --module 4 /home/kali/myexpos/spl/spl_progs/stage20/mod_4.xsm
load --module 5 /home/kali/myexpos/spl/spl_progs/stage20/mod_5.xsm
load --module 7 /home/kali/myexpos/spl/spl_progs/stage20/mod_7.xsm
load --exhandler /home/kali/myexpos/spl/spl_progs/stage20/haltprog.xsm
load --int=timer /home/kali/myexpos/spl/spl_progs/stage20/timer.xsm
load --os /home/kali/myexpos/spl/spl_progs/stage20/os_startup.xsm
exit
```

![[Pasted image 20230923201319.png]]


