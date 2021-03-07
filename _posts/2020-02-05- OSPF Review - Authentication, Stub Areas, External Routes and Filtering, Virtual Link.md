---
title: OSPF Review - Authentication, Stub Areas, LSA's, Route Filtering, Virtual Link
author: Jure Veraja
date: 2021-02-05 18:15:00 +0100
categories: [ccnp]
tags: [ospf]
math: true
---
Here's the topology we will be using for our OSPF configurations.

![ospf_topo](/assets/img/sample/ospf_topo.png)


Just to point out I won’t go into details about every LSA type, but we will describe them as we go, especially external ones.

# OSPF AUTHENTICATION

First lets see how authentication goes between OSPF routers:
You can do it in two ways, either define the ospf authentication option via the interface or in router ospf process, we will do both option for practice, first option will be defining both authentication method and ospf key via interface configuration menu; when you don’t use message-digest option, your key is not hashed and its clear-text, for more security use the second method md5.

```
R6(config-if)#int g2/0
R6(config-if)#ip ospf authentication
R6(config-if)#ip ospf authentication-key ospf_key
---------------------------------------------------------------
R8(config-if)#int g2/0
R8(config-if)#ip ospf authentication
R8(config-if)#ip ospf authentication-key ospf_key
```

Second method would be using md5, and demonstrating using the authentication method in router ospf process, when using this method you affect all links connected to that area so be carefull about that.

```
R4_ABR(config)#router ospf 1
R4_ABR(config-router)#area 1 authentication message-digest
R4_ABR(config)#int g1/0
R4_ABR(config-if)#ip ospf message-digest-key 1 md5 ospf_key
-----------------------------------------------------------------------------
R6(config)#router ospf 1
R6(config-router)#area 1 authentication message-digest
R6(config)#int g1/0
R6(config-if)#ip ospf message-digest-key 1 md5 ospf_key
```
You could leave those two routers without typing the key, they need to match the authentication option between them, otherwise the neighborship would not form. If you don’t type the keys, but match the type of authentication , neighborship will form. Here’s the `debug ip ospf adjacency` info which you will receive if there’s mismatch in the authentication method, Type 0 is no auth, Type 1 is plain text authentication, Type 2 is md5 auth:
```
OSPF-1 ADJ   Gi1/0: Rcv pkt from 10.1.46.4 : Mismatched Authentication type. Input packet specified type 1, we use type 0
```

To verify authentication method and whether it is enabled on the interface or area, you can use `show ip ospf` or `show ip ospf interface` , keep in mind if you used per interface authentication the show ip ospf option will say “area has no authentication”, you have to use the “show ip ospf interface” , then it will say that “message digest authentication enabled” . 

For example on R8 we enabled plain-text authentication on interface and you can see its not showing:

```
R8#show ip ospf
***output omitted***
Area 1
        Number of interfaces in this area is 1
        Area has no authentication
----------------------------------------------------
R8#show ip ospf interface
***output omitted***
Simple password authentication enabled
```

And on R4_ABR since we enabled authentication under router ospf process, it shows in “show ip ospf” command:

```
R4_ABR#show ip ospf
***output omitted***
Area 1
        Number of interfaces in this area is 1
        Area has message digest authentication
```

# OSPF ROUTE FILTERING

OSPF is a bit different than EIGRP when it comes to route filtering. In OSPF you can filter routes coming and going between area (inter-area) otherwise known as LSA Type 3 filtering, which can only be done on ABR ofcourse. Also keep in mind that every router in specific area have to have the same ospf database.

### Filter Lists

LSA Type 3 serves to summarize information from LSA1 and LSA2 from one area and advertise it as summary route to other area.

With inter-area filtering, we’re basically making ABR not advertise Type 3 summary LSA’s, so its being filtered before its even created. Lets demonstrate that by making R4_ABR not advertise type 3 LSA for 10.0.12.0/24 network.

First lets check R6 ospf database to prove that we have the LSA Type 3 for 10.0.12.0/24 network.

```
R6#show ip ospf database summary 10.0.12.0

            OSPF Router with ID (10.1.68.6) (Process ID 1)

                Summary Net Link States (Area 1)

  Routing Bit Set on this LSA in topology Base with MTID 0
  LS age: 245
  Options: (No TOS-capability, DC, Upward)
  LS Type: Summary Links(Network)
  Link State ID: 10.0.12.0 (summary Network Number)
  Advertising Router: 10.1.46.4
  LS Seq Number: 80000001
  Checksum: 0xD80F
  Length: 28
  Network Mask: /24
        MTID: 0         Metric: 2
```

Now lets create the filter:

```
R4_ABR(config)#ip prefix-list DENY_10.0.12.0 deny 10.0.12.0/24
R4_ABR(config)#ip prefix-list DENY_10.0.12.0 permit 0.0.0.0 le 32
R4_ABR(config)#router ospf 1
R4_ABR(config-router)#area 1 filter-list prefix DENY_10.0.12.0 in
```

Now you can see that R6 lost the summary route to 10.0.12.0/24 , it’s empty:

```
R6#show ip ospf database summary 10.0.12.0

            OSPF Router with ID (10.1.68.6) (Process ID 1)
```

When you use the “IN” keyword in the filter-list , you basically filter that specific prefix from being sent into the specified area. You can also use the “OUT” keyword which would mean don’t send that network out of the specified area.

### Area Range Command

Another way to do inter-area filtering is by using the command area # range x.x.x.x/x which is usually used for inter-area route summarization but when you insert the keyword at the end `NOT-ADVERTISE` it makes the command a filter. It doesn’t require any ACL or Prefix-Lists.

Lets try out again on the same network as before, I removed the ip prefix-list from before. Router 6 has the route for 10.0.12.0/24.

```
R6#show ip route
      10.0.0.0/8 is variably subnetted, 11 subnets, 2 masks
O IA     10.0.12.0/24 [110/3] via 10.1.46.4, 00:00:10, GigabitEthernet1/0
O IA     10.0.13.0/24 [110/3] via 10.1.46.4, 00:09:16, GigabitEthernet1/0
O IA     10.0.14.0/24 [110/2] via 10.1.46.4, 00:09:16, GigabitEthernet1/0
***removed some output***
```

Now implementing the area range command on R4_ABR he loses the route and Type 3 LSA.

```
R4_ABR(config-router)#area 0 range 10.0.12.0 255.255.255.0 not-advertise
```

```
R6#show ip route
      10.0.0.0/8 is variably subnetted, 11 subnets, 2 masks
O IA     10.0.13.0/24 [110/3] via 10.1.46.4, 00:09:16, GigabitEthernet1/0
O IA     10.0.14.0/24 [110/2] via 10.1.46.4, 00:09:16, GigabitEthernet1/0
***removed some output***
```

You need to be carefull with this command because it uses only Router and Network LSA’s and then it prevents you from converting those LSA’s into Type 3 summary LSA. 
So if you would try to use this command on some other ABR who is in an area in which that network did not originate, like R9, `area 2 range 10.0.12.0 255.255.255.0 not-advertise` he would accept the command, but nothing would happen. R10 would still receive the route for that network, because R9 does not belong to Area 0 where that network originated from, and he did not have Router or Network LSA’s.

### Route Filtering using Distribute-Lists (Filtering before the LSA comes into the routing table)

This type of filtering allows LSA’s into the database unlike before, but doesn’t let them reach the routing table.

Remember you can’t deny some routers in an area from learning specific LSA but allow others, they must all have identical database, but you can deny them from installing the specific LSA in the routing table.

For this we use `distribute-lists` which we demonstrated in previous posts. In EIGRP you used distribute-lists to prevent routes from coming INTO the eigrp table, but in OSPF its different as we explained. You can use distribute-list on any router, unlike before when we could only do it on ABR.

So lets once again deny the 10.0.12.0/24 network from reaching the R6 routing table, but this time we will let it come into our ospf database.

```
R6(config)#access-list 10 deny 10.0.12.0 0.0.0.255
R6(config)#access-list 10 permit any
R6(config)#router ospf 1
R6(config-router)#distribute-list 10 in
```

Notice how we did this command directly on R6, you don’t need to do it on ABR. Now lets verify:

```
R6(config-router)#do sh ip route
      10.0.0.0/8 is variably subnetted, 10 subnets, 2 masks
O IA     10.0.13.0/24 [110/3] via 10.1.46.4, 00:00:02, GigabitEthernet1/0
O IA     10.0.14.0/24 [110/2] via 10.1.46.4, 00:00:02, GigabitEthernet1/0
***output deleted***
```

```
R6#show ip ospf database summary 10.0.12.0

            OSPF Router with ID (10.1.68.6) (Process ID 1)

                Summary Net Link States (Area 1)

  Routing Bit Set on this LSA in topology Base with MTID 0
  LS age: 519
  Options: (No TOS-capability, DC, Upward)
  LS Type: Summary Links(Network)
  Link State ID: 10.0.12.0 (summary Network Number)
  Advertising Router: 10.1.46.4
  LS Seq Number: 80000001
  Checksum: 0xD80F
  Length: 28
  Network Mask: /24
        MTID: 0         Metric: 2
```

You can see how there’s not route in routing table, but he still has LSA Type 3 summary for that network as he should. You can also do this command on ABR without affecting other routers, because they still need to receive the LSA , ABR can’t deny it, he would just filter his own routing table.

Altough you must be carefull when you’re creating distribute-list when you have a single point of exit for some routers, for example if I would create a distribute-list to deny that network from coming into the routing table on R9, then R10 couldn’t also reach it even though R10 would have the LSA and would install it in routing table, R9 would lose the route to reach 10.0.12.0 since we filtered it. Keep that in mind.

# OSPF DEFAULT ROUTE

You can inject default route in OSPF by using `default information originate | always` command.
The command would look at your routing table and check if you have any default routes (0.0.0.0), static or from some other source and it would inject it into the ospf domain (all ospf routers would receive it,in all areas) as Type 5 external LSA. You can also add the “always” keyword so the default route gets advertised even if you don’t have a default route yourself, in that case all other routers would receive 0.0.0.0 and next-hop would be you.

It is advertised as External Type 5 route with type 2 submetric, as we know external routes can be type-1 or type-2. Type 2 means that metric doesn’t change along the way, it stays the same, in case of default routes it is 1. Important note is that type 2 external routes are usually advertised with metric of 20! But default-route is advertised with metric of 1.

Lets create some null default route on R1 for demonstration, and propagate it:

```
R1(config)#ip route 0.0.0.0 0.0.0.0 null0
R1(config)#router ospf 1
R1(config)#default-information originate
```

You can check for example on R8 how he got the route,and you can see the metric of 1. Also notice the O*E2 route signature, means its external route with type 2 submetric.

```
R8#show ip route
O*E2  0.0.0.0/0 [110/1] via 10.1.68.6, 00:01:59, GigabitEthernet2/0
***output deleted***
```

# OSPF STUB AREAS

There’s four types of OSPF stub areas: stub, totally stubby, not so stubby (NSSA), and totally NSSA.
We will go over each one and demonstrate its purposes.

Basically the difference between the stubby and totally stubby is one allows type 3 LSA’s and denies Type 5 external LSA’s while the others denies both (totally stubby, totally nssa).

### Stub Area and Totally Stubby Area

By making an area `stub area` your ospf process will reset, and both routers need to match the area type.
Stub area makes router receive a default route O*IA via ABR and other inter-area routes ( O IA ). It will not receive any external redistributed routes.

For example lets redistribute something from ASBR and check the routing table on lets say R8. I will create a Loopback0 on ASBR and redistribute it using redistribute connected command under ospf process:

```
R3_ASBR(config)#int l0
R3_ASBR(config-if)#ip add 100.100.100.1 255.255.255.0
R3_ASBR(config-if)#router ospf 1
R3_ASBR(config-router)#redistribute connected subnets
```

R8 routing table:

```
R8#show ip route
O IA     10.2.57.0/24 [110/6] via 10.1.68.6, 00:00:46, GigabitEthernet2/0
O IA     10.2.79.0/24 [110/7] via 10.1.68.6, 00:00:46, GigabitEthernet2/0
      100.0.0.0/24 is subnetted, 1 subnets
O E2     100.100.100.0 [110/20] via 10.1.68.6, 00:02:40, GigabitEthernet2/0
***output deleted***
```

Now let’s make area 1 stub area and see what happens, you have to do this for all routers in area 1, they need to match the area type:

```
R8(config)#router ospf 1
R8(config-router)#area 1 stub
*Feb  2 17:49:11.795: %OSPF-5-ADJCHG: Process 1, Nbr 10.1.68.6 on GigabitEthernet2/0 from FULL to DOWN, Neighbor Down: Adjacency forced to reset
```

Now if you check the routing table on R8 you won’t find external routes, but you will find a default route pointing to the R6 and then R6 also received default route pointing to ABR, as well as summary  routes.

```
R8#show ip route
O*IA  0.0.0.0/0 [110/3] via 10.1.68.6, 00:01:51, GigabitEthernet2/0
O IA     10.2.57.0/24 [110/6] via 10.1.68.6, 00:02:46, GigabitEthernet2/0
O IA     10.2.79.0/24 [110/7] via 10.1.68.6, 00:02:46, GigabitEthernet2/0
***output deleted***
```

Now lets say you have a single point of exit from some area and you decide you don’t need those summary inter-area routes at all, you can just use the default route to get out, you can achieve that by making an area `Totally Stubby Area`. You create it by adding the `no-summary` keyword to the area 1 stub command. And one more thing to note is that you configure this on ABR only, as far as other routers in area 1 they remain only stub routers and that is enough for them.

Before adding the `area 1 stub no-summary` to the R4_ABR you can also check `show ip ospf database summary self-originate` to see that he is creating summaries and injecting them into area 1, after we add this command he will delete those summaries.

```
R4_ABR(config)#router ospf 1
R4_ABR(config-router)#area 1 stub no-summary 
```
Routing table on R8, as you can see he doesn’t have summary routes anymore:

```
R8#show ip route
O*IA  0.0.0.0/0 [110/3] via 10.1.68.6, 00:00:24, GigabitEthernet2/0
      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
O        10.1.46.0/24 [110/2] via 10.1.68.6, 00:11:30, GigabitEthernet2/0
C        10.1.68.0/24 is directly connected, GigabitEthernet2/0
L        10.1.68.8/32 is directly connected, GigabitEthernet2/0
```

You can also verify on ABR that area is stub with no-summary command added to it:

```
R4_ABR#show ip ospf
***output deleted***
Area 1
        Number of interfaces in this area is 1
        It is a stub area, no summary LSA in this area
***output deleted***
```

### Not So Stubby Area (NSSA)

Previously we’ve seen how we filter summary routes and external routes from our ospf database and hence the routing table. But if you still want to do redistribution inside area 1 you need to do something about it, cuz stub area by default denies you from redistributing inside that area.

To do redistribute something inside area 1 we make it NSSA, which still filters external LSA’s coming into but allows us to redistribute inside the area. Now let’s say we want to redistribute something from outside via R8 to area 1, we create a loopback on R8 and redistribute it, but notice the message we receive when you try to do that without creating area 1 to be NSSA.

```
R8(config)#int l0
R8(config-if)#ip add 200.200.200.1 255.255.255.0
R8(config-if)#router ospf 1
R8(config-router)#redistribute connected subnets
%OSPF-4-ASBR_WITHOUT_VALID_AREA: Router is currently an ASBR while having only one area which is a stub area
```

This means you are not attached to any area that can receive external LSA’s. Now lets fix that by making area 1 NSSA, but first you must unmake the previous stub configuration otherwise you would get warning:

```
R8(config-router)#area 1 nssa 
% OSPF: Area is configured as stub area already
```

```
R8(config-router)#no area 1 stub
***neighborship goes down*** 
R8(config-router)#area 1 nssa 
```

You have to make all routers inside area 1 know its an NSSA area. After you did that, you should receive the route to 200.200.200.0/24 network which you previously redistributed from R8. Let’s check the routing table on R6:

```
R6#show ip route
***output deleted***
O IA     10.2.57.0/24 [110/5] via 10.1.46.4, 00:04:43, GigabitEthernet1/0
O IA     10.2.79.0/24 [110/6] via 10.1.46.4, 00:04:43, GigabitEthernet1/0
O IA     10.52.109.0/24 [110/7] via 10.1.46.4, 00:04:43, GigabitEthernet1/0
O N2  200.200.200.0/24 [110/20] via 10.1.68.8, 00:05:13, GigabitEthernet2/0
```

Notice the new route with mark `O N2`, that’s a distinguisher for NSSA redistributed routes, N2 is subtype juts like Type-1 or Type-2 for external Type 5 routes, it can also be N1. Important point to take is that those routes are no longer Type 5 routes, NSSA still prevent’s type 5 route’s , instead its creating a `TYPE 7 LSA` . And since we made area 1 NSSA and all routers in it will accept this type of LSA’s. You should see O N2 on all routers inside area 1 , or the area you made NSSA.

But other areas will not allow Type 7 LSA’s since other area’s are not NSSA, they are just regular areas, like for example area 0. So ABR in our case R4_ABR is converting those Type 7 routes to Type 5 routes, routes and injecting them into area 0 and other area’s , making them think its just regular old external route. Let’s check the ospf database on ABR to prove that:

```
R4_ABR#show ip ospf database external self-originate

            OSPF Router with ID (10.1.46.4) (Process ID 1)

                Type-5 AS External Link States

  LS age: 849
  Options: (No TOS-capability, DC, Upward)
  LS Type: AS External Link
  Link State ID: 200.200.200.0 (External Network Number )
  Advertising Router: 10.1.46.4
R4_ABR#show ip ospf database
                Type-7 AS External Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Tag
200.200.200.0   10.1.68.8       930         0x80000001 0x000984 0

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
100.100.100.0   200.200.200.1   819         0x80000002 0x00EE28 0
200.200.200.0   10.1.46.4       891         0x80000001 0x005061 0
```

And lastly lets check how R1 receives the route to 200.200.200.0/24:

```
R1#show ip route
      100.0.0.0/24 is subnetted, 1 subnets
O E2     100.100.100.0 [110/20] via 10.0.13.3, 00:45:46, GigabitEthernet3/0
O E2  200.200.200.0/24 [110/20] via 10.0.14.4, 00:13:54, GigabitEthernet2/0
```

Another thing to note here is that before when were using area 1 as stub area, ABR was injecting default route, but now since Type 3 LSA’s are permitted in NSSA , he is no longer doing that, but you can still do it if you want to by adding the keywords `area 1 NSSA default-information-originate` .

One more thing we didn’t discuss regarding external routes in OSPF is about what happens when external route gets propagated through the ospf domain, how can let’s say R9 know where the 200.200.200./24 OE 2 route is, how can he know whats the next-hop, since ASBR redistribute OE 2 route with himself as next-hop, ABR’s need to do something about it when they inject that into their own area’s, so they create Type 4 LSA, telling the router’s in specific area (in our case area 2) to get to OE2 route (200.200.200.0/24) send the packets to me,making ABR next hop.

```
R9#show ip route
O E2  200.200.200.0/24 [110/20] via 10.2.79.7, 00:14:25, GigabitEthernet2/0
R9#show ip ospf database
Summary ASB Link States (Area 2)

Link ID         ADV Router      Age         Seq#       Checksum
200.200.200.1   10.52.109.9     976         0x80000001 0x0037EF
```


# OSPF VIRTUAL LINKS

If you check out R10 , you will notice that he is in area 52, and is connecting to area 2 via R9. OSPF basic rule says, each area must touch backbone area (area 0), so what is R10 doing there?

Well sometimes we are faced with situations where we can’t have all routers touching our backbone, so we have to make them virtual think they area, that’s where ospf virtual links come into play.

We basically create a transit area through which our virtual link will go, and will make R10 think he is directly connected to area 0. In our case transit area is area 2. Virtual links or virtual tunnels are built between the router’s in the same area (R9 and R5_ABR), and they are known as tunnel endpoints. Atleast one of the endpoints have to be connected to area 0 to make this configuration work, and an area must not be stub area.


Each router identifies the remote router by its RID not the IP address so don't make that mistake. in configuration: area area-id virtual-link endpoint- RID

Lets create a virtual link on our endpoints in area 2 (R9 and R5_ABR) so that R10 can be virtually connected to area 0:

```
R9(config)#router ospf 1
R9(config-router)#area 2 virtual-link 10.2.57.5

R5_ABR(config)#router ospf 1
R5_ABR(config-router)#area 2 virtual-link 10.52.109.9
```

Now the tunnel should come up and you can verify ` show ip ospf virtual-links` 

```
R5_ABR#show ip ospf virtual-links  
Virtual Link OSPF_VL0 to router 10.52.109.9 is up
  Run as demand circuit
  DoNotAge LSA allowed.
  Transit area 2, via interface GigabitEthernet1/0
***output deleted***
```

You can also check neighborship on R5_ABR it will be seen as he is directly connected to R9, hence making R10 virtually touching the router connected to area 0, and being able to receive routes:

```
R5_ABR#show ip ospf neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.0.25.2         1   FULL/BDR        00:00:34    10.0.25.2       GigabitEthernet2/0
10.52.109.9       0   FULL/  -           -        10.2.79.9       OSPF_VL0
10.2.79.7         1   FULL/DR         00:00:35    10.2.57.7       GigabitEthernet1/0
```

That’s the basics of virtual-links for now, there’s more then meets the eye here, so be carefull with the design.

Hope you enjoyed the read, thank you and see you in the next one!









