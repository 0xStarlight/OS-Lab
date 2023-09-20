# Summary

In this stage, we will introduce disk interrupt handling in XSM. The load statement in SPL translates to LOAD instruction in XSM. The LOAD instruction takes two arguments, a page number and a block number. XSM machine raises the disk interrupt when the disk operation is complete. The disk device driver module is responsible for programming the disk controller hardware for handling disk operations. eXpOS maintains a data structure called Disk Status Table to keep track of these disk-memory transfers. The process suspends its execution by changing its state to WAIT_DISK and invokes the scheduler, allowing other concurrent processes to run. When the load/store transfer is complete, XSM machine raises the hardware interrupt called the disk interrupt. The disk interrupt handler releases the disk by changing the STATUS field in the Disk Status table to 0. It then wakes up all the processes waiting for the disk (by changing the STATE from WAIT_DISK to READY). The IDLE process is designed to take care of situations where all processes are waiting for disk/terminal interrupt.

In this stage, we have to modify the exec system call by replacing the loadi statement by a call to the Disk Load function. The Disk Load function, the Acquire Disk function, and the disk interrupt handler must also be implemented in this stage. The Acquire Disk function in the resource manager module takes the PID of a process as an argument. When the disk is finally free, the process is woken up by the disk interrupt handler. The disk interrupt handler then performs tasks such as setting the STATUS field in the Disk Status table to 0 indicating that the disk is no longer busy and changing the state of the process to READY, which is in WAIT_DISK state. Instead of the loadi statement used to load the disk block to the memory page, invoke the Disk Load function present in the device manager module.

---

# Starting

In this stage, we will introduce disk interrupt handling in XSM. In the previous stage, we used the _loadi_ statement to load a disk block into a memory page. When the [loadi statement](https://exposnitc.github.io/expos-docs/support-tools/spl/) (immediate load) is used for loading, the machine will execute the next instruction only after the block transfer is complete by the [disk controller](https://exposnitc.github.io/expos-docs/arch-spec/interrupts-exception-handling/#disk-controller-interrupt) . A process can use the **load statement** instead of _loadi_ to load a disk block to a memory page. The [load statement](https://exposnitc.github.io/expos-docs/support-tools/spl/)in SPL translates to [LOAD instruction in XSM](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/).

The LOAD instruction takes two arguments, a page number and a block number. The LOAD instruction initiates the transfer of data from the specified disk block to the memory page. The **XSM machine doesn't wait for the block transfer to complete**, it continues with the execution of the next instruction. Instead, the XSM machine provides a hardware mechanism to detect the completion of data transfer. XSM machine raises the [disk interrupt](https://exposnitc.github.io/expos-docs/tutorials/xsm-interrupts-tutorial/#disk-and-console-interrupts)when the disk operation is complete.

In real operating systems, the OS maintains a software module called the disk [device driver](https://en.wikipedia.org/wiki/Device_driver)module for handling disk access. This module is responsible for programming the [disk controller](https://en.wikipedia.org/wiki/Disk_controller) hardware for handling disk operations. When the OS initiates a disk read/write operation from the context of a process, the device driver module is invoked with appropriate arguments. In our present context, the [device manager module](https://exposnitc.github.io/expos-docs/modules/module-04/) integrates a common "driver software" for all devices of XSM. The load and store instructions actually are high level "macro operations" given to you that abstract away the low level details of the device specific code to program the disk controller hardware. The _loadi_ instruction abstracts disk I/O using the method of [polling](https://en.wikipedia.org/wiki/Polling_(computer_science)) whereas the _load_ instruction abstracts [interrupt based](https://en.wikipedia.org/wiki/Asynchronous_I/O)disk I/O.

To initiate the disk transfer using the load statement, first the process has to **acquire** the disk. This ensures that no other process uses the disk while the process which has acquired the disk is loading the disk block to the memory page. eXpOS maintains a data structure called [Disk Status Table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#disk-status-table) to keep track of these disk-memory transfers. The disk status table stores the status of the disk indicating whether the disk is busy or free. The disk status table has a LOAD/STORE bit indicating whether the disk operation is a load or store. The table also stores the page number and the block number involved in the transfer. To keep track of the process that has currently acquired the disk, the PID of the process is also stored in the disk status table. The SPL constant [DISK_STATUS_TABLE](https://exposnitc.github.io/expos-docs/support-tools/constants/)gives the starting address of the Disk Status Table in the[XSM memory](https://exposnitc.github.io/expos-docs/os-implementation/).

After the current process has acquired the disk for loading, it initializes the Disk Status Table according to the operation to be perfromed (read/write). The process then issues the _load_ statement to initiate the loading of the disk block to the memory page. As mentioned earlier, the XSM machine does not wait for the transfer to complete. It continues with the execution of the next instruction. However, virtually in any situation in eXpOS, the process has to wait till the data transfer is complete before proceeding (why?). Hence, the process suspends its execution by changing its state to WAIT_DISK and invokes the scheduler, allowing other concurrent processes to run. (At present, the only concurrent process for the OS to schedule is the IDLE process. However, in subsequent stages we will see that the OS will have more meaningful processes to run.)

When the load/store transfer is complete, XSM machine raises the hardware interrupt called the **disk interrupt**. This interrupt mechanism is similar to the console interrupt. Note that when disk interrupt occurs, XSM machine stops the execution of the currently running process. The currently running process is not the one that has acquired the disk (why?). The disk interrupt handler releases the disk by changing the STATUS field in the Disk Status table to 0. It then wakes up all the processes waiting for the disk (by changing the STATE from WAIT_DISK to READY) which also includes the process which is waiting for the disk-transfer to complete. Then returns to the process which was interrupted by disk controller.

XSM machine disables interrupts when executing in the kernel mode. Hence, the disk controller can raise an interrupt only when the machine is executing in the user mode. Hence the OS has to schedule "some process" even if all processess are waiting for disk/terminal interrupt - for otherwise, the device concerned will never be able to interrupt the processor. The IDLE process is precisely designed to take care of this and other similar situations.

![](https://exposnitc.github.io/expos-docs/assets/img/roadmap/exec2.png)

Control flow for _Exec_ system call

In this stage, **you have to modify the exec system call by replacing the [loadi statement](https://exposnitc.github.io/expos-docs/support-tools/spl/) by a call to the Disk Load function. The Disk Load function (in device manager module), the Acquire Disk function (in resource manager module) and the disk interrupt handler must also be implemented in this stage.** Minor modifications are also required for the boot module.

#### 1.Disk Load (function number = 2,[device manager module](https://exposnitc.github.io/expos-docs/modules/module-04/))[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-18/#1disk-load-function-number-2device-manager-module "Permanent link")

The Disk Load function takes the PID of a process, a page number and a block number as input and performs the following tasks :

1. Acquires the disk by invoking the Acquire Disk function in the [resource manager module](https://exposnitc.github.io/expos-docs/modules/module-00/) (module 0)
2. Set the [Disk Status table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#disk-status-table)entries as mentioned in the algorithm (specified in the above link).
3. Issue the [load statement](https://exposnitc.github.io/expos-docs/support-tools/spl/) to initiate a disk block to memory page DMA transfer.
4. Set the state of the process (with given PID) to WAIT_DISK and invoke the scheduler.

#### 2.Acquire Disk (function number = 3, [resource manager module](https://exposnitc.github.io/expos-docs/modules/module-00/))[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-18/#2acquire-disk-function-number-3-resource-manager-module "Permanent link")

The Acquire Disk function in the resource manager module takes the PID of a process as an argument. The Acquire disk function performs the following tasks :

1. While the disk is busy (STATUS field in the Disk Status Table is 1), set the state of the process to WAIT_DISK and invoke the scheduler. **When the disk is finally free, the process is woken up by the disk interrupt handler.**
2. Lock the disk by setting the STATUS and the PID fields in the Disk Status Table to 1 and PID of the process respectively.

Note

Both Disk Load and Acquire Disk module functions implemented above are final versions according to the algorithm given in respective modules.

#### 3. Implementation of [Disk Interrupt handler](https://exposnitc.github.io/expos-docs/os-design/disk-interrupt/)[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-18/#3-implementation-of-disk-interrupt-handler "Permanent link")

When the disk-memory transfer is complete, XSM raises the disk interrupt. The disk interrupt handler then performs the following tasks :

1. Switch to the kernel stack and back up the register context.
2. Set the STATUS field in the Disk Status table to 0 indicating that disk is no longer busy.
3. Go through all the process table entries, and change the state of the process to READY, which is in WAIT_DISK state.
4. Restore the register context and return to user mode using the `ireturn` statement.

Note

There is no Release Disk function to release the disk instead the disk interrupt handler completes the task of the Release Disk function.

#### 4. Modification to exec system call (interrupt 9 routine)[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-18/#4-modification-to-exec-system-call-interrupt-9-routine "Permanent link")

Instead of the loadi statement used to load the disk block to the memory page, invoke the **Disk Load** function present in the [device manager module](https://exposnitc.github.io/expos-docs/modules/module-04/).

We will initialize another data strucutre as well in this stage. This is the [per-process resource table](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-resource-table). (This step can be deferred to later stages, but since the work involved is simple, we will finish it here). The per-process resource table stores the information about the files and semaphores which a process is currently using. For each process, per-process resource table is stored in the user area page of the process. This table has 8 entries with 2 words each, in total it occupies 16 words. _We will reserve the last 16 words of the User Area Page to store the per-process Resource Table of the process._ In exec, after reacquiring the [user area page](https://exposnitc.github.io/expos-docs/os-design/process-table/#user-area) for the new process, per-process resource table should be initialized in this user area page. Since the newly created process has not opened any files or semaphores, each entry in the per-process table is initialized to -1.

#### 5. Modifications to boot module[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-18/#5-modifications-to-boot-module "Permanent link")

Following modifications are done in boot module :

1. Load the disk interrupt routine from the disk to the memory.
2. Initialize the STATUS field in the Disk Status Table to 0.
3. Initialize the [per-process resource table](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-resource-table) of init process.

Compile and load the modified and newly written files into the disk using XFS-interface. Run the Shell version-I with any program to check for errors.

---
# Disk interrupt

In this stage, we will introduce disk interrupt handling in XSM. In the previous stage, we used the *loadi* statement to load a disk block into a memory page. When the [loadi statement](https://exposnitc.github.io/expos-docs/support-tools/spl/) (immediate load) is used for loading, the machine will execute the next instruction only after the block transfer is complete by the [disk controller](https://exposnitc.github.io/expos-docs/arch-spec/interrupts-exception-handling/#disk-controller-interrupt) . A process can use the **load statement** instead of *loadi* to load a disk block to a memory page. The [load statement](https://exposnitc.github.io/expos-docs/support-tools/spl/) in SPL translates to [LOAD instruction in XSM](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/).

The LOAD instruction takes two arguments, a page number and a block number. The LOAD instruction initiates the transfer of data from the specified disk block to the memory page. The **XSM machine doesn't wait for the block transfer to complete**, it continues with the execution of the next instruction. Instead, the XSM machine provides a hardware mechanism to detect the completion of data transfer. XSM machine raises the [disk interrupt](https://exposnitc.github.io/expos-docs/tutorials/xsm-interrupts-tutorial/#disk-and-console-interrupts) when the disk operation is complete.

> In real operating systems, the OS maintains a software module called the disk [device driver](https://en.wikipedia.org/wiki/Device_driver) module for handling disk access. This module is responsible for programming the [disk controller](https://en.wikipedia.org/wiki/Disk_controller) hardware for handling disk operations. When the OS initiates a disk read/write operation from the context of a process, the device driver module is invoked with appropriate arguments. In our present context, the [device manager module](https://exposnitc.github.io/expos-docs/modules/module-04/) integrates a common "driver software" for all devices of XSM. The load and store instructions actually are high level "macro operations" given to you that abstract away the low level details of the device specific code to program the disk controller hardware. The *loadi* instruction abstracts disk I/O using the method of [polling](https://en.wikipedia.org/wiki/Polling_(computer_science)) whereas the *load* instruction abstracts [interrupt based](https://en.wikipedia.org/wiki/Asynchronous_I/O) disk I/O.
> 

## What is polling?

Polling in the context of operating systems refers to a mechanism used to manage and coordinate various tasks or processes within a computer system. It involves regularly checking the status or condition of a resource or device, such as input/output devices, to determine if they are ready to send or receive data or perform some operation. Polling is an alternative to interrupt-driven I/O, where the hardware generates an interrupt to signal that it needs attention.

Here's how polling works:

1. **Regularly Check the Status**: Instead of relying on interrupts, the operating system or a program will periodically check the status of a particular resource or device. For example, it might check if a keyboard key has been pressed, if a file is ready to be read, or if a network socket has data to transmit.
2. **Resource Availability**: When polling, the system waits in a loop until the resource becomes available or a certain condition is met. It can be an inefficient use of CPU resources because the system keeps checking even when the resource is not ready.
3. **Blocking vs. Non-blocking**: Polling can be implemented in different ways. In blocking polling, the program or system will wait indefinitely until the resource is ready, which can lead to wasted CPU time. In non-blocking polling, the system checks the resource's status and immediately proceeds with other tasks if the resource is not yet ready.
4. **Resource Utilization**: Polling can be more predictable and easier to manage in some cases, but it can also be less efficient than interrupt-driven I/O because it continually consumes CPU cycles to check the status of resources, even if they are not ready.
5. **Real-time Systems**: Polling can be more suitable for real-time systems, where predictability and determinism are crucial, and the system needs to ensure that certain tasks are executed within specific time constraints.
6. **Examples**: Polling is used in various contexts, such as keyboard input polling, where the operating system regularly checks if a key has been pressed; disk I/O polling, where it checks if a disk operation has completed; and network communication, where it checks if data is available for transmission or reception.

In summary, polling in operating systems is a method for resource management and event handling that involves regularly checking the status of resources or devices to determine their readiness for processing. While it can provide predictability in certain situations, it may also be less efficient than interrupt-driven mechanisms in terms of CPU utilization. The choice between polling and interrupt-driven I/O depends on the specific requirements and characteristics of the system and its applications.

### **PER-PROCESS RESOURCE TABLE[¶](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-resource-table)**

The Per-Process Resource Table has 8 entries and each entry is of 2 words. **The last 16 words of the User Area Page are reserved for this**. For every instance of a file opened (or a semaphore acquired) by the process, it stores the index of the Open File Table (or Semaphore Table) entry for the file (or semaphore) is stored in this table. One word is used to store the resource identifier which indicates whether the resource opened by the process is a [FILE](https://exposnitc.github.io/expos-docs/support-tools/constants/) or a [SEMAPHORE](https://exposnitc.github.io/expos-docs/support-tools/constants/). Open system call sets the values of entries in this table for a file.

The per-process resource table entry has the following format.

| Resource Identifier (1 word) | Index of Open File Table/ Semaphore Table entry (1 word) |
| --- | --- |

File descriptor, returned by Open system call, is the index of the per-process resource table entry for that open instance of the file.

A free entry is denoted by -1 in the Resource Identifier field.

### **PER-PROCESS KERNEL STACK[¶](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-kernel-stack)**

Control is tranferred from a user program to a kernel module on the occurence of one of the following events :

1. The user program executes a system call
2. When an interrupt/exception occurs.

In either case, the kernel allocates a separate stack **for each process** (called the kernel stack of the process) which is different from the stack used by the process while executing in the user mode (called the user stack). Kernel modules use the space in the kernel stack for storing local data and do not use the user stack. This is to avoid user "hacks" into the kernel using the application's stack.

In the case of a system call, the application will store the parameters for the system call in its user stack. Upon entering the kernel module (system call), the kernel will extract these parameters from the application's stack and then change the stack pointer to its own stack before further execution. Since the application invokes the kernel module voluntarily, it is the responsibility of the application to save the contents of its registers (except the stack pointer and base pointer registers in the case of the XSM machine) before invoking the system call.

In the case of an interrupt/exception, the user process does not have control over the transfer to the kernel module (interrupt/exception handler). Hence the execution context of the user process (that is, values of the registers) must be saved by the kernel module, before the kernel module uses the machine registers for other purposes, so that the machine state can be restored after completion of the interrupt/exception handler. The kernel stack is used to store the execution context of the user process. This context is restored before the return from the kernel module. (For the implementation of eXpOS on the XSM architecture, the [backup](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/#backup) and [restore](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/#restore) instructions facilitate this).

In addition to the above, if a kernel module invokes another kernel module while executing a system call/interrupt, the parameters to the called module and the return values from the module are passed through the same kernel stack.

Here is a detailed tutorial on [kernel stack management in system calls, interrupts and exceptions](https://exposnitc.github.io/expos-docs/os-implementation/). A separate tutorial is provided for [kernel stack managament during context switch](https://exposnitc.github.io/expos-docs/os-design/timer-stack-management/).

---

# Questions

```ad-question
title:  Can we use the load statement in the boot module code instead of the loadi statement? Why?

No. The modules needed for the execution of load, need to be present in the memory first. And even if they are present, at the time of execution of the boot module, no process or data structures are initialized (like Disk Status Table).
```

```ad-question
title: Why does the disk interrupt handler has to backup the register context?
Disk interrupt handler is a hardware interrupt. When disk interrupt occurs, the XSM machine just pushes IP+2 value on stack and transfers control to disk interrupt. Occurance of a hardware interrupt is unexpected. When the disk interrupt is raised, the process will not have control over it so the process (curently running) cannot backup the registers. That's why interrupt handler must back up the context of the process (currently running) before modifying the machine registers. The interrupt handler also needs to restore the context before returning to user mode.
```

```ad-question
title:  Why doesn't system calls backup the register context?
The process currently running is in full control over calling the interrupt (software interrupt) corresponding to a system call. This allows a process to back up the registers used till that point (not all registers). Note that instead of process, the software interrupt can also back up the registers. But, the software interrupt will not know how many registers are used by the process so it has to back up all the registers. Backing up the registers by a process saves space and time.
```

```ad-question
title: Does the XSM terminal input provide polling based input?
Yes, readi statement provided in SPL gives polling based terminal I/O. But readi statement only works in debug mode. Write operation is always asynchronous.
```

---
# Things to implement
1. [x] mod_0 (disk load and acquire disk)
2. [x] disk interrupt handler
3. [x] int_9 (mod exec syscall)
4. [x] mod_7 (boot module)

> File: console.spl
```c
[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16)+13]=SP;
SP=[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 11]*512-1;

backup;
breakpoint;
alias reqPID R5;
reqPID = [TERMINAL_STATUS_TABLE + 1];
[PROCESS_TABLE + reqPID * 16 + 8] = P0;
breakpoint;
multipush(R0,R1,R2,R3);

R1=9;
R2=reqPID;
call RESOURCE_MANAGER;

multipop(R0,R1,R2,R3);

restore;

SP=[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 13];

ireturn;
```

> File: haltprog.spl
```c
halt;
```

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
if([INODE_TABLE+16*i+8]!=-1)then
    multipush(R0,R1,R2,R3);
    R1 = 2;
    R2 = exitPID;
    R3 = [PTBR+8];
    R4 = [INODE_TABLE+16*i+8];
    call DEVICE_MANAGER;
    multipop(R0,R1,R2,R3);
endif;

if([INODE_TABLE+16*i+9]!=-1)then
    multipush(R0,R1,R2,R3);
    R1 = 2;
    R2 = exitPID;
    R3 = [PTBR+10];
    R4 = [INODE_TABLE+16*i+9];
    call DEVICE_MANAGER;
    multipop(R0,R1,R2,R3);
endif;

if([INODE_TABLE+16*i+10]!=-1)then
    multipush(R0,R1,R2,R3);
    R1 = 2;
    R2 = exitPID;
    R3 = [PTBR+12];
    R4 = [INODE_TABLE+16*i+10];
    call DEVICE_MANAGER;
    multipop(R0,R1,R2,R3);
endif;

if([INODE_TABLE+16*i+11]!=-1)then
    multipush(R0,R1,R2,R3);
    R1 = 2;
    R2 = exitPID;
    R3 = [PTBR+14];
    R4 = [INODE_TABLE+16*i+11];
    call DEVICE_MANAGER;
    multipop(R0,R1,R2,R3);
endif;

// Store the entry point IP (present in the header of first code page) value on top of the user stack

// Making the stack point to the top of the code section
[[PTBR+16]*512]=[[PTBR+8]*512+1];

// Change SP to user stack, change the MODE FLAG back to user mode and return to user mode.

SP = 8 * 512;
[PROCESS_TABLE + 16 * [SYSTEM_STATUS_TABLE+1] + 9] = 0;
ireturn;
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
	return;
endif;

```

> File: mod_4.spl
```c
alias func_num R1;
alias currPID R2;

// Disk Load Function
// The Disk Load function takes the PID of a process, a page number and a block number as input and performs the following tasks
// function number for DISK_LOAD = 2
if(func_num == 2) then
    alias currPgNo R3;
    alias currBlockNo R4;
    multipush(R1,R2,R3,R4);
    // Acquires the disk by invoking the Acquire Disk function in the resource manager module
    // function number for aquire disk is 3 in resource manager
    R1 = 3;
    call RESOURCE_MANAGER;
    multipop(R1,R2,R3,R4);

    // Set the Disk Status tableentries as mentioned in the algorithm
    [DISK_STATUS_TABLE] = 1;
    [DISK_STATUS_TABLE + 1] = 0;
    [DISK_STATUS_TABLE + 2] = currPgNo;
    [DISK_STATUS_TABLE + 3] = currBlockNo;
    [DISK_STATUS_TABLE + 4] = currPID;

    // Issue the load statement to initiate a disk block to memory page DMA transfer.
    load(currPgNo, currBlockNo);

    // Set the state of the process (with given PID) to WAIT_DISK and invoke the scheduler.
    [PROCESS_TABLE +currPID*16 + 4] = WAIT_DISK;

    multipush(R1,R2,R3,R4);
    call MOD_5;
    multipop(R1,R2,R3,R4);
    return;
endif;

if(func_num==3)then
  multipush(R1,R2,R3);
  
  //to call acquire terminal(func_num=8)
  R1=8;
  R2=[SYSTEM_STATUS_TABLE + 1];
  call RESOURCE_MANAGER;
  
  multipop(R1,R2,R3);
  print R3;
  
  multipush(R1,R2,R3);
  
  //to call release terminal(func_num=9)
  R1=9;
  R2=[SYSTEM_STATUS_TABLE + 1];
  call RESOURCE_MANAGER;
  
  //can save retval which is in R0 if needed.
  
  multipop(R1,R2,R3);
endif;

if(func_num == 4) then
  multipush(R1,R2,R3);

  R1 = 8;
  R2 = [SYSTEM_STATUS_TABLE + 1];

  call RESOURCE_MANAGER;
  multipop(R1,R2,R3);

  read;

  [PROCESS_TABLE+16*R2+4] = WAIT_TERMINAL;
  multipush(R1,R2,R3);

  call SCHEDULER;

  multipop(R1,R2,R3);

  alias word R5;
  word = ([PTBR + (R3)*2/512])*512 + (R3)%512;
  [word] = [PROCESS_TABLE + currPID * 16 + 8];

endif;

return;
```

> File: mod_7.spl
```c
//ex-handler
loadi(2, 15);
loadi(3, 16);

//library
loadi(63, 13);
loadi(64, 14);

//Init
loadi(65, 7);
loadi(66, 8);

//Idle
loadi(69, 11);
loadi(70, 12);

//INT=1; timer interrupt
loadi(4, 17);
loadi(5, 18);

//INT=2 disk interrupt
loadi(6, 19);
loadi(7, 20);

//INT=3; terminal interrupt handler
loadi(8, 21);
loadi(9, 22);

//INT=6; read
loadi(14, 27);
loadi(15, 28);

//INT=7; write
loadi(16, 29);
loadi(17, 30);

//INT=9; exec
loadi(20, 33);
loadi(21, 34);

//INT=10; exit
loadi(22, 35);
loadi(23, 36);

//MOD_0: Resource Manager Module
loadi(40, 53);
loadi(41, 54);

//MOD_1: Process Manager Module
loadi(42, 55);
loadi(43, 56);

//MOD_2: Memory Manager Module
loadi(44, 57);
loadi(45, 58);

//MOD_4: Device Manager Module
loadi(48, 61);
loadi(49, 62);

//MOD_5: Context Switch Module
loadi(50, 63);
loadi(51, 64);

//inode table
loadi(59, 3);
loadi(60, 4);

PTBR = PAGE_TABLE_BASE + 20;
PTLR = 10;

//library
[PTBR + 0] = 63;
[PTBR + 1] = "0100";
[PTBR + 2] = 64;
[PTBR + 3] = "0100";

//heap
[PTBR + 4] = 78;
[PTBR + 5] = "0110";
[PTBR + 6] = 79;
[PTBR + 7] = "0110";

//code
[PTBR + 8] = 65;
[PTBR + 9] = "0100";
[PTBR + 10] = 66;
[PTBR + 11] = "0100";
[PTBR + 12] = -1;
[PTBR + 13] = "0000";
[PTBR + 14] = -1;
[PTBR + 15] = "0000";

//stack
[PTBR + 16] = 76;
[PTBR + 17] = "0110";
[PTBR + 18] = 77;
[PTBR + 19] = "0110";

//init
alias init_process_table R6;
init_process_table = PROCESS_TABLE + 1 * 16;
[init_process_table + 1] = 1;
[init_process_table + 4] = CREATED;
[init_process_table + 11] = 80;
[init_process_table + 12] = 0;
[init_process_table + 13] = 8 * 512;
[init_process_table + 14] = PTBR;
[init_process_table + 15] = 10;
[76 * 512] = [65 * 512 + 1];

alias resource_table R0;
resource_table = 80*512 + RESOURCE_TABLE_OFFSET;
[resource_table + 2*0] = -1; 
[resource_table + 2*1] = -1; 
[resource_table + 2*2] = -1; 
[resource_table + 2*3] = -1; 
[resource_table + 2*4] = -1; 
[resource_table + 2*5] = -1; 
[resource_table + 2*6] = -1; 
[resource_table + 2*7] = -1; 

//set other process table statuses to terminated
alias i R0;
i = 2;
while(i < 16) do
    [PROCESS_TABLE + i * 16 + 4] = TERMINATED;
    i = i + 1;
endwhile;

//free pages and things
i = 0;
while (i < 83) do
  [MEMORY_FREE_LIST + i] = 1;
  i = i + 1;
endwhile;

//mem_free_count
[SYSTEM_STATUS_TABLE + 2] = 0;

while (i < 128) do
  [MEMORY_FREE_LIST + i] = 0;
  [SYSTEM_STATUS_TABLE + 2] = [SYSTEM_STATUS_TABLE + 2] + 1;
  i = i + 1;
endwhile;

//wait_mem_count
[SYSTEM_STATUS_TABLE + 3] = 0;

[TERMINAL_STATUS_TABLE] = 0;
[DISK_STATUS_TABLE] = 0;
return;
```

> File: sample_int6.spl
```c
[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 9] = 7;

alias userSP R0;
userSP = SP;

[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16)+13]=SP;
SP=[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 11]*512-1;


alias physicalPageNum R1;
alias offset R2;
alias fileDescPhysicalAddr R3;
physicalPageNum = [PTBR + 2 * ((userSP - 4)/ 512)];
offset = (userSP - 4) % 512;
fileDescPhysicalAddr = (physicalPageNum * 512) + offset;
alias fileDescriptor R4;
fileDescriptor=[fileDescPhysicalAddr];



if (fileDescriptor != -1)
then
   alias physicalAddrRetVal R5;
   physicalAddrRetVal = ([PTBR + 2 * ((userSP - 1) / 512)] * 512) + ((userSP - 1) % 512);
   [physicalAddrRetVal] = -1;
else
   alias word R5;
  word = [[PTBR + 2 * ((userSP - 3) / 512)] * 512 + ((userSP - 3) % 512)];
  //print word;

  //Code for S15
  multipush(R0,R1,R2,R3,R4,R5);
  alias func_num R1;
  alias pid R2;
  alias word_print R3;

  func_num=4;
  pid=[SYSTEM_STATUS_TABLE + 1];
  word_print=word;

  //call to device manager.
  call DEVICE_MANAGER;

  multipop(R0,R1,R2,R3,R4,R5);

  alias physicalAddrRetVal R6;
  physicalAddrRetVal = ([PTBR + 2 * (userSP - 1)/ 512] * 512) + ((userSP - 1) % 512);
  [physicalAddrRetVal] = 0;
endif;

SP=userSP;

[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 9] = 0;

ireturn;

```

> File: sample_timer.spl
```c
[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16)+13]=SP;
SP=[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16)+11]*512-1 ;
backup;

[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 4] = READY;

call MOD_5;

restore;

//Set back Kernel SP, PTBR , PTLR
SP =  [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 13];
//Commented below because already done in scheduler
//PTBR = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 14];
//PTLR = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 15];
//MODE=0
[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 9]=0;

ireturn;
```

> File: disk.spl
```c
// When the disk-memory transfer is complete, XSM raises the disk interrupt. The disk interrupt handler then performs the following tasks 

alias userSP R0;
userSP = SP;

// Switch to the kernel stack and back up the register context.
[PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+13] = SP;  
SP = [PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+11]*512-1;

// backup the register contexts
backup;

// Set the STATUS field in the Disk Status table to 0 indicating that disk is no longer busy.

[DISK_STATUS_TABLE] = 0;

// Go through all the process table entries, and change the state of the process to READY, which is in WAIT_DISK state.

alias i R1;
i = 0;

while(i < 16) do
    if([PROCESS_TABLE + 16 * i + 4] == WAIT_DISK) then
        [PROCESS_TABLE + 16 * i + 4] = READY;
    endif;
    i = i + 1;
endwhile;

// Restore the register context and return to user mode using the ireturn statement.
restore;
SP=userSP;
[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 9] = 0;
ireturn;
```

> File: int_10.spl
```c
[PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+4] = TERMINATED;

alias i R0;
i=1;  //dnt check for idle
while(i<16) do
 if([PROCESS_TABLE+16*i+4]!=TERMINATED) then
  break;
 endif;
i = i+1;
endwhile;

if(i==16) then
  halt;
endif;

call SCHEDULER;
```

> File: mod_0.spl
```c
alias funcNo R1;
alias currPid R2;

// Aquire Disk Function
// The Acquire Disk function in the resource manager module takes the PID of a process as an argument. The Acquire disk function performs the following tasks 
// function number for AQUIRE_DISK = 3
if(funcNo == 3) then
    // While the disk is busy (STATUS field in the Disk Status Table is 1)
    while([DISK_STATUS_TABLE] == 1) do
        // set the state of the process to WAIT_DISK and invoke the scheduler.
        [PROCESS_TABLE + 16 * currPid + 4] = WAIT_DISK;
        multipush(R1,R2);
        call SCHEDULER;
        multipop(R1,R2);
    endwhile;

    // When the disk is finally free, the process is woken up by the disk interrupt handler.

    // Lock the disk by setting the STATUS and the PID fields in the Disk Status Table to 1 and PID of the process respectively.

    [DISK_STATUS_TABLE] = 1;
    [DISK_STATUS_TABLE + 4] = currPid;

    return;
endif;

// Acquire Terminal
if(funcNo==8) then
while([TERMINAL_STATUS_TABLE]==1) do

 [PROCESS_TABLE+16*currPid+4] = WAIT_TERMINAL;
 multipush(R1,R2);
 call SCHEDULER;
 multipop(R1,R2);
 
endwhile;

[TERMINAL_STATUS_TABLE] = 1;  
[TERMINAL_STATUS_TABLE+1] = currPid;

breakpoint;
return;
endif;

// Release Terminal
alias ret R0;
if(funcNo==9) then
if(currPid!=[TERMINAL_STATUS_TABLE+1]) then
 ret = -1;
 return;
endif;

[TERMINAL_STATUS_TABLE] = 0;
alias i R5;
i=0;
while(i<16) do
 if([PROCESS_TABLE+16*i+4]==WAIT_TERMINAL) then
  [PROCESS_TABLE+16*i+4] = READY;
 endif;
i = i+1;
endwhile;

ret = 0;

breakpoint;
return;
endif;
```

> File: mod_2.spl
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
 ireturn;
endif;

[new_process_table + 4] = RUNNING;

multipop(BP);

return;

```

> File: os_startup.spl
```c
loadi(69,11); //idle
loadi(70,12);

loadi(54,67); //module_7
loadi(55,68);

//.....Calling BOOT Module........//

SP = 82*512-1;  //Kernal Stack of Idle Process
call BOOT_MODULE;

//.....Page Table Entry for IDLE process pid=0.....//

PTBR = PAGE_TABLE_BASE;
PTLR=10;

//.....library.......//
[PTBR+0]= 63;    
[PTBR+1]= "0100";
[PTBR+2]= 64;
[PTBR+3]= "0100";


//.....Heap........//
[PTBR+4]= -1;    
[PTBR+5]= "0000";
[PTBR+6]= -1;
[PTBR+7]= "0000";


//.....code.......//
[PTBR+8]=69;    
[PTBR+9]="0100";
[PTBR+10]=70;
[PTBR+11]="0100";
[PTBR+12]=-1;
[PTBR+13]="0000";
[PTBR+14]=-1;
[PTBR+15]="0000";


//.......stack.......//
[PTBR+16]=81;    
[PTBR+17]="0110";
[PTBR+18]=-1;
[PTBR+19]="0000";

[81*512]=[69*512+1];
SP = 8*512;

//....process table
[PROCESS_TABLE] = 0;    //Tick
[PROCESS_TABLE+1] = 0;    //pid
[PROCESS_TABLE+4] = RUNNING;  //State
[PROCESS_TABLE+11] = 82;  //User Area Page
[PROCESS_TABLE+13] = 8*512;  //UPTR
[PROCESS_TABLE+12] = 0;    //KPTR
[PROCESS_TABLE+14] = PAGE_TABLE_BASE;  //PTBR
[PROCESS_TABLE+15] = 10;  //PTLR

[SYSTEM_STATUS_TABLE+1]=0;  //currently running process

ireturn;
```

> File: sample_int7.spl
```c
[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 9] = 5;

alias userSP R0;
userSP = SP;

[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16)+13]=SP;
SP=[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 11]*512-1;


alias physicalPageNum R1;
alias offset R2;
alias fileDescPhysicalAddr R3;
physicalPageNum = [PTBR + 2 * ((userSP - 4)/ 512)];
offset = (userSP - 4) % 512;
fileDescPhysicalAddr = (physicalPageNum * 512) + offset;
alias fileDescriptor R4;
fileDescriptor=[fileDescPhysicalAddr];



if (fileDescriptor != -2)
then
   alias physicalAddrRetVal R5;
   physicalAddrRetVal = ([PTBR + 2 * ((userSP - 1) / 512)] * 512) + ((userSP - 1) % 512);
   [physicalAddrRetVal] = -1;
else
   alias word R5;
  word = [[PTBR + 2 * ((userSP - 3) / 512)] * 512 + ((userSP - 3) % 512)];
  //print word;

  //Code for S15
  multipush(R0,R1,R2,R3,R4,R5);
  alias func_num R1;
  alias pid R2;
  alias word_print R3;

  func_num=3;
  pid=[SYSTEM_STATUS_TABLE + 1];
  word_print=word;

  //call to device manager.
  call DEVICE_MANAGER;

  multipop(R0,R1,R2,R3,R4,R5);

  alias physicalAddrRetVal R6;
  physicalAddrRetVal = ([PTBR + 2 * (userSP - 1)/ 512] * 512) + ((userSP - 1) % 512);
  [physicalAddrRetVal] = 0;
endif;

SP=userSP;

[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 9] = 0;

ireturn;

```

## Load and Execute

```bash
$> ./compile.sh stage18
```

![[Pasted image 20230921030600.png]]

> File: load18.dat
```.
load --library ../expl/library.lib
load --idle /home/kali/myexpos/expl/expl_progs/stage18/infinite.xsm
load --init /home/kali/myexpos/expl/expl_progs/stage18/name.xsm
load --exec /home/kali/myexpos/expl/expl_progs/stage18/odd_nos.xsm
load --int=10 /home/kali/myexpos/spl/spl_progs/stage18/int_10.xsm
load --int=7 /home/kali/myexpos/spl/spl_progs/stage18/sample_int7.xsm
load --int=6 /home/kali/myexpos/spl/spl_progs/stage18/sample_int6.xsm
load --int=9 /home/kali/myexpos/spl/spl_progs/stage18/int_9.xsm
load --int=console /home/kali/myexpos/spl/spl_progs/stage18/console.xsm
load --int=disk /home/kali/myexpos/spl/spl_progs/stage18/disk.xsm
load --module 0 /home/kali/myexpos/spl/spl_progs/stage18/mod_0.xsm
load --module 5 /home/kali/myexpos/spl/spl_progs/stage18/mod_5.xsm
load --module 4 /home/kali/myexpos/spl/spl_progs/stage18/mod_4.xsm
load --module 7 /home/kali/myexpos/spl/spl_progs/stage18/mod_7.xsm
load --module 1 /home/kali/myexpos/spl/spl_progs/stage18/mod_1.xsm
load --module 2 /home/kali/myexpos/spl/spl_progs/stage18/mod_2.xsm
load --exhandler /home/kali/myexpos/spl/spl_progs/stage18/haltprog.xsm
load --int=timer /home/kali/myexpos/spl/spl_progs/stage18/sample_timer.xsm
load --os /home/kali/myexpos/spl/spl_progs/stage18/os_startup.xsm
exit
```

```bash
$> ./xfs-interface < ../test/load_files/load18.dat
$> ./xsm
Input file:
odd_nos.xsm
1
3
5
7
9
11
13
15
17
19
21
23
25
27
29
31
...
```

---
# Assignment

```ad-question
title: Assignment 1
Use the[XSM debugger](https://exposnitc.github.io/expos-docs/support-tools/xsm-simulator/)to print out the contents of the Disk Status Table after entry and before return from the disk interrupt handler.
```

> File: disk.spl
```c
// When the disk-memory transfer is complete, XSM raises the disk interrupt. The disk interrupt handler then performs the following tasks 

alias userSP R0;
userSP = SP;

// Switch to the kernel stack and back up the register context.
[PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+13] = SP;  
SP = [PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+11]*512-1;

// backup the register contexts
backup;

// breapoint to view the dst before entry
breakpoint;

// Set the STATUS field in the Disk Status table to 0 indicating that disk is no longer busy.

[DISK_STATUS_TABLE] = 0;

// breapoint to view the dst after entry
breakpoint;

// Go through all the process table entries, and change the state of the process to READY, which is in WAIT_DISK state.

alias i R1;
i = 0;

while(i < 16) do
    if([PROCESS_TABLE + 16 * i + 4] == WAIT_DISK) then
        [PROCESS_TABLE + 16 * i + 4] = READY;
    endif;
    i = i + 1;
endwhile;

// Restore the register context and return to user mode using the ireturn statement.
restore;
SP=userSP;
[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 9] = 0;
ireturn;
```

![[Pasted image 20230921031042.png]]

















