# Scaling Memcache at Facebook

[Article link](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf)

Notes on learning

- It is facinating to see how simple key-value store has been utilized and fitted in Facebook's massive scale. This phrase describes it well **demand-filled look-aside cache**. 
- `deletes` are idempotent compared to `updates`.
- This memcache separates caching layer from persistant data layer. It protects that layer from high volume of reads.
- Every user query gets translated in to memcache queries has a fanout. This fanout is managed by creating a DAG based on interdependecies between results which are returned from each memcache query.
- Items are distributed across memcache cluster through consistant hashing.
- Read queries to memcache are done over UDP and  updates going through TCP.
- This entire Memcache infra accessed through either a client or set of library calls. Client manages lot of request routing, compression, error handling and request batching part.
- connection coalescing also getting done while querying.
- There is TCP like sliding window mechanism is getting used to make queries to memcached. This pending queue is good reflection of state of memcache cluster performance. [Little's Law](https://www.process.st/littles-law/#:~:text=Put%20simply%2C%20Little's%20law%20is,time%20spent%20in%20the%20queue")
- Learned about [Poisson arrival process](https://preshing.com/20111007/how-to-generate-random-timings-for-a-poisson-process/).
- Product decision about serving slighly stale data.
- For scalability purpose, various pools of memcache were created based on access frequencies of certain keys. Gutter pool was particualrly interesting to prevent cascading failures.
- Various performance improvement by updating various lockings and updates in slab allocation
- Metrics which are getting measured to observe performance based on 
  - User request to memcache Fanout of requests
  - Response time
  - Latency
- Interesting graphs on percentile of user requests on Y axis against various other metrics like distinct memcached servers or Bytes collected from memcached servers on the X-axis.
- Lessons learned
  - improve monitoring, debugging and operational efficiency along with the performance.
  -  Separate cache from persistant storage to make it scale.
  - Operations on stateful components is much complex than stateless one.
  - Simplicty is vital.

