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

---

# Code Implementation
## INT 9 (Exec Sys Call)

```c
// Save user stack value for later use, set up the kernel stack.

[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16)+13]=SP;
SP=[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 11]*512-1;

// Set the MODE FLAG in the process table to system call number of exec.

[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 9] = 9;
alias userSP R0;
userSP = SP;

// Get the argument (name of the file) from user stack.

alias physicalPageNum R1;
alias offset R2;
alias fileDescPhysicalAddr R3;

physicalPageNum = [PTBR + 2 * ((userSP - 4)/ 512)];
offset = (userSP - 4) % 512;
fileDescPhysicalAddr = (physicalPageNum * 512) + offset;

alias fileName R4;
fileName=[fileDescPhysicalAddr];

// Search the memory copy of the inode table for the file, If the file is not present or file is not in XEXE format return to user mode with return value -1 indicating failure (after setting up MODE FLAG and the user stack).

// inode index value
alias i R5;
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

	alias physicalAddrRetVal R6;
	physicalAddrRetVal = ([PTBR + 2 * ((userSP - 1) / 512)] * 512) + ((userSP - 1) % 512);
	[physicalAddrRetVal] = -1;
endif;

// Call the Exit Process function in process manager module to deallocate the resources and pages of the current process.

alias exitPID R7;
exitPID = [SYSTEM_STATUS_TABLE+1];

multipush(R0,R1,R2,R3,R4,R5,R6,R7);

// EXIT_PROCESS = 3 and argument is PID
R1 = 3;
R2 = exitPID;
call PROCESS_MANAGER;

multipop(R0,R1,R2,R3,R4,R5,R6,R7);

// Get the user area page number from the process table of the current process. This page has been deallocated by the Exit Process function. Reclaim the same page by incrementing the memory free list entry of user area page and decrementing the MEM_FREE_COUNT field in the system status table.

alias user_area_page R8;
user_area_page = [PROCESS_TABLE + 16 * exitPID + 11];

[MEMORY_FREE_LIST + user_area_page] = [MEMORY_FREE_LIST + user_area_page] + 1;
[SYSTEM_STATUS_TABLE + 2] = [SYSTEM_STATUS_TABLE + 2] - 1;

// Set the SP to the start of the user area page to intialize the kernel stack of the new process.

SP = user_area_page * 512 - 1;

// New process uses the PID of the terminated process. Update the STATE field to RUNNING and store inode index obtained above in the inode index field in the process table.

[PROCESS_TABLE + 16 * exitPID + 4] = RUNNING;
[PROCESS_TABLE + 16 * exitPID + 7] = i;

// Allocate new pages and set the page table entries for the new process.

alias freePage R9;

// Invoke the Get Free Page function to allocate 2 stack and 2 heap pages. Also validate the corresponding entries in page table.

// allocating stack page 1

multipush(R0,R1,R2,R3,R4,R5,R6,R7,R8);
R1 = GET_FREE_PAGE;
call MEMORY_MANAGER;
freePage = R0;
multipop(R0,R1,R2,R3,R4,R5,R6,R7,R8);

[PTBR + 16] = freePage;
[PTBR + 17] = "0110";

// allocating stack page 2

multipush(R0,R1,R2,R3,R4,R5,R6,R7,R8);
R1 = GET_FREE_PAGE;
call MEMORY_MANAGER;
freePage = R0;
multipop(R0,R1,R2,R3,R4,R5,R6,R7,R8);

[PTBR + 18] = freePage;
[PTBR + 19] = "0110";

// allocating heap page 1

multipush(R0,R1,R2,R3,R4,R5,R6,R7,R8);
R1 = GET_FREE_PAGE;
call MEMORY_MANAGER;
freePage = R0;
multipop(R0,R1,R2,R3,R4,R5,R6,R7,R8);

[PTBR + 4] = freePage;
[PTBR + 5] = "0110";

// allocating heap page 2

multipush(R0,R1,R2,R3,R4,R5,R6,R7,R8);
R1 = GET_FREE_PAGE;
call MEMORY_MANAGER;
freePage = R0;
multipop(R0,R1,R2,R3,R4,R5,R6,R7,R8);

[PTBR + 6] = freePage;
[PTBR + 7] = "0110";

// Find out the number of blocks occupied by the file from inode table. Allocate same number of code pages by invoking the GetFree Page function and update the page table entries.

// Checking DataBlock 1
if([INODE_TABLE + 16 * i + 8] != -1) then
	multipush(R0,R1,R2,R3,R4,R5,R6,R7,R8);
	R1 = GET_FREE_PAGE;
	call MEMORY_MANAGER;
	freePage = R0;
	multipop(R0,R1,R2,R3,R4,R5,R6,R7,R8);

	[PTBR + 8] = freePage;
	[PTBR + 9] = "0100";	

endif;

// Checking DataBlock 2
if([INODE_TABLE + 16 * i + 9] != -1) then
	multipush(R0,R1,R2,R3,R4,R5,R6,R7,R8);
	R1 = GET_FREE_PAGE;
	call MEMORY_MANAGER;
	freePage = R0;
	multipop(R0,R1,R2,R3,R4,R5,R6,R7,R8);

	[PTBR + 10] = freePage;
	[PTBR + 11] = "0100";	

endif;

// Checking DataBlock 3
if([INODE_TABLE + 16 * i + 10] != -1) then
	multipush(R0,R1,R2,R3,R4,R5,R6,R7,R8);
	R1 = GET_FREE_PAGE;
	call MEMORY_MANAGER;
	freePage = R0;
	multipop(R0,R1,R2,R3,R4,R5,R6,R7,R8);

	[PTBR + 12] = freePage;
	[PTBR + 13] = "0100";	

endif;

// Checking DataBlock 4
if([INODE_TABLE + 16 * i + 11] != -1) then
	multipush(R0,R1,R2,R3,R4,R5,R6,R7,R8);
	R1 = GET_FREE_PAGE;
	call MEMORY_MANAGER;
	freePage = R0;
	multipop(R0,R1,R2,R3,R4,R5,R6,R7,R8);

	[PTBR + 14] = freePage;
	[PTBR + 15] = "0100";	

endif;

// Load the code blocks from the disk to the memory pages using loadi statement

// Loading in DataBlock 1
if([INODE_TABLE + 16 * i + 8] != -1) then
	loadi([PTBR + 8],[INODE_TABLE + 16 * i + 8]);
endif;

// Loading in DataBlock 2
if([INODE_TABLE + 16 * i + 9] != -1) then
	loadi([PTBR + 10],[INODE_TABLE + 16 * i + 9]);
endif;

// Loading in DataBlock 3
if([INODE_TABLE + 16 * i + 10] != -1) then
	loadi([PTBR + 12],[INODE_TABLE + 16 * i + 10]);
endif;

// Loading in DataBlock 4
if([INODE_TABLE + 16 * i + 11] != -1) then
	loadi([PTBR + 14],[INODE_TABLE + 16 * i + 11]);
endif;

// Store the entry point IP (present in the header of first code page) value on top of the user stack

// Making the stack point to the top of the code section
[[PTBR+16]*512]=[[PTBR+8]*512+1];

// Change SP to user stack, change the MODE FLAG back to user mode and return to user mode.

SP = 8 * 512;
[PROCESS_TABLE + 16 * [SYSTEM_STATUS_TABLE+1] + 9] = 0;
ireturn;
```

## MOD 1 (Process Manager)

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
	return;
endif;
```

## MOD 2 (Memory Manager)

```c
// According to the function number value present in R1, implement different functions in module 2.

alias retVal R0;
alias funcNo R1;
alias pageNo R2;

// setting page offset index
alias i R3;

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
	
	// The Page number to be released is present in R2. Decrement the corresponding entry in the memory free list.
	[MEMORY_FREE_LIST + pageNo] = [MEMORY_FREE_LIST + pageNo] - 1;
	
	// If that entry in the memory free list becomes zero, then the page is free. So increment the MEM_FREE_COUNT in the system status table.
	if([MEMORY_FREE_LIST] + pageNo == 0) then
		[SYSTEM_STATUS_TABLE + 2] = [SYSTEM_STATUS_TABLE + 2] + 1;

		// Update the STATUS to READY for all processes (with valid PID) which have STATUS as WAIT_MEM.

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
```

## MOD 7 (Boot Module)

```c
// Loading init program "our code"
loadi(65,7);
loadi(66,8);

//loading inode table into memory
loadi(59,3);
loadi(60,4);

//loading console interrupt
loadi(8,21);
loadi(9,22);

//loading int 6 module
loadi (14,27);
loadi (15,28);

//load int 9 module
loadi(20,33);
loadi(21,34);

// loading int 10 module
loadi(22,35);
loadi(23,36);

// loading exception handler routine
loadi(2, 15);
loadi(3, 16);

// Timer interupt loading
loadi(4, 17);
loadi(5, 18);

// loading library code
loadi(63,13);
loadi(64,14);

// Loading int 7 to disk
loadi(16,29);
loadi(17,30);

// Loading idle code from disk to memory
loadi(69,11);
loadi(70,12);

//loading 2nd user process
//loadi(85,70);

// loading module 0 from disk
loadi(40,53);
loadi(41,54);

//loading module 1 from disk
loadi(42,55);
loadi(43,56);

//loading module 2 from disk
loadi(44,57);
loadi(45,58);


// loading module 4 from disk
loadi(48,61);
loadi(49,62);


// loading module 5 from disk
loadi(50,63);
loadi(51,64);

// Setting the page table for idle process
PTBR=PAGE_TABLE_BASE; 
PTLR=10; 


// Setting up page table for init process
PTBR=PTBR+20;

// library
[PTBR+0]=63;
[PTBR+1]="0100";
[PTBR+2]=64;
[PTBR+3]="0100";

// heap
[PTBR+4]=78;
[PTBR+5]="0110";
[PTBR+6]=79;
[PTBR+7]="0110";

// code
[PTBR+8]=65;
[PTBR+9]="0100";
[PTBR+10]=66;
[PTBR+11]="0100";
[PTBR+12]=-1;
[PTBR+13]="0000";
[PTBR+14]=-1;
[PTBR+15]="0000";

// stack
[PTBR+16]=76;
[PTBR+17]="0110";
[PTBR+18]=77;
[PTBR+19]="0110";

// Setting up page table for 2nd user program
PTBR=PTBR+20;

// library
[PTBR+0]=63;
[PTBR+1]="0100";
[PTBR+2]=64;
[PTBR+3]="0100";

// heap
[PTBR+4]=83;
[PTBR+5]="0110";
[PTBR+6]=84;
[PTBR+7]="0110";

// code
[PTBR+8]=85;
[PTBR+9]="0100";
[PTBR+10]=-1;
[PTBR+11]="0000";
[PTBR+12]=-1;
[PTBR+13]="0000";
[PTBR+14]=-1;
[PTBR+15]="0000";

// stack
[PTBR+16]=86;
[PTBR+17]="0110";
[PTBR+18]=87;
[PTBR+19]="0110";

// process table idle
alias PCT R5;
PCT = PROCESS_TABLE;

PTBR= PTBR - 20;
//init
[PCT + 1*16 + 1] = 1;
[PCT + 1*16 + 4] = CREATED;
[PCT + 1*16 + 11] = 80;
[PCT + 1*16 + 12] = 0;
[PCT + 1*16 + 13] = 8*512;
[PCT + 1*16 + 14] = PTBR;
[PCT + 1*16 + 15] = 10;

[76*512]=[65*512 + 1];

PTBR= PTBR + 20;

//user program with pid 2

[PCT + 2*16 + 1] = 2;
[PCT + 2*16 + 4] = CREATED;
[PCT + 2*16 + 11] = 88;
[PCT + 2*16 + 12] = 0;
[PCT + 2*16 + 13] = 8*512;
[PCT + 2*16 + 14] = PTBR;
[PCT + 2*16 + 15] = 10;

[86*512]=[85*512 + 1];

[PCT + 2*16 + 4]= TERMINATED;
[PCT + 3*16 + 4]= TERMINATED;
[PCT + 4*16 + 4]= TERMINATED;
[PCT + 5*16 + 4]= TERMINATED;
[PCT + 6*16 + 4]= TERMINATED;
[PCT + 7*16 + 4]= TERMINATED;
[PCT + 8*16 + 4]= TERMINATED;
[PCT + 9*16 + 4]= TERMINATED;
[PCT + 10*16 + 4]= TERMINATED;
[PCT + 11*16 + 4]= TERMINATED;
[PCT + 12*16 + 4]= TERMINATED;
[PCT + 13*16 + 4]= TERMINATED;
[PCT + 14*16 + 4]= TERMINATED;
[PCT + 15*16 + 4]= TERMINATED;


[TERMINAL_STATUS_TABLE ]= 0;

R1=512 * 57;
alias count R2;
count =0;


while(count < 83) do
    [R1 + count] = 1;
    count = count + 1;
endwhile;

while(count < 128) do
    [R1 + count] = 0;
    count = count + 1;
endwhile;

[SYSTEM_STATUS_TABLE + 2 ] = 45;
[SYSTEM_STATUS_TABLE + 3 ] = 0;

return;

```

```c
int main()
{
decl
    int temp,num,check;
enddecl
begin
    num=1;
    while ( num <= 100 ) do
    	 check = num%2;
    	 if( check != 0) then
	         write(num);
         endif;
         num = num + 1;
    endwhile;
    return 0;
end
}
```

```c
int main()
{
decl
    int temp;
    str file;
enddecl
begin
    temp = write("Input file:");
    temp = read(file);
    temp = exposcall("Exec", file);
    return 0;
end
}
```

## INT 9 WORKING
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

// New process uses the PID of the terminated process. Update the STATE field to RUNNING and store inode index obtained above in the inode index field in the process table.

[PROCESS_TABLE + 16 * exitPID + 4] = RUNNING;
[PROCESS_TABLE + 16 * exitPID + 7] = i;

// Allocate new pages and set the page table entries for the new process.

alias freePage R5;

// Invoke the Get Free Page function to allocate 2 stack and 2 heap pages. Also validate the corresponding entries in page table.

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

// allocating heap page 1

multipush(R0,R1,R2,R3);
R1 = GET_FREE_PAGE;
call MEMORY_MANAGER;
freePage = R2;
multipop(R0,R1,R2,R3);

[PTBR + 4] = freePage;
[PTBR + 5] = "0110";

// allocating heap page 2

multipush(R0,R1,R2,R3);
R1 = GET_FREE_PAGE;
call MEMORY_MANAGER;
freePage = R2;
multipop(R0,R1,R2,R3);

[PTBR + 6] = freePage;
[PTBR + 7] = "0110";

// Find out the number of blocks occupied by the file from inode table. Allocate same number of code pages by invoking the GetFree Page function and update the page table entries.

// Checking DataBlock 1
if([INODE_TABLE + 16 * i + 8] != -1) then
	multipush(R0,R1,R2,R3);
	R1 = GET_FREE_PAGE;
	call MEMORY_MANAGER;
	freePage = R2;
	multipop(R0,R1,R2,R3);

	[PTBR + 8] = freePage;
	[PTBR + 9] = "0100";	

endif;

// Checking DataBlock 2
if([INODE_TABLE + 16 * i + 9] != -1) then
	multipush(R0,R1,R2,R3);
	R1 = GET_FREE_PAGE;
	call MEMORY_MANAGER;
	freePage = R2;
	multipop(R0,R1,R2,R3);

	[PTBR + 10] = freePage;
	[PTBR + 11] = "0100";	

endif;

// Checking DataBlock 3
if([INODE_TABLE + 16 * i + 10] != -1) then
	multipush(R0,R1,R2,R3);
	R1 = GET_FREE_PAGE;
	call MEMORY_MANAGER;
	freePage = R2;
	multipop(R0,R1,R2,R3);

	[PTBR + 12] = freePage;
	[PTBR + 13] = "0100";	

endif;

// Checking DataBlock 4
if([INODE_TABLE + 16 * i + 11] != -1) then
	multipush(R0,R1,R2,R3);
	R1 = GET_FREE_PAGE;
	call MEMORY_MANAGER;
	freePage = R2;
	multipop(R0,R1,R2,R3);

	[PTBR + 14] = freePage;
	[PTBR + 15] = "0100";	

endif;

// Load the code blocks from the disk to the memory pages using loadi statement

// Loading in DataBlock 1
if([INODE_TABLE + 16 * i + 8] != -1) then
	loadi([PTBR + 8],[INODE_TABLE + 16 * i + 8]);
endif;

// Loading in DataBlock 2
if([INODE_TABLE + 16 * i + 9] != -1) then
	loadi([PTBR + 10],[INODE_TABLE + 16 * i + 9]);
endif;

// Loading in DataBlock 3
if([INODE_TABLE + 16 * i + 10] != -1) then
	loadi([PTBR + 12],[INODE_TABLE + 16 * i + 10]);
endif;

// Loading in DataBlock 4
if([INODE_TABLE + 16 * i + 11] != -1) then
	loadi([PTBR + 14],[INODE_TABLE + 16 * i + 11]);
endif;

// Store the entry point IP (present in the header of first code page) value on top of the user stack

// Making the stack point to the top of the code section
[[PTBR+16]*512]=[[PTBR+8]*512+1];

// Change SP to user stack, change the MODE FLAG back to user mode and return to user mode.

SP = 8 * 512;
[PROCESS_TABLE + 16 * [SYSTEM_STATUS_TABLE+1] + 9] = 0;
ireturn;
```