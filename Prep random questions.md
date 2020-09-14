# Prep random questions





- why `buffer_head` and `dentry` are on top of the output of slabtop? this represents highly cached memory usually due to log files.

- `sudo ss -m --info` will provide you memory consumption of the socket.
- TLB usage here the **P** is divided in x86 architecture into multiple levels of PGD (page global dir), PUD (page upper dir), PMD (page middle dir) and lowest leve in PTE. This **PTE** is mapped to **Frame number** in TLB, if it is missed then it have to crawl through entire directory structure.  TLB flushing is initiated by OS 

![Translation_Lookaside_Buffer](https://upload.wikimedia.org/wikipedia/commons/6/6e/Translation_Lookaside_Buffer.png)

- TLB flushing can be avoided by reducing size of the page table (especially PGD) entries by using huge pages.

- /proc/<PID>/numa_maps will provide you output like following which provides summary of which page mapped on which numa node.

  ```
  f9a58021000 default
  7f9a5c006000 default anon=632 dirty=632 N0=419 N1=213 kernelpagesize_kB=4
  7f9a5c286000 default
  7f9a5c287000 default anon=65 dirty=65 N0=46 N1=19 kernelpagesize_kB=4
  7f9a5cac7000 default
  7f9a5cac8000 default anon=130 dirty=130 N0=100 N1=30 kernelpagesize_kB=4
  7f9a5d348000 default
  7f9a5d349000 default anon=2 dirty=2 N0=2 kernelpagesize_kB=4
  7f9a5db49000 default
  7f9a5db4a000 default anon=2 dirty=2 N0=2 kernelpagesize_kB=4
  7f9a5e34a000 default
  7f9a5e34b000 default anon=2 dirty=2 N0=1 N1=1 kernelpagesize_kB=4
  7f9a5eb4b000 default
  7f9a5eb4c000 default anon=2 dirty=2 N1=2 kernelpagesize_kB=4
  7f9a5f34c000 default
  7f9a5f34d000 default anon=789 dirty=789 N0=481 N1=308 kernelpagesize_kB=4
  7f9a61e5e000 default
  7f9a71fde000 default anon=25 dirty=25 N0=19 N1=6 kernelpagesize_kB=4
  7f9a71ff7000 default
  7f9a83e8e000 default anon=4 dirty=4 N0=1 N1=3 kernelpagesize_kB=4
  7f9a83e92000 default
  7f9a86264000 default anon=1 dirty=1 N0=1 kernelpagesize_kB=4
  7f9a86265000 default
  7f9a8665e000 default file=/lib/x86_64-linux-gnu/libc-2.23.so mapped=242 mapmax=76 N0=242 kernelpagesize_kB=4
  7f9a8681e000 default file=/lib/x86_64-linux-gnu/libc-2.23.so
  7f9a86a1e000 default file=/lib/x86_64-linux-gnu/libc-2.23.so anon=4 dirty=4 N0=4 kernelpagesize_kB=4
  7f9a86a22000 default file=/lib/x86_64-linux-gnu/libc-2.23.so anon=2 dirty=2 N0=2 kernelpagesize_kB=4
  7f9a86a24000 default anon=3 dirty=3 N0=3 kernelpagesize_kB=4
  7f9a86a28000 default file=/lib/x86_64-linux-gnu/libpthread-2.23.so mapped=24 mapmax=61 N0=24 kernelpagesize_kB=4
  7f9a86a40000 default file=/lib/x86_64-linux-gnu/libpthread-2.23.so
  7f9a86c3f000 default file=/lib/x86_64-linux-gnu/libpthread-2.23.so anon=1 dirty=1 N0=1 kernelpagesize_kB=4
  7f9a86c40000 default file=/lib/x86_64-linux-gnu/libpthread-2.23.so anon=1 dirty=1 N0=1 kernelpagesize_kB=4
  7f9a86c41000 default anon=1 dirty=1 N0=1 kernelpagesize_kB=4
  7f9a86c45000 default file=/lib/x86_64-linux-gnu/ld-2.23.so mapped=38 mapmax=74 N0=38 kernelpagesize_kB=4
  7f9a86c6c000 default anon=144 dirty=144 N0=82 N1=62 kernelpagesize_kB=4
  7f9a86cfc000 default
  7f9a86d7c000 default anon=1 dirty=1 N0=1 kernelpagesize_kB=4
  ```

  - Command to know the state of memory 

    - `sar -R`  will give you idea of rate at which cache pages and free pages are addressed
    - `sar -r ALL`   will give you all the information about regualr memory statstics like active/inactive slab/annon/stack pages. 
    - `sar -S` will give you swap related activities. one of the interesting metrics `kbswpcad` which indicates amount of pages which are both in cache and it swap.
    - `sar -B` will give you infromation on state of page daemon activity 
      - pgscank/s - Kswapd scan which only happens if free list shrinks to certain level
      - pgscand/s  - blocking application thread to allocate required memory 
      - pgsteal/s - stealing page from the page cache itself as free memory is not availble 
      - pgpgin/s and pgpgout/s - number pages bround in the memory from the disk and pushed out to the disk
    - sar -B and sar -S shows how paging is different that swapping.

  - Vmstat with `-s` option can give you quick understand of the state of system.

  - ```
    pgscan_kswapd_*, pgsteal_kswapd_*
    
    These report respectively the number of pages scanned and reclaimed by kswapd since the system started. The ratio between these values can be interpreted as the reclaim efficiency with a low efficiency implying that the system is struggling to reclaim memory and may be thrashing. Light activity here is generally not something to be concerned with.
    
    pgscan_direct_*, pgsteal_direct_* (proactive reclaimation)
    These report respectively the number of pages scanned and reclaimed by an application directly. This is correlated with increases in the allocstall counter. This is more serious than kswapd activity as these events indicate that processes are stalling. Heavy activity here combined with kswapd and high rates of pgpgin, pgpout and/or high rates of pswapin or pswpout are signs that a system is thrashing heavily.
    
    More detailed information can be obtained using tracepoints.
    ```

  - VM's commited memory is something which process thinks it has for its disposal though it is not physically getting used. 

  - Debug DNS with ebpf? https://blog.cloudflare.com/revenge-listening-sockets/

  - Lmbench various micro benchmarking

  - The address space in `x86_64` is `2^64` wide, but it's too large, that's why a smaller address space is used, only 48-bits wide. So we have a situation where the physical address space is limited to 48 bits, but addressing still performs with 64 bit pointers. How is this problem solved? Look at this diagram:
  
    ```
    0xffffffffffffffff  +-----------+
                        |           |
                        |           | Kernelspace
                        |           |
    0xffff800000000000  +-----------+
                        |           |
                        |           |
                        |   hole    |
                        |           |
                        |           |
    0x00007fffffffffff  +-----------+
                        |           |
                        |           |  Userspace
                        |           |
    0x0000000000000000  +-----------+
    ```
  
    Memor areas fitted in above structure
  
    ```
    0000000000000000 - 00007fffffffffff (=47 bits) user space, different per mm
    hole caused by [48:63] sign extension
    ffff800000000000 - ffff87ffffffffff (=43 bits) guard hole, reserved for hypervisor
    ffff880000000000 - ffffc7ffffffffff (=64 TB) direct mapping of all phys. memory
    ffffc80000000000 - ffffc8ffffffffff (=40 bits) hole
    ffffc90000000000 - ffffe8ffffffffff (=45 bits) vmalloc/ioremap space
    ffffe90000000000 - ffffe9ffffffffff (=40 bits) hole
    ffffea0000000000 - ffffeaffffffffff (=40 bits) virtual memory map (1TB)
    ... unused hole ...
    ffffec0000000000 - fffffc0000000000 (=44 bits) kasan shadow memory (16TB)
    ... unused hole ...
    ffffff0000000000 - ffffff7fffffffff (=39 bits) %esp fixup stacks
    ... unused hole ...
    ffffffff80000000 - ffffffffa0000000 (=512 MB)  kernel text mapping, from phys 0
    ffffffffa0000000 - ffffffffff5fffff (=1525 MB) module mapping space
    ffffffffff600000 - ffffffffffdfffff (=8 MB) vsyscalls
    ffffffffffe00000 - ffffffffffffffff (=2 MB) unused hole
    ```
  
- Usually kernel's `.text` starts here with the `CONFIG_PHYSICAL_START` offset. We have seen it in the post about [ELF64](https://github.com/0xAX/linux-insides/blob/master/Theory/ELF.md):

  ```
    readelf -s vmlinux | grep ffffffff81000000
       1: ffffffff81000000     0 SECTION LOCAL  DEFAULT    1 
   65099: ffffffff81000000     0 NOTYPE  GLOBAL DEFAULT    1 _text
   90766: ffffffff81000000     0 NOTYPE  GLOBAL DEFAULT    1 startup_64
  ```

  Here I check `vmlinux` with `CONFIG_PHYSICAL_START` is `0x1000000`. So we have the start point of the kernel `.text` - `0xffffffff80000000` and offset - `0x1000000`, the resulted virtual address will be `0xffffffff80000000 + 1000000 = 0xffffffff81000000`.

  After the kernel `.text` region there is the virtual memory region for kernel module, `vsyscalls` and an unused hole of 2 megabytes.

- ELF (Executable and Linkable Format) is a standard file format for executable files, object code, shared libraries and core dumps. An ELF object file consists of the following parts:

  - ELF header - describes the main characteristics of the object file: type, CPU architecture, the virtual address of the entry point, the size and offset of the remaining parts, etc...;
  - Program header table - lists the available segments and their attributes. Program header table need loaders for placing sections of the file as virtual memory segments;
  - Section header table - contains the description of the sections.

  Try `readelf -h <bin>` to read header section of the file.

- 