## Console interrupt

- The IN instruction initiates a console input but will not suspend machine execution till some input is read. Machine execution proceeds to the next instruction in the program. When the user enters data, the data is transferred to port P0, and a console interrupt is raised by the console device.
- After the execution of each instruction in unprivileged mode, the machine checks whether a pending disk/console/timer interrupt. If so, the machine does the following actions:
    1. ****Push the IP value into the top of the stack.
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
