# Memory Specification read

Memory timing has been represented in 4 numbers CL - tRCD - tRP - tRAS.

#### clock

Before jumping into above number,  here is briefing about memory clock speed which is repsented in form of PC4-2666 or PC2-2400, where PC4 is equivalent of DDR4 which represents generation of DDR. In DDR PC4-2666, memory running at 1333 MHz frequency `2666/2=1333` . So each cycle for PC4-2666 would be `(10^9)/(1333*10^6)=.75018754688672168042` which is 0.75ns per each cycle.

All four timing numbers are repsented in terms of cycles and you can calculate actual latency by doing multiplication with each clock cycle timing.

Memory chips are arrange in form of rows and coloumn, it needs 

#### CL - CAS Latency

 Cycles required for first bit to read when memory row is already open.

#### tRCD - Row address to Column address Delay

Minimum clock cycle required to open 

#### Memory Bandwidth 
`Theorotical BW calcualtion = Speed  GT/s * 8 bytes per channel * Number of Channels * Number of Sockets`


 
