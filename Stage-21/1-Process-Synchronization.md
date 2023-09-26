# Summary

In this stage of eXpOS, we will add support for process synchronization using Wait and Signal system calls. These system calls will help us design a more advanced shell program. We will also implement Getpid and Getppid system calls.

When a process executes the Wait system call, its execution is suspended till the process whose PID is given as argument to Wait terminates or executes the Signal system call. The process that enters Wait sets its state to WAIT_PROCESS and invokes the scheduler. A process executes the Signal system call to wake up all the processes waiting for it. If a process terminates without invoking Signal, then Exit system call voluntarily wakes up all the processes waiting for it.

When several processes running concurrently share a resource, it is necessary to synchronize access to the shared resource to avoid data inconsistency. Wait and Signal form one pair of primitives that help to achieve synchronization.

In the next stage, a more advanced synchronization primitive that has a state variable associated with it - namely the semaphore - will be added to the OS.

The Shell is a user program that implements an interactive user interface for the OS. In the present stage, we will run the shell as the INIT program, so that the shell will interact with the user.

The system calls Wait, Signal, Getpid, and Getppid are all implemented in interrupt routine 11. Each system call has a different system call number.

Exit Process function is modified so that it wakes up all the processes waiting for the current process. Similarly, the children of the process are set as orphan processes by changing PPID field of child processes to -1.

---
# Starting

In this stage, we will add support for process synchronization using _Wait_ and _Signal_ system calls to eXpOS. With the help of these system calls, we will design a more advanced shell program. We will also implement _Getpid_ and _Getppid_ system calls.

When a process executes the _Wait_ system call, its execution is suspended till the process whose PID is given as argument to _Wait_ terminates or executes the _Signal_ system call. The process that enters _Wait_ sets its state to WAIT_PROCESS and invokes the scheduler.

A process executes the _Signal_ system call to wake up all the processes waiting for it. If a process terminates without invoking _Signal_, then _Exit_ system call voluntarily wakes up all the processes waiting for it.

When several processes running concurrently share a resource (shared memory or file) it is necessary to synchronize access to the shared resource to avoid data inconsistency. _Wait_ and _Signal_ form one pair of primitives that help to achieve synchronization. In general, synchronization primitives help two co-operating processes to ensure that one process stops execution at certain program point, and waits for the other to issue a signal, before continuing execution.

To understand how _Wait_ and _Signal_ help for process synchronization, assume that two processes (say A and B) executing concurrently share a resource. When process A issues the _Wait_ system call with the PID of process B, it intends to wait until process B signals or terminates. When process B is done with the resource, it can invoke the _Signal_ system call to wake up process A (and all other processes waiting for process B). Thus, _Signal_ and _Wait_ can ensure that process A is allowed to access the resource only after process B permits process A to do so.

In the above example suppose process B had finished using the shared resource and had executed _Signal_ system call before process A executed _Wait_ system call, then process A will wait for process B to issue another signal. Hence if process B does not issue another signal, then process A will resume execution only after process B terminates. The issue here is that, although the OS acts on the occurance of a signal immediately, it never records the occurance of the signal for the future.**In other words, Signals are memoryless.**

A more advanced synchronization primitive that has a state variable associated with it - namely the [semaphore](https://en.wikipedia.org/wiki/Semaphore_(programming)) - will be added to the OS in the next stage.

When a process issues the _Exit_ system call, all processes waiting for it must be awakened. We will modify the **Exit Process** function in the [process manager module](https://exposnitc.github.io/expos-docs/modules/module-01/) to wake up all processes waiting for the terminating process. However, there is one special case to handle here. The Exit Process function is invoked by the _Exec_ system call as well. In this case, the process waiting for the current process must not be woken up (why?). The implementation details will be explained below.

Finally, when a process Exits, all its child processes become [orphan processes](https://en.wikipedia.org/wiki/Orphan_process) and their PPID field is set to -1 in the module function **Exit Process**. Here too, if Exit Process in invoked from the _Exec_ system call, the children must not become orphans.

#### Shell Program[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-21/#shell-program "Permanent link")

The Shell is a user program that implements an interactive user interface for the OS. In the present stage, we will run the shell as the INIT program, so that the shell will interact with the user.

The shell asks you to enter a string (called a command). If the string entered is "Shutdown", the program executes the Shutdown system call to halt the OS. Otherwise, the shell program forks and create a child process. The parent process then waits for the child to exit using the Wait system call. The child process will try to execute the command (that is, execute the file with name command.) If no such file exists, Exec fails and the child prints "BAD COMMAND" and exits. Otherwise, the command file will be executed. In either case, upon completion of the child process, the parent process wakes up. The parent then goes on to ask the user for the next command.

#### Implementation of Interrupt routine 11[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-21/#implementation-of-interrupt-routine-11 "Permanent link")

The system calls _Wait_, _Signal_, _Getpid_ and _Getppid_ are all implemented in the interrupt routine 11. Each system call has a different system call number.

- At the beginning of interrupt routine 11, extract the system call number from the user stack and switch to the kernel stack.
- Implement system calls according to the system call number extracted from above step. Steps to implement each system call are explained below.
- Change back to the user stack and return to the user mode.

The system call numbers for Getpid, Getppid, Wait and Signal are 11, 12, 13 and 14 respectively. From ExpL program, these system calls are invoked using [exposcall function](https://exposnitc.github.io/expos-docs/os-spec/dynamicmemoryroutines/).

##### WAIT SYSTEM CALL[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-21/#wait-system-call "Permanent link")

_Wait_ system call takes PID of a process (for which the given process will wait) as an argument.

- Change the MODE FLAG in the [process table](https://exposnitc.github.io/expos-docs/os-design/process-table/)to the system call number.
- Extract the PID from the user stack. Check the valid conditions for argument. A process should not wait for itself or a TERMINATED process. The argument PID should be in valid range (what is the [valid range](https://exposnitc.github.io/expos-docs/os-design/process-table/)?). If any of the above conditons are not satisfying, return to the user mode with `-1` stored as return value indicating failure. At any point of return to user, remember to reset the MODE FLAG and change the stack to user stack.
- If all valid conditions are satisfied then proceed as follows. Change the state of the current process from RUNNING to the tuple [(WAIT_PROCESS, argument PID)](https://exposnitc.github.io/expos-docs/os-design/process-table/) in the process table. Note that the STATE field in the process table is a tuple (allocated 2 words).
- Invoke the scheduler to schedule other processes.
    
    > The following step is executed only when the scheduler runs this process again, which in turn happens only when the state of the process becomes READY again.
    
- Reset the MODE FLAG in the process table of the current process. Store 0 in the user stack as return value and return to the calling program.

##### SIGNAL SYSTEM CALL[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-21/#signal-system-call "Permanent link")

_Signal_ system call does not have any arguments.

- Set the MODE FLAG in the process table to the signal system call number.
- Loop through all process table entries, if there is a process with STATE as tuple (WAIT_PROCESS, current process PID) then change the STATE field to READY.
- Reset the MODE FLAG to 0 in the process table and store 0 as return value in the user stack.

##### GETPID AND GETPPID SYSTEM CALLS[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-21/#getpid-and-getppid-system-calls "Permanent link")

_Getpid_ and _Getppid_ system calls returns the PID of the current process and the PID of the parent process of the current process respectively to the user program. Implement both these system calls in interrupt routine 11.

```ad-note
The system calls implemented above are final and will not change later. See algorithms for Wait/Signal and Getpid/Getppid
```

#### Modifications to Exit Process Function (function number = 3, Process Manager Module)[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-21/#modifications-to-exit-process-function-function-number-3-process-manager-module "Permanent link")

Exit Process function is modified so that it wakes up all the processes waiting for the current process. Similarly, the children of the process are set as orphan processes by changing PPID field of child processes to -1. But when the Exit Process function is invoked from _Exec_ system call, the process is actually not terminating as the new program is being overlayed in the same address space and is executed with the same PID. when Exit Process is invoked from _Exec_ system call, it should not wake up the processes waiting for the current process and also should not set the children as orphan processes. Check the MODE FLAG in the process table of the current process to find out from which system call Exit Process function is invoked.

If MODE FLAG field in the [process table](https://exposnitc.github.io/expos-docs/os-design/process-table/) has system call number not equal to 9 (Exec) implement below steps.

- Loop through the process table of all processes and change the state to READY for the processes whose state is tuple (WAIT_PROCESS, current PID). Also if the PPID of a process is PID of current process, then invalidate PPID field to -1.

```ad-note
The function implemented above is final and will not change later.
```

#### Shutdown system call[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-21/#shutdown-system-call "Permanent link")

To ensure graceful termination of the system we will write _Shutdown_ system call with just a HALT instruction. _Shutdown_ system call is implemented in interrupt routine 15. Create an xsm file with just the HALT instruction and load this file as interrupt routine 15. From this stage onwards, we will use a new version of Shell as our init program. This Shell version will invoke _Shutdown_ system call to halt the system.

In later stages, when a file system is added to the OS, the file system data will be loaded to the memory and modified, while the OS is running. The _Shutdown_ system call will be re-written so that it commits the changes to the file system data to the disk before the machine halts.

#### Modifications to boot module[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-21/#modifications-to-boot-module "Permanent link")

Load interrupt routine 11 and interrupt routine 15 from disk to memory. See disk and memory organization [here](https://exposnitc.github.io/expos-docs/os-implementation/).

---
# Questions

```ad-question
title: Does the eXpOS guarantees that two processes will not wait for each other i.e. circular wait will not happen
No. The present eXpOS does not provide any functionality to avoid circular wait. It is the responsiblity of the user program to make sure that such conditions will not occur.
```

---
# Assignment 1
## Implementations
1. int_11.spl
2. int_15.spl

```ad-question
It is recommended to implement the shell program according to the description given earlier on your own. One implementation of shell program is given [here](https://exposnitc.github.io/expos-docs/test-programs/#test-program-1-shell-version-ii-without-multiuser) . Load this program as the INIT program. Test the shell version by giving different ExpL programs written in previous stages. Remember to load the xsm files of ExpL programs as executables into the disk before trying to execute them using shell.
```

> File: int_11.spl
```c
// The system calls Wait, Signal, Getpid and Getppid are all implemented in the interrupt routine 11. Each system call has a different system call number.


// At the beginning of interrupt routine 11, extract the system call number from the user stack and switch to the kernel stack.

alias userSP R0;
userSP = SP;

[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16)+13]=SP;
SP=[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE+1]*16) + 11]*512-1;

// Implement system calls according to the system call number extracted from above step. Steps to implement each system call are explained below.


// NOTE
// GETPID SYSCALL == 11
// GETPPID SYSCALL == 12
// WAIT SYSCALL == 13
// SIGNAL SYSCALL == 14
// instructionPointer+2, returnValue, Arguement3, Arguement2, Arguement1, syscallNo

alias syscallNo R1;
alias ptbr R2;


ptbr = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 14];
syscallNo = [[ptbr + 2 * ((userSP - 5)/512)] * 512 + ((userSP - 5)%512)];

// GETPID SYSCALL
if(syscallNo == 11) then
	// Change the MODE FLAG in the process tableto the system call number.
	[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1]* 16) + 9] = 11;
	
	// returns the PID of the current process
	[[ptbr+2*((userSP-1)/512)]*512 + (userSP-1)%512] = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 1];

	[PROCESS_TABLE + 16*[SYSTEM_STATUS_TABLE+1] + 9 ] = 0;
endif;

// GETPPID SYSCALL
if(syscallNo == 12) then
	// Change the MODE FLAG in the process tableto the system call number.
	[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1]* 16) + 9] = 12;

	// returns the PPID of the current process
	[[ptbr+2*((userSP-1)/512)]*512 + (userSP-1)%512] = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 2];

	[PROCESS_TABLE + 16*[SYSTEM_STATUS_TABLE+1] + 9 ] = 0;

endif;

// WAIT SYSCALL 
// Wait system call takes PID of a process (for which the given process will wait) as an argument.
if(syscallNo == 13) then
	// Change the MODE FLAG in the process tableto the system call number.
	[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1]* 16) + 9] = 13;
	
	// Extract the PID from the user stack. Check the valid conditions for argument. A process should not wait for itself or a TERMINATED process. The argument PID should be in valid range (what is the valid range?). If any of the above conditons are not satisfying, return to the user mode with -1 stored as return value indicating failure. At any point of return to user, remember to reset the MODE FLAG and change the stack to user stack.

	alias PID R3;
	PID = [[ptbr+2*((userSP-4)/512)]*512 + (userSP-4)%512];

	if((PID == [SYSTEM_STATUS_TABLE +1]) || PID < 0 || PID >=16 || ([PROCESS_TABLE + (PID * 16) + 4] == TERMINATED)) then

		// If any of the above conditons are not satisfying, return to the user mode with -1 stored as return value indicating failure.
		[[ptbr+2*((userSP-1)/512)]*512 + (userSP-1)%512] = -1;

		[PROCESS_TABLE + 16*[SYSTEM_STATUS_TABLE+1] + 9 ] = 0;
		SP = userSP;
		ireturn;

	endif;

	// If all valid conditions are satisfied then proceed as follows. Change the state of the current process from RUNNING to the tuple (WAIT_PROCESS, argument PID) in the process table. Note that the STATE field in the process table is a tuple (allocated 2 words).

	[PROCESS_TABLE + 16*[SYSTEM_STATUS_TABLE+1] + 4 ] = WAIT_PROCESS;
	[PROCESS_TABLE + 16*[SYSTEM_STATUS_TABLE+1] + 5 ] = PID;

	// Invoke the scheduler to schedule other processes.
	call SCHEDULER;

	// The following step is executed only when the scheduler runs this process again, which in turn happens only when the state of the process becomes READY again.

	// Reset the MODE FLAG in the process table of the current process. Store 0 in the user stack as return value and return to the calling program.

	[[ptbr+2*((userSP-1)/512)]*512 + (userSP-1)%512] = 0;
	[PROCESS_TABLE + 16*[SYSTEM_STATUS_TABLE+1] + 9 ] = 0;

endif;

// SIGNAL SYSCALL
// Signal system call does not have any arguments.
if(syscallNo == 14) then
	// Set the MODE FLAG in the process table to the signal system call number.
	[PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1]* 16) + 9] = 14;

	// Loop through all process table entries, if there is a process with STATE as tuple (WAIT_PROCESS, current process PID) then change the STATE field to READY.
	
	alias i R4;
	i = 0;

	while(i < 16) do
		if([PROCESS_TABLE + (i * 16) + 4] == WAIT_PROCESS) then
			[PROCESS_TABLE + (i * 16) + 4] = READY;
		endif;
		i = i + 1;
	endwhile;

	// Reset the MODE FLAG to 0 in the process table and store 0 as return value in the user stack.

	[[ptbr+2*((userSP-1)/512)]*512 + (userSP-1)%512] = 0;
	[PROCESS_TABLE + 16*[SYSTEM_STATUS_TABLE+1] + 9 ] = 0;
endif;

SP = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 13];
ireturn;
```

> File: int_15.spl
```c
halt;
```

> File: load21.dat
```.
load --library ../expl/library.lib
load --idle /home/kali/myexpos/expl/expl_progs/stage21/infinite.xsm
load --init /home/kali/myexpos/expl/expl_progs/stage21/shell.xsm
load --exec /home/kali/myexpos/expl/expl_progs/stage21/even.xsm
load --exec /home/kali/myexpos/expl/expl_progs/stage21/odd.xsm
load --int=6 /home/kali/myexpos/spl/spl_progs/stage21/int_6.xsm
load --int=7 /home/kali/myexpos/spl/spl_progs/stage21/int_7.xsm
load --int=8 /home/kali/myexpos/spl/spl_progs/stage21/int_8.xsm
load --int=9 /home/kali/myexpos/spl/spl_progs/stage21/int_9.xsm
load --int=10 /home/kali/myexpos/spl/spl_progs/stage21/int_10.xsm
load --int=15 /home/kali/myexpos/spl/spl_progs/stage21/int_15.xsm
load --int=11 /home/kali/myexpos/spl/spl_progs/stage21/int_11.xsm
load --int=console /home/kali/myexpos/spl/spl_progs/stage21/console.xsm
load --int=disk /home/kali/myexpos/spl/spl_progs/stage21/disk.xsm
load --module 0 /home/kali/myexpos/spl/spl_progs/stage21/mod_0.xsm
load --module 1 /home/kali/myexpos/spl/spl_progs/stage21/mod_1.xsm
load --module 2 /home/kali/myexpos/spl/spl_progs/stage21/mod_2.xsm
load --module 4 /home/kali/myexpos/spl/spl_progs/stage21/mod_4.xsm
load --module 5 /home/kali/myexpos/spl/spl_progs/stage21/mod_5.xsm
load --module 7 /home/kali/myexpos/spl/spl_progs/stage21/mod_7.xsm
load --exhandler /home/kali/myexpos/spl/spl_progs/stage21/haltprog.xsm
load --int=timer /home/kali/myexpos/spl/spl_progs/stage21/timer.xsm
load --os /home/kali/myexpos/spl/spl_progs/stage21/os_startup.xsm
exit
```

---
# Assignment 2

```ad-question
Write an ExpL program 'pid.expl' which invokes Getpid system call and prints the pid. Write another ExpL program which invokes Fork system call three times back to back. Then, the program shall use Exec system call to execute pid.xsm file. Run this program using the shell.
```

> File: pid.expl
```c
int main()
{
	decl
		int temp, pid;
	enddecl

	begin

		pid = exposcall("Getpid");
		temp = exposcall("Write",-2,pid);

		return 0;
	end
}
```

> File: fork.expl
```c
int main()
{
	decl
		int temp;
	enddecl

	begin

		temp = exposcall("Fork");
		temp = exposcall("Fork");
		temp = exposcall("Fork");
		temp = exposcall("Exec","pid.xsm");

		return 0;
	end
}
```






































