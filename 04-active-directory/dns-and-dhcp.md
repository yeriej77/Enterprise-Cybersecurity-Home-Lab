# Active Directory, DNS and DHCP

## Overview

Windows Server 2025 provides the primary **DNS and DHCP services** for the Active Directory environment.

The server is:

```text
Hostname: DC1
IP Address: 10.0.0.6
Domain: ad.home.com
```

DC1 performs several closely related infrastructure functions:

```text
Active Directory + DNS + DHCP = Centralized Windows Network Services
```

DNS allows domain clients to locate Active Directory services and internal systems.

DHCP automatically provides network configuration to client devices.

pfSense remains the primary network gateway at:

```text
10.0.0.1
```

while DC1 provides:

```text
DNS:  10.0.0.6
DHCP: DC1
```

---

# Network Services Architecture

The general architecture is:

```text
                        Internet
                           │
                           ▼
                        pfSense
                       10.0.0.1
                           │
                           ▼
                    VLAN 10 / LAB-LAN
                       10.0.0.0/24
                           │
                  ┌────────┴────────┐
                  │                 │
                  ▼                 ▼
                 DC1             Client01
              10.0.0.6
                  │
          ┌───────┼────────┐
          │       │        │
          ▼       ▼        ▼
         AD      DNS      DHCP
```

This separates the network gateway role from the Windows infrastructure-services role.

---

# Service Responsibilities

The primary infrastructure responsibilities are divided between pfSense and DC1.

| Service | System | Address |
|---|---|---:|
| Default Gateway | pfSense | `10.0.0.1` |
| Firewall | pfSense | `10.0.0.1` |
| Routing | pfSense | `10.0.0.1` |
| Active Directory | DC1 | `10.0.0.6` |
| DNS | DC1 | `10.0.0.6` |
| DHCP | DC1 | `10.0.0.6` |

This separation is important when troubleshooting.

For example:

```text
Internet routing problem
        │
        ▼
Investigate pfSense
```

while:

```text
Domain-name resolution problem
        │
        ▼
Investigate DC1 / DNS
```

---

# DNS Role

DNS is one of the most important services in an Active Directory environment.

The internal DNS server is:

```text
10.0.0.6
```

and the Active Directory DNS namespace is:

```text
ad.home.com
```

DC1 hosts the DNS information required for domain members to locate internal services.

---

# Why Active Directory Depends on DNS

Active Directory does not rely only on IP addresses.

Clients use DNS to discover important services such as:

- Domain controllers
- LDAP
- Kerberos
- Global Catalog services
- Internal Windows hosts

Conceptually:

```text
Client01
   │
   │ Where is a domain controller?
   ▼
DNS
10.0.0.6
   │
   ▼
Active Directory DNS Records
   │
   ▼
DC1
```

If DNS is configured incorrectly, Active Directory can appear to be broken even though the domain controller itself is running.

---

# Active Directory DNS Zone

The primary internal domain is:

```text
ad.home.com
```

DC1 maintains DNS records for this namespace.

A host such as the domain controller can therefore be referenced as:

```text
dc1.ad.home.com
```

rather than relying only on:

```text
10.0.0.6
```

---

# Active Directory Service Records

Active Directory creates special DNS records called **SRV records**.

These help Windows systems locate specific domain services.

One important record is:

```text
_ldap._tcp.dc._msdcs.ad.home.com
```

This identifies domain controllers providing LDAP services.

It can be tested using:

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.ad.home.com
```

A successful response confirms that Active Directory service discovery information is available through DNS.

---

# Domain Controller Discovery

Windows clients can also directly test domain-controller discovery using:

```powershell
nltest /dsgetdc:ad.home.com
```

This provides information about the domain controller selected by the client.

The process can be simplified as:

```text
Client
  │
  ▼
DNS Lookup
  │
  ▼
SRV Record
  │
  ▼
DC1 Located
  │
  ▼
Authentication / Domain Services
```

---

# Domain Client DNS Configuration

Domain-joined Windows systems should use DC1 as their primary DNS server.

A normal client configuration should therefore resemble:

```text
IPv4 Address:    10.0.0.x
Subnet Mask:     255.255.255.0
Default Gateway: 10.0.0.1
DNS Server:      10.0.0.6
DNS Suffix:      ad.home.com
```

The gateway and DNS server perform different functions.

```text
10.0.0.1
pfSense
   │
   └── Routing / Internet


10.0.0.6
DC1
   │
   └── DNS / AD / DHCP
```

---

# Why Domain Clients Should Not Use Public DNS Directly

A domain client should not normally use a public DNS resolver as its primary DNS server.

For example:

```text
Client
   │
   ▼
1.1.1.1
```

may resolve public websites but does not contain the private Active Directory DNS records for:

```text
ad.home.com
```

This could cause failures involving:

- Domain logon
- Group Policy
- Domain-controller discovery
- Internal hostname resolution
- Kerberos
- LDAP

The preferred design is:

```text
Domain Client
     │
     ▼
DC1
10.0.0.6
     │
     ├── Internal DNS → Resolve locally
     │
     └── External DNS → Forward outward
```

---

# Internal DNS Resolution

When a domain client requests an internal hostname:

```text
dc1.ad.home.com
```

the request follows:

```text
Client01
   │
   ▼
DC1 DNS
10.0.0.6
   │
   ▼
ad.home.com Zone
   │
   ▼
10.0.0.6
```

This allows internal services to be reached by hostname rather than requiring users or applications to remember IP addresses.

---

# External DNS Resolution

For a public Internet hostname, the general path becomes:

```text
Client
   │
   ▼
DC1
10.0.0.6
   │
   ▼
DNS Forwarding Path
   │
   ▼
pfSense / External Resolution
   │
   ▼
Internet DNS
```

The important design principle is that domain clients continue to send their DNS requests to DC1.

DC1 then handles internal and external resolution appropriately.

---

# DNS Forwarding

DNS forwarding allows DC1 to send queries it cannot resolve locally toward an upstream resolver.

The logic is:

```text
DNS Request
    │
    ▼
Does DC1 know the answer?
    │
 ┌──┴──┐
 │     │
Yes    No
 │     │
 ▼     ▼
Return Forward Query
Answer   Upstream
```

This keeps internal Active Directory DNS centralized while still allowing users to access public Internet resources.

---

# Why Centralized DNS Matters

Centralizing DNS through DC1 makes the environment easier to manage.

It provides:

- Internal hostname resolution
- Active Directory service discovery
- Consistent client configuration
- Centralized troubleshooting
- Better control over domain DNS behavior

It also makes infrastructure dependencies easier to understand.

---

# DHCP Role

DC1 also provides DHCP services to client systems.

DHCP automatically provides network configuration instead of requiring every workstation to be configured manually.

Conceptually:

```text
New Client
    │
    ▼
DHCP Request
    │
    ▼
DC1 DHCP
    │
    ▼
Network Configuration
```

The client can receive information such as:

```text
IP Address
Subnet Mask
Default Gateway
DNS Server
Domain Information
```

---

# Why Use DHCP?

Without DHCP, every workstation would require manual configuration.

For example:

```text
Client01
IP address manually configured

Client02
IP address manually configured

Client03
IP address manually configured
```

This becomes difficult to maintain and increases the chance of configuration mistakes.

With DHCP:

```text
                 DC1
                DHCP
                  │
         ┌────────┼────────┐
         │        │        │
         ▼        ▼        ▼
      Client01 Client02 Client03
```

network configuration can be distributed centrally.

---

# DHCP Client Process

The basic DHCP process is commonly described as:

```text
Discover
   │
   ▼
Offer
   │
   ▼
Request
   │
   ▼
Acknowledge
```

or:

```text
DORA
```

The sequence works like this:

```text
Client
  │
  │ DHCP Discover
  ▼
DHCP Server
  │
  │ DHCP Offer
  ▼
Client
  │
  │ DHCP Request
  ▼
DHCP Server
  │
  │ DHCP Acknowledge
  ▼
Client Configured
```

---

# DHCP-Provided Settings

For a LAB-LAN client, DHCP needs to provide network information consistent with the environment.

The important settings include:

```text
Network:         10.0.0.0/24
Subnet Mask:     255.255.255.0
Default Gateway: 10.0.0.1
DNS Server:      10.0.0.6
Domain:          ad.home.com
```

The exact DHCP address pool should be documented directly from the live server configuration rather than assumed.

---

# Static vs DHCP Addresses

Critical infrastructure uses predictable addresses.

Examples include:

```text
pfSense      10.0.0.1
Cisco SW1    10.0.0.2
Proxmox      10.0.0.5
DC1          10.0.0.6
Wazuh        10.0.0.7
Nessus       10.0.0.8
```

Client systems can use DHCP.

This creates a design similar to:

```text
Low / Known Addresses
      │
      └── Infrastructure


DHCP-Managed Addresses
      │
      └── Client Systems
```

The exact DHCP scope boundaries should be based on the live DHCP configuration.

---

# Why Infrastructure Should Not Depend on Random DHCP Addresses

Infrastructure services frequently reference one another by address.

For example:

```text
Domain Clients
      │
      ▼
DNS = 10.0.0.6
```

If DC1 unexpectedly changed addresses:

```text
10.0.0.6
    │
    ▼
Different IP
```

clients could lose access to DNS and Active Directory services.

Critical systems therefore use predictable addressing.

---

# DHCP and Active Directory

DHCP and Active Directory are separate technologies, but they work together in the environment.

```text
DHCP
  │
  └── Gives client network configuration


DNS
  │
  └── Helps client locate domain resources


Active Directory
  │
  └── Authenticates and manages client
```

A domain workstation relies on all three.

---

# Complete Client Startup Path

A simplified Windows-client startup looks like:

```text
Client01 Starts
      │
      ▼
Obtain Network Configuration
      │
      ▼
DHCP
      │
      ▼
Receive IP / Gateway / DNS
      │
      ▼
Contact DNS
      │
      ▼
Locate DC1
      │
      ▼
Authenticate to Domain
      │
      ▼
Apply Group Policy
      │
      ▼
User Environment Ready
```

This demonstrates how one infrastructure service depends on another.

---

# Client Configuration Verification

A Windows client can display its current network configuration using:

```powershell
ipconfig /all
```

Important values to inspect include:

```text
IPv4 Address
Subnet Mask
Default Gateway
DHCP Server
DNS Servers
Primary DNS Suffix
```

For a normal domain-connected client, the important values should align with:

```text
Gateway: 10.0.0.1
DNS:     10.0.0.6
Domain:  ad.home.com
```

---

# Renewing a DHCP Lease

A Windows client can release its current DHCP configuration with:

```powershell
ipconfig /release
```

and request a new lease with:

```powershell
ipconfig /renew
```

This can be useful when troubleshooting DHCP changes.

Care should be taken when running these commands remotely because releasing an address can temporarily remove network connectivity.

---

# DNS Cache Troubleshooting

Windows also maintains a DNS cache.

The cache can be inspected using:

```powershell
ipconfig /displaydns
```

and cleared using:

```powershell
ipconfig /flushdns
```

This can help when an old or incorrect DNS result remains cached after a DNS configuration change.

---

# DNS Testing

A basic lookup can be performed with:

```powershell
nslookup dc1.ad.home.com
```

The DNS server responding should be:

```text
10.0.0.6
```

and the domain controller should resolve correctly.

---

# Testing Public DNS Resolution

Public DNS resolution can be tested using:

```powershell
nslookup google.com
```

If this succeeds, DC1 is successfully resolving or forwarding external DNS requests.

---

# Testing a Specific DNS Server

To query DC1 explicitly:

```powershell
nslookup google.com 10.0.0.6
```

This removes ambiguity about which DNS server is being tested.

---

# Testing Active Directory SRV Records

A more meaningful Active Directory DNS test is:

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.ad.home.com
```

This verifies that DNS contains the records domain clients use to locate LDAP services.

---

# DNS Troubleshooting Scenario

One of the most important troubleshooting lessons in the home lab was that a DNS problem can appear to be an Internet problem.

For example:

```text
ping 1.1.1.1
      │
      ▼
Works
```

but:

```text
nslookup google.com
      │
      ▼
Fails
```

This means basic IP routing is functioning.

The failure is related to name resolution.

---

# Routing vs DNS

These two tests answer different questions.

```powershell
ping 1.1.1.1
```

asks:

> Can the system reach an external IP address?

While:

```powershell
nslookup google.com
```

asks:

> Can the system resolve a hostname?

This distinction is extremely important when troubleshooting Windows domain environments.

---

# DNS Dependency Chain

The home lab DNS dependency can be represented as:

```text
Client
  │
  ▼
DC1
10.0.0.6
  │
  ▼
DNS Resolution / Forwarding
  │
  ▼
pfSense
10.0.0.1
  │
  ▼
External Network
```

If one component fails, external hostname resolution can fail.

---

# Active Directory Dependency Chain

For domain operations:

```text
Client01
    │
    ▼
DNS
10.0.0.6
    │
    ▼
Locate DC1
    │
    ▼
Active Directory
    │
    ▼
Authentication
    │
    ▼
Group Policy
```

This explains why DNS is often described as a foundational service for Active Directory.

---

# DNS Outage Troubleshooting

When name resolution fails, the troubleshooting process is:

```text
Check Client IP
      │
      ▼
Check DNS Server
      │
      ▼
Can Client Reach 10.0.0.6?
      │
      ▼
Is DNS Service Running?
      │
      ▼
Can Internal Records Resolve?
      │
      ▼
Can External Records Resolve?
      │
      ▼
Check Forwarding Path
```

This separates internal Active Directory DNS problems from external forwarding problems.

---

# DHCP Troubleshooting

When a client does not receive correct network configuration:

```text
Client Has Address?
      │
      ▼
Correct Subnet?
      │
      ▼
Correct Gateway?
      │
      ▼
Correct DNS?
      │
      ▼
Correct DHCP Server?
```

If the client receives an automatic address in:

```text
169.254.0.0/16
```

it typically indicates that the system did not successfully obtain a DHCP lease.

---

# APIPA

Windows can assign itself an Automatic Private IP Address when DHCP is unavailable.

The range is:

```text
169.254.0.0/16
```

For example:

```text
169.254.x.x
```

If this appears on Client01 instead of an expected lab address, troubleshooting should focus on the path between the client and the DHCP server.

---

# DHCP Troubleshooting Path

A DHCP issue can exist at several layers:

```text
Client
  │
  ▼
Network Adapter
  │
  ▼
Cisco Switch Port
  │
  ▼
VLAN
  │
  ▼
Network Path
  │
  ▼
DC1 DHCP Service
```

The DHCP service should not automatically be blamed before verifying the network path.

---

# Service Validation on DC1

Windows Server provides management tools for reviewing DNS and DHCP status.

Important areas include:

```text
Server Manager
DNS Manager
DHCP Manager
Event Viewer
Services
```

These tools allow configuration and service health to be inspected from the server itself.

---

# Power Outage Impact

The importance of DNS and DHCP became particularly clear after an unexpected power outage.

When DC1 did not return normally, the environment could experience:

```text
DC1 Unavailable
      │
      ├── DNS Unavailable
      ├── DHCP Unavailable
      ├── AD Authentication Impact
      └── Group Policy Impact
```

Meanwhile, pfSense could still be operational.

This can create a situation where:

```text
Routing works
```

but:

```text
Windows services appear broken
```

---

# Why Existing Clients May Behave Differently During a DHCP Outage

A DHCP outage does not necessarily disconnect every client immediately.

A device with an existing valid lease may continue using its current network configuration temporarily.

However, new systems or systems requiring lease renewal may be unable to obtain proper addressing.

This is why DHCP outages may produce inconsistent symptoms across different endpoints.

---

# Why DNS Failure Can Have Immediate Impact

DNS failure can become noticeable much more quickly.

A client may retain:

```text
IP Address
Gateway
```

but still be unable to locate:

```text
Domain Controller
Internal Hostnames
Internet Hostnames
```

This makes DNS one of the first services to verify when Windows-domain behavior becomes abnormal.

---

# Security Considerations

DNS and DHCP are infrastructure services and should be treated as security-sensitive.

An attacker who can manipulate DNS could potentially redirect users toward unintended systems.

An unauthorized DHCP server could provide clients with malicious:

- Gateways
- DNS servers
- Network settings

Protecting infrastructure services therefore contributes to the overall security of the environment.

---

# DNS Security Relevance

DNS activity can also provide useful security context.

For example:

```text
Endpoint
   │
   ▼
Suspicious Domain Lookup
```

may be useful when investigating malware or command-and-control activity.

As the monitoring environment grows, DNS visibility can provide another layer of detection context.

---

# DHCP Security Relevance

DHCP also helps identify which device received an address.

This can be useful during investigations.

For example:

```text
Security Alert
Source IP: 10.0.0.x
      │
      ▼
Identify Endpoint
      │
      ▼
Review DHCP / Host Information
```

Accurate infrastructure records make security investigations easier.

---

# Relationship to Wazuh

DC1 is monitored by the Wazuh SIEM.

```text
DC1
  │
  ▼
Windows Event Logs
  │
  ▼
Wazuh Agent
  │
  ▼
Wazuh Server
10.0.0.7
```

This provides visibility into security events occurring on one of the most important infrastructure servers.

---

# Relationship to Group Policy

Correct DNS and DHCP configuration is also necessary for reliable Group Policy processing.

```text
Client
  │
  ▼
DNS
  │
  ▼
Locate DC1
  │
  ▼
Active Directory
  │
  ▼
Group Policy
```

If domain-controller discovery fails, Group Policy processing can also fail.

---

# Relationship to Nessus

Authenticated Nessus scanning also depends on stable network and name-resolution infrastructure.

The scanner is located at:

```text
10.0.0.8
```

and scans Windows systems within the lab.

Reliable addressing helps ensure that vulnerability-management results can be associated with the correct systems.

---

# Infrastructure Documentation

Keeping DNS, DHCP, and IP-address information documented helps prevent configuration conflicts.

Important records include:

```text
Static Infrastructure Addresses
DHCP-Managed Clients
Default Gateway
DNS Server
Domain Name
VLAN Membership
```

Documentation becomes increasingly important as additional systems are added.

---

# Current Architecture

```text
                           Internet
                              │
                              ▼
                           pfSense
                          10.0.0.1
                              │
                              ▼
                        LAB-LAN VLAN 10
                         10.0.0.0/24
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
                DC1                     Clients
             10.0.0.6
                 │
      ┌──────────┼───────────┐
      │          │           │
      ▼          ▼           ▼
 Active Directory DNS       DHCP
      │          │           │
      └──────────┼───────────┘
                 │
                 ▼
           Domain Services
```

---

# Future Improvements

As the environment grows, future DNS and DHCP improvements may include:

- More detailed DHCP scope documentation
- DHCP reservations where appropriate
- Improved DNS logging
- Additional DNS monitoring
- Backup and restore testing
- More detailed service-health monitoring
- DHCP redundancy concepts
- DNS redundancy concepts
- Additional domain controllers
- Dedicated management-network integration
- More detailed IP address management

Adding another domain controller in the future could also provide an opportunity to learn Active Directory and DNS redundancy.

---

# High Availability Consideration

The current environment relies heavily on one domain controller:

```text
DC1
10.0.0.6
```

This creates a service dependency for:

```text
DNS
DHCP
Active Directory
Group Policy
```

A production environment would typically reduce this dependency through additional infrastructure.

For example:

```text
               Active Directory
                      │
            ┌─────────┴─────────┐
            │                   │
            ▼                   ▼
           DC1                 DC2
          DNS/AD              DNS/AD
```

The home lab currently uses DC1 while providing an opportunity to explore redundancy in the future.

---

# Troubleshooting Workflow

A practical Windows-network troubleshooting workflow is:

```text
Client Problem
     │
     ▼
ipconfig /all
     │
     ▼
Correct IP?
     │
     ▼
Correct Gateway?
     │
     ▼
Correct DNS?
     │
     ▼
Ping 10.0.0.1
     │
     ▼
Ping 10.0.0.6
     │
     ▼
nslookup dc1.ad.home.com
     │
     ▼
Check SRV Records
     │
     ▼
nltest /dsgetdc:ad.home.com
     │
     ▼
Test Authentication / GPO
```

This progresses from basic networking toward higher-level Active Directory services.

---

# Lessons Learned

Implementing and troubleshooting DNS and DHCP provided practical experience with:

- Windows Server DNS
- Windows Server DHCP
- Active Directory-integrated DNS concepts
- DNS forwarding
- SRV records
- DHCP client configuration
- Domain-controller discovery
- IP troubleshooting
- DNS troubleshooting
- Windows networking commands
- Infrastructure dependency analysis

The most important lesson was:

> **DNS is not an optional supporting service for Active Directory — it is a fundamental part of how Active Directory functions.**

Another important lesson was:

> **Internet connectivity and DNS connectivity are different problems and should always be tested separately.**

A system can have completely functional IP routing while appearing offline because name resolution has failed.

---

# DNS and DHCP Summary

```text
                         Windows Client
                              │
                    ┌─────────┴─────────┐
                    │                   │
                    ▼                   ▼
                  DHCP                 DNS
                    │                   │
                    ▼                   ▼
                   DC1                 DC1
                10.0.0.6            10.0.0.6
                    │                   │
                    │                   ├── ad.home.com
                    │                   │
                    │                   └── External Forwarding
                    │
                    ▼
           Network Configuration
                    │
         ┌──────────┼──────────┐
         │          │          │
         ▼          ▼          ▼
     IP Address   Gateway      DNS
                  10.0.0.1    10.0.0.6
```

Together, DNS and DHCP provide the network foundation required for reliable Active Directory operation throughout the home lab.

---

# Related Documentation

[← Windows Server 2025 and Active Directory](windows-server-2025.md)

[← IP Addressing and Network Services](../01-network-infrastructure/ip-addressing.md)

[← pfSense Firewall and Routing Architecture](../03-pfsense/pfsense-architecture.md)

[← Proxmox Virtualization Architecture](../02-proxmox/proxmox-architecture.md)

[← Main Project README](../README.md)
