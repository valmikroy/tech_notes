# Notes on Art of Application Performance Testing - Molyneaux







### Ch1 : Why Performance test?

- Performance test is neglected cousin of unit, functional and system testing.
- Performance is a feature which comes buit-in in various workflows. This makes connecting steps from one task to another task much faster along with the task itself. 
- How do you measure performance? First we need to have a DAG and then measure the intervals from one vertices to other.



Performance requirement of application

- Service oriented 
  - availbility 
  - resposnse time
- Efficiency oriented 
  - throughput 
  - utilization



Process to incorporate performance feature as a part of a release. In other words, "Performance awareness should be built into the application life cycle as early as possible."

Efficiency oriented performance requirements like throughput and utilization also affects capacity of the application.





### Ch2 : The fundamentals of Effective Application performance testing



Capacity is other side of the coin of Performance.



Performacne testing strategy 

- Tooling and Automation
- Performance environment  with mock data and various depedency services 
- Connected to release pipeline for suit of tests for every release 
- Performance targets , DAG of application queries with expectated latency numbers attached to it
- System level metrics monitoring KPIs



Broader division of Performance testing suit

- Scripting module 
- Test management module 
- Load injector
- Analysis module



Various factors to consider in the performance environment 

- Query reporduction 
- Production like traffic generation through log parsing 
- Transaction replay
- Network or other subsystem simulation for the issue.



Performance targets divided into following

- Availability or uptime
- Concurrency, scalability and throughput 
- Response time

And above affects capacity which can be measured in terms of system metrics 

- Network utilization 
- Server utilization 



Baseline and Test

THeoroticaly calcualtion for the latency response time. Apply queing theory 



Various test design

- baseline test
- load test
- stress test - push application to the breaking point 
- soak or stability test - running test for longer period of time (to discover slow memory leaks)
- smoke test - test which related to changed part of the application - make sure that change which is done is not causing a smoke coming out of the application 
- Isolation test - this test is intended to run various queries to idetify any problems based of prior experiance with the application.





### Ch3 : Process of performance testing 

This chapter highlights broader steps in which performance test process should implemented

- Pre-Engagement requirements capture
- Test enviornment build
- Transactions scripting 
- Performance test build
- Performance test excution 
- Post test, analyze results, report and retest



### Ch4: Effective RCA 

Statistical meaning 

- Standard distribution - median and standard deviation 
  ![Standard_deviation_diagram](https://upload.wikimedia.org/wikipedia/commons/8/8c/Standard_deviation_diagram.svg)

- Nth percetile 
- Response time or latency distribution - histogram 

Defination of load and what point of the load various KPIs started drifting into undesirable region.



- Application KPIs
- System resource KPIs
- Load generator progression 



Load test

- Increase load (with its defination)
- Select a KPI which shows the knee in its metrics numbers which is the indicator of the capacity maxing out.
- Overlay various metrics  including systems metrics to find out why that happened.
- Tune and rerun the tests again.









