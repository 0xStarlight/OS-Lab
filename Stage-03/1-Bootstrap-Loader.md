# Stage 03

## Understanding the Filesystem

```ad-abstract
- Use the XSM Instruction set to write a small _OS startup_ code.
- Load your _OS startup code_ into the _boot block_ of the disk and get this code executed on bootstrap.
```

```ad-attention
- Have a quick look at [XSM Machine Organisation](https://exposnitc.github.io/expos-docs/arch-spec/machine-organization/). (Do not spend more than 15 minutes).
- Have a quick look at [XSM Instruction set](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/). (Do not spend more than 15 minutes).
- It is absolutely necessary to read the [XSM privileged mode execution tutorial](https://exposnitc.github.io/expos-docs/tutorials/xsm-instruction-cycle/) before proceeding further.
```

---
In this stage, you will write a small assembly program to print "HELLO_WORLD" using XSM Instruction set and load it into block 0 of the disk using XFS-Interface as the **OS Startup Code**. As described above, this OS Startup Code is loaded from disk block 0 to memory page 1 by the ROM Code on machine startup and is then executed.

_The steps to do this are explained in detail below._

1) Create the assembly program to print "HELLO_WORLD". The assembly code to print "HELLO_WORLD" :

```nasm
MOV R0, "HELLO_WORLD"
MOV R16, R0
PORT P1, R16
OUT
HALT
```

```nasm
MOV R0, "HELLO_WORLD"
PORT P1, R0
OUT
HALT
```

Save this file as `$HOME/myexpos/spl/spl_progs/helloworld.xsm`.

2) Load the file as OS Startup code to `disk.xfs`using XFS-Interface.  
Invoke the XFS interface and use the following command to load the OS Startup Code

```asm
cd $HOME/myexpos/xfs-interface
./xfs-interface
# load --os $HOME/myexpos/spl/spl_progs/helloworld.xsm
# exit
```

Note that the `--os` option loads the file to Block 0 of the XFS disk.

3) Run the machine  

```asm
cd $HOME/myexpos/xsm
./xsm
```

The machine will halt after printing "HELLO_WORLD".

```asm
HELLO_WORLD
Machine is halting.
```

---

```ad-note
The XSM simulator given to you is an assembly language interpeter for XSM. Hence, it is possible to load and run assembly language programs on the simulator (unlike real systems where binary programs need to be supplied).
```

```ad-question
title: If the OS Startup Code is loaded to some other page other than Page 1, will XSM work fine?
No. This is because after the execution of the ROM Code, IP points to _512_ which is the 1st instruction of Page 1. So if the OS Startup Code is not loaded to Page 1, it results in an [exception](https://exposnitc.github.io/expos-docs/arch-spec/interrupts-exception-handling/) and leads to system crash.
```

---

# Assignment
> Write an assembly program to print numbers from 1 to 20 and run it as the OS Startup code.

```nasm
check:
mov R0, 20
mov R1, 1

print_numbers:
port P1, R1
OUT
inr R1
dcr R0
JZ R0,exit
JMP print_numbers

exit:
HALT
```

+ Load the file
```bash
~/myexpos/xfs-interface> ./xfs-interface
Unix-XFS Interace Version 2.0. 
Type "help" for getting a list of commands.
# load --os /home/kali/myexpos/spl/spl_progs/print_one_to_twenty.xsm
# exit
```

+ Run the machine
```bash
~/myexpos/xsm> ./xsm
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
Machine is halting.
```

## Using Spl
```python
alias counter R0;
counter = 0;
while(counter <= 20) do
  print counter;
  counter = counter + 1;
endwhile;
```

---

# Resources

1. https://exposnitc.github.io/expos-docs/arch-spec/machine-organization/
2. https://exposnitc.github.io/expos-docs/tutorials/xsm-instruction-cycle/
3. https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/


---







