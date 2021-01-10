---
title: EIGRP Review - Authentication, Stub, Off-Set Lists, Route-Filtering, Variance
author: Jure Veraja
date: 2021-01-10 18:33:00 +0100
categories: [ccnp]
tags: [eigrp]
math: true
---

In this example we will use the simple topology below to demonstrate some EIGRP stuff like creating STUB site, using offset-list to change the metric of the routes, route filtering, and so on. 
So lets get going.
![eigrp_topo](/assets/img/sample/eigrp_topo.png)

We will be advertising networks 150.1.1.0/24 and 160.1.1.0/24 from imaginary internet into our eigrp site. We will do normal eigrp peering without redistribution, as we covered that already in the previous post.

On our NET router we just advertise all network's into EIGRP for the sake of simplicity and we will also include authentication for security, that way we protect our network from intruders who might plugin their router and try to peer with us.

First we create key chain in global configuration and specify the key-string which will be used to authenticate the routers. After that when using normal legacy configuration mode, you must specify authentication under the interface configuration.

### NET ROUTER PEERING CONFIGURATION

```
NET(config)#key chain eigrp_key
NET(config-keychain)#key 1
NET(config-keychain-key)#key-string LAB
---------------------------------------
NET(config)#int g1/0
NET(config-if)#ip authentication mode eigrp 1 md5
NET(config-if)#ip authentication key-chain eigrp 1 eigrp_key
------------------------------------------------------------
NET(config)#router eigrp 1
NET(config-router)#network 0.0.0.0
```

### R1,R2,and R3 PEERING CONFIGURATION

For the practice we will be doing NAMED convention for configuring EIGRP on R1 and R2.

You create a key same way as in legacy configuration mode, only difference here is that we apply the key configuration under the EIGRP named process as you see in the example.
Basically in NAMED configuration mode you have hierarchy while doing configuration as you can see in our example, you enter the router process by giving it a name, then after that you specify which address-family you're gona use ipv4 or ipv6, keep in mind you can use both address family under the same process. 
With address-family you also specify the ASN number you're gona use for your EIGRP process. After that you enter config mode called "af" as in address-family, in here you enter the interface on which you want to apply specific configuration, as we did for authentication.
After that you exit the af-interface, and specify the networks you want to advertise in the regular address-family mode.

### R1 / R2
```
R1(config)#key chain eigrp_key
R1(config-keychain)#key 1
R1(config-keychain-key)#key-string LAB
--------------------------------------
R1(config)#router eigrp EIGRP
R1(config-router)#address-family ipv4 autonomous-system 1
R1(config-router-af)#af-interface g1/0
R1(config-router-af-interface)#authentication mode md5
R1(config-router-af-interface)#authentication key-chain eigrp_key
R1(config-router-af)#af-interface g2/0
R1(config-router-af-interface)#authentication mode md5
R1(config-router-af-interface)#authentication key-chain eigrp_key
--------------------------------------
R1(config)#router eigrp EIGRP
R1(config-router)#address-family ipv4 autonomous-system 1
R1(config-router-af)#network 10.10.10.0 255.255.255.0
R1(config-router-af)#network 30.30.30.0 255.255.255.0
```
For R2 configuration is the same except the networks that are advertised are: 20.20.20.0/24 , 40.40.40.0/24, and 50.50.50.0/24.

### R3

R3(config)#key chain eigrp_key
R3(config-keychain)#key 1
R3(config-keychain-key)#key-string LAB
--------------------------------------
R3(config)#router eigrp EIGRP
R3(config-router)#address-family ipv4 autonomous-system 1
R3(config-router-af)#af-interface g2/0     ------ same on g1/0
R3(config-router-af-interface)#authentication mode md5
R3(config-router-af-interface)#authentication key-chain eigrp_key
R3(config-router-af)#af-interface g2/0      ----- same on g1/0
R3(config-router-af-interface)#authentication mode md5
R3(config-router-af-interface)#authentication key-chain eigrp_key
--------------------------------------
R3(config)#router eigrp EIGRP
R3(config-router)#address-family ipv4 autonomous-system 1
R3(config-router-af)#network 30.30.30.0 255.255.255.0
R3(config-router-af)#network 40.40.40.0 255.255.255.0

# Influencing EIGRP Metric - Choosing the prefered route


After we created our EIGRP network and peered with our neighbors we can see the routing table on the R3:

```
R3#show ip route
      150.1.0.0/24 is subnetted, 1 subnets
D        150.1.1.0 [90/2575360] via 40.40.40.1, 00:03:55, GigabitEthernet1/0
                   [90/2575360] via 30.30.30.1, 00:03:55, GigabitEthernet2/0
      160.1.0.0/24 is subnetted, 1 subnets
D        160.1.1.0 [90/2575360] via 40.40.40.1, 00:03:55, GigabitEthernet1/0
                   [90/2575360] via 30.30.30.1, 00:03:55, GigabitEthernet2/0
```                  

We have two ways to reach the 150.1.1.0/24 and 160.1.1.0/24 networks. If we want to prefer one way or the other, or if we don't want both routes being installed in the routing table,
we can do it in multiple ways: we can filter the route before it comes into the routing table, we can change the metric of the route that is advertised, we can adjust bandwith or delay on the interface, or we can use the `Offset-list` 
with which we can match the route and change its metric.

### Bandwith and Delay

Lets first consider changing the bandwidth or delay. That creates a problem, because you can only adjust delay and bandwidth on interfaces, you can't choose a specific route.
So lets say on R3 for network 150.1.1.0/24 you only want to use the route via R2, you want other route through R1 gone from routing table, in other words you want to increase its metric so its not being equal-cost anymore.

You go to interface configuration for interface g2/0, you type some bad delay , lets say ```delay 150``` or some bad bandwith lets say ```bandwith 100``` , and know you see the route
towards 150.1.1.0/24 is using only the path through R2.

But inspect the routing table , check what else happened:

```
R3(config-if)#do sh ip route
      150.1.0.0/24 is subnetted, 1 subnets
D        150.1.1.0 [90/2575360] via 30.30.30.1, 00:00:02, GigabitEthernet2/0
      160.1.0.0/24 is subnetted, 1 subnets
D        160.1.1.0 [90/2575360] via 30.30.30.1, 00:00:02, GigabitEthernet2/0
```

You also affected the network 160.1.1.0/24 because you used the bandwith and delay commands on the interface! Thats the example of why its not very good to use that mechanism to change the route metric.

### Offset-List

We will use other mechanism called `Offset-List` to change the metric only for the route we match with the ACL. So we want R3 to choose the path through R2 for the network 150.1.1.0/24. To achieve that we have to go to R1 and create an offset-list influencing the metric for that route.

### R1 Offset-List

```
R1(config)#access-list 1 permit 150.1.1.0 0.0.0.255
R1(config)#router eigrp EIGRP
R1(config-router)#address-family ipv4 autonomous-system 1
R1(config-router-af)#topology base
R1(config-router-af-topology)#offset-list 1 out 1000
```

So we matched the network 150.1.1.0/24 and told the EIGRP process, when sending that route `OUT` increase its metric by 1000. We could also use the `IN` word but that would change the R1's metric also.
Notice how its all done under the `topology base` subconfiguration mode under the eigrp process.

Lets confirm by looking at the routing table and EIGRP topology table on R3:

```
R3(config-if)#do sh ip route
      150.1.0.0/24 is subnetted, 1 subnets
D        150.1.1.0 [90/2575360] via 40.40.40.1, 00:00:36, GigabitEthernet1/0
      160.1.0.0/24 is subnetted, 1 subnets
D        160.1.1.0 [90/2575360] via 40.40.40.1, 00:00:36, GigabitEthernet1/0
                   [90/2575360] via 30.30.30.1, 00:00:36, GigabitEthernet2/0
```

We can see the route is gone from the routing table. Lets check topology table:

```
R3(config-if)#do sh ip eigrp topo
P 160.1.1.0/24, 2 successors, FD is 329646080
        via 30.30.30.1 (329646080/328990720), GigabitEthernet2/0
        via 40.40.40.1 (329646080/328990720), GigabitEthernet1/0
P 150.1.1.0/24, 1 successors, FD is 329646080
        via 40.40.40.1 (329646080/328990720), GigabitEthernet1/0
        via 30.30.30.1 (329647080/328991720), GigabitEthernet2/0
```

If you take a look at the Feasible Distance (we wont go into detail about FD/RD or state of the route (A,P) and other stuff since we should already know that from CCNA.
The metric for the route through 30.30.30.1 is increased by 1000 if you look at last four digits. Meaning it cant be installed in the routing table since and can't be used for load balancing cuz its metric is now higher than the successors FD.

### VARIANCE

For fun lets return our route we just removed from routing table, but not by deleting the offset-list on R1, lets instead use the variance on R3. Variance is used by specific EIGRP feature called `Unequal Path Load Balancing`.
With it you can multiply the SUCCESSORS FD , and whichever feasible successors route metric is lower than the multiplied number of SUCCESSORs route, it means that feasible successor's route can be installed in the routing table.

So lets configure variance on R3 to see our route for 150.1.1.0/24 through R1 returns:

```
R3(config)#router eigrp EIGRP
R3(config-router)#address-family ipv4 autonomous-system 1
R3(config-router-af)#topology base
R3(config-router-af-topology)#variance 2
```
You configure variance also under "topology base" named mode configuration. In this case we multiplied successors FD by x2.
Lets check the routing table on R3:

```
R3#show ip route
      150.1.0.0/24 is subnetted, 1 subnets
D        150.1.1.0 [90/2575360] via 40.40.40.1, 00:00:38, GigabitEthernet1/0
                   [90/2575367] via 30.30.30.1, 00:00:38, GigabitEthernet2/0
      160.1.0.0/24 is subnetted, 1 subnets
D        160.1.1.0 [90/2575360] via 40.40.40.1, 00:00:38, GigabitEthernet1/0
                   [90/2575360] via 30.30.30.1, 00:00:38, GigabitEthernet2/0
```

And the route is back. It is because we multiplied this metric from our successor by x2: P 150.1.1.0/24, 1 successors, FD is 329646080. And comparing that to metric through R1 329647080 we can see that R1 metric would be significantly lower, that way we are satisfying the rule of Unequal cost, and it can be installed in the routing table.












