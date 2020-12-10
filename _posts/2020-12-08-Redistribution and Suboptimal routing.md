---
title: Redistribution between Routing Protocols and Suboptimal Routing
author: Jure Veraja
date: 2020-12-09 18:33:00 +0100
categories: [CCNP]
tags: [redistribution,suboptimal-routing]
math: true
---

Here we will see an example of redistribution between RIP - EIGRP - OSPF.

We will use this topology example, all ip addresses have been marked:

![redistribution](/assets/img/sample/redistribution.png)

We will first check the configuration on R1 and R2 and then redistribute the Loopback network from RIP into EIGRP and then into OSPF. 

```
hostname R1               
!
interface Loopback0
 ip address 150.1.1.1 255.255.255.0
!
interface GigabitEthernet2/0
 ip address 10.10.1.1 255.255.255.0
!
router rip
 version 2
 network 10.0.0.0
 network 150.1.0.0
 no auto-summary
```

```
hostname R2
!
interface GigabitEthernet0/0
 ip address 10.10.2.2 255.255.255.0
!
interface GigabitEthernet1/0
 ip address 10.10.3.2 255.255.255.0
!
 interface GigabitEthernet2/0
 ip address 10.10.1.2 255.255.255.0
!         
router rip
 version 2
 network 10.0.0.0
```

We can see the routing table on R2 and verify that R1-R2 are doing RIP. We can see the route learned from R1.

``` 
  150.1.0.0/24 is subnetted, 1 subnets
R   150.1.1.0 [120/1] via 10.10.1.1, 00:00:09, GigabitEthernet2/0
```

# Redistribute RIP into EIGRP

### Router 2 EIGRP config: 

```
!
router eigrp 1
 network 10.0.0.0
 redistribute rip metric 1 1 255 1 1500
```

To redistribute RIP into EIGRP, EIGRP requires that you set the SEED metric, othervise the EIGRP would use the inifity `SEED` metric which is the default when you redistribute into EIGRP. 

EIGRP uses a metric that is based on bandwidth, delay, reliability, load, and MTU.

So in our example we used 1 for bandwith, 1 for delay, 255 for reliability, 1 for load, and 1500 our usual MTU. 

```
R2(config-router)#redistribute rip metric ?
  <1-4294967295>  Bandwidth metric in Kbits per second
```

```
R2(config-router)#redistribute rip metric 1 ?
    <0-4294967295>  EIGRP delay metric, in 10 microsecond units
```

```
R2(config-router)#redistribute rip metric 1 1 ?
    <0-255>  EIGRP reliability metric where 255 is 100% reliable
```

```
R2(config-router)#redistribute rip metric 1 1 255 ?
    <1-255>  EIGRP Effective bandwidth metric (Loading) where 255 is 100% loaded
```

```
R2(config-router)#redistribute rip metric 1 1 255 1 ?
    <1-65535>  EIGRP MTU of the path
```

In our topology example it doesn't matter the actual metric the EIGRP calculates from the given values, but in your example it might be different so you can tweak it alittle bit.

After redistribution we can head over to R4 and R5 to check the routes they received:
  
### Router 4 EIGRP route and configuration

```
router eigrp 1
 network 10.0.0.0
 ```

```
D EX     150.1.1.0 
           [170/2560000512] via 10.10.2.2, GigabitEthernet0/0
```

### Router 5 EIGRP route configuration

```
router eigrp 1
 network 10.0.0.0
```

```
D EX     150.1.1.0 
           [170/2560000512] via 10.10.3.2, GigabitEthernet1/0
```

And we succesfully redistributed from RIP to EIGRP. Notice the EIGRP external route code "D EX" and AD is 170 which is default AD for EIGRP external routes.

# Redistribute EIGRP into OSPF

Lets go to ospf process on both router 4 and 5 and configure the OSPF process and redistribute EIGPR into OSPF.

```
router ospf 1
 redistribute eigrp 1 subnets
```

Remember you have to insert subnets keyword in the end othervise only classful network would be advertised.
 
After you've redistributed on both R4 and R5 you can check the routing table:

### Router 4

```
O E2  150.1.1.0 [110/20] via 10.10.4.5, GigabitEthernet2/0
```

### Router 5

```
D EX   150.1.1.0 
           [170/2560000512] via 10.10.3.2, GigabitEthernet1/0
```

Now a couple of interesting things happened here:

Notice how Router 4 no longer uses EIGRP external route to reach 150.1.1.0/24 network, it now goes through g2/0 which is connected to R5. It is happening because that OSPF external route which is injected when we redistributed from EIGRP has lower Administrative Distance (AD) which is 20 by default for type-2 OSPF metric which has code O E2. 

We now have an example of `suboptimal routing` at R4.

Check out Router 5. To reach 150.1.1.0/24 he still has the same EIGRP route he had before. Why is that happening? 

Well, it's actually quite intriguing. To redistribute some route into a routing process you first have to have that route in the routing table. So in this case Router 5 has D EX route for 150.1.1.0/24 network in the routing table, and he redistributes it into OSPF domain. R4 receives his `LSA type 5` advertisement for the network 150.1.10/24. And he then removes the EIGRP route from routing table (altough it still stays in EIGRP topology), R4 then installs that LSA type 5 route to the routing table because it has better metric than EIGRP route.

As we've seen R4 removed the EIGRP route from routing table, and now he couldnt redistribute it via OSPF. Remember you need to have it installed in the routing table!

Think about what would happen if you had perfect synchronization between OSPF processes? What if both R4 and R5 had perfect synchronization and they both at the same time advertise to themselves the redistributed route from EIGRP to reach the 150.1.1.0/24 network? 

What would happen is that they would both install the OSPF external routes in the routing table and remove the EIGRP route, then they would point to each other as next best hop. But before they could even process the traffic and send it out, remember that to redistribute routes into routing process you need to have the route in the routing table! Well, as they both removed the EIGRP route from routing table and installed the OSPF one, now they dont have nothing to redistribute. And when you have nothing to redistribute means that the OSPF route is also gone.. and since the EIGRP route was sitting in the EIGRP topology it gets installed back in the routing table and process repeats all over again cuz you still have the command `redistribute eigrp subnets` under the router ospf process.

You should be able to use `debug ip routing`, to see the route changes.

Good thing is that you won't receive perfect synchronization in ospf so those situations have little chance to happen. What will happen is, whichever router firsts sends LSA type 5, he will keep having the EIGRP route in routing table and will use it. The other router would use that OSPF route that the first router advertised.

Quite interesting situation, It's quite hard to accomplish and almost impossible in a lab scenario but I encourage you nevertheless to try it out. 

Thanks for the read and i hope you found it interesting. All the best to you!













