# Kernel Tracing

Ftrace is setup with debug fs, where you can start tracing functions with following steps

```
cd /sys/kernel/debug/tracing
echo function > current_tracer
echo do_page_fault > set_ftrace_filter
cat trace
```

You can achieve same with CLI frontend tool `trace-cmd`

```
sudo trace-cmd record -p function -l do_page_fault
sudo trace-cmd report
```

Tracer has few areas 

- `/sys/kernel/debug/tracing/available_events` 
  These are various events which happens inside kernel.
- `/sys/kernel/debug/tracing/available_filter_functions`  (Need to figure out need of this part)
- `/sys/kernel/debug/tracing/available_tracers` , this is type of tracers







### Kernel Function Tracing 

#### Tracing kernel function

```
sudo trace-cmd record -p function -l net_rx_action

sudo trace-cmd report
            sshd-1683  [001]  1071.332350: function:             net_rx_action
       trace-cmd-2836  [001]  1071.332562: function:             net_rx_action
          <idle>-0     [001]  1072.201165: function:             net_rx_action
          <idle>-0     [001]  1072.718105: function:             net_rx_action
            sshd-1683  [001]  1072.718340: function:             net_rx_action
       trace-cmd-2836  [001]  1072.718434: function:             net_rx_action

```

This is how you can figure out which process on which CPU is calling the function

You can run same command on given process 

```
sudo trace-cmd record -p function  -l net_rx_action  -P 1683

sudo trace-cmd report  
version = 6
CPU 0 is empty
cpus=2
            sshd-1683  [001]  1289.622913: function:             net_rx_action
            sshd-1683  [001]  1289.623100: function:             net_rx_action
            sshd-1683  [001]  1289.623346: function:             net_rx_action
            sshd-1683  [001]  1289.623599: function:             net_rx_action
            sshd-1683  [001]  1298.013018: function:             net_rx_action
```



#### Tracing with function graph 

```
sudo trace-cmd record -p function_graph   -P 1683

sudo trace-cmd report 




e1000_unmap_and_free_tx_resource.isra.45();
            sshd-1683  [001]  2005.132080: funcgraph_entry:                   |                                              e1000_unmap_and_free_tx_resource.isra.45() {
            sshd-1683  [001]  2005.132081: funcgraph_entry:                   |                                                __dev_kfree_skb_any() {
            sshd-1683  [001]  2005.132081: funcgraph_entry:                   |                                                  consume_skb() {
            sshd-1683  [001]  2005.132082: funcgraph_entry:                   |                                                    skb_release_all() {
            sshd-1683  [001]  2005.132082: funcgraph_entry:                   |                                                      skb_release_head_state() {
            sshd-1683  [001]  2005.132083: funcgraph_entry:                   |                                                        tcp_wfree() {
            sshd-1683  [001]  2005.132084: funcgraph_entry:        0.105 us   |                                                          sk_free();
            sshd-1683  [001]  2005.132084: funcgraph_exit:         1.222 us   |                                                        }
            sshd-1683  [001]  2005.132084: funcgraph_exit:         1.754 us   |                                                      }
            sshd-1683  [001]  2005.132084: funcgraph_entry:        0.339 us   |                                                      skb_release_data();
            sshd-1683  [001]  2005.132085: funcgraph_exit:         2.759 us   |                                                    }
            sshd-1683  [001]  2005.132085: funcgraph_entry:        0.051 us   |                                                    kfree_skbmem();
            sshd-1683  [001]  2005.132086: funcgraph_exit:         4.029 us   |                                                  }
            sshd-1683  [001]  2005.132086: funcgraph_exit:         4.385 us   |                                                }
            sshd-1683  [001]  2005.132086: funcgraph_exit:         5.263 us   |                                              }
            sshd-1683  [001]  2005.132088: funcgraph_entry:        1.156 us   |                                              e1000_clean_rx_irq();
            sshd-1683  [001]  2005.132090: funcgraph_entry:        0.283 us   |                                              e1000_update_itr();
            sshd-1683  [001]  2005.132091: funcgraph_entry:        0.047 us   |                                              e1000_update_itr();
            sshd-1683  [001]  2005.132091: funcgraph_entry:        0.086 us   |                                              napi_complete_done();
            sshd-1683  [001]  2005.132110: funcgraph_exit:       + 30.888 us  |                                            }
            sshd-1683  [001]  2005.132110: funcgraph_exit:       + 33.046 us  |                                          }
            sshd-1683  [001]  2005.132111: funcgraph_entry:        0.475 us   |                                          rcu_bh_qs();
            sshd-1683  [001]  2005.132112: funcgraph_entry:        0.038 us   |                                          __local_bh_enable();
            sshd-1683  [001]  2005.132112: funcgraph_exit:       + 35.683 us  |                                        }
            sshd-1683  [001]  2005.132112: funcgraph_exit:       + 36.975 us  |                                      }
            sshd-1683  [001]  2005.132113: funcgraph_exit:       + 37.544 us  |                                    }
            sshd-1683  [001]  2005.132113: funcgraph_exit:       ! 325.208 us |                                  }
            sshd-1683  [001]  2005.132113: funcgraph_exit:       ! 325.910 us |                                }
            sshd-1683  [001]  2005.132113: funcgraph_exit:       ! 326.474 us |                              }
            sshd-1683  [001]  2005.132114: funcgraph_exit:       ! 327.608 us |                            }
            sshd-1683  [001]  2005.132114: funcgraph_exit:       ! 329.819 us |                          }
            sshd-1683  [001]  2005.132114: funcgraph_exit:       ! 335.245 us |                        }
```



In above timing information 

- `+` indicates the function lasted longer than 10 microseconds
- `!` indicates the function lasted longer than 100 microseconds
- space (or nothing) - the function executed in less than 10 microseconds



Following command will provide list of kernel functions which can be trace

```
vagrant@ubuntu-xenial:~$ sudo trace-cmd list -f | grep ^skb_copy
skb_copy_ubufs
skb_copy_bits
skb_copy
skb_copy_expand
skb_copy_and_csum_bits
skb_copy_and_csum_dev
skb_copy_datagram_iter
skb_copy_datagram_from_iter
skb_copy_and_csum_datagram
skb_copy_and_csum_datagram_msg
```



### Kenel Event Tracing

There are events in kernel which does not map to particualr function calls and can be traced differently

Following way you can find events availble for tracing 

```
vagrant@ubuntu-xenial:~$ sudo cat /sys/kernel/debug/tracing/available_events | grep ^sched
sched:sched_wake_idle_without_ipi
sched:sched_swap_numa
sched:sched_stick_numa
sched:sched_move_numa
sched:sched_process_hang
sched:sched_pi_setprio
sched:sched_stat_runtime
sched:sched_stat_blocked
sched:sched_stat_iowait
sched:sched_stat_sleep
sched:sched_stat_wait
sched:sched_process_exec
sched:sched_process_fork
sched:sched_process_wait
sched:sched_wait_task
sched:sched_process_exit
sched:sched_process_free
sched:sched_migrate_task
sched:sched_switch
sched:sched_wakeup_new
sched:sched_wakeup
sched:sched_waking
sched:sched_kthread_stop_ret
sched:sched_kthread_stop
```



You can record kernel events like following 

```
sudo trace-cmd record -e 'sched:sched_switch'


sudo trace-cmd report | head -10
version = 6
cpus=2
       trace-cmd-2931  [001]  3171.776631: sched_switch:         trace-cmd:2931 [120] S ==> trace-cmd:2933 [120]
       trace-cmd-2932  [000]  3171.776783: sched_switch:         trace-cmd:2932 [120] S ==> swapper/0:0 [120]
          <idle>-0     [000]  3171.776795: sched_switch:         swapper/0:0 [120] R ==> trace-cmd:2932 [120]
       trace-cmd-2932  [000]  3171.776798: sched_switch:         trace-cmd:2932 [120] S ==> swapper/0:0 [120]
       trace-cmd-2933  [001]  3171.776840: sched_switch:         trace-cmd:2933 [120] S ==> swapper/1:0 [120]
          <idle>-0     [001]  3171.778542: sched_switch:         swapper/1:0 [120] R ==> rcu_sched:7 [120]
       rcu_sched-7     [001]  3171.778549: sched_switch:         rcu_sched:7 [120] S ==> swapper/1:0 [120]
          <idle>-0     [001]  3171.782964: sched_switch:         swapper/1:0 [120] R ==> rcu_sched:7 [120]

```



Like above you can also track migration of `ls` command 

```
sudo trace-cmd record -e 'sched_wakeup*' -e sched_switch -e 'sched_migrate*' ls


vagrant@ubuntu-xenial:~$ sudo trace-cmd report
version = 6
cpus=2
              ls-2952  [001]  3343.171802: sched_wakeup:         migration/1:12 [0] success=1 CPU:001
              ls-2952  [001]  3343.171813: sched_wakeup:         trace-cmd:2951 [120] success=1 CPU:001
              ls-2952  [001]  3343.171814: sched_switch:         trace-cmd:2952 [120] R ==> migration/1:12 [0]
     migration/1-12    [001]  3343.171816: sched_migrate_task:   comm=trace-cmd pid=2952 prio=120 orig_cpu=1 dest_cpu=0
     migration/1-12    [001]  3343.171823: sched_switch:         migration/1:12 [0] S ==> trace-cmd:2951 [120]
       trace-cmd-2951  [001]  3343.171829: sched_switch:         trace-cmd:2951 [120] S ==> swapper/1:0 [120]
          <idle>-0     [000]  3343.171864: sched_switch:         swapper/0:0 [120] R ==> trace-cmd:2952 [120]
              ls-2952  [000]  3343.171928: sched_switch:         trace-cmd:2952 [120] D|W ==> swapper/0:0 [120]
          <idle>-0     [001]  3343.171936: sched_wakeup:         trace-cmd:2950 [120] success=1 CPU:001
          <idle>-0     [001]  3343.171938: sched_switch:         swapper/1:0 [120] R ==> trace-cmd:2950 [120]
       trace-cmd-2950  [001]  3343.171944: sched_switch:         trace-cmd:2950 [120] S ==> swapper/1:0 [120]
          <idle>-0     [000]  3343.173992: sched_wakeup:         trace-cmd:2952 [120] success=1 CPU:000
          <idle>-0     [000]  3343.174009: sched_switch:         swapper/0:0 [120] R ==> trace-cmd:2952 [120]
              ls-2952  [000]  3343.174227: sched_switch:         ls:2952 [120] D|W ==> swapper/0:0 [120]
          <idle>-0     [001]  3343.174358: sched_wakeup:         rcu_sched:7 [120] success=1 CPU:001
          <idle>-0     [001]  3343.174366: sched_switch:         swapper/1:0 [120] R ==> rcu_sched:7 [120]
       rcu_sched-7     [001]  3343.174372: sched_switch:         rcu_sched:7 [120] S ==> swapper/1:0 [120]
          <idle>-0     [000]  3343.175044: sched_wakeup:         ls:2952 [120] success=1 CPU:000
          <idle>-0     [000]  3343.175062: sched_switch:         swapper/0:0 [120] R ==> ls:2952 [120]
          <idle>-0     [001]  3343.176360: sched_wakeup:         kworker/u4:0:6 [120] success=1 CPU:001
          <idle>-0     [001]  3343.176365: sched_switch:         swapper/1:0 [120] R ==> kworker/u4:0:6 [120]
    kworker/u4:0-6     [001]  3343.176371: sched_wakeup:         sshd:1683 [120] success=1 CPU:001
    kworker/u4:0-6     [001]  3343.176373: sched_switch:         kworker/u4:0:6 [120] S ==> sshd:1683 [120]
              ls-2952  [000]  3343.176392: sched_wakeup:         trace-cmd:2949 [120] success=1 CPU:000
              ls-2952  [000]  3343.176396: sched_switch:         ls:2952 [120] x ==> trace-cmd:2949 [120]

```









#### Documentation 

- https://www.kernel.org/doc/Documentation/trace/ftrace.txt
- https://jvns.ca/blog/2017/07/05/linux-tracing-systems/
- https://elinux.org/images/0/0c/Bird-LS-2009-Measuring-function-duration-with-ftrace.pdf
- https://static.lwn.net/images/conf/rtlws11/papers/proc/p02.pdf
- https://lwn.net/Articles/425583/
