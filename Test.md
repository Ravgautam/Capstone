# FitLife Gyms Enterprise Network – Layer 2 Switch Design

## 1. Executive Overview

This document describes the revised enterprise network architecture for the fictional organization **FitLife Gyms**, implemented in the NSA630 Capstone lab environment.
The original design used multilayer switches (SW1, SW2) for inter‑VLAN routing; this revision keeps the same physical topology and services but constrains **all switches to Layer 2 only**, moving all Layer 3 and routing functions to the edge routers using a **router‑on‑a‑stick** model.

The solution delivers:
- Segmented wired and wireless access for Staff, IT/Admin, Servers, Guest Wi‑Fi, and optional VoIP.
- Internal services: Active Directory, DNS, DHCP, file shares, monitoring and vulnerability scanning.
- External services: public web site and email with custom domain, reachable via NAT from the Internet.
- Redundancy at the network and server layers using dual routers, L2 redundancy, and virtualized servers.

This document is suitable as implementation and handover documentation for the capstone project.

---

## 2. Scenario and Constraints

### 2.1 Business and Lab Context

- Organization: **FitLife Gyms**, a growing fitness company with one flagship location represented in the lab, with planned expansion to additional sites.
- Business requirements:
  - Internet access for staff and management.
  - Guest Wi‑Fi with Internet‑only access.
  - Centralized authentication and authorization (Active Directory).
  - Internal email and public email with custom domain.
  - Public web presence with optional member portal.
  - Segmented networks for security and manageability.

- Lab environment (NSA630 Capstone):
  - Cisco routers and switches in racks.
  - Lab PCs running Type 1 hypervisors (ESXi/Hyper‑V) hosting server and client VMs.
  - **Capstone network**: 10.128.250.0/24, providing upstream Internet connectivity.
  - At least two public IP addresses reserved for the group; MITT performs static NAT between these public IPs and the group’s addresses in 10.128.250.0/24.

### 2.2 Design Constraints

- **Only Layer 2 switches** are allowed:
  - SW1, SW2, SW3 must operate purely at Layer 2 (no inter‑VLAN routing, no user‑VLAN SVIs).
- Topology must remain the same as the original project:
  - Dual edge routers (R1, R2).
  - Two “core/distribution” switches (SW1, SW2).
  - One access switch (SW3).
  - Two virtualization hosts (HV1, HV2).
- The revised design must still meet the NSA630 rubric requirements for:
  - VLANs and subnetting.
  - Inter‑VLAN connectivity.
  - Redundancy (FHRP, link redundancy, backup routes).
  - Server and services architecture.
  - Security hardening and monitoring.

---

## 3. Logical Topology Overview

### 3.1 High‑Level Components

- **Edge Routers: R1 and R2**
  - WAN edge to Capstone network for Internet access.
  - Perform **all inter‑VLAN routing** using router‑on‑a‑stick subinterfaces.
  - Run FHRP (HSRP) for each VLAN default gateway.
  - Implement NAT/PAT for outbound Internet access and inbound publishing of web and email services.

- **Core/Distribution Switches: SW1 and SW2 (Layer 2 only)**
  - Provide Layer 2 trunking between routers, access switch, and hypervisors.
  - Interconnected via EtherChannel for redundancy and increased bandwidth.
  - No Layer 3 SVIs for user VLANs; only management IPs (e.g., in MGMT VLAN) if permitted.

- **Access Switch: SW3 (Layer 2)**
  - Connects wired Staff and IT/Admin PCs.
  - Provides uplinks for wireless APs.
  - Uplinks via trunks (or EtherChannels) to SW1 and SW2.

- **Wireless Access Points: AP1 and AP2**
  - Provide dual SSIDs: Staff and Guest.
  - Map SSIDs to VLANs 20 (STAFF) and 40 (GUEST_WIFI).

- **Virtualization Hosts: HV1 and HV2**
  - Type 1 hypervisors hosting server and client VMs.
  - Trunk links to SW1/SW2 carrying server, DMZ, IT, MGMT, and VoIP VLANs.

- **Server VMs** (Minimum set):
  - DC1, DC2 – Domain Controllers with AD DS, DNS, optional file services.
  - DHCP1, DHCP2 – DHCP servers (can be separate or roles on DCs).
  - WEB1 – Web server for public site and optional member portal.
  - MAIL1 – Email server for internal and external mail.
  - DB1 – Database server used by WEB1.
  - MGMT1 – Monitoring and vulnerability scanning platform.
  - Optional: WEB2, MAIL2, additional management or logging servers.

### 3.2 Textual Logical Diagram

```text
                Internet / Public IPs
                         |
                    Capstone Network
                      10.128.250.0/24
                         |
                    [Edge Routers]
                    R1 -------- R2
                     |  HSRP   |
                     +---------+
                  L2 trunks (all VLANs)
                     |       |
               +-----+-------+------+
               |  Core / Distribution |
               |   SW1 ======= SW2    |
               |    ^   EtherChannel  |
               +----|-----------------+
                    |        
                   Trunks
                    |
               [Access Switch SW3]
          _________|___________
         |         |          |
     Staff PCs   IT PCs    AP1/AP2
     (VLAN 20)  (VLAN 30)  (Trunk: VLAN 20,40)

    [Virtualization Hosts HV1, HV2]
          |                  |
   L2 trunks (VLAN 10,15,30,50,60)
          |                  |
     +----+----+        +----+----+
     | Servers |        | Servers |
     | DC1,    |        | DHCP,   |
     | DC2,    |        | WEB,    |
     | DB1,    |        | MAIL,   |
     | etc.    |        | MGMT1   |
     +---------+        +---------+
```

Guests connect to SSID **"FitLife‑Guest"** in VLAN 40 and are restricted to Internet‑only access.
Staff connect to SSID **"FitLife‑Staff"** (VLAN 20) with enterprise security.

---

## 4. VLAN and IP Addressing Plan

### 4.1 VLANs and Subnets

The internal address space is 10.10.0.0/16, split into /24 networks per VLAN.
Default gateways are implemented on router subinterfaces (HSRP VIPs), not on switches.

| VLAN ID | Name            | Purpose                         | Subnet           | Gateway (HSRP VIP) |
|--------:|-----------------|---------------------------------|------------------|--------------------|
| 10      | SERVER          | Windows/Linux servers, AD, DB   | 10.10.10.0/24    | 10.10.10.254       |
| 15      | DMZ             | Public web and mail servers     | 10.10.15.0/24    | 10.10.15.254       |
| 20      | STAFF           | Staff workstations              | 10.10.20.0/24    | 10.10.20.254       |
| 30      | IT_ADMIN        | IT/admin workstations           | 10.10.30.0/24    | 10.10.30.254       |
| 40      | GUEST_WIFI      | Guest wireless                  | 10.10.40.0/24    | 10.10.40.254       |
| 50      | MGMT            | Network and hypervisor mgmt     | 10.10.50.0/24    | 10.10.50.254       |
| 60      | VOIP (optional) | IP phones and voice gateways    | 10.10.60.0/24    | 10.10.60.254       |

### 4.2 DHCP Scope Design

- VLAN 10 (SERVER):
  - Servers use static addresses (e.g., 10.10.10.10–10.10.10.100).
  - No general DHCP scope.

- VLAN 15 (DMZ):
  - Public‑facing WEB/MAIL servers use static addresses (e.g., 10.10.15.10–10.10.15.50).
  - No general DHCP scope.

- VLAN 20 (STAFF):
  - Scope: 10.10.20.100–10.10.20.200.
  - Exclusions: gateway, servers, infrastructure addresses.

- VLAN 30 (IT_ADMIN):
  - Scope: 10.10.30.100–10.10.30.200.

- VLAN 40 (GUEST_WIFI):
  - Scope: 10.10.40.50–10.10.40.230.
  - Shorter lease times to accommodate transient guest devices.

- VLAN 50 (MGMT):
  - Static addresses for routers, switches, hypervisors, and MGMT1.

DHCP relay is configured on router subinterfaces so that all VLANs can be served by centralized DHCP servers.

---

## 5. Layer 3 Design – Router‑on‑a‑Stick

### 5.1 Inter‑VLAN Routing on R1/R2

All routing is performed on the routers using IEEE 802.1Q subinterfaces.
Each router has a trunk link toward the core/distribution Layer 2 switches (SW1/SW2).

For each VLAN \(10, 15, 20, 30, 40, 50, 60\):
- R1 has subinterface `G0/0.<VLAN>` with IP address in the corresponding subnet.
- R2 has matching subinterface `G0/0.<VLAN>`.
- HSRP is configured per VLAN to provide a virtual default gateway.

### 5.2 Example Router Subinterface Configuration

**R1 (simplified example):**

```plaintext
interface GigabitEthernet0/0.10
 encapsulation dot1q 10
 ip address 10.10.10.1 255.255.255.0
 standby 10 ip 10.10.10.254
 standby 10 priority 110
 standby 10 preempt

interface GigabitEthernet0/0.15
 encapsulation dot1q 15
 ip address 10.10.15.1 255.255.255.0
 standby 15 ip 10.10.15.254
 standby 15 priority 110
 standby 15 preempt

interface GigabitEthernet0/0.20
 encapsulation dot1q 20
 ip address 10.10.20.1 255.255.255.0
 standby 20 ip 10.10.20.254
 standby 20 priority 110
 standby 20 preempt

! ... repeat for VLAN 30, 40, 50, 60
```

**R2 (matching, lower priority):**

```plaintext
interface GigabitEthernet0/0.20
 encapsulation dot1q 20
 ip address 10.10.20.2 255.255.255.0
 standby 20 ip 10.10.20.254
 standby 20 priority 90
 standby 20 preempt

! similarly for other VLANs
```

### 5.3 Default Routing and Internet Connectivity

- R1 and R2 each have a default route pointing toward the Capstone gateway (e.g., 10.128.250.1).

```plaintext
ip route 0.0.0.0 0.0.0.0 10.128.250.1
```

- The Capstone network in turn provides routes to the Internet and performs its own NAT to the group’s public IPs.

### 5.4 NAT and Port Forwarding

- **Dynamic PAT:**
  - Overload internal addresses 10.10.0.0/16 to a single public IP for outbound Internet access.

- **Static NAT / Port Forwarding:**
  - Map Public IP #1:443 (and optionally 80) to WEB1 in VLAN 15.
  - Map Public IP #2:25/587/993 to MAIL1 in VLAN 15.

Example (conceptual):

```plaintext
ip nat inside source list INSIDE-NET interface GigabitEthernet0/1 overload

ip nat inside source static tcp 10.10.15.10 443 interface GigabitEthernet0/1 443
ip nat inside source static tcp 10.10.15.20 25  interface GigabitEthernet0/1 25
ip nat inside source static tcp 10.10.15.20 587 interface GigabitEthernet0/1 587
ip nat inside source static tcp 10.10.15.20 993 interface GigabitEthernet0/1 993
```

Where:
- `GigabitEthernet0/1` is the interface toward the Capstone / public side.
- ACL `INSIDE-NET` matches traffic from 10.10.0.0/16.

---

## 6. Layer 2 Switching Design

### 6.1 Trunks and VLAN Propagation

- Links from R1 and R2 to the core switches (SW1, SW2) are configured as **802.1Q trunks** carrying all internal VLANs.
- SW1 and SW2 are interconnected via an **EtherChannel** bundle, also configured as a trunk for all VLANs.
- SW3 uplinks to SW1 and SW2 using trunks (or EtherChannels) carrying the required VLANs (20, 30, 40, 50, etc.).
- Hypervisor uplinks to SW1 and SW2 are trunks carrying server/DMZ/IT/MGMT/VoIP VLANs.

All VLANs are created consistently on SW1, SW2, and SW3:

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

### 6.2 EtherChannel and STP

- SW1–SW2:
  - Multiple physical links bundled as an EtherChannel using LACP.
  - Channel group configured as a trunk.

- SW3 uplinks:
  - Either a single EtherChannel to SW1/SW2 (if permitted) or dual independent trunks.
  - Spanning Tree Protocol (STP) is tuned so that SW1 (or SW2) is the root bridge for key VLANs.

STP features:
- PortFast and BPDU Guard enabled on all edge/access ports.
- Root guard optional on trunk ports where appropriate.

### 6.3 Access Port Assignments

- Staff PCs on SW3: access VLAN 20.
- IT/Admin PCs on SW3: access VLAN 30.
- AP1/AP2 on SW3: trunk ports allowing VLAN 20 and 40.
- Hypervisor management NICs (if separate): access VLAN 50.

Example access configuration:

```plaintext
interface FastEthernet0/10
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable
```

---

## 7. Wireless Network Design

- **SSID "FitLife‑Staff"**:
  - Mapped to VLAN 20.
  - Uses WPA2/WPA3‑Enterprise with 802.1X authentication.
  - RADIUS server: NPS on DC1 (and optionally DC2).

- **SSID "FitLife‑Guest"**:
  - Mapped to VLAN 40.
  - Uses WPA2‑PSK or open SSID with captive portal on WEB1.
  - ACLs on routers restrict guests to Internet‑only access.

APs connect as trunks to SW3, carrying VLAN 20 and 40; corresponding VLANs propagate through SW1/SW2 to R1/R2.

---

## 8. Segmentation and Security Policy

### 8.1 VLAN‑Level Segmentation

- **SERVER (VLAN 10)** – internal servers such as domain controllers, database, monitoring.
- **DMZ (VLAN 15)** – public‑facing WEB1/WEB2 and MAIL1/MAIL2.
- **STAFF (VLAN 20)** – staff workstations for trainers and front‑desk.
- **IT_ADMIN (VLAN 30)** – IT and network administrators.
- **GUEST_WIFI (VLAN 40)** – Internet‑only guest access.
- **MGMT (VLAN 50)** – management IPs for network devices, hypervisors, and MGMT1.
- **VOIP (VLAN 60)** – optional voice endpoints.

### 8.2 Inter‑VLAN ACLs on Routers

All inter‑VLAN security policies are implemented using extended ACLs applied on router subinterfaces.

Example policies:

- **Guest VLAN (40):**
  - Permit: DNS, HTTP, HTTPS from VLAN 40 to Internet.
  - Deny: any access to VLANs 10, 15, 20, 30, 50, 60.

- **Staff VLAN (20):**
  - Permit: access to AD (LDAP/Kerberos), internal DNS, DHCP, file shares, internal web apps.
  - Deny: direct management access to routers, switches, hypervisors, and DMZ management.

- **IT_ADMIN VLAN (30):**
  - Permit: SSH/HTTPS to routers and switches (MGMT VLAN addresses).
  - Permit: RDP/WinRM to servers.
  - Permit: HTTPS/SSH to hypervisors and MGMT1.

- **DMZ VLAN (15):**
  - Permit inbound from Internet only for published ports (80/443 for WEB, 25/587/993 for MAIL).
  - Deny DMZ‑initiated access to internal VLANs except DB1 access from WEB1 to SERVER VLAN.

Management plane protection:
- Restrict vty access on routers to MGMT/IT VLAN ranges.
- Use strong local credentials and/or RADIUS/TACACS+.

---

## 9. Server and Services Architecture

### 9.1 Active Directory and Identity

- AD domain: `fitlife-gym.local`.
- Domain controllers: DC1 and DC2 (Windows Server) in VLAN 10.
- Roles:
  - AD DS.
  - Internal DNS.
  - Optional DFS/file services.

Logical structure:
- OUs: `HQ-Staff`, `Trainers`, `IT`, `Servers`, `Workstations`, `Guest-Accounts`.
- Security groups: `IT-Admins`, `FrontDesk-Staff`, `Trainers`, `Finance`, etc.
- GPOs for password policies, lockout policies, security baselines, drive mappings, and desktop restrictions.

### 9.2 DNS (Internal and External)

- **Internal DNS** (on DC1/DC2):
  - Forward lookup zone: `fitlife-gym.local`.
  - Records for all servers and important services.
  - Reverse lookup zones for key subnets.

- **External DNS** (on public DNS provider or simulated BIND VM):
  - Public zone: `fitlife-gym.ca`.
  - A record: `www.fitlife-gym.ca` -> Public IP #1.
  - MX record: `mail.fitlife-gym.ca` -> Public IP #2.

### 9.3 DHCP

- DHCP1 (primary) and DHCP2 (secondary/failover) in VLAN 10.
- Scopes for VLANs 20, 30, 40.
- Options set per scope:
  - Default gateway: HSRP VIP (10.10.x.254).
  - DNS servers: DC1/DC2 IPs.
  - Domain name: `fitlife-gym.local`.
  - Lease durations: longer for staff/IT, shorter for guests.

Routers configured with `ip helper-address` on each VLAN subinterface to relay DHCP broadcasts to DHCP1/DHCP2.

### 9.4 Email System

- MAIL1 in DMZ VLAN 15.
- SMTP/IMAP solution (e.g., hMailServer or Postfix+Dovecot).
- Email addresses: `user@fitlife-gym.ca`.
- Features:
  - Internal send/receive.
  - External send/receive via Public IP #2 and MX record.
  - TLS for SMTP and IMAP.

### 9.5 Web Server and Backend

- WEB1 in DMZ VLAN 15.
- Hosts:
  - Public website (class schedules, membership info).
  - Optional member portal.
- DB1 in SERVER VLAN 10, reachable only from WEB1 on specific DB port (e.g., 3306/TCP).
- Optional WEB2 on second hypervisor for redundancy.

### 9.6 File Services and Shares

- File services provided by DC1/DC2 or a dedicated file server in VLAN 10.
- Typical shares:
  - Staff share, departmental shares, IT admin share.
  - Public read‑only share for common documents.
- NTFS and share permissions based on AD groups.

### 9.7 Monitoring and Vulnerability Scanning

- MGMT1 in MGMT VLAN 50.
- Runs:
  - Network monitoring (e.g., Zabbix/LibreNMS).
  - Central syslog receiver.
  - Vulnerability scanner (e.g., OpenVAS or Nessus Essentials).
- Collects logs from routers, switches, servers.

---

## 10. Internet Access and Capstone Integration

- R1 and R2 each have an interface in the Capstone network 10.128.250.0/24.
- MITT performs static NAT from this subnet to the group’s two (or more) public IP addresses.
- Internal hosts use PAT via R1/R2 to access the Internet.
- External users reach WEB1 and MAIL1 via public IPs and static NAT/port forwarding.

Routing behavior:
- Internal VLANs are directly connected on R1/R2.
- Default route points to the Capstone gateway.
- No internal routing is required on switches.

---

## 11. Security Hardening

### 11.1 Network Device Hardening

- Disable unused switch and router ports, place in an unused VLAN.
- Use SSHv2 for CLI access; disable Telnet.
- Use HTTPS for web management (if required); disable HTTP.
- Configure local user accounts with strong passwords and privilege levels.
- Limit vty access using ACLs (only IT/MGMT subnets).
- Enable logging to MGMT1.

Layer 2 protections:
- **DHCP Snooping** on access VLANs.
- **Dynamic ARP Inspection** where supported.
- **IP Source Guard** on sensitive VLANs.
- **BPDU Guard** and **PortFast** on access ports.

### 11.2 Server and Endpoint Hardening

- Enable host firewalls.
- Install and maintain anti‑malware on Windows systems.
- Apply OS and application updates regularly.
- Use least privilege on service accounts and admin roles.

### 11.3 Monitoring and Incident Response

- Centralized logging to MGMT1.
- Scheduled vulnerability scans against internal and DMZ hosts.
- Regular Nmap scans from MGMT1 to verify ACLs and exposed services.

---

## 12. Redundancy and High Availability

### 12.1 Network Redundancy

- **Routers:**
  - R1 and R2 run HSRP for each VLAN gateway.
  - Interface tracking ensures the router with an active Capstone link is preferred.

- **Links and Switches:**
  - EtherChannel between SW1 and SW2.
  - Dual uplinks between SW3 and SW1/SW2.
  - STP provides loop‑free topology with fast convergence.

### 12.2 Server and Service Redundancy

- Two hypervisors (HV1, HV2) hosting server VMs.
- Redundant DCs (DC1/DC2) and DNS.
- DHCP failover or split scopes.
- Optional WEB2 and MAIL2 for HA.

---

## 13. Implementation Checklist

1. **Create VLANs** on SW1, SW2, SW3.
2. **Configure trunks** between routers and SW1/SW2, between SW1/SW2 themselves, and from SW3 to SW1/SW2.
3. **Enable EtherChannel** on SW1–SW2 and (optionally) SW3 uplinks.
4. **Configure HSRP + subinterfaces** on R1 and R2 for all VLANs.
5. **Set default routes** on R1/R2 toward Capstone.
6. **Configure NAT/PAT** for outbound Internet and inbound web/email.
7. **Implement ACLs** on router subinterfaces for inter‑VLAN security and guest isolation.
8. **Deploy servers** (DCs, DHCP, WEB, MAIL, DB, MGMT) and join them to the domain.
9. **Configure DHCP scopes**, DNS zones, and AD structure.
10. **Set up wireless APs** with SSIDs mapped to VLAN 20 and 40.
11. **Harden network devices and endpoints**, enable monitoring and logging.
12. **Test end‑to‑end connectivity**, failover, and security policies.

---

## 14. Rubric Alignment Summary

- **Topology and IP Design:**
  - Multiple VLANs, subnets, and router‑on‑a‑stick inter‑VLAN routing.
- **Redundancy:**
  - Dual routers, HSRP, EtherChannel, STP redundancy, backup routes.
- **Services:**
  - AD, DNS, DHCP, email, public web, database backend, file services, and monitoring.
- **Security:**
  - VLAN segmentation, ACLs, guest isolation, secure management, device and endpoint hardening.
- **Professionalism:**
  - Clear documentation, consistent addressing and naming, realistic enterprise architecture.
