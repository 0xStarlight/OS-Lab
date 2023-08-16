# Stage 08

```ad-abstract
- Run the XSM machine with Timer enabled.
- Familiarise with timer interrupt handling.
```

```ad-info
Read and understand the [XSM tutorial on Interrupts and Exception handling](https://exposnitc.github.io/expos-docs/tutorials/xsm-interrupts-tutorial/) before proceeding further. (Read only the Timer Interrupt part).
```

---

1. Which physical memory location will contain the physical page number to which return address will be stored by the machine before transferring control to the timer interrupt handler?

A) The stack pointer has the physical page number to return after iret. So, 29696 + (5001/512) = 29714

2. Suppose further that the memory location 29714 contains value 35. What will be the physical memory address to which the XSM machine will copy the value of the next instruction to be executed?

A) Value = 35 * 512 + (5001 % 512 ) = 18313

3. What will be the value stored into the location 18313 by the machine?

A) 3002 This is the (logical) address of the next instruction to be executed after return from the interrupt handler. Note the each XSM instruction occupies two words in memory and hence the next instruction's address is at 3002 (and not 3001).

4. What value will the SP and IP registers contain after the execution of the INT instruction?

A) SP = 5001 and IP = 2048 because in physical address 2048 contains the OS bootstrap code must load the timer interrupt handler into memory starting from 2048.

5. What will be the physical address from which the machine will fetch the next instruction?

A) 2048 beacuse IP = 2048

---

If the XSM simulator is run with the the timer set to some value - say 20, then every time the machine completes execution of 20 instructions in user mode, the timer device will send a hardware signal that interrupts machine execution. The machine will push the IP value of the next user mode instruction to the stack and pass control to the the timer interrupt handler at physical address 2048.

eXpOS design given [here](https://exposnitc.github.io/expos-docs/os-implementation/) requires you to load a timer interrupt routine into two pages of memory starting at memory address 2048 (pages 4 and 5). The routine must be written by you and loaded into disk blocks 17 and 18 so that the OS startup code can load this code into memory pages 4 and 5.

In this stage, we will run the machine with timer on and write a simple timer interrupt handler.

#### Modifications to OS Startup Code[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-08/#modifications-to-os-startup-code "Permanent link")

OS Startup code used in the previous stage has to be modified to load the timer interrupt routine from disk blocks 17 and 18 to memory pages 4 and 5.

```
loadi(4, 17);
loadi(5, 18);
```

#### Timer Interrupt[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-08/#timer-interrupt "Permanent link")

We will write the timer interupt routine such that it just prints "TIMER" and returns to the user program.

```
print "TIMER";
ireturn;
```

1) Save this file in your UNIX machine as $HOME/myexpos/spl/spl_progs/sample_timer.spl

2) Compile this program using the SPL compiler.

3) Load the compiled XSM code as the timer interrupt into the XSM disk using XFS Interface.

```
cd $HOME/myexpos/xfs-interface
./xfs-interface
# load --int=timer $HOME/spl/spl_progs/sample_timer.xsm
# exit
```
4) Recompile and reload the OS Startup code.

5) Run the XSM machine with timer enabled.

```
cd $HOME/myexpos/xsm
./xsm --timer 2
```

































