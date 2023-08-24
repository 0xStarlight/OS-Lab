### System Status Table[¶](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#system-status-table "Permanent link")

It keeps the information about the number of free pages in memory, the number of processes blocked because memory is unavailable, the number of processes in swapped state and, the pid of the process to be scheduled next. The size of the table is 8 words out of which 2 words are unused.

The System Status Table has the following format:

|   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|
|CURRENT_USER_ID|CURRENT_PID|MEM_FREE_COUNT|WAIT_MEM_COUNT|SWAPPED_COUNT|PAGING_STATUS|CURRENT_PID2|LOGOUT_STATUS|

- **CURRENT_USER_ID** (1 word) - specifies the userid of the currently logged in user.
- **CURRENT_PID** (1 word) - specifies the pid of the currently running process.
- **MEM_FREE_COUNT** (1 word) - specifies the number of free pages available in memory.
- **WAIT_MEM_COUNT** (1 word) - specifies the number of processes waiting (blocked) for memory.
- **SWAPPED_COUNT** (1 word) - specifies the number of processes which are swapped. A process is said to be swapped if any of its user stack pages or its kernel stack page is swapped out.
- **PAGING_STATUS** (1 word) - specifies whether swapping is initiated. Swap Out/Swap In are indicated by 0 and 1, respectively. Set to 0 if paging is not in progress.
- **CURRENT_PID2** (1 word) - specifies the pid of the currently running process on the secondary core. This field is used only when eXpOS is running on [NEXSM](https://exposnitc.github.io/expos-docs/arch-spec/nexsm/) (a two-core extension of XSM) machine.
- ***LOGOUT_STATUS** (1 word) - specifies whether logout is initiated on the primary core. Set to 0 if logout is not initiated. This field is used only when eXpOS is running on [NEXSM](https://exposnitc.github.io/expos-docs/arch-spec/nexsm/) (a two-core extension of XSM) machine.

Initially, when the table is set up by the OS startup code, the MEM_FREE_COUNT is initialized to the number of free pages available in the system, WAIT_MEM_COUNT to 0, SWAPPED_COUNT to 0 and PAGING_STATUS to 0.