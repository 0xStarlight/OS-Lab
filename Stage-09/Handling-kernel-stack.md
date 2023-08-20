# Stage 09

```ad-abstract
- Get introduced to setting up process table entry for a user program.
- Familiarise with the management of kernel stack in hardware interrupt handlers.
```

```ad-info
Read and understand the [Kernel Stack Management during Interrupts](https://exposnitc.github.io/expos-docs/os-design/stack-interrupt/)before proceeding further.
```

---
eXpOS requires that when the OS enters an interrupt handler that runs in kernel mode, the interrupt handler must switch to a different stack. This requirement is to prevent user level “hacks” into the kernel through the stack. In the previous stage, though you entered the timer interrupt service routine in the kernel mode, you did not change the stack. In this stage, this will be done.

To isolate the kernel from the user stack, the OS kernel must maintain two stacks for a program - **a user stack and a kernel stack**. In eXpOS, one page called the [user area page](https://exposnitc.github.io/expos-docs/os-design/process-table/#user-area) is allocated for each process. A part of the space in this page will be used for the kernel stack (some other process information also will be stored in this page).

Whenever there is a transfer of program control from the user mode to kernel during interrupts (or exceptions), the interrupt handler will change the stack to the kernel stack of the program (that is, the SP register must point to the top of the kernel stack of the program). Before the machine returns to user mode from the interrupt, the user stack must be restored (that is, the SP register must point to the top of the user stack of the program).

Once we have two stacks for a user program, we need to design some data structure in memory to store the SP values of the two stacks. This is because the SP register of the machine can store only one value.

eXpOS requires you to maintain a [Process Table](https://exposnitc.github.io/expos-docs/os-design/process-table/),where data such as value of the kernel stack pointer, user stack pointer etc. pertaining to each process is stored.

For now, we just have one user program in execution. Hence we will need just one process table entry to be created. Each process table entry contains several fields. But for now, we are only interested in storing only 1) user stack pointer and 2) the memory page allocated as user area for the program.

The process table starts at page number 56 (address 28672). The process table has space for 16 entries, each having 16 words. Each entry holds information pertaining to one user process. Since we have only one process, we will use the first entry (the first 16 words starting at address 28672). Among these, we will be updating only entries for user stack pointer (word 13) and user area page number (word 11) in this stage.

You will modify the previous stage code so that the user program is allocated a user area page. You will also create a process table entry for the program where you will make the necessary entries.

#### Modifications to the OS Startup Code[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-09/#modifications-to-the-os-startup-code "Permanent link")

1) Set the User Area page number in the [Process Table](https://exposnitc.github.io/expos-docs/os-design/process-table/) entry of the current process. Since the first available free page is 80, the User Area page is allocated at the physical page number 80. The [SPL constant](https://exposnitc.github.io/expos-docs/support-tools/constants/)PROCESS_TABLE points to the starting address(28672) of the Process Table.

```nasm
[PROCESS_TABLE + 11] = 80;
```

2) As we are using the first Process Table entry, the PID will be 0. eXpOS kernel is expected to store the PID in the PID field of the process table.

```nasm
[PROCESS_TABLE + 1] = 0;
```

3) The kernel maintains a data structure called [System Status Table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#system-status-table) where the PID of the currently executing user process is maintained. This makes it easy to keep track of the current PID whenever the machine enters any kernel mode routine. The System Status Table is stored starting from memory address 29560. The second field of this table must be set to the PID of the process which is going to be run in user mode.

Set the current PID field in the System Status Table. The [SPL constant](https://exposnitc.github.io/expos-docs/support-tools/constants/) SYSTEM_STATUS_TABLE points to the starting address of the System Status Table.

```nasm
[SYSTEM_STATUS_TABLE + 1] = 0;
```

4) The kernel stack pointer for the process need not be set now as **all interrupt handlers assume that the kernel stack is empty when the handler is entered from user mode**. Thus whenever an interrupt handler is entered from user mode, the kernelstack pointer will be initialized assuming that the stack is empty. (See [Kernel Stack Management during hardware interrupts and exceptions](https://exposnitc.github.io/expos-docs/os-design/stack-interrupt/)).The KPTR value will be used in later stages when kernel modules invoke each other.

#### Timer Interrupt[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-09/#timer-interrupt "Permanent link")

1) Save the current value of User SP into the corresponding Process Table entry. Obtain the process id of the currently executing process from [System Status Table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#system-status-table). This value can be used to get the [Process Table](https://exposnitc.github.io/expos-docs/os-design/process-table/) entry of the currently executing process.

```nasm
[PROCESS_TABLE + ( [SYSTEM_STATUS_TABLE + 1] * 16) + 13] = SP;
```

```ad-warning
title: Important
Registers R0-R15 are user registers. Since you have not saved the register values into the stack yet, you should be careful not to write any code that alters these registers till the user context is saved into the stack. Registers R16-R19 are marked for kernel use and hence the kernel can modify them. The SPL compiler will use these registers to translate your SPL code.
```

2) Set the SP to beginning of the kernel stack.User Area Page number is the 11th word of the Process Table. The initial value of SP must be set to this `address*512 - 1`.

```nasm
// Setting SP to UArea Page number * 512 - 1
SP = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 11] * 512 - 1;
```

3) Save the user context to the kernel stack using the [Backup](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/#backup) instruction.

```nasm
backup;
```

4) Print "timer".

```nasm
print "TIMER";
```

5) Restore the user context from the kernel stack and set SP to the user SP saved in Process Table, before returning to user mode.

```nasm
restore;
SP = [PROCESS_TABLE + ( [SYSTEM_STATUS_TABLE + 1] * 16) + 13];
```

6) Use ireturn statement to switch to user mode.

```nasm
ireturn;
```

## Final Code

> File: sample_timer_09.spl
```nasm
// Save the current value of User SP into the corresponding Process Table entry
[PROCESS_TABLE + ( [SYSTEM_STATUS_TABLE + 1] * 16) + 13] = SP;

// Setting SP to UArea Page number * 512 - 1
SP = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 11] * 512 - 1;

// backup
backup;

print "TIMER";

restore; 
SP = [PROCESS_TABLE + ( [SYSTEM_STATUS_TABLE + 1] * 16) + 13];
<<<<<<< Updated upstream
=======

>>>>>>> Stashed changes
ireturn;
```

>File: os_startup_09.spl
```nasm
// ./spl ./spl_progs/os_startup_09.spl
// load the library code
loadi(63, 13);
loadi(64, 14);

// load timer inturrupt
loadi(4, 17);
loadi(5, 18);

// load init program
loadi(65,7);
loadi(66,8);

// load int 10 program
loadi(22,35);
loadi(23,36);

// load exception handler
loadi(2,15);
loadi(3,16);

// setting page table base reg
PTBR = PAGE_TABLE_BASE;
PTLR = 10;

// setting user area page
[PROCESS_TABLE + 11] = 80;
[PROCESS_TABLE + 1] = 0;

// PID in system status table
[SYSTEM_STATUS_TABLE + 1] = 0;


//Library
[PTBR+0] = 63;
[PTBR+1] = "0100";
[PTBR+2] = 64;
[PTBR+3] = "0100";

//Heap
[PTBR+4] = 78;
[PTBR+5] = "0110";
[PTBR+6] = 79;
[PTBR+7] = "0110";

//Code
[PTBR+8] = 65;
[PTBR+9] = "0100";
[PTBR+10] = 66;
[PTBR+11] = "0100";
[PTBR+12] = -1;
[PTBR+13] = "0000";
[PTBR+14] = -1;
[PTBR+15] = "0000";

//Stack
[PTBR+16] = 76;
[PTBR+17] = "0110";
[PTBR+18] = 77;
[PTBR+19] = "0110";

SP = 8*512;
[76*512] = [65 * 512 + 1];

// return
ireturn;
```

> File: cube_stage_07.xsm
```nasm
0
2056
0
0
0
0
0
0
MOV R0, 1 
MOV R2, 5
GE R2, R0
JZ R2, 2076
MOV R1, R0
MUL R1, R0
MUL R1, R0
BRKP
ADD R0, 1
JMP 2058
INT 10
```

> File: haltprog.spl
```nasm
// ./spl ./spl_progs/halt.spl
halt;
```

>  File: Load_09.dat
```nasm
load --library ../expl/library.lib
load --init /home/kali/myexpos/expl/expl_progs/cube_stage_07.xsm
load --int=10 /home/kali/myexpos/spl/spl_progs/haltprog.xsm
load --exhandler /home/kali/myexpos/spl/spl_progs/haltprog.xsm
load --int=timer /home/kali/myexpos/spl/spl_progs/stage09/sample_timer_09.xsm
load --os /home/kali/myexpos/spl/spl_progs/stage09/os_startup_09.xsm
exit
```

## Execute and run the timer

```bash
$> ./xfs-interface < ../test/load08.dat
$> ./xsm --timer 2 
```

---

# Assignment

```ad-question
Print the process id of currently executing process in timer interrupt before returning to user mode. You can look up this value from the System Status Table.
```

### Change to be made
> File: sample_timer_09.spl
```nasm
// Save the current value of User SP into the corresponding Process Table entry
[PROCESS_TABLE + ( [SYSTEM_STATUS_TABLE + 1] * 16) + 13] = SP;

// Setting SP to UArea Page number * 512 - 1
SP = [PROCESS_TABLE + ([SYSTEM_STATUS_TABLE + 1] * 16) + 11] * 512 - 1;

// backup
backup;

print "TIMER";
print [PROCESS_TABLE + 1];

restore; 
SP = [PROCESS_TABLE + ( [SYSTEM_STATUS_TABLE + 1] * 16) + 13];
ireturn;
```













































