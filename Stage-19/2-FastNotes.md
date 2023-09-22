# Fast Notes

+ The exception handler occupies **page 2 and 3 in the memory and blocks 15 and 16 in the disk**.
+ Four Special Registers for exception handling
	+ EC
	+ EIP
	+ EPN
	+ EMA
+ The cause of the exception is obtained from the value present in the EC register.
+ The types of exceptions and the corresponding EC values
	+  Illegal Instruction ( EC = 1)
	+ Illegal Mem Address ( EC = 2 )
	+ Arithmetic Exception (EC = 3)
	+ The above exceptions can not be corrected and hence are halted as they are human errors
+ 4th Exception which we can work on is
	+ Page Fault ( EC = 0 )
	+  The exception occured not because of any error from the side of the application, but because the OS had not loaded the page and set the page tables. In such case, **the exception handler resumes the execution of the process after allocating the required page(s) for the process and attaching the page(s) to the process (by setting page table entries appropriately). If the faulted page is a code page, the OS needs to load the page from the disk to the newly allocated memory.**
+ Modules
	+ Module 0 : Resource Manager
	  ![[Pasted image 20230921141350.png]]
	+ Module 1: Process Manager
	  ![[Pasted image 20230921141402.png]]
	+ Module 2: Memory Manager
	  ![[Pasted image 20230921141413.png]]
	+ Module 3: File Manager
	  ![[Pasted image 20230921141425.png]]
	+ Module 4: Device Manager
	  ![[Pasted image 20230921141437.png]]
	+ Module 5: Scheduler Module
	  ![[Pasted image 20230921141450.png]]
	+ Module 6: Page Module
	  ![[Pasted image 20230921141502.png]]
	+ Module 7: Boot Module
	+ Module 8: Access Control Module
	  ![[Pasted image 20230921141516.png]]
