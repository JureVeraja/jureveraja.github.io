---
title: IPSec Site-to-Site between Sonicwall and Mikrotik
author: Jure Veraja
date: 2020-11-19 18:33:00 +0100
categories: [Networking, Sonicwall-Mikrotik]
tags: [sonicwall, mikrotik]
math: true
---
In this post i will show you how to configure IPsec tunnel between Sonicwall and Mikrotik.

I assume you have some knowledge of RouterOS so lets first do the configuration on Mikrotik.

These are addresses on Mikrotik interfaces.
![miki_address](/assets/img/sample/miki_address.png)

Lets head over to the tab `IP -> IPsec -> Peer Profiles` and configure the profile in which we will specify the encryption/hashing method which will be used to setup Phase 1 secure tunnel in which two peers will negotiate.
![miki_peerprofile](/assets/img/sample/miki_peerprofile.png)

Choose the encryption/hashing method along with Diffie Helman group (DH) which best suits your needs and your environment.

`NAT traversal` allows systems behind NATs to request and establish secure connections on demand,and in order to have ESP packets traverse NAT.

`(DPD) Dead Peer Detection` is a method of detecting a dead Internet Key Exchange (IKE) peer. For example there can be situations where two peers loose connectivity to each other but they can't identify the loss of connectivity because the Security Associations (SA) which were formed during the PHASE1 negotiation remain in the UP state until their lifetime timer expires. What happens in this case is the traffic is then routed through the tunnel which actually doesn't exist.

After this lets go to the `Peers` tab and configure our peer which in this case would be Sonicwall.

![miki_peer1](/assets/img/sample/miki_peer1.png)

We specify the ip address of our remote peer, along with the Profile we just created in the step before. 
In our example we are using Authentication Method with pre shared key. 
For Exchange Mode and compatibility with Sonicwall i've had most success with `Agressive mode`, for whatever reason sometimes my tunnels would not establish using `Main mode`.

To sucessfuly and securely communicate using IPsec, the IKE is using Main mode or Aggressive mode (Phase 1) which authenticates and/or encrypts the peers. And Quick Mode (Phase 2) which negotiates and agrees on which traffic will be sent across the VPN.

Difference between Agressive and Main mode is that Aggressive sends fewer messages (packet exchanges)(3) while Main mode requires 6 messages during Phase 1 negotiation. Also Agressive mode does not provide `Peer Identity Protection`, meaning the peers exchange their identity without encryption, unless certificates are used.

So to conclude, Agressive Mode is not as secure as Main Mode, but it is faster. Also it is mostly used when one of the peers have dynamic external ip address.

Next, on the `Advanced` under the `Peers` we set our ID to user fqdn, with this ID we will present ourselves to the peer. You can type any value here. The reason i use `user fqdn` is the same reason im using Agressive mode. I've had problems with establishing connections while using different ID types. Also set the "send initial contact" if you're using dynamic ip address at this location, and you are initiating a connection to a sonicwall which has static public ip address.

![miki_peer2](/assets/img/sample/miki_peer2.png)

After this we will go to the `Policy Proposals` tab. Here we will configure encryption/hashing policy that the peers will use for Phase 2 negotiation. Phase 1 negotiates a security association (SA) between two IKE peers, which enables the peers to communicate securely in Phase 2 and form IPSec SA's. Both sides must have the same matching values.

![miki_policyproposal](/assets/img/sample/miki_policyproposal.png)

Now we come to the point where we need to specify what traffic we want to send through the IPSec tunnel. Go to the `Policies` tab and add the `Source Address` (traffic from your site which you want tunneled) , and the `Destination Address` (traffic at the remote site with which you want to communicate) in our case network.
![miki_policy1](/assets/img/sample/miki_policy1.png)
On the `Action` tab window, for Action: specify `encrypt`, meaning the traffic from your network to remote network will be encrypted. Set Level: to `unique` for the situations when you are having multiple subnets crossing the tunnel. If you leave it at Level: `require`, only the first policy will match and be pushed through the tunnel. For IPsec Protocols: use `esp`, and use the `Tunnel` mode which encrypts whole IP packet.

![miki_policy2](/assets/img/sample/miki_policy2.png)









