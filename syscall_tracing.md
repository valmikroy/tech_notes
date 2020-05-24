Strace command to know how much time spent in which system call 

```
$ sudo strace   -c -f find / (you can replace this with PID)
<OUTPUT of the command>
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 40.23    3.557542           6    613413         1 write
 22.84    2.019646           4    540898           fcntl
 10.68    0.944289           4    238782           close
 10.04    0.887954           7    121824           newfstatat
  9.16    0.809689           7    120312           getdents
  4.28    0.378801           6     61544           openat
  2.76    0.244169           4     61556           fstat
  0.01    0.001220          11       108           brk
  0.00    0.000020           1        23           mmap
  0.00    0.000011           6         2           munmap
  0.00    0.000010           0        24        12 open
  0.00    0.000009           5         2           fstatfs
  0.00    0.000006           6         1           futex
  0.00    0.000000           0        10           read
  0.00    0.000000           0        14           mprotect
  0.00    0.000000           0         2           rt_sigaction
  0.00    0.000000           0         1           rt_sigprocmask
  0.00    0.000000           0         2           ioctl
  0.00    0.000000           0         8         8 access
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         1           uname
  0.00    0.000000           0         1           fchdir
  0.00    0.000000           0         1           getrlimit
  0.00    0.000000           0         2         2 statfs
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         1           set_robust_list
------ ----------- ----------- --------- --------- ----------------
100.00    8.843366               1758535        23 total
```



You can look for errors while doing system calls.



We have something strace similar tool for manage user space function 

```
sudo ltrace    -c -f find /

% time     seconds  usecs/call     calls      function
------ ----------- ----------- --------- --------------------
 50.32   42.448145          57    733693 free
 49.68   41.906170          57    733690 malloc
  0.00    0.000074          74         1 realloc
------ ----------- ----------- --------- --------------------
100.00   84.354389               1467384 total
```



When certain process is not performaning well

- check for CPU utilization, user space or kernel space 
- if it is kernel space then look for which systemcall is being expensive 
- Run these tracing only for non-prod process because it does SIGSTOP and SIGCONT to the process 


