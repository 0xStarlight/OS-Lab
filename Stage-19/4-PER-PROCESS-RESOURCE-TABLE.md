
# PER-PROCESS RESOURCE TABLE
The Per-Process Resource Table has 8 entries and each entry is of 2 words. **The last 16 words of the User Area Page are reserved for this**. For every instance of a file opened (or a semaphore acquired) by the process, it stores the index of the Open File Table (or Semaphore Table) entry for the file (or semaphore) is stored in this table. One word is used to store the resource identifier which indicates whether the resource opened by the process is a [FILE](https://exposnitc.github.io/expos-docs/support-tools/constants/) or a [SEMAPHORE](https://exposnitc.github.io/expos-docs/support-tools/constants/). Open system call sets the values of entries in this table for a file.

The per-process resource table entry has the following format.

|   |   |
|---|---|
|Resource Identifier (1 word)|Index of Open File Table/ Semaphore Table entry (1 word)|

File descriptor, returned by Open system call, is the index of the per-process resource table entry for that open instance of the file.

A free entry is denoted by -1 in the Resource Identifier field.

#### PER-PROCESS KERNEL STACK[¶](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-kernel-stack "Permanent link")

Control is tranferred from a user program to a kernel module on the occurence of one of the following events :

1. The user program executes a system call
2. When an interrupt/exception occurs.

In either case, the kernel allocates a separate stack **for each process** (called the kernel stack of the process) which is different from the stack used by the process while executing in the user mode (called the user stack). Kernel modules use the space in the kernel stack for storing local data and do not use the user stack. This is to avoid user "hacks" into the kernel using the application's stack.

In the case of a system call, the application will store the parameters for the system call in its user stack. Upon entering the kernel module (system call), the kernel will extract these parameters from the application's stack and then change the stack pointer to its own stack before further execution. Since the application invokes the kernel module voluntarily, it is the responsibility of the application to save the contents of its registers (except the stack pointer and base pointer registers in the case of the XSM machine) before invoking the system call.

In the case of an interrupt/exception, the user process does not have control over the transfer to the kernel module (interrupt/exception handler). Hence the execution context of the user process (that is, values of the registers) must be saved by the kernel module, before the kernel module uses the machine registers for other purposes, so that the machine state can be restored after completion of the interrupt/exception handler. The kernel stack is used to store the execution context of the user process. This context is restored before the return from the kernel module. (For the implementation of eXpOS on the XSM architecture, the [backup](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/#backup) and [restore](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/#restore) instructions facilitate this).

In addition to the above, if a kernel module invokes another kernel module while executing a system call/interrupt, the parameters to the called module and the return values from the module are passed through the same kernel stack.

Here is a detailed tutorial on [kernel stack management in system calls, interrupts and exceptions](https://exposnitc.github.io/expos-docs/os-implementation/). A separate tutorial is provided for [kernel stack managament during context switch](https://exposnitc.github.io/expos-docs/os-design/timer-stack-management/).