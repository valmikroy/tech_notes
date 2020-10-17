# Linux Block Layer

Understanding extracted from Neil Brown's LWN [article](https://lwn.net/Articles/736534/) 

![[Block layer diagram]](/Users/abhsawan/git/tech_notes/images/lwn-neil-blocklayer.png)

- Block devices maps to `S_IFBLK` inodes in the kernel which keeps track of major and minor numbers. Internally `i_bdev` field in the inode is mapped to [`struct block_device`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/fs.h?h=v4.13#n415) which maps to `block_device->bd_inode` which involved in actually IO of the device. 
  (what other type of inodes are there and what about network socket inodes?)

- `block_device->bd_inode` provides page cache implementation defined in  [`fs/block_dev.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/block_dev.c?h=v4.13), [`fs/buffer.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/buffer.c?h=v4.13).

- Block IO involves `read`, `write` and nowadays `discard`.

- All block devices in Linux are represented by [`struct gendisk`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/genhd.h?h=v4.13#n171) — a "generic disk" which can be associated with multiple block devices which are part of the same physical device with logical partitioning.

- A  data structure ([`struct bio`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/blk_types.h?h=v4.14-rc1#n46)) that carries read and write requests and other control requests, from the `block_device` (previously  `struct gendisk` ).   [`generic_make_request()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-core.c?h=v4.13#n2114) or, equivalently, [`submit_bio()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-core.c?h=v4.13#n2228) get called to submit `bio`. Both of these calls can be blocked for short time.

- Above `submit_bio()` or `generic_make_request()` calls function pointed `make_request_fn()` setup by `blk_queue_make_request()` during registration time.

- Recursion Avoidance

  - `md` used for software RAID and `dm` used for LVM which can stack up the devices.
  - to avoid recursion in the request due to above stacking `current->bio_list` get attached to  in the [`struct task_struct`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/sched.h?h=v4.13#n519) for the current process. 
  - there is get split with the help of  [`bio_split()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/bio.c?h=v4.13#n1843) and [`bio_chain()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/bio.c?h=v4.13#n326) functions and then one half of it submitted in time to   [`generic_make_request()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-core.c?h=v4.13#n2114).
  - `mempool` is used to allocate bios.

- Device queue plugging

  - so it can be more efficient to gather a batch of requests together and submit them as a unit.
  - plugging happens only to an empty queue. 
  - plugging happens when `schedule()` is called on the process which gives it enough async opportunity to batch all `bio`s.
  - With per-process plugging, it is often possible to create a per-process list of `bio`s, and then take the spinlock just once to merge them all into the common queue.

- It also performs some other simple tasks such as updating the `pgpgin` and `pgpgout` statistics in `/proc/vmstat`.

- Merged `bio`s are held togethere as a part of [`struct request`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/blkdev.h?h=v4.13#n134) and this structure sits in the queue defined by  [`struct request_queue`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/blkdev.h?h=v4.13#n386) for each device represeted by  [`struct gendisk`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/genhd.h?h=v4.13#n171). Various part of above structures is exposed in `/sys/block/*/queue/` sysfs directories.

- All these requests are either sit in single queue or multiple queues depedning on support provided by underlying hardware.

- Single queues are designed orginally for spinning disk which had a seek time as queue component of access latency.

- A driver registers a `request_fn()` using `blk_init_queue_node()` and this function is called whenever there are new requests on the queue that can usefully be processed. 

-   Single-queue schedulers

  - `noop` - does [simple sorting](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/elevator.c?h=v4.13#n371) where never allowing a read to be moved ahead of a write or vice-versa. And do first in first out queuing.
  - `deadline` - This algorithm tries to put an upper limit on the delay imposed on any request while retaining the possibility of forming large batches.
  - `cfq` - more complex, It aims to provide fairness between different processes, and, when configured with control groups, between different groups of processes as well. It has queue for each process (and/or group of processes) with timeslice assigned to it like process scheduling queue.

- Technique of `tagging` get used to tag a request before sending it to hardware device and notification is sent back to software layer upon the completion of request.

- Multiple-queue 

  - This benefitted largly when hardware can support multiple read/write/other operations in parallel.

  - two layers of queue exists, one in the software staging layer and another in hardware layer govern by device driver.

  - software layer queues are defined per-CPU and/or per-NUMA node basis. So merging can happen on these levels. Represented by  ([`struct blk_mq_ctx`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-mq.h?h=v4.13#n8)) 

  - hardare dispatch queues are allocated by on device driver layer represented by  ([`struct blk_mq_hw_ctx`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/blk-mq.h?h=v4.13#n10)). 

  - In this setup, tagging is used to track the request.

  - The pluggable multi-queue schedulers have [22 entry points](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/elevator.h?h=v4.13#n95), used by insert_requests() and dispatch_request()

  - along with mq-noop and mq-deadline, bfq is Budget Fair Queueing has been introduced.

    

