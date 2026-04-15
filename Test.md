# Network Device Configuration Commands
## Based on Capstone Project Topology

---

## Topology Summary

| Device | Type | Uplink | Management IP |
|--------|------|--------|---------------|
| R-A | ISR 4331 | g0/0 → MITT (.191) | 10.10.50.1 |
| R-B | ISR 4331 | g0/0/0 → MITT (.193) | 10.10.50.2 |
| S-A | 2960-24T | g0/1 → R-A | 10.10.50.5 |
| S-B | 2960-24T | g0/1 → R-B | 10.10.50.3 |
| S-C | 2960-24T | Trunks to S-A & S-B | 10.10.50.6 |

### VLANs

| VLAN | Name | Subnet | Purpose |
|------|------|--------|---------|
| 10 | SERVER | 10.10.10.0/24 | Servers (DC, DHCP, DNS, DB) |
| 15 | DMZ | 10.10.15.0/24 | Web & Mail servers |
| 20 | STAFF | 10.10.20.0/24 | Front-desk / Staff |
| 30 | IT-ADMIN | 10.10.30.0/24 | IT / Admin tools |
| 40 | GUEST | 10.10.40.0/24 | Guest Wi-Fi |
| 50 | MGMT | 10.10.50.0/24 | Management |

### HSRP Virtual IPs (Gateway for each VLAN)

| VLAN | R-A IP (.1) | R-B IP (.2) | HSRP VIP (.254) |
|------|-------------|-------------|-----------------|
| 10 | 10.10.10.1 | 10.10.10.2 | 10.10.10.254 |
| 15 | 10.10.15.1 | 10.10.15.2 | 10.10.15.254 |
| 20 | 10.10.20.1 | 10.10.20.2 | 10.10.20.254 |
| 30 | 10.10.30.1 | 10.10.30.2 | 10.10.30.254 |
| 40 | 10.10.40.1 | 10.10.40.2 | 10.10.40.254 |
| 50 | 10.10.50.1 | 10.10.50.2 | 10.10.50.254 |

---

## 1. Router R-A (ISR 4331)

```
enable
configure terminal
hostname R-A
!
!--- Disable DNS lookup
no ip domain-lookup
!
!--- Enable password encryption
service password-encryption
!
!--- Set hostname and domain for SSH
ip domain-name fitlife-gym.ca
!
!--- Create SSH user (MGMT: SSH login = Admin, no password)
username Admin privilege 15 secret Admin
!
!--- Generate RSA keys for SSH
crypto key generate rsa general-keys modulus 2048
!
!--- Enable SSH version 2
ip ssh version 2
!
!--- Console line
line console 0
 login local
 logging synchronous
 exec-timeout 10 0
!
!--- VTY lines for SSH
line vty 0 4
 login local
 transport input ssh
 logging synchronous
 exec-timeout 10 0
!
!--- WAN Interface to MITT (10.128.209.0/24)
interface GigabitEthernet0/0
 description UPLINK-TO-MITT
 ip address 10.128.209.191 255.255.255.0
 no shutdown
!
!--- LAN Trunk Interface to S-A (enable for subinterfaces)
interface GigabitEthernet0/1
 description TRUNK-TO-SA
 no ip address
 no shutdown
!
!--- Subinterface VLAN 10 - SERVER
interface GigabitEthernet0/1.10
 description VLAN10-SERVER
 encapsulation dot1Q 10
 ip address 10.10.10.1 255.255.255.0
 standby 10 ip 10.10.10.254
 standby 10 priority 110
 standby 10 preempt
!
!--- Subinterface VLAN 15 - DMZ
interface GigabitEthernet0/1.15
 description VLAN15-DMZ
 encapsulation dot1Q 15
 ip address 10.10.15.1 255.255.255.0
 standby 15 ip 10.10.15.254
 standby 15 priority 110
 standby 15 preempt
!
!--- Subinterface VLAN 20 - STAFF
interface GigabitEthernet0/1.20
 description VLAN20-STAFF
 encapsulation dot1Q 20
 ip address 10.10.20.1 255.255.255.0
 standby 20 ip 10.10.20.254
 standby 20 priority 110
 standby 20 preempt
!
!--- Subinterface VLAN 30 - IT/ADMIN
interface GigabitEthernet0/1.30
 description VLAN30-ITADMIN
 encapsulation dot1Q 30
 ip address 10.10.30.1 255.255.255.0
 standby 30 ip 10.10.30.254
 standby 30 priority 110
 standby 30 preempt
!
!--- Subinterface VLAN 40 - GUEST
interface GigabitEthernet0/1.40
 description VLAN40-GUEST
 encapsulation dot1Q 40
 ip address 10.10.40.1 255.255.255.0
 standby 40 ip 10.10.40.254
 standby 40 priority 110
 standby 40 preempt
!
!--- Subinterface VLAN 50 - MGMT (Native)
interface GigabitEthernet0/1.50
 description VLAN50-MGMT
 encapsulation dot1Q 50
 ip address 10.10.50.1 255.255.255.0
 standby 50 ip 10.10.50.254
 standby 50 priority 110
 standby 50 preempt
!
!--- Default Route to MITT
ip route 0.0.0.0 0.0.0.0 10.128.209.1
!
!--- Enable IP routing
ip routing
!
end
write memory
```

---

## 2. Router R-B (ISR 4331)

```
enable
configure terminal
hostname R-B
!
no ip domain-lookup
service password-encryption
ip domain-name fitlife-gym.ca
!
username Admin privilege 15 secret Admin
crypto key generate rsa general-keys modulus 2048
ip ssh version 2
!
line console 0
 login local
 logging synchronous
 exec-timeout 10 0
!
line vty 0 4
 login local
 transport input ssh
 logging synchronous
 exec-timeout 10 0
!
!--- WAN Interface to MITT (10.128.209.0/24)
interface GigabitEthernet0/0/0
 description UPLINK-TO-MITT
 ip address 10.128.209.193 255.255.255.0
 no shutdown
!
!--- LAN Trunk Interface to S-B
interface GigabitEthernet0/0/1
 description TRUNK-TO-SB
 no ip address
 no shutdown
!
!--- Subinterface VLAN 10 - SERVER
interface GigabitEthernet0/0/1.10
 description VLAN10-SERVER
 encapsulation dot1Q 10
 ip address 10.10.10.2 255.255.255.0
 standby 10 ip 10.10.10.254
 standby 10 priority 100
!
!--- Subinterface VLAN 15 - DMZ
interface GigabitEthernet0/0/1.15
 description VLAN15-DMZ
 encapsulation dot1Q 15
 ip address 10.10.15.2 255.255.255.0
 standby 15 ip 10.10.15.254
 standby 15 priority 100
!
!--- Subinterface VLAN 20 - STAFF
interface GigabitEthernet0/0/1.20
 description VLAN20-STAFF
 encapsulation dot1Q 20
 ip address 10.10.20.2 255.255.255.0
 standby 20 ip 10.10.20.254
 standby 20 priority 100
!
!--- Subinterface VLAN 30 - IT/ADMIN
interface GigabitEthernet0/0/1.30
 description VLAN30-ITADMIN
 encapsulation dot1Q 30
 ip address 10.10.30.2 255.255.255.0
 standby 30 ip 10.10.30.254
 standby 30 priority 100
!
!--- Subinterface VLAN 40 - GUEST
interface GigabitEthernet0/0/1.40
 description VLAN40-GUEST
 encapsulation dot1Q 40
 ip address 10.10.40.2 255.255.255.0
 standby 40 ip 10.10.40.254
 standby 40 priority 100
!
!--- Subinterface VLAN 50 - MGMT
interface GigabitEthernet0/0/1.50
 description VLAN50-MGMT
 encapsulation dot1Q 50
 ip address 10.10.50.2 255.255.255.0
 standby 50 ip 10.10.50.254
 standby 50 priority 100
!
!--- Default Route to MITT
ip route 0.0.0.0 0.0.0.0 10.128.209.1
!
ip routing
!
end
write memory
```

---

## 3. Switch S-A (2960-24T)

```
enable
configure terminal
hostname S-A
!
no ip domain-lookup
service password-encryption
ip domain-name fitlife-gym.ca
!
username Admin privilege 15 secret Admin
crypto key generate rsa general-keys modulus 2048
ip ssh version 2
!
line console 0
 login local
 logging synchronous
!
line vty 0 4
 login local
 transport input ssh
 logging synchronous
!
!--- Create VLANs
vlan 10
 name SERVER
vlan 15
 name DMZ
vlan 20
 name STAFF
vlan 30
 name IT-ADMIN
vlan 40
 name GUEST
vlan 50
 name MGMT
!
!--- Management VLAN SVI
interface vlan 50
 ip address 10.10.50.5 255.255.255.0
 no shutdown
!
ip default-gateway 10.10.50.254
!
!--- g0/1: Trunk to R-A
interface GigabitEthernet0/1
 description TRUNK-TO-RA
 switchport mode trunk
 switchport trunk allowed vlan 10,15,20,30,40,50
 no shutdown
!
!--- f0/1, f0/2: EtherChannel to S-B (Port-Channel 1)
interface range FastEthernet0/1 - 2
 description ETHERCHANNEL-TO-SB
 channel-group 1 mode active
 no shutdown
!
interface Port-channel1
 description ETHERCHANNEL-TO-SB
 switchport mode trunk
 switchport trunk allowed vlan 10,15,20,30,40,50
!
!--- f0/3, f0/4: NIC Teaming to Server H/V1 (Port-Channel 2 - VLAN 10)
interface range FastEthernet0/3 - 4
 description NIC-TEAMING-SERVER-HV1
 channel-group 2 mode on
 no shutdown
!
interface Port-channel2
 description NIC-TEAMING-SERVER-HV1
 switchport mode access
 switchport access vlan 10
!
!--- f0/5: Access port to PCPT1 (VLAN 50 MGMT)
interface FastEthernet0/5
 description PCPT1-MGMT
 switchport mode access
 switchport access vlan 50
 no shutdown
!
!--- f0/7, f0/8: Trunk to S-C
interface range FastEthernet0/7 - 8
 description TRUNK-TO-SC
 switchport mode trunk
 switchport trunk allowed vlan 10,15,20,30,40,50
 no shutdown
!
!--- Shutdown unused ports
interface range FastEthernet0/6, FastEthernet0/9 - 24
 shutdown
!
interface GigabitEthernet0/2
 shutdown
!
end
write memory
```

---

## 4. Switch S-B (2960-24T)

```
enable
configure terminal
hostname S-B
!
no ip domain-lookup
service password-encryption
ip domain-name fitlife-gym.ca
!
username Admin privilege 15 secret Admin
crypto key generate rsa general-keys modulus 2048
ip ssh version 2
!
line console 0
 login local
 logging synchronous
!
line vty 0 4
 login local
 transport input ssh
 logging synchronous
!
!--- Create VLANs
vlan 10
 name SERVER
vlan 15
 name DMZ
vlan 20
 name STAFF
vlan 30
 name IT-ADMIN
vlan 40
 name GUEST
vlan 50
 name MGMT
!
!--- Management VLAN SVI
interface vlan 50
 ip address 10.10.50.3 255.255.255.0
 no shutdown
!
ip default-gateway 10.10.50.254
!
!--- g0/1: Trunk to R-B
interface GigabitEthernet0/1
 description TRUNK-TO-RB
 switchport mode trunk
 switchport trunk allowed vlan 10,15,20,30,40,50
 no shutdown
!
!--- f0/1, f0/2: EtherChannel to S-A (Port-Channel 1)
interface range FastEthernet0/1 - 2
 description ETHERCHANNEL-TO-SA
 channel-group 1 mode active
 no shutdown
!
interface Port-channel1
 description ETHERCHANNEL-TO-SA
 switchport mode trunk
 switchport trunk allowed vlan 10,15,20,30,40,50
!
!--- f0/11, f0/12: Trunk to S-C
interface range FastEthernet0/11 - 12
 description TRUNK-TO-SC
 switchport mode trunk
 switchport trunk allowed vlan 10,15,20,30,40,50
 no shutdown
!
!--- Shutdown unused ports
interface range FastEthernet0/3 - 10, FastEthernet0/13 - 24
 shutdown
!
interface GigabitEthernet0/2
 shutdown
!
end
write memory
```

---

## 5. Switch S-C (2960-24T)

```
enable
configure terminal
hostname S-C
!
no ip domain-lookup
service password-encryption
ip domain-name fitlife-gym.ca
!
username Admin privilege 15 secret Admin
crypto key generate rsa general-keys modulus 2048
ip ssh version 2
!
line console 0
 login local
 logging synchronous
!
line vty 0 4
 login local
 transport input ssh
 logging synchronous
!
!--- Create VLANs
vlan 10
 name SERVER
vlan 15
 name DMZ
vlan 20
 name STAFF
vlan 30
 name IT-ADMIN
vlan 40
 name GUEST
vlan 50
 name MGMT
!
!--- Management VLAN SVI
interface vlan 50
 ip address 10.10.50.6 255.255.255.0
 no shutdown
!
ip default-gateway 10.10.50.254
!
!--- f0/7, f0/8: Trunk from S-A
interface range FastEthernet0/7 - 8
 description TRUNK-FROM-SA
 switchport mode trunk
 switchport trunk allowed vlan 10,15,20,30,40,50
 no shutdown
!
!--- f0/11, f0/12: Trunk from S-B
interface range FastEthernet0/11 - 12
 description TRUNK-FROM-SB
 switchport mode trunk
 switchport trunk allowed vlan 10,15,20,30,40,50
 no shutdown
!
!--- f0/1: Access port to Server H/V2 (VLAN 15 DMZ)
interface FastEthernet0/1
 description SERVER-HV2-DMZ
 switchport mode access
 switchport access vlan 15
 no shutdown
!
!--- f0/5: Access port to Front-desk PC (VLAN 20 STAFF)
interface FastEthernet0/5
 description FRONT-DESK-STAFF
 switchport mode access
 switchport access vlan 20
 no shutdown
!
!--- f0/2: Access port to Management Tools PC (VLAN 30 IT/ADMIN)
interface FastEthernet0/2
 description MGMT-TOOLS-ITADMIN
 switchport mode access
 switchport access vlan 30
 no shutdown
!
!--- g0/1: Trunk to Access Point (VLAN 40 GUEST)
interface GigabitEthernet0/1
 description TRUNK-TO-AP
 switchport mode trunk
 switchport trunk allowed vlan 40
 no shutdown
!
!--- Shutdown unused ports
interface range FastEthernet0/3 - 4, FastEthernet0/6, FastEthernet0/9 - 10, FastEthernet0/13 - 24
 shutdown
!
interface GigabitEthernet0/2
 shutdown
!
end
write memory
```

---

## End-Device IP Addressing Summary

| Device | VLAN | IP Address | Subnet Mask | Default Gateway |
|--------|------|------------|-------------|-----------------|
| Server H/V1 (DC1/DHCP/DNS) | 10 | 10.10.10.50 | 255.255.255.0 | 10.10.10.254 |
| Server H/V1 (DC2/DB1) | 10 | 10.10.10.150 | 255.255.255.0 | 10.10.10.254 |
| Server H/V2 (WEB1) | 15 | 10.10.15.10 | 255.255.255.0 | 10.10.15.254 |
| Server H/V2 (MAIL1) | 15 | 10.10.15.110 | 255.255.255.0 | 10.10.15.254 |
| PCPT1 (MGMT PC) | 50 | 10.10.50.250 | 255.255.255.0 | 10.10.50.254 |
| Front-desk PC | 20 | DHCP | 255.255.255.0 | 10.10.20.254 |
| Management Tools PC | 30 | DHCP or Static | 255.255.255.0 | 10.10.30.254 |
| Guest Laptop (via AP) | 40 | DHCP | 255.255.255.0 | 10.10.40.254 |

---

## Key Design Notes

- **HSRP**: R-A is the Active router (priority 110 with preempt) for all VLANs; R-B is Standby (priority 100, default). Virtual IP is .254 on each subnet.
- **EtherChannel**: LACP (mode active) between S-A f0/1-2 and S-B f0/1-2 provides link redundancy.
- **NIC Teaming**: Server H/V1 uses two NICs bonded to S-A f0/3-4 via Port-Channel 2.
- **Trunk links**: S-A↔S-C (f0/7-8) and S-B↔S-C (f0/11-12) carry all VLANs.
- **Wireless**: AP connects to S-C g0/1 on a trunk allowing VLAN 40 (Guest).
- **Management SSH**: All devices use local user "Admin" with SSH v2 access.
- **Default route**: Both routers point 0.0.0.0/0 to 10.128.209.1 (MITT gateway).
