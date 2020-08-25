





### Process



`include/linux/sched.h`

```C
struct task_struct { 

...

  // -1 unrunnable , 0 runnable , >0 stopped:
	volatile long 							state;
	
	// Current CPU:
	unsigned int 								cpu; 
	cpumask_t 									cpus_allowed;
	
	const struct sched_class 		*sched_class;
	struct sched_entity 				se; 
	int 												prio;
	
	pid_t 											pid;
	
	
	// Real parent process:
	struct task_struct __rcu 		*real_parent;
	struct list_head 						children;
	
	
	// Filesystem information:
	struct fs_struct						*fs;
	
	// Open file information:
	struct files_struct 				*files; 
	
	
	...
}
```

- Above get manged in doublly linked `task_list`

- When a process makes a `clone()` syscall, `_do_fork()` is executed due to system call definitions of clone in `kernel/fork.c.` call reached to `copy_process()` which does most of the intersting parts like 

  - error handling and the duplicate task with `dup_task_struct()`

  - scheduling specific work done by `sched_fork()`

  - based on the clone flag parent's resource get copied, this is important while forking for a thread because linux treats thread as a process with some additional resource sharing with the parent. Thread cloning flags

    ```
    clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND, 0);
    ```

  - `alloc_pid()` gives a pid to the process.

- `_do_fork()` wakes up newly created task with `wake_up_new_task()`

- Note about golang subroutine - they do not translate into OS threads.

  

  

#### Exercise 
- look at `pidstat` cheat sheets to know more information about process
- `execsnoop` is the only efficient way to track past spawned processes.
  

  





### Scheduling 

There are scheduling classes 

- **SCHED NORMAL** - normal process class
- **SCHED FIFO ** - this is the class which run in the real time priority without any time slice, that means it process with this class of scheduling runs till it is finished or interrupted by other process with the higher priority.
- **SCHED RR** - this is the real time priority with the time slice. This process gets preempted.



O(1) scheduler was the old scheduler where according to nice value attached to each process there was a fixed time slice given to process. It is assumed, that the scheduler assigns timeslices of 100ms to processes with default priority (nice value 0) and timeslices of 5ms to processes with the least priority (nice value 19). Process priorities between 0 and 19 are also mapped to fixed timeslices. This had a lot of inefficencies and got replaced in `2.6` kernel with CFS.



#### CFS 

This algorithm try to be fair with the scheduling prioirites of the task. Amount of time can be taken by a process depends on 

- niceness 
- amount of time spent by a process on the CPU

This also manages real-time priority tasks.



Configuration param `sched_latency_ns` which defines targeted preemption latency for CPU bound tasks. That means, it defines timeslice for each tasks based on number tasks in the run queue.

```
timeslice for the task = sched_latency_ns * (task's weight/total weight of tasks in the run queue)
```

There is a complimentry param `sched_min_granularity_ns` which governs minimum value of the `timeslice` of the task to reduce OS's preempting overheads.

When number of tasks becomes more than the ratio of `sched_latency_ns/sched_min_granularity_ns` then `sched_latency_ns`  value get replaced by ` sched_min_granularity_ns * total tasks`.



Once above timeslice is assigned to the process it get scheduled for a run on the CPU and accounted its CPU execution time as process's runtime. This process runtime is abstracted in the form of `vruntime` based on priority of the process. Higher priority process are recorded for lesser than their actual runtime on the CPU. This enabled them to have more CPU time.

`vruntime` is a part of the `sched_entity` struct, which itself is referenced in the `task_struct` of a process. This update is done by `update_curr` which get invovked periodically and adds value of `calc_delta_fair` to task's `curr->vruntime` in the weighted form.



```C
//Update the current task’s runtime statistics.
static void update_curr(struct cfs_rq *cfs_rq)
{
	struct sched_entity *curr = cfs_rq ->curr; 
  u64 now = rq_clock_task(rq_of(cfs_rq)); 
  u64 delta_exec;
...
  delta_exec = now - curr->exec_start; 
  curr->exec_start = now;
...
//statistics 
  schedstat_set(curr->statistics.exec_max, 
  max(delta_exec, curr->statistics.exec_max));

  curr->sum_exec_runtime += delta_exec; 
  schedstat_add(cfs_rq->exec_clock, delta_exec);
 
  curr->vruntime += calc_delta_fair(delta_exec, curr);     
  update_min_vruntime(cfs_rq);
}
```





Now, processes are sorted based on the `vruntime` in the form of red-black balanced binary tree. Any insert, update and delete can be made in O(log(n)). This tree's leftmost node is always with the least runtime which becomes the next runnable process. This happens with the `pick_next_task`.



IO bound tasks get removed from the above RB tree with the `dequeue_entity` while getting marked as a NONRUNNABLE. Task marks itself as sleeping and puts itself on the wait queue. , calls `schedule()` to select next process to run. When such tasks wakes up and get added back into the run queue, it will have vruntime setup to 0 which makes it most runnable task.





Preemption of the task can only happen when it is not holding any locks and kernel keeps track of the count of locks by the task in the form of `preempt_count`.  Task will only get preempted if that count is zero. Sometimes higher priority task sets up `need_resched` on currently blocked task to get rescheduled immidiately once it comes out of interrupt handler.





#### Exercise 

- where IO bound processes sleeps? the work queue ? How such queue is implemented. How such process get awaken.

- `vruntime` ??

- Scheduling related ebpf exercises

- confirm SCHD_FIFO behaviour 

- what are kernel watchdogs?

- `sched_migration_cost_ns` 

- this [doc](https://doc.opensuse.org/documentation/leap/archive/42.1/tuning/html/book.sle.tuning/cha.tuning.taskscheduler.html)

- which file `update_curr` belongs to.

  

