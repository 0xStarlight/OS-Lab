# Stage 06
## Running a user program

```ad-abstract
title: Learning Objectives
- Learn how to set up the address space for an application.
- Run an init program in user mode from the OS startup code.
```

```ad-info
title: Pre-requisite Reading
- It is absolutely necessary to read and understand the tutorial on [XSM Unprivileged Mode Execution](https://exposnitc.github.io/expos-docs/tutorials/xsm-unprivileged-tutorial/) before proceeding further.
- Have a quick look at the XSM specification documentation on [Virtual Machine Model](https://exposnitc.github.io/expos-docs/virtual-machine-spec/) and [Address Translation Mechanism](https://exposnitc.github.io/expos-docs/arch-spec/paging-hardware/).
```

Before proceeding further, try to solve the following question that test your understanding of [XSM Unprivileged Mode Execution](https://exposnitc.github.io/expos-docs/tutorials/xsm-unprivileged-tutorial/)

---
> In OS jargon, a user program in execution is called a **"process"**. Thus, in this stage, you are going to run the first user process. Typically the OS maintains some memory data structures associated with each process - like the process table, page table, user area etc. For now, we will not be concerned with most of these data structures except the page table. In later stages, you will be introduced to these data structures one by one.

```ad-note
At many places in this roadmap a process is identified with the underlying program in execution when there can be no scope for confusion.
```

```ad-note
In later stages, you will see that eXpOS actually schedules the idle process once before the INIT process is scheduled for the first time. This is done to ensure that the idle process is scheduled for execution at least once, so that the OS data structures associated with the idle process are not left un-initialized.
```
#### User Program[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-06/#user-program "Permanent link")

The following code illustrates the INIT program used in this stage. It computes squares of first 5 numbers. The value of Register R1 during each iteration will hold the result.

```nasm
//Program to calculate Squares of first 5 numbers

// R0 will hold value of n
// R1 will hold value of n^2

//Initialising R0(n) to 1
MOV R0, 1

_L1:

// Exit loop if n < 5
MOV R2, 5
GE R2, R0
JZ R2, _L2

// Computing n^2 in R1
MOV R1, R0
MUL R1, R0

//breakpoint instruction (to view contents of R1)
BRKP

// n = n + 1
ADD R0, 1

JMP _L1

_L2:

EXIT

// End of Program.
```

While executing in the user mode, the machine uses logical addressing scheme. The machine translates logical addresses to physical addresses using the [address translation mechanism](https://exposnitc.github.io/expos-docs/arch-spec/paging-hardware/).

In this stage, we will use a simple logical memory model where the first two logical pages are alloted for code (address 0 - 1023) and the third logical page is alloted for the stack (address 1024 - 1535). The actual logical memory model used in eXpOS is different and will be explained in the later stages.

The above code contains labels that are not recognised by the XSM machine.

Since the code section occupies first two pages according to our memory model, the code address begins from logical address 0 . Hence, we will translate the labels accordingly.

The code is given in bold and the corresponding addresses are added for reference. In the roadmap, the path of the file is assumed to be `$HOME/myexpos/expl/expl_progs/squares.xsm`

```nasm

    0   MOV R0, 1 
    2   MOV R2, 5
    4   GE R2, R0
    6   JZ R2, 18
    8   MOV R1, R0
    10  MUL R1, R0
    12  BRKP
    14  ADD R0, 1
    16  JMP 2
    18  INT 10
```

```nasm
MOV R0, 1 
MOV R2, 5
GE R2, R0
JZ R2, 18
MOV R1, R0
MUL R1, R0
BRKP
ADD R0, 1
JMP 2
INT 10
```

The methods for terminal input and output of user programs have not been studied till now. (Note that IN and OUT are privileged instructions and cannot be used in user mode programs). Hence you have to use the debug mode to view the contents of register R1 to watch the ouput. **Interrupt handlers** for input and output from user programs will be discussed in later stages.

Load this file to the XSM disk as the INIT program using XFS interface.

```nasm
load --init $HOME/myexpos/expl/expl_progs/squares.xsm
```

The xfs-interface will store `squares.xsm` program to disk blocks 7-8.

#### INT 10[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-06/#int-10 "Permanent link")

At the end of the program, a user program calls the exit system call to return control back to the operating system. This is acheived by an INT 10 instruction. INT 10 instruction will invoke the software interrupt handler 10. This interrupt handler is responsible for graceful termination of the user program.

Interrupt handlers and system calls will be covered in detail in later stages of the roadmap.

Since we have only one user process for now, we will write the interrupt 10 handler with only the "halt" statement.

1) Create a file haltprog.spl with a single halt statement.

```nasm
halt;
```

2) Compile the program

3) Load the compiled code as INT 10 from the xfs-interface

```
load --int=10 ../spl/spl_progs/haltprog.xsm
```

#### Exception handler[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-06/#exception-handler "Permanent link")

We also load the exception handler routine to memory. The machine may raise an exception if it encounters any unexpected events like illegal instruction, invalid address, page table entry for a logical page not set valid etc. Our default action is to halt machine execution in the case of an exception. In later stages you will learn to handle exceptions in a more elaborate way.

Load the `haltprog.xsm` used above as the exception handler using XFS-interface

```nasm
load --exhandler ../spl/progs/haltprog.xsm
```

#### OS Startup Code[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-06/#os-startup-code "Permanent link")

The OS startup code of any operating system, which is the first piece of OS code to be executed on bootstrap, is responsible for loading the rest of the OS into the memory, initialize OS data structures and set up the first user program for execution.

In this stage, we will write the OS startup code to load the init program and setup the OS data structures necessary to run the program as a process. Finally, the OS startup code will transfer control to the init program using the IRET instruction.

1) Load the INIT program from the disk to the memory. In the memory, init program is stored in pages 65-66. The blocks 7-8 from disk is to be loaded to the memory pages 65-66 by the OS startup Code. (See [Memory Organization and Disk Organization](https://exposnitc.github.io/expos-docs/os-implementation/)).

```nasm
loadi(65,7); 
loadi(66,8);
```

Load the INT10 module from the disk to the memory.

```nasm
loadi(22,35); 
loadi(23,36);
```

Load the exception handler routine from the disk to the memory.

```nasm
loadi(2, 15); 
loadi(3, 16);
```

Note the use of the `loadi` instruction for loading a disk block to a memory page. The loadi instruction will suspend the execution of the XSM machine till the disk to memory transfer is completed. XSM will execute the next instruction after the transfer is complete. (In later stages you will use the load instruction that can help to speed up execution).

2) [Page Table](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-page-table) for INIT must be set up for address translation scheme to work correctly. This is because INIT is a user process and all addresses generated are logical. Machine translates these logical addresses to physical addresses by looking up the page table for INIT.

The PTBR or Page Table Base Register stores the starting address of the page table of a process.We must set PTBR to the starting address of the page table of INIT. The [eXpOS memory organization](https://exposnitc.github.io/expos-docs/os-implementation/) stipulates that the page tables are stored from memory address 29696. Here since we are running the first user program, we will use the first few entries of this memory region for setting up the page table for the INIT process. The [SPL constant](https://exposnitc.github.io/expos-docs/support-tools/constants/) PAGE_TABLE_BASE holds the value 29696.

You need two pages for storing the INIT program code (loaded from disk blocks 7 and 8) and one additional page for stack (why?). Hence, PTLR is set to value 3.

```nasm
PTBR = PAGE_TABLE_BASE;
PTLR = 3;
```

3) In the page table of INIT, set page numbers 65 and 66 for code and 76 for stack. (Pages 67 - 75 are reserved. See [Memory Organisation](https://exposnitc.github.io/expos-docs/os-implementation/).) Thus, the first word of each entry must be set to the corresponding physical page number (65 ,66 and 76).Set the second word ([Auxiliary information](https://exposnitc.github.io/expos-docs/arch-spec/paging-hardware/#aux_info)) for pages 65 and 66 to "0100" and page 76 to "0110". This sets the code pages "read only" and stack "read/write". (why?)

```nasm
[PTBR+0] = 65;
[PTBR+1] = "0100";
[PTBR+2] = 66;
[PTBR+3] = "0100";
[PTBR+4] = 76;
[PTBR+5] = "0110";
```

```ad-note
Here we have introduced a simple memory model with 2 page code and 1 page stack memory.

The actual memory model which you will be using is different and will be explained in the later stages.
```

4) The OS Startup Code transfers control of execution to the user program using an IRET instruction. An IRET performs the following operations

i. The privilege mode is changed from KERNEL to USER mode.

ii. The instruction pointer is set to the value at the top of the user stack

iii. The value of SP is decremented by 1

The code of this program must execute from logical address 0. Hence IP or the instruction pointer needs to be set to 0 before the user program starts execution.

As IP cannot be set explicitly, push 0, which is the value of starting IP to the top of the stack, and IRET instruction will implicitly set the IP to this value.

Since the OS Startup Code runs in KERNEL mode, physical address must be used to access the top of the stack. Stack of INIT process is allocated at physical page number 76. Its corresponding physical address is 76 * 512. The stack pointer must be set to point to this address so that IRET fetches the correct address.

```nasm
[76*512] = 0;
SP = 2*512;
```

5)Use the _ireturn_ instruction to transfer control to user program. _ireturn_ translates to IRET machine instruction

```nasm
ireturn;
```
#### Making Things Work[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-06/#making-things-work "Permanent link")

1) Save the OS startup Code as `$HOME/myexpos/spl/spl_progs/os_startup.spl`. Compile this file using SPL compiler.

```nasm
cd $HOME/myexpos/spl
./spl $HOME/myexpos/spl/spl_progs/os_startup.spl
```

2) This will generate a file `$HOME/myexpos/spl/spl_progs/os_startup.xsm`. Load this file as the OS startup code to disk.xfs using the XFS Interface. Invoke the XFS interface and use the following command to load the OS Startup Code.

```nasm
# load --os $HOME/myexpos/spl/spl_progs/os_startup.xsm
# exit
```

3) Run the machine in debug mode. (We will disable the timer for now).

```nasm
cd $HOME/myexpos/xsm/
./xsm --debug --timer 0
```

4) View the contents of `R1` at each step.

```nasm
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
PTLR = 3;

// breakpoint
breakpoint;

// setting page table for init , phy pg no and auxiliary info
[PTBR+0] = 65;
[PTBR+1] = "0100";
[PTBR+2] = 66;
[PTBR+3] = "0100";
[PTBR+4] = 76;
[PTBR+5] = "0110";

// setting IP to 0 and pointing the SP to the top of the stack
[76*512] = 0;
SP = 2*512;

// return
ireturn;
```

---

# Assignment

```ad-question
Change virtual memory model such that code occupies logical pages 4 and 5 and the stack lies in logical page 8. You will have to modify the user program as well as the os startup code.
```

+ OS Startup Code
```nasm
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
PTLR = 9


// setting page table for init , phy pg no and auxiliary info
[PTBR+8] = 65;
[PTBR+9] = "0100";
[PTBR+10] = 66;
[PTBR+11] = "0100";
[PTBR+16] = 76;
[PTBR+17] = "0110";

// setting SP to the top of the stack
[76*512] = 2048;
SP = 8*512;

// return
ireturn;
```

+ Multiply code
```nasm
MOV R0, 1 
MOV R2, 5
GE R2, R0
JZ R2, 2066
MOV R1, R0
MUL R1, R0
BRKP
ADD R0, 1
JMP 2050
INT 10
```

```ad-note
page 1 --> ptbr + (1*2)

page 2 --> ptbr + (2*2)

page 3 --> ptbr + (3*2)

page 4 --> ptbr + (4*2)
```

```ad-tip

    2048   MOV R0, 1 
    2050   MOV R2, 5
    2052   GE R2, R0
    2054   JZ R2, 2066
    2056   MOV R1, R0
    2058  MUL R1, R0
    2060  BRKP
    2062  ADD R0, 1
    2064  JMP 2050
    2066  INT 10
```
























