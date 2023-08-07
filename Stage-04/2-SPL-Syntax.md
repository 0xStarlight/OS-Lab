# Keywords
```bash
alias   define  encrypt breakpoint  inline
halt    goto    call    return  ireturn
backup  restore readi   read    print
loadi   load    store   do  while
endwhile    break   continue    if  then
else    endif   tsl*    start*  reset*
```

---

# Operators
```bash
(   )   ;   [   ]   /   *   +   -   %   :
>   <   >=  <=  !=  ==  =   &&  ||  !
```

---

# Registers
```bash
SPL allows the use of 25 registers (R0-R15, BP, SP, IP, PTBR, PTLR, EIP, EC, EPN, EMA) and 4 ports (P0-P3). P0 and P1 are used for standard input and standard output respectively.
```

| Register Type                                    | Register Name |
| ------------------------------------------------ | ------------- |
| Program Registers                                | R0 - R15      |
| Reserved Registers (For the use of SPL compiler) | R16 - R19     |
| Base Pointer                                     | BP            |
| Stack Pointer                                    | SP            |
| Instruction Pointer                              | IP            |
| Page Table Base Register                         | PTBR          |
| Page Table Length Register                       | PTLR          |
| Exception Instruction Pointer                    | EIP           |
| Exception Cause                                  | EC            |
| Exception Page Number                            | EPN           |
| Exception Memory Address                         | EMA           |
| Input Port                                       | P0            |
| Output Port                                      | P1            |
| Unused Ports                                     | P2, P3        |
| Core Flag                                        | CORE              |

---
# Logical Expressions

Logical expressions may be formed by combining arithmetic expressions using relational operators. The relational operators supported by SPL are >, <, >=, <=, !=, ==. Standard meanings apply to these operators. A relational operator will take in two arguments and return 1 if the relation is valid and 0 otherwise. 

The relational operators can also be applied to strings. <, >, <=, >= compares two strings lexicographically. != and == checks for equality in the case of strings. If one of the operands is a string, the other operand will also be considered as a string.

Examples:

```
 "adam" < "apple" // This returns 1 
 "hansel" == "gretel" // This returns 0 
 "3" == 3 // This returns 1, as 3 will be treated as "3" 
```

Logical expressions themselves may be combined using logical operators, && (logical and) , || (logical or) and ! (not).

---
# Addressing Expressions

Memory of the machine can be directly accessed in an SPL program. A word in the memory is accessed by specifying the addressing element, i.e. memory location within [ ]. This corresponds to the value stored in the given address. An arithmetic expression or an addressing expression can be used to specify the address.

```bash
[1024], [R3], [R5+[R7]+128], [INODE\_TABLE + R2] etc.
```

---
# Define Statement

The define statement is used to define a symbolic name for a value. Define statements should be used before any other statement in an SPL program. The keyword define is used to associate a literal to a symbolic name.

```nasm
define constant_name value;
define DISK\_BLOCK 437;
```

---
# Alias Statement

An **alias** statement is used to associate a register/port with a name. Alias statements can be used anywhere in the program.

**SYNTAX :** `alias alias_name register_name;`
```nasm
alias counter R0;
```

---

# Breakpoint Statement

The **breakpoint** statement is used to debug the program. The program when run in the [debug mode](https://exposnitc.github.io/expos-docs/support-tools/xsm-simulator/) pauses the execution at this instruction.

**SYNTAX :** `breakpoint;`

---

# Assignment Statement

The SPL assignment statement assigns the value of an expression/value stored in a memory address to a register/memory address. **=** is the assignment operator used in SPL. The operand on the right hand side of the operator is assigned to the left hand side.

**SYNTAX :** `Register / Alias / [Address] = Register / Port / Number / String / Expression / [Address];`

```nasm
R2 = P0;   
[PTBR + 3] = [1024] + 10;   
R1 = "hello world";             
```

---

# If Statement

**If** statement specifies the conditional execution of two branches according to the value of a logical expression. If the expression evaluates to 1, the **if** branch is executed, otherwise the **else** branch is executed. The **else** part is optional.

**SYNTAX :**
```nasm

if (logical expression) then
        statements;  
else
        statements;  
endif;
```

---

# While Statement

**While** statement iteratively executes a set of statements based on a condition. The condition is defined using a logical expression. The statements are iteratively executed as long as the condition is true.

**SYNTAX :**
```nasm

while (logical expression) do
    statements;
endwhile;

```

---

### Break Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#break-statement "Permanent link")

**Break** statement when used inside a while loop block, stops the execution of the loop in which it is used and passes the control of execution to the next statement after the loop. This statement cannot be used anywhere else other than while loop.

**SYNTAX :** `break;`

---
### Continue Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#continue-statement "Permanent link")

**Continue statement** when used inside a while loop block, skips the current iteration of the loop and passes the control to the next iteration after checking the loop condition.

**SYNTAX :** `continue;`

---
### ireturn Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#ireturn-statement "Permanent link")

**ireturn** statement or the Interrupt Return statement is used to pass control from a kernel mode interrupt service routine to the user mode program which invoked it.

**SYNTAX :** `ireturn ;`

The **ireturn** is generally used at the end of an interrupt code. This statement translates to [IRET machine instruction](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/).

---

### Read/Print Statements[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#readprint-statements "Permanent link")

The **read** and **print** statements are used as standard input and output statements. The **read** statement initiates the transfer of a string from the console to the standard input port P0 using the IN machine instruction. The machine proceeds to execute the next instruction without waiting for the completion of the string transfer.

Note

String read or printed must not exceed 10 characters.

The **print** statement outputs value of a register or an integer/string literal or value of a memory location.

**SYNTAX :** `read;`

`print** Register / Number / String / Expression / [Address];`

---

### Readi Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#readi-statement "Permanent link")

The **readi** statement reads a value from the standard input device and stores it in a register using the INI machine instruction (which can be used only in debug mode).

Note

String read must not exceed 10 characters.

**SYNTAX :** `readi Register;`

---

### Load/Store Statements[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#loadstore-statements "Permanent link")

Loading and storing between the disk and the memory of the XSM machine can be accomplished using **load** and **store** statements in SPL. The machine proceeds to execute the next instruction without waiting for the completion of the block transfer. **load** statement loads the block specified by _block_number_ from the disk to the the page specified by the _page_number_ in the memory. **store** statement stores the page specified by _page_number_ in the memory to the the block specified by the _block_number_ in the disk. The _page_number_ and _block_number_ can be specified using arithmetic expressions.

**SYNTAX :**

`load (page_number, block_number); store (page_number, block_number);`

---
### Loadi Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#loadi-statement "Permanent link")

Loading from the disk to the memory of the XSM machine can also be accomplished using **loadi** statement in SPL. But here, the machine will continue execution of the next instruction only after the block transfer is completed. **loadi** statement loads the block specified by _block_number_ from the disk to the the page specified by the _page_number_ in the memory. The _page_number_ and _block_number_ can be specified using arithmetic expressions.

**SYNTAX :** `loadi (page\_number, block\_number);`

---

### Multipush Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#multipush-statement "Permanent link")

Multipush statement is used to push a sequence of registers into the memory locations starting from the address pointed to by SP. The registers are pushed in the order in which they are specified in the statement.

**SYNTAX :** `multipush (Register1, Register2, ...);`

---
### Multipop Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#multipop-statement "Permanent link")

Multipop statement is used to pop a sequence of registers from the memory locations starting from the address pointed to by SP. The registers are popped in the reverse order in which they are specified in the statement.

**SYNTAX :** `multipop (Register1, Register2, ...);`

---

### Backup Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#backup-statement "Permanent link")

The **backup** statement is used to backup all the machine registers (except SP, IP, exception flag registers and ports) into the memory locations starting from the address pointed to by SP in the order : BP, PTBR, PTLR, R0 - R19. The value of SP gets incremented accordingly.

**SYNTAX :** `backup;`

This statement translates to the [BACKUP machine instruction](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/).

---
### Restore Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#restore-statement "Permanent link")

The **restore** statement is used to restore the backed up machine registers from the memory. The registers are restored from contiguous memory locations starting from the address pointed to by SP in the order : R19-R0, PTLR, PTBR, BP. The value of SP gets decremented accordingly.

**SYNTAX :** `restore;`

This statement translates to the [RESTORE machine instruction](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/).

---
### Encrypt Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#encrypt-statement "Permanent link")

The **encrypt** statement replaces the value in the register Ri with its encrypted value.

**SYNTAX :** `encrypt Ri;`

This statement translates to the [ENCRYPT machine instruction](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/).

---
### Goto Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#goto-statement "Permanent link")

The **goto** statement transfers control to the specified labelled statement.

**SYNTAX :** `goto label / INT_n / MOD_n / constants;` (See [SPL constants](https://exposnitc.github.io/expos-docs/support-tools/constants/))

`goto label1; goto INT\_7; goto MOD\_2; goto MEMORY\_MANAGER;`

Note

label should be defined within the module.

---
### Call Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#call-statement "Permanent link")

The **call** statement saves procedure linking information on the stack and branches to the procedure specified by the argument.

**SYNTAX :** `call label / INT_n / MOD`_n / constants;` (See [SPL constants](https://exposnitc.github.io/expos-docs/support-tools/constants/))

`call swap\_func; call INT\_7; call MOD\_2; call MEMORY\_MANAGER;`

Note

label should be defined within the module.

Call statement translates to the [CALL machine instruction](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/).

---
### Return Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#return-statement "Permanent link")

The **return** statement is used to transfer the control from a subroutine to the calling program in the kernel mode and the return address is popped from the stack.

**SYNTAX :** `return;`

This statement translates to the [RET machine instruction](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/).

---
### Halt Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#halt-statement "Permanent link")

The **halt** statement is used to halt the machine.

**SYNTAX :** `halt;`

This statement translates to [HALT machine instruction](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/).

---
### Inline Statement[¶](https://exposnitc.github.io/expos-docs/support-tools/spl/#inline-statement "Permanent link")

The **inline** statement is used to give XSM machine instructions directly within an SPL program.

**SYNTAX :** `inline "MACHINE INSTRUCTION"`;

`inline "JMP 11776";`













