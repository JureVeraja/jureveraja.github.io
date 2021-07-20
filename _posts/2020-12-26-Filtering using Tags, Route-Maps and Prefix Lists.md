---
title: Filtering using Tags, Route-Maps and Prefix Lists
author: Jure Veraja
date: 2020-12-26 13:33:00 +0100
categories: [networking]
tags: [tagging, route-maps]
math: true
---

To continue based on the previous post about Redistribution between Routing Protocols and Suboptimal Routing.
Here we will try to resolve the issue we encountered while redistributing from EIGRP into OSPF and we will also
introduce new issue which happens if we redistribute back from OSPF into the EIGRP domain.
We will use the same topology as last time except the addition of R3,just for the sake of RIP redistribution.

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

So we must tag the routes coming from the EIGRP into the OSPF. First we will create ACL to match the route we want to tag (we can also do it with ip prefix-list command). After that we will match that ACL in the route-map, and then we will insert the route-map into the OSPF routing process.

```
R4(config)#access-list 1 permit 150.1.1.0 0.0.0.255
R4(config)#route-map TAG_EIGRP_TO_OSPF permit 10
R4(config-route-map)#match ip address 1
R4(config-route-map)#set tag 100
R4(config)#route-map TAG_EIGRP_TO_OSPF permit 20
R5(config)#router ospf 1
R4(config)#redistribute eigrp 1 subnets route-map TAG_EIGRP_TO_OSPF

R5(config)#access-list 1 permit 150.1.1.0 0.0.0.255
R5(config)#route-map TAG_EIGRP_TO_OSPF permit 10
R5(config-route-map)#match ip address 1
R5(config-route-map)#set tag 200
R5(config)#route-map TAG_EIGRP_TO_OSPF permit 20
R5(config)#router-ospf 1
R5(config)#redistribute eigrp 1 subnets route-map TAG_EIGRP_TO_OSPF
```

You can confirm that the route is being tagged by issuing `show ip ospf database`
```
R4#show ip ospf database
Type-5 AS External Link States
150.1.1.0       10.10.4.4       67          0x80000003 0x008E0B `100`

R5#show ip ospf database
Type-5 AS External Link States
150.1.1.0       10.10.3.5       6           0x80000001 0x006BB7 `200`
150.1.1.0       10.10.4.4       67          0x80000003 0x008E0B `100`
```

R5 has both routes and route tags, one from himself, and one from R4 because he got advertisement from OSPF for OE2 routes which were redistributed from EIGRP into OSPF. Once he got the advertisement first, he removes the EIGRP external route for 150.1.1.0/24 network from routing table and installs the OE2 route. Because of this he cannot advertise the OE2 route to R4. We explained this previously in the last post.

After we implemented tagging for routes coming into the OSPF, now we must prevent those routes from being redistributed back into the EIGRP based on their tags. On R4 we will prevent routes with tag 200, and on R5 we will prevent with tag 100. We will do this using route-maps again, and then inserting it into redistribution process.

First we must match the tag with deny statement, and in seq 20 permit all other routes. After that we insert the route-map in the eigrp process, saying to redistribute ospf routes with the route-map we just created which denies the routes with specified tag.

```
R4(config)#route-map OSPF_INTO_EIGRP deny 10
R4(config-route-map)#match tag 200
R4(config)#route-map OSPF_INTO_EIGRP permit 20
R4(config)#router eigrp 1
R4(config)#redistribute ospf 1 metric 100000 1 255 1 1500 route-map OSPF_INTO_EIGRP

R5(config)#route-map OSPF_INTO_EIGRP deny 10
R5(config-route-map)#match tag 100
R5(config)#route-map OSPF_INTO_EIGRP permit 20
R5(config)#router eigrp 1
R5(config)#redistribute ospf 1 metric 100000 1 255 1 1500 route-map OSPF_INTO_EIGRP
```

We used deny statement for the routes that match the tags, and in seq 20 we permited all other routes.

Now back on R3 you can confirm that the matched route for 150.1.1.0/24 was not redistributed from OSPF into the EIGRP, based on what we've just accomplished. We told R4 to not redistribute route 150.1.1.0/24 coming from R5 (tag 200), and to R5 we told not to redistribute route 150.1.1.0/24 coming from R4 (tag 100).

```
R3#show ip route
D EX     150.1.1.0 [170/26112] via 10.10.1.2, 00:09:22, GigabitEthernet2/0
```

I hope you found this post helpfull, thank you for the reading and see you on the next one! All the best!


  
