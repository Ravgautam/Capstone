# FitLife Gyms Layer 2 Packet Tracer Implementation

## Overview

This report translates the provided high‑level FitLife Gyms Layer 2 design into an implementable Packet Tracer configuration that matches the attached topology image (dual routers R-A/R-B, three switches S-A/S-B/S-C, server block, staff/IT PCs, guest Wi‑Fi AP, and Internet cloud). It follows standard Cisco "router‑on‑a‑stick" practices for inter‑VLAN routing and L2‑only switches.

The focus is on concrete device roles, VLAN/IP allocations, trunk and EtherChannel layout, and example IOS configuration snippets that can be adapted directly in the lab.

## Device Roles and Topology Mapping

### Routers

- **R-A (left ISR4331)**
  - Primary HSRP router for all VLANs.
  - One LAN interface (for example, `G0/0`) toward S-A/S-B via trunk.
  - One WAN interface (for example, `G0/1`) toward the Capstone/Cloud network 10.128.250.0/24.
- **R-B (right ISR4331)**
  - Secondary HSRP router for all VLANs.
  - Same interface roles and addressing model as R-A, with different last octet and lower HSRP priority.

Both routers perform inter‑VLAN routing using IEEE 802.1Q subinterfaces on the LAN trunk and implement NAT and ACLs at the edge, as is standard in router‑on‑a‑stick inter‑VLAN designs.

### Switches

- **S-A (top‑left 2960)**
  - L2 distribution switch.
  - Trunk uplinks to R-A and R-B.
  - EtherChannel trunk to S-B.
  - Access/trunk port toward the server block (Server‑PT) representing VLAN 10/15/50.
- **S-B (top‑right 2960)**
  - L2 distribution switch.
  - Trunk uplinks to R-A and R-B (mirroring S-A).
  - EtherChannel trunk to S-A.
- **S-C (bottom 2960)**
  - L2 access switch.
  - Dual trunks (or EtherChannel) upward toward S-A and S-B.
  - Access ports for Staff VLAN 20, IT/Admin VLAN 30.
  - Trunk port to AP (carrying VLAN 20 and VLAN 40).

### Endpoints

- **Server‑PT (Server2 in the green block)**
  - Represents multiple server roles (DC1/DC2, DHCP, WEB1, MAIL1, DB1, MGMT1).
  - Connected to S-A access port in VLAN 10 (SERVER) for simplicity; additional logical addressing can be simulated on the same NIC.
- **Staff PCs (Front‑desk, Trainers, Finance)**
  - Connected to S-C access ports in VLAN 20 (STAFF).
- **IT/ADMIN PCs (IT Staff, Management Tools)**
  - Connected to S-C access ports in VLAN 30 (IT_ADMIN).
- **Access Point AP + Guest Laptop**
  - AP trunk to S-C, tagged VLAN 20 (FitLife‑Staff SSID) and VLAN 40 (FitLife‑Guest SSID).
  - Guest laptop associated to Guest SSID, receives VLAN 40 addressing and Internet‑only access.

## VLAN and IP Plan (Aligned to Diagram)

The VLANs and subnets follow the table from the design document.

| VLAN | Name       | Subnet          | HSRP VIP (gateway) |
|------|------------|-----------------|--------------------|
| 10   | SERVER     | 10.10.10.0/24   | 10.10.10.254       |
| 15   | DMZ        | 10.10.15.0/24   | 10.10.15.254       |
| 20   | STAFF      | 10.10.20.0/24   | 10.10.20.254       |
| 30   | IT_ADMIN   | 10.10.30.0/24   | 10.10.30.254       |
| 40   | GUEST_WIFI | 10.10.40.0/24   | 10.10.40.254       |
| 50   | MGMT       | 10.10.50.0/24   | 10.10.50.254       |
| 60   | VOIP       | 10.10.60.0/24   | 10.10.60.254       |

Suggested router addresses (example):

- R-A: `.1` in each VLAN (10.10.x.1).
- R-B: `.2` in each VLAN (10.10.x.2).
- HSRP: `.254` as per table.

## Router Configuration Blueprint

### 1. Base Interface and Subinterfaces (R-A)

```plaintext
interface GigabitEthernet0/0
 description Trunk to S-A/S-B
 no shutdown

! VLAN 10 SERVER
interface GigabitEthernet0/0.10
 encapsulation dot1q 10
 ip address 10.10.10.1 255.255.255.0
 standby 10 ip 10.10.10.254
 standby 10 priority 110
 standby 10 preempt

! VLAN 15 DMZ
interface GigabitEthernet0/0.15
 encapsulation dot1q 15
 ip address 10.10.15.1 255.255.255.0
 standby 15 ip 10.10.15.254
 standby 15 priority 110
 standby 15 preempt

! VLAN 20 STAFF
interface GigabitEthernet0/0.20
 encapsulation dot1q 20
 ip address 10.10.20.1 255.255.255.0
 standby 20 ip 10.10.20.254
 standby 20 priority 110
 standby 20 preempt

! VLAN 30 IT_ADMIN
interface GigabitEthernet0/0.30
 encapsulation dot1q 30
 ip address 10.10.30.1 255.255.255.0
 standby 30 ip 10.10.30.254
 standby 30 priority 110
 standby 30 preempt

! VLAN 40 GUEST_WIFI
interface GigabitEthernet0/0.40
 encapsulation dot1q 40
 ip address 10.10.40.1 255.255.255.0
 standby 40 ip 10.10.40.254
 standby 40 priority 110
 standby 40 preempt

! VLAN 50 MGMT
interface GigabitEthernet0/0.50
 encapsulation dot1q 50
 ip address 10.10.50.1 255.255.255.0
 standby 50 ip 10.10.50.254
 standby 50 priority 110
 standby 50 preempt

! VLAN 60 VOIP (optional)
interface GigabitEthernet0/0.60
 encapsulation dot1q 60
 ip address 10.10.60.1 255.255.255.0
 standby 60 ip 10.10.60.254
 standby 60 priority 110
 standby 60 preempt
```

### 2. Matching Subinterfaces (R-B)

```plaintext
interface GigabitEthernet0/0
 description Trunk to S-A/S-B
 no shutdown

interface GigabitEthernet0/0.10
 encapsulation dot1q 10
 ip address 10.10.10.2 255.255.255.0
 standby 10 ip 10.10.10.254
 standby 10 priority 90
 standby 10 preempt

interface GigabitEthernet0/0.15
 encapsulation dot1q 15
 ip address 10.10.15.2 255.255.255.0
 standby 15 ip 10.10.15.254
 standby 15 priority 90
 standby 15 preempt

! Repeat similarly for VLANs 20, 30, 40, 50, 60
```

### 3. WAN and Default Routes

Assuming each router connects to the Capstone network via `G0/1` with addresses `10.128.250.10` (R-A) and `10.128.250.11` (R-B):

```plaintext
interface GigabitEthernet0/1
 description Upstream to Capstone/Cloud
 ip address 10.128.250.10 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.128.250.1
```

### 4. NAT and ACL Skeleton (Edge)

```plaintext
access-list 100 permit ip 10.10.0.0 0.0.255.255 any

ip nat inside source list 100 interface GigabitEthernet0/1 overload

interface GigabitEthernet0/0
 ip nat inside
interface GigabitEthernet0/0.10
 ip nat inside
! ... repeat inside on all LAN subinterfaces ...
interface GigabitEthernet0/1
 ip nat outside
```

## Switch Configuration Blueprint

### 1. VLAN Creation (All Switches)

```plaintext
vlan 10
 name SERVER
vlan 15
 name DMZ
vlan 20
 name STAFF
vlan 30
 name IT_ADMIN
vlan 40
 name GUEST_WIFI
vlan 50
 name MGMT
vlan 60
 name VOIP
```

### 2. Trunks to Routers (S-A and S-B)

```plaintext
interface FastEthernet0/1
 description Trunk to R-A
 switchport mode trunk
 switchport trunk allowed vlan 10,15,20,30,40,50,60
 spanning-tree portfast trunk

interface FastEthernet0/2
 description Trunk to R-B
 switchport mode trunk
 switchport trunk allowed vlan 10,15,20,30,40,50,60
 spanning-tree portfast trunk
```

### 3. EtherChannel Between S-A and S-B

```plaintext
! On S-A
interface range FastEthernet0/3 - 4
 channel-group 1 mode active

interface Port-channel1
 description EtherChannel to S-B
 switchport mode trunk
 switchport trunk allowed vlan 10,15,20,30,40,50,60

! On S-B
interface range FastEthernet0/3 - 4
 channel-group 1 mode active

interface Port-channel1
 description EtherChannel to S-A
 switchport mode trunk
 switchport trunk allowed vlan 10,15,20,30,40,50,60
```

### 4. Uplinks from S-C to S-A/S-B

```plaintext
interface FastEthernet0/1
 description Trunk to S-A
 switchport mode trunk
 switchport trunk allowed vlan 20,30,40,50

interface FastEthernet0/2
 description Trunk to S-B
 switchport mode trunk
 switchport trunk allowed vlan 20,30,40,50
```

### 5. Access Ports for PCs and Servers

```plaintext
! On S-C (Staff and IT PCs)
interface FastEthernet0/10
 description Front-desk PC
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable

interface FastEthernet0/11
 description Trainers PC
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable

interface FastEthernet0/12
 description Finance PC
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable

interface FastEthernet0/13
 description IT Staff PC
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 spanning-tree bpduguard enable

interface FastEthernet0/14
 description Management Tools PC
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 spanning-tree bpduguard enable

! On S-A (Server block)
interface FastEthernet0/10
 description Server2 (DC/DHCP/WEB/MAIL/DB/MGMT)
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### 6. Trunk to AP

```plaintext
interface FastEthernet0/5
 description Trunk to AP (FitLife-Staff & FitLife-Guest)
 switchport mode trunk
 switchport trunk allowed vlan 20,40
 spanning-tree portfast trunk
```

## DHCP Helpers

```plaintext
interface GigabitEthernet0/0.20
 ip helper-address 10.10.10.10
 ip helper-address 10.10.10.11

interface GigabitEthernet0/0.30
 ip helper-address 10.10.10.10
 ip helper-address 10.10.10.11

interface GigabitEthernet0/0.40
 ip helper-address 10.10.10.10
 ip helper-address 10.10.10.11
```

## Guest VLAN ACL Example

```plaintext
ip access-list extended GUEST-IN
 deny   ip 10.10.40.0 0.0.0.255 10.10.0.0 0.0.255.255
 permit ip 10.10.40.0 0.0.0.255 any

interface GigabitEthernet0/0.40
 ip access-group GUEST-IN in
```

## Basic L2 Security (example)

```plaintext
ip dhcp snooping
ip dhcp snooping vlan 20,30,40

interface FastEthernet0/1
 ip dhcp snooping trust

interface FastEthernet0/10
 spanning-tree portfast
 spanning-tree bpduguard enable
```

## Verification Checklist

- `show vlan brief` on each switch.
- `show interfaces trunk` on S-A/S-B/S-C.
- `show etherchannel summary` on S-A/S-B.
- `show ip interface brief` and `show standby brief` on R-A/R-B.
- Test pings between VLANs and out to the Internet cloud.
