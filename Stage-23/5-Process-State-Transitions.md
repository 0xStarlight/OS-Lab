## Process State Transition Diagram in eXpOS[¶](https://exposnitc.github.io/expos-docs/os-design/state-diag/#process-state-transition-diagram-in-expos "Permanent link")

The state transitions that a process in eXpOS can undergo are shown in the following diagram. The events that cause each transition are explained below the diagram.

![](https://exposnitc.github.io/expos-docs/assets/img/state_trans.png)

The events that cause the transitions:

**A** : CREATED -> RUNNING: The Scheduler has scheduled the process for execution for the first time.

**B** : RUNNING -> WAIT_TERMINAL : The process is either waiting for access to the terminal or for data to be inputted through terminal.

RUNNING -> WAIT_DISK : The process is either waiting for access to the disk or for the disk operation to finish.

**C** : RUNNING -> WAIT_SEMAPHORE : The semaphore which the process is trying to use, is found to be locked.

RUNNING -> WAIT_FILE : The file which the process is trying to read/write, is found to be locked.

RUNNING -> WAIT_BUFFER : The buffer which the process is trying to use, is found to be locked.

RUNNING -> WAIT_MEM : The process requires a free memory page but there are none in the memory.

**D** : RUNNING -> WAIT_PROCESS : The process is waiting for another process to either exit or to signal it.

**E** : RUNNING -> TERMINATED: The process has either completed execution or has invoked an Exit System Call.

**F** : WAIT_SEMAPHORE -> READY : The semaphore for which the process was waiting, is now unlocked.

WAIT_FILE -> READY : The file for which the process was waiting, is now unlocked.

WAIT_BUFFER -> READY : The buffer for which the process was waiting, is now unlocked.

WAIT_MEM -> READY : There are free pages in memory.

**G** : WAIT_TERMINAL -> READY : The input data has been read from terminal and terminal is free to be used by any process.

WAIT_DISK -> READY : The disk operation is complete.

**H** : WAIT_PROCESS -> READY : The process has either received a signal from the process it was waiting for or the latter has exited the system.

**I** : RUNNING -> READY : Context switch caused by the timer interrupt routine. 

**J** : READY -> RUNNING : The Scheduler has scheduled the process for execution.

**K** : ALLOCATED -> CREATED : When a PCB entry is allocated for a process that is being newly created by the Fork system call, its state is set to ALLOCATED. The state is changed to CREATED, once the Fork system call completes the creation of the process.

```ad-note
The process can go from any state other than running state to the swapped state. A process, once swapped out will not be be swapped back into memory unless it's state becomes READY.
```

```ad-note
A process may be unexpectredly TERMINATED due to various reasons like exceptions, logout, shutdown etc.
```

