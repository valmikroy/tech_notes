# SVE: Distributed Video Processing at Facebook Scale



[Article Link](http://www.qhuangcs.com/papers/sosp_sve.pdf)



- Directed Acyclic Graph (DAG) of the programmtic processing of video is something provide application programmers flexibiity. 

- Video needs verification and repairing like sync of an audio with video, skipped frames,  translation related verification, thumbnail extraction, facial recognition, video classification.

- continuously adapt bitrates as the network conditions

- Higher bitrates and less user-visible bu￿ering are desirable.

- Various  frameworks evaluated

  - Batch processing likes Map reduce, spark and Dryad  - this needs data in sequence which increses latency. Video frame processing can not be done in parallel unless you have large amount of frames buffered.
  - Stream processing like Storm, Spark - these are designed to work on continues queries and not discrete events.

- Mission - We also wanted a system with specialized **fault tolerance**, **overload control**, and **scheduling** that takes advantage of the unique characteristics of videos and our video workload.

- GOP based division of video on the player end helps parallelizing.

-  Write through cache for saving segments.

- video tracks - spatial image compression as well as temporal motion compression across a series of frames.

- Higher and lower priority queues from where job is getting fetched.

- Metadata read much frequently than it get read.

- Parellel procesisng of GOPed video chucks.

- taking advantage of Client side video processing  

- Device battery , availble bandwidth, 

- Segmenation is a trade off - more parallelization leads to less efficient compression because less number of frames for a reference.

- GOP params, distance between two I-frames and distance between two P (predictive) frames.

- preference between lower latency and the best compression ratio

- Constatnt frame rate and evenly distributed GOPs - most desirable 

- Variable frame rate can make things tricker 

- Quirks about audio starting at negative time value need to be adjusted.

- Fix misalign timestamp 

- Work fetching segments from cache instead of hitting storage

- After parallization - all segments and its components need to be glued together.

- DAG execution - group amorizes scheduling overhead.

- any system at scale, faults are not only inevitable but common. 

- Retry strategy for parallel tasks  and handle their failures and its monitoring.

- Client connectivity and buffering.

- Clinet Device side failures due to OS updates breaking video or audio codec.

- retrasmission of segments - that is the reason write through cache is useful.

- Sharding segments

- worker failures and direct correction with end to end latency 

- Sources of overloads 

  - Organic - ice bucket challenge 
  - Chaos engineering 
  - Bug induced overload 

- delaying latency-insensitive tasks during overload. Switch for FIFO to LIFO

- Map of video traffic to various datacenter and then its update in the event of falures. This affinity of video streams to dataceters to improve locality. 

- Live streaming 

  - througput does not matter 
  - parallel processing does not matter 
  - sequential fast procesisng is the only thing matters

- two types of workloads: streamed dedicated live encoding and batched full video processing.

- Linux namespaces for sandboxing 

  