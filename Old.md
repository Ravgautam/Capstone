# Enterprise Network Design for "FitLife Gyms" – NSA630 Capstone Architecture

## Executive Overview

This report proposes a CCNA‑level, enterprise‑style network and services architecture for a fictional gym organization, "FitLife Gyms," tailored to the NSA630 capstone environment and rubric. The design uses limited Cisco routers/switches, virtualized servers, and at least two public IPs from the MITT Capstone network to build a secure, redundant, and scalable demo environment.[^1]

The solution delivers segmented wired and wireless access for staff, IT, servers, and guest Wi‑Fi, internal enterprise services (Active Directory, DNS, DHCP, file shares), and external services (web and email with custom domain), all with end‑to‑end connectivity, NAT to the Internet, and security hardening aligned with the detailed implementation rubric.[^1]

## Scenario and Constraints

The project is implemented in the NSA630 Capstone lab, where physical Cisco routers and switches connect to lab computers running Type 1 hypervisors (ESXi/Hyper‑V) via "Lab" bridged adapters. These Cisco devices uplink to the MITT "Capstone" network 10.128.250.0/24, which statically NATs selected internal IPs to at least two public IP addresses reserved for each group.[^1]

The client, FitLife Gyms, is a growing fitness business with one flagship location in the demo and planned expansion to additional branches in the future. IT requirements mirror the example scenarios in the rubric: Internet access, a directory service, user accounts, internal email, a public website, network and system redundancy, security, and at least four logical networks/user groups: Guest Wi‑Fi, Staff (trainers and front‑desk), IT/Admin, and Servers.[^1]

## Logical Topology Overview

### High‑Level Components

The design follows a collapsed core model appropriate for limited lab devices while conceptually matching enterprise core/distribution/access layers. The main components are:[^1]

- Edge Router Pair (R1, R2) – dual WAN routers connected to the Capstone network, running HSRP and performing NAT/PAT.
- Core/Distribution Switch Pair (SW1, SW2) – multilayer switches providing inter‑VLAN routing, FHRP default gateways, and EtherChannel uplinks.
- Access Switch (SW3) – access‑layer switch for wired staff, IT, and AP uplinks.
- Wireless Access Points (AP1, AP2) – dual SSID (Staff and Guest) mapped to corresponding VLANs.
- Virtualization Hosts (HV1, HV2) – Type 1 hypervisors hosting all Windows and Linux servers.[^1]
- Servers (VMs):
  - DC1, DC2 – Domain Controllers with AD DS, DNS, and DFS/file shares.
  - DHCP1, DHCP2 – DHCP servers (can also be roles on DC1/DC2 in a smaller build).
  - MAIL1 – SMTP/IMAP email server (e.g., hMailServer or Postfix/Dovecot) for custom domain.
  - WEB1 – public web server (IIS/Apache/Nginx) for www.fitlife‑gym.ca.
  - DB1 – backend database server (MySQL/MariaDB/SQL Server Express) for web application.
  - MGMT1 – monitoring/vulnerability scanner (e.g., Zabbix/LibreNMS + OpenVAS/Nessus Essentials).

### Textual Logical Topology Diagram

```text
                Internet / MITT Capstone Public IPs
                             |
                      Capstone Network
                        10.128.250.0/24
                             |
                      [Edge Routers]
                      R1 -------- R2
                       |  HSRP   |
                       +---------+
                             |
                     Routed Link (e.g. /30)
                             |
                 +---------------------------+
                 |  Core / Distribution      |
                 |  SW1 ======= SW2          |
                 |   ^  EtherChannel  ^      |
                 +---|----------------|------+
                     |                |
                     | (Trunks with multiple VLANs)
                     |
                 [Access Switch SW3]
           _________|___________
          |         |          |
      Staff PCs   IT PCs    AP1/AP2
      (VLAN 20)  (VLAN 30)  (Trunk: VLAN 20,40)

      [Virtualization Hosts HV1, HV2]
            |               |
         Trunk links (VLAN 10, 15, 30, 50, 60)
            |               |
     +------+-----+   +-----+------+
     |  Servers   |   |   Servers   |
     |  DC1/DC2   |   | DHCP, WEB, |
     |  MAIL, DB  |   | MGMT       |
     +------------+   +------------+

Guest Wi‑Fi users connect via SSID "FitLife‑Guest" mapped to VLAN 40,
which is ACL‑restricted to Internet access only through R1/R2.
```

This logical topology supports segmented traffic paths, central routing on the core switches, and redundancy via dual routers, dual core switches, and dual hypervisors.

## IP Addressing and VLAN Plan

FitLife Gyms uses a private IPv4 block 10.10.0.0/16 internally, subdivided into /24 networks per VLAN for clarity and ease of demonstration. The design can be scaled later by summarizing these /24s into larger supernets at the distribution or core layers.

### VLAN and Subnet Table

| VLAN ID | Name            | Purpose                         | Subnet           | Gateway IP     |
|--------:|-----------------|---------------------------------|------------------|----------------|
| 10      | SERVER          | Windows/Linux servers, AD, DB   | 10.10.10.0/24    | 10.10.10.1     |
| 15      | DMZ             | Public‑facing WEB/MAIL servers  | 10.10.15.0/24    | 10.10.15.1     |
| 20      | STAFF           | Trainers, front‑desk, finance   | 10.10.20.0/24    | 10.10.20.1     |
| 30      | IT_ADMIN        | IT staff, management tools      | 10.10.30.0/24    | 10.10.30.1     |
| 40      | GUEST_WIFI      | Member/guest wireless Internet  | 10.10.40.0/24    | 10.10.40.1     |
| 50      | MGMT            | Out‑of‑band / device mgmt       | 10.10.50.0/24    | 10.10.50.1     |
| 60      | VOIP (optional) | Future IP phones                | 10.10.60.0/24    | 10.10.60.1     |

Default gateways for each VLAN are configured as SVI interfaces on SW1/SW2 with FHRP (e.g., HSRP), using virtual IPs (e.g., 10.10.x.254) if desired to clearly distinguish them from physical switch IPs.

### DHCP Scope Plan

- VLAN 10 – servers use static IPs outside DHCP range (e.g., 10.10.10.10–10.10.10.100), no DHCP scope.
- VLAN 15 – DMZ servers use static IPs (e.g., 10.10.15.10–10.10.15.50), no general DHCP scope.
- VLAN 20 – DHCP scope 10.10.20.100–10.10.20.200 for staff clients.
- VLAN 30 – DHCP scope 10.10.30.100–10.10.30.200 for IT/admin clients.
- VLAN 40 – DHCP scope 10.10.40.50–10.10.40.230 for guest devices.
- VLAN 50 – static management addresses for network devices, hypervisors, and MGMT1.

DHCP relay (ip helper‑address) is configured on SVIs pointing to DHCP1 and DHCP2 so that a single HA DHCP infrastructure can serve all client VLANs.[^1]

## Network Design: Core, Distribution, Access

### Layering Strategy

Because device count is limited, the design uses a collapsed core where SW1 and SW2 jointly provide core and distribution functions, with SW3 serving as the access layer. This matches enterprise design principles while remaining realistic for the capstone lab.[^1]

- Edge Layer – R1/R2 provide WAN access, NAT, and act as the default route for the campus.
- Core/Distribution – SW1/SW2 handle inter‑VLAN routing, FHRP default gateways, routing to/from R1/R2, and EtherChannel aggregation.
- Access – SW3 connects wired endpoints and APs, with VLANs extended from the core and port‑security features applied.

### Routing Design

- Inter‑VLAN routing: done on SW1/SW2 using SVIs per VLAN with HSRP for gateway redundancy.
- Routing protocol: OSPF single‑area design between R1, R2, SW1, and SW2, advertising:
  - Internal VLAN subnets from SW1/SW2.
  - Default route from R1/R2 into the campus.
- Backup static/default routes:
  - R1 and R2 each have a static default route to the Capstone network next‑hop; if OSPF fails, the static route ensures basic Internet reachability.

This satisfies the rubric’s requirements for backup routes and practical routing design.[^1]

### Switching Design

- SW1 and SW2 interconnected via EtherChannel (LACP) to provide link redundancy and higher throughput.
- SW3 uplinks to both SW1 and SW2 with two separate EtherChannels or dual uplinks (depending on available interfaces), forming a triangle topology without Layer 2 loops thanks to spanning tree tuning.
- Access ports assigned to appropriate VLANs (e.g., VLAN 20 for staff, VLAN 30 for IT) with PortFast and BPDU Guard.

## Segmentation and Security Strategy

### VLAN and Subnet Segmentation

Segmentation follows the rubric’s requirement for Layer 2 (VLAN) and Layer 3 (subnetting) separation.[^1]

- SERVER VLAN 10 – all internal AD, DNS, file, and core infrastructure servers.
- DMZ VLAN 15 – public‑facing web and email servers, separated by ACLs and firewalls from internal servers.
- STAFF VLAN 20 – trainer and admin client devices.
- IT_ADMIN VLAN 30 – IT operations workstations with elevated access for management.
- GUEST_WIFI VLAN 40 – Internet‑only access for gym members.
- MGMT VLAN 50 – dedicated management IPs for routers, switches, hypervisors, and monitoring server.

### Inter‑VLAN ACL Policy

Security is implemented using extended ACLs applied inbound and outbound on SVIs and router interfaces.

Example policy:

- Guest VLAN (40)
  - Permit: outbound HTTP/HTTPS and DNS to the Internet.
  - Deny: any access to internal server, staff, IT, and management subnets.
- Staff VLAN (20)
  - Permit: access to AD, DNS, DHCP, file shares, internal web apps.
  - Deny: direct access to DMZ management interfaces, MGMT VLAN, and network devices.
- IT_ADMIN VLAN (30)
  - Permit: SSH/HTTPS to routers, switches, servers, hypervisors, and MGMT1.
  - Permit: RDP/WinRM to servers.
- DMZ VLAN (15)
  - Permit: inbound traffic from Internet only to published services (80/443 to WEB1, 25/587/993 to MAIL1 via NAT).
  - Deny: DMZ‑initiated connections into SERVER or user VLANs except strictly required (e.g., DB1 access from WEB1 to VLAN 10 on 3306/TCP).

Standard ACLs may also be used for management plane protection (e.g., restricting VTY access).

### Wireless Segmentation

- SSID "FitLife‑Staff" – mapped to STAFF VLAN 20, using WPA2/WPA3‑Enterprise with 802.1X against RADIUS (NPS on DC1) for user‑based authentication.
- SSID "FitLife‑Guest" – mapped to VLAN 40, using WPA2‑PSK or open with captive portal; ACLs on WLAN controller or AP sub‑interfaces mirror guest restrictions.

This ensures that wireless clients inherit the same segmentation and ACL policies as wired devices.

## Redundancy and High Availability

The rubric emphasizes Layer 2/3 redundancy, EtherChannel, FHRP, and backup routes, along with server and services redundancy. FitLife Gyms’ design incorporates redundancy at multiple layers.[^1]

### Network Redundancy

- Edge Routers – R1 and R2 connected to the Capstone network, forming an HSRP pair providing a virtual default gateway IP for SW1/SW2.
- FHRP on SVIs – SW1 and SW2 run HSRP for each VLAN, providing:
  - Virtual IP as default gateway for clients (e.g., 10.10.20.254).
  - Preemption and tracking of uplink interfaces to favor the switch with the active uplink to R1.
- EtherChannel –
  - SW1–SW2 inter‑switch links bundled into an EtherChannel to provide link redundancy and increased bandwidth.
  - SW3 uplinks to both SW1 and SW2 using EtherChannel or dual links with spanning tree.
- Backup Routes –
  - OSPF used as primary; static backup default route on SW1/SW2 pointing to the HSRP virtual IP on R1/R2 or directly toward the Capstone network.

### Server and Service Redundancy

- Virtualization – two hypervisors (HV1, HV2) hosting redundant VMs; critical services are deployed in pairs.
- Active Directory – DC1 and DC2 both host AD DS and internal DNS zones; if one fails, authentication and name resolution continue.[^1]
- DHCP – DHCP1 (primary) and DHCP2 (secondary/standby or failover partner) share scopes; exclusion ranges prevent IP conflicts.[^1]
- Web – optional WEB2 VM on HV2 with standby or round‑robin DNS for www.fitlife‑gym.ca.
- Email – MAIL1 with a secondary MX record pointing to MAIL2 (optional) or to an external smart host if available.
- Monitoring – MGMT1 watches network and services and can trigger alerts; this is not strictly redundant but can be quickly rebuilt from templates.

This design satisfies the rubric’s availability and redundancy sections for both network and server infrastructure.[^1]

## Server and Services Architecture

### Active Directory and Identity

- AD Forest/Domain: fitlife‑gym.local.
- Domain Controllers: DC1 and DC2 (Windows Server), both running:
  - AD DS.
  - Integrated DNS.
  - SYSVOL/NETLOGON for logon scripts.

AD structure:

- OUs: HQ‑Staff, Trainers, IT, Servers, Workstations, Guest‑Accounts (for temporary member accounts if needed).
- Groups: role‑based security groups (e.g., IT‑Admins, FrontDesk‑Staff, Trainers, Finance) mapped to file share and GPO permissions.
- GPOs:
  - Password and lockout policies.
  - Baseline security settings (firewall, SMB signing, RDP restrictions).
  - Drive mapping for shared folders (e.g., \fileserver\StaffShare, \fileserver\ITShare).
  - Desktop hardening (disable local admin tools for standard users).

AD satisfies the directory service requirements in the rubric, including users, groups, GPOs, and ACLs for network shares.[^1]

### DNS (Internal and External)

- Internal DNS:
  - Hosted on DC1/DC2 for zone fitlife‑gym.local.
  - Contains A records for all servers (DCs, WEB, MAIL, DB, MGMT) and important aliases (CNAMEs) for roles.
  - Reverse lookup zones for key subnets.
- External DNS:
  - Hosted by a public DNS provider or simulated via Linux BIND VM (DNS‑EXT) reachable via public IP.
  - Public zone fitlife‑gym.ca with records:
    - A: www.fitlife‑gym.ca -> public IP #1 (NAT to WEB1/WEB2 in DMZ).
    - MX: mail.fitlife‑gym.ca -> public IP #2 (NAT to MAIL1).

This covers both global and local DNS requirements.[^1]

### DHCP

- DHCP1 – primary DHCP server for VLANs 20, 30, 40.
- DHCP2 – secondary DHCP server configured with failover or split scopes.

Scopes include:

- Correct subnet mask, default gateway (HSRP VIP), DNS servers (DC1/DC2), domain name (fitlife‑gym.local), and lease durations aligned with VLAN characteristics (shorter for guest Wi‑Fi).

DHCP relays (ip helper‑address) on SVIs ensure that VLANs do not require separate DHCP servers, satisfying the requirement for centralized DHCP services and scopes.[^1]

### Email System

- MAIL1 – Windows or Linux VM running an SMTP/IMAP solution such as hMailServer (Windows) or Postfix + Dovecot (Linux).
- Domain: fitlife‑gym.ca for email addresses (e.g., user@fitlife‑gym.ca).
- Functions:
  - Internal send/receive between staff members.
  - External send/receive via public IP #2 and MX record.
  - TLS for secure SMTP (STARTTLS) and IMAPS.

ACLs and DMZ placement restrict inbound ports to 25/587/993 via NAT and prevent direct access to internal subnets from MAIL1 except for AD‑integrated authentication (e.g., via LDAP over TLS) if implemented.

### Web Server and Backend

- WEB1 – Front‑end web server hosting:
  - Public gym site (class schedules, membership info).
  - Optional secure member portal.
- DB1 – Backend database in SERVER VLAN 10, reachable only from WEB1 via restricted ports (e.g., 3306/TCP).
- Optional WEB2 – secondary web server for HA.

The web application design satisfies front‑end and database backend requirements in the rubric.[^1]

### File Services and Shares

- DFS or simple file server role on DC1/DC2 for:
  - Staff shared drive with ACLs by department.
  - IT admin repository with scripts and documentation.
  - Public read‑only share for common documents.

Permissions are group‑based, and ACLs are applied at both share and NTFS levels.

### Virtualization Architecture

- Type 1 hypervisor (HV1, HV2) on lab machines as required by rubric.[^1]
- VM placement balanced between hosts (e.g., DC1, DHCP1, WEB1, DB1 on HV1; DC2, DHCP2, MAIL1, MGMT1 on HV2).
- Virtual switches and port groups aligned with VLAN IDs and mapped to trunk uplinks from SW1/SW2.

## Internet Access and NAT Design

### Capstone Network Integration

- R1 and R2 each have an interface in 10.128.250.0/24 (Capstone network) connected via the rack’s T port.[^1]
- MITT performs static NAT from this Capstone subnet to the group’s assigned public IP addresses.[^1]
- The design assumes at least two public IPs:
  - Public IP #1 – general PAT for outbound Internet traffic plus inbound HTTPS to web.
  - Public IP #2 – MX record target for inbound email (SMTP/IMAP).

### NAT on Edge Routers

On R1/R2:

- Dynamic PAT:
  - Overload internal private addresses (10.10.0.0/16) to Public IP #1 for outbound web browsing and updates.
- Static NAT / Port Forwarding:
  - WEB1: map Public IP #1:443 (and optionally 80) to 10.10.15.x (WEB1) in DMZ VLAN 15.
  - MAIL1: map Public IP #2:25/587/993 to 10.10.15.y (MAIL1) in DMZ VLAN 15.

Router ACLs control which internal addresses can be NATed and which inbound ports are allowed, forming a simple edge firewall function.

### Default Routing and Failover

- SW1/SW2 default route points to HSRP VIP on R1/R2.
- Capstone network provides upstream route to the Internet; if R1 fails, HSRP failover to R2 ensures continuity.

This end‑to‑end design ensures internal hosts can reach external services and that external users can reach only the intended public services.

## Security Hardening Measures

The rubric specifically calls for secure protocols, network device hardening, end‑device hardening, and monitoring/vulnerability scanning. The design includes practical measures at each layer.[^1]

### Secure Protocols

- Administrative access uses SSHv2 and HTTPS; Telnet and HTTP management are disabled.
- SNMPv3 is used where possible; SNMPv1/v2c are disabled or restricted.
- RADIUS (NPS) or TACACS+ (optional) for centralized device authentication for routers and switches.
- TLS certificates on WEB1 and MAIL1 to encrypt HTTPS and email protocols.

### Switch and Router Hardening

- Disable unused ports and place them in a black‑hole VLAN.
- Enable DHCP Snooping, Dynamic ARP Inspection, and IP Source Guard on access ports for user VLANs.
- Configure BPDU Guard and PortFast on edge ports to protect against STP manipulation.
- Disable unnecessary services (CDP on external‑facing interfaces, HTTP server, unused routing protocols).
- Use strong local admin credentials stored securely per the rubric’s password management requirement.[^1]

### End‑Device Hardening

- Windows Defender or equivalent anti‑malware enabled on all Windows endpoints.
- Local firewalls enabled with rules enforcing least privilege (allow only required outbound ports; block inbound except management from IT VLAN).
- Automatic OS and application updates via Windows Update or package managers in Linux.
- Standard user accounts for day‑to‑day operations; no direct domain admin logons to workstations.

### Monitoring and Vulnerability Scanning

- MGMT1 runs:
  - Syslog or log collection for routers, switches, and servers.
  - NMS (e.g., Zabbix/LibreNMS) to monitor interface status, CPU, memory, and key services.
  - Vulnerability scanner (e.g., OpenVAS/Nessus Essentials) to periodically scan internal and DMZ hosts.
- Regular Nmap scans to verify exposed ports and confirm ACL effectiveness.

These measures directly address the security hardening and monitoring requirements in the rubric.[^1]

## Optional Enhancements for Bonus Marks

To gain additional/exceptional work points, the design can be extended with features that remain within CCNA scope but demonstrate initiative and creativity.[^1]

Possible enhancements:

- Site‑to‑Site VPN Simulation – add a second "branch gym" router and small LAN, connected via an IPSec tunnel over the Capstone network, demonstrating secure inter‑site communication.
- Remote Access VPN – configure SSL or IPsec remote access VPN for IT admins to manage the network from offsite lab machines.
- Captive Portal for Guest Wi‑Fi – use a lightweight captive portal on WEB1 that forces guests to accept terms of service before gaining Internet access.
- Syslog and SIEM‑like Dashboard – aggregate logs from Cisco devices and servers into a central log server with dashboards showing security events.
- QoS for Critical Services – prioritize VoIP (VLAN 60) and management traffic to show awareness of quality of service.
- Configuration Management – maintain running‑config backups (e.g., via scripts) and document change management as part of professionalism and documentation.

These enhancements build on the core design without introducing tools or complexity beyond what is expected at CCNA and capstone level.

## Mapping to NSA630 Rubric

The proposed architecture is explicitly aligned with the detailed implementation rubric.[^1]

- Documentation – logical and physical topologies, addressing tables, and server/service configs are clearly defined and can be turned into diagrams and configuration documents.[^1]
- Networking – practical network topology, IP addressing, WLANs, end‑to‑end connectivity, VLANs/subnets, ACLs, EtherChannel, FHRP, and backup routes are all present.[^1]
- Server & Services – Type 1 hypervisor, email with custom domain, AD (users, groups, GPOs, network shares, file ACLs), DNS (global and local), DHCP scopes, web front‑end plus database backend, and server/service redundancy are implemented.[^1]
- Security Hardening – secure protocols, switch and router best practices, endpoint hardening, and monitoring/vulnerability scanning are integrated into the design.[^1]
- Additional Work – optional enhancements offer clear paths to exceed core requirements and demonstrate professional‑grade enterprise design thinking.[^1]

This design therefore forms a strong blueprint for a capstone implementation that is both realistic and achievable in the provided lab environment.

---

## References

1. [NSA630-Capstone-Project-1.pdf](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/95504348/f37539eb-aa49-4068-b1e3-f550a3c9febd/NSA630-Capstone-Project-1.pdf?AWSAccessKeyId=ASIA2F3EMEYE3S6YLFYA&Signature=UNkwaiWg50IIw8miJyt6SHiJdQM%3D&x-amz-security-token=IQoJb3JpZ2luX2VjECAaCXVzLWVhc3QtMSJHMEUCIESpOZkPJ1M2HAIm5LYht2tdTenRED3CK%2F%2F3LR6ofTCFAiEAgcJZYOEKw8gWJyfOGGpNiCCOr9jYVVyMmGKKIwH1QX0q%2FAQI6f%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARABGgw2OTk3NTMzMDk3MDUiDJWPE4fvU7gZdOmPoirQBDNvPtXWoTf4fuQD%2Bug35eQ2lOlkTgZzmK0qn4Hn%2B2MuaKdo81UE4%2BvaMIJeSkYL7uaNLS%2B3reaCyYhmqLREm%2BHZlhR9dPhvNLlX3fV1HZ69q%2B5rzN4My3jHCSS%2FcWHAjAeLHJcPLbSoz1eo4TQivISqww1Q8ew3c4Aq6oJPk%2BEe0gMw1ZZ9FuUyjeZb5HMhuRVKo9I1Uk66Qk8TCqO%2B8vhHInNuxSRB7KEHY5oBMV7hjV%2BMFH1rXOqO3FVqnThMMR08Wp1qtA9A6i7JSt6FKD9sFTG4EyXZR9vL9gIM8TF4Ud%2B7uUpGWQANlu%2Bqf8ABx%2FzN0108e9ygiT%2FTptY7ZzQCOqu6zl13DuCszjd2UiQgwI497N5dw3Zkr2wnBRDZr2BlrI6CMjqsVd8YSx9l%2FfFEbj1d6oA61X%2BPzCRXgSnmrQ9sF7vAt3P12ZdFbIEwqUtpDlI%2Flan3wrkq0e0jUOvH0IqvwHHpiMPuuwihy1p9YKbBNSEUlRuyeG3fLDkuulK9J3yWai1rhujHptFKY0ZoNj75%2FtpCy7x%2B7MxWLNbIB%2FIpitRTwF0HUeusXbhldEx1JNYQDE5IFTRHqkV34P3ddZdTFBhtK%2FlGyOz3Aj0r%2BzdUeKZMIpha%2B9TQWS8EaKQ%2BH6Rxrm5CyD6c0FEtUW4GqXIIyqzKmUwVcvfOLCJoTeyoqQNXueAfcRTVxwmece3ufQSdz0QHAF6aaYAGF0McVU2DvH8No9%2BovH1vQXbQMvpvvSMON7ncbvPSXGSin3dpFG9t33F935qgcaTVklowl8zUzgY6mAHRKhUT7pSrZI7VYFFgqYgWm7J57uQcV95P4aOGFYLkVlJGfkz2kjQ6u1FLcobrSL2pF1AJieEAYmqjSQevIlLqc8RDkj8hq6jC%2F1JvcEFA47AfTUfl5qmglkzL4qRT2Hw9Yy9OyYTj4%2Bqyj0WQQj8ht%2F9v8%2FQLscHrZoIaITK77MctIR%2F0ORJ4ZBaN%2BAf0gVEtYVvzB7WZ6g%3D%3D&Expires=1775580138) - Page 1 of 5 
 
NSA630 – NSA Capstone Project 
(December 5, 2025) 
Introduction  
    The capstone pr...

