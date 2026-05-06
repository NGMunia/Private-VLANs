# Private-VLANs


This topology uses Private VLANs (PVLANs) to solve the residential isolation problem, alongside conventional VLANs for corporate staff and a centralized DHCP server. Everything is wired up through a two-tier switching hierarchy with a routed gateway to the internet.

---

## Table of Contents

- [The Problem This Solves](#the-problem-this-solves)
- [Topology Overview](#topology-overview)
- [Network Design Breakdown](#network-design-breakdown)
  - [VLAN Scheme](#vlan-scheme)
  - [PVLAN — How It Works Here](#pvlan--how-it-works-here)
  - [Corporate Segment (VLAN 20)](#corporate-segment-vlan-20)
  - [Core Switch — The Brain](#core-switch--the-brain)
  - [Gateway / Router](#gateway--router)
- [Configuration Snippets](#configuration-snippets)
  - [BLK-A-SW — Block Access Switch (PVLAN)](#blk-a-sw--block-access-switch-pvlan)
  - [CORE-SW — Layer 3 Switch](#core-sw--layer-3-switch)
  - [Gateway Router](#gateway-router)


---

## The Problem This Solves

Imagine a residential building where each flat gets internet through a shared switch. 
You *could* give every flat its own VLAN, but that is not scalable. A 50-flat building means 50 VLANs, 50 SVI interfaces on the core switch, 50 DHCP pools. Management becomes a nightmare fast.

**Private VLANs solve this.** One primary VLAN handles the addressing and routing. Isolated secondary VLANs under it prevent any lateral communication between tenants — at the access switch level. The hosts never know their neighbors exist. They can only talk upward to the promiscuous port (the core switch SVI), which routes their traffic out.

The corporate segment is a straightforward separate VLAN. DHCP is centralized.

Cisco router is configured as a DHCP/DNS serve, sitting in its own network, serving scopes for both the residential block and the corporate floor via DHCP relay.

---

## Topology Overview

Below is the topology:
![Topology](/Topology.png)

---

## Network Design Breakdown

### VLAN Scheme

| VLAN | Type              | Name        | Subnet             | Purpose                          |
|------|-------------------|-------------|--------------------|----------------------------------|
| 20   | Primary PVLAN     | RESIDENTIAL | 192.168.20.0/26    | Residential block addressing     |
| 200  | Isolated PVLAN    | FLAT-ISO    | 192.168.20.0/26    | Per-flat tenant isolation        |
| 201  | Community VLAN    | LOBBY       | 192.168.20.0/26    | Lobby Devices PCs                |
| 10   | Standard VLAN     | CORPORATE   | 192.168.20.0/24    | Corporate LAN                    |
|  -   | Services BLK      | DHCP/DNS    | 10.0.0.6/32        | DHCP/DNS server, DNS fowarding   |


> **Note on PVLAN addressing:** Isolated/community ports in VLAN 200 still use the 192.168.20.0/26 address space. The PVLAN relationship maps VLAN 200/201 traffic up to primary VLAN 20, so the SVI on CORE-SW only needs one IP for the entire residential block — `192.168.20.0/26`.

---

### PVLAN — How It Works Here

PVLANs introduce a two-level VLAN hierarchy:

- **Primary VLAN (20):** The parent. Defines the IP subnet and carries traffic to/from the promiscuous port (the uplink to CORE-SW).
- **Isolated VLAN (200):** A secondary VLAN under VLAN 20. Ports in this VLAN can communicate with promiscuous ports only — never with each other, even within the same secondary VLAN.
- **Community VLAN (201):** Ports in this VLAN can communicate with each other and promiscuous ports only — never with Isolated ports.

In this topology:
- **HSE-1 and HSE-2 connect to BLK-A-SW on isolated ports** in VLAN 200. This means the two home routers are completely blind to each other at Layer 2.
- **PC3 and PC4 connect to BLK-A-SW on community ports** in VLAN 201. This means the two PCs can communicate with each other at Layer 2.
- **BLK-A-SW's uplink to CORE-SW is a promiscuous trunk.** The core switch sees both VLAN 20 and the PVLAN mapping, which allows it to route residential traffic while still enforcing the isolation policy.
- **CORE-SW has a promiscuous SVI on VLAN 10** (`interface Vlan20`). This is the gateway destination isolated ports and Community ports can reach directly out the internet.



---

### Corporate Segment (VLAN 10)

PC1 connects to Corp-SW on standard access ports in VLAN 10. This segment (`192.168.10.0/24`) is fully standard — no PVLAN complexity needed here since corporate staff are expected to collaborate and share resources normally.

Inter-VLAN communication from VLAN 10, VLAN 20 to host 10.0.0.6 (the server) is permitted and routed by the core switch SVI.


---

### Core Switch — The Brain

CORE-SW is a Layer 3 switch and does all the heavy lifting:

- **Terminates SVIs** for VLAN 10, and 20 — acting as the default gateway for each segment
- **DHCP relay** — `ip helper-address` on each SVI forwards DHCP broadcasts to the server 10.0.0.6
- **Dynamic routing** — OSPF is used as the dynamic routing protocol.

---

### Gateway / Router

The gateway handles NAT (overload/PAT) for all RFC 1918 (private address) traffic going to the internet. 
- One interface toward CORE-SW (LAN-facing NAT inside)
- One interface toward the internet (WAN-facing NAT Outside)
- A `ip nat inside` / `ip nat outside` pair with an overload statement

---

## Configuration Snippets

### BLK-A-SW — Block Access Switch (PVLAN)

```ios
!
vlan 20
 name RESIDENTIAL-VLAN
  private-vlan primary
  private-vlan association 200-201
!
vlan 200
  private-vlan isolated
!
vlan 201
  private-vlan community
!
!
policy-map Network-Access-Policy
 class class-default
  police cir 1000000 conform-action transmit  exceed-action drop
!
!
interface Ethernet0/0
 switchport private-vlan host-association 20 200
 switchport mode private-vlan host
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 service-policy input Network-Access-Policy
 service-policy output Network-Access-Policy
!
interface Ethernet0/1
 switchport private-vlan host-association 20 200
 switchport mode private-vlan host
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 service-policy input Network-Access-Policy
 service-policy output Network-Access-Policy
!
interface Ethernet3/0
 switchport private-vlan host-association 20 201
 switchport mode private-vlan host
 service-policy input Network-Access-Policy
 service-policy output Network-Access-Policy
!
interface Ethernet3/1
 switchport private-vlan host-association 20 201
 switchport mode private-vlan host
 service-policy input Network-Access-Policy
 service-policy output Network-Access-Policy
!
interface Ethernet3/2
 shutdown
!
interface Ethernet3/3
 switchport private-vlan mapping 20 200-201
 switchport mode private-vlan promiscuous

```

> **Important:** PVLAN configuration requires VTP to be in **Transparent mode** on all switches participating in the private VLAN — VTP will not propagate PVLAN config correctly in Server/Client mode.

```ios
! Set VTP to transparent on all PVLAN switches
vtp mode transparent
```

---

### CORE-SW — Layer 3 Switch

```ios
!
vlan 20
 name RESIDENTIAL-VLAN
  private-vlan primary
  private-vlan association 200-201
!
vlan 200
  private-vlan isolated
!
vlan 201
  private-vlan community
!
!
interface Port-channel1
 switchport trunk allowed vlan 10
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface Ethernet0/0
 switchport trunk allowed vlan 10
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
!
interface Ethernet0/1
 switchport trunk allowed vlan 10
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active

!
interface Ethernet2/0
 description LINK TO RESIDENTIAL BLOCK ONLY!
 switchport private-vlan mapping 20 200,201
 switchport mode private-vlan promiscuous

!
interface Ethernet3/2
 no switchport
 ip address 10.0.0.5 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0
!
interface Ethernet3/3
 no switchport
 ip address 10.0.0.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0

!
interface Vlan10
 description CORPORATE VLAN SVI
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 10.0.0.6
 ip ospf 1 area 0
!
interface Vlan20
 description RESIDENTIAL VLAN SVI
 ip address 192.168.20.1 255.255.255.192
 ip helper-address 10.0.0.6
 ip ospf 1 area 20
!
router ospf 1
 router-id 1.1.1.1
 auto-cost reference-bandwidth 100000
 passive-interface Vlan10
 passive-interface Vlan20
 passive-interface Ethernet3/2
!

```
---

### Gateway Router

```ios
! LAN-facing interface toward CORE-SW
!
interface Ethernet0/0
 description LINK TO CORE SWITCH
 ip address 10.0.0.2 255.255.255.252
 ip nat inside
 ip virtual-reassembly in
 ip ospf network point-to-point
 ip ospf 1 area 0
 duplex auto
!
interface Ethernet0/3
 ip address dhcp
 ip nat outside
 ip virtual-reassembly in
 duplex auto
!
router ospf 1
 router-id 3.3.3.3
 auto-cost reference-bandwidth 100000
 passive-interface Ethernet0/3
 default-information originate
!
ip nat inside source list nat-acl interface Ethernet0/3 overload
!
ip access-list standard nat-acl
 permit ip 192.168.20.0 0.0.0.127 any
 permit ip 192.168.10.0 0.0.0.255 any
 permit ip host 10.0.0.6 any

```
## Configuration Snippet of the DNS/DHCP server

```ios
!
ip dhcp excluded-address 192.168.20.1 192.168.20.10
ip dhcp excluded-address 192.168.10.1 192.168.10.10
!
ip dhcp pool RESIDENTIAL-DHCP-POOL
 network 192.168.20.0 255.255.255.128
 default-router 192.168.20.1
 dns-server 10.0.0.6
 lease 0 3

ip dhcp pool CORPORATE-DHCP-POOL
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 10.0.0.6
 lease 0 3
!
!
!
ip name-server 8.8.8.8
ip name-server 8.8.4.4
!
interface Ethernet0/0
 ip address 10.0.0.6 255.255.255.252
 no ip route-cache
 duplex auto
!
ip default-gateway 10.0.0.5
!
ip dns server
---