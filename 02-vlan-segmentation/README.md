# Lab 2 — VLAN Segmentation and Inter-VLAN Routing

## Goal

Split one physical switch into two separate logical networks using VLANs, then let those VLANs talk to each other through a single router link using the router-on-a-stick method.

## Topology and VLAN plan

One router, one switch, and four PCs. Two VLANs separate the traffic.

| VLAN | Name | Switch ports | Network | Gateway |
|------|------|--------------|---------|---------|
| 10 | Students | Fa0/2 – Fa0/3 | 192.168.10.0/24 | 192.168.10.1 |
| 20 | Faculty | Fa0/4 – Fa0/5 | 192.168.20.0/24 | 192.168.20.1 |

![Network topology](screenshots/01-network-topology.png)

## Switch: create VLANs and assign ports

```cisco
vlan 10
 name Students
vlan 20
 name Faculty
exit

! Put the Students ports into VLAN 10 (access mode = belongs to one VLAN)
interface range fa0/2-3
 switchport mode access
 switchport access vlan 10
exit

! Put the Faculty ports into VLAN 20
interface range fa0/4-5
 switchport mode access
 switchport access vlan 20
exit
```

![show vlan brief](screenshots/02-switch-show-vlan-brief.png)

## Switch: trunk the link to the router

A trunk carries traffic for many VLANs over one link, tagging each frame so the other end knows which VLAN it belongs to.

```cisco
interface fa0/1
 switchport mode trunk      ! carry all VLANs to the router, tagged with 802.1Q
exit
show interface trunk        ! confirm Fa0/1 is trunking with 802.1q encapsulation
```

![show interface trunk](screenshots/03-switch-show-interface-trunk.png)

![Switch running config](screenshots/04-switch-running-config.png)

## Router: subinterfaces (router-on-a-stick)

The single physical link is split into two logical subinterfaces, one per VLAN. Each subinterface is the default gateway for its VLAN and uses 802.1Q tagging to match the trunk.

```cisco
interface gigabitEthernet 0/0.10      ! subinterface for VLAN 10
 encapsulation dot1Q 10               ! tag/expect VLAN 10 frames here
 ip address 192.168.10.1 255.255.255.0   ! gateway for the Students network
exit

interface gigabitEthernet 0/0.20      ! subinterface for VLAN 20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0   ! gateway for the Faculty network
exit
show ip route                         ! both networks should show as connected
show ip interface brief
```

![show ip route](screenshots/05-router-show-ip-route.png)

![show ip interface brief](screenshots/06-router-show-ip-interface-brief.png)

## PC addressing

Each PC was given a static IP, mask, and gateway matching its VLAN.

![PC0 (VLAN 10)](screenshots/07-pc0-ip-config.png)

![PC1 (VLAN 10)](screenshots/08-pc1-ip-config.png)

![PC2 (VLAN 20)](screenshots/09-pc2-ip-config.png)

## Connectivity testing

Same-VLAN pings worked immediately. Cross-VLAN pings worked once routing was in place, with the first packet occasionally timing out while ARP resolved, then succeeding.

![Ping from PC0](screenshots/10-ping-pc0.png)

![Ping from PC2](screenshots/11-ping-pc2.png)

![Inter-VLAN ping after trunk fix](screenshots/12-ping-pc2-after-fix.png)

## Key lesson

The most useful takeaway: inter-VLAN pings failed at first because the switch-to-router link was not a trunk, so VLAN-tagged frames could not cross it. Setting `switchport mode trunk` on Fa0/1 fixed it, because the router could then receive both VLANs' tagged traffic on its subinterfaces. Allow the tagged link first, and the routing follows.

## Outcome

VLAN segmentation worked, the trunk carried both VLANs, the router subinterfaces routed between them, and all intra-VLAN and inter-VLAN tests passed.
