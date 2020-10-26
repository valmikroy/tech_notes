# System Design CDN

#### Why are we doing this?

- Provide customer delightful video streaming experience.
- Support rapid customer base.
- Protect our infrastructure due to this growth



#### How

- What kind of cache we have?
  - user generated content?
  - copyrighted content?
  - Object size of the content??
- Cache 
   - always has predication
   - always has eviction possibity 
- delighful?
  - low video latency
  - built in resiliency 
- Absorb rapid increase
  - more content to be cached 
  - cache need to be up to date with newer content
  - multi region CDN with popular content by region
- Protect infrastruture 
   - Create technological buffer against spikes.
   - Protect video content if any - DRM?
- - 



#### What



##### Cache

- 
- Bring cache closer to the user? 
- Populate cache with popular content?

  - What is the possible size of popular content? (Memory)
  - How much network capacity needed at the edge? (Network)
  - Assuming we are not IO bound otherwise we will need connection handling capacity? (CPU)
- We will provision some capacity for the undiscover items.
- Cache warming infrastructure and sophasticated algorithms 

##### Load handling
- BGP 

- L3 DSR

- L7 proxy - connection handling and TLS termination. 

- Failure Isn't an Option, It's a Fact

- Here is [BGP ECMP](https://blog.cloudflare.com/cloudflares-architecture-eliminating-single-p/) routing example

- One level of cache near to the user. - fat cache host with the storage attached to it. which will servrve multiple edge hosts. This is useful when object size whichis cached is much larger.

- Gradual Load shifting on the origin side.

- Inbound traffic is much smaller than outbound traffic - DSR - Direct Server Return

- Client IP infromation doesnt get lost in DSR

- Need for L4 router 

  ```
  The second tier is the glue between the stateless world of IP routers and the stateful land of L7 load-balancing. It is implemented with L4 load-balancing. The terminology can be a bit confusing here: this tier routes IP datagrams (no TCP termination) but the scheduler uses both destination IP and port to choose an available L7 load-balancer. The purpose of this tier is to ensure all members take the same scheduling decision for an incoming packet.
  ```

  

Load balancing tiers

- Tier1 DNS BGP

- Tier2 L4DSR/L3DSR
- Tier3 L7 

Load is a function of the volume and the average cost of requests to the system, so rate limiting is important under backend slowness.





Nitty grities 

- Randomize DNS name to do cache eviction.





Division of the layers

- Physcal bare metal 
- Platform on top of it  - OS and containerization capabilities 
- Various services on the plarform





Service infra 

- Stateless services
  - failover 
  - distribution of information of the state of the backend
- Stateful services
  - replication 
  - routing to the data
- efficent cache which - Trasiant state
  - warming up the cache with predications?
  - managing eviction policies 



  

No single point of failure. 

Equipement failure is one thing - Who will protect against human failure? Process + Automation will do that.



Traffic doesnt move even after failover

Traffic oscillations 







Latency numbers 

- TCP handshake (1xRTT)
- TLS connection (2xRTT) for resumed connection with TLS session management (1xRTT)
- HTTP (1xRTT)

