# Stage 15

```ad-abstract
- Familiarise with passing of parameters to modules.
- Implement Resource Manager and Device Manager modules for terminal output handling.
```

## **Module 0: Resource Manager**

This module is responsible for allocating and releasing the different resources. Note that the Terminal and Disk devices are freed by the corresponding interrupt handlers.

- **Before the use of a resource, a process has to first acquire the required resource by invoking the resource manager. A process can acquire a resource if the resource is not already acquired by some other process. If the resource requested by a process is not available, then that process has to be blocked until the resource becomes free.**
- In the meanwhile, other processes may be scheduled.

### Terminal Status Table[¶](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#terminal-status-table)

The Terminal Status Table keeps track of the Read/Write operations done on the terminal. Every time a Read or Write system call is invoked on the terminal, the PID of the process that invoked the system call is stored in Terminal Status Table. The table also contains information on the status of the terminal (whether free or busy). The size of the table is 4 words of which the last 2 are unused.

Every entry of the Terminal Status Table has the following format:

|STATUS|PID|Unused|
|---|---|---|

- **STATUS** (1 word) - specifies whether the terminal is free or is being used by a process to read or write. This field is initially set to 0. It is changed to 1 whenever terminal is busy. The Terminal Interrupt Handler sets back the status to 0 upon completion of Terminal Read. The Write system call sets back the status to 0 upon completion of Terminal Write.
- **PID** (1 word) - specifies the PID of the process which is currently using the terminal. This field is invalid when STATUS is 0.
- **Unused** (2 words)

**Note**

The Terminal Table is present in page 57 of the memory (see [Memory Organisation](https://exposnitc.github.io/expos-docs/os-implementation/)), and the SPL constant [TERMINAL_STATUS_TABLE](https://exposnitc.github.io/expos-docs/support-tools/constants/) points to the starting address of the table.

### Acquire Terminal[¶](https://exposnitc.github.io/expos-docs/modules/module-00/#acquire-terminal)

_**Description**_ : Locks the Terminal device. Assumes a valid PID is given as input.

```

while ( Terminal device is locked ){    /* Check the Status field in theTerminal Status Table */
    Set state of the process as ( WAIT_TERMINAL , - );
    Call theswitch_context() function from theScheduler Module.
}

Lock the Terminal device by setting the Status and PID fields in theTerminal Status Table.

return;

```

Called by the Terminal Read and Terimnal Write functions of the [Device Manager Module](https://exposnitc.github.io/expos-docs/modules/module-04/).

### Release Terminal[¶](https://exposnitc.github.io/expos-docs/modules/module-00/#release-terminal)

_**Description**_ : Frees the Terminal device. Assumes a valid PID is given as input.

```

If PID given as input is not equal to the LOCKING PID in theTeminal Status Table, return -1.

    Release the lock on the Terminal by updating the Terminal Status Table.;

    loop through the process table{
       if (the process state is ( WAIT_TERMINAL , - ) ){
             Set state of process as (READY , _ )
         }
    }

    Return 0

```

Called by the Terimnal Write function in the [Device Manager Module](https://exposnitc.github.io/expos-docs/modules/module-04/).

# Module 4: Device Manager

Handles Terminal I/O and Disk operations (Load and Store).

|Function Number|Function Name|Arguments|Return Value|
|---|---|---|---|
|DISK_STORE = 1|Disk Store|PID, Page Number, Block Number|NIL|
|DISK_LOAD = 2|Disk Load|PID, Page Number, Block Number|NIL|
|TERMINAL_WRITE = 3|Terminal Write|PID, Word|NIL|
|TERMINAL_READ = 4|Terminal Read|PID, Address|NIL|

![https://exposnitc.github.io/expos-docs/assets/img/modules/DeviceManager.png](https://exposnitc.github.io/expos-docs/assets/img/modules/DeviceManager.png)

### Terminal Write[¶](https://exposnitc.github.io/expos-docs/modules/module-04/#terminal-write)

_**Description**_ : Reads a word from the Memory address provided to the terminal. Assumes a valid PID is given.

```

    Acquire the lock on the terminal device by calling the Acquire_Terminal() function
    in theResource Manager module;

    Use the print statement to print the contents of the word
    to the terminal;

    Release the lock on the terminal device by calling the Release_Terminal() function
    in theResource Manager module;

    return;

```

Called by the Write system call.

### Terminal Read[¶](https://exposnitc.github.io/expos-docs/modules/module-04/#terminal-read)

_**Description**_ : Reads a word from the terminal and stores it to the memory address provided. Assumes a valid PID is given.

```

Acquire the lock on the disk device by calling the Acquire_Terminal() function
in theResource Manager module;

Use the read statement to read the word from the terminal;

Set the state as (WAIT_TERMINAL, - );

Call theswitch_context() function from theScheduler Module.

Copy the word from theInput Buffer of theProcess Table of the process corresponding to PID
to the memory address provided.

return;

```

Called by the Read system call.

> The Terminal Interrupt Handler will transfer the contents of the input port P0 to the Input Buffer of the process.

## Setting up

- Abstract of how this works

<aside> 💡 Acquire Terminal and Release Terminal are not directly invoked from the write system call. Write system call invokes a function called Terminal Write present in [device manager module](https://exposnitc.github.io/expos-docs/modules/module-04/) (Module 4). Terminal Write function acts as anabstract layer between the write system call and terminal handling functions in resource manager module. The function number for Terminal Write is 3 which is stored in register R1. The other arguments are PID of the current process and the word to be printed which are passed through registers R2 and R3 respectively. Terminal Write first acquires the terminal by calling Acquire Terminal. It prints the word (present in R3) passed as an argument. It then frees the terminal by invoking Release Terminal.

</aside>

- Since the invoked module will be modifying the contents of the machine registers during its execution, The invoker must save the registers in use into the (kernel) stack of the process before invoking the module. The module sets its return value in register R0 before returning to the caller. The invoker must extract the return value, pop back the saved registers and resume execution. SPL provides the facility to push and pop multiple registers in one statement using multipush and mutlipop respectively. Refer to the usage of multipush and multipop statements in [SPL](https://exposnitc.github.io/expos-docs/support-tools/spl/) before proceeding further.

> There is one important conceptual point to be explained here relating to resource acquisition. The Acquire Terminal function described above waits in a loop, in which it repeatedly invokes the scheduler if the terminal is not free. **This kind of a waiting loop is called a busy loop or [busy wait](https://en.wikipedia.org/wiki/Busy_waiting).**

<aside> 💡 `Why can't the process wait just once for a resource and simply proceed to acquire the resource when it is scheduled?`

The reason a process can't wait just once for a resource and then proceed when it's scheduled is because of the potential for contention and timing issues in a multi-process environment. Let's break it down:

**1. Competition for Resources:** In a computer system, multiple processes run concurrently, and they often need to use shared resources like memory, printers, or network connections. Since these resources can't be used by more than one process at the same time, a mechanism is needed to manage how processes access and use them.

**2. Timing and Unpredictability:** When a process requests a resource and then waits for it to be available, there's no guarantee that the resource will be immediately free when the process is scheduled to run again. This is because other processes might also be competing for the same resource, and they might get scheduled before the waiting process does.

**3. Risk of Missed Opportunities:** If a process waits just once and then proceeds when it's scheduled, it might miss its chance to use the resource even if it becomes available during its scheduled time. For instance, imagine a process waiting for a printer. If the printer becomes available while another process is running, the waiting process might not have a chance to execute until after the printer is taken again.

**4. Fairness and Efficiency:** Allowing a process to wait just once and proceed might lead to unfair resource allocation. One process could potentially monopolize a resource, causing other processes to wait indefinitely. This wouldn't be an efficient use of resources and could degrade system performance.

**5. Busy Loops and Waiting:** The concept of using a busy loop (busy wait) to repeatedly check for resource availability is used to prevent missed opportunities. By continuously checking for resource availability, a process increases its chances of grabbing the resource as soon as it becomes available. While this may seem inefficient, it's used as a basic technique to manage resource contention.

In summary, the main challenge is the unpredictability of resource availability and the potential for unfairness if processes only wait once. Using a busy loop ensures that a process has a better chance of acquiring a resource as soon as it's available, and it prevents monopolization by any single process. While busy loops might not be the most efficient solution in all cases, they serve as a fundamental mechanism to handle resource contention in multi-process environments.

</aside>

![[Pasted image 20230831001952.png]]


#### Modifying INT 7 routine[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-15/#modifying-int-7-routine "Permanent link")

Interrupt routine 7 implemented in stage 10 is modified as given below to invoke Terminal Write function present in Device Manager module. Instead of print statement, write code to invoke Terminal Write function. Rest of the code remains intact.

1) Push all registers used till now in this interrupt routine using multipush statement in SPL.

`multipush(R0, R1, R2, R3,...); // number of registers will depend on your code`

2) Store the function number of Terminal Write in register R1, PID of the current process in register R2 and word to be printed to the terminal in register R3.

3) Call module 4 using [call statement](https://exposnitc.github.io/expos-docs/support-tools/spl/).

4) Ignore the value present in R0 as Terminal Write does not have any return value.

5) Use multipop statement to restore the registers pushed. Specify the same order of registers used in multipush as registers are popped in the **reverse order** in which they are specified in the multipop statement.

`multipop(R0, R1, R2, R3,...);`

#### Implementation of Module 4 (Device Manager Module)[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-15/#implementation-of-module-4-device-manager-module "Permanent link")

In this stage, we will implement only Terminal Write function in this module.

1) Function number and current PID are stored in registers R1 and R2. Give meaningful names to these arguments.

`alias functionNum R1; alias currentPID R2;`

2) Terminal write function has a function number 3. If the functionNum is 3, implement the following steps else return using return statement.

Calling Acquire Terminal :-

3) Push all the registers used till now in this module using the multipush statement in SPL as done earlier.

4) Store the function number 8 in register R1 and PID of the current process from the [System Status table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#system-status-table) in register R2 (Can use currentPID, as it already contain current PID value).

5) Call module 0.

6) Ignore the value present in R0 as Acquire Terminal does not have any return value.

7) Use the multipop statement to restore the registers as done earlier.

8) Print the word in register R3, using the print statement.

Calling Release Terminal :-

9) Push all the registers used till now using the multipush statement as done earlier.

10) Store the function number 9 in register R1 and PID of the current process from the System Status table in register R2 (Can use currentPID, as it already contain current PID value).

11) Call module 0.

12) Return value will be stored in R0 by module 0. Save this return value in any other register if needed.

13) Restore the registers using the multipop statement.

14) Return using the return statement.

#### Implementation of Module 0 code for terminal handling[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-15/#implementation-of-module-0-code-for-terminal-handling "Permanent link")

1) Function number is present in R1 and PID passed as an argument is stored in R2. Give meaningful names to these registers to use them further.

`alias functionNum R1; alias currentPID R2;`

2) In Module 0, for the Acquire Terminal function (functionNum = 8) implement the following steps.

1. **The current process should wait in a loop until the terminal is free**. Repeat the following steps if STATUS field in the Terminal Status table is 1(terminal is allocated to other process).
    1. Change the state of the current process in its process table entry to WAIT_TERMINAL.
    2. Push the registers used till now using the multipush statement.
    3. Call the scheduler to schedule other process as this process is waiting for terminal.
    4. Pop the registers pushed before. (Note that this code will be executed only after the scheduler schedules the process again, which in turn occurs only after the terminal was released by the holding process by invoking the release terminal function.)
2. Change the STATUS field to 1 and PID field to currentPID in the Terminal Status Table.
3. Return using the return statement.

3) For the Release Terminal function (functionNum = 9) implement the following steps.

1. currentPID and PID stored in the Terminal Status table should be same. If these are not same, then process is trying to release the terminal without acquiring it. If this case occurs, store -1 as the return value in register R0 and return from the module.
2. Change the STATUS field in the Terminal Status table to 0, indicating terminal is released.
3. Update the STATE to READY for all processes (with valid PID) which have STATE as WAIT_TERMINAL.
4. Save 0 in register R0 indicating success.
5. Return to the caller.

#### Modifying Boot Module code[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-15/#modifying-boot-module-code "Permanent link")

1. Load Module 0 from disk pages 53 and 54 to memory pages 40 and 41.
2. Load Module 4 from disk pages 61 and 62 to memory pages 48 and 49.
3. Initialize the STATUS field in the [Terminal Status table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#terminal-status-table)as 0. This will indicate that the terminal is free before scheduling the first process.

#### Making things work[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-15/#making-things-work "Permanent link")

1. Compile and load boot module code, module 0, module 4, modified INT 7 routine using XFS-interface.
2. Run the machine with two programs one printing even numbers and another printing odd numbers from 1 to 100 along with the idle process.

---

```ad-question
title: According to eXpOS resource management system introduced here, will Deadlock occur? If yes, explain it with a situation. If no, which of the four conditions of Deadlock are not satisfied?

Deadlock will not occur according to the resource management system implemented here. As hold and wait, circular wait conditions are not satisfied (there is only one resource - the terminal - now).
```

```ad-info
See [link](https://en.wikipedia.org/wiki/Deadlock#Necessary_conditions) for a set of neccessary conditions for deadlock.
```

---

# Assignment 1

```ad-question
Set a **breakpoint** (see [SPL breakpoint instruction](https://exposnitc.github.io/expos-docs/support-tools/spl/)) just before return from the Acquire Terminal and the Release Terminal functions in the Resource Manager module to dump the Terminal Status Table (see [XSM debugger](https://exposnitc.github.io/expos-docs/support-tools/xsm-simulator/)for various printing options).
```

### Files Required
1. [x] boot_module.spl
2. [x] haltprog.spl
3. [x] int_10.spl
4. [x] mod_0.spl
5. [x] mod_4.spl
6. [x] mod_5.spl
7. [x] os_startup.spl
8. [x] sample_int7.spl
9. [x] sample_timer.spl
10. [ ] even_nos.expl
11. [ ] odd_nos.expl
12. [ ] prime.expl 

> File: boot_module.spl
```c
//MOD_0
loadi(40,53);
loadi(41,54);

//MOD_4
loadi(48,61);
loadi(49,62);

//lib
loadi(63,13);
loadi(64,14);

//even.xsm
loadi(84,69);

//scheduler
loadi(50,63);
loadi(51,64);

//init
loadi(65,7);
loadi(66,8);

//int 10
loadi(22,35);
loadi(23,36);

//int 7
loadi(1*16,29);
loadi(17,30);

//exception
loadi(2,15);
loadi(3,1*16);

//timer
loadi(4,17);
loadi(5,18);

//FOR INIT
//pid
[PROCESS_TABLE+(1*16+1)]=1;

//userareapgnum
//Setting user area pg num
[PROCESS_TABLE + (1*16+11)]=80;
//UPTR FOR INIT
[PROCESS_TABLE + (1*16+13)]=8*512;

//KPTR FOR INIT
[PROCESS_TABLE+(1*16+12)]=0;

[PROCESS_TABLE+(1*16+14)]=(PAGE_TABLE_BASE+20);
[PROCESS_TABLE+(1*16+15)]=10;
[PROCESS_TABLE+(1*16+4)]=CREATED;

//init
PTBR=PAGE_TABLE_BASE+20;
PTLR=10;

//lib
[PTBR+0]=63;
[PTBR+1]="0100";
[PTBR+2]=64;
[PTBR+3]="0100";

//heap
[PTBR+4]=78;
[PTBR+5]="0110";
[PTBR+6]=79;
[PTBR+7]="0110";

//Code
[PTBR+8]=65;
[PTBR+9]="0100";
[PTBR+10]=66;
[PTBR+11]="0100";
[PTBR+12]=-1;
[PTBR+13]="0000";
[PTBR+14]=-1;
[PTBR+15]="0000";

//stack
[PTBR+1*16]=76;
[PTBR+17]="0110";
[PTBR+18]=77;
[PTBR+19]="0110";

[76 * 512] = [65*512 + 1];

//FOR even.xsm
//pid
[PROCESS_TABLE+(2*16+1)]=2;

//user area pagenum
//Setting user area pg num
[PROCESS_TABLE + (2*16+11)]=85;
//UPTR
[PROCESS_TABLE + (2*16+13)]=8*512;

//KPTR
[PROCESS_TABLE+(2*16+12)]=0;

[PROCESS_TABLE+(2*16+14)]=(PAGE_TABLE_BASE+40);
[PROCESS_TABLE+(2*16+15)]=10;
[PROCESS_TABLE+(2*16+4)]=CREATED;

//
PTBR=PAGE_TABLE_BASE+40;
PTLR=10;

//lib
[PTBR+0]=63;
[PTBR+1]="0100";
[PTBR+2]=64;
[PTBR+3]="0100";

//heap
[PTBR+4]=86;
[PTBR+5]="0110";
[PTBR+6]=87;
[PTBR+7]="0110";

//Code
[PTBR+8]=84;
[PTBR+9]="0100";
[PTBR+10]=-1;
[PTBR+11]="0000";
[PTBR+12]=-1;
[PTBR+13]="0000";
[PTBR+14]=-1;
[PTBR+15]="0000";

//stack
[PTBR+1*16]=88;
[PTBR+17]="0110";
[PTBR+18]=89;
[PTBR+19]="0110";

[88 * 512] = [84*512 + 1];

//other process table entries,state=terminated
[PROCESS_TABLE + 3*16 + 4] = TERMINATED;
[PROCESS_TABLE + 4*16 + 4] = TERMINATED;
[PROCESS_TABLE + 5*16 + 4] = TERMINATED;
[PROCESS_TABLE + 6*16 + 4] = TERMINATED;
[PROCESS_TABLE + 7*16 + 4] = TERMINATED;
[PROCESS_TABLE + 8*16 + 4] = TERMINATED;
[PROCESS_TABLE + 9*16 + 4] = TERMINATED;
[PROCESS_TABLE + 10*16 + 4] = TERMINATED;
[PROCESS_TABLE + 11*16 + 4] = TERMINATED;
[PROCESS_TABLE + 12*16 + 4] = TERMINATED;
[PROCESS_TABLE + 13*16 + 4] = TERMINATED;
[PROCESS_TABLE + 14*16 + 4] = TERMINATED;
[PROCESS_TABLE + 15*16 + 4] = TERMINATED;

//set status=0 in terminal status table
[TERMINAL_STATUS_TABLE]=0;

return;
```

> File: haltprog.spl
```c
halt;
```

> File: int_10.spl
```c
[PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+4] = TERMINATED;

alias i R0;
i=1;	//dnt check for idle
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

// Acquire Terminal
if(funcNo==8) then
while([TERMINAL_STATUS_TABLE]==1) do

 [PROCESS_TABLE+16*currPid+4] = WAIT_TERMINAL;
 multipush(R1,R2,R3);
 call SCHEDULER;
 multipop(R1,R2,R3);
 
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

> File: mod_4.spl
```c
alias func_num R1;
alias currPID R2;

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

return;
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

SP = 82*512-1;	//Kernal Stack of Idle Process
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
[PROCESS_TABLE] = 0;		//Tick
[PROCESS_TABLE+1] = 0;		//pid
[PROCESS_TABLE+4] = RUNNING;	//State
[PROCESS_TABLE+11] = 82;	//User Area Page
[PROCESS_TABLE+13] = 8*512;	//UPTR
[PROCESS_TABLE+12] = 0;		//KPTR
[PROCESS_TABLE+14] = PAGE_TABLE_BASE;	//PTBR
[PROCESS_TABLE+15] = 10;	//PTLR

[SYSTEM_STATUS_TABLE+1]=0;	//currently running process

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

> File: odd_nos.expl
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
	         temp = exposcall ( "Write" , -2, num );
         endif;
         num = num + 1;
    endwhile;
    return 0;
end
}
```

> File: even_nos.expl
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
    	 if( check == 0) then
	         temp = exposcall ( "Write" , -2, num );
         endif;
         num = num + 1;
    endwhile;
    return 0;
end
}
```

> File: prime_nos.expl
```c
decl
	int IsPrime(int n);
enddecl

int IsPrime(int n)
{
decl
	int i,check,res;
enddecl

begin
	res = 1;
	i = 2;
    while (  (i*i) <= n ) do
    	 check = (n % i);
    	 if( check == 0) then
    	 	 res = 0;
         endif;
         i = i+1;
    endwhile;

	return res;
end
}

int main()
{
decl 
	int i,temp;
enddecl
begin
	i = 2;
	while( i <= 100) do
		if(i <= 1) then
			temp = exposcall("Write", -2, i);
			i = i + 1;
		endif;
		if(IsPrime(i) == 1) then
			temp = exposcall("Write", -2, i);
		endif;
		i = i + 1;
	endwhile;
	return 0;
end
}
```

---
# Compile and Executing
### 1. Compiling
```bash
$> ./compile.sh stage15
```

![[Pasted image 20230831013940.png]]
### 2. Loading File
> File: load15.dat
```bash
load --library ../expl/library.lib
load --idle /home/kali/myexpos/expl/expl_progs/stage15/even_nos.xsm
load --init /home/kali/myexpos/expl/expl_progs/stage15/odd_nos.xsm
load --exec /home/kali/myexpos/expl/expl_progs/stage15/prime.xsm
load --int=10 /home/kali/myexpos/spl/spl_progs/stage15/int_10.xsm
load --int=7 /home/kali/myexpos/spl/spl_progs/stage15/sample_int7.xsm
load --module 0 /home/kali/myexpos/spl/spl_progs/stage15/mod_0.xsm
load --module 5 /home/kali/myexpos/spl/spl_progs/stage15/mod_5.xsm
load --module 4 /home/kali/myexpos/spl/spl_progs/stage15/mod_4.xsm
load --module 7 /home/kali/myexpos/spl/spl_progs/stage15/boot_module.xsm
load --exhandler /home/kali/myexpos/spl/spl_progs/stage15/haltprog.xsm
load --int=timer /home/kali/myexpos/spl/spl_progs/stage15/sample_timer.xsm
load --os /home/kali/myexpos/spl/spl_progs/stage15/os_startup.xsm
exit
```
### 3. Executing
```bash
$> ./xfs-interface < ../test/load15.dat
$> ./xsm --debug --timer 80
debug> tst
debug> c
```

![[Pasted image 20230831014238.png]]

```bash
$> ./xsm --timer 80
```

![[Pasted image 20230831014540.png]]

















