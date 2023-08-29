# Stage 14
```ad-abstract
Implement a preliminary version of the Round Robin scheduling algorithm as an eXpOS module.
```

---

Multiprogramming refers to running more than one process simultaneously. In this stage, you will implement an initial version of the [Round Robin scheduler](https://en.wikipedia.org/wiki/Round-robin_scheduling) used in eXpOS. You will hand create another user process (apart from idle and init) and schedule its execution using the timer interrupt.

1) Write an ExpL program to print the odd numbers from 1-100. Load this odd program as init.
```nasm
load --init <path to odd.xsm>;
```

Using the `--exec` option of XFS interface, you can load an executable program into the XFS disk. XFS interface will load the executable into the disk and create [Inode table](https://exposnitc.github.io/expos-docs/os-design/disk-ds/#inode-table) entry for the file. XFS interface will also create a [root entry](https://exposnitc.github.io/expos-docs/os-design/disk-ds/#root-file) for the loaded file. From the Inode Table Entry, you will be able to find out the disk blocks where the contents of the file are loaded by XFS interface. Recall that these were discussed in detail in Stage 2 : [Understanding the File System](https://exposnitc.github.io/expos-docs/roadmap/stage-02/))

2) Write an ExpL program to print the even numbers from 1-100. Load this even program as an executable.
```nasm
load --exec <path to even.xsm>;
```

Dump the [inode table](https://exposnitc.github.io/expos-docs/os-design/disk-ds/#inode-table)using _dump --inodeusertable_ command in xfs-interface. Check the disk address of code blocks of even.xsm.

#### Modifications to the boot module code[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-14/#modifications-to-the-boot-module-code "Permanent link")

1) Load the code pages of the even program from disk to memory.

2) Set the [Process Table](https://exposnitc.github.io/expos-docs/os-design/process-table/) entry and [PageTable](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-page-table) entries for setting up a process for the even program. You should set up the PTBR, PTLR, UPTR, KPTR, User Area Page Number etc. and also initialize the process state as CREATED in the process table entry for the even process. Set the PID field in the process table entry to 2.

_Make sure that you do not allot memory pages that are already allotted to some other process or reserved for the operating system._

3) Set the starting IP of the new process on top of its user stack.

4) We will implement the scheduler as a seperate module that can be invoked from the timer ISR (Interrupt Service Routine).

The eXpOS design stipulates that the scheduler is implemented as MODULE_5, and loaded in [disk blocks](https://exposnitc.github.io/expos-docs/os-implementation/) 63 and 64 of the XFS disk. The boot module must load this module from disk to [memory pages](https://exposnitc.github.io/expos-docs/os-implementation/) 50 and 51. (We will take up the implementation of the module soon below).

```nasm
loadi(50,63);
loadi(51,64);
```

1) First 3 process table entries are occupied. Initialize STATE field of all other process table entries to TERMINATED. This will be useful while finding the next process to schedule using round robin scheduling algorithm. Note that when the STATE field in the process table entry is marked as TERMINATED, this indicates that the process table entry is free for allocation to new processes.

#### Modifications to Timer Interrupt Routine[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-14/#modifications-to-timer-interrupt-routine "Permanent link")

As we are going to write the scheduler code as a separate module (MOD_5), we will modify the timer interrupt routine so that it calls that module.

When the timer ISR calls the scheduler, the active kernel stack will be that of the currently RUNNING process. The scheduler assumes that the timer handler would have saved the user context of the current process (values of R0-R19 and BP registers) into the kernel stack before the call. It also assumes that the state of the process has been changed to READY.However, the machine's SP register will still point to the top of the kernel stack of the currently running process at the time of the call.

The scheduler first saves the values of the registers SP, PTBR and PTLR to the process table entry of the current process. Next, it must decide which process to run next. This is done using the [Round Robin Scheduling algorithm](https://en.wikipedia.org/wiki/Round-robin_scheduling). Having decided on the new process, the scheduler loads new values into SP, PTBR and PTLR registers from the process table entry of the new process. It also updates the system status table to store PID of new process. If the state of the new process is READY, then the scheduler changes the state to RUNNING. Now, the scheduler returns using the return instruction.

```ad-note
The control flow at this point is tricky and must be carefully understood. The key point to note here is that although the scheduler module was called by one process (from the timer ISR), since the stack was changed inside the scheduler, **the return is to a program instruction in some other process! (determined by the value on top of the kernel stack of the newly scheduled process)**.

The return is to that instruction which immediately follows the _call scheduler_ instruction in the newly selected process. (why? - ensure that you understand this point clearly.) An exception to this rule happens only when the newly selected process to be scheduled is in the CREATED state.

Here, the process was never run and hence there is no return address in the kernel stack. Hence, the scheduler directly kick-starts execution of the process by initiating user mode execution of the process (using the ireturn instruction). The design of eXpOS guarantees that a process can invoke the scheduler module only from the kernel mode. Consequently, the return address will be always stored on top of the kernel stack of the process.

The round robin scheduling algorithm generally schedules the "next process" in the process table that is in CREATED/READY state. (There are exceptions to this rule, which we will encounter in later stages.) Moreover, in the present stage, a process will invoke the scheduler only from the timer interrupt. We will see other situations in later stages.
```

As noted above, the timer resumes execution from the return address stored on the top of the kernel stack of the new process. The timer will restore the user context of the new process from the stack and return to the user mode, resulting in the new process being executed.

If the scheduler finds that the new process is in state CREATED and not READY, then as noted above, the timer ISR would not have set any return address in its kernel stack previously. In this case, the scheduler will set the state of the process to RUNNING and initialize machine registers PTLR and PTBR. Now, the scheduler proceeds to run the process in user mode.Hence, SP is set to the top of the user stack. The scheduler then starts the execution of the new process by transferring control to user mode using the IRET instruction.

The scheduler expects that when a process is in the CREATED state, the following values have been already set in the process table. (In the present stage, the OS startup code/Boot module is responsible for setting up these values.)

1. The state of the process has been set to CREATED.
2. The UPTR field of the process table entry has been set to the top of the user stack (and the stack-top contains the address of the instruction to be fetched next when the process is run in the user mode).
3. PTBR, PTLR, User Area Page Number and KPTR fields in the process table entry has been set up.

**It is absolutely necessary to be clear about [Kernel Stack Management during Module calls](https://exposnitc.github.io/expos-docs/os-design/stack-module/) and [Kernel Stack Management during Context Switch](https://exposnitc.github.io/expos-docs/os-design/timer-stack-management/) before proceeding further.**

Modify the timer interrupt routine as explained above using the algorithm given [here](https://exposnitc.github.io/expos-docs/os-design/timer/). (Ignore the part relating to the swapping operation as it will be dealt in a later stage.)

#### Context Switch Module (Scheduler Module)[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-14/#context-switch-module-scheduler-module "Permanent link")

The scheduler module (module 5) saves the values of SP, PTBR and PTLR registers of the current process in its process table entry. It finds a new process to schedule which is in READY or CREATED state and has a valid PID (PID not equal to -1). Initialize the registers SP, PTBR, PTLR according to the values present in the process table entry of the new process selected for scheduling. Also update the System status table.

Write an SPL program for the scheduler module (module 5) as given below:

1. Obtain the PID of the current process from the [System Status Table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#system-status-table).
2. Push the BP of the current process on top of the kernel stack. (See the box below)
3. Obtain the [Process Table](https://exposnitc.github.io/expos-docs/os-design/process-table/) entry corresponding to the current PID.
4. Save SP % 512 in the kernel stack pointer field, also PTBR and PTLR into the corresponding fields in the Process Table entry.
5. Iterate through the Process Table entries, starting from the succeeding entry of the current process to find a process in READY or CREATED state.
6. If no such process can be found, select the idle process as the new process to be scheduled. Save PID of new process to be scheduled as newPID.
7. Obtain User Area Page number and kernel stack pointer value from Process Table entry of the new process and set SP as (User Area Page number) * 512 + (Kernel Stack pointer value).
8. Restore PTBR and PTLR from the corresponding fields in the Process Table entry of the new process.
9. Set the PID of the new process in the current PID field of the System Status Table.
10. If the new process is in CREATED state, then do the following steps.
    1. Set SP to the value in the UPTR field of the process table entry.
    2. Set state of the newly scheduled process as RUNNING.
    3. Store 0 in the MODE FLAG field in the process table of the process.
    4. Switch to the user mode using the ireturn statement.
11. Set the state of the new process as RUNNING.
12. Restore the BP of the new process from the top of it's kernel stack.
13. Return using return statement.

```ad-note
In later stages you will modify the scheduler module to the final form given [here](https://exposnitc.github.io/expos-docs/modules/module-05/).
```

```ad-note
In the present stage, the scheduler module is called only from the time interrupt handler. The timer interrupt handler already contains the instruction to backup the register context of the current process. Hence, the scheduler does not have to worry about having to save the user register context (including the value of the BP register) of the current process. What then is the need for the scheduler to push the BP register?

The reason is that, in later stages, the scheduler may be called from [kernel modules](https://exposnitc.github.io/expos-docs/os-design/) other than the timer interrupt routine. Such calls typically happen when an application invokes a [system call](https://exposnitc.github.io/expos-docs/os-design/)and the system call routine invokes a kernel module which in turn invokes the scheduler. Whenever this is the case, the OS kernel expects that the application saves all the user mode registers **except the BP register** before making the system call.

For instance, if the application is written in ExpL and compiled using the ExpL compiler given to you, the compiler saves all the user registers**except BP** before making the system call. The ExpL compiler expects that the OS will save the value of the BP register before scheduling another application process. This explains why the scheduler needs to save the BP register before a context switch.
```

#### Modifications to INT 10 handler[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-14/#modifications-to-int-10-handler "Permanent link")

The ExpL compiler sets every user program to execute the INT 10 instruction (exit system call) at the end of execution to terminate the process gracefully. In previous stages, we wrote an INT 10 routine containing just a halt instruction. Hence, if any process invoked INT 10 upon exit, the machine would halt and no other process would execute further. However, to allow multiple processes to run till completion, INT 10 must terminate only the process which invoked it, and schedule other surviving processes. (INT 10 shall set the state of the dying process to TERMINATED). If all processes except idle are in TERMINATED state, then INT 10 routine can halt the system.

Write INT 10 program in SPL following below steps :

1. Change the state of the invoking process to [TERMINATED](https://exposnitc.github.io/expos-docs/support-tools/constants/).
2. Find out whether all processes except idle are terminated. In that case, halt the system.Otherwise invoke the scheduler

There will be no return to this process as the scheduler will never schedule this process again.

#### Making things work[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-14/#making-things-work "Permanent link")

1. Compile and load the Boot module code, timer interrupt routine, scheduler module (module 5) and interrupt 10 routine into disk using XFS interface.
2. Run XSM machine with timer enabled.

---

```ad-question
title: When does the OS kernel invoke the scheduler from some routine other than the timer interrupt handler?
In later stages, if a process gets blocked inside a kernel module (waiting for some resource), then the process will set its state to "WAITING" and will invoke the scheduler. Later when the process is back in READY state (as the resource becomes free) and the scheduler selects the process for running, execution returns to the instruction following the call to the scheduler in the kernel module.
```

---

# Assignment 1
```ad-question
Write ExpL programs to print odd numbers, even numbers and prime numbers between 1 and 100. Modify the boot module code accordingly and run the machine with these 3 processes along with idle process.
```

```nasm
loadi(65,7); //init
loadi(66,8);

loadi(88,69);  //exec file

loadi(22, 35); //int10
loadi(23, 36);

loadi(2,15); //excep handler
loadi(3,16);

loadi(63,13); //library
loadi(64,14);

loadi(4,17); //timer int
loadi(5,18);

loadi(16,29); //write int
loadi(17,30);

loadi(50,63); //Shedular_module
loadi(51,64);

loadi(48, 61); //MOD_4
loadi(49, 62);

loadi(50,63); //MOD_7
loadi(51,64);

//.....Page Table Entry for INIT process pid=1.....//

PTBR = PAGE_TABLE_BASE+20;
PTLR=10;

//.....library.......//
[PTBR+0]= 63;
[PTBR+1]= "0100";
[PTBR+2]= 64;
[PTBR+3]= "0100";


//.....Heap........//
[PTBR+4]= 78;
[PTBR+5]= "0110";
[PTBR+6]= 79;
[PTBR+7]= "0110";


//.....code.......//
[PTBR+8]=65;
[PTBR+9]="0100";
[PTBR+10]=66;
[PTBR+11]="0100";
[PTBR+12]=-1;
[PTBR+13]="0000";
[PTBR+14]=-1;
[PTBR+15]="0000";


//.......stack.......//
[PTBR+16]=76;
[PTBR+17]="0110";
[PTBR+18]=77;
[PTBR+19]="0110";

[76*512]=[65*512+1];

//....process table
[PROCESS_TABLE+16] = 0; 	   //Tick
[PROCESS_TABLE+16+1] = 1;          //pid
[PROCESS_TABLE+16+4] = CREATED;    //State
[PROCESS_TABLE+16+11] = 80;        //User Area Page
[PROCESS_TABLE+16+13] = 8*512;     //UPTR
[PROCESS_TABLE+16+12] = 0;         //KPTR
[PROCESS_TABLE+16+14] = PAGE_TABLE_BASE+20;   //PTBR
[PROCESS_TABLE+16+15] = 10;        //PTLR

//.....Page Table Entry for Exec file process pid=2.....//

PTBR = PAGE_TABLE_BASE+40;
PTLR=10;

//.....library.......//
[PTBR+0]= 63;
[PTBR+1]= "0100";
[PTBR+2]= 64;
[PTBR+3]= "0100";


//.....Heap........//
[PTBR+4]= 83;
[PTBR+5]= "0110";
[PTBR+6]= 84;
[PTBR+7]= "0110";


//.....code.......//
[PTBR+8]=88;
[PTBR+9]="0100";
[PTBR+10]=-1;
[PTBR+11]="0000";
[PTBR+12]=-1;
[PTBR+13]="0000";
[PTBR+14]=-1;
[PTBR+15]="0000";


//.......stack.......//
[PTBR+16]=85;
[PTBR+17]="0110";
[PTBR+18]=86;
[PTBR+19]="0110";

[85*512]=[88*512+1];

//....process table
[PROCESS_TABLE+16*2] = 0;  	     //Tick
[PROCESS_TABLE+16*2+1] = 2;          //pid
[PROCESS_TABLE+16*2+4] = CREATED;    //State
[PROCESS_TABLE+16*2+11] = 87;        //User Area Page
[PROCESS_TABLE+16*2+13] = 8*512;     //UPTR
[PROCESS_TABLE+16*2+12] = 0;         //KPTR
[PROCESS_TABLE+16*2+14] = PAGE_TABLE_BASE+40;   //PTBR
[PROCESS_TABLE+16*2+15] = 10;        //PTLR

alias i R0;
i = 3;
while(i<16) do
[PROCESS_TABLE+16*i+4] = TERMINATED;
i = i+1;
endwhile;

[TERMINAL_STATUS_TABLE] = 0;  	     
return;
```

```nasm
halt;
```

```c
[PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+4] = TERMINATED;

alias i R0;
i=1;
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

restore;
SP = [PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+13];     //User_SP
[PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+9] = 0;       //MODE_FLAG
ireturn;


```

```nasm
alias currPID R6;
currPID = [SYSTEM_STATUS_TABLE + 1];

// push BP
multipush(BP);

[PROCESS_TABLE + currPID * 16 + 12] = SP % 512;
[PROCESS_TABLE + currPID * 16 + 14] = PTBR;
[PROCESS_TABLE + currPID * 16 + 15] = PTLR;

alias i R7;
alias flag R8;
alias newPID R10;
flag=0;
i =(1+currPID )% 16;

while(i != currPID)do  
  if( [PROCESS_TABLE + i*16 + 4] == CREATED || [PROCESS_TABLE + i*16 + 4] == READY ) then
    flag = 1;
    newPID = i; 
    break;
  endif;
  i=(i+1)%16;
endwhile; 
 
if(flag == 0) then
  newPID = 0;
endif;

SP = [PROCESS_TABLE + newPID * 16 + 11]* 512 + [PROCESS_TABLE + newPID * 16 + 12];
PTBR = [PROCESS_TABLE + newPID * 16 + 14];
PTLR = [PROCESS_TABLE + newPID * 16 + 15];

[SYSTEM_STATUS_TABLE + 1] = newPID;

if([PROCESS_TABLE + newPID * 16 +4 ] == CREATED) then
  SP = [PROCESS_TABLE + newPID * 16 + 13];
  [PROCESS_TABLE + newPID * 16 + 4] = RUNNING;
  [PROCESS_TABLE + newPID * 16 + 9] = 0;
  breakpoint;
  ireturn;
endif;

[PROCESS_TABLE + newPID * 16 +4] = RUNNING;
multipop(BP);
return;

```

```nasm
PUSH BP
MOV BP,SP
MOV R1,6
MOV R3,3
MOV R0,BP
MOV R6,BP
SUB R0,R1
SUB R6,R3
BRKP
MOV R6,[R6]
MOV R0,[R0]
BRKP
MOV R5,"Write"
EQ R5,R0
JZ R5,62
MOV R0, 5
MOV R2, -2
PUSH R0
PUSH R2
PUSH R6
PUSH R0
PUSH R0
INT 7
POP R1
POP R1
POP R1
POP R1
POP R1
POP BP
RET

```

```
alias counter R0;
counter = 0;

while(counter <= 20) do
	if(counter % 2 == 1) then
	print counter;
	endif;
	counter = counter + 1;
endwhile;

```

```nasm
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

```
[PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+9]=5;	
alias userSP R0;
userSP = SP;

[PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+13]=SP;
SP = [PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+11]*512-1;

alias fileDisc R1;
fileDisc = [[PTBR+2*((userSP-4)/512)]*512+(userSP-4)%512];

if(fileDisc!=-2)
then
	alias retVal R5;
	retVal = ([PTBR+2*((userSP-1)/512)]*512)+((userSP-1)%512);
	[retVal] = -1;
else
	alias word R5;
        word  = userSP-3;
	multipush(R0,R1,R2,R3,R5);

    // calling module
	R1 = 3;						
	R2 = [SYSTEM_STATUS_TABLE+1];
	R3 = word;

	call DEVICE_MANAGER;	
	
	multipop(R0,R1,R2,R3,R5);
	
	alias retVal R6;
        retVal = ([PTBR+2*((userSP-1)/512)]*512)+((userSP-1)%512);
	[retVal]=0;
endif;

SP = userSP;
[PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+9]=0;
ireturn;


```

```nasm
[PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+13] = SP;		//UPTR
SP = [PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+11]*512-1;	//SP point to kernal stack
backup;

[PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+4] = READY;

//Tick Field

alias i R0;
i=0;
while(i<16) do
 if([PROCESS_TABLE+16*i+4]!=TERMINATED) then
  [PROCESS_TABLE+16*i] = [PROCESS_TABLE+16*i] + 1;
 endif;  
 i = i+1; 
endwhile;

breakpoint;
call SCHEDULER;

restore;
SP = [PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+13];	//User_SP
[PROCESS_TABLE+16*[SYSTEM_STATUS_TABLE+1]+9] = 0;  	//MODE_FLAG
ireturn;

```