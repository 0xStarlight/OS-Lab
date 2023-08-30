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

### Files Required
1. [ ] boot_module.spl
2. [ ] os_startup.spl
3. [ ] haltprog.spl
4. [ ] int10.spl
5. [ ] mod_5.spl
6. [ ] sample_int7.spl
7. [ ] sample_timer.spl
8. [ ] odd_nos.spl
9. [ ] even_nos.spl
10. [ ] prime_nos.spl

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
loadi(16,29);
loadi(17,30);

//exception
loadi(2,15);
loadi(3,16);

//timer
loadi(4,17);
loadi(5,18);

//FOR INIT
//pid
[PROCESS_TABLE+(16+1)]=1;

//userareapgnum
//Setting user area pg num
[PROCESS_TABLE + (16+11)]=80;
//UPTR FOR INIT
[PROCESS_TABLE + (16+13)]=8*512;

//KPTR FOR INIT
[PROCESS_TABLE+(16+12)]=0;

[PROCESS_TABLE+(16+14)]=(PAGE_TABLE_BASE+20);
[PROCESS_TABLE+(16+15)]=10;
[PROCESS_TABLE+(16+4)]=CREATED;

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
[PTBR+16]=76;
[PTBR+17]="0110";
[PTBR+18]=77;
[PTBR+19]="0110";

[76 * 512] = [65*512 + 1];

//FOR even.xsm
//pid
[PROCESS_TABLE+(32+1)]=2;

//userareapgnum
//Setting user area pg num
[PROCESS_TABLE + (32+11)]=85;
//UPTR
[PROCESS_TABLE + (32+13)]=8*512;

//KPTR
[PROCESS_TABLE+(32+12)]=0;

[PROCESS_TABLE+(32+14)]=(PAGE_TABLE_BASE+40);
[PROCESS_TABLE+(32+15)]=10;
[PROCESS_TABLE+(32+4)]=CREATED;

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
[PTBR+16]=88;
[PTBR+17]="0110";
[PTBR+18]=89;
[PTBR+19]="0110";

[88 * 512] = [84*512 + 1];

//other process table entries,state=terminated
[PROCESS_TABLE+(48+4)]=TERMINATED;
[PROCESS_TABLE+(64+4)]=TERMINATED;
[PROCESS_TABLE+(80+4)]=TERMINATED;
[PROCESS_TABLE+(96+4)]=TERMINATED;
[PROCESS_TABLE+(112+4)]=TERMINATED;
[PROCESS_TABLE+(128+4)]=TERMINATED;
[PROCESS_TABLE+(144+4)]=TERMINATED;
[PROCESS_TABLE+(160+4)]=TERMINATED;
[PROCESS_TABLE+(176+4)]=TERMINATED;
[PROCESS_TABLE+(192+4)]=TERMINATED;
[PROCESS_TABLE+(208+4)]=TERMINATED;
[PROCESS_TABLE+(224+4)]=TERMINATED;
[PROCESS_TABLE+(240+4)]=TERMINATED;

//set status=0 in terminal status table
[TERMINAL_STATUS_TABLE]=0;

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

> File: haltprog.spl
```c
halt;
```

> File: int10.spl
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

> File: sample_int7.spl
```c
// to indicate what we gonna execute
[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 9] = 5;

// storing user SP
alias userSP R0;
userSP = SP;

// Switching user stack to kernal stack
[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 13] = SP;
SP = [PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 11]*512 - 1;
backup;

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
	print word;
	// Setting return value 0 if we succeed
	alias physicalAddrRetVal R6;
	physicalAddrRetVal = ([PTBR + 2 * (userSP - 1)/ 512] * 512) + ((userSP - 1) % 512);
	[physicalAddrRetVal] = 0;
endif;

restore;
[PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 9] = 0;
SP = [PROCESS_TABLE + [SYSTEM_STATUS_TABLE + 1] * 16 + 13];
ireturn;
```

> File: sample_timer.spl
```c
//breakpoint;
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
//breakpoint;
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
$> ./compile.sh stage14
```

![[Pasted image 20230830175948.png]]
### 2. Loading File
> File: load_14.dat
```bash
load --library ../expl/library.lib
load --idle /home/kali/myexpos/expl/expl_progs/stage14/even_nos.xsm
load --init /home/kali/myexpos/expl/expl_progs/stage14/odd_nos.xsm
load --exec /home/kali/myexpos/expl/expl_progs/stage14/prime.xsm
load --int=10 /home/kali/myexpos/spl/spl_progs/stage14/int10.xsm
load --int=7 /home/kali/myexpos/spl/spl_progs/stage14/sample_int7.xsm
load --module 5 /home/kali/myexpos/spl/spl_progs/stage14/mod_5.xsm
load --module 7 /home/kali/myexpos/spl/spl_progs/stage14/boot_module.xsm
load --exhandler /home/kali/myexpos/spl/spl_progs/stage14/haltprog.xsm
load --int=timer /home/kali/myexpos/spl/spl_progs/stage14/sample_timer.xsm
load --os /home/kali/myexpos/spl/spl_progs/stage14/os_startup.xsm
exit
```

### 3. Executing
```bash
$> ./xfs-interface < ../test/load14.dat
$> ./xsm --timer 80
```

![[Pasted image 20230830175729.png]]


