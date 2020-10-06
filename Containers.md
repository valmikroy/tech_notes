# Containers 

What are the containers

Namspace and Cgroups mismash 



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





# Namespace 



each process in one namespace of each kind  (similar to cgroup)



PID NS

- restrict PID



Network NS

- own Localhost
- iptables , routing rable , sockets 
- you can move network interface 



MNT NS

- mount of file system 
- each user has /tmp



UTS NS

- own hostname



IPC NS

- semaphore , shared memory and message queue



User namsespace 

- UID and GID container map 
- usable security 
- root inside container 



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























