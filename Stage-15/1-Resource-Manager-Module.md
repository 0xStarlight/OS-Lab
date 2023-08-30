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


































