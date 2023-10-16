# Expos File System Implementation

There are two categories of file data structures.
	1. Disk Data Structures
	2. Transient Data Structures

The first category consists of data that remains in the disk even when the machine is shut down (*disk data structures​*).
## Disk Data structures[¶](https://exposnitc.github.io/expos-docs/tutorials/filesystem-implementation/#disk-data-structures "Permanent link")

### 1. Disk blocks and Disk Free List[¶](https://exposnitc.github.io/expos-docs/tutorials/filesystem-implementation/#1-disk-blocks-and-disk-free-list "Permanent link")

- File data is stored in disk blocks, with each file having up to 4 blocks.
- The OS provides a sequential interface to users, even if the data is stored non-contiguously on disk.
- The Inode table entry for a file stores the block numbers of the disk blocks containing the file data.
- The disk free list is a global structure indicating allocated and free disk blocks.
- The Write system call allocates disk blocks for files as needed.
- The Write system call uses the Get Free Block function from the Memory Manager Module to allocate blocks.
- Disk blocks are deallocated when files are removed using the Delete system call, invoking the Release Block function.
- The disk free list is updated when blocks are allocated or released.
- Meta-data for files is stored in data structures like the Inode table and the root file.

### 2. Inode Table[¶](https://exposnitc.github.io/expos-docs/tutorials/filesystem-implementation/#2-inode-table "Permanent link")

- The Inode table is a global data structure with an entry for each file in the file system.
- When a file is created using the Create system call or loaded into the disk using the XFS interface, a new Inode entry is created for the file.
- The Inode entry of a file stores various attributes, including filename, file size, owner's user-id, file type, access permissions, and block numbers of allocated disk blocks (up to four blocks).
- When a file is created using the Create system call, only an Inode entry is created initially, with no disk blocks allocated, resulting in a file size of 0.
- The filename and access permissions are provided as arguments to the Create system call and are set accordingly.
- eXpOS allows the creation of data files using Create (executable files can only be loaded externally).
- The user-id of the process executing the Create system call becomes the owner of the file (eXpOS is a single-terminal system with a single user at a time).
- As data is written to the file using the Write system call, new disk blocks may be allocated, and the block numbers are recorded in the Inode table.
- Files can be created with Exclusive Access or Open Access permission. The access permission is specified as an argument to the Create system call.
- With exclusive access, Delete and Write system calls will only succeed if executed by the root user or the owner of the file. Other users cannot modify or delete such files.
- Open access permission allows all users to perform any operation on the file.

### 3. Root File[¶](https://exposnitc.github.io/expos-docs/tutorials/filesystem-implementation/#3-root-file "Permanent link")

- The root file is a data structure that contains human-readable information about each file in the file system.
- The eXpFS file system lacks hierarchical directories, and all files are listed at a single level.
- Each file has an entry in the root file, and the kth entry in the root file corresponds to the file with index k in the Inode table.
- The root file entry for a file includes filename, file size, file type, user name, and access permissions.
- Some data in the Inode table is duplicated in the root file because the root file is designed to be readable by user programs using the Read system call (unlike the Inode table, which is accessed exclusively by the OS).
- An application-readable root file allows the implementation of commands like "ls" (similar to the Unix "ls" command) as user-mode programs.
- Write and Delete system calls are not permitted on the root file.
- The only data in the root file entry not present in the Inode table is the user name of the file's owner.
- The Inode table entry of a file contains the user-id of the owner, which can be used to index into the user table to find the corresponding username.
- When a file is created, the Create system call initializes both the root file entries and the Inode table entries for the file.
- When the file size is changed in the Inode table due to a write to the file using the Write system call, the file size value in the root file also needs to be updated to reflect the change.

### 4. User Table[¶](https://exposnitc.github.io/expos-docs/tutorials/filesystem-implementation/#4-user-table "Permanent link")

- The User table contains the names of each user with an account in the system.
- While the User table is not directly associated with the file system, it's relevant to understand this data structure for file system implementation purposes.
- The details of how and when User table entries are created are not discussed in the context of file system implementation.
- Each user has an entry in the User table, consisting of:
	- Username
	- Encrypted password
- The OS assigns a user-id to each user, where the user-id is the index of the user's entry in the User table.
- The first two entries of the User table (user-id 0 and 1) are reserved for special users: "kernel" and "root."
- When a process executes the Create system call to create a file:
  - The Create system call looks up the process table entry of the calling process to find the user-id of the process executing Create.
  - It sets the user-id field in the Inode table to indicate the owner of the file.
  - The system call then looks up the User table entry corresponding to the user-id to retrieve the username.
  - It sets the username field in the root file entry created for the new file to reflect the owner of the file.

---

The second category of data structures are transient - they are "alive" only when the OS is running (in-memory data structures). These data structures are described below:

## Transient data structures[¶](https://exposnitc.github.io/expos-docs/tutorials/filesystem-implementation/#transient-data-structures "Permanent link")

- Transient data structures are in-memory data structures that are active only when the OS is running.
- When a user process opens, reads, or writes an already created file using the Open system call, a new "open instance" is created. The OS keeps track of the number of open instances of a file.
- Each open instance of a file has an associated seek pointer, initialized to the beginning of the file (value 0) when opened. Reading or writing to the file updates the position corresponding to the seek value, and the seek value is incremented. If a process opens a file and then forks, the seek pointer is shared between the parent and child processes. The shared seek pointer advances when either process reads or writes to the file, and it can be modified using the Seek system call, allowing multiple processes to share file access.
- When a process closes an open instance using the Close system call, the Close system call checks if the open instance is shared by other processes (e.g., parent, child, sibling). If it is shared, the OS decrements the "share count" of the open instance. If the last process that shared the open instance closes the file, the share count reaches zero, and the open instance is closed.
- To implement this file access and sharing mechanism, the OS maintains several global data structures:
  - The file status table (also called the inode status table)
  - The open file table
  - A per-process resource table containing information about open file instances specific to each process.
- The OS also manages a buffer cache for caching data blocks of files in current use. A buffer table is used to manage the data related to the buffer cache.

These data structures and mechanisms are essential for managing file access, sharing, and caching within the operating system.

Refer the following
1. [Open File Table](3-Memory-DS#Open File Table)
2. [File (Inode) Status Table](3-Memory-DS#File (Inode) Status Table)

### 1. File (Inode) Status Table[¶](https://exposnitc.github.io/expos-docs/tutorials/filesystem-implementation/#1-file-inode-status-table "Permanent link")

- The File (Inode) Status Table contains an entry for each file in the file system, and the index of an entry in the Inode table corresponds to the index in the File Status Table.
- The table serves two main purposes:
  1. To keep track of how many times each file has been opened using the open system call.
  2. To provide a mechanism for processes to lock a file before making updates to the file's data or metadata.
- When a file is opened using the Open system call, the file open count field in the corresponding File Status Table entry is incremented, providing a global count of the number of open instances of that file.
- When a process attempts to access a file within a file system call, the system call code must lock access to the file to prevent concurrent access by other processes. This locking is essential for safety during concurrent execution.
- The system call code locks the file by setting the locking-PID field of the File Status Table entry to the PID (Process ID) of the process executing the system call.
- After completing the system call, the system call code must unlock the file before returning to user mode. The Acquire Inode and Release Inode functions of the Resource Manager Module handle file access regulation, including locking and unlocking.
- The SPL constant FILE_STATUS_TABLE is set to the beginning address of the File Status Table in memory, as part of the system's memory organization.

### 2. Open File Table[¶](https://exposnitc.github.io/expos-docs/tutorials/filesystem-implementation/#2-open-file-table "Permanent link")

- When a process opens a file using the Open system call and subsequently uses the Fork system call, the open instance of the file is shared between the parent and the child processes. Additional forks can lead to more processes sharing the same open instance, requiring a mechanism to track the count of such processes. The Open File Table serves this purpose.
- Each time a file is opened by a process, an entry is created in the Open File Table for that open instance. This entry contains three fields:
	- The index of the Inode table of the file.
	- The count of the number of processes sharing the open instance, initially set to 1 when the file is opened. The count is incremented when the process executes a Fork system call, reflecting the correct number of processes sharing the open instance. (Note: This count is different from the file open count in the File Status Table.)
	- The seek pointer for the open instance is stored in the Open File Table. Read and write operations on the open instance must read from or write to this position in the file and advance the pointer. When a file is initially opened, the seek position is set to 0. Importantly, the seek pointer is shared among all processes sharing the open instance.
- The SPL constant OPEN_FILE_TABLE is set to the beginning address of the Open File Table in memory, as part of the system's memory organization.
- When a process executes a Read or Write system call on an open instance, the system call handler, in addition to performing the data read or write operation on the file, advances the seek pointer in the Open File Table entry corresponding to that open instance.

### 3. Per-Process Resource Table[¶](https://exposnitc.github.io/expos-docs/tutorials/filesystem-implementation/#3-per-process-resource-table "Permanent link")

- When a process opens a file, a new entry is created in the per-process resource table specific to that process.
  - This entry has two fields:
    a) A flag indicating whether it corresponds to a file or a semaphore.
    b) The index of the corresponding entry in the Open File Table or Semaphore Table, depending on whether it's an open file or semaphore instance. For this discussion, we focus on the case when the entry corresponds to an open file.
- The Open system call returns the index of an entry in the resource table as the file descriptor to the user. This file descriptor is used as an argument in subsequent Read, Write, Seek, and Close system calls. These system calls use the descriptor to identify the open instance, determined uniquely by the Open File Table index associated with the file descriptor.
- When a process forks a child process using the Fork system call, the entries in the resource table of the parent process are copied to the resource table of the child process. As a result, the child inherits the open file instances from the parent.
- An example scenario is provided: Process B is a child of Process A, and both share an open instance of a file named "myfile.dat" with an Inode index of 5. The open file table index for this open instance is, for example, 2. The diagram illustrates the various table entries related to this open instance.

This mechanism allows processes to manage and share open file instances and ensures that child processes inherit their parent's open instances when using the Fork system call.

![[Pasted image 20231007125949.png]]

### 4.​ ​ Memory buffer Cache[¶](https://exposnitc.github.io/expos-docs/tutorials/filesystem-implementation/#4-memory-buffer-cache "Permanent link")

- When a process attempts to Read or Write data from/to a file, the relevant block of the file is first loaded into a disk buffer in memory. The Read/Write operation is then performed on a copy of the block stored in this buffer.
- The operating system (OS) maintains four memory buffer pages as a cache, numbered 0, 1, 2, and 3. These buffers are located in memory pages 71, 72, 73, and 74, as part of the system's memory organization.
- A simple buffering scheme is used: When there's a request for the ith disk block, it is brought into the buffer with the number (i mod 4). This means that the buffer location is determined by taking the remainder of i divided by 4.
- If the buffer is currently holding another disk block when a new block is requested for loading, the OS checks whether the current block in the buffer needs to be written back to disk (is dirty) before replacing it with the requested block.

### 5.​ ​ Buffer Table[¶](https://exposnitc.github.io/expos-docs/tutorials/filesystem-implementation/#5-buffer-table "Permanent link")

- The Buffer Table is used to manage the buffer cache. It contains one entry per buffer page, and each entry has:
  a) The block number of the disk block currently stored in the buffer page. If the buffer is unallocated, the disk block number is set to -1.
  b) A flag indicating whether the block was modified after loading (dirty).
  c) The PID of the process that has locked the buffer page (-1 if no process has locked the buffer).
- The locking PID field helps prevent concurrent access to the same buffer page by multiple processes. When a process attempts to read/write a specific data block of a file using the Read/Write system call, it determines the buffer number where the block should be loaded and locks that buffer to ensure exclusive access.
- To illustrate the operation of file system calls with these data structures, consider the example of a process executing a Write operation on an open file instance. The process uses the file descriptor to find the corresponding entry in the open file table and retrieves the seek pointer and Inode table entry for the file.
- The Write operation locks the Inode (using the Acquire Inode function of the Resource Manager Module), determines the disk block to write to based on the seek position and Inode entry, and invokes the Buffered Write function of the File Manager Module to perform the write.
- Buffered Write calculates the buffer number for the block and locks the buffer (using the Acquire Buffer function of the Resource Manager Module). It checks if the required disk block is already in the buffer. If not, it loads the block into the buffer. If the buffer is currently holding another disk block and that block is dirty, it must be written back to disk before the new block is loaded.
- Write Back is performed using the Disk Store function of the Device Manager Module, which acquires a resource lock (calls the Acquire Disk function of the Resource Manager Module) before committing to the disk.
- After ensuring the buffer page is available, Buffered Write brings the required disk block into the buffer page using the Disk Load function of the Device Manager Module. Then, it writes the data into the buffer and sets the dirty bit in the buffer table entry.
- Note that Write does not immediately store the modified buffer back to the disk. Modifications are committed only when a subsequent write operation requires the buffer to be loaded with a different disk block. All dirty buffers are committed to the disk before system shutdown.
- Read operations follow a similar process but do not set the dirty bit as the data is not modified.
- Resource acquisitions during Read/Write system calls follow a specific order: Inode, buffer, and disk. These resources must be released in reverse order to prevent circular waits and ensure deadlock prevention.

### 6.​ In-Memory Copy of Disk data structures[¶](https://exposnitc.github.io/expos-docs/tutorials/filesystem-implementation/#6-in-memory-copy-of-disk-data-structures "Permanent link")

- The operating system (OS) maintains an in-memory copy of all the disk data structures, including the inode table, user table, root file, and the disk free list.
- While the OS is running, various changes can occur, such as creating new users or creating/modifying/deleting files. In such cases, the updates are made to the memory copy of the corresponding data structures and not directly to the disk copy.
- Before the system is shut down, the OS is responsible for writing back the memory copy of all disk data structures and any dirty buffers to the physical disk. This step is crucial to ensure that the changes made during the system's runtime are permanently saved to the disk.
- It's important to note that the file system implementation described here is not crash-resilient. In other words, if the OS crashes before or during the write-back process, the memory-to-disk updates may be incomplete or partial. This can lead to inconsistent states in the disk data structures. As a consequence, one or more files may become corrupted, and the disk may require reformatting to recover from such issues.






