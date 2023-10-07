# PER-PROCESS DISK MAP TABLE[¶](https://exposnitc.github.io/expos-docs/os-design/process-table/#per-process-disk-map-table "Permanent link")

The per-process Disk Map Table stores the disk block number corresponding to the pages of each process. The Disk Map Table has 10 entries for a single process. When the memory pages of a process are swapped out into the disk, the corresponding disk block numbers of those pages are stored in this table. It also stores block numbers of the code pages of the process.

The entry in the disk map table entry has the following format:

| 0      | 1      | 2              | 3              | 4              | 5              | 6              | 7              | 8                    | 9                    | 
| ------ | ------ | -------------- | -------------- | -------------- | -------------- | -------------- | -------------- | -------------------- | -------------------- |
| Unused | Unused | Heap 1 in disk | Heap 2 in disk | Code 1 in disk | Code 2 in disk | Code 3 in disk | Code 4 in disk | Stack Page 1 in disk | Stack Page 2 in disk |

If a memory page is not stored in a disk block, the corresponding entry must be set to -1.

```ad-note

The Disk Map Table is present in page 58 of the memory (see [Memory Organisation](https://exposnitc.github.io/expos-docs/os-implementation/)), and the SPL constant [DISK_MAP_TABLE](https://exposnitc.github.io/expos-docs/support-tools/constants/) points to the starting address of the table. DISK_MAP_TABLE + PID*10 gives the begining address of disk map table entry corresponding to the process with identifier PID.
```



