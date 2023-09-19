
# Summary

In Stage16 it explains how to handle console interrupts in the XSM console. The IN instruction is used to read data from the console into the input port P0, and it can only be executed inside a system call/module. When data is entered from the keyboard, the XSM machine raises a console interrupt, which helps the OS infer that the execution of the IN instruction is complete.

The console interrupt handler is responsible for transferring the data from port P0 to the process that is waiting for the data. A process executing the IN instruction sets its state to WAIT_TERMINAL and invokes the scheduler, and it must resume execution only after the XSM machine sends an interrupt upon data arrival.

User programs can invoke the read system call using the library interface. For a terminal read, the file descriptor (-1 for terminal input) is passed as the first argument. The read system call invokes the Terminal Read function present in the Device Manager Module. The Terminal Read function performs the following: acquires the terminal, issues an IN instruction, sets its state as WAIT_TERMINAL, invokes the scheduler, and transfers data present in the input buffer field of the process table into the word address (passed as an argument) after a console interrupt wakes up this process.

The OS maintains a global data structure called the terminal status table that stores information about the current state of the terminal. A process can acquire the terminal by invoking the Acquire Terminal function of the resource manager module.

The implementation of read system call (interrupt 6 routine) involves setting the MODE FLAG in the process table of the current process to the system call number, saving and initializing register SP, retrieving file descriptor from user stack, storing -1 as return value in user stack if file descriptor is not -1, and retrieving word address sent as an argument from user stack if file descriptor is -1.

In addition, we will add Terminal Read function in Module 4, which calls Acquire Terminal function, initializes registers R1 and R2 with function number of Acquire Terminal and PID of current process respectively, and uses read statement for requesting to read from the terminal. After invoking the scheduler and returning from it, Terminal Read function converts logical address to physical address and stores the value present in input buffer field of process table to that physical address of word.

The console interrupt handler interrupts the current process and executes when a console interrupt occurs. It transfers data from port P0 to the process that is waiting for data and wakes up all processes in WAIT_TERMINAL state by setting their state to READY.

---
# Starting
## Console interrupt

- The IN instruction initiates a console input but will not suspend machine execution till some input is read. Machine execution proceeds to the next instruction in the program. When the user enters data, the data is transferred to port P0, and a console interrupt is raised by the console device.
- After the execution of each instruction in unprivileged mode, the machine checks whether a pending disk/console/timer interrupt. If so, the machine does the following actions:
    1. Push the IP value into the top of the stack.
    2. Set IP to value stored in the interrupt vector table entry for the timer interrupt handler. The vector table entry for timer interrupt is located at physical address 493 in page 0 (ROM) of XSM and the value 2048 is preset in this location. Hence, the IP register gets value 1. The machine then switches to to privileged mode and address translation is disabled. Hence, next instruction will be fetched from physical address 2048. (See Boot ROM and Boot block section in [XSM Machine Organization](https://exposnitc.github.io/expos-docs/arch-spec/machine-organization/) documentation)

## Implementation

- A process must use the [XSM instruction IN](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/) to **read data from the console into the input** [port P0](https://exposnitc.github.io/expos-docs/arch-spec/machine-organization/). IN is a privileged instruction and can be executed only inside a system call/module. Hence, to read data from the console, a user process invokes the [read system call](https://exposnitc.github.io/expos-docs/os-spec/systemcallinterface/). The read system call invokes the Terminal Read function present in [Device Manager module](https://exposnitc.github.io/expos-docs/modules/module-04/) (Module 4). The IN instruction will be executed within this Terminal Read function.
- The most important fact about the **IN instruction is that it will not wait for the data to arrive in P0**. Instead, the XSM machine continues advancing the instruction pointer and executing the next instruction. Hence there must be some hardware mechanism provided by XSM to detect arrival of data in P0.
- When does data arrive in P0? This happens when some string/number is entered from the key-board and ENTER is pressed. At this time, **the XSM machine will raise the console interrupt**. Thus the console interrupt is the hardware mechanism that helps the OS to infer that the execution of the IN instruction is complete.

> What happens after terminal read is invoked?
> 
> - A process executing the "IN" instruction (Terminal Read) sets its state to "WAIT_TERMINAL" and notifies the scheduler.
> - This process effectively goes to sleep, allowing other processes to execute on the CPU.
> - The process will stay in the "WAIT_TERMINAL" state until an interrupt is received from the XSM machine indicating that the data has arrived in P0.
> - When the interrupt arrives, the scheduler is informed that the waiting process can now continue execution, as the required data is available.
> - The process is then moved from the "WAIT_TERMINAL" state back to the "running" state, and its execution resumes from where it was paused during the "IN" instruction.

- When the console interrupt occurs, the machine interrupts the current process (note that some other process would be running) and executes the console interrupt handler. (The interrupt mechanism is similar to the timer interrupt. The current value of IP+2 is pushed into the stack and control transfers to the interrupt handler - see [XSM machine execution tutorial](https://exposnitc.github.io/expos-docs/tutorials/xsm-interrupts-tutorial/#disk-and-console-interrupts) for details).It is the responsiblity of the **console interrupt handler to transfer the data arrived in port P0 to the process which is waiting for the data**. This is done by copying the value present in port P0 into the **input buffer** field of the [process table](https://exposnitc.github.io/expos-docs/os-design/process-table/) entry of the process which has requested for the input. **Console interrupt handler also wakes up the process in WAIT_TERMINAL by setting its state to READY**. (Other processes in WAIT_TERMINAL state are also set to READY by the console interrupt handler.)
- Each process maintains an input buffer which stores the last data read by the process from the console. On the occurance of a terminal interrupt, the interrupt handler uses the PID field of the terminal status table to identify the correct process that had acquired the terminal for a read operation. The handler transfer the data from the input port to the input buffer of the process.
- User programs can invoke the read system call using the library interface. For a terminal read, the file descriptor (-1 for terminal input) is passed as the first argument. The second argument is a variable to store number/string from console. Refer to the read system call calling convention [here](https://exposnitc.github.io/expos-docs/os-spec/dynamicmemoryroutines/).ExpL library converts exposcall to [low level system call interface](https://exposnitc.github.io/expos-docs/os-design/sw-interface/) for read system call, to invoke interrupt 6.

<aside> 💡 User programs can invoke the read system call using the library interface. For a terminal read, the file descriptor (-1 for terminal input) is passed as the first argument. The second argument is a variable to store number/string from console.

</aside>

- The read system call (Interrupt 6) invokes the **Terminal Read** function present in the [Device manager Module](https://exposnitc.github.io/expos-docs/modules/module-04/). Reading from the terminal and storing the number/string (read from console) in the address provided is done by the Terminal Read function. Function number for the Terminal Read function, current PID and address where the word has to be stored are sent as arguments through registers R1, R2 and R3 respectively. After coming back from Terminal Read function, it is expected that the word address (passed as argument to read system call) contains the number/string entered in the terminal.

> **When the Acquire Terminal function assigns the terminal to a process, it enters the PID of the process into the PID field of the terminal status table**. The Terminal Read function must perform the following

1. Acquire the terminal
2. Issue an IN instruction (SPL read statement translates to XSM instruction IN)
3. Set its state as WAIT_TERMINAL
4. Invoke the scheduler
5. After console interrupt wakes up this process, transfer data present in the input buffer field of the process table into the word address (passed as an argument).
![[Pasted image 20230903021624.png]]

---

# Practice 1
```ad-question
Write an ExpL program which reads two numbers from console and finds the GCD using Euclidean's algorithm and print the GCD. Load this program as init program.
```

> File: boot_module.spl
```nasm
//MOD_0
loadi(40,53);
loadi(41,54);

//MOD_4
loadi(48,61);
loadi(49,62);

//lib
loadi(63,13);
loadi(64,14);

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

//int 6
loadi(14,27);
loadi(15,28);

//int 3(terminal interrupt handler)
loadi(8,21);
loadi(9,22);

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

//other process table entries,state=terminated
[PROCESS_TABLE + 2*16 + 4] = TERMINATED;
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

> File: os_startup.spl
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

> File: console.spl
```nasm
[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16)+13]=SP;
SP=[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 11]*512-1;

backup;

alias reqPID R5;
reqPID = [TERMINAL_STATUS_TABLE + 1];
[PROCESS_TABLE + reqPID * 16 + 8] = P0;

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
```nasm
halt;
```

> File: int_10.spl
```nasm
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
```nasm
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
```nasm
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

> File: mod_5.spl
```nasm
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

> File: sample_int6.spl
```nasm
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

> File: sample_int7.spl
```nasm
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

> File: sample_timer.spl
```nasm

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

> File: gcd.expl
```c
decl
 int ExtendedEuclid(int a,int b);
enddecl

int ExtendedEuclid(int a,int b)
{
 decl
  int r0,r1,s0,s1,t0,t1,qi,ri,si,ti;
 enddecl

 begin
  r0 = a;
  r1 = b;
  s0 = 1;
  s1 = 0;
  t0 = 0;
  t1 = 1;

  while(r1 != 0) do
   qi = r0/r1;
   ri = r0 - (qi*r1);
   si = s0 - (qi*s1);
   ti = t0 - (qi*t1);
   r0 = r1;
   r1 = ri;
   s0 = s1;
   s1 = si;
   t0 = t1;
   t1 = ti;
  endwhile;

  write(r0);
  return 0;
 end
}

int main()
{
 decl
  int a,b,c;
 enddecl

 begin
  read(a);
  read(b);
  c = ExtendedEuclid(a,b);
  return 0;
 end 
}
```

## Compile
```bash
$> ./compile.sh stage16
```

![[Pasted image 20230903145757.png]]

## Load the files
> File: load16.dat
```bash
load --library ../expl/library.lib
load --idle /home/kali/myexpos/expl/expl_progs/stage16/infinite.xsm
load --init /home/kali/myexpos/expl/expl_progs/stage16/gcd.xsm
load --int=10 /home/kali/myexpos/spl/spl_progs/stage16/int_10.xsm
load --int=7 /home/kali/myexpos/spl/spl_progs/stage16/sample_int7.xsm
load --int=6 /home/kali/myexpos/spl/spl_progs/stage16/sample_int6.xsm
load --int=console /home/kali/myexpos/spl/spl_progs/stage16/console.xsm
load --module 0 /home/kali/myexpos/spl/spl_progs/stage16/mod_0.xsm
load --module 5 /home/kali/myexpos/spl/spl_progs/stage16/mod_5.xsm
load --module 4 /home/kali/myexpos/spl/spl_progs/stage16/mod_4.xsm
load --module 7 /home/kali/myexpos/spl/spl_progs/stage16/boot_module.xsm
load --exhandler /home/kali/myexpos/spl/spl_progs/stage16/haltprog.xsm
load --int=timer /home/kali/myexpos/spl/spl_progs/stage16/sample_timer.xsm
load --os /home/kali/myexpos/spl/spl_progs/stage16/os_startup.xsm
exit
```

```bash
$> ./xfs-interface < ../test/load16.dat
```

![[Pasted image 20230903145931.png]]

## Execute xsm

```bash
$> ./xsm
4
16
4
```

![[Pasted image 20230903150045.png]]

---

```ad-question
title: Q1. Is it possible that, the running process interrupted by the console interrupt be the same process that had acquired the terminal for reading?
No, The process which has acquired the terminal will be in WAIT_TERMINAL state after issuing a terminal read until the console interrupt occurs. Hence, this process will not be scheduled until console interrupt changes it's state to READY.
```

---

# Assignment 1
```ad-question
Write an ExpL program to read N numbers in an array, sort using bubble sort and print the sorted array to the terminal. Load this program as init program and run the machine.
```

> File: assign_1.expl
```c
decl 
   int n,arr[50],i,j,dup; 
enddecl

int main()
{
  begin
  read(n);

  i=0;
  while(i<n) do
    read(arr[i]);
    i = i+1;
  endwhile;

  i=0;
  while(i<n) do
    j=i;
    while(j<n) do
      if(arr[i]>arr[j]) then
        dup = arr[i];
        arr[i] = arr[j];
        arr[j] = dup;
      endif;
      j = j + 1;
    endwhile;
    i = i+1;
  endwhile;

  i=0;
  while(i<n) do
    write(arr[i]);
    i = i+1;
  endwhile;

  return 0;
  end
}
```

---

# Assignment 2

```nasm
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

















