# Network Architecture

## Overview

The network architecture for this home lab was designed to simulate a small enterprise environment with centralized routing, segmentation, virtualization, identity services, remote access, and security monitoring.

The environment uses **pfSense** as the primary router and firewall, a **Cisco Catalyst switch** for Layer 2 switching and VLAN connectivity, and **Proxmox VE** as the virtualization platform hosting the majority of the lab infrastructure.

The primary internal network is:

```text
10.0.0.0/24
```

The design separates major infrastructure roles while allowing pfSense to control communication between internal systems, IoT devices, remote VPN users, and the Internet.

---

## Logical Network Architecture

![Enterprise Cybersecurity Home Lab Network Architecture](../diagrams/network-architecture.jpeg)

> **Note:** This is a logical network diagram. pfSense is hosted as a virtual machine on the Proxmox server but is positioned at the top of the diagram because it functions as the logical network edge and default gateway for the lab.

---

# Core Infrastructure

| Device    | Role                            |   IP Address |
| --------- | ------------------------------- | -----------: |
| pfSense   | Router / Firewall / VPN Gateway |   `10.0.0.1` |
| Cisco SW1 | Managed Layer 2 Switch          |   `10.0.0.2` |
| Proxmox   | Virtualization Host             |   `10.0.0.5` |
| DC1       | Active Directory / DNS / DHCP   |   `10.0.0.6` |
| Wazuh01   | SIEM / Security Monitoring      |   `10.0.0.7` |
| Nessus    | Vulnerability Scanner — Planned |   `10.0.0.8` |
| Client01  | Domain-Joined Windows Client    | `10.0.0.100` |
| Jump PC   | Administrative Workstation      | `10.0.0.101` |

---

# Network Flow

At a high level, traffic moves through the environment as follows:

```text
Internet
   │
   ▼
ISP Router
   │
   ▼
pfSense
   │
   ▼
Cisco SW1
   │
   ▼
Proxmox / Physical Devices
   │
   ├── DC1
   ├── Wazuh01
   ├── Client01
   ├── Future Nessus Server
   └── Other Lab Systems
```

pfSense acts as the primary control point between the lab and external networks.

It provides:

* Routing
* Firewall enforcement
* Network Address Translation
* Inter-VLAN routing
* Remote-access VPN services
* Policy-based routing
* Surfshark VPN connectivity
* DNS forwarding/resolution services

---

# pfSense Network Edge

pfSense operates as the primary router and firewall for the home lab.

```text
LAN Gateway: 10.0.0.1
```

Although pfSense is hosted as a virtual machine inside Proxmox, its virtual network interfaces connect the WAN and internal lab networks.

From a logical perspective, pfSense sits between the ISP connection and the rest of the environment.

Its responsibilities include:

* Providing the default gateway for lab systems
* Filtering traffic using firewall rules
* Routing between VLANs
* Performing NAT for Internet access
* Hosting the OpenVPN remote-access server
* Maintaining a Surfshark OpenVPN client connection
* Applying policy-based routing for selected networks
* Controlling access to IoT devices

This makes pfSense one of the most critical components of the environment.

---

# Cisco Switching Infrastructure

A Cisco Catalyst switch provides Layer 2 connectivity for physical devices and the Proxmox host.

Management address:

```text
10.0.0.2
```

The switch currently operates within the primary LAB-LAN network.

The switch is responsible for:

* Physical Ethernet connectivity
* VLAN assignment
* Access-port configuration
* 802.1Q trunking
* Carrying VLAN traffic toward Proxmox and pfSense

The link between the Cisco switch and Proxmox is configured as an **802.1Q trunk**.

This allows multiple VLANs to traverse a single physical connection.

---

# Proxmox Trunk Connection

The primary switch-to-Proxmox connection carries multiple VLANs.

The trunk currently allows:

```text
VLAN 1
VLAN 10
VLAN 20
VLAN 99
```

VLAN 10 is configured as the native VLAN.

Conceptually:

```text
Cisco SW1
     │
     │ 802.1Q Trunk
     │ VLANs 1,10,20,99
     │ Native VLAN 10
     ▼
Proxmox
```

This design allows Proxmox virtual machines to connect to different VLANs without requiring separate physical network adapters for every network segment.

---

# Proxmox Networking

The Proxmox server uses:

```text
IP Address: 10.0.0.5
Default Gateway: 10.0.0.1
```

The primary Proxmox bridge is VLAN-aware.

This allows virtual machines connected to the bridge to participate in the appropriate network based on their assigned VLAN configuration.

Proxmox currently hosts infrastructure including:

* pfSense
* Wazuh
* Windows systems
* Linux systems

A future Nessus vulnerability scanner is also planned.

Because pfSense itself is virtualized on Proxmox, maintaining the correct switch trunk and Proxmox bridge configuration is essential for network availability.

---

# VLAN Design

Network segmentation is being implemented using VLANs.

| VLAN    | Name       | Purpose                             | Status   |
| ------- | ---------- | ----------------------------------- | -------- |
| VLAN 10 | LAB-LAN    | Primary lab infrastructure          | Active   |
| VLAN 20 | IOT        | Smart devices and IoT systems       | Active   |
| VLAN 99 | Management | Dedicated infrastructure management | Planned  |
| VLAN 1  | Default    | Default VLAN / limited use          | Existing |

---

## VLAN 10 — LAB-LAN

VLAN 10 contains the primary trusted infrastructure.

Examples include:

```text
pfSense       10.0.0.1
Cisco SW1     10.0.0.2
Proxmox       10.0.0.5
DC1           10.0.0.6
Wazuh01       10.0.0.7
Client01      10.0.0.100
```

This VLAN contains most of the systems used for administration, domain services, virtualization, and security monitoring.

---

## VLAN 20 — IoT

VLAN 20 was created to separate IoT and smart devices from the primary lab network.

A Cisco access port is assigned to VLAN 20 for IoT connectivity.

Example:

```text
Cisco Switch
     │
     │ Access VLAN 20
     ▼
IoT Device
```

Traffic originating from VLAN 20 is controlled through pfSense firewall rules.

This allows the lab to restrict what IoT devices can communicate with rather than placing them directly on the trusted LAB-LAN network.

Where policy-based routing is applied, outbound IoT traffic can also be directed through the Surfshark VPN connection.

---

# VLAN 99 — Management Network

VLAN 99 is reserved for a future dedicated management network.

The long-term design is to move management interfaces such as:

* Cisco switch management
* Proxmox management
* pfSense administration
* Other infrastructure management interfaces

onto a dedicated network segment.

This would further separate administrative traffic from normal workstation and server traffic.

VLAN 99 is therefore documented as:

```text
Management — Planned
```

rather than as a fully deployed management network.

---

# Active Directory Services

DC1 provides the primary Windows infrastructure services for the lab.

```text
Hostname: DC1
IP Address: 10.0.0.6
Domain: ad.home.com
```

DC1 provides:

* Active Directory Domain Services
* DNS
* DHCP
* Group Policy
* Domain authentication

Domain-joined systems use the domain controller for internal DNS resolution.

Example:

```text
Client01
   │
   ▼
DC1
10.0.0.6
   │
   ├── Active Directory
   ├── DNS
   └── DHCP
```

This allows resources such as:

```text
dc1.ad.home.com
```

to resolve internally while maintaining access to public Internet DNS through the configured forwarding path.

---

# Remote Access Architecture

pfSense hosts an OpenVPN remote-access server.

The remote-access VPN network is:

```text
10.8.0.0/24
```

The traffic flow is:

```text
Remote Device
     │
     │ Internet
     ▼
pfSense OpenVPN Server
     │
     ▼
10.8.0.0/24
     │
     ▼
Internal Lab Resources
10.0.0.0/24
```

Once authenticated, authorized VPN clients can access internal resources such as:

* pfSense
* Proxmox
* Windows Server
* Wazuh
* Domain resources
* Linux systems

This provides remote administration without exposing individual management interfaces directly to the public Internet.

---

# Outbound Surfshark VPN

A separate OpenVPN client connection is configured on pfSense for Surfshark.

This VPN serves a different purpose from the remote-access VPN.

### Remote Access OpenVPN

```text
Internet
   ↓
Remote User
   ↓
pfSense
   ↓
Home Lab
```

### Surfshark OpenVPN Client

```text
Home Lab
   ↓
pfSense
   ↓
Surfshark VPN
   ↓
Internet
```

Selected traffic can be routed through the Surfshark gateway using pfSense policy-based routing.

This allows certain networks or hosts to use the VPN for outbound Internet access while maintaining internal connectivity.

---

# Security Architecture

The network uses multiple security layers rather than relying on a single control.

These include:

### Network Segmentation

VLANs separate trusted infrastructure from IoT devices.

### Firewall Enforcement

pfSense firewall rules determine which networks and hosts are permitted to communicate.

### Remote Access VPN

Administrative resources can be accessed remotely through an encrypted VPN tunnel rather than being directly exposed to the Internet.

### Outbound VPN

Selected outbound traffic can be routed through Surfshark.

### Centralized Identity

Active Directory provides centralized authentication and Group Policy management.

### Centralized Monitoring

Wazuh receives telemetry from systems throughout the environment for security monitoring and detection.

---

# Example Traffic Paths

## Domain Client to Active Directory

```text
Client01
10.0.0.100
     │
     ▼
Cisco SW1
     │
     ▼
DC1
10.0.0.6
```

This traffic supports:

* Authentication
* DNS
* Group Policy
* File sharing
* Domain services

---

## Client Internet Traffic

```text
Client01
     │
     ▼
Cisco SW1
     │
     ▼
pfSense
     │
     ▼
Internet / Configured VPN Gateway
```

pfSense determines how the traffic is routed based on the firewall rule and gateway policy applied to the source network.

---

## IoT Internet Traffic

```text
IoT Device
     │
     ▼
VLAN 20
     │
     ▼
Cisco SW1
     │
     ▼
pfSense
     │
     ├── Firewall Rules
     │
     └── Policy-Based Routing
              │
              ▼
        Surfshark VPN
              │
              ▼
           Internet
```

---

## Remote Administration

```text
Remote Administrator
        │
        ▼
     Internet
        │
        ▼
pfSense OpenVPN
        │
        ▼
   10.8.0.0/24
        │
        ▼
Internal Resources
```

---

# Design Decisions

Several architectural decisions were made intentionally during this project.

### Centralized Routing Through pfSense

Using pfSense as the primary gateway provides a single point for firewall enforcement, VPN configuration, routing, and traffic analysis.

### Virtualized Infrastructure

Hosting services on Proxmox reduces the amount of physical hardware required while allowing multiple operating systems and security platforms to operate simultaneously.

### VLAN Segmentation

VLANs provide separation between different device categories without requiring completely separate physical switches.

### Dedicated Domain Services

Using Windows Server for Active Directory, DNS, and DHCP provides experience with centralized enterprise identity and management.

### Centralized Security Monitoring

Wazuh provides a central location for collecting and analyzing security telemetry from Windows and Linux endpoints.

---

# Troubleshooting Considerations

Because several services depend on one another, failures can affect multiple parts of the environment.

Examples encountered during this project include:

* DNS failures causing apparent Internet outages
* Incorrect pfSense gateway policies interrupting connectivity
* VLAN configuration issues preventing IoT Internet access
* VPN routing rules interfering with internal connectivity
* Proxmox outages affecting virtualized infrastructure
* Power outages disrupting VM startup order
* Switch trunk configuration affecting pfSense and VM connectivity

Troubleshooting typically follows the network path from the client toward the destination:

```text
Client
  ↓
IP Configuration
  ↓
Switch / VLAN
  ↓
Default Gateway
  ↓
pfSense Firewall
  ↓
Routing / VPN
  ↓
DNS
  ↓
Destination
```

This approach helps isolate whether a problem exists at the endpoint, switching layer, firewall, routing layer, DNS layer, or external network.

---

# Current Architecture Status

### Implemented

* pfSense routing and firewall
* Cisco managed switching
* Proxmox virtualization
* VLAN 10 LAB-LAN
* VLAN 20 IoT
* Windows Server Active Directory
* DNS
* DHCP
* Wazuh SIEM
* OpenVPN remote access
* Surfshark OpenVPN client
* Policy-based routing
* Domain-joined Windows clients

### Planned / Future Expansion

* VLAN 99 dedicated management network
* Nessus vulnerability scanner
* Additional network segmentation
* Additional security monitoring
* Expanded detection engineering
* More controlled attack simulations

---

# Skills Demonstrated

This portion of the home lab demonstrates practical experience with:

* TCP/IP networking
* IPv4 addressing
* Default gateways
* Cisco IOS
* VLANs
* Access ports
* 802.1Q trunking
* Proxmox networking
* Virtual switches and bridges
* pfSense
* Firewall rules
* NAT
* Inter-VLAN routing
* Policy-based routing
* OpenVPN
* Active Directory networking
* DNS
* DHCP
* Network segmentation
* Security monitoring
* Network troubleshooting

---

[← Back to Main Project](../README.md)
