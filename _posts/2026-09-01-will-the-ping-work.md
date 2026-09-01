---
title: Will the ping work?
date: 2026-09-01 17:50:00 +0530
categories: [Networking]
tags: [vlans, switching, trunks]
---

We will start with something basic, but **fun**.

Tell me this, what will happen in the following scenarios? Think, then look at the solution.

We will work with this topology for all the questions in this post.

![Two switches connected by a trunk. HostA is in VLAN 10 on SW1. HostB is in VLAN 20 on SW2.](/assets/img/posts/will-the-ping-work/topology-trunk.png)

## Scenario A

In the above topology: HostA (`10.10.10.10`) -- VLAN 10 -- SW1 -- TRUNK -- SW2 -- VLAN 20 -- HostB (`10.10.10.20`).

Will the ping work from Host A to Host B?

**Answer:** NO. It fails.

As we know, VLANs are local to the switches, and when we connect two switches via trunk, the trunk carries multiple VLANs and preserves their separation via the 802.1Q tag. So HostA sends traffic (let's say ARP to begin with) to SW1, it gets placed in VLAN 10. We verify that using `show mac address-table vlan` — this is the isolation.

Then the ARP broadcast reaches the trunk, gets tagged as VLAN 10, and reaches SW2 as part of VLAN 10 still. HostA's ARP broadcast is confined to VLAN 10. HostB (VLAN 20) never receives it, so ARP never resolves — ping fails at resolution.

We will see HostA send ARP and it will reach the trunk, but then SW2 receives a VLAN 10 frame and floods it only within VLAN 10 — where there are no host ports (HostB is VLAN 20). So HostB still never gets it, thus ping fails.

`show mac address-table`

![MAC address table on SW1 showing HostB learned in VLAN 20](/assets/img/posts/will-the-ping-work/mac-table-sw1.png)

Note: SW1 _has_ learned HostB's MAC — but in the VLAN 20 table (learned across the trunk). HostA's frame is a VLAN 10 frame, and MAC lookups are **per-VLAN**, so the VLAN 10 lookup never sees the VLAN 20 entry. Knowing a MAC in a _different_ VLAN is useless — the isolation is by VLAN membership. A VLAN 10 lookup _never consults_ the VLAN 20 table.

**ARP packet capture**

![Packet capture of HostA sending ARP](/assets/img/posts/will-the-ping-work/arp-capture-1.png)

![Packet capture showing the ARP staying in VLAN 10](/assets/img/posts/will-the-ping-work/arp-capture-2.png)

## Scenario B

Same topology, but now the link between SW1 and SW2 is configured as access in the respective VLANs, like below. Will Host A in VLAN 10 successfully ping Host B in VLAN 20?

![SW1 and SW2 connected by access ports in VLAN 10 and VLAN 20](/assets/img/posts/will-the-ping-work/topology-access.png)

**Answer:** YES. Wait, different VLANs and still this works? Why?

Because access ports send and receive untagged frames. Tagging is the property of a trunk port. Because the frame from SW1 arrives untagged, SW2 doesn't know it was ever "VLAN 10." SW2 applies _its own_ access VLAN to the untagged frame, VLAN 20. Now the frame is in VLAN 20, where HostB lives. Two different VLAN numbers got **stitched into one broadcast domain** by an untagged link.

![Scenario B capture showing the path from HostA](/assets/img/posts/will-the-ping-work/scenario-b-1.png)

![Scenario B capture on the second switch](/assets/img/posts/will-the-ping-work/scenario-b-2.png)

No tagging of the packet.

![Capture showing the frame between switches is untagged](/assets/img/posts/will-the-ping-work/untagged-frame.png)

![Follow-up capture of the untagged access link](/assets/img/posts/will-the-ping-work/scenario-b-4.png)

**HostB learnt as VLAN 10 on SW1:**

![MAC table on SW1 showing HostB in VLAN 10](/assets/img/posts/will-the-ping-work/mac-sw1-hostb-vlan10.png)

**HostA learnt as VLAN 20 on SW2:**

![MAC table on SW2 showing HostA in VLAN 20](/assets/img/posts/will-the-ping-work/mac-sw2-hosta-vlan20.png)

## Scenario C

In the below topology, the link between SW1 and SW2 is a trunk with native VLAN 10 on SW1 Gig0/0 and native VLAN 20 on SW2 Gig0/0. Will the ping work?

![Trunk between SW1 and SW2 with mismatched native VLANs](/assets/img/posts/will-the-ping-work/topology-native-mismatch.png)

**Answer:** Nope. Why?
