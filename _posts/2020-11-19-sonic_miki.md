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

Lets head over to the tab IP -> IPsec -> Peer Profiles and configure the profile in which we will specify the encryption method which will be used to setup Phase 1 secure tunnel in which two peers will negotiate.
![miki_peerprofile](/assets/img/sample/miki_peerprofile.png)



