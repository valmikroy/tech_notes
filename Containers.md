# Control Groups aka GCgroups

What are the containers

Namspace and Cgroups mismash 



Controlling of process is a very old problem, it has been tackled

- Having parent and child process relationship where all children belongs to a parent.
- TTY terminal based groups based on session ID. This has some limitations.
- All has a limitation where you can join group but you can not exit group.









Control Groups - 

- meatring 
- group CPUs
- crowd control 



Each subsystem has its own hirarchy 

each process is part of different cgroups hierarchy 

whole system is a cgroups 



Memory cgroup : account 

- memory page granularity 
- file backed and anon pages 
- active and inactive memory 
- each page charged to one cgroup - sharing page and charging has discrepency 
- soft and hard limit - limits physical, total and kernel memory 
- notification - oom-notiifcation - freeze cgroup and let program deal with it
- there are some overheads for managing memory pages 



HugeTLB cgroup 

- process can be gridy to use huge pages 



CPU cgroups 

- allows track CPU usage in terms of time.  Per CPU track 
- Allows you to set weights - what are those weight?
- you can not set limit 



CPUset cgroup 

- pin groups to specific CPU(s)
- great for NUMA systems 
- avoid process bouncing between CPUs
- read more



BlockIO

- keeps track of IO for each group
- read and write , sync and async IO
- throtlles 



Net_CLS and Net_Prio

- you have to use tc to shape traffic coming from particualr cgroup.



Devices cgroup

- permissions for read/write to particualr device 
- /dev/random - fed with entrophy  in linux 
- /dev/net/tune - /dev/kvm and /dev/dri 



Freezer cgroup 

- sigSTOP 
- freeze entire container 
- move the container and unfreeze it 



Subtles

- top cgroup by systemd
- top level control 





# [Namespace](https://lwn.net/Articles/531114/) 

The purpose of each namespace is to wrap a particular global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource. It provides a group of processes with the illusion that they are the only processes on the system.



each process in one namespace of each kind  (similar to cgroup)

MNT NS `mnt_namespace`

- mount of file system 
- each user has /tmp
- `CLONE_NEWNS`



`unshare()` system call which allows calling process to be a part of the namespace.



`setns()`  this disaasoaciates one namespace to get attached to another namespace.



`struct nsproxy` in the `struct task_struct` holds information about process's namespaces. The pid namespace is an exception -- it's accessed using `task_active_pid_ns`.  The pid namespace here is the namespace that tasks's children will use. `/include/linux/nsproxy.h` for more code reading.









How mount works

- mount does check user permission who called the comand 
- kernel check if you have implementation for given filesystem on the device
- VFS abstracted call for that file system will get called `mount_bdev`
- superblock structure get created in memory by VFS. Fill in superblock structure by calling filesystem related functions.
- The specific filesystem fills the in-memory superblock with all the relevant information in disk (e.g., blocksize, what capabilities are available, check for corruption, etc..). Most of the times the in-memory superblock is very similar to the one in disk.





Various mount NS [use cases](https://www.ibm.com/developerworks/linux/library/l-mount-namespaces/index.html)

- per user mount
- mount sharing







UTS NS `uts_namespace`

- own hostname
- `CLONE_NEWUTS`





IPC NS `ipc_namespace`

- semaphore , shared memory and message queue
- `CLONE_NEWIPC`



PID NS `pid_namespace` `task_active_pid_ns` `CLONE_NEWPID`

- restrict PID
- One of the main benefits of PID namespaces is that containers can be migrated between hosts while keeping the same process IDs for the processes inside the container. 
- From the point of view of a particular PID namespace instance, a process has two PIDs: the PID inside the namespace, and the PID outside the namespace on the host system. PID namespaces can be nested: a process will have one PID for each of the layers of the hierarchy starting from the PID namespace in which it resides through to the root PID namespace. 





Network NS `CLONE_NEWNET`

- own Localhost
- iptables , routing rable , sockets 
- you can move network interface 
- various functions does skb mapping to particualr device namespace.
- `possible_net_t	 nd_net` in the `net_device` holds the namespace for that device.
- Usually network namespaces are managed by virutal network interface which present inside and outside of the container environment.
- physical hardware remains in the root namespace.





User namsespace `CLONE_NEWUSER`

- UID and GID namespace map 
- usable security 
- root inside the namespace



Namespace manipulation 

- clone() accept flag for NS 
- bind mount 



Copy-on-Write storage 

- layered file system 
- docker storage drivers 



SELinux/AppArmor inside docker 

what is [CAP_SYS_ADMIN](https://lwn.net/Articles/486306/)  



Container runtimes 

- LXC 
- Systemd-nspawn
- docker 
- runC - stripped down docker - uses libcontainer - just to start - live migration 



No depneds on NS and Cgrou 

- OpenVZ - actively maitained 
- Jails and zones - bsd and solaris 





