# FitLife Gym — Network Infrastructure Technical Documentation

**NSA630 Capstone Project**
**Manitoba Institute of Trades and Technology (MITT)**
**Date: April 2026**

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Project Background and Scenario](#2-project-background-and-scenario)
3. [Network Design Overview](#3-network-design-overview)
4. [IP Addressing Scheme](#4-ip-addressing-scheme)
5. [VLAN Design and Layer 2 Segmentation](#5-vlan-design-and-layer-2-segmentation)
6. [Network Device Inventory](#6-network-device-inventory)
7. [Router Configurations](#7-router-configurations)
   - 7.1 [Router R-A (Primary)](#71-router-r-a-primary)
   - 7.2 [Router R-B (Secondary)](#72-router-r-b-secondary)
8. [Switch Configurations](#8-switch-configurations)
   - 8.1 [Switch S-A (Core/Distribution)](#81-switch-s-a-coredistribution)
   - 8.2 [Switch S-B (Distribution)](#82-switch-s-b-distribution)
   - 8.3 [Switch S-C (Access Layer)](#83-switch-s-c-access-layer)
9. [Redundancy and High Availability](#9-redundancy-and-high-availability)
   - 9.1 [HSRP — First Hop Redundancy](#91-hsrp--first-hop-redundancy)
   - 9.2 [EtherChannel — Link Aggregation](#92-etherchannel--link-aggregation)
   - 9.3 [Backup Routing](#93-backup-routing)
10. [Network Address Translation (NAT)](#10-network-address-translation-nat)
11. [DHCP and DNS Services](#11-dhcp-and-dns-services)
12. [Server and Service Infrastructure](#12-server-and-service-infrastructure)
    - 12.1 [Server VLAN (VLAN 10)](#121-server-vlan-vlan-10)
    - 12.2 [DMZ VLAN (VLAN 15)](#122-dmz-vlan-vlan-15)
13. [Access Control Lists (ACLs)](#13-access-control-lists-acls)
    - 13.1 [Staff VLAN 20 ACL](#131-staff-vlan-20-acl)
    - 13.2 [IT Admin VLAN 30 ACL](#132-it-admin-vlan-30-acl)
    - 13.3 [Guest Wi-Fi VLAN 40 ACL](#133-guest-wi-fi-vlan-40-acl)
    - 13.4 [DMZ VLAN 15 ACLs](#134-dmz-vlan-15-acls)
    - 13.5 [VTY Access Control](#135-vty-access-control)
14. [Wireless LAN Design](#14-wireless-lan-design)
15. [Security Hardening](#15-security-hardening)
    - 15.1 [Switch Hardening](#151-switch-hardening)
    - 15.2 [Router Hardening](#152-router-hardening)
    - 15.3 [Network Device Access Control](#153-network-device-access-control)
    - 15.4 [Logging and Monitoring](#154-logging-and-monitoring)
16. [Active Directory and Domain Services](#16-active-directory-and-domain-services)
17. [Email Services](#17-email-services)
18. [Web Application Services](#18-web-application-services)
19. [Physical Topology Summary](#19-physical-topology-summary)
20. [Running Configurations Reference](#20-running-configurations-reference)

---

## 1. Executive Summary

This document provides the complete technical documentation for the FitLife Gym IT infrastructure, designed and implemented as part of the NSA630 Capstone Project at MITT. The project delivers a fully functional, segmented, redundant, and secured network environment for a fitness business operating under the domain **fitlife-gym.local** (with the public-facing domain **fitlife-gym.ca**).

The infrastructure encompasses two Cisco ISR 4331 routers operating in an HSRP active/standby pair, three Cisco Catalyst switches interconnected via LACP EtherChannels in a full-mesh topology, six VLANs for traffic segmentation, a DMZ hosting public-facing web and email services, an internal server VLAN housing Active Directory domain controllers with integrated DNS and DHCP, and comprehensive ACL-based traffic filtering. NAT provides internet connectivity for all internal hosts, while static NAT entries expose the web server and mail server to the Capstone WAN for external access.

---

## 2. Project Background and Scenario

**Scenario Company:** FitLife Gym

FitLife Gym is a growing fitness business that requires a professional IT infrastructure to support its daily operations. The gym needs internet access for staff and guest members, a centralized directory service for employee account management, internal and external email communications, a public-facing website for class schedules and membership information, and strong network security to protect member data and business operations.

The infrastructure must support the following user groups with distinct network access requirements:

- **Staff** — Front desk, trainers, and finance personnel who need access to internal applications and the internet but should not have administrative access to network infrastructure.
- **IT Administrators** — The technical team responsible for managing servers, network equipment, and all IT services, requiring full administrative access.
- **Guest Wi-Fi Users** — Gym members and visitors who need basic internet access and access to the public gym website, but must be completely isolated from all internal resources.
- **Servers** — Internal infrastructure services including Active Directory domain controllers, DNS, and DHCP.
- **DMZ** — Public-facing services including the web server and email server, accessible from the internet and internal networks with controlled access.
- **Management** — A dedicated VLAN for out-of-band network device management.

---

## 3. Network Design Overview

The FitLife Gym network follows a collapsed core/distribution architecture with a separate access layer. Two routers (R-A and R-B) serve as the default gateways for all VLANs using HSRP, providing first-hop redundancy. Three switches (S-A, S-B, S-C) form the switching fabric, interconnected with LACP EtherChannels in a full-mesh triangle for link redundancy and increased bandwidth.

**Design Principles:**

- **Defense in depth** — Multiple layers of security including ACLs at the router level, DHCP snooping and Dynamic ARP Inspection at the switch level, and port-level protections (PortFast, BPDU Guard).
- **Segmentation** — Six VLANs ensure logical isolation of traffic by function and trust level.
- **Redundancy** — HSRP for gateway failover, EtherChannel for link redundancy, dual routers for WAN path diversity, and dual domain controllers for service availability.
- **Least privilege** — ACLs restrict each VLAN to only the access it requires; guest users are tightly confined to DNS and web browsing.

**WAN Connectivity:**

Both routers connect to the MITT Capstone network at 10.128.250.0/24, which provides upstream connectivity to the internet via static NAT to MITT's public IP addresses. The default route on both routers points to the Capstone gateway at 10.128.250.254.

---

## 4. IP Addressing Scheme

### Subnet Allocation

| VLAN ID | VLAN Name    | Network           | Subnet Mask       | Usable Range              | Default Gateway (HSRP VIP) |
|---------|-------------|-------------------|--------------------|---------------------------|----------------------------|
| 10      | SERVER      | 10.10.10.0/24     | 255.255.255.0      | 10.10.10.1 – 10.10.10.253 | 10.10.10.254               |
| 15      | DMZ         | 10.10.15.0/24     | 255.255.255.0      | 10.10.15.1 – 10.10.15.253 | 10.10.15.254               |
| 20      | STAFF       | 10.10.20.0/24     | 255.255.255.0      | 10.10.20.1 – 10.10.20.253 | 10.10.20.254               |
| 30      | IT_ADMIN    | 10.10.30.0/24     | 255.255.255.0      | 10.10.30.1 – 10.10.30.253 | 10.10.30.254               |
| 40      | GUEST_WIFI  | 10.10.40.0/24     | 255.255.255.0      | 10.10.40.1 – 10.10.40.253 | 10.10.40.254               |
| 50      | MGMT        | 10.10.50.0/24     | 255.255.255.0      | 10.10.50.1 – 10.10.50.253 | 10.10.50.254               |
| —       | WAN (Capstone) | 10.128.250.0/24 | 255.255.255.0      | —                         | 10.128.250.254             |

### Static IP Assignments — Routers

| Device | Interface             | IP Address       | VLAN/Role        |
|--------|-----------------------|------------------|------------------|
| R-A    | Gi0/0/0.10            | 10.10.10.1       | SERVER           |
| R-A    | Gi0/0/0.15            | 10.10.15.1       | DMZ              |
| R-A    | Gi0/0/0.20            | 10.10.20.1       | STAFF            |
| R-A    | Gi0/0/0.30            | 10.10.30.1       | IT_ADMIN         |
| R-A    | Gi0/0/0.40            | 10.10.40.1       | GUEST_WIFI       |
| R-A    | Gi0/0/0.50            | 10.10.50.1       | MGMT             |
| R-A    | Gi0/0/1               | 10.128.250.11    | WAN (Capstone)   |
| R-B    | Gi0/0/1.10            | 10.10.10.2       | SERVER           |
| R-B    | Gi0/0/1.15            | 10.10.15.2       | DMZ              |
| R-B    | Gi0/0/1.20            | 10.10.20.2       | STAFF            |
| R-B    | Gi0/0/1.30            | 10.10.30.2       | IT_ADMIN         |
| R-B    | Gi0/0/1.40            | 10.10.40.2       | GUEST_WIFI       |
| R-B    | Gi0/0/1.50            | 10.10.50.2       | MGMT             |
| R-B    | Gi0/0/0               | 10.128.250.12    | WAN (Capstone)   |

### Static IP Assignments — Switches (Management)

| Device | Interface | IP Address    | VLAN |
|--------|-----------|---------------|------|
| S-A    | VLAN 50   | 10.10.50.5    | MGMT |
| S-B    | VLAN 50   | 10.10.50.3    | MGMT |
| S-C    | VLAN 50   | 10.10.50.6    | MGMT |

### Static IP Assignments — Servers and Services

| Device/Service       | IP Address     | VLAN | Role                                    |
|----------------------|----------------|------|-----------------------------------------|
| DB1 (DC1/DC2)       | 10.10.10.50    | 10   | Domain Controller, DNS Server, DHCP Server |
| Server-PT (Hyper-V) | 10.10.10.150   | 10   | Virtualization Host                     |
| WEB1                 | 10.10.15.10    | 15   | Web Server (fitlife-gym.ca)             |
| MAIL1                | 10.10.15.20    | 15   | Mail Server (SMTP/IMAP)                |
| Captive Portal / Secondary Web | 10.10.15.110 | 15 | Secondary Web Service             |
| Syslog/MGMT PC       | 10.10.50.250   | 50   | Centralized Logging                    |

### NAT Public Address Mappings

| Public (Capstone) IP | Internal IP    | Services           |
|----------------------|----------------|--------------------|
| 10.128.250.21        | 10.10.15.10    | HTTP (80), HTTPS (443) |
| 10.128.250.22        | 10.10.15.20    | SMTP (25), Submission (587), IMAPS (993) |

---

## 5. VLAN Design and Layer 2 Segmentation

The network uses six operational VLANs plus a blackhole VLAN for unused ports:

| VLAN ID | Name        | Purpose                                                                 |
|---------|-------------|-------------------------------------------------------------------------|
| 1       | Default     | Native VLAN — disabled on all interfaces, not used                      |
| 10      | SERVER      | Internal server infrastructure (AD, DNS, DHCP, Hyper-V)                |
| 15      | DMZ         | Demilitarized zone for public-facing services (web server, mail server)|
| 20      | STAFF       | Staff workstations — front desk, trainers, finance                     |
| 30      | IT_ADMIN    | IT administrator workstations with elevated access privileges           |
| 40      | GUEST_WIFI  | Guest wireless access — internet only, isolated from internal resources|
| 50      | MGMT        | Out-of-band management for network devices and syslog collection       |
| 999     | BLACKHOLE   | Unused/disabled ports assigned here and administratively shut down      |

All trunk links between switches and from switches to routers carry VLANs 1, 10, 15, 20, 30, 40, and 50 using 802.1Q encapsulation. VLAN 1 is included in the allowed list but is shut down on all SVIs to prevent any traffic from traversing it.

---

## 6. Network Device Inventory

| Device | Model     | Hostname | Role                          | Uplink                        |
|--------|-----------|----------|-------------------------------|-------------------------------|
| R-A    | ISR 4331  | R-A      | Primary Router / HSRP Active  | Gi0/0/1 → Capstone WAN       |
| R-B    | ISR 4331  | R-B      | Secondary Router / HSRP Standby | Gi0/0/0 → Capstone WAN     |
| S-A    | 2960-24TT | S-A      | Core/Distribution Switch      | Gi0/1 → R-A (trunk)          |
| S-B    | 2960-24TT | S-B      | Distribution Switch           | Gi0/1 → R-B (trunk)          |
| S-C    | 2960-24TT | S-C      | Access Layer Switch           | EtherChannel to S-A and S-B  |

All network devices are part of the **fitlife-gym.local** domain and run SSH version 2 for secure remote management.

---

## 7. Router Configurations

### 7.1 Router R-A (Primary)

R-A serves as the **primary gateway** for all VLANs with HSRP priority 110 and preempt enabled on every sub-interface. It connects to switch S-A via a trunk on GigabitEthernet0/0/0 using router-on-a-stick configuration with 802.1Q sub-interfaces for each VLAN.

**WAN Interface:** GigabitEthernet0/0/1 at 10.128.250.11/24, designated as the `ip nat outside` interface.

**Key Configuration Highlights:**

- Router-on-a-stick with six 802.1Q sub-interfaces on Gi0/0/0 (VLANs 10, 15, 20, 30, 40, 50).
- All internal sub-interfaces marked `ip nat inside`.
- DHCP relay (`ip helper-address 10.10.10.50`) configured on VLANs 20, 30, and 40 to forward DHCP requests to the server at DB1.
- HSRP group number matches the VLAN ID for consistency (group 10 on VLAN 10, group 15 on VLAN 15, etc.).
- ACLs applied inbound on VLAN 15 (both in and out), VLAN 20, VLAN 30, and VLAN 40.
- PAT configured using access list INSIDE-NET to overload through the WAN interface.
- Static NAT entries map DMZ services to Capstone public addresses.
- Default route to 10.128.250.254 for all internet-bound traffic.
- Syslog forwarding to 10.10.50.250.

### 7.2 Router R-B (Secondary)

R-B mirrors R-A's configuration as the **standby gateway** with HSRP priority 90 on all VLANs. It connects to switch S-B via a trunk on GigabitEthernet0/0/1.

**WAN Interface:** GigabitEthernet0/0/0 at 10.128.250.12/24, designated as `ip nat outside`.

R-B carries identical ACLs, NAT rules, DHCP relay settings, and routing configuration. In the event R-A fails, R-B automatically assumes the active HSRP role for all VLANs, maintaining uninterrupted network connectivity for all users and services.

---

## 8. Switch Configurations

### 8.1 Switch S-A (Core/Distribution)

S-A is the primary switching node. It connects upstream to R-A via Gi0/1 (trunk) and has EtherChannel links to both S-B and S-C.

**Port Assignments:**

| Port(s)     | Mode   | Assignment          | Description                  |
|-------------|--------|---------------------|------------------------------|
| Fa0/1–Fa0/2 | Trunk  | Po1 (to S-B)        | LACP EtherChannel            |
| Fa0/3       | Access | VLAN 15             | WEB_SERVER_PRIMARY           |
| Fa0/4       | Access | VLAN 15             | WEB_SERVER_BACKUP (shutdown) |
| Fa0/5       | Access | VLAN 50             | Management device            |
| Fa0/6       | Access | VLAN 10             | Server (DHCP snooping trust) |
| Fa0/7–Fa0/8 | Trunk  | Po2 (to S-C)        | LACP EtherChannel            |
| Fa0/9–Fa0/24| Access | VLAN 999            | Unused — shutdown            |
| Gi0/1       | Trunk  | To R-A              | Router uplink                |
| Gi0/2       | Access | VLAN 10             | Server (DHCP snooping trust) |

**Security Features Enabled:** DHCP Snooping (VLANs 10,15,20,30,40,50), Dynamic ARP Inspection (same VLANs), PortFast and BPDU Guard on all access ports, trusted ports on trunks and server/DHCP ports.

**Management IP:** 10.10.50.5 on VLAN 50 SVI.

### 8.2 Switch S-B (Distribution)

S-B serves as the secondary distribution switch, connecting upstream to R-B via Gi0/1 and forming EtherChannels to S-A (Po1) and S-C (Po3).

**Port Assignments:**

| Port(s)       | Mode   | Assignment          | Description                  |
|---------------|--------|---------------------|------------------------------|
| Fa0/1–Fa0/2   | Trunk  | Po1 (to S-A)        | LACP EtherChannel            |
| Fa0/3–Fa0/10  | Access | VLAN 999            | Unused — shutdown            |
| Fa0/11–Fa0/12 | Trunk  | Po3 (to S-C)        | LACP EtherChannel            |
| Fa0/13–Fa0/24 | Access | VLAN 999            | Unused — shutdown            |
| Gi0/1         | Trunk  | To R-B              | Router uplink                |
| Gi0/2         | Access | VLAN 999            | Unused — shutdown            |

S-B currently has no access-layer devices directly connected; it functions purely as a distribution/redundancy path.

**Management IP:** 10.10.50.3 on VLAN 50 SVI.

### 8.3 Switch S-C (Access Layer)

S-C is the primary access-layer switch where end-user devices and access points connect. It forms EtherChannels to both S-A (Po2) and S-B (Po3).

**Port Assignments:**

| Port(s)       | Mode   | Assignment          | Description                  |
|---------------|--------|---------------------|------------------------------|
| Fa0/1         | Access | VLAN 20             | Staff PC (Front-desk)        |
| Fa0/2         | Access | VLAN 30             | IT_Admin PC                  |
| Fa0/3         | Access | VLAN 20             | Trainers PC                  |
| Fa0/4         | Access | VLAN 20             | Finance PC                   |
| Fa0/5         | Access | VLAN 30             | Management_Tools PC          |
| Fa0/6         | Access | VLAN 20             | AP-Staff (Wireless AP)       |
| Fa0/7–Fa0/8   | Trunk  | Po2 (to S-A)        | LACP EtherChannel            |
| Fa0/9–Fa0/10  | Access | VLAN 999            | Unused — shutdown            |
| Fa0/11–Fa0/12 | Trunk  | Po3 (to S-B)        | LACP EtherChannel            |
| Fa0/13–Fa0/24 | Access | VLAN 999            | Unused — shutdown            |
| Gi0/1         | Access | VLAN 40             | AP-Guest (Guest Wireless AP) |
| Gi0/2         | Access | VLAN 999            | Unused — shutdown            |

**Management IP:** 10.10.50.6 on VLAN 50 SVI.

---

## 9. Redundancy and High Availability

### 9.1 HSRP — First Hop Redundancy

Hot Standby Router Protocol (HSRP) is configured on every VLAN sub-interface across both routers. The HSRP group number is aligned with the VLAN ID for operational clarity.

| VLAN | HSRP Group | Virtual IP      | R-A Priority | R-B Priority | Active Router |
|------|-----------|-----------------|-------------|-------------|---------------|
| 10   | 10        | 10.10.10.254    | 110         | 90          | R-A           |
| 15   | 15        | 10.10.15.254    | 110         | 90          | R-A           |
| 20   | 20        | 10.10.20.254    | 110         | 90          | R-A           |
| 30   | 30        | 10.10.30.254    | 110         | 90          | R-A           |
| 40   | 40        | 10.10.40.254    | 110         | 90          | R-A           |
| 50   | 50        | 10.10.50.254    | 110         | 90          | R-A           |

Preemption is enabled on both routers, meaning R-A will reclaim the active role automatically once it recovers from a failure. End devices use the virtual IP (x.x.x.254) as their default gateway, ensuring seamless failover.

### 9.2 EtherChannel — Link Aggregation

Three LACP (Link Aggregation Control Protocol) EtherChannels form a full-mesh triangle between all three switches, providing link redundancy and doubled bandwidth on each path.

| EtherChannel | Endpoints      | Member Ports (each side) | Mode  | Allowed VLANs           |
|-------------|----------------|--------------------------|-------|--------------------------|
| Po1         | S-A ↔ S-B      | Fa0/1, Fa0/2             | LACP (active) | 1,10,15,20,30,40,50 |
| Po2         | S-A ↔ S-C      | Fa0/7, Fa0/8 (S-A) ↔ Fa0/7, Fa0/8 (S-C) | LACP (active) | 1,10,15,20,30,40,50 |
| Po3         | S-B ↔ S-C      | Fa0/11, Fa0/12 (both)    | LACP (active) | 1,10,15,20,30,40,50 |

All EtherChannel member ports are configured as trunks with DHCP snooping trust and ARP inspection trust enabled to allow control-plane traffic to pass correctly.

### 9.3 Backup Routing

Both R-A and R-B maintain independent default routes to the Capstone gateway at 10.128.250.254. If one router fails, the surviving router continues to provide internet connectivity via its own WAN interface. The HSRP failover ensures internal clients seamlessly redirect traffic through the newly active gateway.

---

## 10. Network Address Translation (NAT)

NAT is configured identically on both R-A and R-B to ensure continuity during failover.

### PAT (Port Address Translation / Overload)

All internal traffic (10.10.0.0/16) is translated using PAT through the respective router's WAN interface:

- **R-A:** Overloads through GigabitEthernet0/0/1 (10.128.250.11)
- **R-B:** Overloads through GigabitEthernet0/0/0 (10.128.250.12)

The standard ACL `INSIDE-NET` permits the source range `10.10.0.0 0.0.255.255` for NAT.

### Static NAT Entries

Static NAT maps expose DMZ services to the Capstone WAN:

| Internal Host    | Internal Port | Capstone (Public) IP | External Port | Service              |
|-----------------|--------------|---------------------|--------------|----------------------|
| 10.10.15.10     | TCP 80       | 10.128.250.21       | TCP 80       | HTTP (Web Server)    |
| 10.10.15.10     | TCP 443      | 10.128.250.21       | TCP 443      | HTTPS (Web Server)   |
| 10.10.15.20     | TCP 25       | 10.128.250.22       | TCP 25       | SMTP (Mail Server)   |
| 10.10.15.20     | TCP 587      | 10.128.250.22       | TCP 587      | SMTP Submission      |
| 10.10.15.20     | TCP 993      | 10.128.250.22       | TCP 993      | IMAPS (Mail Server)  |

---

## 11. DHCP and DNS Services

### DHCP

The DHCP server runs on **DB1** at **10.10.10.50** (VLAN 10, Server VLAN). DHCP relay agents (`ip helper-address 10.10.10.50`) are configured on the router sub-interfaces for VLANs 20 (Staff), 30 (IT Admin), and 40 (Guest Wi-Fi). This allows clients in those VLANs to obtain IP addresses from the centralized DHCP server across VLAN boundaries.

VLANs 10 (Server), 15 (DMZ), and 50 (Management) use static IP addressing for infrastructure stability.

DHCP snooping is enabled on VLANs 10, 15, 20, 30, 40, and 50 across all three switches. Trusted ports include trunk uplinks, server ports, and the DHCP server port itself. The `no ip dhcp snooping information option` command is configured to prevent Option 82 insertion issues in the environment. Additionally, `ip dhcp relay information trust-all` is set on both routers.

### DNS

DNS resolution is provided by the domain controller DB1 at 10.10.10.50, serving both internal zone resolution for `fitlife-gym.local` and forwarding external queries. The GUEST_VLAN_40 ACL explicitly permits UDP DNS traffic to 10.10.10.50, ensuring guest clients can resolve domain names while remaining isolated from all other internal services.

---

## 12. Server and Service Infrastructure

### 12.1 Server VLAN (VLAN 10)

The Server VLAN hosts critical internal infrastructure:

| Server         | IP Address    | Services                                               |
|----------------|---------------|--------------------------------------------------------|
| DB1            | 10.10.10.50   | Active Directory (DC1/DC2), DNS Server, DHCP Server   |
| Server-PT (HV1)| 10.10.10.150  | Hyper-V Virtualization Host                            |

DB1 runs dual domain controllers (DC1 and DC2) providing Active Directory redundancy. NIC Teaming is configured on server connections to S-A for link-level redundancy, visible in the network topology as dual connections from the servers to the switch.

### 12.2 DMZ VLAN (VLAN 15)

The DMZ hosts services that must be accessible from both internal networks and the internet:

| Server | IP Address    | Services                            | Public NAT Address   |
|--------|---------------|-------------------------------------|----------------------|
| WEB1   | 10.10.15.10   | Web Server (HTTP/HTTPS), fitlife-gym.ca | 10.128.250.21    |
| MAIL1  | 10.10.15.20   | Mail Server (SMTP/Submission/IMAPS) | 10.128.250.22        |
| Secondary Web | 10.10.15.110 | Secondary Web Service / Captive Portal | —             |

The web server connects to back-end database services on VLAN 10 via permitted ACL entries for SQL Server (TCP 1433) and MySQL (TCP 3306).

---

## 13. Access Control Lists (ACLs)

ACLs enforce the principle of least privilege across all VLANs. Each ACL is applied inbound on the respective VLAN sub-interface at the router level.

### 13.1 Staff VLAN 20 ACL

**ACL Name:** `STAFF_VLAN_20_IN` — Applied inbound on VLAN 20 sub-interfaces of R-A and R-B.

**Policy Intent:** Staff can access the internet and internal web/application services but cannot use remote management protocols (SSH, Telnet, RDP) to connect to any internal infrastructure.

| # | Action | Protocol | Source          | Destination         | Port     | Purpose                         |
|---|--------|----------|-----------------|---------------------|----------|---------------------------------|
| 1 | Permit | UDP      | Any             | Any                 | BOOTPS   | Allow DHCP relay                |
| 2 | Deny   | TCP      | 10.10.20.0/24   | 10.10.0.0/16        | 22       | Block SSH to internal hosts     |
| 3 | Deny   | TCP      | 10.10.20.0/24   | 10.10.0.0/16        | Telnet   | Block Telnet to internal hosts  |
| 4 | Deny   | TCP      | 10.10.20.0/24   | 10.10.0.0/16        | 3389     | Block RDP to internal hosts     |
| 5 | Permit | IP       | 10.10.20.0/24   | Any                 | —        | Allow all other traffic         |

### 13.2 IT Admin VLAN 30 ACL

**ACL Name:** `IT_ADMIN_VLAN_30_IN` — Applied inbound on VLAN 30 sub-interfaces.

**Policy Intent:** IT administrators have full internet access plus explicit permissions for remote management of servers (RDP, WinRM, SSH, HTTPS) and network management devices (SSH, HTTPS on VLAN 50).

| # | Action | Protocol | Source          | Destination         | Port     | Purpose                          |
|---|--------|----------|-----------------|---------------------|----------|----------------------------------|
| 1 | Permit | UDP      | Any             | Any                 | BOOTPS   | Allow DHCP relay                 |
| 2 | Permit | TCP      | 10.10.30.0/24   | 10.10.50.0/24       | 22       | SSH to management VLAN           |
| 3 | Permit | TCP      | 10.10.30.0/24   | 10.10.50.0/24       | 443      | HTTPS to management VLAN         |
| 4 | Permit | TCP      | 10.10.30.0/24   | 10.10.10.0/24       | 3389     | RDP to servers                   |
| 5 | Permit | TCP      | 10.10.30.0/24   | 10.10.10.0/24       | 5985     | WinRM HTTP to servers            |
| 6 | Permit | TCP      | 10.10.30.0/24   | 10.10.10.0/24       | 5986     | WinRM HTTPS to servers           |
| 7 | Permit | TCP      | 10.10.30.0/24   | 10.10.10.0/24       | 22       | SSH to servers                   |
| 8 | Permit | TCP      | 10.10.30.0/24   | 10.10.10.0/24       | 443      | HTTPS to servers                 |
| 9 | Permit | IP       | 10.10.30.0/24   | Any                 | —        | Allow all other traffic          |

### 13.3 Guest Wi-Fi VLAN 40 ACL

**ACL Name:** `GUEST_VLAN_40_IN` — Applied inbound on VLAN 40 sub-interfaces.

**Policy Intent:** Guest users may only resolve DNS, access the gym's web server (and secondary web service), browse the internet via HTTP/HTTPS, and use ICMP. All access to internal VLANs is explicitly denied.

| # | Action | Protocol | Source          | Destination              | Port     | Purpose                       |
|---|--------|----------|-----------------|--------------------------|----------|-------------------------------|
| 1 | Permit | UDP      | Any             | Any                      | BOOTPS   | Allow DHCP relay              |
| 2 | Permit | UDP      | 10.10.40.0/24   | 10.10.10.50              | DNS      | DNS queries to internal DNS   |
| 3 | Permit | TCP      | 10.10.40.0/24   | 10.10.15.10              | 80       | HTTP to gym web server        |
| 4 | Permit | TCP      | 10.10.40.0/24   | 10.10.15.10              | 443      | HTTPS to gym web server       |
| 5 | Permit | TCP      | 10.10.40.0/24   | 10.10.15.110             | 80       | HTTP to secondary web service |
| 6 | Permit | TCP      | 10.10.40.0/24   | 10.10.15.110             | 443      | HTTPS to secondary web service|
| 7 | Permit | UDP      | Any             | 224.0.0.2                | 1985     | HSRP multicast                |
| 8 | Permit | ICMP     | Any             | Any                      | —        | Allow ping (troubleshooting)  |
| 9 | Deny   | IP       | 10.10.40.0/24   | 10.10.10.0/24            | —        | Block access to Server VLAN   |
| 10| Deny   | IP       | 10.10.40.0/24   | 10.10.15.0/24            | —        | Block access to DMZ VLAN      |
| 11| Deny   | IP       | 10.10.40.0/24   | 10.10.20.0/24            | —        | Block access to Staff VLAN    |
| 12| Deny   | IP       | 10.10.40.0/24   | 10.10.30.0/24            | —        | Block access to IT Admin VLAN |
| 13| Deny   | IP       | 10.10.40.0/24   | 10.10.50.0/24            | —        | Block access to MGMT VLAN     |
| 14| Deny   | IP       | 10.10.40.0/24   | 10.10.60.0/24            | —        | Block access to reserved range|
| 15| Permit | UDP      | 10.10.40.0/24   | Any                      | DNS      | External DNS resolution       |
| 16| Permit | TCP      | 10.10.40.0/24   | Any                      | 80       | Internet HTTP browsing        |
| 17| Permit | TCP      | 10.10.40.0/24   | Any                      | 443      | Internet HTTPS browsing       |
| 18| Deny   | IP       | Any             | Any                      | —        | Implicit deny all             |

This is the most restrictive ACL in the environment, implementing a whitelist-only approach for guest users.

### 13.4 DMZ VLAN 15 ACLs

Two ACLs control traffic flow into and out of the DMZ:

**`DMZ_VLAN_15_IN`** (applied inbound — traffic originating from the DMZ):

- The web server (10.10.15.10) is permitted to send HTTP/HTTPS responses to VLANs 20, 30, 40, 50, and 10.
- The web server is permitted to connect to database services on VLAN 10 (TCP 1433 for SQL Server, TCP 3306 for MySQL).
- All other DMZ-to-internal traffic is denied.
- DMZ hosts can freely reach the internet (permit ip to any after the internal deny).

**`DMZ_VLAN_15_OUT`** (applied outbound — traffic destined to the DMZ):

- Internal networks (10.10.0.0/16) can reach DMZ hosts.
- External sources can reach the web server on HTTP/HTTPS and the mail server on SMTP, Submission, and IMAPS.
- All other inbound DMZ traffic is denied.

### 13.5 VTY Access Control

**ACL Name:** `VTY_ACCESS` — Applied to all VTY lines (0–15) on every network device.

Only hosts on VLAN 30 (IT_ADMIN, 10.10.30.0/24) and VLAN 50 (MGMT, 10.10.50.0/24) can SSH into routers and switches. All other source networks are denied. Transport is restricted to SSH only (`transport input ssh`).

---

## 14. Wireless LAN Design

Two wireless access points provide Wi-Fi coverage for different user groups:

| Access Point | Connected To         | Port    | VLAN | SSID Purpose           |
|-------------|----------------------|---------|------|------------------------|
| AP-Staff    | S-C, Fa0/6           | Access  | 20   | Staff wireless access  |
| AP-Guest    | S-C, Gi0/1           | Access  | 40   | Guest/member Wi-Fi     |

The staff AP bridges wireless clients directly into VLAN 20, giving them the same network access as wired staff workstations. The guest AP bridges into VLAN 40, where the restrictive GUEST_VLAN_40_IN ACL ensures isolation from all internal resources.

---

## 15. Security Hardening

### 15.1 Switch Hardening

All three switches implement the following security measures:

- **DHCP Snooping** enabled on VLANs 10, 15, 20, 30, 40, and 50. Only designated trusted ports (trunks, server ports, DHCP server port) can send DHCP server messages. This prevents rogue DHCP server attacks.
- **Dynamic ARP Inspection (DAI)** enabled on the same VLANs. DAI validates ARP packets against the DHCP snooping binding table, preventing ARP spoofing and man-in-the-middle attacks. Trusted ports are configured on trunks and server connections.
- **PortFast** enabled on all access ports to allow immediate transition to the forwarding state, eliminating the 30-second STP delay for end devices.
- **BPDU Guard** enabled on all access (PortFast) ports. If a BPDU is received on these ports, the port is automatically disabled, preventing unauthorized switches or STP manipulation attacks.
- **Unused Port Security** — All unused ports are assigned to VLAN 999 (blackhole VLAN), set to access mode, and administratively shut down. This prevents unauthorized devices from connecting to the network.
- **Spanning Tree Protocol** — PVST (Per-VLAN Spanning Tree) is configured across all switches, providing loop prevention on a per-VLAN basis.
- **VLAN 1 disabled** — The default VLAN 1 SVI is shut down with no IP address on all switches.

### 15.2 Router Hardening

- **SSH Version 2** enforced on both routers (`ip ssh version 2`). Telnet is not enabled.
- **VTY line access** restricted to VLAN 30 and VLAN 50 via the `VTY_ACCESS` standard ACL.
- **Domain name** configured as `fitlife-gym.local` for SSH key generation.
- **Username-based authentication** with the `Admin` user account.
- **Unused interfaces** (Gi0/0/2, Serial0/1/0, Serial0/1/1) are administratively shut down on both routers.
- **Console logging synchronous** enabled to prevent log messages from disrupting CLI input.

### 15.3 Network Device Access Control

Remote management access to all network devices follows a strict policy:

- Only SSH (no Telnet) is permitted on VTY lines.
- Access is restricted to source IPs in VLAN 30 (IT Admin) and VLAN 50 (Management).
- The `Admin` user account is configured on both routers.
- Switches are manageable via their VLAN 50 SVI addresses.

### 15.4 Logging and Monitoring

All five network devices (R-A, R-B, S-A, S-B, S-C) send syslog messages to the centralized logging server at **10.10.50.250** in the Management VLAN. This provides a single point of log collection for security monitoring, troubleshooting, and audit trail purposes.

NetFlow version 9 is enabled on both routers (`ip flow-export version 9`) for traffic flow analysis and network monitoring.

---

## 16. Active Directory and Domain Services

The internal domain **fitlife-gym.local** is managed by dual domain controllers (DC1 and DC2) running on DB1 at 10.10.10.50 in the Server VLAN.

**Active Directory provides:**

- **User Accounts** — Centralized authentication for all staff and IT administrator accounts.
- **Security Groups** — Role-based groups for Staff, IT Admins, Trainers, Finance, and Management to control access to resources.
- **Group Policy Objects (GPOs)** — Centralized configuration management for domain-joined workstations, including security policies, software deployment, and desktop configuration.
- **Network Shares** — Domain-wide file and folder sharing with NTFS and share-level ACLs to enforce access control based on group membership.
- **Dual Domain Controllers** — DC1 and DC2 provide directory service redundancy; if one domain controller fails, the other continues to authenticate users and process Group Policy.

---

## 17. Email Services

The mail server MAIL1 (10.10.15.20) in the DMZ provides internal and external email communication using the custom domain namespace.

**Supported Protocols:**

| Protocol | Port | Direction | Purpose                     |
|----------|------|-----------|-----------------------------|
| SMTP     | 25   | Inbound   | Receiving email from internet|
| Submission | 587 | Outbound  | Sending email (authenticated)|
| IMAPS    | 993  | Inbound   | Secure mailbox access (IMAP over TLS)|

The mail server is accessible from the internet via static NAT to 10.128.250.22. Internal users across all staff VLANs can access their email through IMAPS, and the server can relay outbound messages through standard SMTP.

---

## 18. Web Application Services

WEB1 (10.10.15.10) hosts the FitLife Gym website at **http://www.fitlife-gym.ca**, providing class schedules, membership information, and other public-facing content.

**Architecture:**

- **Front-end:** Web server on VLAN 15 (DMZ) serving HTTP (port 80) and HTTPS (port 443).
- **Back-end:** Database connectivity to servers on VLAN 10 via SQL Server (TCP 1433) and MySQL (TCP 3306), permitted by the DMZ ACL.
- **Public Access:** Available to the internet via static NAT mapping to 10.128.250.21.
- **Internal Access:** Accessible from Staff (VLAN 20), IT Admin (VLAN 30), Guest (VLAN 40), and Management (VLAN 50) networks.

A secondary web service at 10.10.15.110 is also available, accessible from the Guest VLAN, potentially serving as a captive portal or member-facing web application.

---

## 19. Physical Topology Summary

The physical layout connects as follows:

```
                    [MITT Capstone WAN: 10.128.250.0/24]
                         |                    |
                    Gi0/0/1              Gi0/0/0
                    [R-A]  ← HSRP →  [R-B]
                    Gi0/0/0              Gi0/0/1
                         |                    |
                    Gi0/1                Gi0/1
                    [S-A] ──── Po1 ──── [S-B]
                      |  \              /  |
                      |   Po2      Po3    |
                      |      \    /       |
                      |       [S-C]       |
                      |     (Access)      |
                      |                   |
              ┌───────┴───────┐     
         Fa0/3,Fa0/6,Gi0/2   Fa0/5
         [DMZ Servers]     [Servers]  [MGMT]
         [WEB1, MAIL1]    [DB1, HV1]

                    S-C Access Ports:
              Fa0/1: Staff PC (VLAN 20)
              Fa0/2: IT Admin PC (VLAN 30)
              Fa0/3: Trainers PC (VLAN 20)
              Fa0/4: Finance PC (VLAN 20)
              Fa0/5: Mgmt Tools PC (VLAN 30)
              Fa0/6: AP-Staff (VLAN 20)
              Gi0/1: AP-Guest (VLAN 40)
```

---

## 20. Running Configurations Reference

The complete running configurations for all five network devices are maintained as separate reference files:

| Device | Configuration File | Description                           |
|--------|--------------------|---------------------------------------|
| R-A    | R-A.txt            | Primary router, HSRP active, NAT, ACLs |
| R-B    | R-B.txt            | Secondary router, HSRP standby, NAT, ACLs |
| S-A    | S-A.txt            | Core/distribution switch, server connections |
| S-B    | S-B.txt            | Distribution switch, R-B uplink       |
| S-C    | S-C.txt            | Access layer switch, end-user ports   |

---

## Appendix A: Credential Management

All network devices use the following administrative credentials:

- **Username:** Admin
- **SSH Access:** Enabled (SSHv2 only)
- **Domain:** fitlife-gym.local

Credentials should be stored securely using a password management solution. In a production environment, RADIUS or TACACS+ authentication would be implemented for centralized AAA (Authentication, Authorization, and Accounting).

---

## Appendix B: Summary of Security Controls by Layer

| Layer | Control                    | Scope                              |
|-------|----------------------------|------------------------------------|
| Physical | Unused ports shutdown, VLAN 999 | All switches                 |
| Data Link | DHCP Snooping, DAI, BPDU Guard, PortFast | All switches       |
| Network | VLAN segmentation, ACLs    | Routers (R-A, R-B)                 |
| Transport | Port-specific ACL rules    | Per-VLAN on routers                |
| Application | SSH v2 only, VTY ACL      | All network devices                |
| Monitoring | Syslog, NetFlow v9         | All devices → 10.10.50.250         |

---

*End of Documentation*
