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
  - Signals are similar to [interrupts](https://en.wikipedia.org/wiki/Interrupt), the difference being that interrupts are mediated by the processor and handled by the [kernel](https://en.wikipedia.org/wiki/Kernel_(operating_system)) while signals are mediated by the kernel (possibly via system calls) and handled by processes. The kernel may pass an interrupt as a signal to the process that caused it (typical examples are [SIGSEGV](https://en.wikipedia.org/wiki/SIGSEGV), [SIGBUS](https://en.wikipedia.org/wiki/SIGBUS), [SIGILL](https://en.wikipedia.org/wiki/Signal_(IPC)#SIGILL) and [SIGFPE](https://en.wikipedia.org/wiki/Signal_(IPC)#SIGFPE)).
  
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

- Soft limits (designated via -aS) are boundaries that, if breached, cause a **SIGX** signal to be sent to the running process indicating it must lower its consumption immediately. 

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

- **INT 80** for system call execution.

- Kernel per-map space for page tables to break cicular depedency to map (???)

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
    
- `eventfd` and `inotify` mechanism to create event notification system.

- ASN number for each organization who does peering on exchanges.

- MAX payload possible 1369 = 1500 - 40 (IPv6) - 20 (TCP) - 10 (Time) - 61 (Max TLS overhead))

- ICMP responded by the kernel, uneven timing in the `ping` response is the sign of busy softirq.

- Busy poll - example https://www.chelsio.com/wp-content/uploads/resources/T5-10Gb-BusyPoll-Chelsio-vs-Intel.pdf

- TCP RACK (Recent ACK) loss recovery uses the notion of time instead of packet sequence (FACK) or counts (dupthresh).

- TCP Tail loss probe - goal is to reduce tail latency of short transactions. it achieves that by  converting retransmission timeouts (RTOs) occuring due to tail losses (losses at end of transactions) into fast recovery.

- `TCP_NOTSENT_LOWAT`  https://lwn.net/Articles/560082/  reduce memory usage on the socket.

- ` /proc/sys/kernel/kptr_restrict` 

- Systemd - [rethinking PID 1](http://0pointer.de/blog/projects/systemd.html)

    - Systemd brings up the userspace, we want to do it faster.

    - It does it faster by opening required sockets and then starts server and client processes which are going to use those sockets in parallel. This reduces boottime.

    - Above socket approach it uses for filesystem reads with the help of autofs which allows `open()` call to be blocking until real VFS swaps the mountpoint.

    - Shell scripts are slow and forks a lot - verify that by looking at your PID after first boot login session.

    - CGroups are getting used to do babysitting of the group of processes which is much efficient. Logging is layered on it.

    -  Systemd has units for executation, they all can be depedent on each other. Unit types

        - `service` - actual daemon processes with start and stop options 
        - `sockets` - various sockets to which above `service` units must be attached.
        - `device` - device `udev` support.
        - `mount` - mount points 
        - `automount` - autofs mounting
        - `target` - this is logical grouping of various units which can define internel depedencies
        - `snapshot` - this is for rollbacks and emergency shell creation.

        ![image-20201024140658320](images/systemd_daemon_arch.png)



### Linux kernel packet traverse    

![linux_kernel_nw_traverse](/Users/abhisawa/git/lc-practice/tech_notes/images/linux_kernel_nw_traverse.jpg)









#### TLS

TLS performace [compare by Intel](https://software.intel.com/content/www/us/en/develop/articles/improving-openssl-performance.html)

Kernel level TLS performance aka [ktls](https://www.kernel.org/doc/html/latest/networking/tls-offload.html)



#### CPU

Turbo 

```
$ sudo cpupower frequency-info
analyzing CPU 0:
  driver: intel_pstate
  CPUs which run at the same hardware frequency: 0
  CPUs which need to have their frequency coordinated by software: 0
  maximum transition latency:  Cannot determine or is not supported.
  hardware limits: 1.20 GHz - 3.10 GHz
  available cpufreq governors: performance powersave
  current policy: frequency should be within 1.20 GHz and 3.10 GHz.
                  The governor "powersave" may decide which speed to use
                  within this range.
  current CPU frequency: Unable to call hardware
  current CPU frequency: 1.20 GHz (asserted by call to kernel)
  boost state support:
    Supported: yes
    Active: yes
```

Monitor CPU freq shift

```
turbostat --debug
```

Learn more about [Busy Polling](https://netdevconf.info/2.1/papers/BusyPollingNextGen.pdf) for network sockets.

Reduce effect of CPU power management on 

```shell
# kernel cmdline options


#https://gist.github.com/Brainiarc7/8dfd6bb189b8e6769bb5817421aec6d1
processor.max_cstates=1 
intel_idle.max_cstate=0

#https://www.kernel.org/doc/html/v5.0/admin-guide/pm/cpuidle.html
idle=poll 
```

CPU affinity and negative effect on `runqlat`



#### Memory 

`vm.zone_reclaim_mode`

NUMA commands 

```shell
numactl --interleave=all
numactl --hardware
numastat -n -c
numastat -m -c
```

FB platform [single NUMA](https://engineering.fb.com/data-center-engineering/facebook-s-new-front-end-server-design-delivers-on-performance-without-sucking-up-power/)



NUMA related kernel activities

```shell
perf stat -e sched:sched_stick_numa,sched:sched_move_numa,sched:sched_swap_numa,migrate:mm_migrate_pages,minor-faults -p PID
```





#### PCI

PCIE troubleshooting [101](https://intrepid.warped.com/~scotte/OldBlogEntries/web/index-5.html)

Disable PCIe power management 

```
# kernel commandline
pcie_aspm=off
```

![img](/Users/abhisawa/git/lc-practice/tech_notes/images/pcie-table.png)source: https://en.wikipedia.org/wiki/PCI_Express#History_and_revisions

[PCIe tunig guide](https://community.mellanox.com/docs/DOC-2496)



#### NIC







##### BQL

This is accomplished by adding a layer which enables and disables queuing to the driver queue (qdiscs) based on calculating the minimum buffer size required to avoid starvation under the current system conditions.

This creates back pressure on the qdiscs.

 The BQL algorithm is self tuning so you probably don’t need to mess with this too much.  But you can tune it with values in dir `/sys/devices/pci0000:00/0000:00:14.0/net/eth0/queues/tx-0/byte_queue_limits`



##### Ethtool and softstat

To find pkt drop in device driver space

```shell
ethtool -S eth0 | egrep 'miss|over|drop|lost|fifo'
```

Look for CPU squeezes and collisions in OS softirq space with `/proc/net/softnet_stat` Tune time and packet budget.



Enable Coalescing on NIC to reduce interrupt volume 

```shell
ethtool -c eth0
```

Tune RSS and RFS (needs CPU affinity)



##### OS network stack

- Collect system-wide TCP metrics via `/proc/net/snmp` and `/proc/net/netstat`.
- Aggregate per-connection metrics obtained either from `ss -n --extended --info`, or from calling `getsockopt(TCP_INFO)`/`getsockopt(TCP_CC_INFO)` inside your webserver.
- [tcptrace](https://linux.die.net/man/1/tcptrace)(1)’es of sampled TCP flows.
- Analyze RUM metrics from the app/browser.



##### TCP tunes

![img](/Users/abhisawa/git/lc-practice/tech_notes/images/linux-tcp-stack.png)



- Pacing - Fair queing with qdisc 
- TSO - TCP Segment Offload - `net.ipv4.tcp_min_tso_segs`.
- TSQ - TCP small queues - `net.ipv4.tcp_limit_output_bytes`
- Congension control - BBR (througput remains higher during packet loss )
- [TLP](https://lwn.net/Articles/542642/) and [RACK](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=eb9fae328faff9807a4ab5c1834b19f34dd155d4) to work on ACK processing and loss detection.





Some tcp optimization 

- `net.ipv4.tcp_notsent_lowat`   apprently useful for HTTP2
- `net.ipv4.tcp_tw_recycle=1` DO NOT USE - broken in kernel
- `net.ipv4.tcp_timestamps=0` DO NOT USE - timestamp allows various SACK and window scaling functionlity which we will loose.



As for sysctls that you should be using:

- `net.ipv4.tcp_slow_start_after_idle=0`—the main problem with slowstart after idle is that “idle” is defined as one RTO, which is too small.
- `net.ipv4.tcp_mtu_probing=1`—[useful if there are ICMP blackholes between you and your clients](https://blog.cloudflare.com/ip-fragmentation-is-broken/) (most likely there are).
- `net.ipv4.tcp_rmem`, `net.ipv4.tcp_wmem`—should be tuned to fit BDP, just don’t forget that [bigger isn’t always better](https://blog.cloudflare.com/the-story-of-one-latency-spike/).
- `echo 2 > /sys/module/tcp_cubic/parameters/hystart_detect`—if you are using fq+cubic, this [might help with tcp_cubic exiting the slow-start too early](https://groups.google.com/forum/#!topic/bbr-dev/g1tS1HUcymE).

It also worth noting that there is an RFC draft (though a bit inactive) from the author of curl, Daniel Stenberg, named [TCP Tuning for HTTP](https://github.com/bagder/I-D/blob/gh-pages/httpbis-tcp/draft.md), that tries to aggregate all system tunings that may be beneficial to HTTP in a single place.



#### Flamegraph and perf optimization 

Check performance of following library calls

- Zlib - compression 

- Malloc

- PCRE

  ```
  funclatency /srv/nginx-bazel/sbin/nginx:ngx_http_regex_exec -u
  ```



#### TLS

Reference 

Most of the optimizations I’ll be mentioning are covered in the [High Performance Browser Networking](https://hpbn.co/)’s “ [Optimizing for TLS](https://hpbn.co/transport-layer-security-tls/#optimizing-for-tls)” section and [Making HTTPS Fast(er)](https://www.youtube.com/watch?v=iHxD-G0YjiU) talk at nginx.conf 2014. 

Tunings mentioned in this part will affect both performance and security of your web server, if unsure, please consult with [Mozilla’s Server Side TLS Guide](https://wiki.mozilla.org/Security/Server_Side_TLS) and/or your Security Team.

- TLS connection resume without handshake - TLS session tickets or TLS session cache
- OSCP stapling 
- TLS record size 

![TLS_record_size](/Users/abhisawa/git/lc-practice/tech_notes/images/TLS_record_size.jpg)



20KB of application payload will get divided into 1400 bytes of ~15 TCP segments which are more than TCP congestion window. 



[Overall reference post](https://dropbox.tech/infrastructure/optimizing-web-servers-for-high-throughput-and-low-latency)



# Troubleshooting

USE - Utilization - Saturation - Errors

| resource           | type        | metric                                                       |
| ------------------ | ----------- | ------------------------------------------------------------ |
| CPU                | utilization | CPU utilization (either per-CPU or a system-wide average),   |
| CPU                | saturation  | run-queue length or scheduler latency(aka                    |
| Memory capacity    | utilization | available free memory (system-wide)                          |
| Memory capacity    | saturation  | anonymous paging or thread swapping (maybe "page scanning" too) |
| Network interface  | utilization | RX/TX throughput / max bandwidth,                            |
| Storage device I/O | utilization | device busy percent                                          |
| Storage device I/O | saturation  | wait queue length                                            |
| Storage device I/O | errors      | device errors ("soft", "hard", ...)                          |





Scheduler latency 

```shell
perf sched record -a sleep 6
perf sched latency -s max
```

[Run queue latency](http://www.brendangregg.com/blog/2016-10-08/linux-bcc-runqlat.html) 

```shell
runqlat
```







Now for some harder combinations (again, try to think about these first!):



| resource            | type        | metric                                                       |
| ------------------- | ----------- | ------------------------------------------------------------ |
| CPU                 | errors      | eg, correctable CPU cache ECC events or faulted CPUs (if the OS+HW supports that) |
| Memory capacity     | errors      | eg, failed malloc()s (although this is usually due to virtual memory exhaustion, not physical) |
| Network             | saturation  | saturation related NIC or OS events; eg "dropped", "overruns" |
| Storage controller  | utilization | depends on the controller; it may have a max IOPS or throughput that can be checked vs current activity |
| CPU interconnect    | utilization | per port throughput / max bandwidth (CPU performance counters) |
| Memory interconnect | saturation  | memory stall cycles, high CPI (CPU performance counters)     |
| I/O interconnect    | utilization | bus throughput / max bandwidth (performance counters may exist on your HW; eg, Intel "uncore" events) |







# SAR

Using SAR would be first step towards troubleshooting any system. We will take a look at each subsystem at a time.



## CPU 

- `-m` provides CPU package related information, many of them do not work as they are very platform dependent.
- `-P <ALL/CPU number>` will provide standard places for CPU utilization.
- `-u <ALL/CPU number>` will provide advance view of CPU utilization like 
  - `irq` and `softirq` handling 
  - `steal` is involuntery wait for virtual CPU availability



## Scheduler

Scheduling is another subsystem which reflects health of the system

`-q` is complimenty to what we get through load average, it also provides number of tasks in the tasklist in  `plist-sz` column. It also shows count of runnable and blocked processes. This also provides load avergaes numbers which is even [complicated in linux](http://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html) where it considers `TASK_UNINTERRUPTIBLE` also in under consideration which include uninterruptable lock and disk io waiting along with CPU resources. That means load average more than CPU counts in Linux might include threads which are blocked on the non-CPU type of resources. It is very confusing in Linux to decide on load average.



Because linux load averages can includes more than CPU, to know specific CPU demands, you can run 

- `mpstat`  for per CPU utilization 
- `pidstat -p <ALL/PID> 1` for per process CPU utilization 
- `/proc/PID/schedstats` for per thread CPU timings where you will find 3 numbers 
  - time spent on CPU 
  - time spent waiting in a runqueue 
  - count of time slices run on the CPU
- `/proc/schedstat` CPU run queue latency 
- `vmstat 1` can provide you run queue length 



You can run a full debug of each processes allocated time in scheduling with folllowing one liner 

``` 
$ awk 'NF > 7 { if ($1 == "task") { if (h == 0) { print; h=1 } } else { print } }' /proc/sched_debug

 S           task   PID         tree-key  switches  prio     wait-time             sum-exec        sum-sleep
 I         rcu_gp     3        21.995531         2   100         0.000000         0.002975         0.000000 0 0 /
 I     rcu_par_gp     4        23.496139         2   100         0.000000         0.001857         0.000000 0 0 /
 I   kworker/0:0H     6      1187.772800         6   100         0.000000         0.036662         0.000000 0 0 /
 I   mm_percpu_wq     9        29.066425         2   100         0.000000         0.002497         0.000000 0 0 /
 S    ksoftirqd/0    10    128714.543436      4792   120         0.000000        33.758851         0.000000 0 0 /
 S    migration/0    12         0.000000    198777     0         0.000000      1763.040884         0.000000 0 0 /
 S        cpuhp/0    13      3586.151423         8   120         0.000000         0.193625         0.000000 0 0 /
 S     khugepaged    60    128714.543436     58304   139         0.000000       629.884360         0.000000 0 0 /
 I     scsi_tmf_1   178      1115.471018         2   100         0.000000         0.025049         0.000000 0 0 /
 I   kworker/0:1H   223    128714.543436     38621   100         0.000000       100.824080         0.000000 0 0 /
 I         cryptd   326      1845.269827         2   100         0.000000         0.022569         0.000000 0 0 /
 S     multipathd   416         0.000000     25929     0         0.000000       467.263788         0.000000 0 0 /system.slice/multipathd.service
 S     multipathd   419         0.000000         6     0         0.000000         0.183746         0.000000 0 0 /system.slice/multipathd.service
 S          loop3   428    127770.543906       520   100         0.000000         8.003048         0.000000 0 0 /
```



## Memory 

- `sar -B`

```
09:35:01 AM  pgpgin/s pgpgout/s   fault/s  majflt/s  pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff
09:45:01 AM    550.82    629.41  67380.81      0.00 182789.00      0.00      0.00      0.00      0.00
09:55:01 AM   3924.61   1557.42  67804.17      0.00 186321.15      0.00      0.00      0.00      0.00
10:05:01 AM    577.42    659.43  68622.82      0.00 187206.07      0.00      0.00      0.00      0.00
10:15:01 AM    590.52    663.89  70290.50      0.00 190128.47      0.00      0.00      0.00      0.00
10:25:01 AM    608.76    675.59  70878.25      0.00 193471.10      0.00      0.00      0.00      0.00
10:35:01 AM    615.69    702.96  71677.72      0.00 195602.52      0.00      0.00      0.00      0.00
10:45:01 AM    625.39    698.00  72230.19      0.00 199102.83      0.00      0.00      0.00      0.00
10:55:01 AM   4388.83   1865.92  73875.07      0.00 204701.71      0.00      0.00      0.00      0.00
11:05:01 AM    648.40    869.32  75274.57      0.00 207472.02      0.00      0.00      0.00      0.00
11:15:01 AM    660.22    729.57  76614.26      0.00 211789.20      0.00      0.00      0.00      0.00
11:25:01 AM    671.43    742.83  77571.92      0.00 215410.96      0.00      0.00      0.00      0.00
11:35:01 AM    679.13    752.46  78134.20      0.00 217607.37      0.00      0.00      0.00      0.00
11:45:01 AM    698.52    760.63  78790.67      0.00 221150.22      0.00      0.00      0.00      0.00
11:55:01 AM   4816.39   1620.73  80107.70      0.02 228635.72      0.00      0.00     77.29      0.00 <<

```

Pgscank - Number of pages scanned by the kswapd daemon per second.

Pgscand - Number of pages scanned directly per second.

pgsteal/s - Number of pages the system has reclaimed from cache (pagecache and  swapcache)  per  second  to satisfy its memory demands.

- `sar -S`

```
09:35:01 AM kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
09:45:01 AM   7968508     30976      0.39      1000      3.23
09:55:01 AM   7968508     30976      0.39      1000      3.23
10:05:01 AM   7968508     30976      0.39      1000      3.23
10:15:01 AM   7968508     30976      0.39      1000      3.23
10:25:01 AM   7968508     30976      0.39      1000      3.23
10:35:01 AM   7968508     30976      0.39      1000      3.23
10:45:01 AM   7968508     30976      0.39      1000      3.23
10:55:01 AM   7968508     30976      0.39      1000      3.23
11:05:01 AM   7968508     30976      0.39      1000      3.23
11:15:01 AM   7968508     30976      0.39      1000      3.23
11:25:01 AM   7968508     30976      0.39      1000      3.23
11:35:01 AM   7968508     30976      0.39      1000      3.23
11:45:01 AM   7968508     30976      0.39      1000      3.23
11:55:01 AM   7967996     31488      0.39      1484      4.71 <<
12:05:01 PM   7967996     31488      0.39      1484      4.71
12:15:01 PM   7967996     31488      0.39      1484      4.71
12:25:01 PM   7967996     31488      0.39      1484      4.71
12:35:01 PM   7968508     30976      0.39      1168      3.77
12:45:01 PM   7968508     30976      0.39      1168      3.77
12:55:01 PM   7968508     30976      0.39      1168      3.77
Average:      7968482     31002      0.39      1030      3.32

```

kbswpcad : Amount of cached swap memory in kilobytes.  This is  memory  that  once  was  swapped  out,  is swapped  back  in but still also is in the swap area (if memory is needed it doesn't need to be swapped out again because it is already in the swap area. This saves I/O).

- `sar -R`

```
09:15:01 AM   frmpg/s   bufpg/s   campg/s
09:25:01 AM   -550.64      0.00    134.69
09:35:01 AM   -252.31      0.00    139.26
09:45:01 AM   -374.95      0.00    137.65
09:55:01 AM    776.30      0.00   -745.74
10:05:01 AM   -416.63      0.00    147.68
10:15:01 AM   -403.32      0.00    146.84
10:25:01 AM   -322.58      0.00    150.67
10:35:01 AM   -294.70      0.00    155.98
10:45:01 AM   -215.79      0.00    158.61
10:55:01 AM    477.34      0.00   -832.49
11:05:01 AM   -554.04      0.00    161.00
11:15:01 AM   -619.24      0.00    163.25
11:25:01 AM    -14.84      0.00    169.01
11:35:01 AM   -504.22      0.00    167.76
11:45:01 AM   -308.36      0.00    170.76
11:55:01 AM   3350.31      0.00  -3270.30 <<
12:05:01 PM   -580.81      0.00    174.54
12:15:01 PM   -385.49      0.00    177.57
12:25:01 PM    -51.66      0.00    179.94
12:35:01 PM   -244.81      0.00    154.52
12:45:01 PM   -415.27      0.00    181.85
12:55:01 PM   1036.26      0.00   -939.86
01:05:01 PM   -489.08      0.00    184.65
Average:       -64.85      0.00    -34.59
```

frmpg/s : Number of memory pages freed by the system per second.  A negative value represents a number of pages allocated by the system.  Note that a page has a size of 4 kiB or 8 kiB according to  the machine architecture.

bufpg/s: Number  of  additional memory pages used as buffers by the system per second.  A negative value means fewer pages used as buffers by the system.

campg/s : Number of additional memory pages cached by the system per  second.   A  negative  value  means fewer pages in the cache.

- `sar -W`

```
09:15:01 AM  pswpin/s pswpout/s
09:25:01 AM      0.00      0.00
09:35:01 AM      0.00      0.00
09:45:01 AM      0.00      0.00
09:55:01 AM      0.00      0.00
10:05:01 AM      0.00      0.00
10:15:01 AM      0.00      0.00
10:25:01 AM      0.00      0.00
10:35:01 AM      0.00      0.00
10:45:01 AM      0.00      0.00
10:55:01 AM      0.00      0.00
11:05:01 AM      0.00      0.00
11:15:01 AM      0.00      0.00
11:25:01 AM      0.00      0.00
11:35:01 AM      0.00      0.00
11:45:01 AM      0.00      0.00
11:55:01 AM      0.00      0.29 <<
12:05:01 PM      0.00      0.00
12:15:01 PM      0.00      0.00
12:25:01 PM      0.00      0.00
12:35:01 PM      0.00      0.00
12:45:01 PM      0.00      0.00
12:55:01 PM      0.00      0.00
01:05:01 PM      0.00      0.00
Average:         0.00      0.00
```

- `sar -r ALL`

```
09:15:01 AM kbmemfree kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty  kbanonpg    kbslab  kbkstack   kbpgtbl  kbvmused
09:25:01 AM  12980848  52948860     80.31      5028  25506636  55404888     74.94  25443020  24851140       696  24781428   1438048     41920    116556         0
09:35:01 AM  12375540  53554168     81.23      5028  25840720  55436396     74.99  25709504  25185144       936  25046760   1438996     41840    116516         0
09:45:01 AM  11476032  54453676     82.59      5028  26170940  55421904     74.97  26273440  25515320       928  25610804   1439836     42336    116368         0
09:55:01 AM  13340976  52588732     79.76      5028  24379400  55405352     74.94  26210576  23722160       632  25544528   1434700     42400    116428         0
10:05:01 AM  12341488  53588220     81.28      5028  24733668  55400144     74.94  26849828  24076312       748  26185380   1433380     44208    116660         0
10:15:01 AM  11374176  54555532     82.75      5028  25085852  55369476     74.90  27465732  24428416       696  26800264   1434352     42800    115828         0
10:25:01 AM  10600268  55329440     83.92      5028  25447332  55401024     74.94  27879084  24789888       636  27214508   1433976     44208    116512         0
10:35:01 AM   9892100  56037608     85.00      5028  25822152  55445960     75.00  28209968  25164628       988  27544864   1434712     44416    116596         0
10:45:01 AM   9374384  56555324     85.78      5028  26202684  55483632     75.05  28338792  25545156       444  27674112   1433956     46016    116632         0
10:55:01 AM  10519384  55410324     84.04      5028  24205768  55494164     75.06  29205224  23546032       376  28536956   1424864     46176    116100         0
11:05:01 AM   9190216  56739492     86.06      5028  24592024  55473336     75.04  30146252  23925604       120  29472972   1426408     46976    117032         0
11:15:01 AM   7702160  58227548     88.32      5028  24984312  55499684     75.07  31241992  24317828       760  30568908   1426864     46736    117196         0
11:25:01 AM   7666564  58263144     88.37      5028  25389756  55474756     75.04  30874372  24723192       120  30199476   1426480     46640    116800         0
11:35:01 AM   6456904  59472804     90.21      5028  25792236  55454720     75.01  31678256  25125648       380  31004688   1427440     48512    116768         0
11:45:01 AM   5717096  60212612     91.33      5028  26201924  55502972     75.08  32013288  25535280       760  31338532   1427564     47904    116584         0
11:55:01 AM  13768296  52161412     79.12      5028  18343012  55483588     75.05  31808480  17815356       196  31273288   1299288     48048    117116         0 <<
12:05:01 PM  12374888  53554820     81.23      5028  18761736  55503776     75.08  32782104  18234068       444  32246992   1297996     48336    117560         0
12:15:01 PM  11450564  54479144     82.63      5028  19187524  55492768     75.06  33275684  18657884       320  32738344   1300296     49408    116800         0
12:25:01 PM  11326420  54603288     82.82      5028  19619916  55498792     75.07  32958776  19090260      1044  32421592   1300900     49088    117288         0
12:35:01 PM  10739144  55190564     83.71      5028  19990592  55440936     74.99  33188996  19456100       184  32647356   1298904     49568    117060         0
12:45:01 PM   9742864  56186844     85.22      5028  20426880  55428044     74.97  33754140  19891436       232  33212244   1299496     50048    117620         0
12:55:01 PM  12229056  53700652     81.45      5028  18171976  55468396     75.03  33534152  17640368      1020  32995332   1282328     49312    117744         0
01:05:01 PM  11053784  54875924     83.23      5028  18615696  55498824     75.07  34263104  18081252       708  33721924   1281624     49920    118664         0
01:15:01 PM  10636404  55293304     83.87      5028  19068160  55459132     75.02  34225696  18533624       404  33683420   1281744     50864    118152         0
Average:     17934645  47995063     72.80      5028  24888771  55471470     75.03  21092870  24263861       563  20460698   1425706     38693    116688         0
```

KbSlab - slab data structure

kbstack - stack structure in kernel

Kbpgtbl - kernel page table size 

Kbnonpg - Amount of non-file backed pages in kilobytes mapped into userspace page tables. 




