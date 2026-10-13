# pfSense Firewall and Routing Architecture

## Overview

pfSense serves as the primary **router, firewall, VPN gateway, and Layer 3 security boundary** for the home lab.

Unlike the original deployment, where pfSense ran directly on physical hardware, the current firewall operates as a **virtual machine hosted on Proxmox VE**.

pfSense is responsible for connecting the internal lab environment to external networks while enforcing security policies between different network segments.

Core functions include:

- Default gateway services
- Stateful firewalling
- Network Address Translation (NAT)
- Inter-VLAN routing
- Network segmentation
- OpenVPN remote access
- Surfshark OpenVPN client connectivity
- Policy-based routing
- DNS forwarding support
- Traffic control between trusted and untrusted networks

The primary LAB-LAN gateway is:

```text
10.0.0.1
```

---

# Role in the Home Lab

pfSense sits at the logical network edge.

```text
                     Internet
                        │
                        ▼
                     pfSense
                    10.0.0.1
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
       LAB-LAN         IOT       Remote VPN
       VLAN 10       VLAN 20     10.8.0.0/24
          │
          ▼
       Cisco SW1
          │
          ▼
       Proxmox
          │
     ┌────┼────┐
     │    │    │
    DC1 Wazuh Nessus
```

Although pfSense is a virtual machine running on Proxmox, it functions logically as the primary network gateway.

---

# Current pfSense Deployment

| Setting | Value |
|---|---|
| Platform | pfSense |
| Deployment | Virtual Machine |
| Hypervisor | Proxmox VE |
| LAN Gateway | `10.0.0.1` |
| Primary LAN | `10.0.0.0/24` |
| Primary VLAN | VLAN 10 — LAB-LAN |
| Remote VPN Network | `10.8.0.0/24` |
| VPN Technologies | OpenVPN |
| Outbound VPN Provider | Surfshark |

pfSense is one of the most critical virtual machines in the environment because many other systems depend on it for connectivity.

---

# Migration from Bare Metal to Proxmox

pfSense was originally deployed directly on physical hardware.

As the home lab evolved, it was migrated into Proxmox.

The architecture changed from:

```text
Physical pfSense
      │
      ▼
Lab Network
```

to:

```text
Dell PowerEdge R710
        │
        ▼
    Proxmox VE
        │
        ▼
     pfSense VM
        │
        ▼
    Lab Network
```

Virtualizing pfSense reduced the amount of dedicated physical hardware required and made the firewall part of the broader virtualization environment.

---

# Why Virtualize pfSense?

Running pfSense as a virtual machine provides several advantages.

### Hardware Consolidation

The firewall can run on the existing Dell PowerEdge server rather than requiring a separate system.

### Easier Recovery

VM configuration, snapshots, and backups can help simplify recovery.

### Flexible Networking

Virtual interfaces can connect pfSense to multiple networks through the Proxmox networking layer.

### Lab Expansion

Additional VLANs and network interfaces can be introduced without requiring a new physical firewall.

---

# Virtualization Trade-Off

Virtualizing the firewall also introduces an important dependency.

```text
Proxmox Down
     │
     ▼
pfSense Down
     │
     ▼
Gateway Down
     │
     ▼
Network Connectivity Impacted
```

If the Proxmox host is offline, pfSense is also unavailable.

Because pfSense provides the primary gateway for the environment, this can affect:

- Internet access
- VLAN routing
- VPN connectivity
- NAT
- Firewall enforcement

This dependency became particularly important during power outages and Proxmox maintenance.

---

# LAN Network

The primary LAB-LAN network uses:

```text
Network: 10.0.0.0/24
Gateway: 10.0.0.1
```

Critical systems include:

| Device | IP Address | Role |
|---|---:|---|
| pfSense | `10.0.0.1` | Gateway / Firewall |
| Cisco SW1 | `10.0.0.2` | Managed Switch |
| Proxmox | `10.0.0.5` | Hypervisor |
| DC1 | `10.0.0.6` | AD / DNS / DHCP |
| Wazuh | `10.0.0.7` | SIEM |
| Nessus | `10.0.0.8` | Vulnerability Scanner |
| Jump PC | `10.0.0.101` | Administrative Workstation |

pfSense provides the default gateway used by these systems when communicating outside the local subnet.

---

# Default Gateway Function

When a system communicates with another host inside:

```text
10.0.0.0/24
```

the traffic can remain within the local network.

When the destination exists outside the local subnet:

```text
Client
10.0.0.x
   │
   │ Remote Destination
   ▼
pfSense
10.0.0.1
   │
   ▼
Routing Decision
```

pfSense determines where the traffic should be forwarded.

---

# Routing Responsibilities

pfSense performs several types of routing within the lab.

```text
Internet Routing
Inter-VLAN Routing
VPN Routing
Policy-Based Routing
Remote-Access Routing
```

This makes pfSense the primary Layer 3 device in the environment.

The Cisco switch primarily provides Layer 2 connectivity, while pfSense controls communication between separate IP networks.

---

# Firewall Architecture

pfSense uses a stateful firewall.

Rather than simply forwarding traffic between networks, firewall rules determine what communication is permitted.

Conceptually:

```text
Source Network
      │
      ▼
   pfSense
      │
 Firewall Rules
      │
   ┌──┴──┐
   │     │
 Allow  Block
   │
   ▼
Destination
```

Traffic is evaluated based on factors such as:

- Source
- Destination
- Protocol
- Port
- Interface
- Gateway policy

---

# Stateful Firewalling

pfSense maintains connection state.

For example, if an internal workstation is permitted to initiate a web connection:

```text
Client
  │
  │ HTTPS Request
  ▼
Internet
```

pfSense tracks that connection and permits the expected return traffic.

This means firewall policies can allow legitimate session responses without requiring unrestricted inbound access.

---

# Network Segmentation

One of the primary security uses of pfSense is enforcing separation between different network segments.

The VLAN design currently includes:

| VLAN | Name | Purpose |
|---|---|---|
| VLAN 10 | LAB-LAN | Trusted lab infrastructure |
| VLAN 20 | IOT | IoT and smart devices |
| VLAN 99 | Management | Planned management network |

The design allows each network to have its own security policy.

---

# Inter-VLAN Routing

VLANs separate traffic at Layer 2.

When systems in different VLANs need to communicate, traffic must pass through pfSense.

```text
VLAN 20
IoT Device
    │
    ▼
Cisco SW1
    │
    ▼
Proxmox
    │
    ▼
pfSense
    │
 Firewall Rules
    │
    ▼
VLAN 10
LAB-LAN
```

This provides a central point for enforcing access controls between security zones.

---

# IoT Segmentation

VLAN 20 is used for IoT devices.

The objective is to avoid placing less-trusted smart devices directly on the same trusted network as:

- Domain controllers
- Hypervisors
- Security tools
- Administrative workstations

Conceptually:

```text
IoT Device
    │
    ▼
VLAN 20
    │
    ▼
pfSense
    │
    ├── Internet → Controlled
    │
    ├── LAB-LAN → Restricted
    │
    └── Management → Restricted
```

This reduces the potential attack surface if an IoT device becomes compromised.

---

# Management VLAN

VLAN 99 is planned as a dedicated management network.

The goal is eventually to isolate interfaces such as:

```text
Proxmox Management
pfSense Administration
Cisco Switch Management
Security Infrastructure
```

from normal workstation and IoT traffic.

A future architecture could resemble:

```text
Administrative Workstation
          │
          ▼
       VLAN 99
          │
          ▼
       pfSense
          │
          ├── Proxmox
          ├── Cisco SW1
          └── Security Tools
```

Access would be controlled through firewall policies.

---

# Network Address Translation

pfSense performs Network Address Translation for internal systems accessing external networks.

Conceptually:

```text
Internal Client
10.0.0.x
    │
    ▼
pfSense
    │
    │ NAT
    ▼
Public / External Network
```

Private addresses are not directly routable across the public Internet.

pfSense translates internal traffic to the appropriate outbound interface address.

---

# OpenVPN Remote Access

pfSense hosts an OpenVPN remote-access server.

The VPN network is:

```text
10.8.0.0/24
```

This allows authorized remote devices to securely connect to the home lab.

```text
Remote Computer
      │
      │ Internet
      ▼
pfSense OpenVPN
      │
      ▼
10.8.0.0/24
      │
      ▼
Internal Lab
10.0.0.0/24
```

The VPN provides remote administrative access without exposing individual management services directly to the Internet.

---

# Remote Access Use Cases

Once connected through OpenVPN, authorized remote users can access internal resources such as:

- pfSense
- Proxmox
- Windows Server
- Wazuh
- Nessus
- Other lab resources

Traffic remains subject to pfSense firewall policies.

---

# Remote VPN vs Outbound VPN

Two separate VPN functions exist in the environment.

## Remote-Access OpenVPN

This allows a remote user to enter the home lab.

```text
Remote User
     │
     ▼
Internet
     │
     ▼
pfSense
     │
     ▼
Home Lab
```

---

## Surfshark OpenVPN Client

This sends selected home-lab traffic outward through a commercial VPN provider.

```text
Home Lab Device
      │
      ▼
pfSense
      │
      ▼
Surfshark VPN
      │
      ▼
Internet
```

These two VPN configurations serve completely different purposes.

---

# Surfshark VPN Client

pfSense maintains an OpenVPN client connection to Surfshark.

This provides an alternative outbound gateway.

Instead of all Internet traffic automatically following the normal ISP path:

```text
Client
  │
  ▼
pfSense
  │
  ▼
ISP
```

selected traffic can follow:

```text
Client
  │
  ▼
pfSense
  │
  ▼
Surfshark OpenVPN
  │
  ▼
VPN Exit Server
  │
  ▼
Internet
```

---

# Policy-Based Routing

pfSense supports policy-based routing by allowing a firewall rule to specify a particular gateway.

This makes it possible to determine outbound routing based on the source network or device.

Conceptually:

```text
LAB-LAN
   │
   ▼
Firewall Rule
Gateway: ISP
   │
   ▼
Internet


IOT
   │
   ▼
Firewall Rule
Gateway: Surfshark
   │
   ▼
VPN
   │
   ▼
Internet
```

This provides greater control than relying only on the default routing table.

---

# Surfshark Testing

The Surfshark configuration was validated by checking the public IP address seen by external services.

When traffic successfully traversed the VPN, the public address changed from the ISP-assigned address to a Surfshark exit address.

This confirmed that:

```text
Client
   │
   ▼
pfSense
   │
   ▼
Surfshark Tunnel
   │
   ▼
Internet
```

was operating correctly.

Testing the externally visible IP address became an important validation technique during VPN troubleshooting.

---

# VPN Endpoint Troubleshooting

During configuration, different Surfshark endpoints were tested.

The OpenVPN connection also required troubleshooting cipher compatibility and endpoint configuration before becoming stable.

This reinforced the importance of checking:

- OpenVPN connection status
- Encryption settings
- Tunnel interface
- Assigned gateway
- Firewall rules
- NAT
- DNS
- Public IP address

rather than assuming that an OpenVPN status of "connected" automatically means client traffic is actually using the tunnel.

---

# DNS Architecture

DC1 provides internal DNS services at:

```text
10.0.0.6
```

Domain clients query DC1 so they can resolve:

```text
ad.home.com
```

and Active Directory service records.

External resolution follows the configured forwarding path toward pfSense.

Conceptually:

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

---

# DNS Troubleshooting

DNS issues have previously caused systems to appear as though they had lost Internet connectivity.

A useful diagnostic distinction is:

```text
Can ping external IP?
        │
        ├── No → Routing / Firewall / Gateway
        │
        └── Yes
              │
              ▼
       Can resolve hostname?
              │
              ├── No → DNS
              └── Yes → Higher-level issue
```

This is especially important because Active Directory also relies heavily on DNS.

---

# pfSense and Active Directory

pfSense and DC1 serve different infrastructure roles.

```text
pfSense
   │
   ├── Gateway
   ├── Firewall
   ├── Routing
   └── VPN


DC1
   │
   ├── Active Directory
   ├── DNS
   ├── DHCP
   └── Group Policy
```

A network problem therefore requires identifying which infrastructure component actually owns the failing service.

---

# Firewall Rule Troubleshooting

When traffic fails, pfSense firewall rules are one of several possible causes.

The troubleshooting approach should not begin by randomly creating allow rules.

Instead:

```text
Verify Client IP
      │
      ▼
Verify VLAN
      │
      ▼
Verify Gateway
      │
      ▼
Verify pfSense Interface
      │
      ▼
Verify Firewall Rule
      │
      ▼
Verify Gateway Selection
      │
      ▼
Verify NAT
      │
      ▼
Verify Destination
```

This prevents firewall rules from being blamed for problems that actually originate elsewhere.

---

# Packet Path Troubleshooting

Understanding the packet path is one of the most important skills developed during this project.

For example, IoT Internet traffic may travel through:

```text
IoT Device
    │
    ▼
Cisco Access Port
    │
    ▼
VLAN 20
    │
    ▼
Cisco Trunk
    │
    ▼
Proxmox vmbr0
    │
    ▼
pfSense
    │
    ├── Firewall Rule
    ├── Gateway Selection
    └── NAT
            │
            ▼
      Surfshark / ISP
            │
            ▼
         Internet
```

A failure can occur at any point in this path.

---

# Packet Capture

pfSense packet capture is useful when firewall or routing behavior is unclear.

Packet capture can help determine whether traffic:

- Reaches an interface
- Leaves an interface
- Uses the expected source address
- Uses the correct destination
- Traverses the VPN tunnel

The ability to observe traffic directly can help distinguish configuration assumptions from actual packet behavior.

---

# OpenVPN Diagnostics

VPN troubleshooting may require checking whether tunnel traffic is actually present.

For example, packet capture or command-line tools can show UDP traffic to an OpenVPN endpoint.

This helps answer questions such as:

```text
Is pfSense sending VPN packets?

Is the tunnel communicating with the remote endpoint?

Is client traffic entering the tunnel?

Is return traffic arriving?
```

This approach was used during VPN troubleshooting in the lab.

---

# Firewall Rule Order

pfSense evaluates interface firewall rules in order.

Conceptually:

```text
Rule 1
  │
  ├── Match? → Apply
  │
  └── No
       │
       ▼
Rule 2
  │
  ├── Match? → Apply
  │
  └── No
       │
       ▼
Next Rule
```

This makes rule placement important.

A broad rule above a restrictive rule could prevent the lower rule from having the intended effect.

---

# Least-Privilege Firewalling

The long-term goal of the firewall architecture is to follow least privilege.

Instead of:

```text
Allow Any → Any
```

where possible, policies should identify:

```text
Required Source
Required Destination
Required Protocol
Required Port
Required Gateway
```

This is particularly important when implementing the planned management VLAN.

---

# Example Security Zones

The environment is evolving toward a security-zone model.

```text
TRUSTED
VLAN 10 — LAB-LAN

LESS TRUSTED
VLAN 20 — IOT

ADMINISTRATIVE
VLAN 99 — Management

REMOTE
10.8.0.0/24 — VPN Clients

EXTERNAL
Internet
```

pfSense acts as the enforcement point between these zones.

---

# Firewall as a Security Boundary

This architecture provides a stronger design than relying only on endpoint firewalls.

For example:

```text
Compromised IoT Device
       │
       ▼
    VLAN 20
       │
       ▼
    pfSense
       │
 Firewall Policy
       │
       X
   DC1 / Proxmox
```

Network controls provide another defensive layer even if the endpoint itself becomes compromised.

---

# Defense-in-Depth

pfSense is one layer in a broader security architecture.

```text
Network Segmentation
        │
        ▼
     pfSense
        │
        ▼
Active Directory / GPO
        │
        ▼
Endpoint Auditing / Sysmon
        │
        ▼
       Wazuh
        │
        ▼
      Nessus
```

No single layer is expected to provide complete security.

---

# Administrative Access

The pfSense web interface is a sensitive management service.

Administrative access should be limited to trusted systems.

The future VLAN 99 design will provide stronger separation for management services.

Conceptually:

```text
Jump PC
   │
   ▼
VLAN 99
   │
   ▼
pfSense Management
```

Normal IoT or user devices should not require access to the pfSense administration interface.

---

# Configuration Backups

pfSense configuration backups are an important part of disaster recovery.

The firewall contains critical configuration including:

- Interfaces
- VLANs
- Firewall rules
- NAT
- OpenVPN
- Certificates
- Gateways
- DNS-related settings
- Routing configuration

A configuration loss could affect the entire network.

Backing up the pfSense configuration makes recovery significantly faster than rebuilding the firewall manually.

---

# Change Management

Firewall changes can have immediate network-wide consequences.

A safe change process is:

```text
Document Current State
        │
        ▼
Back Up Configuration
        │
        ▼
Make One Change
        │
        ▼
Apply
        │
        ▼
Test Connectivity
        │
        ├── Works → Continue
        │
        └── Fails → Roll Back / Troubleshoot
```

Making several firewall, routing, and VPN changes simultaneously makes troubleshooting significantly harder.

---

# Troubleshooting Lessons

pfSense troubleshooting in this lab reinforced several important principles.

### A VPN Being Connected Does Not Prove Traffic Uses It

The tunnel may be active while clients continue using the normal ISP gateway.

### DNS Can Mimic Internet Failure

Routing may be functional while hostname resolution is broken.

### Firewall Rules Are Only One Layer

Switching, VLANs, Proxmox, routing, NAT, and DNS should also be checked.

### Policy Routing Changes the Packet Path

The gateway selected by a firewall rule can completely change where traffic exits.

### Virtualization Adds Another Layer

pfSense networking also depends on Proxmox and the Cisco trunk.

---

# Infrastructure Dependencies

pfSense depends on:

```text
Dell R710
   │
   ▼
Proxmox
   │
   ▼
Virtual Networking
   │
   ▼
Cisco Trunk
```

Meanwhile, many other systems depend on pfSense:

```text
pfSense
   │
   ├── Proxmox Internet Access
   ├── DC1 External Connectivity
   ├── Wazuh Connectivity
   ├── Nessus Connectivity
   ├── Client Internet Access
   ├── VLAN Routing
   └── VPN Connectivity
```

This makes pfSense both dependent on the virtualization infrastructure and critical to the services running on it.

---

# Future Improvements

Planned improvements to the pfSense architecture include:

- Complete VLAN 99 management deployment
- Restrict management interfaces to authorized systems
- Continue refining IoT firewall policies
- Improve network aliases and rule organization
- Improve VPN failover behavior
- Expand network monitoring
- Improve firewall logging
- Document rule sets in greater detail
- Improve disaster-recovery procedures
- Integrate future network segments as the lab grows

---

# Skills Demonstrated

This portion of the project demonstrates practical experience with:

### Firewall Administration

- pfSense
- Stateful firewalling
- Firewall rule design
- Rule ordering
- Network security zones

### Routing

- Default gateways
- Inter-VLAN routing
- Static routing concepts
- Policy-based routing
- VPN routing

### VPN Technologies

- OpenVPN server
- OpenVPN client
- Remote-access VPN
- Commercial VPN integration
- Tunnel troubleshooting

### Networking

- NAT
- VLANs
- DNS forwarding
- Gateway selection
- Packet paths
- Packet capture

### Security Architecture

- Network segmentation
- Least privilege
- IoT isolation
- Management network planning
- Defense-in-depth

---

# What I Learned

Building and troubleshooting pfSense reinforced that a firewall is much more than a device that blocks ports.

pfSense became the central network policy engine for the lab.

I gained hands-on experience with:

- Stateful firewall rules
- NAT
- VLAN routing
- Remote-access VPNs
- Outbound VPNs
- Policy-based routing
- DNS troubleshooting
- Packet-path analysis
- Virtualized firewall architecture
- Network segmentation

The most important lesson was:

> **To troubleshoot a firewall correctly, you must understand the complete packet path.**

A connection can fail because of:

```text
Endpoint
   ↓
VLAN
   ↓
Switch
   ↓
Proxmox
   ↓
pfSense Interface
   ↓
Firewall Rule
   ↓
Gateway
   ↓
NAT
   ↓
VPN
   ↓
DNS
   ↓
Destination
```

Understanding each layer makes troubleshooting more systematic and reduces unnecessary configuration changes.

---

# pfSense Architecture Summary

```text
                         Internet
                            │
                  ┌─────────┴─────────┐
                  │                   │
                  ▼                   ▼
                ISP              Surfshark VPN
                  │                   ▲
                  │                   │
                  └─────────┬─────────┘
                            │
                            ▼
                         pfSense
                        10.0.0.1
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
       VLAN 10           VLAN 20          OpenVPN
       LAB-LAN             IOT            10.8.0.0/24
          │                 │                 │
          ▼                 ▼                 ▼
    Trusted Systems     IoT Devices      Remote Users
```

pfSense provides the routing and security foundation connecting the internal lab, segmented networks, VPN users, and external Internet services.

---

# Related Documentation

[← Proxmox Power Outage Recovery](../02-proxmox/outage-recovery.md)

[← Proxmox Virtual Networking](../02-proxmox/virtual-networking.md)

[← VLAN Design and Network Segmentation](../01-network-infrastructure/vlan-design.md)

[← Network Architecture](../01-network-infrastructure/network-architecture.md)

[← Main Project README](../README.md)
