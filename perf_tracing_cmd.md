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






