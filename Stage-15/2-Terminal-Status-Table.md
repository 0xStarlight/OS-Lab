# Terminal Status Table

The Terminal Status Table keeps track of the Read/Write operations done on the terminal. Every time a Read or Write system call is invoked on the terminal, the PID of the process that invoked the system call is stored in Terminal Status Table. The table also contains information on the status of the terminal (whether free or busy). The size of the table is 4 words of which the last 2 are unused.

Every entry of the Terminal Status Table has the following format:

| 0  | 1  | 2  |
|---|---|---|
|STATUS|PID|Unused|

- **STATUS** (1 word) - specifies whether the terminal is free or is being used by a process to read or write. This field is initially set to 0. It is changed to 1 whenever terminal is busy. The Terminal Interrupt Handler sets back the status to 0 upon completion of Terminal Read. The Write system call sets back the status to 0 upon completion of Terminal Write.
- **PID** (1 word) - specifies the PID of the process which is currently using the terminal. This field is invalid when STATUS is 0.
- **Unused** (2 words)

```ad-note
The Terminal Table is present in page 57 of the memory (see [Memory Organisation](https://exposnitc.github.io/expos-docs/os-implementation/)), and the SPL constant [TERMINAL_STATUS_TABLE](https://exposnitc.github.io/expos-docs/support-tools/constants/) points to the starting address of the table.
```

