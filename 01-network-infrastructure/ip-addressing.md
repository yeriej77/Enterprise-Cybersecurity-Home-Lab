# IP Addressing and Network Services

## Overview

The home lab uses a structured IPv4 addressing plan to keep critical infrastructure predictable while allowing workstation and client systems to receive network configuration dynamically.

The primary trusted network is the **LAB-LAN**, which currently uses:

```text
10.0.0.0/24
```

The environment combines:

- Static IP addressing for infrastructure
- DHCP for client systems
- Active Directory-integrated DNS
- pfSense as the default gateway
- A separate subnet for OpenVPN remote-access clients
- VLAN-based network segmentation

This addressing design makes the environment easier to manage, troubleshoot, secure, and expand.

---

# Primary LAB-LAN Network

The primary lab network uses the following configuration:

| Setting | Value |
|---|---|
| Network | `10.0.0.0/24` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `10.0.0.1` |
| Internal DNS Server | `10.0.0.6` |
| Active Directory Domain | `ad.home.com` |
| NetBIOS Domain | `HOMELAB` |
| VLAN | VLAN 10 — LAB-LAN |

A `/24` network provides addresses from:

```text
10.0.0.0 - 10.0.0.255
```

The normal usable host range is:

```text
10.0.0.1 - 10.0.0.254
```

The network address is:

```text
10.0.0.0
```

and the broadcast address is:

```text
10.0.0.255
```

---

# Core Infrastructure Addressing

Critical infrastructure uses predictable IP addresses.

| Device | Hostname | IP Address | Primary Role | Addressing |
|---|---|---:|---|---|
| pfSense | pfSense | `10.0.0.1` | Router / Firewall / VPN Gateway | Static |
| Cisco Switch | SW1 | `10.0.0.2` | Managed Layer 2 Switch | Static |
| Proxmox | pve | `10.0.0.5` | Virtualization Host | Static |
| Domain Controller | DC1 | `10.0.0.6` | AD / DNS / DHCP | Static |
| Wazuh Server | wazuh01 | `10.0.0.7` | SIEM / Security Monitoring | Static |
| Nessus Server | Nessus | `10.0.0.8` | Vulnerability Scanner | Static |
| Client01 | Client01 | DHCP | Domain-Joined Workstation | Dynamic |
| Jump PC | Jump PC | `10.0.0.101` | Administrative Workstation | Static / Reserved |

The low-numbered addresses are intentionally used for major infrastructure systems so they are easy to recognize during administration and troubleshooting.

---

# Addressing Layout

The LAB-LAN addressing structure can be viewed conceptually as:

```text
10.0.0.0/24
│
├── 10.0.0.1       pfSense
│                   Router / Firewall
│
├── 10.0.0.2       SW1
│                   Cisco Switch
│
├── 10.0.0.5       pve
│                   Proxmox Hypervisor
│
├── 10.0.0.6       DC1
│                   AD / DNS / DHCP
│
├── 10.0.0.7       wazuh01
│                   SIEM
│
├── 10.0.0.8       Nessus
│                   Vulnerability Scanner
│
├── 10.0.0.x       DHCP Clients
│                   Domain Workstations
│
└── 10.0.0.101     Jump PC
                    Administrative Workstation
```

---

# Why Infrastructure Uses Static IP Addresses

Infrastructure services need to remain reachable at predictable addresses.

Systems such as:

- pfSense
- Cisco SW1
- Proxmox
- DC1
- Wazuh
- Nessus

should not unexpectedly change IP addresses after a reboot or DHCP lease renewal.

For example, DC1 provides several critical services at:

```text
10.0.0.6
```

Other systems depend on this address for:

- Active Directory authentication
- DNS
- DHCP
- Group Policy
- Domain controller discovery
- Internal hostname resolution

If DC1 unexpectedly received a different address, multiple services throughout the environment could fail.

The same concept applies to pfSense.

pfSense must remain reachable at:

```text
10.0.0.1
```

because it is the default gateway used by the LAB-LAN.

---

# Default Gateway

The default gateway for systems on the primary network is:

```text
10.0.0.1
```

This address belongs to pfSense.

When a host wants to communicate with a destination outside its local subnet, it forwards the traffic to pfSense.

Example:

```text
Client01
10.0.0.x
    │
    │ Destination is outside
    │ 10.0.0.0/24
    ▼
pfSense
10.0.0.1
    │
    ├── Firewall Policy
    ├── Routing
    ├── NAT
    ├── VLAN Routing
    └── VPN Gateway Selection
```

pfSense then determines how the traffic should be handled.

Depending on the configuration, traffic may:

- Be allowed
- Be blocked
- Be routed to another internal network
- Be translated using NAT
- Exit through the normal ISP connection
- Exit through the Surfshark VPN
- Travel to a remote VPN client network

---

# DNS Architecture

The internal DNS server is:

```text
10.0.0.6
```

This is DC1, the Windows Server domain controller.

The Active Directory DNS domain is:

```text
ad.home.com
```

Domain-joined Windows systems use DC1 for DNS rather than querying public DNS servers directly.

---

## Why Active Directory Depends on DNS

Active Directory relies heavily on DNS for locating domain services.

A domain-joined workstation must be able to find:

- Domain controllers
- LDAP services
- Kerberos services
- Global Catalog services
- Internal hostnames

For example:

```text
dc1.ad.home.com
```

should resolve to:

```text
10.0.0.6
```

Active Directory also creates special DNS service records such as:

```text
_ldap._tcp.dc._msdcs.ad.home.com
```

These records allow domain clients to discover available domain controllers.

---

# DNS Resolution Flow

Internal DNS resolution follows this general path:

```text
Client01
    │
    │ DNS Query
    ▼
DC1
10.0.0.6
    │
    ├── Internal ad.home.com name?
    │
    │        ▼
    │   Resolve Locally
    │
    └── External Internet name?
             │
             ▼
          pfSense
         10.0.0.1
             │
             ▼
      External DNS Resolution
```

DC1 is authoritative for the internal:

```text
ad.home.com
```

DNS zone.

Queries that cannot be resolved internally can be forwarded toward pfSense for external resolution.

---

# Why Clients Should Use DC1 for DNS

Domain-joined systems should use:

```text
10.0.0.6
```

as their DNS server.

They should not normally use public DNS servers such as:

```text
8.8.8.8
```

or:

```text
1.1.1.1
```

as their primary DNS server.

Using a public resolver directly could prevent the workstation from resolving internal Active Directory records.

A healthy domain client should therefore have network settings similar to:

```text
IP Address:      10.0.0.x
Subnet Mask:     255.255.255.0
Default Gateway: 10.0.0.1
DNS Server:      10.0.0.6
DNS Suffix:      ad.home.com
```

---

# DHCP Architecture

DHCP services are provided by DC1.

This allows normal client systems to automatically receive their network configuration rather than requiring every device to be manually configured.

The basic DHCP process is:

```text
Client
   │
   │ DHCP Request
   ▼
DC1
DHCP Server
   │
   ▼
Client Receives
   │
   ├── IP Address
   ├── Subnet Mask
   ├── Default Gateway
   ├── DNS Server
   └── Domain Information
```

A domain workstation may therefore receive settings similar to:

```text
IPv4 Address:    10.0.0.x
Subnet Mask:     255.255.255.0
Default Gateway: 10.0.0.1
DNS Server:      10.0.0.6
```

This provides centralized network configuration for client devices.

---

# Static vs Dynamic Addressing

The environment uses a combination of static and dynamic addressing.

## Static Addressing

Static addressing is used for infrastructure where predictable connectivity is important.

Examples include:

```text
pfSense
SW1
Proxmox
DC1
Wazuh
Nessus
Jump PC
```

Benefits include:

- Predictable firewall rules
- Easier administration
- Easier monitoring
- Easier troubleshooting
- Consistent DNS configuration
- Reliable service dependencies

---

## Dynamic Addressing

DHCP is used for systems where the exact IP address is less important.

Examples include:

```text
Domain workstations
Temporary virtual machines
Test clients
Other endpoint devices
```

Using DHCP reduces the amount of manual network configuration required for client systems.

---

# OpenVPN Addressing

Remote-access VPN clients use a separate IP network:

```text
10.8.0.0/24
```

The OpenVPN server runs on pfSense.

Remote clients receive addresses from this VPN network rather than being placed directly into the LAB-LAN subnet.

The traffic path is:

```text
Remote Laptop
      │
      │ Internet
      ▼
pfSense OpenVPN Server
      │
      ▼
10.8.0.0/24
      │
      ▼
pfSense Firewall / Routing
      │
      ▼
LAB-LAN
10.0.0.0/24
```

This allows remote users to securely access authorized resources inside the home lab.

---

# Why the VPN Uses a Separate Subnet

Using a dedicated VPN subnet provides several benefits.

It allows administrators to:

- Identify traffic coming from remote systems
- Apply specific firewall rules to VPN clients
- Restrict which resources remote users can access
- Troubleshoot remote connectivity separately
- Keep VPN clients outside the normal LAN broadcast domain

Conceptually:

```text
Remote VPN Clients
10.8.0.0/24
      │
      ▼
   pfSense
      │
   Firewall
      │
      ▼
LAB-LAN
10.0.0.0/24
```

The two networks are connected through routing rather than functioning as one flat network.

---

# VLAN Addressing

The environment currently uses the following VLAN design:

| VLAN | Name | Purpose | Status |
|---|---|---|---|
| VLAN 10 | LAB-LAN | Trusted lab infrastructure | Active |
| VLAN 20 | IOT | IoT and smart devices | Active |
| VLAN 99 | Management | Dedicated management network | Planned |
| VLAN 1 | Default | Default / limited use | Existing |

The primary LAB-LAN currently uses:

```text
10.0.0.0/24
```

The exact addressing for future or evolving VLAN networks should be documented based on their actual pfSense interface configuration.

No subnet should be added to project documentation until it has actually been configured and verified.

---

# One VLAN, One Subnet

A routed VLAN should normally have its own unique IP subnet.

Conceptually:

```text
VLAN 10
LAB-LAN
Subnet A
    │
    ▼
 pfSense
    ▲
    │
VLAN 20
IOT
Subnet B
```

The VLAN provides Layer 2 segmentation.

The IP subnet provides Layer 3 separation.

Traffic moving between the networks must pass through pfSense.

This is what makes it possible to enforce firewall rules between VLANs.

---

# Inter-VLAN Traffic

Devices in different VLANs do not communicate directly through the Layer 2 switching environment.

Instead:

```text
Source VLAN
     │
     ▼
Cisco SW1
     │
     ▼
pfSense
     │
 Firewall Rules
     │
     ▼
Destination VLAN
```

pfSense decides whether communication should be permitted.

This provides a major security advantage over placing every device into one flat subnet.

---

# Client Network Verification

Windows systems can display their complete network configuration with:

```powershell
ipconfig /all
```

Important settings to verify include:

```text
IPv4 Address
Subnet Mask
Default Gateway
DHCP Server
DNS Servers
DNS Suffix
```

For a normal domain client on the LAB-LAN, the expected configuration should generally include:

```text
Network:  10.0.0.0/24
Gateway:  10.0.0.1
DNS:      10.0.0.6
Domain:   ad.home.com
```

---

# Testing the Default Gateway

The first Layer 3 connectivity test is normally the local gateway.

From Windows:

```powershell
ping 10.0.0.1
```

A successful reply confirms that the workstation can reach pfSense.

If the gateway cannot be reached, troubleshooting should focus first on:

- Local IP configuration
- Switch port
- VLAN assignment
- Network cable
- Virtual bridge configuration
- pfSense LAN interface

rather than Internet services.

---

# Testing Infrastructure Connectivity

Individual infrastructure systems can also be tested.

### Cisco SW1

```powershell
ping 10.0.0.2
```

### Proxmox

```powershell
ping 10.0.0.5
```

### DC1

```powershell
ping 10.0.0.6
```

### Wazuh

```powershell
ping 10.0.0.7
```

### Nessus

```powershell
ping 10.0.0.8
```

These tests help determine how far communication is successfully traveling through the network.

---

# Testing Internal DNS

Internal DNS can be tested with:

```powershell
nslookup dc1.ad.home.com
```

The responding DNS server should be:

```text
10.0.0.6
```

and the hostname should resolve to the domain controller.

---

# Testing Active Directory DNS Records

Active Directory service records can also be queried.

Example:

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.ad.home.com
```

Successful results indicate that Active Directory DNS records are available to the client.

These SRV records are important because Windows systems use them to locate domain controllers.

---

# Testing External DNS

A useful troubleshooting technique is comparing IP connectivity with DNS connectivity.

For example:

```powershell
ping 1.1.1.1
```

tests basic Internet reachability by IP.

Then:

```powershell
nslookup google.com
```

tests DNS resolution.

If the IP address works but the hostname does not resolve, the problem is likely related to DNS rather than general Internet routing.

---

# DNS Troubleshooting Experience

One important lesson from building this environment was that DNS failures can appear to be complete network outages.

For example:

```text
Ping IP Address      → Works

Open Website by Name → Fails
```

This indicates that Layer 3 connectivity may still be functioning while DNS resolution is broken.

The DNS dependency chain in the lab is:

```text
Client
  │
  ▼
DC1
10.0.0.6
  │
  ▼
pfSense
10.0.0.1
  │
  ▼
External DNS
```

Troubleshooting each step separately helps isolate the failure.

---

# Infrastructure Dependency Chain

Several systems depend on one another.

For example:

```text
Client01
   │
   ├── DNS ──────────────► DC1
   │                       10.0.0.6
   │
   ├── Authentication ───► DC1
   │
   ├── Group Policy ─────► DC1
   │
   └── Internet ─────────► pfSense
                           10.0.0.1
```

DC1 also depends on pfSense for external connectivity:

```text
DC1
 │
 ├── Internal DNS
 │
 └── External DNS Forwarding
          │
          ▼
       pfSense
       10.0.0.1
```

Proxmox similarly depends on pfSense as its gateway:

```text
Proxmox
10.0.0.5
    │
    ▼
pfSense
10.0.0.1
    │
    ▼
Internet / Other Networks
```

Understanding these dependencies is critical when troubleshooting infrastructure outages.

---

# Troubleshooting Methodology

When a system cannot communicate, addressing is checked before firewall rules or advanced routing.

A typical troubleshooting process is:

```text
Check Physical Link
        │
        ▼
Check IP Address
        │
        ▼
Check Subnet Mask
        │
        ▼
Check VLAN
        │
        ▼
Check Default Gateway
        │
        ▼
Ping Gateway
        │
        ▼
Check DNS Server
        │
        ▼
Test DNS Resolution
        │
        ▼
Inspect pfSense Rules
        │
        ▼
Inspect Routing / VPN
```

This prevents simple addressing problems from being mistaken for more complicated firewall or routing problems.

---

# Example Troubleshooting Scenario

Suppose Client01 cannot access the Internet.

The first check would be:

```powershell
ipconfig /all
```

Verify that the client has:

```text
10.0.0.x address
255.255.255.0 subnet mask
10.0.0.1 gateway
10.0.0.6 DNS server
```

Next:

```powershell
ping 10.0.0.1
```

If this fails, investigate the local network path.

If it succeeds:

```powershell
ping 1.1.1.1
```

If that succeeds but:

```powershell
nslookup google.com
```

fails, the issue is likely DNS-related.

This layered troubleshooting approach helps identify the actual failure domain.

---

# Security Benefits of Structured Addressing

IP addressing by itself is not a security control.

However, an organized addressing scheme makes security controls easier to implement and manage.

Predictable infrastructure IP addresses make it easier to:

- Create firewall rules
- Restrict administrative access
- Configure vulnerability scans
- Configure SIEM agents
- Identify critical systems in security logs
- Investigate suspicious traffic
- Create network diagrams
- Document dependencies
- Troubleshoot incidents

For example:

```text
10.0.0.6
```

can immediately be recognized as DC1.

Likewise:

```text
10.0.0.7
```

identifies the Wazuh SIEM server.

This makes logs and network activity easier to understand.

---

# Relationship to Security Monitoring

The addressing plan is also useful when analyzing Wazuh or firewall events.

For example:

```text
Source IP:      10.0.0.100
Destination IP: 10.0.0.6
```

can quickly indicate communication between a workstation and the domain controller.

Similarly:

```text
Source IP:      10.0.0.8
Destination IP: 10.0.0.100
```

could represent Nessus scanning the Windows client.

A documented IP inventory adds context to security events.

---

# Future Addressing Improvements

As the environment continues to grow, the IP architecture can become more segmented.

Planned improvements include:

- Dedicated addressing for VLAN 99
- Dedicated management interfaces
- Additional network segmentation
- Separate client and server networks
- Documented DHCP scopes
- DHCP reservations where appropriate
- More restrictive inter-VLAN routing
- Dedicated firewall aliases for infrastructure groups
- Improved IP inventory documentation

Possible future network categories could include:

```text
LAB-LAN
Management
Servers
Security Tools
IoT
Guest Devices
Remote VPN
```

Any future subnet will be documented only after it has actually been configured.

---

# What I Learned

Designing and troubleshooting the IP addressing architecture reinforced several important networking concepts.

I gained practical experience with:

- IPv4 addressing
- `/24` subnetting
- Static IP configuration
- DHCP
- Default gateways
- DNS
- Active Directory DNS
- DNS forwarding
- VPN addressing
- Inter-VLAN routing
- Network dependency mapping
- Connectivity testing
- Troubleshooting DNS separately from routing

One of the most important lessons from this project was that:

> **IP addressing, DNS, DHCP, routing, VLANs, and Active Directory all depend on one another.**

A failure in one service can appear as a problem somewhere else.

Understanding the full traffic and dependency path makes troubleshooting significantly faster and more accurate.

---

# Network Infrastructure Summary

At this stage, the network infrastructure portion of the home lab includes:

```text
Internet
   │
   ▼
pfSense
10.0.0.1
   │
   ▼
Cisco SW1
10.0.0.2
   │
   ▼
VLAN 10 — LAB-LAN
10.0.0.0/24
   │
   ├── Proxmox      10.0.0.5
   ├── DC1          10.0.0.6
   ├── Wazuh        10.0.0.7
   ├── Nessus       10.0.0.8
   ├── Client01     DHCP
   └── Jump PC      10.0.0.101

Remote Access
   │
   ▼
OpenVPN
10.8.0.0/24
```

This addressing architecture provides the foundation for the virtualization, firewall, Active Directory, SIEM, vulnerability-management, and security-testing components of the home lab.

---

# Related Documentation

[← VLAN Design and Network Segmentation](vlan-design.md)

[← Cisco Switch Configuration](cisco-switch.md)

[← Network Architecture](network-architecture.md)

[← Main Project README](../README.md)
