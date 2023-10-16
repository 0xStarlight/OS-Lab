# File Read

## Summary
### Pre-Requisites
1. File Status Table ( Memory DS)
2. Buffer Table (Memory DS)
3. Open File Table (Memory DS)
4. Per-Process Resource Table (Memory DS)
### Objective
1. Open, Close and Read system calls
2. Read content with read system call
3. Modify Fork system call


## Starting

We'll learn how to open and close files using Open and Close system calls, as well as how to read file contents with the Read system call. Additionally, we'll modify the Fork system call and Free User Area Page function in the process manager module.

### INT 5 (Implementation)
1. Open system call = 2
2. Close system call = 3

![[Pasted image 20231016191332.png]]

## Open system call
![[Pasted image 20231016194316.png]]
- The open system call is used in a user program to provide the filename as an argument.
- To perform read/write operations on a file, the process needs to open the file first.
- The open system call creates a new open instance for the file and returns a file descriptor, which is the index of the new per-process resource table entry created for the open instance.
- All further operations on the open instance are performed using this file descriptor.
- It is possible for a process to open a file multiple times, and each time a different open instance and new descriptor are created.
- The global data structure called the Open file table keeps track of all the open file instances in the system.
- Whenever the Open system call is invoked with any file name, a new entry is created in this table.
- The File status table is another global data structure that maintains an entry for every file in the system, not just for opened files

- The open system call creates new entries in two tables: the per-process resource table and the open file table.
- The process keeps track of an open instance by storing the index of the open file table entry in the corresponding resource table entry.
- When a file is opened, the open instance count in the open file table is set to 1.
- The seek position is initialized to the starting of the file, which is 0.

- The file open count in the file status table entry is incremented by one each time a file is opened.
- The open system call invokes the Open function of the file manager module to handle the file status table and open file table.
- When a process executes a Fork system call, the open instances of files and semaphores created by that process are shared between the current process and its child.
- As a result of the Fork system call, the open instance count in the open file table entry corresponding to the open instance is incremented by one.

- There are two types of counts related to file opening: FILE OPEN COUNT and OPEN INSTANCE COUNT.
- FILE OPEN COUNT keeps track of how many times the Open system call has been used to open a file globally in the system.
- OPEN INSTANCE COUNT keeps track of the number of open instances of a file at a given point in time.
- Each time a Close system call is invoked on a file by any process, the FILE OPEN COUNT is decremented.
- Each open instance of a file can be shared between multiple processes using the Fork system call.
- The OPEN INSTANCE COUNT value of a particular open instance keeps track of the number of processes currently sharing that instance of the file.

```ad-note
For a process to read/write a file, it must first open the file. Only data and root files can be opened. The Open operation returns a file descriptor which identifies the open instance of the file. An application can open the same file several times and each time, a different descriptor will be returned by the Open operation.

The OS associates a seek position with every open instance of a file. The seek position indicates the current location of file access (read/write). The Open system call initilizes the seek position to 0 (beginning of the file). The seek position can be modified using the [Seek system call](https://exposnitc.github.io/expos-docs/os-design/seek/).

The [root file](https://exposnitc.github.io/expos-docs/os-design/disk-ds/#root-file) can be opened for Reading by specifying the filename as **_"root"_**. Note that the Root file is different from the other files - It has a reserved memory page copy. So this will be treated as a special case in all related system calls.
```

### Parameters
#### 1. Arguments
+ Filename (String)

#### 2. Return Value
![[Pasted image 20231016195459.png]]
### Algorithm
```c

    Set the MODE_FLAG in the process table entry to 2, 
    indicating that the process is in the open system call.

    //Switch to Kernel Stack - See Kernel Stack Management during System Calls. 
    Save the value of SP to the USER SP field in the Process Table entry of the process.
    Set the value of SP to the beginning of User Area Page.

    Find a free Per-Process Resource Table entry.
                Find the PID of the current process from the System Status Table.
                Find the User Area page number from the Process Table entry.
                The  Per-Process Resource Table is located at the  RESOURCE_TABLE_OFFSET from the base of the  User Area Page .              
        Find a free Resource Table entry.  
        If there is no free entry, return -3.  

    Call the open() function from the File Manager module to get the Open File table entry.

    If Open fails, return the error code.

    Set the Per-Process Resource Table entry
        Set the Resource Identifier field to FILE . 
        Set the Open File Table index field to the free Open File Table entry found.         

    Set the MODE_FLAG in the process table entry to 0.

    Restore SP to User SP.

    Return the index of the Per-Process Resource Table entry.   /* success */
    /* The index of this entry is the File Descriptor of the file. */

```

```ad-question
title: Why must a free Per Process Resource Table entry be found before calling the open() module function?
```

```ad-question
title: Why should the "root" file be treated seperately? Where is this change implimented for the open system call?
```

```ad-question
title: Why do we maintain OPEN INSTANCE COUNT in Open File table and FILE OPEN COUNT in File Status table? Why do we need two tables?
```

## Close system call

![[Pasted image 20231016195335.png]]

- The block discusses the process of closing an open instance of a file.
- When a process is done with reading or writing operations on a file, it can close the open instance of that file.
- This is done by using the Close system call.
- It is important to note that even if a process does not explicitly close the open instance by invoking the Close system call, the open instance will still be closed automatically.
- This automatic closure happens at the termination of the process by using the Exit system call.

+ The close system call is used to close a file in a program.
+ It takes a file descriptor, which is an index in the per-process resource table, as an argument from the user program.
+ The close system call invalidates the per-process resource table entry for the given file descriptor by storing -1 in the Resource Identifier field.
+ To decrement the share count of the open instance in the open file table and update the file status table accordingly, the Close function of the file manager module is invoked by the Close system call.

```ad-note
The Close system call closes an open file. The file descriptor ceases to be valid once the close system call is invoked.
```
### Parameters
#### Arguments
+ File Descriptor (Integer)

#### 2. Return Value
![[Pasted image 20231016195617.png]]

### Algorithm
```c

Set the MODE_FLAG in the process table entry to 3, 
indicating that the process is in the close system call.

//Switch to Kernel Stack - See Kernel Stack Management during System Calls. 
Save the value of SP to the USER SP field in the Process Table entry of the process.
Set the value of SP to the beginning of User Area Page.

If file descriptor is invalid, return -1.    /* File descriptor value should be within the range 0 to 7 (both included). */

Locate the Per-Process Resource Table of the current process.
    Find the PID of the current process from the System Status Table.
    Find the User Area page number from the Process Table entry.
    The Per-Process Resource Table is located at the  RESOURCE_TABLE_OFFSET from the base of the  User Area Page

If the Resource identifier field of the Per Process Resource Table entry is invalid or does not indicate a FILE, return -1.   

/* No file is open with this file descriptor. */

Get the index of the Open File Table entry from Per-Process Resource Table entry.

Call the close() function in the File Manager module with the Open File Table index as arguement.

Invalidate the Per-Process Resource Table entry.

Set the MODE_FLAG in the process table entry to 0.
Switch back to the user stack.

Return from system call with 0.    /* success */
```

```ad-question
title: Why did we not check if the file is locked?
```





