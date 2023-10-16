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

```ad-tip
title: Algorithm Explained
1. The Open function is invoked by the Open system call to update the file status table and open file table when a file is opened.
2. The Open function takes a file name as an argument and locates the inode index for the file in the inode table.
3. The inode for the file is locked using the Acquire Inode function of the resource manager module to prevent other processes from deleting the file concurrently.
4. A new entry is created in the open file table, and its index is returned to the caller. This index is stored in the per-process resource table entry by the Open system call.
5. The fields of the open file table entry are initialized, with the INODE INDEX field set to 0 if the file is the "root" file.
6. The FILE OPEN COUNT field is incremented by one in the file status table entry for the file, except if the file is the "root" file.
7. The lock on the file is released by invoking the Release Inode function of the resource manager module before returning to the caller
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

```ad-tip
title: Algorithm Explained
1. The Close function is invoked by the Close system call to update the file status table and the open file table when a file is closed.
2. Close takes an open file table index as an argument.
3. Close function decrements the share count (i.e OPEN INSTANCE COUNT field in the open file table entry) as the process no longer shares the open instance of the file.
4. When the share count becomes zero, this indicates that all processes sharing that open instance of the file have closed the file.
5. Hence, open file table entry corresponding to that open instance of the file is invalidated by setting the INODE INDEX field to -1.
6. The open count of the file (FILE OPEN COUNT field in file status table entry) is decremented.
```

```ad-question
title: Why did we not check if the file is locked?
```

## Fork system call

### Modifications
+ The Fork System call needs a modification.
+ When a process forks to create a child process, the files opened by the parent are now shared between child and parent.
+ To reflect this change, the OPEN INSTANCE COUNT field in the open file table is incremented for each open file instance in the per-process resource table of parent process.

```ad-tip
title: Modifications (detail)
1. Copy the per-process resource table of a parent to a child process.
2. If a resource in the per-process resource table is a file, it will be identified using the Resource Identifier field in the table.
3. The open file table index will be used to locate the entry of the file in the open file table.
4. The OPEN INSTANCE COUNT field in the open file table entry will be incremented.
5. This incrementation is necessary to keep track of the number of instances of the file that are currently open.
6. This change in the Fork system call is already implemented in stage 22.
7. The semaphore table is also updated in stage 22 to ensure synchronization between processes accessing shared resources.
```

## Process Manager Module
> Free User Area Page (Function no. = 2)
### Modifications
+ When a process ends, all the files it has opened must be closed.
+ This is performed by the Free User Area page function.
+ The Close function of the file manager module is called for each open file in the per-process resource table of the process.

```ad-tip
title: Modification (detail)
1. The block of code iterates through each entry in the per-process resource table of a process.
2. It checks if the resource is valid and is a file by checking the Resource Identifier field in the per-process resource table.
3. If the resource is a valid file, it invokes the Close function of the file manager module to close the file.
4. The change in the Free User Area Page to release the unrelased semaphores has already been done in stage 22, so it does not need to be repeated here.
```

## Interrupt routine 6 

Interrupt routine 6 in stage 16 reads data only from the terminal. In this stage, the Read system call is modified to read data from files. The Read system call has a system call number of 7. In ExpL programs, the Read system call is called using the exposcall function.

![[Pasted image 20231016233632.png]]

## Read System call

- The read system call is a function that takes two inputs: a file descriptor and the address of a word where the data should be read.
- The purpose of the read system call is to lock the inode corresponding to the file descriptor at the beginning of the system call.
- The inode is a data structure that contains information about a file, such as its size, permissions, and location on the storage device.
- Locking the inode ensures that other processes or threads cannot access or modify the file while it is being read.
- After the data has been read, the read system call releases the lock on the inode at the end of the system call.
- The release of the lock allows other processes or threads to access or modify the file if needed.
- To lock and release the inode, the read system call utilizes two functions from the resource manager module: Acquire Inode and Release Inode.
- The Acquire Inode function is responsible for acquiring the lock on the inode, while the Release Inode function releases the lock.
- By using these functions, the read system call ensures that proper synchronization and protection mechanisms are in place to prevent data corruption or conflicts during the reading process.

+ The "read" system call is used to read data from a file.
+ It reads a single word from the position pointed to by the value of LSEEK.
+ LSEEK is a value stored in the open file table entry and indicates the current position in the file.
+ The word read from the file is then stored in the memory address provided as input.
+ After reading the word, LSEEK is incremented by one, indicating that the next read operation should start from the next word in the file.

- The eXpOS operating system uses a buffer cache to store disk blocks in memory.
- The buffer cache can store up to four disk blocks at a time.
- The cache pages are numbered 0, 1, 2, and 3 and are stored in memory pages 71, 72, 73, and 74.
- When a disk block needs to be loaded into memory, it is stored in the cache page determined by the block number modulo 4.
- For example, if the disk block number is 195, it will be stored in cache page number 3, which corresponds to memory page number 74.
- The file manager module provides functions called Buffered Read and Buffered Write to handle buffer management.
- The Read function uses the Buffered Read function to bring the required disk block into the memory buffer.
- Once the disk block is loaded into the buffer, the word at the specified position (pointed to by LSEEK) can be read from the buffer.

+ Reading from a root file does not require a buffer because the root file is already loaded into memory at boot-time.
+ The memory copy of the root file is present in memory page 62, and the start address of this page is denoted by the SPL constant ROOT_FILE.
+ To read from the root file, the word at a specific position (denoted by LSEEK) is copied into the provided address.
+ The memory address provided as an argument is a logical address, but since the system call runs in kernel mode, this logical address needs to be translated to a physical address before data can be stored.

- The read system call in an operating system needs to lock three resources before using them: the Inode (file), buffer, and disk.
- These resources are locked in a specific order: first the Inode, then the buffer, and finally the disk.
- The resources are released in the reverse order: first the disk, then the buffer, and finally the Inode.
- This order of resource acquisition is important to prevent processes from getting into a circular wait for resources.
- Circular wait refers to a situation where two or more processes are waiting for each other's resources, leading to a deadlock.
- By following the order of locking and releasing resources, the operating system ensures that circular wait is avoided and deadlocks are prevented.

### Algorithm
```c

Set the MODE_FLAG in the process table entry to 7, 
indicating that the process is in the read system call.

//Switch to Kernel Stack - See Kernel Stack Management during System Calls. 
Save the value of SP to the USER SP field in the Process Table entry of the process.
Set the value of SP to the beginning of User Area Page.

If input is to be read from terminal    /* indicated by a file descriptor value of -1 */
    Call the terminal_read()function in the Device manager  Module .

/* If not terminal, read from file. */
 else 

    If file descriptor is invalid, return -1.    /* File descriptor value should be within the range 0 to 7 (both included). */

    Locate the Per-Process Resource Table of the current process.
                Find the PID of the current process from the System Status Table.
                Find the User Area page number from the Process Table entry.
                The  Per-Process Resource Table is located at the  RESOURCE_TABLE_OFFSET from the base of the  User Area Page .
    
    If the Resource identifier field of the Per Process Resource Table entry is invalid or does not indicate a FILE, return -1.  
    /* No file is open with this file descriptor. */

    Get the index of the Open File Table entry from the Per Process Resource Table entry.

    Get the index of the Inode Table entry from the Open File Table entry. 

    Acquire the Lock on the File by calling the acquire_inode() function in the Resource Manager module.
    If acquiring the inode fails, return -1.

    Get the Lseek position from the Open File Table entry.

    Get the physical address curresponding to the logical address of Memory Buffer address given as input.

    If the File corresponds to Root file ( indicated by Inode index as INODE_ROOT)  
                  If the lseek value is equal to the root file size(480), release_inode() return -2. 

                  Read from the word at lseek position in memory copy of root file to the translated memory address. 
                  /* Use SPL Constant ROOT_FILE */

                  Increment the Lseek position in the Open File Table.        
     else 
        If lseek position is same as the file size, release_inode() and return -2.  /* End of file reached */

        Find the disk block number and the position in the block from which input is read.
            Get the block index from lseek position.   /* lseek/512 gives the index of the block */
            Get the disk block number corresponding to the block index from the Inode Table .
            Get the offset value from lseek position.   /* lseek%512 gives the position to be read from.*/
        
        Read the data from the File Buffer by calling the buffered_read() function in the File Manager module.

        Increment the Lseek position in the Open File Table.

    Release the Lock on the File by calling the release_inode() function in the Resource Manager module.

Switch back to the user stack by resoting USER SP from the process table.
Set the MODE_FLAG in the process table entry of the parent process to 0.

Return 0.   /* success */

```

















