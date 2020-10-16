# Facebook Taiji

Notes on this [article](https://research.fb.com/publications/taiji-managing-global-user-traffic-for-large-scale-internet-services-at-the-edge/) 

- Connection aware routing - group traffic of highly connected users to the same datacenter. This creates bucketization of user. This also has product as an additional dimension.
- User based grouping also increases proabability of cache hit rate as users will be sharing same content.
- Bipartite architecture - which has few datacenters with on-ramp and off-ramp across the globe to route traffic there.
- Design pillers for Taiji
  - Capacity crunch -  spillover traffic due to surges or deployments
  - Product heterogeneity - different products with sticky or stateless conenctions
  - Hardware heterogeneity  - different type of SKUs spreaded across the data center
  - Fault tolerance  - handle any kind of datacenter failures 
- Constraint optimization solver - linear equations with the constraint solver
- L4 and L7 LBs in the onramp and offramp locations.
- offramp will make connection to reach to webserver which then queries various microservices to get the response.
- some chaos monkey like system simulates worst case scenario.
- Ebb and flow of diurnal traffic patterns.
- Two components 
  - Runtime - this generates routing table based on various load feedback
  - Traffic pipeline - this takes the routing table and apply user buckting logic for efficient routing. 
- load feedback 
  - request per second - stateless 
  - sticky traffic - user sessions
  - hardware utilization - MIPS counter to normalize heterogeneous processor architectures.
- Edge to datacenter latency optimization for routing table.
- Optimal solver using mixed integer programming.
- Safegurds for traffic shift - it should do it gradually which will help to warm up the cache. Make sure smooth transition of traffic.
- Doing user based routing requires edge to hold large hash table into memory to hold the mapping between user and datacenter.
- BucketID in the cookie
- hermetic test environment
- Configuration generation at the edge LB level using zookeeper and in case of failure use last known good configuration.
- Site events - failure mitigation where datacenters are explicitely marked as `DOWN`
- Chaos traffic should not trigger traffic shift to certain level.
- First and second level derivate on the request count to get rps and acceleration in rps to provide load feedback earlier. This can be done after cubic spline interpolation curve fitting.
- Criterias based on latency (latency aware balancing)
  - utilization of web frontends
  - optimizing RTT for user
  - handling failures in the path seamlessly 
- Lessons 
  - Customize load balacing is key managing infra unlization 
  - Build system to cope up with infrastrucutre evaluation 
  - Keep debugging in mind 

