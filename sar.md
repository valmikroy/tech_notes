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



## Memory 

```
$ sar -r ALL  1
Linux 5.4.0-1009-aws (ip-172-31-25-145)         05/26/20        _x86_64_        (8 CPU)

02:17:59    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty  kbanonpg    kbslab  kbkstack   kbpgtbl  kbvmused
02:18:00     31861692  31702868    705532      2.15     14888    144308   1786092      5.43    712164     78252       908    654312    160124      4480      4240     12372
02:18:01     31861692  31702868    705532      2.15     14888    144308   1786092      5.43    712164     78252       908    654320    160124      4480      4240     12372
02:18:02     31861692  31702868    705532      2.15     14888    144308   1786092      5.43    712164     78252       908    654320    160124      4480      4240     12372
02:18:03     31861684  31702860    705536      2.15     14888    144308   1786092      5.43    712408     78252       908    654584    160128      4480      4244     12372
02:18:04     31857612  31698812    709536      2.16     14888    144308   1786092      5.43    715028     77040       908    655880    160200      4512      4304     12468
02:18:05     31858700  31700016    708188      2.15     14888    144568   1786092      5.43    713696     77224       740    654492    160200      4480      4220     12428
02:18:06     31858708  31700024    708180      2.15     14888    144568   1786092      5.43    713696     77224       740    654492    160200      4480      4220     12428
02:18:07     31858708  31700024    708180      2.15     14888    144568   1786092      5.43    713696     77224       740    654492    160200      4480      4220     12428
02:18:08     31858708  31700024    708180      2.15     14888    144568   1786092      5.43    713696     77224       740    654492    160200      4480      4220     12428
02:18:09     31858708  31700024    708180      2.15     14888    144568   1786092      5.43    713696     77224       740    654492    160200      4480      4220     12428
02:18:10     31859260  31700604    707660      2.15     14888    144568   1786092      5.43    712688     76996       740    653200    160168      4448      4160     12412
02:18:11     31859260  31700604    707660      2.15     14888    144568   1786092      5.43    712688     76996       740    653200    160168      4448      4160     12412
02:18:12     31859260  31700604    707660      2.15     14888    144568   1786092      5.43    712688     76996       740    653200    160168      4448      4160     12412
^C

Average:     31859668  31700938    707350      2.15     14888    144468   1786092      5.43    713113     77474       805    654267    160170      4475      4219     12410
```


```
r$ sar -B 1
Linux 5.4.0-1009-aws (ip-172-31-25-145)         05/26/20        _x86_64_        (8 CPU)

02:21:32     pgpgin/s pgpgout/s   fault/s  majflt/s  pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff
02:21:33         0.00      0.00      5.00      0.00     13.00      0.00      0.00      0.00      0.00
02:21:34         0.00      0.00     64.00      0.00     17.00      0.00      0.00      0.00      0.00
02:21:35         0.00      0.00     29.00      0.00      6.00      0.00      0.00      0.00      0.00
02:21:36         0.00      0.00      2.00      0.00      6.00      0.00      0.00      0.00      0.00
02:21:37         0.00      0.00      2.00      0.00      6.00      0.00      0.00      0.00      0.00
02:21:38         0.00      0.00      1.00      0.00      6.00      0.00      0.00      0.00      0.00
02:21:39         0.00      0.00      2.00      0.00      6.00      0.00      0.00      0.00      0.00
02:21:40         0.00      0.00      2.00      0.00      6.00      0.00      0.00      0.00      0.00
02:21:41         0.00      0.00     54.00      0.00      9.00      0.00      0.00      0.00      0.00
^C

Average:         0.00      0.00     17.89      0.00      8.33      0.00      0.00      0.00      0.00
```


