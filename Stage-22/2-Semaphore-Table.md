# Semaphore Table[¶](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#semaphore-table "Permanent link")

The Semaphore Table contains details about the semaphores used by all the processes. It consists of MAX_SEM_COUNT (= 32 in the present eXpOS version) entries. Thus, there can be at most MAX_SEM_COUNT = 32 semaphores used in the system at any time. For every semaphore entry in the per-process resource table, there is a corresponding entry in the semaphore table.

A process can acquire a semaphore using the Semget system call, which returns a unique semid for the semaphore. Semid is basically the index of the semaphore table entry for the semaphore.

The Process count field specifies the number of processes which are sharing the semaphore. Semget sets the Process Count to 1. When a process forks, all semaphores acquired by the parent is shared with the child and hence, Process Count is incremented. Semrelease system call decrements the Process Count. Process Count is also decremented when a process exits.

SemLock system call sets the Locking PID field to the PID of the process invoking it. When the semaphore is not locked, Locking PID is -1.

Each semaphore table entry is of size 4 words of which the last two are unused.

| 0           | 1             | 2       |
| ----------- | ------------- | ------ |
| LOCKING PID | PROCESS COUNT | Unused |

- **LOCKING PID** (1 word)- specifies the PID of the process which has locked the semaphore. The entry is set to -1 if the semaphore is not locked by any processes.
- **PROCESS COUNT** (1 word) - specifies the number of processes which are sharing the semaphore. A value of 0 indicates that the semaphore has not been acquired by any process.
- **Unused** (2 words)

An unused entry is indicated by 0 in the PROCESS COUNT field.

```ad-note
The Semaphore Table is present in page 56 of the memory (see [Memory Organisation](https://exposnitc.github.io/expos-docs/os-implementation/)), and the SPL constant [SEMAPHORE_TABLE](https://exposnitc.github.io/expos-docs/support-tools/constants/) points to the starting address of the table.
```
