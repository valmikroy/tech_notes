#Linux TX Path 



### Ground work for sendto() and sendmsg()



```c
sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
```



Protocol family like `AF_INET` has a registration routine setup by `inet_init`  in `./net/ipv4/af_inet.c` for each protocol like TCP, UDP, ICMP, and RAW.



`inet_create` function takes argument to above `socket` call and search for registred protocol.

```C
static const struct net_proto_family inet_family_ops = {
        .family = PF_INET,
        .create = inet_create, /* registration */ 
        .owner  = THIS_MODULE,
};
```



This `inet_create` has a loop to get operation functions for given protocol. 

```C
/* Look for the requested type/protocol pair. */
lookup_protocol:
        err = -ESOCKTNOSUPPORT;
        rcu_read_lock();
        list_for_each_entry_rcu(answer, &inetsw[sock->type], list) {

                err = 0;
                /* Check the non-wild match. */
                if (protocol == answer->protocol) {
                        if (protocol != IPPROTO_IP)
                                break;
                } else {
                        /* Check for the two wild cases. */
                        if (IPPROTO_IP == protocol) {
                                protocol = answer->protocol;
                                break;
                        }
                        if (IPPROTO_IP == answer->protocol)
                                break;
                }
                err = -EPROTONOSUPPORT;
        }


/* way later in the code */

sock->ops = answer->ops;
```



`inet_protosw`  struct holds ops function pointers for each protocol in the given family



```C
/* Upon startup we insert all the elements in inetsw_array[] into
 * the linked list inetsw.
 */
static struct inet_protosw inetsw_array[] =
{
        {
                .type =       SOCK_STREAM,
                .protocol =   IPPROTO_TCP,
                .prot =       &tcp_prot,
                .ops =        &inet_stream_ops,
                .no_check =   0,
                .flags =      INET_PROTOSW_PERMANENT |
                              INET_PROTOSW_ICSK,
        },

        {
                .type =       SOCK_DGRAM,
                .protocol =   IPPROTO_UDP,
                .prot =       &udp_prot,          /* protocol specific functions */
                .ops =        &inet_dgram_ops,   /* INET function to handle dgram */
                .no_check =   UDP_CSUM_DEFAULT,
                .flags =      INET_PROTOSW_PERMANENT,
       },

			/* .... more protocols ... */
```



INET DGRAM related functions are here 

```C
const struct proto_ops inet_dgram_ops = {
  .family		   = PF_INET,
  .owner		   = THIS_MODULE,

  /* ... */

  .sendmsg	   = inet_sendmsg,
  .recvmsg	   = inet_recvmsg,

  /* ... */
};
EXPORT_SYMBOL(inet_dgram_ops);
```



And UDP protocol specific functions defined with `struct proto udp_prot`  in `./net/ipv4/udp.c`



```C
struct proto udp_prot = {
  .name		   = "UDP",
  .owner		   = THIS_MODULE,

  /* ... */

  .sendmsg	   = udp_sendmsg,
  .recvmsg	   = udp_recvmsg,

  /* ... */
};
EXPORT_SYMBOL(udp_prot);
```





### Socket system call layer

Operations for socket happens through system calls like `sendto`

```C
ret = sendto(socket, buffer, buflen, 0, &dest, sizeof(dest));
```

and operations for `sendto` system calls is attached in `./net/socket.c`

```C
/*
 *      Send a datagram to a given address. We move the address into kernel
 *      space and check the user space data area is readable before invoking
 *      the protocol.
 */

SYSCALL_DEFINE6(sendto, int, fd, void __user *, buff, size_t, len,
                unsigned int, flags, struct sockaddr __user *, addr,
                int, addr_len)
{
	/*  ... code ... */

	err = sock_sendmsg(sock, &msg, len);

	/* ... code  ... */
}
```



So chain of calls in the userspace will for following chain

- sendto                   
- sock_sendmsg       
- inet_sendmsg        
- udp_sendmsg       

after this it get passed on to lower protocols. System call layer where destination address get formatted in big or little endian to adjust with network order.  



`sock_sendmsg` will start chain of calls like following 

-  sock_sendmsg

- __sock_sendmsg

- __sock_sendmsg_nosec

  ```C
  static inline int __sock_sendmsg_nosec(struct kiocb *iocb, struct socket *sock,
                                         struct msghdr *msg, size_t size)
  {
          struct sock_iocb *si =  ....
  
  				/* other code ... */
  
          return sock->ops->sendmsg(iocb, sock, msg, size);  /* call to inet_sendmsg */
  }
  ```

- inet_sendmsg 
  This function manages `AF_INET` protocol family and at this level it records the last CPU on which this flow was processed using `sock_rps_record_flow`.

  ```C
  int inet_sendmsg(struct kiocb *iocb, struct socket *sock, struct msghdr *msg,
                   size_t size)
  {
    struct sock *sk = sock->sk;
  
    sock_rps_record_flow(sk);
  
    /* We may need to bind the socket. */
    if (!inet_sk(sk)->inet_num && !sk->sk_prot->no_autobind &&
        inet_autobind(sk))
            return -EAGAIN;
  
    return sk->sk_prot->sendmsg(iocb, sk, msg, size); /* call to udp_sendmsg */
  }
  EXPORT_SYMBOL(inet_sendmsg);
  ```

  

   

### UDP protocol layer



##### Corking

- Corking allows a user program request that the kernel accumulate data from multiple calls to `send` into a single datagram before sending. 

- Can be enabled 

  - with `UDP_CORK` in the socket option
  - with `MSG_MORE` flag to `send`, `sendto`, or `sendmsg` 

- Code block with checks `up->pending` inside `udp_sendmsg` for corking

  ```C
  int udp_sendmsg(struct kiocb *iocb, struct sock *sk, struct msghdr *msg,
                  size_t len)
  {
  
  	/* variables and error checking ... */
  
    fl4 = &inet->cork.fl.u.ip4;
    if (up->pending) {
            /*
             * There are pending frames.
             * The socket lock must be held while it's corked.
             */
            lock_sock(sk);
            if (likely(up->pending)) {
                    if (unlikely(up->pending != AF_INET)) {
                            release_sock(sk);
                            return -EINVAL;
                    }
                    goto do_append_data;
            }
            release_sock(sk);
    }
  ```

  



##### Destination address and port 

- either socket already has destination address determined from previous calls.
- or it is passed as auxilary structure with `sendto(socket, buffer, buflen, 0, &dest, sizeof(dest));`



in udp_sendmsg()

```C
  /*
   *      Get and verify the address.
   */
  if (msg->msg_name) { /* address parsing provided by send() &dest structure */
          struct sockaddr_in *usin = (struct sockaddr_in *)msg->msg_name;
          if (msg->msg_namelen < sizeof(*usin))
                  return -EINVAL;
          if (usin->sin_family != AF_INET) {
                  if (usin->sin_family != AF_UNSPEC)
                          return -EAFNOSUPPORT;
          }

          daddr = usin->sin_addr.s_addr;
          dport = usin->sin_port;
          if (dport == 0)
                  return -EINVAL;
  } else { /* address fetched from the socket */
          if (sk->sk_state != TCP_ESTABLISHED)
                  return -EDESTADDRREQ;
          daddr = inet->inet_daddr;
          dport = inet->inet_dport;
          /* Open fast path for connected socket.
             Route will not be used, if at least one option is set.
           */
          connected = 1;
  }
```



##### Hooks for IP options handling 

###### Ancillary messages via `sendmsg` -> `ip_cmsg_send`

UDP layer gets some hooks to modify IP layer information through these Ancillary messages. Some examples `IP_PKTINFO`, `IP_TTL`, `IP_TOS`.  NOTE, these values are handled `per packet` basis but they can be handled on the socket basis so can be applied to all packets.

in udp_sendmsg()

```C
if (msg->msg_controllen) {
        err = ip_cmsg_send(sock_net(sk), msg, &ipc,
                           sk->sk_family == AF_INET6);
        if (err)
                return err;
        if (ipc.opt)
                free = 1;
        connected = 0;
}
```

`ip_cmsg_send` handles thoese ancillary messages from `./net/ipv4/ip_sockglue.c`

###### Packet vs Socker IP options

Selection of IP options 

- for that specific packet passed through systemcall arguments
- in those arguments are not present then through socket options
  in udp_sendmsg()

```C
if (!ipc.opt) {
        struct ip_options_rcu *inet_opt;

        rcu_read_lock();
        inet_opt = rcu_dereference(inet->inet_opt);
        if (inet_opt) {
                memcpy(&opt_copy, inet_opt,
                       sizeof(*inet_opt) + inet_opt->opt.optlen);
                ipc.opt = &opt_copy.opt;
        }
        rcu_read_unlock();
}
```



###### source record route  and other IP options

in udp_sendmsg()

```C
ipc.addr = faddr = daddr;

if (ipc.opt && ipc.opt->opt.srr) {
        if (!daddr)
                return -EINVAL;
        faddr = ipc.opt->opt.faddr;
        connected = 0;
}

	/* ... code  ... */

tos = get_rttos(&ipc, inet);
if (sock_flag(sk, SOCK_LOCALROUTE) ||
    (msg->msg_flags & MSG_DONTROUTE) ||
    (ipc.opt && ipc.opt->opt.is_strictroute)) {
        tos |= RTO_ONLINK;
        connected = 0;
}


```



###### Multicast src and dst address determination

If multicast then find outgoing interface and source address for the packet.  Outgoing interface can be overriden by `IP_PKTINFO`

in udp_sendmsg()

```C
if (ipv4_is_multicast(daddr)) {
        if (!ipc.oif)
                ipc.oif = inet->mc_index;
        if (!saddr)
                saddr = inet->mc_addr;
        connected = 0;
} else if (!ipc.oif)
        ipc.oif = inet->uc_index;
```



##### Routing 

###### Routing entry check

If socket connected then check `sk_dst_check` 

```C
if (connected)
        rt = (struct rtable *)sk_dst_check(sk, 0);
```

If socket is not connected or `sk_dst_check` returns negative then start building routing entry 

```C
if (rt == NULL) {
        struct net *net = sock_net(sk);

        fl4 = &fl4_stack;
        flowi4_init_output(fl4, ipc.oif, sk->sk_mark, tos,
                           RT_SCOPE_UNIVERSE, sk->sk_protocol,
                           inet_sk_flowi_flags(sk)|FLOWI_FLAG_CAN_SLEEP,
                           faddr, saddr, dport, inet->inet_sport);

	/* ... code  ... */



/*  above flow structured passed through SELinux to fetch security ID  */ 
security_sk_classify_flow(sk, flowi4_to_flowi(fl4));

/* create a route entry */ 
rt = ip_route_output_flow(net, fl4, sk);

/* Error condition while generating route */   
  
if (IS_ERR(rt)) {
  err = PTR_ERR(rt);
  rt = NULL;
  if (err == -ENETUNREACH)
    IP_INC_STATS(net, IPSTATS_MIB_OUTNOROUTES);
  goto out;
}  
  
```

`IPSTATS_MIB_OUTNOROUTES` updates `OutNoRoutes` entry in following stat file.

```shell
/proc/net/snmp

Ip: Forwarding DefaultTTL InReceives InHdrErrors InAddrErrors ForwDatagrams InUnknownProtos InDiscards InDelivers OutRequests OutDiscards OutNoRoutes ReasmTimeout ReasmReqds ReasmOKs ReasmFails FragOKs FragFails FragCreates
Ip: 1 64 25922988125 0 0 15771700 0 0 25898327616 22789396404 12987882 51 1 10129840 2196520 1 0 0 0
```



###### Broadcast check 

```C
err = -EACCES;
if ((rt->rt_flags & RTCF_BROADCAST) &&
    !sock_flag(sk, SOCK_BROADCAST))
        goto out;
if (connected)
        sk_dst_set(sk, dst_clone(&rt->dst));

```



###### MSG_CONFIRM and ARP cache

This is a check to make sure that you fetch ARP entry from the cache and let it not go stale by setting up a flag.

```C
if (msg->msg_flags&MSG_CONFIRM)
          goto do_confirm;
back_from_confirm:

	/* ... code  ... */

do_confirm:
        dst_confirm(&rt->dst);
        if (!(msg->msg_flags&MSG_PROBE) || len)
                goto back_from_confirm;
        err = 0;
        goto out;
```



##### Prepare data for transmit (skb creation)

If socket is not corked then 

`ip_make_skb` -> `ip_setup_cork`->`__ip_append_data` ->`__ip_make_skb`  forms an skb

else 

`ip_append_data`->`__ip_append_data` -> `__ip_make_skb` forms an skb



###### Lockless fast path for the non-corking case (udp_sendmsg)

```C
if (!corkreq) {
        skb = ip_make_skb(sk, fl4, getfrag, msg->msg_iov, ulen,
                          sizeof(struct udphdr), &ipc, &rt,
                          msg->msg_flags);
        err = PTR_ERR(skb);
        if (!IS_ERR_OR_NULL(skb))
                err = udp_send_skb(skb, fl4);
        goto out;
}
```



`ip_make_skb` which builds skb in `./net/ipv4/ip_output.c`

- This considers MTU size and UDP framgmentation offloading capability of NIC while building an skb with specific size.

- `ip_make_skb` calls `ip_setup_cork` which calls `__ip_append_data` 

  ```C
  err = __ip_append_data(sk, fl4, &queue, &cork,
                         &current->task_frag, getfrag,
                         from, length, transhdrlen, flags);
  ```

  If you notice it requires `queue` which is `sk_buff_head` and `cork` which is `inet_cork` structure. In this non-corked path `ip_make_skb` creates faux `queue` and `cork`for above call.

- errors while above appending will result in data drop with `__ip_flush_pending_frames`

- if no erros occur then `__ip_make_skb` on the return value of `ip_make_skb` 

  ```C
  return __ip_make_skb(sk, fl4, &queue, &cork);
  ```

If no error occurs then `udp_send_skb` get called from `udp_sendmsg` as final step of transmission. In case of any error, UDP error account is updated.



###### Slow path for corked UDP sockets with no preexisting corked data (udp_sendmsg)

`ip_append_data`->`__ip_append_data` -> `__ip_make_skb`  gets an skb

- lock the socket
- create a flow structure for this UDP flow is prepared for corking.
- append to existing data.

```C
  lock_sock(sk);
  if (unlikely(up->pending)) {
          /* The socket is already corked while preparing it. */
          /* ... which is an evident application bug. --ANK */
          release_sock(sk);

          LIMIT_NETDEBUG(KERN_DEBUG pr_fmt("cork app bug 2\n"));
          err = -EINVAL;
          goto out;
  }
  /*
   *      Now cork the socket to pend data.
   */
  fl4 = &inet->cork.fl.u.ip4;
  fl4->daddr = daddr;
  fl4->saddr = saddr;
  fl4->fl4_dport = dport;
  fl4->fl4_sport = inet->inet_sport;
  up->pending = AF_INET;

do_append_data:
  up->len += ulen;
  err = ip_append_data(sk, fl4, getfrag, msg->msg_iov, ulen,
                       sizeof(struct udphdr), &ipc, &rt,
                       corkreq ? msg->msg_flags|MSG_MORE : msg->msg_flags);
```



`ip_append_data` above will check

- if `MSG_PROBE` flag is sent? if yes, then no data is going to be sent and this packet is for path probing (PMTU).
- if socket send queue is empty then `ip_setup_cork` called to initiate corking.





**__ip_append_data**

Then it calls `__ip_append_data` which is a complex function and reached in both corked and uncorked state of the socket.

- It sets up NIC hardare related feature flags like

  -  `NETIF_F_UFO` - UDP fragmentation offloading    
  - `NETIF_F_SG`  - Scatter/Gather IO - this is support for ring buffers

- It also tracks socket send queue usage (represented by `sk->sk_wmem_alloc` ) through `sock_wmalloc`  which is tuned by `net.core.wmem_max` and `net.core.wmem_default`.If queuing fails due to lack of memory then `SOCK_NOSPACE` error flag get set up which reflects in `SNDBUFERRORS` error stats in ``/proc/net/snmp` .

   

- Increments in error stats if needed



Upon successful completion of `__ip_append_data`  if socket is corked then control is passed upward in `udp_sendmsg` for further queuing or call to `udp_push_pending_frames` to flush the queue.

in udp_sendmsg()

```C
if (err)
        udp_flush_pending_frames(sk);
else if (!corkreq)
        err = udp_push_pending_frames(sk);
else if (unlikely(skb_queue_empty(&sk->sk_write_queue)))
        up->pending = 0;
release_sock(sk);
```

if socket is NOT corked then `__ip_make_skb` gets called to dequeue the packet as a return value of the `__ip_append_data` call.



**Errors**

if

- The non-corking fast path failed to make an skb or `udp_send_skb` reports an error, or
- `ip_append_data` fails to append data to a corked UDP socket, or
- `udp_push_pending_frames` returns an error received from `udp_send_skb` when trying to transmit a corked skb

the `SNDBUFERRORS` statistic will be incremented only if the error received was `ENOBUFS` (no kernel memory available) or the socket has `SOCK_NOSPACE` set (the send queue is full):

```C
/*
 * ENOBUFS = no kernel mem, SOCK_NOSPACE = no sndbuf space.  Reporting
 * ENOBUFS might not be good (it's not tunable per se), but otherwise
 * we don't have a good statistic (IpOutDiscards but it can be too many
 * things).  We could add another new stat but at least for now that
 * seems like overkill.
 */
if (err == -ENOBUFS || test_bit(SOCK_NOSPACE, &sk->sk_socket->flags)) {
        UDP_INC_STATS_USER(sock_net(sk),
                        UDP_MIB_SNDBUFERRORS, is_udplite);
}
return err;
```



###### udp_send_skb

Final step of `udp_sendmsg`  which will hand off packet from UDP layer to IP layer after adding UDP headers and various checksum related work 

Headers

```C
static int udp_send_skb(struct sk_buff *skb, struct flowi4 *fl4)
{
				/* useful variables ... */

        /*
         * Create a UDP header
         */
        uh = udp_hdr(skb);
        uh->source = inet->inet_sport;
        uh->dest = fl4->fl4_dport;
        uh->len = htons(len);
        uh->check = 0;
```



Checksums

```C
if (is_udplite)                                  /*     UDP-Lite      */
        csum = udplite_csum(skb);

else if (sk->sk_no_check == UDP_CSUM_NOXMIT) {   /* UDP csum disabled */

        skb->ip_summed = CHECKSUM_NONE;
        goto send;

} else if (skb->ip_summed == CHECKSUM_PARTIAL) { /* UDP hardware csum */

        udp4_hwcsum(skb, fl4->saddr, fl4->daddr);
        goto send;

} else
        csum = udp_csum(skb);

/** code **/

uh->check = csum_tcpudp_magic(fl4->saddr, fl4->daddr, len,
                              sk->sk_protocol, csum);
if (uh->check == 0)
        uh->check = CSUM_MANGLED_0;
```



 And send after updating UDP stats in `/proc/net/snmp`

```C
send:
  err = ip_send_skb(sock_net(sk), skb);
  if (err) {
          if (err == -ENOBUFS && !inet->recverr) {
                  UDP_INC_STATS_USER(sock_net(sk),
                                     UDP_MIB_SNDBUFERRORS, is_udplite);
                  err = 0;
          }
  } else
          UDP_INC_STATS_USER(sock_net(sk),
                             UDP_MIB_OUTDATAGRAMS, is_udplite);
  return err;
```



`udp_send_skb` will call `ip_send_skb` to pass on skb to IP layer.



### IP protocol layer

`ip_send_skb` is defined in `./net/ipv4/ip_output.c ` which calls `ip_local_out` and bumps IP error statistics OutDiscards in `/proc/net/snmp` with the help of `net_xmit_errno`. 

```C
int ip_send_skb(struct net *net, struct sk_buff *skb)
{
        int err;

        err = ip_local_out(skb);
        if (err) {
                if (err > 0)
                        err = net_xmit_errno(err);
                if (err)
                        IP_INC_STATS(net, IPSTATS_MIB_OUTDISCARDS);
        }

        return err;
}
```

##### `ip_local_out` and `__ip_local_out`

- determines the length of the IP packet

- computes the IP checksum

- pass on packet to the netfilter 

- if netfilter check comes positive then it calls next step through  `dst_output`

  ```C
  int ip_local_out(struct sk_buff *skb)
  {
          int err;
  
          err = __ip_local_out(skb);
          if (likely(err == 1))
                  err = dst_output(skb);      /* Next step */
  
          return err;
  }
  
  int __ip_local_out(struct sk_buff *skb)
  {
          struct iphdr *iph = ip_hdr(skb);
  
          iph->tot_len = htons(skb->len);    /* length */
          ip_send_check(iph);                /* checksum */ 
          return nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT, skb, NULL,
                         skb_dst(skb)->dev, dst_output);   /* netfilter */
  }
  ```

NOTE:  all those IPTABLE rules will get executed as a part of  `nf_hook` which will be in the context of the systemcall. Imagine that every write call on the socket will hit `nf_hook` and if those rules are expensive then it will take more CPU time in the system space.



##### `dst_output` destination cache and `ip_output`

`dst_output` calls the IP output function attached to the socket. This function was attached in `udp_sendmsg` call with the help of `ip_route_output_flow` and `sk_dst_set`. In this case the output function is `ip_output`

```C
/* Output packet to network from transport.  */
static inline int dst_output(struct sk_buff *skb)
{
        return skb_dst(skb)->output(skb);  /* call to ip_output */
}
```

`ip_output` 

- will increment number of bytes and packets count in `/proc/net/snmp`. 
- It passes the skb to `ip_finish_output` through netfilter hook `NF_HOOK_COND`

```C
int ip_output(struct sk_buff *skb)
{
        struct net_device *dev = skb_dst(skb)->dev;

        IP_UPD_PO_STATS(dev_net(dev), IPSTATS_MIB_OUT, skb->len);

        skb->dev = dev;
        skb->protocol = htons(ETH_P_IP);

        return NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, skb, NULL, dev,
                            ip_finish_output,
                            !(IPCB(skb)->flags & IPSKB_REROUTED));
}
```

Above netfilter hook might modify IP packet to handle SNAT or similar kind of functionality where destination IP might have changed completely. To handle that delivery 

`ip_finish_output`

- passes packet again through `dst_output` based on its flags
- fragment the packet based on path MTU discovery to the smallest possible size to avoid any fragmentation.
- and calls `ip_finish_output2`

```C
static int ip_finish_output(struct sk_buff *skb)
{
#if defined(CONFIG_NETFILTER) && defined(CONFIG_XFRM)
        /* Policy lookup after SNAT yielded a new policy */
        if (skb_dst(skb)->xfrm != NULL) {
                IPCB(skb)->flags |= IPSKB_REROUTED;
                return dst_output(skb);  /* re-entry to dst_output with new IP dst */
        }
#endif
        if (skb->len > ip_skb_dst_mtu(skb) && !skb_is_gso(skb))
                return ip_fragment(skb, ip_finish_output2);  /* fragmentation based on PMTU */
        else
                return ip_finish_output2(skb); /* call to next level */
}
```



`ip_finish_output2`

- updates stats for multicast or broadcast packets
- updates skb headroom with `skb_realloc_headroom`
- pass this skb for neighbour lookup for a next hop by using  `rt_nexthop` and `__ipv4_neigh_lookup_noref`
- if next hop not found then create one by looking at routing and ARP tables `__neigh_create`
- then send packet to the next level `dst_neigh_output`
- or update error stats for the failure 



```C
static inline int ip_finish_output2(struct sk_buff *skb)
{

				/* variable declarations */

        if (rt->rt_type == RTN_MULTICAST) {
                IP_UPD_PO_STATS(dev_net(dev), IPSTATS_MIB_OUTMCAST, skb->len);
        } else if (rt->rt_type == RTN_BROADCAST)
                IP_UPD_PO_STATS(dev_net(dev), IPSTATS_MIB_OUTBCAST, skb->len);

        /* Be paranoid, rather than too clever. */
        if (unlikely(skb_headroom(skb) < hh_len && dev->header_ops)) {
                struct sk_buff *skb2;

                skb2 = skb_realloc_headroom(skb, LL_RESERVED_SPACE(dev));
                if (skb2 == NULL) {
                        kfree_skb(skb);
                        return -ENOMEM;
                }
                if (skb->sk)
                        skb_set_owner_w(skb2, skb->sk);
                consume_skb(skb);
                skb = skb2;
        }
  
  
  /** code **/
  
        rcu_read_lock_bh();
        nexthop = (__force u32) rt_nexthop(rt, ip_hdr(skb)->daddr);
        neigh = __ipv4_neigh_lookup_noref(dev, nexthop);
        if (unlikely(!neigh))
                neigh = __neigh_create(&arp_tbl, &nexthop, dev, false);
  
    /** code **/

        if (!IS_ERR(neigh)) {
                int res = dst_neigh_output(dst, neigh, skb);  /* sending skb to the next level */

                rcu_read_unlock_bh();
                return res;
        }
        rcu_read_unlock_bh();

        net_dbg_ratelimited("%s: No header cache and no neighbour!\n",
                            __func__);
        kfree_skb(skb);
        return -EINVAL;
}
```



`dst_neigh_output`

- it will warm up the destination cache 

  ```C
  static inline int dst_neigh_output(struct dst_entry *dst, struct neighbour *n,
                                     struct sk_buff *skb)
  {
          const struct hh_cache *hh;
  
          if (dst->pending_confirm) {
                  unsigned long now = jiffies;
  
                  dst->pending_confirm = 0;
                  /* avoid dirtying neighbour */
                  if (n->confirmed != now)
                          n->confirmed = now; /* updates with current jiffie */ 
          }
  ```

- execute Path towards `dev_queue_xmit` either through `neigh_hh_output` or through `n->output` where `n` is setup in the `ip_finish_output2` while determining next hop.

  ```C
          rcu_read_lock_bh();
          nexthop = (__force u32) rt_nexthop(rt, ip_hdr(skb)->daddr);
          neigh = __ipv4_neigh_lookup_noref(dev, nexthop);
          if (unlikely(!neigh))
                  neigh = __neigh_create(&arp_tbl, &nexthop, dev, false);
  
  ```

  `neigh_hh_output` is called when hardware header is already setup from previous packet delvery to same desination.
  `n->output` does some neighbor lookup and calls `neigh_resolve_output`  to do first time hop related calcuations.

  

`n->output` and `neigh_resolve_output`

following structure gets populated by the call `__neigh_create` in `ip_finish_output2` based on state of neighbour

```C
static const struct neigh_ops arp_hh_ops = {
        .family =               AF_INET,
        .solicit =              arp_solicit,
        .error_report =         arp_error_report,
        .output =               neigh_resolve_output,
        .connected_output =     neigh_resolve_output,
};
```

This `neigh_resolve_output` get executed to check status of neighbour ad then call `dev_queue_xmit`.

```C
/* Slow and careful. */

int neigh_resolve_output(struct neighbour *neigh, struct sk_buff *skb)
{
        struct dst_entry *dst = skb_dst(skb);
        int rc = 0;

        if (!dst)
                goto discard;

        if (!neigh_event_send(neigh, skb)) {
                int err;
                struct net_device *dev = neigh->dev;
                unsigned int seq;
                
          
          /**    code    **/
          
        if (dev->header_ops->cache && !neigh->hh.hh_len)
                 neigh_hh_init(neigh, dst);
          
          /**    code    **/

        do {
                  __skb_pull(skb, skb_network_offset(skb));
                  seq = read_seqbegin(&neigh->ha_lock);
                  err = dev_hard_header(skb, dev, ntohs(skb->protocol),
                                              neigh->ha, NULL, skb->len);
         } while (read_seqretry(&neigh->ha_lock, seq));
          
          
          /**    code    **/

         if (err >= 0)
                        rc = dev_queue_xmit(skb); /* passing on to next layer */
                else
                        goto out_kfree_skb;
                 
          /**    code    **/

out:
        return rc;
discard:
        neigh_dbg(1, "%s: dst=%p neigh=%p\n", __func__, dst, neigh);
out_kfree_skb:
        rc = -EINVAL;
        kfree_skb(skb);
        goto out;
}
EXPORT_SYMBOL(neigh_resolve_output);
```

- it attempts to resolve a neighbour which is not connected or has no cached hardware header.
- `neigh_event_send` and` __neigh_event_send` (from `./net/core/neighbour.c` ) will help to send a probe to decide on the neighbour's state 
- Depending on the state update the hardware header through seqlock and `dev_hard_header()` 
- Upon successful update, call `dev_queue_xmit(skb)` 
- If there is any failure before that then it logs an error at the `discard` label.

  

Few procfs settings

- /proc/sys/net/ipv4/neigh/default/delay_first_probe_time
- /proc/sys/net/ipv4/neigh/default/app_solicit
- /proc/sys/net/ipv4/neigh/default/mcast_solicit
- /proc/sys/net/ipv4/neigh/default/unres_qlen



### Qdiscs layer 

`dev_queue_xmit` from`./net/core/dev.c.` manages the qdisc layer where qdisc is mapped with each TX ring buffer of each NIC device.

```C
int dev_queue_xmit(struct sk_buff *skb)
{
        return __dev_queue_xmit(skb, NULL);
}
EXPORT_SYMBOL(dev_queue_xmit);


static int __dev_queue_xmit(struct sk_buff *skb, void *accel_priv)
{
        struct net_device *dev = skb->dev;
        struct netdev_queue *txq;
        struct Qdisc *q;
        int rc = -ENOMEM;

        skb_reset_mac_header(skb);

        /* Disable soft irqs for various locks below. Also
         * stops preemption for RCU.
         */
        rcu_read_lock_bh();

        skb_update_prio(skb);
  
  /* code continues */
  
        txq = netdev_pick_tx(dev, skb, accel_priv);

  /* code continues */
```



- `skb_reset_mac_header` reset the skb header pointers to read hardware frame headers
- `rcu_read_lock_bh` bottom half RCU (Read Copy Update) lock. 
- `skb_update_prio` network cgroup priority
- Move on to TX queue selection with `netdev_pick_tx`





`netdev_pick_tx`  from `./net/core/flow_dissector.c`does a queue selection which is attached to NIC ring buffer. 

- skips all selection if there is only one queue 
- check for device driver defined hardware specific queue selection function `ndo_select_queue`
- if above functionality is not present then pick using `__netdev_pick_tx`

```C
struct netdev_queue *netdev_pick_tx(struct net_device *dev,
                                    struct sk_buff *skb,
                                    void *accel_priv)
{
        int queue_index = 0;

        if (dev->real_num_tx_queues != 1) { /* selection skip if single queue */
                const struct net_device_ops *ops = dev->netdev_ops;
                if (ops->ndo_select_queue)
                  			/* NIC HW based selection through device driver */
                        queue_index = ops->ndo_select_queue(dev, skb,
                                                            accel_priv);
                else
                  			/* kernel based queue selection with XPS hook */
                        queue_index = __netdev_pick_tx(dev, skb);

                if (!accel_priv)
                        queue_index = dev_cap_txqueue(dev, queue_index);
        }

        skb_set_queue_mapping(skb, queue_index);
        return netdev_get_tx_queue(dev, queue_index);
}
```

`__netdev_pick_tx` will select queue 

- by looking at XPS configuration using `get_xps_queue`
- or by using `skb_tx_hash`  which calls `__skb_tx_hash` defined in ` ./net/core/flow_dissector.c`  which is an expored symbol. 



Continuing with `__dev_queue_xmit` which eventually calls `__dev_xmit_skb` which is armed with qdiscs and defined in `./net/core/dev.c`.

NOTE: there is another path which goes directly to `dev_hard_start_xmit` skipping qdiscs if device is a loopback. I have skipped that part of code here.















 





















