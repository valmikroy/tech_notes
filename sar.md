# Notes around using SAR analysis 

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

Rough diagramatic representation on how kernel space proc scheduling happens 
![linux_prod_sched](images/linux_proc_sched.png)




