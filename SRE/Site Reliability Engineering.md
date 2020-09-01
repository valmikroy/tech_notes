# Site Reliability Engineering

Who is SRE? Software development engineers managing operations.

Devops and SRE falls into the same category.

An SRE team is responsible for the 

#### availability, 

- Time based availability 

![img](assets/a3KcNq4YhUaJjp2HXuhNTap3CqAk-3s_NOZILk4hhqe0p4q99fP7mgOjrqOhl2aJniSerd_jT8wv0PH9kGtkucgRHP7D1FWBm5hBpA=s0.png)

- Aggregate availability 
  ![img](assets/IBFAixHY-GpJm8DW8EL9yKCaKiRdv_-3lHtYonZy40NLio19oPiLWnT24w9U45LSTb02-OYmrqtD2PLRy2ujoQcb81MlSTtjp8qYYTw=s0.png)



latency,

performance, 

efficiency, 

change management, 

monitoring, 

emergency response, 

and capacity planning of their service(s). 







Embracing risk

- Risk tolerance 
- Error budget 



- SLI - service level indicator 

- SLO - service level objective  - a target value or range of values for a service level that is measured by an SLI.

 lower bound ≤ SLI ≤ upper bound. 

- , SLAs are service level *agreements*: an explicit or implicit contract with your users that includes consequences of meeting (or missing) the SLOs they contain. 



possible indicators

- User-facing serving systems, such as the Shakespeare search frontends, generally care about availability, latency, and throughput
- Storage systems often emphasize latency, availability, and durability.
- Big data systems, such as data processing pipelines, tend to care about throughput and end-to-end latency.
- All systems should care about correctness: was the right answer returned, the right data retrieved, the right analysis done?



have few SLOs, perfection can wait, avoid absolutes.







Elimination of Toil 

- Toil is ok but it stangnats the progress and should be automated.
- O(n) growth 

Spotting toil 

- through process runbooks
- through oncall pages 



Monitoring 

- Whitebox monitoring  - internal service stats , logs ,  
- Blackbox monitoring - User side metrics 



Alert 

- page should be above noval problem 
- they should be actionable without know a root cause 

RCA - outages 



Long terms objectives

- trend analysis 
- retrospective and debugging



Golden metrics

- Latency 
- Traffic 
- Errors
- Saturation ( function of above three )



Performance and instrumentation is tail end of the work when comes to optimization of your system.



Automation 

- what to automate? how to identify 



Release engineering 



​	















​	



