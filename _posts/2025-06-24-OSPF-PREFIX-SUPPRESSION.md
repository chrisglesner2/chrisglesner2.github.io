---
title: OSPF PREFIX SUPPRESSION
date: 2025-06-24 07:53:15 +0000
categories: [CCIE, NETWORKING, CISCO, EIGRP]
tags: [ccie, networking]     # TAG names should always be lowercase
---

# Theory

A transit network in OSPF is simply a network of which connects routers together. According to RFC 6860, transit-only networks are advertised in LSAs, but they're generally not needed for data traffic. With that in mind, OSPF can natively suppress these routes. Since Type-2 LSAs describe either broadcast or non-broadcast multi-access networks, a transit network would conceivably have multiple hosts on it. Once OSPF prefix suppression is enabled on an interface,however, the DR for that segment will advertise the transit network with a /32 mask. This will cause other routers to interpret this as an invalid prefix, causing them to exclude it from their routing tables.

## Topology

In our lab here, I'm modeling this after Narbik's CCIE Enterprise Infrastructure book. R1 and R3 have a transit network of 13.1.1.0/24. 

![OSPF Prefix Suppression Topology](/assets/ospf-prefix-suppression-topology.png)

## Configuration

R5 has a Type-2 LSA in Area 1 for the 13.1.1.1/24 network. In this case, R1 (the DR on the segment), says that R1 (0.0.0.1 RID) and R3 (0.0.0.3 RID) are attached to this 13.1.1.0/24 transit network.

```
R5#sh ip ospf 1 1 database network 13.1.1.1

            OSPF Router with ID (0.0.0.5) (Process ID 1)

                Net Link States (Area 1)

  LS age: 1801
  Options: (No TOS-capability, DC)
  LS Type: Network Links
  Link State ID: 13.1.1.1 (address of Designated Router)
  Advertising Router: 0.0.0.1
  LS Seq Number: 80000007
  Checksum: 0xF628
  Length: 32
  Network Mask: /24
        Attached Router: 0.0.0.1
        Attached Router: 0.0.0.3
```

You can either apply suppression globally or on a per-interface basis. I'll be applying this on R3's Gi0/1 interface and R1's Gi0/3 interface.

```
R1(config)#int gi0/3
R1(config-if)#ip ospf prefix-suppression 

R3(config)#int gi0/1
R3(config-if)#ip ospf prefix
R3(config-if)#ip ospf prefix-suppression 
```

Now, R5 has a Type-2 LSA from R1 for the transit network with a mask of /32. When a router sees a /32 mask regarding a transit network in a Type-2 LSA, it understands that this isn't for forwarding packets. As a result, R5 doesn't have a route.

```
R5#sh ip route 13.1.1.1
% Network not in table
```

## Notes

All of this is to preserve the adjacency between R1 and R3. We want to enable OSPF on that interface, have them form an adjacency, and exchange routes. However, we don't want them to advertise their transit link. Given the restriction of needing an adjacency, we couldn't simply remove the **network** command and strip the 13.1.1.0/24 network from the LSDB.