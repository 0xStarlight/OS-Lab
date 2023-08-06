# Stage 02

```ad-abstract
+ Load/retrieve data and executable files from/to your host (Unix) system into the XSM disk.
+ Explain the disk data structures of the XFS file system - INODE table, disk free list and root file.
+ Find out the data blocks into which a data/executable file is stored in the XSM disk by examining the INODE table and root file.
```

```ad-attention
- Quickly go through the [Filesystem (eXpFS) Specification](https://exposnitc.github.io/expos-docs/os-spec/expfs/) and [XFS-Interface Specification](https://exposnitc.github.io/expos-docs/support-tools/xfs-interface/) (interface between the UNIX System and eXpFS). Do not spent more than 30 minutes!
```

---
## 1. Run the XFS Interface
```bash
cd $HOME/myexpos/xfs-interface
./xfs-interface
```

## 2. Init the disk
```bash
# fdisk
# exit
```

```ad-note
The fdisk command converts the raw disk into the filesystem format recognised by the eXpOS operating system. It initialises the disk data structures such as disk free list, inode table, user table and root file.
```

## 3. load the file to the XFS disk
```bash
# load --data $HOME/myexpos/sample.dat
```

## 4. Copy the inode table
```bash
# copy 3 4 $HOME/myexpos/inode_table.txt
# exit
```

## 5. Copy inode table
```bash
# dump --inodeusertable
```

```ad-question
**Q1.** When a file is created entries are made in the Inode table as well as the Root file. What is the need for this duplication?


**Ans.:** Inode table is a data structure which is accessible only in Kernel mode, whereas Root file is accessible both in Kernel and User mode. This enables the user to search for a file from an application program itself by reading the Root file.
```

---

```ad-question
title: Assignment 1
Copy the contents of Root File (from Block 5 of XFS disk) to a UNIX file $HOME/myexpos/root_file.txt and verify that an entry for sample.dat is made in it also.
```

## Answer
```
copy 5 5 $HOME/myexpos/test/out.txt
[[1-Bootstrap-Loader]]```

---

```ad-question
title: Assignment 2
Delete the sample.dat from the XSM machine using xfs-interface and note the changes for the entries for this file in inode table, root file and disk free list.
```

## Answer
```bash
rm sample.dat
df --> check them
```

---
# Misc
## Find the length of file
```python
file_path = input('Enter file path: ')
#file_path = "/home/kali/myexpos/test/testing.dat"
chunk_size=15
count = 0
tmp = 0
try:
	with open(file_path,'r') as file:
		for line in file:
			for char in line:
				tmp += 1
				if(tmp ==15 and char != '\n'):
					count +=1
					tmp = 0
				elif (char == '\n'):
					tmp = 0
					count +=1

		print(f"count : {count}")

except FileNotFoundError:
	print("File Not Found")
except IOError:
	print("Cant read the file")
```



























