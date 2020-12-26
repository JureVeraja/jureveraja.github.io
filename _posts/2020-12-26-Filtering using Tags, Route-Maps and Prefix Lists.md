---
title: Filtering using Tags, Route-Maps and Prefix Lists
author: Jure Veraja
date: 2020-12-26 13:33:00 +0100
categories: [ccnp]
tags: [tags, route-maps]
math: true
---

To continue based on the previous post about Redistribution between Routing Protocols and Suboptimal Routing.
Here we will try to resolve the issue we encountered while redistributing from EIGRP into OSPF and we will also
introduce new issue which happens if we redistribute back from OSPF into the EIGRP domain.
We will use the same topology as last time except the addition of R3 which will do redistribution from RIP into EIGRP.

![redistribution2](/assets/img/sample/redistribution2.png)

After we redistributed from EIGRP into OSPF, and introduced the problem which can happen in the previous post. 
Now imagine what can happen if you redistribute those routes from OSPF into EIGRP?

A routing loop will occur, lets check the configuration and show commands.

### Router 4 and Router 5

```
R4# show run | s router eigrp
router eigrp 1
 network 10.10.2.0 0.0.0.255
 redistribute ospf 1 metric 100000000 1 255 1 1500
 
 R4# show run | s router eigrp
router eigrp 1
 network 10.10.2.0 0.0.0.255
 redistribute ospf 1 metric 100000000 1 255 1 1500
 ```
 
Now route 150.1.1.0/24 would be redistributed from OSPF to EIGRP and R3 would have two (or three ways, remember the previous post) to reach
the destination. He would use either the route through R2 or R4/R5, depending on the metric that was calculated during the redistribution.
 
### R3 Routing table and Topology table
 
```
D EX     150.1.1.0 [170/3072] via 10.10.3.5, 00:22:36, GigabitEthernet1/0
```
```
R3#show ip eigrp topology 150.1.1.0/24
EIGRP-IPv4 Topology Entry for AS(1)/ID(10.10.3.3) for 150.1.1.0/24
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 3072
  Descriptor Blocks:
  10.10.3.5 (GigabitEthernet1/0), from 10.10.3.5, Send flag is 0x0
      Composite metric is (3072/512), route is External
  10.10.1.2 (GigabitEthernet2/0), from 10.10.1.2, Send flag is 0x0
      Composite metric is (26112/25856), route is External   
```

As we can see better metric was calculated for the route which was redistributed from OSPF into EIGRP. And now 
R3 prefers the path to R5, which would eventually create a loop, because R5 points to R4 for the 150.1.1.0/24 network
which was redistributed from R4 EIGRP to OSPF as we demonstrated in previous post.

Lets check once again the route on R5 and then do the traceroute from R3 to prove that we have routing loop.

### R5 Route for 150.1.1.0/24

```
O E2     150.1.1.0 [110/20] via 10.10.4.4, 00:27:09, GigabitEthernet2/0
```

### R3 Traceroute

```
R3#traceroute 150.1.1.1
Type escape sequence to abort.
Tracing the route to 150.1.1.1
  1 10.10.3.5 12 msec 12 msec 12 msec
  2 10.10.4.4 28 msec 20 msec 24 msec
  3 10.10.2.3 20 msec 24 msec 32 msec
  4 10.10.3.5 36 msec 48 msec 60 msec
  5 10.10.4.4 76 msec 44 msec 56 msec
  6 10.10.2.3 60 msec 52 msec 52 msec
```

We can resolve those issues by using three examples: Lowering the AD, Filtering using prefix-lists, and tagging the routes then blocking them by referencing the route-maps.

# LOWERING THE ADMINISTRATIVE DISTANCE

One way to deal with this issue is by lowering the `AD` of the EIGRP external routes on R4 and R5, that way they will always be more prefered than the OSPF routes.
Lets test it out by changing AD on R4 and R5: the command is "distance eigrp "internal" "external""

```
R4(config)#router eigrp 1
R4(config-router)#distance eigrp 90 109

R5(config)#router eigrp 1
R5(config-router)#distance eigrp 90 109
```

Now we can see the routing table immediately changed and they prefer EIGRP routes:

```
R4(config-router)#do sh ip route
  D EX     150.1.1.0 [109/26368] via 10.10.2.3, 00:00:10, GigabitEthernet0/0
R5(config-router)#do sh ip route
  D EX     150.1.1.0 [109/26368] via 10.10.3.3, 00:00:05, GigabitEthernet1/0
```

Now you can check the R3 and see that it doesn't have the route to R5 anymore because that route is gone from the routing table on R5 and therefore can't be redistributed.
```
R3#show ip route
D EX     150.1.1.0 [170/26112] via 10.10.1.2, 00:09:10, GigabitEthernet2/0
```
# FILTERING USING PREFIX-LIST

Another way we can deal with the issue of routing loop is that we can control which routes get installed in the routing table from 
OSPF database by using Distribute Lists. So if we filter the route from being installed in the routing table, it can't be redistributed.

First we will create prefix-lists then we will apply them to the OSPF routing process:

```
R4(config)#ip prefix-list DENY_150.1.1.0 seq 10 deny 150.1.1.0/24
R4(config)#ip prefix-list DENY_150.1.1.0 seq 20 permit 0.0.0.0/0 le 32
R4(config)#router ospf 1
R4(config-router)#distribute-list prefix DENY_150.1.1.0 in

R5(config)#ip prefix-list DENY_150.1.1.0 seq 10 deny 150.1.1.0/24
R5(config)#ip prefix-list DENY_150.1.1.0 seq 20 permit 0.0.0.0/0 le 32
R5(config)#router ospf 1
R5(config-router)#distribute-list prefix DENY_150.1.1.0 in
```

With the command `distribute-list prefix DENY_150.1.1.0 in` we filterd the 150.1.1.0/24 network from being installed from the OSPF database into the routing table by using the keyword `IN` at the end of the command. Also remember to permit all other routes, which is what seq 20 is doing.

Prove it by checking the routing table:

```
R4#show ip route
D EX     150.1.1.0 [170/26368] via 10.10.2.3, 00:08:52, GigabitEthernet0/0
R5#show ip route
D EX     150.1.1.0 [170/26368] via 10.10.3.3, 00:09:34, GigabitEthernet1/0
```
# FILTERING ROUTES BY USING TAG AND ROUTE-MAPs





  
