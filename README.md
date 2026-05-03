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
  - [Windows Server 2012 — DHCP Scopes](#windows-server-2012--dhcp-scopes)

---

## The Problem This Solves

Imagine a residential building where each flat gets internet through a shared switch. 
You *could* give every flat its own VLAN, but that is not scalable. A 50-flat building means 50 VLANs, 50 SVI interfaces on the core switch, 50 DHCP pools. Management becomes a nightmare fast.

**Private VLANs solve this.** One primary VLAN handles the addressing and routing. Isolated secondary VLANs under it prevent any lateral communication between tenants — at the access switch level. The hosts never know their neighbors exist. They can only talk upward to the promiscuous port (the core switch SVI), which routes their traffic out.

The corporate segment is a straightforward separate VLAN. DHCP is centralized on a Windows Server 2012 machine sitting in its own network, serving scopes for both the residential block and the corporate floor via DHCP relay.

---

## Topology Overview

Below is the topology:
![Topology](/Topology.png)

---

## Network Design Breakdown

### VLAN Scheme

| VLAN | Type              | Name        | Subnet             | Purpose                          |
|------|-------------------|-------------|--------------------|----------------------------------|
| 10   | Primary PVLAN     | RESIDENTIAL | 192.168.20.0/26    | Residential block addressing     |
| 200  | Isolated PVLAN    | FLAT-ISO    | (shares VLAN 20)   | Per-flat tenant isolation        |
| 10   | Standard VLAN     | STAFF       | 192.168.10.0/24    | Corporate staff PCs              |
| 30   | Standard VLAN     | SERVERS     | 192.168.30.0/24    | DHCP/services server             |

> **Note on PVLAN addressing:** Isolated ports in VLAN 200 still use the 192.168.20.0/26 address space. The PVLAN relationship maps VLAN 200 traffic up to primary VLAN 20, so the SVI on CORE-SW only needs one IP for the entire residential block — `192.168.20.0/26`.

---

### PVLAN — How It Works Here

PVLANs introduce a two-level VLAN hierarchy:

- **Primary VLAN (20):** The parent. Defines the IP subnet and carries traffic to/from the promiscuous port (the uplink to CORE-SW).
- **Isolated VLAN (200):** A secondary VLAN under VLAN 20. Ports in this VLAN can communicate with promiscuous ports only — never with each other, even within the same secondary VLAN.

In this topology:
- **HSE-1 and HSE-2 connect to BLK-A-SW on isolated ports** in VLAN 200. This means the two home routers are completely blind to each other at Layer 2.
- **BLK-A-SW's uplink to CORE-SW is a promiscuous trunk.** The core switch sees both VLAN 20 and the PVLAN mapping, which allows it to route residential traffic while still enforcing the isolation policy.
- **CORE-SW has a promiscuous SVI on VLAN 10** (`interface Vlan20`). This is the only destination isolated ports can reach directly — which is exactly what we want.

The result: Host1 and Host2 share a subnet and share a gateway, but cannot ARP for each other, cannot ping each other, and cannot establish any Layer 2 session directly.

```
Host1 (192.168.10.10) --[isolated VLAN 200]--> BLK-A-SW
                                                    |
                                         [promiscuous trunk]
                                                    |
                                    CORE-SW SVI 192.168.20.1  ← only reachable L2 destination
```

---

### Corporate Segment (VLAN 20)

PC1 connects to Corp-SW on standard access ports in VLAN 10. This segment (`192.168.10.0/24`) is fully standard — no PVLAN complexity needed here since corporate staff are expected to collaborate and share resources normally.

Inter-VLAN communication from VLAN 10 to VLAN 30 (the server) is permitted and routed by the core switch SVI.


---

### Core Switch — The Brain

CORE-SW is a Layer 3 switch and does all the heavy lifting:

- **Terminates SVIs** for VLAN 10, and 20 — acting as the default gateway for each segment
- **PVLAN mapping** — the VLAN 20 SVI is configured as a promiscuous port mapping, allowing it to serve both primary and isolated VLANs
- **DHCP relay** — `ip helper-address` on each SVI forwards DHCP broadcasts to the server in VLAN 30
- **Default route** — a static route or routing protocol pointing to the Gateway for internet-bound traffic
- **Uplink to Gateway** — either a routed L3 link or a trunk, depending on whether NAT lives on the router or somewhere upstream

---

### Gateway / Router

The gateway handles NAT (overload/PAT) for all RFC 1918 traffic going to the internet. In a real deployment this would be where your ISP handoff lives. In this lab it's a simple router with:
- One interface toward CORE-SW (LAN-facing)
- One interface toward the internet (WAN-facing)
- A `ip nat inside` / `ip nat outside` pair with an overload statement

---

## Configuration Snippets

### BLK-A-SW — Block Access Switch (PVLAN)

```ios
! Define the VLANs
vlan 10
 name RESIDENTIAL-PRIMARY
 private-vlan primary
!
vlan 20
 name RESIDENTIAL VLAN
  private-vlan primary
  private-vlan association 200
!
vlan 200
  private-vlan isolated
!
interface Ethernet0/0
 switchport private-vlan host-association 20 200
 switchport mode private-vlan host
 spanning-tree portfast edge
 spanning-tree bpduguard enable
!
interface Ethernet0/1
 switchport private-vlan host-association 20 200
 switchport mode private-vlan host
 spanning-tree portfast edge
 spanning-tree bpduguard enable
!
interface Ethernet0/2
 switchport private-vlan host-association 20 200
 switchport mode private-vlan host
 spanning-tree portfast edge
 spanning-tree bpduguard enable
!
interface Ethernet0/3
 switchport private-vlan host-association 20 200
 switchport mode private-vlan host
 spanning-tree portfast edge
 spanning-tree bpduguard enable
!
interface Ethernet3/3
 switchport private-vlan mapping 20 200
 switchport mode private-vlan promiscuous

 switchport private-vlan mapping 10 110
```

> **Important:** PVLAN configuration requires VTP to be in **Transparent mode** on all switches participating in the private VLAN — VTP will not propagate PVLAN config correctly in Server/Client mode.

```ios
! Set VTP to transparent on all PVLAN switches
vtp mode transparent
```

---

### CORE-SW — Layer 3 Switch

```ios
! vlan 10
 name CORPORATE VLAN
!
vlan 20
 name RESIDENTIAL VLAN
  private-vlan primary
  private-vlan association 200
!
vlan 200
  private-vlan isolated
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
 switchport private-vlan mapping 20 200
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
 ip helper-address 192.168.30.254
 ip ospf 1 area 0
!
interface Vlan20
 description RESIDENTIAL VLAN SVI
 ip address 192.168.20.1 255.255.255.192
 ip helper-address 192.168.30.254
 ip ospf 1 area 20
!
router ospf 1
 router-id 1.1.1.1
 auto-cost reference-bandwidth 100000
 passive-interface Vlan10
 passive-interface Vlan20
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
 permit 192.168.30.254
 permit 192.168.20.0 0.0.0.63
 permit 192.168.10.0 0.0.0.255

```

---