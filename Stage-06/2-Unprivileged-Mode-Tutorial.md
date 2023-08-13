# Unprivileged Mode Tutorial

### Q1. How does the machine translate a logical address – say 1001- to the corresponding physical address? The machine does the following sequence of actions. Let us assume that PTBR contains value – say 3000, set previously.

```.
1. Calculate,
    logical_page_number = logical_address DIV page_size
                  = (1001 DIV 512) = 1. 
2. Calculate,
    offset = logical_address MOD page_size
           = (1001 MOD 512)
           = 489. 

3. Find, the page_table_address = contents of PTBR = 3000.

4. Find, 
    physical_page_number = value stored in address (page_table_address + 2 x logical_page_number)  
                 = value stored in address (3000 + 2 x 1)   
                 = value stored in address 3002.  

   Suppose that this value is 7. (The minimum value possible is 0 and the maximum value possible is 127 – why?). 


5. Calculate,
    physical_address = physical_page_number x page_size + offset
               = 7 x 512 + 489 = 4073. 
```

---

> The contents of an XEXE executable file containing this code (header included) would be as below:
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
MOV R1, 0
CMP R0, 10
JNZ 2070
ADD R1, R0
ADD R0, 1
JMP 2060
..
..
```

---

## Boot ROM image

<img src="D:/NITC/OS_LAB/OS-Lab-Master/~Attachments/bootrom.png" />

---

> Example: In our running example, suppose the instruction at logical address 2070 is INT 4:

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
MOV R1, 0
CMP R0, 10
JNZ 2070
ADD R1, R0
ADD R0, 1
JMP 2060
INT 4
```

The INT 4 instruction will push the logical address of the next instruction (2072) into the physical location corresponding to the top of the stack (SP contains logical address of the top of the stack). The INT instruction then sets IP register to address 5120 by refering to the vector table and changes the machine mode to privileged. Hence, the next instruction will be fetched from 5120. Later when the interrupt handler executes an IRET instruction, the IP value (2072) to be popped off the stack so that execution continues with the instruction at logical address 2072. (Note that the above description had assumed that the INT handler has not changed the value of PTBR. What would happen if PTBR is changed?)

```ad-note
Suppose you are designing the loader program of an operating system to load and execute unknown applications, how will you figure out where must be code pages of the application loaded?

In general, there is no way unless there is a prior agreement with the application programmer. Hence, each operating system publishes an interface specification called Application Binary Interface (ABI) that fixes this and several other matters. In the eXpOS project, the ABI convention is that the application code must be loaded to logical pages 4,5,6 and 7. The details are given in the eXpOS ABI given here. Thus the code area of an eXpOS application will start at address 2048. The above example had followed this eXpOS ABI.
```
