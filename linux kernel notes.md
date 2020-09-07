





### Process



`include/linux/sched.h`

```C
struct task_struct {

...

  // -1 unrunnable , 0 runnable , >0 stopped:
	volatile long 							state;

	// Current CPU:
	unsigned int 								cpu;
	cpumask_t 									cpus_allowed;

	const struct sched_class 		*sched_class;
	struct sched_entity 				se;
	int 												prio;

	pid_t 											pid;


	// Real parent process:
	struct task_struct __rcu 		*real_parent;
	struct list_head 						children;


	// Filesystem information:
	struct fs_struct						*fs;

	// Open file information:
	struct files_struct 				*files;


	...
}
```

- Above get manged in doublly linked `task_list`

- When a process makes a `clone()` syscall, `_do_fork()` is executed due to system call definitions of clone in `kernel/fork.c.` call reached to `copy_process()` which does most of the intersting parts like

  - error handling and the duplicate task with `dup_task_struct()`

  - scheduling specific work done by `sched_fork()`

  - based on the clone flag parent's resource get copied, this is important while forking for a thread because linux treats thread as a process with some additional resource sharing with the parent. Thread cloning flags

    ```
    clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND, 0);
    ```

  - `alloc_pid()` gives a pid to the process.

- `_do_fork()` wakes up newly created task with `wake_up_new_task()`

- Note about golang subroutine - they do not translate into OS threads.





#### Exercise

- look at `pidstat` cheat sheets to know more information about process
- `execsnoop` is the only efficient way to track past spawned processes.








### Scheduling

There are scheduling classes

- **SCHED NORMAL** - normal process class
- **SCHED FIFO ** - this is the class which run in the real time priority without any time slice, that means it process with this class of scheduling runs till it is finished or interrupted by other process with the higher priority.
- **SCHED RR** - this is the real time priority with the time slice. This process gets preempted.



O(1) scheduler was the old scheduler where according to nice value attached to each process there was a fixed time slice given to process. It is assumed, that the scheduler assigns timeslices of 100ms to processes with default priority (nice value 0) and timeslices of 5ms to processes with the least priority (nice value 19). Process priorities between 0 and 19 are also mapped to fixed timeslices. This had a lot of inefficencies and got replaced in `2.6` kernel with CFS.



#### CFS

This algorithm try to be fair with the scheduling prioirites of the task. Amount of time can be taken by a process depends on

- niceness
- amount of time spent by a process on the CPU

This also manages real-time priority tasks.



Configuration param `sched_latency_ns` which defines targeted preemption latency for CPU bound tasks. That means, it defines timeslice for each tasks based on number tasks in the run queue.

```
timeslice for the task = sched_latency_ns * (task's weight/total weight of tasks in the run queue)
```

There is a complimentry param `sched_min_granularity_ns` which governs minimum value of the `timeslice` of the task to reduce OS's preempting overheads.

When number of tasks becomes more than the ratio of `sched_latency_ns/sched_min_granularity_ns` then `sched_latency_ns`  value get replaced by ` sched_min_granularity_ns * total tasks`.



Once above timeslice is assigned to the process it get scheduled for a run on the CPU and accounted its CPU execution time as process's runtime. This process runtime is abstracted in the form of `vruntime` based on priority of the process. Higher priority process are recorded for lesser than their actual runtime on the CPU. This enabled them to have more CPU time.

`vruntime` is a part of the `sched_entity` struct, which itself is referenced in the `task_struct` of a process. This update is done by `update_curr` which get invovked periodically and adds value of `calc_delta_fair` to task's `curr->vruntime` in the weighted form.



sched/fair.c

```C
//Update the current task’s runtime statistics.
static void update_curr(struct cfs_rq *cfs_rq)
{
	struct sched_entity *curr = cfs_rq ->curr;
  u64 now = rq_clock_task(rq_of(cfs_rq));
  u64 delta_exec;
...
  delta_exec = now - curr->exec_start;
  curr->exec_start = now;
...
//statistics
  schedstat_set(curr->statistics.exec_max,
  max(delta_exec, curr->statistics.exec_max));

  curr->sum_exec_runtime += delta_exec;
  schedstat_add(cfs_rq->exec_clock, delta_exec);

  curr->vruntime += calc_delta_fair(delta_exec, curr);     
  update_min_vruntime(cfs_rq);
}
```



`sched_migration_cost_ns` is reference number to determine the cache hotness and to void its migration.

Now, processes are sorted based on the `vruntime` in the form of red-black balanced binary tree. Any insert, update and delete can be made in O(log(n)). This tree's leftmost node is always with the least runtime which becomes the next runnable process. This happens with the `pick_next_task`.



IO bound tasks get removed from the above RB tree with the `dequeue_entity` while getting marked as a NONRUNNABLE. Task marks itself as sleeping and puts itself on the wait queue. , calls `schedule()` to select next process to run. When such tasks wakes up and get added back into the run queue, it will have vruntime setup to 0 which makes it most runnable task.





Preemption of the task can only happen when it is not holding any locks and kernel keeps track of the count of locks by the task in the form of `preempt_count`.  Task will only get preempted if that count is zero. Sometimes higher priority task sets up `need_resched` on currently blocked task to get rescheduled immidiately once it comes out of interrupt handler.



- WatchDog and migration has real time priority.



#### Exercise

- where IO bound processes sleeps? the work queue ? How such queue is implemented. How such process get awaken.

- `vruntime` ??

- Scheduling related ebpf exercises

- confirm SCHD_FIFO behaviour

- what are kernel watchdogs?

- this [doc](https://doc.opensuse.org/documentation/leap/archive/42.1/tuning/html/book.sle.tuning/cha.tuning.taskscheduler.html)

- how perf tool works

- Nice time shown in the top output?









# Interrrupts

- Interrupt processing is done while normal scheduling is halted.
- Interrupt execution is divided into
  - Top Halves
  - Bottom Halves (softirqs, tasklets and work queues)
- Interrupt Descriptor Table (IDT) which stores ISR and an entry for each interrupt is called a gate.
- PCI-MSI - Message Signaled Interrupts are different PCI beast
- `/proc/interrupts`



#### Bottom Halves

This section of the code carried out by following

- Softirq : these are the actions setup during compile time of the kernel and get triggered by various IRQs as apart of bottom halves processing. Multiple sofirq instances can run in parallel. This get scheduled in the IRQ context.
- Tasklet: these are implemented on top of softirq and defined at the runtime. It has a serial execution.
- Workqueue:  this is a worker thread attached per processor. These usually get scheduled outside of the irq context so they can be blocked or sleep and has less memory allocation restirctions. Kworker process can be seen in `kworker/%u:%d%s (cpu, id, priority)` format.



Softirqs are used generally for urgent execution like netowrk traffic handling. Various other work actions implemented through softirq can be found `/proc/softirqs`











####Exercise

- MSI read up
- get some examples which uses tasklet and workqueue for  process
- how you monitor performance of the above?







# Kernel Synchronization Methods





##### Atomic

Atomic data types and operations will safeguard updates and reads inside the kernel.



##### Spinlock

Simple synchronization which lock the access for single process who acquires the lock and all other process keeps spinning around the lock untill it gets released.

- It can not be blocked for long time because spinning processes consumes CPU.
- this mechanism is much cheaper to implement



##### Semaphore

A semaphore is like waiting queue attached to a spin lock where processes will be waiting on the queue instead of spinning.

Semaphore also allows multiple threads to acquire a lock.



Exercise

- what is this IPCS based semaphore, I guess they are different than kernel space.



##### Mutex

Mutex is the semaphore with only one thread/process allowing to acquire lock on the resource.

- Mutex has to be released in same context
- this does not allow recursive locking, that means same thread can not lock on same mutex again.
- look the following where mutex is based on spin lock



 linux/kernel/locking/mutex.c

```C
void __sched mutex_unlock(struct mutex *lock) {
	#ifndef CONFIG_DEBUG_LOCK_ALLOC
	if (__mutex_unlock_fast(lock)) return;
	#endif
__mutex_unlock_slowpath(lock, _RET_IP_);
}
```

```C
static noinline void __sched __mutex_unlock_slowpath( struct mutex *lock , unsigned long ip)
{
	struct task_struct *next = NULL; DEFINE_WAKE_Q(wake_q);
	unsigned long owner;
	mutex_release(&lock->dep_map, 1, ip);
/*
* Release the lock before (potentially) taking the spinlock such that * other contenders can get on with things ASAP.
*
* Except when HANDOFF, in that case we must not clear the owner field, * but instead set it to the top waiter.
*/

owner = atomic_long_read(&lock->owner);

  for (;;) {
		unsigned long old;
		#ifdef CONFIG_DEBUG_MUTEXES
    DEBUG_LOCKS_WARN_ON(__owner_task(owner) != current); 		        
    DEBUG_LOCKS_WARN_ON(owner & MUTEX_FLAG_PICKUP);
    #endif
		if (owner & MUTEX_FLAG_HANDOFF) break;
		old = atomic_long_cmpxchg_release(&lock->owner, owner, 		
                                      __owner_flags(owner));
		if (old == owner) {
			if (owner & MUTEX_FLAG_WAITERS)
				break;

      return;
		}
		owner = old;
  }

  spin_lock(&lock->wait_lock);
  debug_mutex_unlock(lock);
	if (!list_empty(&lock->wait_list)) {
		/* get the first entry from the wait-list: */
    struct mutex_waiter *waiter = list_first_entry(&lock->wait_list,
				struct mutex_waiter , list);

    next = waiter ->task;

    debug_mutex_wake_waiter(lock, waiter);
    wake_q_add(&wake_q, next);
  }

  if (owner & MUTEX_FLAG_HANDOFF)
    __mutex_handoff(lock, next);

  spin_unlock(&lock->wait_lock);

  wake_up_q(&wake_q);
}
```



##### Others

- There are other special locks like Read/Write lock where two activities are separated out for locking.

- There is something called RCU which is another synchronization mechanism in place.

- There is `sequential lock` which is like semaphore with the ordered queue.

- There usued to be `Big Kernel lock` which has been phased out.

- `Completion variable` is a simple solution to use of the semaphore where thread will wait to let complete variable value updates.

- Interrupt disabling in IRQ context and bottom haves

  ```C
  spin_lock_irqsave(&lock, flags);
  spin_unlock_irqrestore(&lock, flags);
  
  spin_lock_bh ();
  spin_unlock_bh ();
  ```

- Preemption disabling - this is available , how?



# System calls

Regarding implementatoin of the system calls (weekend)

https://blog.packagecloud.io/eng/2016/04/05/the-definitive-guide-to-linux-system-calls/

Dump VDSO and play with it

https://stackoverflow.com/questions/17157820/access-vdsolinux







#### Exercise

- go through coreutils
- how [linking](https://www.gnu.org/software/libc/manual/html_node/Auxiliary-Vector.html) of the vDSO happens in linux
- The address of the `__kernel_vsyscall` function is written into an [ELF auxilliary vector](https://www.gnu.org/software/libc/manual/html_node/Auxiliary-Vector.html) where a user program or library (typically `glibc`) can find it and use it.
- how linking of libraries happen in general




# Timer



#### Exercise

- how sleep functionality is implemneted using timer.


  =======

# Timer

Some information about timer https://blog.packagecloud.io/eng/2017/03/08/system-calls-are-much-slower-on-ec2/

https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-1.html









```shell
$ dmesg | grep "clocksource:"
[    0.000000] clocksource: refined-jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645519600211568 ns
[    0.000000] clocksource: hpet: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 79635855245 ns
[    0.832521] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    2.236031] clocksource: Switched to clocksource hpet
[    2.451204] clocksource: acpi_pm: mask: 0xffffff max_cycles: 0xffffff, max_idle_ns: 2085701024 ns
[    4.550832] clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0x24093d6e846, max_idle_ns: 440795249997 ns
[    5.564342] clocksource: Switched to clocksource tsc
```





kthread lauched by `rest_init()`

```shell
$ ps ax |grep kthreadd | grep -v grep
    2 ?        S      0:00 [kthreadd]
```





# Booting





#### BIOS and pre-BIOS

- Computer wakes up 

- wakes up CPU 

- default values in CPU registers 

- kicks in real mode 

- 20 bit address bus has capability to address 1MB of memory but CPU registers are 16bit then it has ability address 64KB at once. So 1MB memory divided into 64KB segments. those will be 16 segments.

- Physical address (20bit) = Segment selection * 16 + offset.  (CS:IP)

- Maximum theorotical addressable memory in this setting will be 64KB above 1MB. 

  ```assembly
  >>> hex((0xffff << 4) + 0xffff)
  '0x10ffef'  <- 1MB + 64KB - 16bytes
  ```

  

- In practise it becomes `0x00ffef` with the [A20 line](https://en.wikipedia.org/wiki/A20_line) disabled.

- With all the initial values in `CS` there is the address `0xfffffff0`, which is 16 bytes below 4GB called reset vector position.

- Reset vector where CPU finds its first instruction to execute which points to BIOS.

- BIOS selects boot device then continue booting 

- worth looking at  [coreboot](https://www.coreboot.org/Developer_Manual/Memory_map) documentation where BIOS ROM get mapped to address location.

  ```
  0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
  ```

- after this BIOS selects boot device which loads boot loader

#### Boot loader

- boot loader split between MBR and first sector  of the booting device
- Boot loader fill in some headers in the linker code and loads kernel boot sector market with magic header `MZ`
  - Jump to this magic address which know as `start_of_setup`
    - Make sure that all segment register values are equal
    - Set up a correct stack, if needed
    - Set up [bss](https://en.wikipedia.org/wiki/.bss)
    - Jump to the C code `main()` in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)

#### Protected mode (before long mode)

- protected mode moved from 20 bit addressing (1MB) to 32 bit addressing (4GB)
- this mode also supports segmentation and paging. Previously memory segments were 64KB size but in protected mode they are different. 
- In protected mode size and location of each segment descriebd by data structure `Segment Descriptor` which is 64-bit in size. These segment descritors are stored in a data structure called the `Global Descriptor Table (GDT)`.
- GDT addressed stored in CPU 48 bit register called GDTR and can be retrived by `lgdt gdt` instruction. 
- ![img](SRE/assets/gdt_and_ldt_mapping.png)

Index can come from either GDT or LDT. GDT is to store kernel related addresses and LDT is created for each process. 

GDT or LDT plus offset will give you physical location of the memory.
**Logical Address = Segment Selector (16 bit) + Offset (13 bit)**



![img](SRE/assets/segment_selector.png)

checkout Privillage level in RPL

 The following steps are needed to get a physical address in protected mode:

- The segment selector must be loaded in one of the segment registers.

- The CPU tries to find a segment descriptor at the offset `GDT address + Index` from the selector and then loads the descriptor into the *hidden* part of the segment register.

- If paging is disabled, the linear address of the segment, or its physical address, is given by the formula: Base address (found in the descriptor obtained in the previous step) + Offset.

- In short, it **initializes GDT** because kernel needs to start addressing memory.

- `boot_params` structure gets populated with `copy_boot_params(void)` in `main.c` , reading them from location where boot loader has stored them.

- `biosregs` structure gets populated and **serial console gets initialized**. Interrupt 0x10 gets called to print very first charater and later serial console method get used. tty.C

- **Heap gets initalized.** 

- Checking CPU for its flags and **presense for a long mode**. BIOS 0x15 interrupt used to inform BIOS about switch into that mode.  CPU validation through the `validate_cpu` function from [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/cpu.c) source code file.

- **BIOS  0xE820 memory reporting** to the operating system using interrupt 0x15. We can see that as first line in dmesg.Ultimately, this function collects data from the address allocation table and writes this data into the `e820_entry` array:

  - start of memory segment
  - size of memory segment
  - type of memory segment (whether the particular segment is usable or reserved)

- **Initialization of the keyboard** with a call to the [`keyboard_init`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) calls 0x16 interrupt.

- Query various BIOS functions like APM and Disk drives.

- Setup a video mode.  `vid_mode=ask` in grub which usually set to vga or vesa.

- Transition to **protected mode**.

  - function call - `go_to_protected_mode` - in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). 
  - it disables NMI, try to enable A20, resets match processor, masks all programmable interrupt controllers except IRQ2 (i think its a keyboard)
  - Sets up Interrupt Discriptor Table. `setup_idt` 
  - Sets up GDT `setup_gdt`
  - jump with  `protected_mode_jump`  in [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pmjump.S) to enable  `PE` (Protection Enable) bit in the `CR0` control register:

  

#### Journey towards long mode - 64 bit

- [x86 boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) 

- `bzimage` is a gzipped package consisting of `vmlinux`, `header` and `kernel setup code` and job of `kernel setup code` is to prepare to enter into a long mode with `vmlinux` execution after its decompression. So far we have been executing `kernel setup code`.

- [head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) will decompress the kernel and enter the long mode. 

- **Setup a stack space** and verification of CPU for long mode support.  Push `startup_64` function to the stack which then CPU extracts from the stack while shifting to long mode. 

- Move processor into long mode or straight in 64 bit mode

  - 8 new general purpose registers from `r8` to `r15`
  - All general purpose registers are 64-bit now
  - A 64-bit instruction pointer - `RIP`
  - A new operating mode - Long mode;
  - 64-Bit Addresses and Operands;
  - RIP Relative Addressing (we will see an example of this in the coming parts).
  - Enable [PAE](https://en.wikipedia.org/wiki/Physical_Address_Extension);
  - Build page tables and load the address of the top level page table into the `cr3` register
  - Enable `EFER.LME` in the MSR to  `0xC0000080`
  - Enable paging ( just for 4G memory )

- Linux addresses – Virtual address to linear address to physical address
  ![img](SRE/assets/virtual_linear_physical_addresses.png)

  - ***Virtual addresses\*** are used by an application program. They consist of a 16-bit selector and a 32-bit offset. In the flat memory model, the selectors are preloaded into segment registers CS, DS, SS, and ES, which all refer to the same linear address. They need not be considered by the application. Addresses are simply 32-bit near pointers.
  - ***Linear addresses\*** are calculated from virtual addresses by segment translation. The base of the segment referred to by the selector is added to the virtual offset, giving a 32-bit linear address. Under RTTarget-32, virtual offsets are equal to linear addresses since the base of all code and data segments is 0.
  - ***Physical addresses\*** are calculated from linear addresses through paging. The linear address is used as an index into the Page Table where the CPU locates the corresponding physical address.

  

- The Linux kernel uses `4-level` paging, and we generally **build 6 page tables**:

  - One `PML4` or `Page Map Level 4` table with one entry;
  - One `PDP` or `Page Directory Pointer` table with four entries;
  - Four Page Directory tables with a total of `2048` entries.

- Enable paging and execute `lret` instuction which will start executing  `startup_64` function from the stack. 

- That should put system into 64 bit more which is a native mode for x86_64.



#### Decompress the kernel

-  from the `64-bit` entry point - `startup_64` which is located in the [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) 

- The `extract_kernel` function is defined in the [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/misc.c) source code file and takes six arguments:

  - `rmode` - a pointer to the [boot_params](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h) structure which is filled by either the bootloader or during early kernel initialization;
  - `heap` - a pointer to `boot_heap` which represents the start address of the early boot heap;
  - `input_data` - a pointer to the start of the compressed kernel or in other words, a pointer to the `arch/x86/boot/compressed/vmlinux.bin.bz2` file;
  - `input_len` - the size of the compressed kernel;
  - `output` - the start address of the decompressed kernel;
  - `output_len` - the size of the decompressed kernel;

  All arguments will be passed through registers as per the [System V Application Binary Interface](http://www.x86-64.org/documentation/abi.pdf). We've finished all the preparations and can now decompress the kernel.

-  [boot_params](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h) structure holds arguments which are passed to kernel image executable after its decompression.

- After we initialize the heap pointers, the next step is to call the `choose_random_location` function from the [arch/x86/boot/compressed/kaslr.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/kaslr.c) source code file. kASLR  which allows decompression of the kernel into a random address, for security reasons..

-  `Decompressing Linux...`

- Kernel is an ELF executable.

- sections of linux kernel 

  ```  
   readelf -l vmlinux
  
  Elf file type is EXEC (Executable file)
  Entry point 0x1000000
  There are 5 program headers, starting at offset 64
  
  Program Headers:
    Type           Offset             VirtAddr           PhysAddr
                   FileSiz            MemSiz              Flags  Align
    LOAD           0x0000000000200000 0xffffffff81000000 0x0000000001000000
                   0x000000000144c000 0x000000000144c000  R E    0x200000
    LOAD           0x0000000001800000 0xffffffff82600000 0x0000000002600000
                   0x0000000000257000 0x0000000000257000  RW     0x200000
    LOAD           0x0000000001c00000 0x0000000000000000 0x0000000002857000
                   0x000000000002d000 0x000000000002d000  RW     0x200000
    LOAD           0x0000000001c84000 0xffffffff82884000 0x0000000002884000
                   0x0000000000d7c000 0x0000000000d7c000  RWE    0x200000
    NOTE           0x0000000001000e84 0xffffffff81e00e84 0x0000000001e00e84
                   0x00000000000001ec 0x00000000000001ec         0x4
  
   Section to Segment mapping:
    Segment Sections...
     00     .text .notes __ex_table .rodata .pci_fixup .tracedata __ksymtab __ksymtab_gpl __ksymtab_strings __init_rodata __param __modver
     01     .data __bug_table .vvar
     02     .data..percpu
     03     .init.text .altinstr_aux .init.data .x86_cpu_dev.init .parainstructions .altinstructions .altinstr_replacement .iommu_table .apicdrivers .exit.text .smp_locks .data_nosave .bss .brk .init.scratch
     04     .notes
  ```

These segments of elf binaries will get loaded.

After the kernel is relocated, we return from the `extract_kernel` function to [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S).

The address of the kernel will be in the `rax` register and we jump to it:

```assembly
jmp    *%rax
```



#### Booting

- Initialize memory map for memory above 4G. `initialize_identity_maps`
- choose andom memory location to extract kernel without abstructing memory occupied by `initrd`.



So far we are doing just a preparation for execution of the first Kernel code.

That's all. Now we are in the kernel! Executing..... Booting....



# Kernel Initialization

This is kernel's first byte execution onwards 



#### Before `start_kernel` execution

- Allocate address for paging - FIX base address for all tables.
- Enable [SME](https://en.wikipedia.org/wiki/Zen_(microarchitecture)#Enhanced_security_and_virtualization_support) 
-  `__START_KERNEL_map` is a base virtual address of the kernel text, so if we subtract `__START_KERNEL_map`, we will get physical addresses of the `level2_kernel_pgt` and `level2_fixmap_pgt`.
- `*_kernel_pgt` pointing to regualr pages and  `*_fixmap_pgt` will point to VDSO syscalls mappings.
- There are something called `early_dynamic_pgts` which holds pages for initial kernel execution.
- read MSR for CPU features like [NX](https://en.wikipedia.org/wiki/NX_bit) support availble then write MSR.
- Also update bit in control register 
  - `X86_CR0_PE` - system is in protected mode;
  - `X86_CR0_MP` - controls interaction of WAIT/FWAIT instructions with TS flag in CR0;
  - `X86_CR0_ET` - on the 386, it allowed to specify whether the external math coprocessor was an 80287 or 80387;
  - `X86_CR0_NE` - enable internal x87 floating point error reporting when set, else enables PC style x87 error detection;
  - `X86_CR0_WP` - when set, the CPU can't write to read-only pages when privilege level is 0;
  - `X86_CR0_AM` - alignment check enabled if AM set, AC flag (in EFLAGS register) set, and privilege level is 3;
  - `X86_CR0_PG` - enable paging.
- Initialize thread structure which is required to start a process with `thread_info` which for x86 becomes a part of `task_struct` structure .
- Very first thread gets initialized by `INIT_TASK_DATA(THREAD_SIZE)` in [arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S) and this macro is defined in [include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/blob/master/include/asm-generic/vmlinux.lds.h)
- Reload GDT 
- Create GDT page for `per-cpu` variable which gets created with `INIT_PER_CPU_VAR` , You can read in details about `per-CPU` variables in the [Concepts/per-cpu](https://0xax.gitbook.io/linux-insides/summary/concepts/linux-cpu-1) post.
- Initialize IRQ stack with its handlers with MSRs.
- jump to execution of `x86_64_start_kernel` through`initial_code` pointer.



#### `x86_64_start_kernel`  from the [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c).





#### Early interrupt and exception handling

Total interrupt vector numbers `0-255` , multiplying vector with 16 will give you an IDT entry for that vector.  CPU uses special register `IDTR` for Interrupt Descriptor Table and `lidt` instruction for loading base address of the table into this register.

Type of interrupts 

- Software interrupts - when a software signals CPU that it needs kernel attention. These interrupts are generally used for system calls;
- Hardware interrupts - when a hardware event happens, for example button is pressed on a keyboard;
- Exceptions - interrupts generated by CPU, when the CPU detects error, for example division by zero or accessing a memory page which is not in RAM. `0-31` are assigned for this exception handling.



Following table shows `0-31`exceptions:

```
----------------------------------------------------------------------------------------------
|Vector|Mnemonic|Description         |Type |Error Code|Source                   |
----------------------------------------------------------------------------------------------
|0     | #DE    |Divide Error        |Fault|NO        |DIV and IDIV                          |
|---------------------------------------------------------------------------------------------
|1     | #DB    |Reserved            |F/T  |NO        |                                      |
|---------------------------------------------------------------------------------------------
|2     | ---    |NMI                 |INT  |NO        |external NMI                          |
|---------------------------------------------------------------------------------------------
|3     | #BP    |Breakpoint          |Trap |NO        |INT 3                                 |
|---------------------------------------------------------------------------------------------
|4     | #OF    |Overflow            |Trap |NO        |INTO  instruction                     |
|---------------------------------------------------------------------------------------------
|5     | #BR    |Bound Range Exceeded|Fault|NO        |BOUND instruction                     |
|---------------------------------------------------------------------------------------------
|6     | #UD    |Invalid Opcode      |Fault|NO        |UD2 instruction                       |
|---------------------------------------------------------------------------------------------
|7     | #NM    |Device Not Available|Fault|NO        |Floating point or [F]WAIT             |
|---------------------------------------------------------------------------------------------
|8     | #DF    |Double Fault        |Abort|YES       |An instruction which can generate NMI |
|---------------------------------------------------------------------------------------------
|9     | ---    |Reserved            |Fault|NO        |                                      |
|---------------------------------------------------------------------------------------------
|10    | #TS    |Invalid TSS         |Fault|YES       |Task switch or TSS access             |
|---------------------------------------------------------------------------------------------
|11    | #NP    |Segment Not Present |Fault|NO        |Accessing segment register            |
|---------------------------------------------------------------------------------------------
|12    | #SS    |Stack-Segment Fault |Fault|YES       |Stack operations                      |
|---------------------------------------------------------------------------------------------
|13    | #GP    |General Protection  |Fault|YES       |Memory reference                      |
|---------------------------------------------------------------------------------------------
|14    | #PF    |Page fault          |Fault|YES       |Memory reference                      |
|---------------------------------------------------------------------------------------------
|15    | ---    |Reserved            |     |NO        |                                      |
|---------------------------------------------------------------------------------------------
|16    | #MF    |x87 FPU fp error    |Fault|NO        |Floating point or [F]Wait             |
|---------------------------------------------------------------------------------------------
|17    | #AC    |Alignment Check     |Fault|YES       |Data reference                        |
|---------------------------------------------------------------------------------------------
|18    | #MC    |Machine Check       |Abort|NO        |                                      |
|---------------------------------------------------------------------------------------------
|19    | #XM    |SIMD fp exception   |Fault|NO        |SSE[2,3] instructions                 |
|---------------------------------------------------------------------------------------------
|20    | #VE    |Virtualization exc. |Fault|NO        |EPT violations                        |
|---------------------------------------------------------------------------------------------
|21-31 | ---    |Reserved            |INT  |NO        |External interrupts                   |
----------------------------------------------------------------------------------------------
```



64-bit mode IDT entry has following structure:

```
127                                                                             96
 --------------------------------------------------------------------------------
|                                                                               |
|                                Reserved                                       |
|                                                                               |
 --------------------------------------------------------------------------------
95                                                                              64
 --------------------------------------------------------------------------------
|                                                                               |
|                               Offset 63..32                                   |
|                                                                               |
 --------------------------------------------------------------------------------
63                               48 47      46  44   42    39             34    32
 --------------------------------------------------------------------------------
|                                  |       |  D  |   |     |      |   |   |     |
|       Offset 31..16              |   P   |  P  | 0 |Type |0 0 0 | 0 | 0 | IST |
|                                  |       |  L  |   |     |      |   |   |     |
 --------------------------------------------------------------------------------
31                                   16 15                                      0
 --------------------------------------------------------------------------------
|                                      |                                        |
|          Segment Selector            |                 Offset 15..0           |
|                                      |                                        |
 --------------------------------------------------------------------------------
```

Where:

- `Offset` - is offset to entry point of an interrupt handler;
- `DPL` - Descriptor Privilege Level;
- `P` - Segment Present flag;
- `Segment selector` - a code segment selector in GDT or LDT
- `IST` - provides ability to switch to a new stack for interrupts handling.

And the last `Type` field describes type of the `IDT` entry. There are three different kinds of gates for interrupts:

- Task gate
- Interrupt gate
- Trap gate

Interrupt and trap gates contain a far pointer to the entry point of the interrupt handler. Interrupt gates clears `IF` bit above to disables all the interrupt during its execution and set it back on upon completion of execution.



`    idt_setup_early_handler()` in [arch/x86/kernel/idt.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/idt.c)  populates exception handling interrupt routines in `early_idt_handler_array`.

After that we push `vector number` on the stack and jump on the `early_idt_handler_common` which is generic interrupt handler for now. After all, every nine bytes of the `early_idt_handler_array` array consists of optional push of an error code, push of `vector number` and jump instruction to `early_idt_handler_common`.



you can see refereces to above functionality in the `objdump -D vmlinux`



- Most common exception handled in this stage is page fault with   in [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c)

- For more information about exception table, you can refer to [Documentation/x86/exception-tables.txt](https://github.com/torvalds/linux/blob/master/Documentation/x86/exception-tables.txt).

- Traps are handled  by  `fixup_bug` function is defined in [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c).

  

#### Retriving boot params and other 

- we stored boot params of the kernel during real mode which will get retrieved with `copy_bootdata(__va(real_mode_data))`
- reserve memory for Extended BIOS Data Area located in the top of conventional memory and contains data about ports, disk parameters and etc...
- call the `start_kernel`

All above steps done by following 

```C
void __init x86_64_start_reservations(char *real_mode_data)
{
    if (!boot_params.hdr.version)
        copy_bootdata(__va(real_mode_data));

    reserve_ebda_region();

    start_kernel();
}
```





# Kernel entry point `start_kernel`

The main purpose of the `start_kernel` is to launch the first `init` process and do following initilization to do that launch 

-  to enable [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt), 
- to initialize processor id, 
- to enable early [cgroups](http://en.wikipedia.org/wiki/Cgroups) subsystem, 
- to setup per-cpu areas, 
- to initialize different caches in [vfs](http://en.wikipedia.org/wiki/Virtual_file_system), 
- to initialize memory manager, rcu, vmalloc, scheduler, IRQs, ACPI and many many more.



Begning of `start_kernel` you get kernel command lines and parsed arguments (result of the `parse_args` function). 





Very first canary task is getting defined in the form of `tast_struct`  by following

```
struct task_struct init_task = INIT_TASK(init_task);
```

and `set_task_stack_end_magic` is the function ( defined in the [kernel/fork.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/fork.c#L297) ) which marks the end of the stack of this task. This canary stack is special so we have to call this function to avoid stack overflow for this task.

`task_struct` defination can be found  in [include/linux/sched.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/sched.h#L1278)

You can see the definition of the `INIT_TASK` macro in [include/linux/init_task.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init_task.h) 

`thread_info` structure for this canary task written manually at the bottom of its stack. Process stack in linux is 4 memory pages long which is 4 * 4KB = 16KB. `thread_info` structure size is 62 bytes and that is how its location gets calculated. Note that `thread_union` represented as the [union](http://en.wikipedia.org/wiki/Union_type) and not structure, it means that `thread_info` and stack share the memory space ([why?](http://www.quora.com/In-Linux-kernel-Why-thread_info-structure-and-the-kernel-stack-of-a-process-binds-in-union-construct) ). 



we fill the `stack_canary` field of `task_struct` with random value collected from  [TSC](http://en.wikipedia.org/wiki/Time_Stamp_Counter) and write it on the top of the IRQ stack (I DO NOT know why?). Now time for processor activation but before that it disables all the local IRQs for the current CPU and calls  `boot_cpu_init`.





 `boot_cpu_init` initializes various CPU masks for the bootstrap processor which is running our cannary task by setting up its CPU variables and setting up bitmap mask.



The [per-cpu](https://0xax.gitbook.io/linux-insides/summary/concepts/linux-cpu-1) variables manupulation done by `this_cpu` macros which are defined in defined in the [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/percpu-defs.h). More on CPU bit mask at  [cpumask](https://0xax.gitbook.io/linux-insides/summary/concepts/linux-cpu-2) or [documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt).





Print a banner 

```C
#define pr_notice(fmt, ...) \
    printk(KERN_NOTICE pr_fmt(fmt), ##__VA_ARGS__)


/* 
Linux version 4.0.0-rc6+ (alex@localhost) (gcc version 4.9.1 (Ubuntu 4.9.1-16ubuntu6) ) #319 SMP
*/
```





Reserve memory for the kernel  `_text` and `_data` segments. This is done by `memblock_reserve` function called with address params like 

```C
memblock_reserve(__pa_symbol(_text), (unsigned long)__bss_stop - (unsigned long)_text);
```



Then reserves the memory for the ramdisk image whose location can be fetched by reading `boot_params` with following function 

```C
static u64 __init get_ramdisk_image(void)
{
        u64 ramdisk_image = boot_params.hdr.ramdisk_image;

        ramdisk_image |= (u64)boot_params.ext_ramdisk_image << 32;

        return ramdisk_image;
}
```



Why  `boot_params` and shift left it on `32` ?  [Documentation/x86/zero-page.txt](https://github.com/0xAX/linux/blob/0a07b238e5f488b459b6113a62e06b6aab017f71/Documentation/x86/zero-page.txt)



Reserve memory for ramdisk with following 

```C
memblock_reserve(ramdisk_image, ramdisk_end - ramdisk_image);
```



above two steps happens in the begining of the `setup_arch` function which starts `architecture-specific` bootstrapping defined under `arch/` directory.  We are executing  `setup_arch` function defined in the [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c)



# architecture-specific initialization

We are entring into  `setup_arch`  which is called by `start_kernel`

#### Debug and exception handling

- setup breakpoints - ability to debug kernel and exception handlers. In x86 architecture `INT` , `INTO` and

  `INT3` are special instructions which allow a task to explicitly call an interrupt handler. The `INT3` instruction calls the breakpoint ( `#BP` ) handler.
   `early_trap_init` defined in the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c). This functions sets `#DB` and `#BP` handlers and reloads [IDT](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)

- In addition to per-thread stacks, there are a couple of specialized stacks associated with each CPU. [Kernel stacks](https://www.kernel.org/doc/Documentation/x86/kernel-stacks)   `x86_64` provides feature which allows to switch to a new `special` stack for during any events as non-maskable interrupt and etc... And the name of this feature is - `Interrupt Stack Table`. These are useful for exception handling and debugging

- Implementation of the `#DB` handler as other handlers is in this [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) and defined with the `idtentry` assembly macro `idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK` 

- During interrupt handling all CPU registers are backed up on the stack and that take 120 bytes of space.

#### ioremap initialization 

This is where initialization of the pages for IO memory mapped and port based communication.    in the [arch/x86/mm/ioremap.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/mm/ioremap.c) . Memory map setup with the call of the `setup_memory_map` function. 

* /proc/ioports - provides a list of currently registered port regions used for input or output communication with a device;
* /proc/iomem   - provides current map of the system's memory for each physical device.

Entire hirarchey of devices captured in the tree like structure where parent has siblings and children devices.

**NOTE**: `EXPORT_SYMBOL` macro exports the given symbol for dynamic linking or in other words it makes a symbol accessible to dynamically loaded modules.

#### obtain major and minor for root device (device for ramdisk to mount)

```
ROOT_DEV = old_decode_dev(boot_params.hdr.root_dev);
```



#### Setup memory map

 call of the `setup_ memory_map` function and kernel message like following 

```C
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009d7ff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009d800-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000e0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x00000000be825fff] usable
[    0.000000] BIOS-e820: [mem 0x00000000be826000-0x00000000be82cfff] ACPI NVS
...
...
```

initialization of the `x86_init` is in the [arch/x86/kernel/x86_init.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/x86_init.c) which will hold information to print above message.



#### parsing BIOS Enhanced Disk Device information

Parsing of the `setup_data` with `parse_setup_data` function and copying BIOS EDD to the safe place. 



#### Memory discriptor for kernel

Every process has its own memory structure (`struct mm_struct`  in the  [include/linux/mm_types.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/mm_types.h)) attached to its own task structure (`struct task_struct`). `mm` points to the process address space and `active_mm` points to the active address space

If process with the no address space that means it is a kernel thread  (more about it you can read in the [documentation](https://www.kernel.org/doc/Documentation/vm/active_mm.txt)).

Here  kernel's text, data and brk gets initiated.



#### NX bit

 `x86_configure_nx` function which sets the `_PAGE_NX` flag depends on support of [NX bit](http://en.wikipedia.org/wiki/NX_bit).

#### parsing of the kernel params

 `parse_early_param` which internally calls  `early_param` which calls ` __setup_param` which eventually uses `struct obs_kernel_param` pointers to do relevent parsing for particular params. This functionality provides code the ability to conduct various kernel related tasks 

After this, quick call to

-  `x86_report_nx`
- `memblock_x86_reserve_range_setup_data` to do some more memory remmaping 
- collect multi processor specifications with  `acpi_mps_check` function from the [arch/x86/kernel/acpi/boot.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/acpi/boot.c)  It checks the built-in `MPS` or [MultiProcessor Specification](http://en.wikipedia.org/wiki/MultiProcessor_Specification) table.
- `pcibios_setup`  from the [drivers/pci/pci.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch) for early scan of these devices.
- and  `e820_reserve_setup_data` for E820 memory area sanitization.



#### DMI scanning -  [Desktop Management Interface](http://en.wikipedia.org/wiki/Desktop_Management_Interface) 

```C
dmi_scan_machine();
dmi_memdev_walk();
```

defined in the [drivers/firmware/dmi_scan.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/firmware/dmi_scan.c)

`dmi_scan_machine` function goes through the [System Management BIOS](http://en.wikipedia.org/wiki/System_Management_BIOS) structures and extracts information.

 `dmi_memdev_walk` goes over memory devices mapped area.

#### SMP

 Parsing of the  configuration with  `find_smp_config` function.  [MultiProcessor Specification](http://www.intel.com/design/pentium/datashts/24201606.pdf)



#### Some more memory shuffling 

-  call of the `early_alloc_pgt_buf` function which allocates the page table buffer for early stage. 
-  `reserve_real_mode` - reserves low memory from `0x0` to 1 megabyte for the trampoline to the real mode.
-  `setup_real_mode` function setups trampoline to the [real mode](http://en.wikipedia.org/wiki/Real_mode) code.

 

#### Kernel log buffer

The `setup_log_buf` function setups kernel cyclic buffer, call the `log_buf_add_cpu` function which increase size of the buffer for every CPU



#### initrd memory space reserve

print information about the `initrd` size. We can see the result of this in the `dmesg` output:

```C
[0.000000] RAMDISK: [mem 0x36d20000-0x37687fff]
```

and relocate `initrd` to the direct mapping area with the `relocate_initrd` function.  



#### io_delay_init

 `io_delay_init` from the [arch/x86/kernel/io_delay.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/io_delay.c). This function allows to override default I/O delay `0x80` port.



#### Allocate area for DMA

with the `dma_contiguous_reserve` function which is defined in the [drivers/base/dma-contiguous.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/base/dma-contiguous.c).  DMA needs CMA aka contiguous memory allocations.



#### Initialization of the sparse memory (NUMA aware paging)

The `Sparsemem` is a special foundation in the linux kernel memory manager which used to split memory area into different memory banks in the [NUMA](http://en.wikipedia.org/wiki/Non-uniform_memory_access) systems. Let's look on the implementation of the `paginig_init` function



Every `NUMA` node is divided into a number of pieces which are called - `zones`. So, `zone_sizes_init` function from the [arch/x86/mm/init.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/mm/init.c) initializes size of zones.



#### vsyscall

Setting of the `trampoline_cr4_features` which must contain content of the `cr4` [Control register](http://en.wikipedia.org/wiki/Control_register) . This CR4 register is going to use in the execution of vsyscalls.

 `map_vsyscal` from the [arch/x86/kernel/vsyscall_64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vsyscall_64.c). This function maps memory space for [vsyscalls](https://lwn.net/Articles/446528/)

We can see definition of the `__vsyscall_page` in the [arch/x86/kernel/vsyscall_emu_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vsyscall_emu_64.S).  The `__vsyscall_page` symbol points to the aligned calls of the `vsyscalls` as `gettimeofday`, etc.



Other steps

-  `mcheck_init` function initializes `Machine check Exception` 
-  `register_refined_jiffies` which registers [jiffy](http://en.wikipedia.org/wiki/Jiffy_%28time%29) 





# Back to `start_kernel`

- call of the `mm_init_cpumask` function. This function sets the [cpumask](https://0xax.gitbook.io/linux-insides/summary/concepts/linux-cpu-2) pointer to the memory descriptor `cpumask`.
- 



  









