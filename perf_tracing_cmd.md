# Linux Kernel Perf analysis 

Laymans notes on Brendan Gregg's [perf post](http://www.brendangregg.com/perf.html).

### Installation 

I am using ubuntu based system on which `perf` comes as a part of package, in reality it is a part of [the kernel source](https://github.com/torvalds/linux/tree/master/tools/perf). At some point you will also need kernel debug symbols and kernel source code to be installed on the same host to do some advance experimentation.

- Install `perf`

  ```
  sudo apt-get install   linux-tools-`uname -r`
  ```

- Install kernel debug symbols on Ubuntu 

  ```
  echo "deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse
  deb http://ddebs.ubuntu.com $(lsb_release -cs)-updates main restricted universe multiverse
  deb http://ddebs.ubuntu.com $(lsb_release -cs)-proposed main restricted universe multiverse" | sudo tee -a /etc/apt/sources.list.d/ddebs.list
  
  
  sudo apt install ubuntu-dbgsym-keyring
  
  sudo apt-get update
  
  sudo apt-get install linux-image-$(uname -r)-dbgsym
  ```

- Install kernel source 

  ```
  # following command should usually work but you might want to do apt-cache search for linux-source and map for the version manually.
  
  sudo apt-get install linux-source-$(uname -r)
  
  # Kernel source gets dropped in /usr/src/linux-$(version) directory in the form of tar.bz2, you have to extract that source code from where your booted kernel got built.
  
  
  ```

- remove symbol restriction

  ```
  echo 0 | sudo tee /proc/sys/kernel/kptr_restrict
  ```

  
- install BCC tools in the mix 
```
# ubuntu focal 
sudo apt-get install -y bpfcc-tools

# other ubuntus 

sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 4052245BD4284CDD
echo "deb https://repo.iovisor.org/apt/$(lsb_release -cs) $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/iovisor.list
sudo apt-get update
sudo apt-get install bcc-tools libbcc-examples linux-headers-$(uname -r)


```


### Analysis 

Perf can act  on exported kernel function symbols which you can find in `/proc/kallsyms` and marked with `T` in front of it. Look for `man nm` to understand meaning of that notation.



Julia Evan's blog post about [tracing](https://jvns.ca/blog/2017/07/05/linux-tracing-systems/) does a good categorization in digramatic fashion about linux tracing systems. There are following type of tracing support 

-  kprobes is mechanism which injects assembly code in its runtime image and allow you to fetch various kinds of data. This is lightwight and can allow you to do production monitoring. You need to do some code reading to create a safe probing mechanism. Looks for simple kprobe injection [script.](https://github.com/brendangregg/perf-tools/blob/master/kernel/kprobe)
- tracepoint is the mechanism where provision for tracing of certain function of kernel is already written and it need to be externally activiated. Here is a [script](https://github.com/brendangregg/perf-tools/blob/master/kernel/functrace) in simialr fashion. In kernel, you can find trace events defined by  `TRACE_EVENT` macro. This also relies on ring buffer size which can be overrrun with too much of data.
- uprobe is simialr to kprobes but for userspace functions, especially `malloc()` in libc.
- UDST has something to do with DTrace and userspace probing which I did not explore, apparently this is possible with python and other scripting languages.
- LTTNG never even bothered to look up 

 

Above mechanisms can be used to collect data by using frameworks like perf, eBPF, ftrace, systemtap with tools like perf, bcc, ftrace, trace-cmd, kernelshark and many more.



  ### Perf

Perf is a mechansim which relies on system call `perf_event_open` is used to make kernel to write a data in ring buffer and read during this call execution. I am very unclear about the mechanism. You can inject trace, kprobes and uprobes through perf.



#### Tracing 

Here you can find trace points 

```
sudo perf list 2>&1 | grep Tracepoint
```

Here you can trace some of the events 

```
# list processes which are hitting trace point
sudo perf trace -e   kmem:kmalloc

# number of counts 
sudo perf stat -a -e   kmem:kmalloc -I 1000
```



#### Probing

This is also refered as Dynamic tracing where you attach a probe to particular kernel funciton 

```
# kernel probe
sudo perf probe --add clear_huge_page

# count of a probe got hit
sudo perf stat -a -e probe:clear_huge_page -I 10000

sudo perf probe --del=probe:clear_huge_page


# User probe
sudo perf probe --exec=/lib/x86_64-linux-gnu/libc.so.6 --add malloc

# print count of malloc calls
sudo perf stat -a -e probe_libc:malloc -I 10000

# trace system wide malloc 
sudo perf record -e probe_libc:malloc -a
sudo perf report -n


sudo perf probe --exec=/lib/x86_64-linux-gnu/libc.so.6  --del malloc

```



Here it get more interesting  where you can probe function definations if you have access to kernel source code on the same host

```
perf probe -V tcp_sendmsg
Available variables at tcp_sendmsg
        @<tcp_sendmsg+0>
                int     ret
                size_t  size
                struct msghdr*  msg
                struct sock*    sk

# probe for a size param value above  
sudo perf probe --add 'tcp_sendmsg size’
sudo perf trace  -e probe:tcp_sendmsg -a  (show you the process and the call to tcp_sendmsg)
sudo perf probe --del probe:tcp_sendmsg


You can also look at the place this symbol has exported 

$  perf probe -L tcp_sendmsg
<tcp_sendmsg@/build/linux-aws-9iwObr/linux-aws-5.4.0/net/ipv4/tcp.c:0>
      0  int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size)
      1  {
      2         int ret;


      4         lock_sock(sk);
      5         ret = tcp_sendmsg_locked(sk, msg, size);
      6         release_sock(sk);


      8         return ret;
         }
         EXPORT_SYMBOL(tcp_sendmsg);

```


Some interesting commands 

#### CPU cache hit rate 

```
$ sudo llcstat-bpfcc | egrep -e 'xlinpack|MISS'
PID      NAME             CPU     REFERENCE         MISS    HIT%
25368    xlinpack_xeon64  7          723800       446300  38.34%
25362    xlinpack_xeon64  1          769300       131500  82.91%
25376    xlinpack_xeon64  15         528600            0 100.00%
25367    xlinpack_xeon64  6          730600          200  99.97%
25363    xlinpack_xeon64  2           68400       394800   0.00%
25364    xlinpack_xeon64  3          773900       145700  81.17%
25365    xlinpack_xeon64  4         1120900       147500  86.84%
25373    xlinpack_xeon64  12        1006900          700  99.93%
25377    xlinpack_xeon64  16         737100            0 100.00%
25366    xlinpack_xeon64  5           63400       399600   0.00%
25371    xlinpack_xeon64  10         703600       148400  78.91%
25372    xlinpack_xeon64  11         657000          500  99.92%
25374    xlinpack_xeon64  13         632900            0 100.00%
25369    xlinpack_xeon64  8         1052900         1700  99.84%
25361    xlinpack_xeon64  18        1339700         1700  99.87%
25378    xlinpack_xeon64  17         546400            0 100.00%
25370    xlinpack_xeon64  9          796800       149000  81.30%
25375    xlinpack_xeon64  14         556000       148300  73.33%
25376    xlinpack_xeon64  33          35700          100  99.72%
```



#### Instructions per cycle 

Simple CPUs can process single instruction per cycle and if it is taking multiple cycles then is blocked on some external resources. Todays multiscalar processors can do more than 1 instruction per cycle.  This can be observed as a part of PMU perf stat 

```
$ sudo perf stat -C 10   -- sleep 5

 Performance counter stats for 'CPU(s) 10':

           5001.57 msec cpu-clock                 #    1.000 CPUs utilized
                 2      context-switches          #    0.000 K/sec
                 0      cpu-migrations            #    0.000 K/sec
                 0      page-faults               #    0.000 K/sec
       13579218554      cycles                    #    2.715 GHz
       32896171252      instructions              #    2.42  insn per cycle
         246551416      branches                  #   49.295 M/sec
            367615      branch-misses             #    0.15% of all branches

       5.001763686 seconds time elapsed

```



Running this stat with NOP cycles, you can find out max this CPU can do 

```
$ sudo perf stat -ddd ./noploop

 Performance counter stats for './noploop':

       1987.561576      task-clock (msec)         #    1.000 CPUs utilized
                 2      context-switches          #    0.001 K/sec
                 0      cpu-migrations            #    0.000 K/sec
                39      page-faults               #    0.020 K/sec
     5,037,506,873      cycles                    #    2.535 GHz                      (30.71%)
    20,043,572,014      instructions              #    3.98  insn per cycle           (38.56%)
        10,600,102      branches                  #    5.333 M/sec                    (38.76%)
            31,028      branch-misses             #    0.29% of all branches          (38.82%)
        21,060,842      L1-dcache-loads           #   10.596 M/sec                    (38.82%)
            37,500      L1-dcache-load-misses     #    0.18% of all L1-dcache hits    (38.70%)
             7,415      LLC-loads                 #    0.004 M/sec                    (30.65%)
             4,361      LLC-load-misses           #   58.81% of all LL-cache hits     (30.59%)
   <not supported>      L1-icache-loads
            42,357      L1-icache-load-misses                                         (30.59%)
        20,842,244      dTLB-loads                #   10.486 M/sec                    (30.59%)
             1,295      dTLB-load-misses          #    0.01% of all dTLB cache hits   (30.59%)
               562      iTLB-loads                #    0.283 K/sec                    (30.59%)
                 0      iTLB-load-misses          #    0.00% of all iTLB cache hits   (30.59%)
   <not supported>      L1-dcache-prefetches
   <not supported>      L1-dcache-prefetch-misses

       1.988068021 seconds time elapsed
```

And then push your processor to achieve that


#### Snoop syscall
Here is easy of way to snoop on `open()` syscall in realtime

```
sudo perf probe --add 'do_sys_open filename:string'
sudo perf record --no-buffering -e probe:do_sys_open -o - -a | PAGER=cat perf script -i -
sudo perf probe --del do_sys_open
```
So here is the `exec()` syscall to monitor small lived processes 

```
perf probe --add 'do_execve +0(+0(%si)):string +0(+8(%si)):string +0(+16(%si)):string +0(+24(%si)):string'
perf record --no-buffering -e probe:do_execve -a -o - | PAGER="cat -v" stdbuf -oL perf script -i -
perf probe --del do_execve
```





