---
title: Spanning Tree Transition States
date: 2026-10-03 15:20:00 +0530
categories: [Networking]
tags: [stp, switching, pvst]
---

For this one we are going back to basics.

We are going to understand STP (PVST; `protocol ieee`) transition states, i.e., what happens when we transition from one state to the other.

Let's recap what STP port states are:

| State      | Rx BPDU | Tx BPDU | Learn MAC (CAM) | Forward user frames |
| ---------- | ------- | ------- | --------------- | ------------------- |
| Disabled   | No      | No      | No              | No                  |
| Blocking   | Yes     | No      | No              | No                  |
| Listening  | Yes     | Yes     | No              | No                  |
| Learning   | Yes     | Yes     | Yes             | No                  |
| Forwarding | Yes     | Yes     | Yes             | Yes                 |

Let's take a simple example of the link on ACC-HQ-01 Gig0/1 towards HostA. The port facing the host is by default in designated state: no BPDU received, only forwarding. But it does go through the standard transition phase LIS -> LRN -> FWD.

![Lab topology with ACC-HQ-01 Gig0/1 facing HostA](/assets/img/posts/spanning-tree-transition-states/topology.png)

Details from port Gig0/1:

![show spanning-tree detail on ACC-HQ-01 Gig0/1, designated forwarding toward HostA](/assets/img/posts/spanning-tree-transition-states/gi0-1-detail.png)

Trivia: Compare with details from port Gig0/0 and see how only Designated Root information stays consistent, but designated bridge details vary based on each link.

![show spanning-tree detail on ACC-HQ-01 Gig0/0, root forwarding toward the distribution switch](/assets/img/posts/spanning-tree-transition-states/gi0-0-detail.png)

Okay, back to Gig0/1. Now I will shut the link and then no shut it and see at each stage what happens at each state.

1. `*Sep 21 05:01:36.768: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1, changed state to down`

2. CAM empty, STP details not populated, link is admin-down.

![Empty MAC table and no STP state on Gig0/1 while the port is admin-down](/assets/img/posts/spanning-tree-transition-states/admin-down.png)

3. No shut the link: Gig0/1. The port immediately transitions to LISTENING. BPDU sent count goes up (3 to 5 to 7). BPDU receive would also go up if it wasn't a host-facing port. No MAC entry still, no user traffic / ping allowed still.

`*Sep 21 05:28:00.118: STP: VLAN0010 Gi0/1 -> listening`

![Gig0/1 designated listening with BPDU sent climbing 3 to 7 and an empty CAM](/assets/img/posts/spanning-tree-transition-states/listening.png)

4. Port transitions to LEARNING state at exactly 15 seconds. BPDU sent increases; again, BPDU receive would also go up if it wasn't a host-facing port. MAC will be learnt (but CML 2.9 won't show for some reason).

`*Sep 21 05:28:15.118: STP: VLAN0010 Gi0/1 -> learning`

![Gig0/1 designated learning with BPDU sent still climbing and zero transitions to forwarding](/assets/img/posts/spanning-tree-transition-states/learning.png)

5. Finally, the port transitions into FORWARDING, again at exactly 15 seconds from the previous state. BPDU count goes up, CAM table remains populated.

`*Sep 21 05:28:30.120: STP: VLAN0010 Gi0/1 -> forwarding`

![Gig0/1 designated forwarding with a dynamic MAC learned on VLAN 10](/assets/img/posts/spanning-tree-transition-states/forwarding.png)

Time to wrap up this topic. You now understand the 30 seconds *until forwarding*: Forward Delay 15s in Listening, then another 15s in Learning.

All states have a purpose in accurately converging the spanning tree.

Their purpose is to make a role change (RP/DP/blocking) loop-safe! There is no handshake in STP, so it needs to wait it out for it to assume a stable role.

**Blocking:** I can't be either Root or Designated role. My purpose is to stay put, no learning, no forwarding.

**Listening:** I could be Root or Designated port. Or I could go back to Blocking. Let's "listen" to the incoming and outgoing BPDUs. Don't modify CAM yet (what if I go back to Blocking and incorrectly indicate on CAM that a particular MAC is learnt by me).

**Learning:** Now this seems stable enough and I could be Root or Designated. Let's install MAC for forwarding state to use this, so that the first packet is not flooded and is unicast. Another 15 seconds here.

**Forwarding:** Steady state (is definitely a Root or Designated port).
