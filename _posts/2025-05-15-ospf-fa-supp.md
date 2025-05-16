---
title: OSPF Forwarding Address Suppression
date: 2025-05-15
categories: ["Routing", "Networking"]
tags: ["Cisco", "Networking", "OSPF", "Service Provider", "Enterprise"]
---

# OSPF Forwarding Address Suppression

The OSPF FA Suppression feature in Translated Type-5 LSAs cause an NSSA ABR to translate Type-7 LSAs to Type-5 LSAs, but the forwarding address in the LSA changes to 0.0.0.0 rather than the node that originates the Type-7 LSA. As a result, routers do not need to inject the original forwarding address into the backbone in order for traffic to reach the original ASBR. The main reasoning for the feature that I've found in Cisco documentation is to consolidate the amount of routes of which are introduced into the backbone. 

# Topology

Let's begin with our topology. 

<img src="/assets/ospf-supp-top.png" alt="OSPF FA Suppression Topology">

It's not the prettiest -- but hey, I'm not an artist. This topology was originally published by Rene at [gns3vault.com](https://gns3vault.com/ospf/ospf-suppress-forward-address). 

I'll assume the reader understands the OSPF configuration to an extent. In a nutshell, Evelyn is peered to Charlie via RIP. Evelyn has advertised Loopback0 to Charlie; Charlie is redistributing this into the NSSA area.


Here's a snippet of Charlie's OSPF database. You can see this node is acting as an ASBR and originating the Type-7 LSA into Area 1 for Evelyn's Loopback0. 

```
Charlie#sh ip ospf database nssa-external self-originate 

            OSPF Router with ID (192.168.35.3) (Process ID 1)

                Type-7 AS External Link States (Area 1)

  LS age: 88
  Options: (No TOS-capability, Type 7/5 translation, DC, Upward)
  LS Type: AS External Link
  Link State ID: 1.0.0.0 (External Network Number )
  Advertising Router: 192.168.35.3
  LS Seq Number: 80000001
  Checksum: 0x3656
  Length: 36
  Network Mask: /8
        Metric Type: 2 (Larger than any link state path)
        MTID: 0 
        Metric: 20 
        Forward Address: 192.168.13.3
        External Route Tag: 0
```

If we look at Berta inside of Area 2, you can see this node has an external Type-5 LSA with a forwarding address of 192.168.13.3 (Charlie's Gi0/0 IP address).

```
Berta#sh ip ospf database external 

            OSPF Router with ID (192.168.24.4) (Process ID 1)

                Type-5 AS External Link States

  LS age: 177
  Options: (No TOS-capability, DC, Upward)
  LS Type: AS External Link
  Link State ID: 1.0.0.0 (External Network Number )
  Advertising Router: 192.168.13.1
  LS Seq Number: 80000001
  Checksum: 0x713D
  Length: 36
  Network Mask: /8
        Metric Type: 2 (Larger than any link state path)
        MTID: 0 
        Metric: 20 
        Forward Address: 192.168.13.3
        External Route Tag: 0
```

Now, I'll use a prefix list on Jake to ensure that we're filtering for 192.168.13.0/24. 

```
area 2 filter-list prefix DENY-FA in
ip prefix-list DENY-FA seq 10 deny 192.168.13.0/24
ip prefix-list DENY-FA seq 20 permit 0.0.0.0/0 le 32
```
Berta still has the LSA for Evelyn's Loopback0, however, I cannot ping as Berta can't recursively route to the forwarding address of 192.168.13.3. To rectify this, I'll have Alan (the ABR) suppress the forwarding address when it translates the Type 5 LSA into a Type 7 LSA.

```
area 1 nssa translate type7 suppress-fa 
```

I can confirm this by looking at the Type 5 LSA from Berta's perspective. 

```
Berta#sh ip ospf da ex

            OSPF Router with ID (192.168.24.4) (Process ID 1)

                Type-5 AS External Link States

  LS age: 26
  Options: (No TOS-capability, DC, Upward)
  LS Type: AS External Link
  Link State ID: 1.0.0.0 (External Network Number )
  Advertising Router: 192.168.13.1
  LS Seq Number: 80000002
  Checksum: 0x6BBB
  Length: 36
  Network Mask: /8
        Metric Type: 2 (Larger than any link state path)
        MTID: 0 
        Metric: 20 
        Forward Address: 0.0.0.0
        External Route Tag: 0
```

Works like a charm! I can ping as well.

```
Berta#ping 1.1.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1.1.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/3/3 ms
```

I'll admit I wish I knew some of the historical knowledge around this piece of technology. I can only intuit that routers were much less powerful a couple of decades ago. As a result, they couldn't run an SPF on a fairly frequent basis and maintain an LSDB with redundant LSAs.  

This suppression does introduce a problem though. If there were two ABRs in the path, you would usually load-balance traffic among those two nodes. However, when you suppress the forwarding address, all traffic will be 'steered' through the ABR originating the Type-5 LSA. 