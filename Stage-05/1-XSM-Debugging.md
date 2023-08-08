# Stage 05
## XSM Debugging

```ad-abstract
Getting familiarised with the XSM Debugger.
```

```ad-info
Read and understand the [Debugger Specification](https://exposnitc.github.io/expos-docs/support-tools/xsm-simulator/)
```

### Introduction[¶](https://exposnitc.github.io/expos-docs/support-tools/xsm-simulator/#introduction "Permanent link")

The **XSM (eXperimental String Machine)** Simulator is used to simulate the XSM hardware.

Within your XSM directory, use the following command to run the simulator

`./xsm [--timer #1] [--disk #2] [--console #3] [--debug]`

- **_Syntax_** : `--timer value`  
    **_Semantics_** : This flag sets the number of user mode instructions after which timer interrupt is triggered to the value specified. **--timer 0** disables the timer. The range of value is from 0 to 1024. **_Default Value_** : 20
    
- **_Syntax_** : `--disk value`  
    **_Semantics_** : This flag sets the number of user mode instructions after which the disk interrupt is triggered to the value specified. The range of value is from 20 to 1024. Note that count begins only after a LOAD or STORE machine instruction gets executed. **_Default Value_** : 20
    
- **_Syntax_** : `--console value`  
    **_Semantics_** : This flag sets the number of user mode instructions after which console interrupt is triggered to the value specified. The range of value is from 20 to 1024. Note that count begins only after a IN machine instruction gets executed. **_Default Value_** : 20
    
- **_Syntax_** : `--debug`  
    **_Semantics_** : This flag sets the machine into DEBUG mode when it encounters a BRKP machine instruction. Any BRKP instruction in the program will be ignored by the machine if this flag is not set. Further details are given in the section below. The machine instruction INI gets enabled only in DEBUG mode.
    

### Debugging[¶](https://exposnitc.github.io/expos-docs/support-tools/xsm-simulator/#debugging "Permanent link")

The `--debug` flag is used to debug the running machine. When this flag is set and the machine encounters a breakpoint instruction, the machine enters the DEBUG mode. In this mode a prompt is displayed which allows the user to enter commands to inspect the state of the machine.

The commands in DEBUG mode are :

- **_Syntax_** : `step` / `s`  
    **_Semantics_** : The execution proceeds by a single step.
    
- **_Syntax_** : `step <N>` / `s <N>`  
    **_Semantics_** : The execution proceeds by N number of steps.
    
- **_Syntax_ :** `continue` / `c`  
    **_Semantics_ :** The execution proceeds till the next breakpoint (BRKP) instruction.
    
- **_Syntax_ :** `continue <N>` / `c <N>`  
    **_Semantics_ :** The execution proceeds till the next N'th occurance of the breakpoint (BRKP) instruction.
    
- **_Syntax_** : `reg` / `r`  
    **_Semantics_** : Displays the contents of all the machine registers namely IP, SP, BP, PTBR, PTLR, EIP, EC, EPN, EMA, R0-R19 in that order.
    
- **_Syntax_** : `reg <register_name>` / `r <register_name>`  
    **_Semantics_** : Displays the contents of the specified register.  
    Sample usage: r R5, reg PTLR
    
- **_Syntax_** : `mem <page_num>` / `m <page_num>`  
    **_Semantics_** : Writes the contents of the memory page `<page_num>` to the file "mem" in the XSM folder.  
    Sample usage: mem 5, m 20
    
- **_Syntax_** : `mem <page_num_1> <page_num_2>` / `m <page_num_1> <page_num_2>`  
    **_Semantics_** : Writes the contents of the memory from pages `<page_num_1>` to `<page_num_2>` to the file "mem" in XSM folder.  
    Sample usage: mem 5 8, m 0 10
    
- **_Syntax_** : `pcb` / `p`  
    **_Semantics_** : Displays the Process Table entry of the current process.
    
- **_Syntax_** : `pcb <pid>` / `p <pid>`  
    **_Semantics_** : Displays the Process Table entry of the process with the given `<pid>`.
    
- **_Syntax_** : `pagetable` / `pt`  
    **_Semantics_** : Displays the Page Table at the location pointed by PTBR (Page Table Base Register).
    
- **_Syntax_** : `pagetable <pid>` / `pt <pid>`  
    **_Semantics_** : Displays the `<pid>th` Page Table.
    
- **_Syntax_** : `diskmaptable` / `dmt`  
    **_Semantics_** : Displays the Disk Map Table of the current process.
    
- **_Syntax_** : `diskmaptable <pid>` / `dmt <pid>`  
    **_Semantics_** : Displays the Disk Map Table of the process with the given `<pid>`.
    
- **_Syntax_** : `resourcetable` / `rt`  
    **_Semantics_** : Displays the Per-process Resource Table of the current process.
    
- **_Syntax_** : `resourcetable <pid>` / `rt <pid>`  
    **_Semantics_** : Displays the Per-process Resource Table of the process with the given `<pid>`.
    
- **_Syntax_** : `filetable` / `ft`  
    **_Semantics_** : Displays the Open File Table.
    
- **_Syntax_** : `semtable` / `st`  
    **_Semantics_** : Displays the Semaphore Table.
    
- **_Syntax_** : `memfreelist` / `mf`  
    **_Semantics_** : Displays the Memory Free List.
    
- **_Syntax_** : `filestatus` / `fst`  
    **_Semantics_** : Displays the File Status Table.
    
- **_Syntax_** : `diskstatus` / `dst`  
    **_Semantics_** : Displays the Disk Status Table.
    
- **_Syntax_** : `systemstatus` / `sst`  
    **_Semantics_** : Displays the System Status Table.
    
- **_Syntax_** : `terminalstatus` / `tst`  
    **_Semantics_** : Displays the Terminal Status Table.
    
- **_Syntax_** : `buffertable` / `bt`  
    **_Semantics_** : Displays the Buffer Table.
    
- **_Syntax_** : `inodetable` / `it`  
    **_Semantics_** : Displays the memory copy of the Inode Table.
    
- **_Syntax_** : `usertable` / `ut`  
    **_Semantics_** : Displays the memory copy of the User Table.
    
- **_Syntax_** : `diskfreelist` / `df`  
    **_Semantics_** : Displays the memory copy of the Disk Free List.
    
- **_Syntax_** : `rootfile` / `rf`  
    **_Semantics_** : Displays the memory copy of the Root File.
    
- **_Syntax_** : `location <address>` / `l <address>`  
    **_Semantics_** : Displays the content at memory address after address translation.
    
- **_Syntax_** : `val <address>` / `v <address>`  
    **_Semantics_** : Displays the content at memory address without address translation.
    
- **_Syntax_** : `watch <physical_address>` / `w <physical_address>`  
    **_Semantics_** : Sets a watch point to this address. Watch point is used to track changes of a particular memory location. Whenever a word which is watched is altered, program execution is stopped and the debug interface is invoked. Atmost 16 watch points can be set.
    
- **_Syntax_** : `watchclear` / `wc`  
    **_Semantics_** : Clears all the watch points.
    
- **_Syntax_** : `list` / `ls`  
    **_Semantics_** : List 10 instructions before and after the current instruction .
    
- **_Syntax_** : `page <address>` / `pg <address>`  
    **_Semantics_** : Displays the Page Number and Offset for the given `<address>`.
    
- **_Syntax_** : `exit` / `e`  
    **_Semantics_** : Exits the debug prompt and halts the machine.
    
- **_Syntax_** : `help` / `h`  
    **_Semantics_** : Displays commands in brief.
    
- **_Syntax_** : `accesslocktable` / `alt`  
    **_Semantics_** : Displays the Access Lock Table. (Only available on [NEXSM](https://exposnitc.github.io/expos-docs/arch-spec/nexsm/) simulator used in Stage 28 of [Roadmap](https://exposnitc.github.io/expos-docs/roadmap/))

---
# Breakpoint

In this stage you will write an SPL program with a **breakpoint** statement. The breakpoint statement translates to the [BRKP](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/) machine instruction and is used for debugging. If the XSM machine is run in the [Debug mode](https://exposnitc.github.io/expos-docs/support-tools/xsm-simulator/) , on encountering the BRKP instruction, the machine simulator will suspend the program execution and allow you to inspect the values of the registers, memory, os data structures etc. Execution resumes only after you instruct the simulator to proceed.

1) Write an SPL code to generate odd numbers from 1 to 10. Add a debug instruction in between :

```python
alias counter R0;
counter = 0;
while(counter <= 10) do
  if(counter%2 != 0) then
    breakpoint;
  endif;
  counter = counter + 1;
endwhile;
```

2) Compile the program using the SPL compiler.

3) Load the compiled xsm code as OS startup code into the XSM disk using the XFS interface.

4) Run the machine in debug mode.

```bash
cd $HOME/myexpos/xsm 
./xsm --debug
```

5) The Machine pauses after the execution of the first BRKP instruction. View the contents of registers using the command

```bash
reg
```

Enter the following command

```bash
mem 1
```

This will write the contents of memory page 1 to the file mem inside the xsm folder (if xsm is run from any other directory then the file mem will be created in that directory). Open this file and view the contents.

Use the following command step to the next instruction.

```bash
s
```

6) Press c to continue execution till the BRKP instruction is executed again. You can see that the content of R0 register changes during each iteration.

```bash
c
```










