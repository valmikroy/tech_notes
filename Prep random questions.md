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

  - Vsyscall or fast syscalls do not appear in `strace` output.
  
  - Where can I get sourcode for glibc 
  
    ```
    git clone git://sourceware.org/git/glibc.git
    ```
  
  - Generate core dump by setting up `ulimit -c unlimited` before crashing application. You can generate core file with `kill -s SIGSEGV $$`. Core file is nothing but the memory map of all instructions which get executed. 
  
  - Use Core file to analyze the dump using gdb , reference links [gdb example](https://gist.github.com/jarun/ea47cc31f1b482d5586138472139d090) and [debugging containers](https://sysdig.com/blog/troubleshooting-containers/).
  
  - `debug.exception-trace` sysctl setting enables segfault messages.
  
  - Various legacy vsyscall execution mechanism , boot-time or compile-time params
  
    - CONFIG_LEGACY_VSYSCALL_NATIVE. - dangerous one - the vsyscall page contains native machine code that just calls the respective time()/getcpu()/… system calls and can be directly executed from a process.
    - CONFIG_LEGACY_VSYSCALL_EMULATE - generates page fault to move to kernel space - which will execute the page fault handler and then emulate the system call on behalf of the process, rather than letting it execute native machine code from the *vsyscall* page itself. 
    - CONFIG_LEGACY_VSYSCALL_NONE - calling process is sent a *SIGSEGV* signal
  
    
  
    
  
    
  
    