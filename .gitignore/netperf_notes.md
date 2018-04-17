Notes while reading netperf [guide](https://github.com/multipath-tcp/netperf/blob/master/doc/netperf.txt)

- `--enable-omni` provides new set of test suits , this is availble for version 2.5 onwards.
- `--enable-histogram` should provide histogram of output
- `--enable-demo` give `-D` to get interval based output like `iperf`
- service demand metrics in output, like `S.dem` 
```
The service demand is the normalization of CPU utilization and work
performed.  For a _STREAM test it is the microseconds of CPU time
consumed to transfer on KB (K == 1024) of data.  For a _RR test it is
the microseconds of CPU time consumed processing a single transaction.
For both CPU utilization and service demand, lower is better.

Service demand can be particularly useful when trying to gauge the
effect of a performance change.  It is essentially a measure of
efficiency, with smaller values being more efficient and thus "better."
```
- general syntax is `netperf <global> -- <test-specific>`
- `-a` and `-A` for send and recive buffer tuneups on local and remote systems.
- `-b` enabled burst ( `-enable-intervals` )
- `-B` for brief description
- `-c` and `-C`  with `<rate>` is to calculate CPU utilization with rate (which I am unclear about)
- `-ddddd` debug
- `-D <interval,units>` this is iperf style demo mode which takes intervals
- `-f G|M|K|g|m|k|x` output units 
- `-F <fillfile>' fillfile , file content will fill up buffer
- `-H <optionspec>'` give remote host name along with options like `-H linger,4` which will enable `linger` option with `ipv4`. Similar option availble for local client host with `-L`.
- `-I <optionspec>' confidence level
```
          -I 99,5
     asks netperf to be 99% confident that the measured mean values for
     throughput and CPU utilization are within +/- 2.5% of the "real"
     mean values.  If the `-i' option is specified and the `-I' option
     is omitted, the confidence defaults to 99% and the width to 5%
     (giving +/- 2.5%)
```


Example

```
          netperf -H tardy.cup -i 3 -I 99,5
          TCP STREAM TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to tardy.cup.hp.com (15.244.44.58) port 0 AF_INET : +/-2.5%  99% conf.
          !!! WARNING
          !!! Desired confidence was not achieved within the specified iterations.
          !!! This implies that there was variability in the test environment that
          !!! must be investigated before going further.
          !!! Confidence intervals: Throughput      :  6.8%
          !!!                       Local CPU util  :  0.0%
          !!!                       Remote CPU util :  0.0%

          Recv   Send    Send
          Socket Socket  Message  Elapsed
          Size   Size    Size     Time     Throughput
          bytes  bytes   bytes    secs.    10^6bits/sec

           32768  16384  16384    10.01      40.23
```           
In the example above we see that netperf did not meet the desired confidence intervals.  Instead of being 99% confident it was within +/- 2.5% of the real mean value of throughput it is only confident it was within +/-3.4%.  In this example, *increasing* the `-i' option (described below) and/or *increasing the iteration length* with the `-l' option might resolve the situation.


- `-i <sizespec>` This option enables the calculation of confidence intervals and sets the minimum and maximum number of iterations to run in attempting to achieve the desired confidence interval.

- `-j` This option instructs netperf to keep additional timing statistics when explicitly running an Omni Tests.

- `-o` and `-O` are offsets in buffersizes in local and remote systems.
- `-P` boolean flag to display test banner.

for all `_RR` type of tests, there is options to tune send and receive bufffers on local and remote systems with `-s` and `-S` respectively. How that affects `-a` and `-A` options which I could not find. 


