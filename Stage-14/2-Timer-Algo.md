# Timer Interrupt Handler

The hardware requirement specification for eXpOS assumes that the machine is equipped with a timer device that sends periodic hardware interrupts. The timer interrupt handler internally invokes the eXpOS [scheduler module](https://exposnitc.github.io/expos-docs/modules/module-05/) .

![[Pasted image 20230829141847.png]]

---

### Algorithm:[¶](https://exposnitc.github.io/expos-docs/os-design/timer/#algorithm "Permanent link")

```ad-note
Switch to the Kernel Stack. /* See [kernel stack management during system calls](https://exposnitc.github.io/expos-docs/os-design/stack-smcall/) */ Save the value of SP to the USER SP field in the [Process Table](https://exposnitc.github.io/expos-docs/os-design/process-table/) entry of the process. Set the value of SP to the beginning of User Area Page.

Backup the register context of the current process using the [BACKUP](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/) instruction.

/* This code is relevant only when the Pager Module is implemented in Stage 27 */

	If swapping is initiated, /* check System Status Table */
	{
	    /* Call Swap In/Out, if necessary */
	
	    if the current process is the Swapper Daemon and Paging Status is SWAP_OUT,
	        Call the swap_out() function in the Pager Module.
	
	    else if the current process is the Swapper Daemon and Paging Status is SWAP_IN, 
	        Call the swap_in() function in the Pager Module.
	
	    else if the current process is Idle,                          
	        /* Swapping is ongoing, but the daemon is blocked for some disk operation and idle is being run now */
	        /* Skip to the end to perform context switch. */
	
	}
	
	else           /* Swapping is not on now.  Check whether it must be initiated */
	{
	    if (MEM_FREE_COUNT < MEM_LOW)       /* Check the System Status Table */
	        /* Swap Out to be invoked during next Timer Interrupt */
	        Set the Paging Status in System Status Table to SWAP_OUT.
	
	    else if (there are swapped out processes)            /* Check SWAPPED_COUNT in System Status Table */
	        if (Tick of any Swapped Out process > MAX_TICK or MEM_FREE_COUNT > MEM_HIGH)
	            /* Swap In to be invoked during next Timer Interrupt */
	            Set the Paging Status in System Status Table to SWAP_IN.
	
	}

Change the state of the current process in its Process Table entry from RUNNING to READY.

Loop through the process table entires and increment the TICK field of each process.

Invoke the [context switch module](https://exposnitc.github.io/expos-docs/modules/module-05/) .

Restore the register context of the process using [RESTORE](https://exposnitc.github.io/expos-docs/arch-spec/instruction-set/) instruction. 

Set SP as the user SP saved in the Process Table entry of the new process. Set the MODE_FLAG in the [process table](https://exposnitc.github.io/expos-docs/os-design/process-table/) entry to 0.

ireturn.
```



