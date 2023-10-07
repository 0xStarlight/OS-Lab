# Summary

+ The OS will add support for semaphores, which allow concurrent processes to handle the critical section problem.
+ There are four actions related to semaphores: 
	1. acquiring
	2. releasing
	3. locking
	4. unlocking
+ A process must acquire a semaphore before locking or unlocking it. When a process forks, the semaphores currently acquired are shared between the child and the parent.
+ The Semaphore table is a global data structure used to store details of semaphores currently used by all processes. The per-process resource table keeps track of the resources currently used by each process.
+ Semget and Semrelease are implemented in interrupt routine 13, while SemLock and SemUnLock are implemented in interrupt routine 14.
+ Semget is used to acquire a new semaphore, while Semrelease is used to detach a semaphore from the process.
+ Acquire Semaphore function finds a free entry in the semaphore table and sets the PROCESS COUNT to 1 in that entry. Release Semaphore function unlocks the semaphore and wakes up all processes waiting for it.

---
# Starting

In this stage, we will add support for [semaphores](https://en.wikipedia.org/wiki/Semaphore_%28programming%29) to the OS. Semaphores are primitives that allow concurrent processes to handle the [critical section](https://en.wikipedia.org/wiki/Critical_section) problem. A typical instance of the critical section problem occurs when a set of processes share memory or files. Here it is likely to be necessary to ensure that the processes do not access the shared data (or file) simultaneously to ensure data consistency. eXpOS provides **binary semaphores** which can be used by user programs (ExpL programs) to synchronize the access to the shared resources so that data inconsistency will not occur.

There are four actions related to semaphores that a process can perform. Below are the actions along with the corresponding eXpOS system calls

1. Acquiring a semaphore - _Semget_ system call
2. Releasing a semaphore - _Semrelease_ system call
3. Locking a semaphore - _SemLock_ system call
4. Unlocking a semaphore - _SemUnLock_ system call

To use a semaphore, first a process has to acquire a semaphore. **When a process forks, the semaphores currently acquired by a process is shared between the child and the parent.** A process can lock and unlock a semaphore only after acquiring the semaphore. The process can lock the semaphore when it needs to enter into the critical section. After exiting from the critical section, the process unlocks the semaphore allowing other processes (with which the semaphore is shared) to enter the critical section. After the use of a semaphore is finished, a process can detach the semaphore by releasing the semaphore.

A process maintains record of the semaphores acquired by it in its [per-process resource table](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-resource-table) . eXpOS uses the data structure, [semaphore table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#semaphore-table) to manage semaphores. Semaphore table is a global data structure which is used to store details of semaphores currently used by all the processes. The Semaphore table has 32 ( [MAX_SEM_COUNT](https://exposnitc.github.io/expos-docs/support-tools/constants/) ) entries. This means that only 32 semaphores can be used by all the processes in the system at a time. Each entry in the semaphore table occupies four words of which the last two are currently unused. For each semaphore, the PROCESS COUNT field in it's semaphore table entry keeps track of the number of processes currently sharing the semaphore. If a process locks the semaphore, the LOCKING PID field is set to the PID of that process. LOCKING PID is set to -1 when the semaphore is not locked by any process. An invalid semaphore table entry is indicated by PROCESS COUNT equal to 0. The SPL constant [SEMAPHORE_TABLE](https://exposnitc.github.io/expos-docs/support-tools/constants/) gives the starting address of the semaphore table in the [memory](https://exposnitc.github.io/expos-docs/os-implementation/) . See [semaphore table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#semaphore-table) for more details.

The [per-process resource table](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-resource-table) of each process keeps track of the resources (semaphores and files) currently used by the process. The per-process resource table is stored in the last 16 words of the [user area page](https://exposnitc.github.io/expos-docs/os-design/process-table/#user-area) of a process. Per-process resource table can store details of at most eight resources at a time. Hence the total number of semaphores and files acquired by a process at a time is at most eight. Each per process resource table entry contains two words. The first field, called the **Resource Identifier** field, indicates whether the entry corresponds to a file or a semaphore. For representing the resource as a file, the SPL constant [FILE](https://exposnitc.github.io/expos-docs/support-tools/constants/) (0) is used and for semaphore, the SPL constant [SEMAPHORE](https://exposnitc.github.io/expos-docs/support-tools/constants/) (1) is used. The second field stores the index of the semaphore table entry if the resource is a semaphore. (If the resource is a file, an index to the open file table entry will be stored - we will see this in later stages.) See the description of [per-process resource table](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-resource-table) for details.

![](https://exposnitc.github.io/expos-docs/assets/img/roadmap/sem.png)

Control flow for _Semaphore_ system calls

#### Implementation of Interrupt routine 13[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-22/#implementation-of-interrupt-routine-13 "Permanent link")

The system calls Semget and Semrelease are implemented in the interrupt routine 13. Semget and Semrelease has system call numbers 17 and 18 respectively.

- Extract the system call number from the user stack and switch to the kernel stack.
- Implement system calls Semget and Semrelease according to the system call number extracted from above step. Steps to implement these system calls are explained below.
- Change back to the user stack and return to the user mode.

##### SEMGET SYSTEM CALL[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-22/#semget-system-call "Permanent link")

**_Semget_**system call is used to acquire a new semaphore. _Semget_ finds a free entry in the [per-process resource table](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-resource-table) . _Semget_ then creates a new entry in the semaphore table by invoking the **Acquire Semaphore** function of [resource manager module](https://exposnitc.github.io/expos-docs/modules/module-00/) . The index of the semaphore table entry returned by Acquire Semaphore function is stored in the free entry of per-process resource table of the process. Finally, _Semget_ system call returns the index of newly created entry in the per-process resource table as **semaphore descriptor** (SEMID).

Implement _Semget_ system call using the detailed algorithm provided [here](https://exposnitc.github.io/expos-docs/os-design/semaphore-algos/#semget-system-call) .

##### SEMRELEASE SYSTEM CALL[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-22/#semrelease-system-call "Permanent link")

**_Semrelease_**system call takes semaphore desciptor (SEMID) as argument from user program. _Semrelease_ system call is used to detach a semaphore from the process. _Semrelease_ releases the acquired semaphore and wakes up all the processes waiting for the semaphore by invoking the **Release Semaphore** function of [resource manager module](https://exposnitc.github.io/expos-docs/modules/module-00/) . _Semrelease_ also invalidates the per-process resource table entry corresponding to the SEMID given as an argument.

Implement _Semrelease_ system call using the detailed algorithm provided [here](https://exposnitc.github.io/expos-docs/os-design/semaphore-algos/#semrelease-system-call).

```ad-note
If any semaphore is not released by a process during execution using _Semrelease_ system call, then the semaphore is released at the time of termination of the process in _Exit_ system call.
```

#### Acquire Semaphore (function number = 6, [resource manager module](https://exposnitc.github.io/expos-docs/modules/module-00/))[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-22/#acquire-semaphore-function-number-6-resource-manager-module "Permanent link")

**Acquire Semaphore** function takes PID of the current process as argument. _Acquire Semaphore_ finds a free entry in the semaphore table and sets the PROCESS COUNT to 1 in that entry. Finally, _Acquire Semaphore_ returns the index of that free entry of semaphore table.

Implement _Acquire Semaphore_ function using the detailed algorithm provided in resource manager module link above.

#### Release Semaphore (function number = 7, [resource manager module](https://exposnitc.github.io/expos-docs/modules/module-00/))[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-22/#release-semaphore-function-number-7-resource-manager-module "Permanent link")

**Release Semaphore** function takes a semaphore index (SEMID) and PID of a process as arguments. If the semaphore to be released is locked by current process, then _Release Semaphore_ function unlocks the semaphore and wakes up all the processes waiting for this semaphore. _Release Semaphore_ function finally decrements the PROCESS COUNT of the semaphore in its corresponding semaphore table entry.

Implement _Release Semaphore_ function using the detailed algorithm provided in resource manager module link above.

#### Implementation of Interrupt routine 14[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-22/#implementation-of-interrupt-routine-14 "Permanent link")

The system calls _SemLock_ and _SemUnLock_ are implemented in the interrupt routine 14. _SemLock_ and _SemUnLock_ has system call numbers 19 and 20 respectively.

- Extract the system call number from the user stack and switch to the kernel stack.
- Implement system calls SemLock and SemUnLock according to the system call number extracted from above step. Steps to implement these system calls are explained below.
- Change back to the user stack and return to the user mode.

##### SEMLOCK SYSTEM CALL[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-22/#semlock-system-call "Permanent link")

**SemLock**system call takes a semaphore desciptor (SEMID) as an argument from user program. A process locks the semaphore it is sharing using the _SemLock_ system call. If the requested semaphore is currently locked by some other process, the current process blocks its execution by changing its [STATE](https://exposnitc.github.io/expos-docs/os-design/process-table/#state) to the tuple (WAIT_SEMAPHORE, semaphore table index of requested semaphore) until the requested semaphore is unlocked. When the semaphore is unlocked, then STATE of the current process is made READY (by the process which has unlocked the semaphore).When the current process is scheduled and the semaphore is still unlocked the current process locks the semaphore by changing the LOCKING PID in the semaphore table entry to the PID of the current process. When the process is scheduled but finds that the semaphore is locked by some other process, current process again waits in the busy loop until the requested semaphore is unlocked.

Implement _SemLock_ system call using the detailed algorithm provided [here](https://exposnitc.github.io/expos-docs/os-design/semaphore-algos/#semlock-system-call).

##### SEMUNLOCK SYSTEM CALL[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-22/#semunlock-system-call "Permanent link")

**SemUnLock** system call takes a semaphore desciptor (SEMID) as argument. A process invokes _SemUnLock_ system call to unlock the semaphore. _SemUnLock_ invalidates the LOCKING PID field (store -1) in the semaphore table entry for the semaphore. All the processes waiting for the semaphore are made READY for execution.

Implement _SemUnLock_ system call using the detailed algorithm provided [here](https://exposnitc.github.io/expos-docs/os-design/semaphore-algos/#semunlock-system-call).

```ad-note
The implementation of **Semget** , **Semrelease** , **SemLock** , **SemUnLock** system calls and **Acquire Semaphore** , **Release Semaphore** module functions are final.
```

#### Modifications to _Fork_ system call[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-22/#modifications-to-fork-system-call "Permanent link")

In this stage, _Fork_ is modified to update the semaphore table for the semaphores acquired by the parent process. When a process forks, the semaphores acquired by the parent process are now shared between parent and child. To reflect this change, PROCESS COUNT field is incremented by one in the semaphore table entry for every semphore shared between parent and child. Refer algorithm for [fork system call](https://exposnitc.github.io/expos-docs/os-design/fork/).

- While copying the per-process resource table of parent to the child process do following -
- If the resource is semaphore (check the Resource Identifier field in the [per-process resource table](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-resource-table)), then using the sempahore table index, increment the PROCESS COUNT field in the [semaphore table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#semaphore-table)entry.

#### Modifications to Free User Area Page (function number = 2, [process manager module](https://exposnitc.github.io/expos-docs/modules/module-01/))[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-22/#modifications-to-free-user-area-page-function-number-2-process-manager-module "Permanent link")

The user area page of every process contains the [per-process resource table](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-resource-table) in the last 16 words. When a process terminates, all the semaphores the process has acquired (and haven't released explicitly) have to be released. This is done in the _Free User Area Page_ function. The**ReleaseSemaphore** function of resource manager module is invoked for every valid semaphore in the per-process resource table of the process.

- For each entry in the per-process resource table of the process do following -
- If the resource is valid and is semaphore (check the Resource Identifier field in the [per-process resource table](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-resource-table) ), then invoke**Release Semaphore** function of [resource manager module](https://exposnitc.github.io/expos-docs/modules/module-00/) .

```ad-note
**Fork** system call and **Free User Area Page** function will be further modified in later stages for the file resources
```

#### Modifications to boot module[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-22/#modifications-to-boot-module "Permanent link")

- Initialize the [semaphore table](https://exposnitc.github.io/expos-docs/os-design/mem-ds/#semaphore-table) by setting PROCESS COUNT to 0 and LOCKING PID to -1 for all entries.
- Load interrupt routine 13 and 14 from the disk to the memory. See [memory organisation](https://exposnitc.github.io/expos-docs/os-implementation/).

#### Making things work[¶](https://exposnitc.github.io/expos-docs/roadmap/stage-22/#making-things-work "Permanent link")

Compile and load the newly written/modified files to the disk using XFS-interface.

---
# Questions

```ad-question
title: When a process waiting for a sempahore is scheduled again after the sempahore is unlocked, is it possible that the process finds the sempahore still locked?
Yes, it is possible. As some other process waiting for the semaphore could be scheduled before the current process and could have locked the semaphore. In this case the present process finds the semaphore locked again and has to wait in a busy loop until the required sempahore is unlocked.
```

```ad-question
title: A process first locks a semaphore using SemLock system call and then forks to create a child. As the semaphore is now shared between child and parent, what will be locking status for the semaphore?
The sempahore will still be locked by the parent process. In Fork system call, the PROCESS COUNT in the semaphore table is incremented by one but LOCKING PID field is left untouched.
```

---
# Assignments

## Implementations
1. int13 (SEMGET and SEMRELEASE)
2. int14 (SemLock and SemUnlock)
3. mod0 (Acquire and Release Semaphore) (Resource Manager)
4. int8 (Fork Modification)
5. mod1 (FreeUserAreaPage Modification) (Process Manager)
6. mod7 (Boot Module)

> File: int_13.spl
```c
alias userSP R10;
alias currPID R9;
userSP = SP;
currPID = [SYSTEM_STATUS_TABLE+1];

//Work
//UPTR set to Stack Ptr
[PROCESS_TABLE + ( currPID * 16) + 13] = SP;

//UA Page No.
SP = [PROCESS_TABLE + (currPID * 16) + 11] * 512 - 1;

//Accessing physical fileDescriptor location

alias physicalPageNum R1;
alias offset R2;
alias sysCallNoPhysicalAddr R3;


//Try to find sysCallNoPhysicalAddr and the sysCallNo

physicalPageNum = [PTBR + 2 * ((userSP - 5)/ 512)];
offset = (userSP - 5) % 512;
sysCallNoPhysicalAddr = (physicalPageNum * 512) + offset;

alias sysCallNo R4;
sysCallNo=[sysCallNoPhysicalAddr];


//temp: R1,R2,R3, imp : R4,R9,R10
// print sysCallNo;

[PROCESS_TABLE + currPID * 16 + 9] = sysCallNo;


//SemGet SysCall
if (sysCallNo == 17)then
    //resourceTableIndex is the SEMID
    alias resourceTableIndex R5;
    alias resourceTableBase R6;
    alias semIndex R7;

    resourceTableIndex = 0;
    resourceTableBase = [PROCESS_TABLE + currPID * 16 + 11]*512 + RESOURCE_TABLE_OFFSET;

    //Finding Empty Resource Table Entry
    while(resourceTableIndex < 8 && [resourceTableBase + 2*resourceTableIndex ] != -1 )do
        resourceTableIndex = resourceTableIndex + 1;
    endwhile;

    // breakpoint;

    if (resourceTableIndex == 8)then
        physicalPageNum = [PTBR + 2 * ((userSP - 1)/ 512)];
        offset = (userSP - 1) % 512;
        [(physicalPageNum * 512) + offset] = -1;
    else
        //To indicate Semaphore
        [resourceTableBase + 2*resourceTableIndex ] = 1;

        //Calling Acquire Semaphore from RESOURCE_MANAGER
        multipush(R1,R2,R3,R4,R5,R6,R9,R10);
        R1 = 6;
        R2 = currPID;
        call RESOURCE_MANAGER;
        multipop(R1,R2,R3,R4,R5,R6,R9,R10);
        semIndex = R0;

        //Checking if Semaphore is available
        if ( semIndex == -1 )then
            physicalPageNum = [PTBR + 2 * ((userSP - 1)/ 512)];
            offset = (userSP - 1) % 512;
            [(physicalPageNum * 512) + offset] = -2;

        else
        //Set the second entry of PerProcessResourceTable and return index of resource table
            [resourceTableBase + 2*resourceTableIndex + 1 ] = semIndex;
            physicalPageNum = [PTBR + 2 * ((userSP - 1)/ 512)];
            offset = (userSP - 1) % 512;
            [(physicalPageNum * 512) + offset] = resourceTableIndex;
        endif;



    endif;

endif;


//SemRelease SysCall
if (sysCallNo == 18 )then
    alias resourceTableBase R5;
    alias semID R6;

    resourceTableBase = [PROCESS_TABLE + currPID * 16 + 11]*512 + RESOURCE_TABLE_OFFSET;

    physicalPageNum = [PTBR + 2 * ((userSP - 4)/ 512)];
    offset = (userSP - 4) % 512;
    semID = [(physicalPageNum * 512) + offset];

    //If invalid Semaphore ID given ( Out of range or not semaphore )
    if (semID < 0 || semID >7 || [resourceTableBase + 2*semID] != 1 )then
        physicalPageNum = [PTBR + 2 * ((userSP - 1)/ 512)];
        offset = (userSP - 1) % 512;
        [(physicalPageNum * 512) + offset] = -1;
    else
        //Calling Release Semaphore
        multipush(R1,R2,R3,R4,R5,R6,R9,R10);
        R1 = 7;
        R2 = [resourceTableBase + 2*semID + 1];
        R3 = currPID;
        call RESOURCE_MANAGER;
        multipop(R1,R2,R3,R4,R5,R6,R9,R10);

        //Invalidating Semaphore in PerProcessResourceTable
        [resourceTableBase + 2*semID] = -1;


    endif;

endif;


// Reset Mode Flag
[PROCESS_TABLE + currPID * 16 + 9] = 0;

//Reset User Stack
SP = [PROCESS_TABLE +  currPID * 16 + 13];
ireturn;
```

> File: int_14.spl
```c
alias userSP R10;
alias currPID R9;
userSP = SP;
currPID = [SYSTEM_STATUS_TABLE+1];

//Work
//UPTR set to Stack Ptr
[PROCESS_TABLE + ( currPID * 16) + 13] = SP;

//UA Page No.
SP = [PROCESS_TABLE + (currPID * 16) + 11] * 512 - 1;

//Accessing physical fileDescriptor location

alias physicalPageNum R1;
alias offset R2;
alias sysCallNoPhysicalAddr R3;


//Try to find sysCallNoPhysicalAddr and the sysCallNo

physicalPageNum = [PTBR + 2 * ((userSP - 5)/ 512)];
offset = (userSP - 5) % 512;
sysCallNoPhysicalAddr = (physicalPageNum * 512) + offset;

alias sysCallNo R4;
sysCallNo=[sysCallNoPhysicalAddr];


//temp: R1,R2,R3, imp : R4,R9,R10
// print sysCallNo;

[PROCESS_TABLE + currPID * 16 + 9] = sysCallNo;

//SemLock Syscall
if ( sysCallNo == 19 )then
    alias semID R5;
    alias resourceTableBase R6;
    alias semaphoreIndex R7;

    resourceTableBase = [PROCESS_TABLE + currPID * 16 + 11]*512 + RESOURCE_TABLE_OFFSET;

    physicalPageNum = [PTBR + 2 * ((userSP - 4)/ 512)];
    offset = (userSP - 4) % 512;
    semID = [(physicalPageNum * 512) + offset];

    //Setting return to -1 if invalid SEMID
    if (  semID < 0 || semID > 7 ||  [resourceTableBase + 2*semID] != 1)then
        physicalPageNum = [PTBR + 2 * ((userSP - 1)/ 512)];
        offset = (userSP - 1) % 512;
        [(physicalPageNum * 512) + offset] = -1;
    else

        semaphoreIndex = [resourceTableBase + 2*semID + 1];

        //Semaphore is not Free
        while ( [SEMAPHORE_TABLE + 4*semaphoreIndex] != -1 )do
            [PROCESS_TABLE + currPID * 16 + 4] = WAIT_SEMAPHORE;
            [PROCESS_TABLE + currPID * 16 + 5] = semaphoreIndex;

            multipush(R1,R2,R3,R4,R5,R6,R7,R9,R10);
            call SCHEDULER;
            multipop(R1,R2,R3,R4,R5,R6,R7,R9,R10);

        endwhile;

        //Semaphore is free
        [SEMAPHORE_TABLE + 4*semaphoreIndex] = currPID;

    endif;
endif;

//SemUnlock Syscall
if ( sysCallNo == 20 )then
    alias semID R5;
    alias resourceTableBase R6;
    alias semaphoreIndex R7;

    resourceTableBase = [PROCESS_TABLE + currPID * 16 + 11]*512 + RESOURCE_TABLE_OFFSET;

    physicalPageNum = [PTBR + 2 * ((userSP - 4)/ 512)];
    offset = (userSP - 4) % 512;
    semID = [(physicalPageNum * 512) + offset];

    //Setting return to -1 if invalid SEMID
    if (  semID < 0 || semID > 7 ||  [resourceTableBase + 2*semID] != 1)then
        physicalPageNum = [PTBR + 2 * ((userSP - 1)/ 512)];
        offset = (userSP - 1) % 512;
        [(physicalPageNum * 512) + offset] = -1;
    else

        semaphoreIndex = [resourceTableBase + 2*semID + 1];
        if ( [SEMAPHORE_TABLE + 4*semaphoreIndex] != -1 )then
            if ( [SEMAPHORE_TABLE + 4*semaphoreIndex] != currPID )then
                physicalPageNum = [PTBR + 2 * ((userSP - 1)/ 512)];
                offset = (userSP - 1) % 512;
                [(physicalPageNum * 512) + offset] = -2;
            else
                //Unlocking semaphore
                [SEMAPHORE_TABLE + 4*semaphoreIndex] = -1;

                //Changing all process to READY which are waiting for the semaphore
                alias i R8;
                i=0;
                while(i<16)do
                    if ( [PROCESS_TABLE + i * 16 + 4] == WAIT_SEMAPHORE && [PROCESS_TABLE + i * 16 + 5] == semaphoreIndex )then
                        [PROCESS_TABLE + i * 16 + 4] = READY;
                    endif;
                    i = i+1;
                endwhile;
                // breakpoint;
            endif;
        endif;

    endif;

endif;


// Reset Mode Flag
[PROCESS_TABLE + currPID * 16 + 9] = 0;

//Reset User Stack
SP = [PROCESS_TABLE +  currPID * 16 + 13];
ireturn;
```

> NOTE: rest code will be in the files

> File: load22.dat
```.
load --library ../expl/library.lib
load --idle /home/kali/myexpos/expl/expl_progs/stage22/infinite.xsm
load --init /home/kali/myexpos/expl/expl_progs/stage22/shell.xsm
load --exec /home/kali/myexpos/expl/expl_progs/stage22/rw.xsm
load --exec /home/kali/myexpos/expl/expl_progs/stage22/odd.xsm
load --exec /home/kali/myexpos/expl/expl_progs/stage22/even.xsm
load --exec /home/kali/myexpos/expl/expl_progs/stage22/pid.xsm
load --int=6 /home/kali/myexpos/spl/spl_progs/stage22/int_6.xsm
load --int=7 /home/kali/myexpos/spl/spl_progs/stage22/int_7.xsm
load --int=8 /home/kali/myexpos/spl/spl_progs/stage22/int_8.xsm
load --int=9 /home/kali/myexpos/spl/spl_progs/stage22/int_9.xsm
load --int=10 /home/kali/myexpos/spl/spl_progs/stage22/int_10.xsm
load --int=15 /home/kali/myexpos/spl/spl_progs/stage22/int_15.xsm
load --int=11 /home/kali/myexpos/spl/spl_progs/stage22/int_11.xsm
load --int=13 /home/kali/myexpos/spl/spl_progs/stage22/int_13.xsm
load --int=14 /home/kali/myexpos/spl/spl_progs/stage22/int_14.xsm
load --int=console /home/kali/myexpos/spl/spl_progs/stage22/console.xsm
load --int=disk /home/kali/myexpos/spl/spl_progs/stage22/disk.xsm
load --module 0 /home/kali/myexpos/spl/spl_progs/stage22/mod_0.xsm
load --module 1 /home/kali/myexpos/spl/spl_progs/stage22/mod_1.xsm
load --module 2 /home/kali/myexpos/spl/spl_progs/stage22/mod_2.xsm
load --module 4 /home/kali/myexpos/spl/spl_progs/stage22/mod_4.xsm
load --module 5 /home/kali/myexpos/spl/spl_progs/stage22/mod_5.xsm
load --module 7 /home/kali/myexpos/spl/spl_progs/stage22/mod_7.xsm
load --exhandler /home/kali/myexpos/spl/spl_progs/stage22/haltprog.xsm
load --int=timer /home/kali/myexpos/spl/spl_progs/stage22/timer.xsm
load --os /home/kali/myexpos/spl/spl_progs/stage22/os_startup.xsm
exit
```

```ad-question
title: Assignment 1
The reader-writer program given [here](https://exposnitc.github.io/expos-docs/test-programs/test-program-04/) has two writers and one reader. The parent process will create two child processes by invoking fork. The parent and two child processes share a buffer of one word. At a time only one process can read/write to this buffer. To acheive this, these three processes use a shared semaphore. A writer process can write to the buffer if it is empty and the reader process can only read from the buffer if it is full. Before the word in the buffer is overwritten the reader process must read it and print the word to the console. The parent process is the reader process and its two children are writers. One child process writes even numbers from 1 to 100 and other one writes odd numbers from 1 to 100 to the buffer. The parent process reads the numbers and prints them on to the console. Compile the program given in link above and execute the program using the shell. The program must print all numbers from 1 to 100, but not necessarily in sequential order.
```

> The only thing to change is the EXPL File

> File: rw.expl
```c
type
Share
{
  int isempty;
  int data;
}
endtype

decl
  Share head;
enddecl

int main()
{
decl
  int temp, x, pidone, pidtwo, semid, iter, counter;
enddecl

begin
  x = exposcall("Heapset");
  semid = exposcall("Semget");
  head = exposcall("Alloc", 2);
  head.isempty = 1;
  pidone = exposcall("Fork");

  if (pidone == 0) then
    iter = 1;
    while(iter <= 100) do
      temp = exposcall("SemLock", semid);
      temp = head.isempty;

      if(temp == 1) then
        head.data = iter;
        head.isempty = 0;
        iter = iter + 2;
      endif;

      temp = exposcall("SemUnLock", semid);

      counter = 0;
      while(counter < 50) do
        counter = counter + 1;
      endwhile;
    endwhile;
  else
    pidtwo =  exposcall("Fork");

    if(pidtwo == 0) then
      iter = 2;
      while(iter <= 100) do
        temp = exposcall("SemLock", semid);
        temp = head.isempty;

        if(temp == 1) then
          head.data = iter;
          head.isempty = 0;
          iter = iter + 2;
        endif;

        temp = exposcall("SemUnLock", semid);
      endwhile;
    else
      iter = 1;
      while(iter <= 100) do
        temp = exposcall("SemLock", semid);
        temp = head.isempty;

        if(temp == 0) then
          x = head.data;
          head.isempty = 1;
          temp = exposcall("Write", -2, x);
          iter = iter + 1;
        endif;

        temp = exposcall("SemUnLock", semid);
      endwhile;
    endif;
  endif;

  temp = exposcall("Semrelease", semid);
  return 0;
end
}
```

```ad-question
title: Assignment 2
The ExpL programs given [here](https://exposnitc.github.io/expos-docs/test-programs/test-program-13/) describes a parent.expl program and a child.expl program. The parent.xsm program will create 8 child processes by invoking Fork 3 times. Each of the child processes will print the process ID (PID) and then, invokes the Exec system call to execute the program "child.xsm". The child.xsm program stores numbers from PID*100 to PID*100 + 9 onto a linked list and prints them to the console (each child process will have a seperate heap as the Exec system call alocates a seperate heap for each process). Compile the programs given in the link above and execute the parent program (parent.xsm) using the shell. The program must print all numbers from PID*100 to PID*100+9, where PID = 2 to 9, but not necessarily in sequential order. **Also, calculate the maximum memory usage, number of disk access and number of context switches** (by modifying the OS Kernel code).
```

> File: parent.expl
```c
int main()
{
    decl
        int temp,pid;
    enddecl

    begin
        pid = exposcall("Fork");
        pid = exposcall("Fork");
        pid = exposcall("Fork");

        if(pid==-1) then
            temp=exposcall("Write", -2, "Fork Error");
        else
            temp=exposcall("Write", -2, pid);
        endif;

        temp = exposcall("Exec", "child.xsm");
        return 0;
    end
}
```

> File: child.expl
```c
type
    List
    {
        int data;
        List next;
    }
endtype

decl
    List head, p, q;
enddecl

int main()
{
decl
    int x, temp, pid, counter;
enddecl

begin
    x = initialize();
    pid = exposcall("Getpid");

    head=null;
    counter=0;
    while(counter<10) do
        p=alloc();
        p.data=pid*100 + counter;
        p.next=null;

        if(head==null) then
            head=p;
            q=p;
        else
            q.next=p;
            q=q.next;
        endif;

        counter=counter+1;
    endwhile;

    p=head;
    while(p!=null) do
        write(p.data);
        p=p.next;
    endwhile;

    return 0;
end
}
```

```ad-question
title: Assignment 3
The two ExpL programs given [here](https://exposnitc.github.io/expos-docs/test-programs/test-program-14/) perform merge sort in two different ways. The first one is done in a sequential manner and the second one, in a concurrent approach. Values from 1 to 64 are stored in decreasing order in a linked list and are sorted using a recursive merge sort function. In the concurrent approach, the process is forked and the merge sort function is called recursively for the two sub-lists from the two child processes. Compile the programs given in the link above and execute each of them using the shell. The program must print values from 1 to 64 in a sorted manner. Also, calculate **the maximum memory usage, number of contexts switches and the number of switches to KERNEL mode.**
```

> File: mss.expl
```c
type    
    List    
    {    
        int data;    
        List next;    
    }    
endtype    

decl    
    List head;    
    List mergeSort(List top);    
    List merge(List a, List b);    
enddecl    

List mergeSort(List top)    
{    
  decl    
    List slow, fast, a, b;    
  enddecl    

  begin    
    if((top!=null) AND (top.next!=null)) then    
      slow=top;    
      fast=top.next;    

      //Divide the list into two parts    
      while(fast!=null) do    
        fast=fast.next;    
        if(fast!=null) then    
          slow=slow.next;    
          fast=fast.next;    
        endif;    
      endwhile;    

      a=top;    
      b=slow.next;    
      slow.next=null;    

      //Recursively call merge sort    
      a=mergeSort(a);    
      b=mergeSort(b);    

      //Merge the two lists    
      top=merge(a, b);    
    endif;    

    return top;    
  end    
}    

List merge(List a, List b)    
{    
  decl    
    List result;    
  enddecl    

  begin    
    result=null;    

    if(a==null) then    
      result=b;    
    endif;    
    if(b==null) then    
      result=a;    
    endif;    

    if(a!=null AND b!=null) then    
      if(a.data<=b.data) then    
        result=a;    
        result.next=merge(a.next, b);    
      else    
        result=b;    
        result.next=merge(a, b.next);    
      endif;    
    endif;    

    return result;    
  end    
}    

int main()    
{    
  decl    
    int x, counter;    
    List p, q;    
  enddecl    

  begin    
    x = initialize();    

    //Storing values in descending order    
    head=null;    
    counter=0;    
    while(counter<64) do    
      p=alloc();    
      p.data=64-counter;    
      p.next=null;    

      if(head==null) then    
        head=p;    
        q=p;    
      else    
        q.next=p;    
        q=q.next;    
      endif;    

      counter=counter+1;    
    endwhile;    

    //Calling Merge Sort    
    head=mergeSort(head);    

    //Printing Values    
    p=head;    
    while(p!=null) do    
      write(p.data);    
      p=p.next;    
    endwhile;    
    return 1;    
  end    
}
```

> File: msc.expl
```c
type
    List
    {
        int data;
        List next;
    }
    Share
    {
        List link;
    }
endtype
decl
  int x, semid;
  List head;
  List mergeSort(List top);
  List merge(List a, List b);
enddecl
List mergeSort(List top)
{
  decl
    int len, pid;
    List slow, fast, a, b;
    Share s;
  enddecl

  begin
    len=1;
    if((top!=null) AND (top.next!=null)) then
      x=exposcall("SemLock", semid);
      slow=top;
      fast=top.next;

      //Divide the list into two parts
      while(fast!=null) do
        fast=fast.next;
        len=len+1;
        if(fast!=null) then
          slow=slow.next;
          fast=fast.next;
          len=len+1;
        endif;
      endwhile;

      a=top;
      b=slow.next;
      slow.next=null;
      x=exposcall("SemUnLock", semid);

      //If size less than 4, don't Fork
      if(len<=4) then
        a=mergeSort(a);
        b=mergeSort(b);
      else
        x=exposcall("SemLock", semid);
        s=alloc();
        x=exposcall("SemUnLock", semid);

        pid=exposcall("Fork");

        //If Fork is not possible, do sequential
        if(pid==-1) then
          a=mergeSort(a);
          b=mergeSort(b);
        else
          s.link=null;

          //Sort from two different process
          if(pid!=0) then
            a=mergeSort(a);
          else
            b=mergeSort(b);
            s.link=b;             //Store head in shared location
            x=exposcall("Exit");
          endif;
          x=exposcall("Wait", pid);
          b=s.link;
        endif;

        x=exposcall("SemLock", semid);
        free(s);
        x=exposcall("SemUnLock", semid);
      endif;

      //Merge the two lists
      x=exposcall("SemLock", semid);
      top=merge(a, b);
      x=exposcall("SemUnLock", semid);
    endif;

    return top;
  end
}

List merge(List a, List b)
{
  decl
    List result;
  enddecl

  begin
    result=null;

    if(a==null) then
      result=b;
    endif;
    if(b==null) then
      result=a;
    endif;

    if(a!=null AND b!=null) then
      if(a.data<=b.data) then
        result=a;
        result.next=merge(a.next, b);
      else
        result=b;
        result.next=merge(a, b.next);
      endif;
    endif;

    return result;
  end
}

int main()
{
  decl
    int x, counter;
    List p, q;
  enddecl

  begin
    x = initialize();
    semid = exposcall("Semget");

    //Storing values in descending order
    head=null;
    counter=0;
    while(counter<64) do
      p=alloc();
      p.data=64-counter;
      p.next=null;

      if(head==null) then
        head=p;
        q=p;
      else
        q.next=p;
        q=q.next;
      endif;

      counter=counter+1;
    endwhile;
    //Calling Merge Sort
    head=mergeSort(head);
    //Printing Values
    p=head;
    while(p!=null) do
      write(p.data);
      p=p.next;
    endwhile;
    x = exposcall("Semrelease");
    return 1;
  end
}
```


















