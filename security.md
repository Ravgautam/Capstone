Yes—this project is expected to have explicit *hardening* on routers, switches, and endpoints.  
Below is a set of configurations that will easily meet the rubric’s “Level 5 / All” requirements for:

- Switches hardening (≥8 items)
- Routers hardening (≥6 items)
- Securing protocols
- End‑device hardening
- Monitor and scan

Use/adjust interface numbers to match your Packet Tracer file.

***
## 1. Switch hardening (S‑A, S‑B, S‑C)
These are standard security best practices for access and distribution switches: port security, DHCP snooping, IP Source Guard, storm control, STP protection, and secure management. [reddit](https://www.reddit.com/r/networking/comments/3b2kzh/best_practice_for_end_user_facing_ports_on_cisco/)
### 1.1 Global switch settings
```plaintext
! Secure management
hostname SA             ! SB / SC on other switches
no ip domain-lookup
ip domain-name fitlife-gym.local
!
enable secret STRONG_ENABLE_PASSWORD
!
username netadmin privilege 15 secret STRONG_SSH_PASSWORD
!
crypto key generate rsa modulus 2048
ip ssh version 2
!
line vty 0 4
 transport input ssh
 login local
 exec-timeout 5 0
 logging synchronous
!
! Only allow management from IT_ADMIN (10.10.30.0/24) and MGMT (10.10.50.0/24)
ip access-list standard MGMT-VTY
 permit 10.10.30.0 0.0.0.255
 permit 10.10.50.0 0.0.0.255
line vty 0 4
 access-class MGMT-VTY in

! Disable unused services
no ip http server
no ip http secure-server
cdp run              ! optional in lab, but you can 'no cdp run' on access switches
service password-encryption

! Logging + NTP
logging 10.10.50.10          ! MGMT1 syslog server
service timestamps log datetime msec
ntp server 10.10.50.10       ! if MGMT1 runs NTP
```
### 1.2 VLAN 50 management interface
```plaintext
vlan 50
 name MGMT

interface vlan 50
 ip address 10.10.50.11 255.255.255.0   ! change last octet per switch
 no shutdown

ip default-gateway 10.10.50.254         ! HSRP VIP on routers
```
### 1.3 Secure access ports (Staff / IT / Servers)
Use this template on **all edge ports** (PCs, servers, printers, AP trunk is handled later):

```plaintext
interface range Fa0/10 - 14     ! example access ports
 description USER/ENDPOINT PORTS
 switchport mode access
 switchport access vlan 20       ! or 30 or 10 depending on the device
 spanning-tree portfast
 spanning-tree bpduguard enable
!
! Port security
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security aging time 5
 switchport port-security aging type inactivity
!
! Storm control
 storm-control broadcast level 5.00
 storm-control multicast level 5.00
 storm-control action shutdown
```

- Port security + BPDU Guard + storm control on user ports are widely recommended to prevent rogue devices, STP attacks and layer‑2 storms. [reddit](https://www.reddit.com/r/networking/comments/3b2kzh/best_practice_for_end_user_facing_ports_on_cisco/)
### 1.4 DHCP Snooping + IP Source Guard
```plaintext
ip dhcp snooping
ip dhcp snooping vlan 20,30,40
!
! Trust only uplinks toward routers / distribution switches
interface Fa0/1
 description Uplink to S-A or router
 ip dhcp snooping trust
!
interface range Fa0/10 - 14
 ip dhcp snooping limit rate 15
 ip verify source            ! IP Source Guard
```

- DHCP snooping + IP Source Guard prevent rogue DHCP and IP spoofing on access ports. [reddit](https://www.reddit.com/r/networking/comments/3b2kzh/best_practice_for_end_user_facing_ports_on_cisco/)
### 1.5 Trunks and STP protection
On switch–switch and switch–router trunks:

```plaintext
interface Fa0/1
 description Trunk to S-A / R-A
 switchport mode trunk
 switchport trunk allowed vlan 10,15,20,30,40,50,60
 spanning-tree portfast trunk
!
! If you want to prevent unauthorized switches from becoming root:
spanning-tree mode rapid-pvst
spanning-tree vlan 1,10,15,20,30,40,50,60 root primary
spanning-tree portfast default
spanning-tree bpduguard default
```

- BPDU Guard and careful STP root configuration are core switch‑hardening measures. [sclabs.blogspot](http://sclabs.blogspot.com/2014/10/ccnp-switch-securing-switched-networks.html)

These give you **more than 8** distinct hardening items for each switch: secure management, SSH only, VTY ACL, port security, BPDU Guard, PortFast, storm control, DHCP snooping, IP Source Guard, STP hardening, logging, NTP.

***
## 2. Router hardening (R‑A, R‑B)
Focus on management security, disabling unused services, protecting control plane, securing HSRP, and using ACLs. [cisco](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_cfg/configuration/xe-16/sec-usr-cfg-xe-16-book/sec-login-enhance.html)
### 2.1 Global security settings
```plaintext
hostname RA
no ip domain-lookup
ip domain-name fitlife-gym.local

enable secret STRONG_ENABLE_PASSWORD

username netadmin privilege 15 secret STRONG_SSH_PASSWORD

service password-encryption
service tcp-keepalives-in
service tcp-keepalives-out

crypto key generate rsa modulus 2048
ip ssh version 2

! login enhancements against brute force
login block-for 60 attempts 5 within 60
login on-failure log
login on-success log

! Disable legacy services
no ip http server
no ip http secure-server
no service pad
no cdp run           ! or at least on WAN interface

! Logging + NTP
logging 10.10.50.10
service timestamps log datetime msec
ntp server 10.10.50.10
```

- Cisco “login block” and SSH‑only remote access are recommended to mitigate brute‑force attacks. [cisco](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_cfg/configuration/xe-16/sec-usr-cfg-xe-16-book/sec-login-enhance.html)
### 2.2 Secure VTY lines
```plaintext
ip access-list standard MGMT-VTY
 permit 10.10.30.0 0.0.0.255
 permit 10.10.50.0 0.0.0.255

line vty 0 4
 transport input ssh
 login local
 access-class MGMT-VTY in
 exec-timeout 5 0
 logging synchronous
```

- Limiting VTY access to management subnets is a common router‑hardening guideline. [linkedin](https://www.linkedin.com/pulse/cisco-router-security-best-practices-bharatcxo-2n0bf)
### 2.3 Interface‑level protections
On **LAN subinterfaces** (example for VLAN 20):

```plaintext
interface GigabitEthernet0/0.20
 description STAFF
 encapsulation dot1q 20
 ip address 10.10.20.1 255.255.255.0
 standby 20 ip 10.10.20.254
 standby 20 priority 110
 standby 20 preempt
 standby 20 authentication md5 FITLIFE-HSRP-KEY
 no ip redirects
 no ip proxy-arp
```

- Disabling ICMP redirects and proxy ARP, and authenticating HSRP, are recommended to reduce spoofing and redirection attacks. [linkedin](https://www.linkedin.com/pulse/cisco-router-security-best-practices-bharatcxo-2n0bf)

On **WAN interface** to the Capstone/Cloud:

```plaintext
interface GigabitEthernet0/1
 description Upstream to Capstone
 ip address 10.128.250.10 255.255.255.0
 no ip redirects
 no ip proxy-arp
 no cdp enable
```
### 2.4 Control-plane ACLs / management
Optionally:

```plaintext
! Restrict SNMP to MGMT server only, if you use SNMPv2
access-list 20 permit 10.10.50.10
snmp-server community STRONGSTRING RO 20
```

Or better, configure **SNMPv3** with authentication and privacy if supported; SNMPv3 is recommended over v1/v2 for secure monitoring. [globalknowledge](https://www.globalknowledge.com/us-en/resources/resource-library/articles/how-to-secure-cisco-routers-and-switches/)

Altogether you have **6+** router‑hardening controls: SSH‑only, VTY ACL, login block, disable services, HSRP auth, no redirects/proxy‑arp, control‑plane / SNMP protection, logging and NTP.

***
## 3. Securing protocols
To satisfy the “securing protocols” rubric line, ensure:

- **Management protocols**  
  - Only SSH on routers and switches (done above).  
  - Disable Telnet, HTTP; if GUI is required, use HTTPS with strong credentials. [linkedin](https://www.linkedin.com/pulse/cisco-router-security-best-practices-bharatcxo-2n0bf)
- **Routing/HSRP**  
  - Add `standby X authentication md5 KEY` on each HSRP group.  
- **SNMP**  
  - Prefer SNMPv3 with auth+priv; if lab doesn’t support it, at least limit SNMPv2 communities to MGMT1 via ACL. [globalknowledge](https://www.globalknowledge.com/us-en/resources/resource-library/articles/how-to-secure-cisco-routers-and-switches/)
- **Wireless AP**  
  - Staff SSID: WPA2‑Enterprise or at least WPA2‑PSK with strong passphrase.  
  - Guest SSID: WPA2‑PSK or portal; ACLs on router for Internet‑only access (you already planned this).  

***
## 4. End‑device hardening (servers and PCs)
Use Windows security baselines and Defender / firewall policies via Group Policy; this is in line with Microsoft’s recommended baselines for Windows Server and Windows 10/11. [learn.microsoft](https://learn.microsoft.com/en-us/defender-endpoint/use-group-policy-microsoft-defender-antivirus)

Key items (describe these in your report as “implemented via GPO / local policy”):

- **OS and patching**
  - Enable automatic updates on all servers and clients.
  - Remove/disable unnecessary roles and services on servers (e.g., no web server on DCs).

- **Accounts and authentication**
  - Domain password and lockout policy (complex passwords, history, lockout after a few failed attempts).
  - Disable local Administrator or randomize its password using LAPS.
  - Use least‑privilege accounts for services (WEB1/DB1/MAIL1 service accounts).

- **Host firewall and antivirus**
  - Windows Defender (or other AV) enabled and centrally managed; regular scans scheduled. [argonsys](https://argonsys.com/microsoft-cloud/library/security-baseline-final-for-windows-10-v1809-and-windows-server-2019/)
  - Windows Defender Firewall enabled for all profiles; block inbound by default except required ports (RDP to IT_ADMIN, web/SMTP on DMZ, etc.). [ravenswoodtechnology](https://www.ravenswoodtechnology.com/windows-security-baselines-best-practices/)

- **Hardening templates**
  - Apply Microsoft Security Compliance Toolkit baseline GPOs to servers and domain controllers (e.g., Windows Server 2019 baseline). [learn.microsoft](https://learn.microsoft.com/en-us/windows/security/operating-system-security/device-management/windows-security-configuration-framework/windows-security-baselines)

- **Other**
  - BitLocker or disk encryption (where supported) on laptops / critical servers.  
  - Disable USB mass‑storage via GPO if required.  
  - Screen‑lock timeout for users.

This gives you several end‑device hardening controls that clearly meet “All” in the rubric row.

***
## 5. Monitor and scan
To get full marks on “Monitor and Scan”, configure MGMT1 to collect logs and perform regular scans. [ravenswoodtechnology](https://www.ravenswoodtechnology.com/windows-security-baselines-best-practices/)

- **Syslog**
  - All routers and switches already send `logging 10.10.50.10`; configure MGMT1 (e.g., Syslog‑NG, Kiwi, or Linux syslog) to receive and store logs.  
- **Time synchronization**
  - MGMT1 runs NTP; all network devices and servers sync time to it so logs line up.  
- **SNMP monitoring**
  - Enable SNMP (preferably v3) on routers and switches and add them to a monitoring tool on MGMT1 (LibreNMS, Zabbix, etc.). [globalknowledge](https://www.globalknowledge.com/us-en/resources/resource-library/articles/how-to-secure-cisco-routers-and-switches/)
- **Vulnerability and port scanning**
  - Run periodic OpenVAS/Nessus scans from MGMT1 against:
    - DMZ servers (WEB1/MAIL1)  
    - Internal servers (DC1/DC2, DB1)  
    - Sample staff/IT endpoints  
  - Use Nmap to verify only expected ports are open per VLAN and that guest VLAN cannot reach internal networks (confirm ACLs).

- **Endpoint security monitoring**
  - Enable Windows Defender AV reporting / alerts to a central console where possible. [learn.microsoft](https://learn.microsoft.com/en-us/defender-endpoint/use-group-policy-microsoft-defender-antivirus)

***
## How to present this for the rubric
In your documentation or presentation, explicitly list the hardening items under headings like:

- “Switch Hardening – 11 Configurations Implemented”
- “Router Hardening – 8 Configurations Implemented”
- “End Device Hardening – Server and Client Baselines”
- “Monitoring and Scanning – Syslog, SNMP, AV, Vulnerability Scans”

Then reference the IOS configs and GPO settings above as evidence.  

If you want, share your current running‑config for one switch and one router, and I can mark up exactly what to add so you can copy‑paste into Packet Tracer.
