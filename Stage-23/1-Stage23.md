# File Creation and Deletion 
---

# Summary

This stage discusses how files are created and deleted using file system calls, and how the Shutdown system call is modified to commit the changes made by Create and Delete system calls in the memory copy of the disk data structures back into the disk. The Inode table and root file store details of every eXpOS file stored in the disk and have MAX_FILE_NUM entries. The Create and Delete system calls are implemented in interrupt routine 4, while the Shutdown system call is implemented in interrupt routine 15. The Acquire Inode and Release Inode functions are used to lock and unlock the inode (file) respectively. Finally, the changes made in the memory copies of the inode table, root file, and disk free list are committed to the disk using the Disk Store function of device manager module, followed by halting the system using the SPL statement halt.

---

# Starting

#### Create System Call
- The Create system call is used to create an empty file with a specified name.
- When Create is called, it initializes the disk data structures, including the Inode table and root file, with metadata related to the new file.
- The Delete system call is responsible for deleting the record of a file with a given name from the Inode table and root file.
- In addition to removing the record, Delete also releases the disk blocks that were occupied by the file being deleted.
- The Shutdown system call is modified to ensure that the changes made by the Create and Delete system calls in the memory copy of the disk data structures are committed back into the physical disk. This step is crucial to make the changes permanent.

#### Inode Table and Root File
- Inode table and root file are used to store details of every file in the eXpOS disk. The system allows a maximum of MAX_FILE_NUM (60) files to be stored on the disk, and as a result, both the inode table and root file contain MAX_FILE_NUM entries.
- Each file in the inode table is identified by an index corresponding to its record in the inode table. The entries in both the inode table and root file share the same index for each file.
- To use these disk data structures during the OS's runtime, they must be loaded from the disk into memory.
- The OS maintains a memory copy of the inode table in memory pages 59 and 60, and the memory copy of the root file is found in memory page 62. The memory organization is further detailed to understand the specific memory locations for these data structures.

#### Interrupt routine 4
![[Pasted image 20231011110342.png]]
The system calls _Create_ and _Delete_ are implemented in the interrupt routine 4. _Create_ and _Delete_ have system call numbers 1 and 4 respectively. From ExpL programs, these system calls are called using [exposcall function](https://exposnitc.github.io/expos-docs/os-spec/dynamicmemoryroutines/) .


## Functions

### Create system call
- The Create system call accepts two arguments from the user program: 
	- The filename
	- Permission flag (0 - exclusive OR 1 - open-access)).
- Create is designed to create data files, and it is recommended to use the ".dat" extension for file names.
- When Create is called, it searches for a free entry in the inode table, which is indicated by a -1 in the FILE NAME field.
- The fields in the free inode table entry and the corresponding root file entry are initialized with metadata related to the new file.
- The USERID field in the inode table entry is set to the USERID field from the process table of the current executing process. This designates the user executing the Create system call as the owner of the newly created file.
- The USERNAME field in the root file entry is initialized with the username associated with the USERID. This username can be obtained from the memory copy of the user table, using the USERID as an index.
- The user table on the disk is initially configured by the XFS-interface during disk formatting and includes entries for two users: "kernel" and "root."

#### Algorithm for create system call
> The Create operation takes as input a filename. If the file already exists, then the system call returns 0 (success). Otherwise, it creates an empty file by that name, sets the file type to [DATA](https://exposnitc.github.io/expos-docs/support-tools/constants/), file size to 0, userid to that of the process (from the [process table](https://exposnitc.github.io/expos-docs/os-design/process-table/)) and permission as given in the input in the [Inode Table](https://exposnitc.github.io/expos-docs/os-design/disk-ds/#inode-table). It also creates a root entry for that file.

```.
Set the MODE_FLAG in the process table entry to 1, 
indicating that the process is in the create system call.

If the file is present in the system, return 0.   /* Check the Inode Table  */ 

Find the index of a free entry in the Inode Table. 
If no free entry found, return -1.   /* Maximum number of files reached */

In the Inode Table entry found above, set FILE NAME to the given file name, FILE SIZE to 0 and FILE TYPE to DATA.
In the Inode Table entry, set the block numbers to -1.  /* No disk blocks are allocated to the file */

Set the USER ID to the USERID of the process /* See the process table for user id */
Set the PERMISSION to the permission supplied as input.

In the Root file entry corresponding to the Inode Table index, 
set the FILE NAME, FILE SIZE, FILE TYPE, USERNAME and PERMISSION fields.

Set the MODE_FLAG in the process table entry to 0.

Return from the system call with 0.  /* success */
```

### Delete System Call
- The Delete system call accepts the file name as an argument from the user program and is used to delete files.
- A file cannot be deleted if it is currently opened by one or more processes. To delete a file, Delete first acquires a lock on the file by invoking the Acquire Inode function from the resource manager module.
- Delete then invalidates the record of the file in the entries of both the inode table and the root file, effectively removing all references to the file.
- Additionally, the disk blocks allocated to the file are released and marked as available for future use.
- Finally, Delete releases the lock on the file by invoking the Release Inode function of the resource manager module.
- A subtlety in file deletion involves handling disk blocks in the buffer cache. If any of the disk blocks of the deleted file are present in the buffer cache and are marked as "dirty," the OS would typically write back the buffer page to the disk block when another disk block needs to be loaded into the same buffer page. However, such write-back is unnecessary and can even be problematic if the file is being deleted. Therefore, the Delete system call must clear the "dirty bit" in the buffer table for all buffered disk blocks associated with the file. This ensures that these blocks are not inadvertently written back to the disk, as the file is being deleted.



---

# Questions

```ad-question
title: What would happen if we do not initilize the FILE OPEN COUNT in the File Status Table to -1? (Create Syscall)

Answer: This specifies the number of open instances of the file. This field is set to -1 if there are no open instances of the file in the system.
```

```ad-question
title: In the Inode table entry why do we set the block number to -1 after creaion of the file? (Create Syscall)
Answer: There is no data inside the file
```


---

# Assignments



> NOTES INCOMPLETE >> will do >> im too lazy now 
































