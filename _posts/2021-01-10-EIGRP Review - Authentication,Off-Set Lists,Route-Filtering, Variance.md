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

```
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
```

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

# ROUTE FILTERING

For route filtering in EIGRP we use `distribute-list` which can be refering to ACL, prefix-list or route-map for matching of the routes we want to filter.

Lets start by creating a distribute list which is referencing an ACL.
We will keep the same idea as we did above, lets filter the 150.1.1.0/24 from coming out of R1 so R3 doesn't know about it and will use the other path through R2.

### Filtering using ACL 
```
R1(config)#ip access-list standard 1
R1(config-std-nacl)#deny 150.1.1.0 0.0.0.255
R1(config-std-nacl)#permit any
--------------------------------------------
R1(config)#router eigrp EIGRP
R1(config-router)#address-family ipv4 autonomous-system 1
R1(config-router-af)#topology base
R1(config-router-af-topology)#distribute-list 1 out
```

When using ACL, you use DENY statement for routes you wan't to deny with distribute-list and permit all other. In distribute-list we used `OUT` statement for the routes that match ACL 1 apply the statements OUTBOUND.

Confirm by checking the routing table on R3:

```
R3#show ip route
      150.1.0.0/24 is subnetted, 1 subnets
D        150.1.1.0 [90/2575360] via 40.40.40.1, 00:00:54, GigabitEthernet1/0
      160.1.0.0/24 is subnetted, 1 subnets
D        160.1.1.0 [90/2575360] via 40.40.40.1, 00:03:29, GigabitEthernet1/0
                   [90/2575367] via 30.30.30.1, 00:03:29, GigabitEthernet2/0
```

### Filtering using Prefix-List

Now we remove the previous distribute-list and we create a new one:

```
R1(config)#ip prefix-list DENY_150.1.1.0/24 deny 150.1.1.0/24
R1(config)#ip prefix-list DENY_150.1.1.0/24 seq 20 permit 0.0.0.0/0 le 32
-------------------------------------------------------------------------
R1(config)#router eigrp EIGRP
R1(config-router)#address-family ipv4 autonomous-system 1
R1(config-router-af)#topology base
R1(config-router-af-topology)#distribute-list prefix DENY_150.1.1.0/24 out
```
Here basically everything is similar as with ACL, except instead of permit any here you must specify 0.0.0.0/0 le 32, meaning all routes that are equal to /32 or less than /32.
The outcome of the routing table is the same, you can test it yourself.

### Filtering using Route-Maps

When using the route-maps logic kind of changes,and it can be confusing sometimes. Lets start by using ACL in combination with route-map:

```
R1(config)#access-list 1 permit 150.1.1.0 0.0.0.255
-------------------------------------------------
R1(config)#route-map DENY_150.1.1.0/24 deny 10
R1(config-route-map)#match ip address 1
R1(config)#route-map DENY_150.1.1.0/24 permit 20
------------------------------------------------
R1(config)#router eigrp EIGRP
R1(config-router)#address-family ipv4 autonomous-system 1
R1(config-router-af)#topology base
R1(config-router-af-topology)#distribute-list route-map DENY_150.1.1.0/24 out
```

In this example, we used a PERMIT statement in the ACL, it basically means ACL is permited to do anything that route-map wanted. And when we look at route-map config, it said DENY all routes that match the ACL 1 as we can see on sequence 10.

You must not forget the sequence 20 route-map part, because otherwise you would deny all routes since there is an invisible deny statement. So creating an empty route-map with the next sequence number , in our case 20, and just using the permit statement will permit all other routes.

Here is where it gets confusing:

If you used deny in ACL and deny in route-map, what would happen is when the route-map checked sequence 10, found a match for ACL 1 and saw a deny statement in ACL which means "move to the next route-map sequence" it would just ignore it, and move to the sequence 20, and sequence 20 would permit all routes, meaning nothing would get filtered.

So to conclude a DENY in ACL means - move to the next route-map sequence. Remember that.

We get same result as before:
```
R3#show ip route
      150.1.0.0/24 is subnetted, 1 subnets
D        150.1.1.0 [90/2575360] via 40.40.40.1, 00:00:54, GigabitEthernet1/0
      160.1.0.0/24 is subnetted, 1 subnets
D        160.1.1.0 [90/2575360] via 40.40.40.1, 00:03:29, GigabitEthernet1/0
                   [90/2575367] via 30.30.30.1, 00:03:29, GigabitEthernet2/0
```

Theres many ways you can play with route filtering. With this we conclude that topic for now.

# EIGRP STUB

Here we will be introducing an EIGRP concept of `STUB` router. EIGRP stub router does not advertise routes that it learnes from other EIGRP peers. It advertises by default only connected and summary routes, but it can be configured to receive or advertise whatever combination we want (redistributed routes,connected or summary routes). 

STUB router is usefull and most often used at branch locations or remote sites that do not have large number or routers or networks, and because those routers are usually on the WAN edge and by implementing the STUB router we are limiting our `QUERY` domain, meaning when some route to reach x.x.x.x network goes active in the EIGRP topology table, that router will start sending Queries to neighboring routers asking if anyone got a route to reach x.x.x.x network. 
This way if we dont want our query to reach so far as our branch is, we can limit that by making branch routers EIGRP STUBs. They tell their neighbor's in the hello packet that they are STUBs and they wont receive queries from neighbors.

Important thing to consider is , if there's other networks behind STUB router, remember that by default it doesn't advertise them.

Lets check the configuration on our BRANCH router and the routes it learned:

```
BRANCH#show ip route
      50.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        50.50.50.0/24 is directly connected, GigabitEthernet3/0
L        50.50.50.2/32 is directly connected, GigabitEthernet3/0
      60.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        60.60.60.0/24 is directly connected, GigabitEthernet1/0
L        60.60.60.1/32 is directly connected, GigabitEthernet1/0
      150.1.0.0/24 is subnetted, 1 subnets
D        150.1.1.0 [90/2575360] via 50.50.50.1, 00:00:12, GigabitEthernet3/0
      160.1.0.0/24 is subnetted, 1 subnets
D        160.1.1.0 [90/2575360] via 50.50.50.1, 00:00:12, GigabitEthernet3/0
      170.1.0.0/24 is subnetted, 1 subnets
D        170.1.1.0 [90/10880] via 60.60.60.2, 00:00:28, GigabitEthernet1/0
```

To shorten the table, we will display only important routes, that is the neighbors routes and routes from NET. The same thing can be seen on the BRANCH_2 router. As we can see all is well.

Now lets check the routes on R2 to prove that he is receiving the route for network 170.1.1.0/24 from BRANCH_2.

```
R2(config)#do sh ip route
      150.1.0.0/24 is subnetted, 1 subnets
D        150.1.1.0 [90/2575360] via 50.50.50.1, 00:08:20, GigabitEthernet3/0
      160.1.0.0/24 is subnetted, 1 subnets
D        160.1.1.0 [90/2575360] via 50.50.50.1, 00:08:20, GigabitEthernet3/0
      170.1.0.0/24 is subnetted, 1 subnets
D        170.1.1.0 [90/16000] via 50.50.50.2, 00:07:53, GigabitEthernet3/0
```

As we can see he is indeed receiving the routes from BRANCH routers. Now lets make our BRANCH router a STUB router, lets see what will happen.

```
BRANCH(config)#router eigrp EIGRP
BRANCH(config-router)#address-family ipv4 autonomous-system 1
BRANCH(config-router-af)#eigrp stub
```

After making BRANCH a stub router neighborship gets reset and check the routing table on R2:

```
R2#show ip route
      150.1.0.0/24 is subnetted, 1 subnets
D        150.1.1.0 [90/2570240] via 20.20.20.1, 01:01:33, GigabitEthernet2/0
      160.1.0.0/24 is subnetted, 1 subnets
D        160.1.1.0 [90/2570240] via 20.20.20.1, 01:01:33, GigabitEthernet2/0
```


It lost the route to 170.1.1.0/24 network because BRANCH router is now a STUB router and is not advertising any outside networks, this is the problem you need to be carefull about. 
If you check the routing table on BRANCH_2 you would also see that it lost all routes to the outside.

We can also see how R2 sees BRANCH router as a stub with this command:

```
R2#show ip eigrp neighbors  detail
EIGRP-IPv4 VR(EIGRP) Address-Family Neighbors for AS(1)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
2   50.50.50.2              Gi3/0                    12 00:11:41   85   510  0  42
   Version 11.0/2.0, Retrans: 0, Retries: 0, Prefixes: 1
   Topology-ids from peer - 0 
   Stub Peer Advertising (CONNECTED SUMMARY ) Routes
   Suppressing queries
```

There's a way to resolve this issue in newer versions of IOS, since im labbing this on IOS 15.2 i dont have the command to achieve that, its called EIGRP stub site feature. But basically the resolution is to make BRANCH router recognize which interfaces are WAN and which are LAN. And then EIGRP stub site feature allows STUB router to advertise itself as a stub to peers ONLY on the WAN interfaces but allow it to exchange routes learned on LAN interfaces. There's more details about this so you can research it incase you ever implement EIGRP stubs.

Thank you for reading, and see you on the next one! All the best!






