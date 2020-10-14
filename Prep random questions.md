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
    - various memory zones - https://utcc.utoronto.ca/~cks/space/blog/linux/KernelMemoryZones

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
  
  - Use Core file to analyze the dump using gdb , reference links [gdb example](https://gist.github.com/jarun/ea47cc31f1b482d5586138472139d090) and [debugging containers](https://sysdig.com/blog/troubleshooting-containers/). More [examples](https://www.dedoimedo.com/computers/gdb.html)
  
  - `debug.exception-trace` sysctl setting enables segfault messages.
  
  - Various legacy vsyscall execution mechanism , boot-time or compile-time params
  
    - CONFIG_LEGACY_VSYSCALL_NATIVE. - dangerous one - the vsyscall page contains native machine code that just calls the respective time()/getcpu()/… system calls and can be directly executed from a process.
    - CONFIG_LEGACY_VSYSCALL_EMULATE - generates page fault to move to kernel space - which will execute the page fault handler and then emulate the system call on behalf of the process, rather than letting it execute native machine code from the *vsyscall* page itself. 
    - CONFIG_LEGACY_VSYSCALL_NONE - calling process is sent a *SIGSEGV* signal
  
- Pipes 
  - They are the connectors where output of a process passes through kernel space buffer and fed into another process.
  - pipe buffer size `/proc/sys/fs/pipe-max-size`
  
- [Binary and Hex basic page](https://www.bottomupcs.com/chapter01.xhtml)
  
- [Numbers](https://www.bottomupcs.com/types.xhtml)
  
  - `char`  8bit (1 byte), `int` short 16bit (2 bytes), `int` long 32bit (4 bytes), int long long 64bit (8 bytes)
  - one's complement and two's complement.
  - In scientific notation the value `123.45` might commonly be represented as `1.2345x102`. We call `1.2345` the *mantissa* or *significand*, `10` is the *radix* and `2` is the *exponent*.
  
- CPU 
  - CPU loads data from the memory to registers and then does computation on those stored data. 
  
  - CPU keeps a track of next instruction and based on guesswork it modifies the pointer to next instruction significanly. This calls branching.
  
  - Executing a single instruction consists of a particular cycle of events
    - Fetch : get the instruction from memory into the processor.
    - Decode : internally decode what it has to do (in this case add).
    - Execute : take the values from the registers, actually add them together
    - Store : store the result back into another register. You might also see the term retiring the instruction.
    
  - Inside CPUure 3.2. Inside the CPU
  
    ![img](images/inside_CPU.png)
  - To occupy all the above CPU components to raise efficiency, it uses pipelining. Sometimes pipeline can get flushed if branch predication fails. This is parallelized execution of various blocks called superscalar architecture.
  - Sorting of the data reduces branch predictions.
  - RISC has single instruction doing simple things vs CISC has single instruction doing multiple things. RISC compiler who generates code need to be smarter at the same time chip design for RISC becomes much simpler.


- Memory
The CPU can only directly fetch instructions and data from cache memory, located directly on the processor chip. Cache memory must be loaded in from the main system memory.

The reason caches are effective is because computer code generally exhibits two forms of locality
- Spatial locality suggests that data within blocks is likely to be accessed together.
- Temporal locality suggests that data that was used recently will likely be used again shortly.

**Cache replacement strategy** 

![img](images/Memory_Caching_Strategy.png)

- Direct - only one place to place memory block, this forces other block in the same slot to be replaced.
- 4 - Way set associative - 4 possible places for given memory block, whichever is free can be utilized without eviction.
- Fully associative - put block anywhere, this might need some LRU managment.



- Type of files 
  - Regualr file
  - Directory file
  - Block Speical file
  - Character file 
  - FIFO
  - Socket
  - Symbolic link
  - With new posix standard following are represented as files
    - Message queues
    - Semaphore 
    - Shared memory object
- User ID of the process - it has real user id , saved user id and effective user id. This is a provision for working with set guid.
- Dir `+x` bit allows to `cd` in the directory. Dir `+r` will give you permission to read dir with `ls`. `+w` will allow you to create a file.
- user id of the file will be effective user ID of the process and group ID would be the same as process unless directory setGID bit is set.
- `umask` invert of the mask.
- `ls` command shows modification time in the output.
- `read` returns `0` for any holed area in the file.
- Files 
  ![img](images/files_block_struct.png)
- Dirs
  ![img](images/Dir_block_structure.png)

- Hard links to dir will cause loops

- Following symlink is an atomic operation - that is the reason we used it in our deployment mechanism. How it can be used for deployments? https://temochka.com/blog/posts/2017/02/17/atomic-symlinks.html

- each entry in dir has a fix size so it grows baed of number of entries in it.

- Stream IO with multiple byte read for each charater which is getting used for Unicodes.

- Buffering for file descritors 

  - Files are fully buffered 
  - STDIN and STDOUT are line buffered  (only when connected to tty device)
  - STDERR is unbuffered

- | **Storage Class**   | **Declaration**         | **Storage**   | **Default Initial Value** | **Scope**                                                    | **Lifetime**              |
  | ------------------- | ----------------------- | ------------- | ------------------------- | ------------------------------------------------------------ | ------------------------- |
  | **auto**            | Inside a function/block | Memory        | Unpredictable             | Within the function/block                                    | Within the function/block |
  | **register**        | Inside a function/block | CPU Registers | Garbage                   | Within the function/block                                    | Within the function/block |
  | **extern**          | Outside all functions   | Memory        | Zero                      | Entire the file and other files where the variable is declared as extern | program runtime           |
  | **Static (local)**  | Inside a function/block | Memory        | Zero                      | Within the function/block                                    | program runtime           |
  | **Static (global)** | Outside all functions   | Memory        | Zero                      | Global                                                       | program runtime           |

- Variables which are 

  - global and static are stored in the data segment. 
  - And constants are stored in the code segment.
  - Variables which are not initialized but statically declared as a part dynamically loaded libraries are stored in `bss` area. 

- `fsync()` systemcall to flush everything from memory to disk. `fflush` does something similar for streaming IO.

- File table entry 

  - stores the file offset. 
  - Each file table entry gets created upon creation of new File Descriptor. 
  - `fork()` and `dup()` will point to same file table entry through different file descriptors.

  ![linux-file-description](images/linux-file-description.png)

- [Standard IO library functions](https://en.wikipedia.org/wiki/C_file_input/output) which abstract underlying POSIX related details of the OS.

- user `nobody` - special user who can access files which are readable or writable for the world.

- `utmp` gives you existing user logins on the terminal, `wtmp` will provide historic information. `btmp` keeps track of failed logins.

- Environment variable is and pointer to the pointers

  ![Figure 7.5 Environment consisting of five C character strings](images/process_env_vars.png)

`NULL` pointer below is to extend that variable list. Initial pointer is stored on top of the stack.

![Figure 7.5 Environment consisting of five C character strings](images/process_env_var_stored.png)



More accurate representation considering kernel space is here
![proc_kern_virt_mem_map](images/proc_kern_virt_mem_map.png)



- `setjmp` and `longjmp` to jump between functions. They clean all stack frames between functions while jumping. Effect of such jumps on various type of variables also depends on gcc optimization flags.

- `getrlimit` and `setrlimit` syscalls used to setup ulimit while spawning.  The `prlimit` allows to set and read the resource limits of a process specified by PID.  `struct rlimit` gets used to track various [limits](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-6.html). 

- `vfork()` gurantees that child will run before parent. This has some security issues.

- `exec()` file reads first line for `#!` and accordingly spawn the interprater of the script.

- `SIGCHLD` gets delivered to parent which get caught by `wait` functions to avoid zombies.

- Hash  `#`  has to be universal line commenter for every interprater language because the way `exec()` calls  processes shebang.

- Shell session 
  ![Figure 9.9 Summary of job control features with foreground and background jobs, and terminal driver](images/Process_control_shell_session.png)

- summarize `nohup`, `disown` and `&`:
  
  - `&` puts the job in the background, that is, makes it block on attempting to read input, and makes the shell not wait for its completion.
  - `disown` removes the process from the shell's job control, but it still leaves it connected to the terminal. One of the results is that the shell won't send it a `SIGHUP`. Obviously, it can only be applied to background jobs, because you cannot enter it when a foreground job is running.
  - `nohup` disconnects the process from the terminal, redirects its output to `nohup.out` and shields it from `SIGHUP`. One of the effects (the naming one) is that the process won't receive any sent `SIGHUP`. It is completely independent from job control and could in principle be used also for foreground jobs (although that's not very useful).
  
- Signals 
  
  - they get generated 
  - they get delivered to the process
  - they stay pending till they get caught, most of them defaulted to be ignored.
  - process catches signal with the setup handlers.
  - signals can be blocked by process and some of the real time operating system they can be queued.
  
- looks like SIGNAL interrupted system calls never get restarted in linux by default.

- `/proc/stat` used by `vmstat` or `dstat`.

- Interrupts - why we have more than 255 interrupts on x86 architecture? That is because of IO-APIC , read this 

- debug symbol table maps compiled binary instructions to corrosponding variable functions or lines in the source code.  

- [TCP BBR experiment](https://medium.com/@atoonk/tcp-bbr-exploring-tcp-congestion-control-84c9c11dc3a9#:~:text=Bottleneck%20Bandwidth%20and%20Round%2Dtrip,developed%20at%20Google%20in%202016.&text=BBR%20tackles%20this%20with%20a,to%20determine%20the%20sending%20rate.) by creating delay and packet loss

  ```
  # introduce latency 
  tc qdisc replace dev enp0s20f0 root netem latency 70ms
  # introduce latency and pkt drop
  tc qdisc replace dev enp0s20f0 root netem loss 1.5% latency 70ms
  
  # start BBR
  sysctl -w net.ipv4.tcp_congestion_control=bbr
  
  
  # socket stats
  ss -tni
  ```

  ![Image for post](images/TCP+BBR+compare.png)

- Blocked process in `vmstat` output are the one waiting on blocked IO and running or runnable process are shown in the other column. It is coming from `/proc/stats` populated [here](https://github.com/torvalds/linux/blob/63bef48fd6c9d3f1ba4f0e23b4da1e007db6a3c0/fs/proc/stat.c#L201)

- `lsattr` shows extended file attributes, some of them are provided by the file system and immutable by `chattr`. Here is an article with [description](http://www.linuxintheshell.com/2013/04/23/episode-028-extended-attributes-lsattr-and-chattr/) 

- The **inode** (index node) is a data structure in a Unix-style file system that describes a file-system object such as a file or a directory. Each **inode** stores the attributes and disk block locations of the object's data. There are direct pointers (usually for smaller files) but then there are indirect pointers which are for larger files which helps to keep **inode** structure compact.

- Background processes , can be setup by `&` and when they try to do anything with STDIN or STDOUT then they are stopped by SIGTTIN or SIGTTOUT signal.

- Tmux is a server and every connected terminal is a client which get served by its session.

- Process state codes

  ```shell
  PROCESS STATE CODES
         Here are the different values that the s, stat and state output
         specifiers (header "STAT" or "S") will display to describe the state of
         a process.
         D    Uninterruptible sleep (usually IO)
         R    Running or runnable (on run queue)
         S    Interruptible sleep (waiting for an event to complete)
         T    Stopped, either by a job control signal or because it is being
  	    traced.
         X    dead (should never be seen)
         Z    Defunct ("zombie") process, terminated but not reaped by its
  	    parent.
  
         For BSD formats and when the stat keyword is used, additional
         characters may be displayed:
         <    high-priority (not nice to other users)
         N    low-priority (nice to other users)
         L    has pages locked into memory (for real-time and custom IO)
         s    is a session leader
         l    is multi-threaded (using CLONE_THREAD, like NPTL pthreads do)
         +    is in the foreground process group
  ```

- `forcefsck` empty file to force check on next reboot.

- Knowing process ID , you can go through processes' `/proc/PID/` dir to know activities like listening sockets and many others.

- TLS and its assymetric and symmetric handshake

  - The client contacts the server using a secure URL (HTTPS…).
  - The server sends the client its certificate and public key. 
  - The client verifies this with a Trusted Root Certification Authority to ensure the certificate is legitimate. There are chain of signatures on the server certificate by various authorities (like verisign). Those signatures can be verified by their respective public keys, these publics comes as a part of standard openssl package or browser bundle. 
  - The client and server negotiate the strongest type of encryption that each can support, the supported symmetric cipher. Client generates the secret for the symmentric cipher.
  - The client encrypts a session (secret) key with the server’s public key, and sends it back to the server.
  - The server decrypts the client communication with its private key, and the session is established.
  - The session key (symmetric encryption) is now used to encrypt and decrypt data transmitted between the client and server.
  - This whole things take atleast 110ms (on top of TCP handshake of 50ms)
  - When a key exchange uses Ephemeral Diffie-Hellman a temporary DH key is generated for every connection and thus the same key is never used twice. This enables Forward Secrecy (FS), which means that if the long-term private key of the server gets leaked, past communication is still secure.

- maximum length of arguments for a new process `getconf ARG_MAX` . In practice, subtract existing enviornment size with 

  ```shell
  echo $(( $(getconf ARG_MAX) - $(env | wc -c) ))
  ```

  But then also keep some room for spwaned shell to modify env variables 

  ```shell
  expr `getconf ARG_MAX` - `env|wc -c` - `env|wc -l` \* 4 - 2048
  ```

- NTP

  - Stratum 0 serves as a reference clock and is the most accurate and highest precision time server (e.g., atomic clocks, GPS clocks, and radio clocks.) Stratum 1 servers take their time from Stratum 0 servers and so on up to Stratum 15; Stratum 16 clocks are not synchronized to any source. 

  -  Ideally, it would work to have three or more Stratum 0 or Stratum 1 servers and use those servers as primary masters. 

  - ```shell
    $ ntpq -p
         remote           refid      st t when poll reach   delay   offset  jitter
    ==============================================================================
     0.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
     1.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
     2.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
     3.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
     ntp.ubuntu.com  .POOL.          16 p    -   64    0    0.000    0.000   0.000
    *ntp3.junkemailf 216.218.254.202  2 u   53   64   37    1.893    1.029   0.703
    +time.cloudflare 10.98.8.8        3 u   48   64   37   25.829   -0.308   0.787
    -44.190.6.254    23.131.160.7     3 u   51   64   37    2.410   -1.374   0.760
    +time.cloudflare 10.98.8.8        3 u   53   64   37   25.297   -0.014   0.783
    -50-205-244-112- 50.205.244.27    2 u   48   64   37   52.013    0.720   0.760
    -ntp1.wiktel.com .PPS.            1 u   51   64   37   72.899   -2.498   0.713
    #dns2.fuedo.de   192.53.103.104   2 u   44   64   35  159.522    4.682   0.738
    
    
    $ timedatectl status
                          Local time: Wed 2020-09-23 18:08:49 UTC
                      Universal time: Wed 2020-09-23 18:08:49 UTC
                            RTC time: Wed 2020-09-23 18:08:50
                           Time zone: Etc/UTC (UTC, +0000)
           System clock synchronized: yes
    systemd-timesyncd.service active: yes
                     RTC in local TZ: no
    ```

  - The reach column contains the results of the most recent eight NTP updates. If all eight are successful, this field will read 377. This number is in octal, so eight successes in octal will be represented by 377.

- Soft limits (designated via -aS) are boundaries that, if breached, cause a SIGX signal to be sent to the running process indicating it must lower its consumption immediately. 

  Hard limits (designated via -aH) on the other hand, are final upper bounds for each respective limit type. These values are the absolute maximum that a process can reach before the operating system issues a SIGKILL signal to the process. This signal cannot be caught and will immediately terminate the process.

- `ulimit` vs `cgroups`

  -  `cgroups` are considered for allocate resources among groups of tasks, while `ulimit` only works on a process level. For example, `ulimits` are getting inherited from parent but those are indepdent from the parent's limits accounting. `cgroups` accounting is applied for group or processes.

- Object vs Block storage.

- The Linux Unified Key Setup (LUKS) is a disk encryption specification.

- Close socket without killing process

  ```
  - Find the offending process: netstat -np
  - Find the socket file descriptor: lsof -np $PID
  - Debug the process: gdb -p $PID
  - Close the socket: call close($FD)
  - Close the debugger: quit
  ```

- INT 80 for system call execution.

- Kernel per-map space for page tables to break cicular depedency to map 

- Implementation of KASLR and KPTI patching. KPTI moves most of the kernel pages out of the process mapping and use Process context ID to reduce performance reduction.  Individual TLB entries can be flushed using PC-ID. 

- Every process has its own page table, this pointers gets loaded into CR3 register. 

-  Time Stamp Counter (TSC) - this is on die clock

- RAM+swap to be overcommitted, 

- ```
  $ while true; do mkdir x; cd x; done
  ```

  This script will create a directory structure that is as deep as possible. Each subdirectory "x" will create a dentry (directory entry) that is pinned in non-reclaimable kernel memory. Such a script can potentially consume all available memory before filesystem quotas or other filesystem limits kick in, and, as a consequence, other processes will not receive service from the kernel because kernel memory has been exhausted. (One can monitor the amount of kernel memory being consumed by the above script via the `dentry` entry in `/proc/slabinfo`.)

- ```
  cat /proc/sys/kernel/random/entropy_avail
  ```

  Entropy is the measure of the random numbers available from /dev/urandom, and if you run out, you can’t make SSL connections. To check the status of your server’s entropy, just run the above.

  ***Linux Entropy Lifecycle***

  ![User-added image](images/linux_entropy.png)
  
  
 - Linux Network Traffic Control
    - The NTC mechanism is managed by the tc program. This tool allows a "queueing discipline" (or "qdisc") to be attached to each network interface. Some qdiscs are "classful" and these can have other qdiscs attached beneath them, one for each "class" of packet. If any of these secondary qdiscs are also classful, a further level is possible, and so on. 
    - Each packet will be classified by the various filters into one of the queues in the tree and then will propagate up to the root, possibly being throttled (for example by the Token Bucket Filter, tbf, qdisc) or being competitively scheduled (e.g. by the Stochastic Fair Queueing, sfq, qdisc). Once it reaches the root, it is transmitted.


