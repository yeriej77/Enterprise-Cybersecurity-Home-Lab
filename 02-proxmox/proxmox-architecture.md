# Proxmox Virtualization Architecture

## Overview

The core compute platform for the home lab is a **Dell PowerEdge R710** running **Proxmox Virtual Environment (Proxmox VE)**.

Proxmox provides the virtualization layer that allows multiple infrastructure, security, and Windows systems to operate on a single physical server.

The environment uses Proxmox to host systems such as:

- pfSense
- Windows Server 2025 Domain Controller
- Wazuh SIEM
- Tenable Nessus
- Windows client systems
- Linux systems

This design allows the lab to simulate a small enterprise environment without requiring a separate physical server for every service.

---

# Physical Host

The virtualization platform runs on:

| Component | Value |
|---|---|
| Server | Dell PowerEdge R710 |
| Hypervisor | Proxmox VE |
| Current Platform | Proxmox VE 9 |
| Hostname | `pve` |
| Management IP | `10.0.0.5` |
| Default Gateway | `10.0.0.1` |
| Primary Network | VLAN 10 — LAB-LAN |

The Proxmox management interface is available over HTTPS on the standard Proxmox management port:

```text
https://10.0.0.5:8006
```

Access to this interface provides centralized management of the virtualized environment.

---

# Role of Proxmox

Proxmox acts as the compute layer for most of the home lab.

Conceptually:

```text
Dell PowerEdge R710
        │
        ▼
   Proxmox VE
    10.0.0.5
        │
        ├── pfSense
        │
        ├── DC1
        │
        ├── Wazuh
        │
        ├── Nessus
        │
        ├── Windows Clients
        │
        └── Linux Systems
```

Instead of each service requiring dedicated physical hardware, Proxmox provides isolated virtual machines that share the physical server's:

- CPU
- Memory
- Storage
- Network interfaces

---

# Virtual Machine Infrastructure

Several major lab services are virtualized on the Proxmox host.

| Virtual Machine | IP Address | Primary Function |
|---|---:|---|
| pfSense | `10.0.0.1` | Router / Firewall / VPN |
| DC1 | `10.0.0.6` | Active Directory / DNS / DHCP |
| Wazuh | `10.0.0.7` | SIEM / Security Monitoring |
| Nessus | `10.0.0.8` | Vulnerability Management |
| Client01 | DHCP | Domain-Joined Windows Client |

Additional Linux and testing systems can be deployed as required.

---

# Virtualized pfSense

One of the most important architectural decisions in the lab was moving pfSense from a standalone installation into a Proxmox virtual machine.

pfSense now performs several network security functions while running as a VM:

- Default gateway
- Firewall
- NAT
- Inter-VLAN routing
- OpenVPN remote access
- Surfshark VPN connectivity
- Policy-based routing
- DNS-related network services

Logically, pfSense sits at the network edge:

```text
Internet
   │
   ▼
pfSense VM
10.0.0.1
   │
   ▼
Internal Network
```

Physically, however, pfSense runs inside the Proxmox server:

```text
Dell R710
   │
   ▼
Proxmox
   │
   ▼
pfSense VM
```

This distinction is important when understanding the network architecture.

---

# Virtualization Dependency

Virtualizing the firewall creates an important infrastructure dependency.

If the Proxmox host is unavailable:

```text
Proxmox Down
     │
     ▼
pfSense VM Down
     │
     ▼
Routing / Firewall Unavailable
     │
     ▼
Lab Network Connectivity Impacted
```

Because pfSense provides the default gateway for the environment, the health of the Proxmox host directly affects network availability.

This makes:

- Host stability
- VM startup order
- Backups
- Snapshots
- Power recovery

especially important.

---

# Proxmox Networking

Proxmox connects to the Cisco switch using an 802.1Q trunk.

The Cisco trunk carries:

```text
VLAN 1
VLAN 10
VLAN 20
VLAN 99
```

VLAN 10 is currently configured as the native VLAN.

The high-level connection is:

```text
Cisco SW1
    │
    │ 802.1Q Trunk
    │
    │ VLAN 1
    │ VLAN 10
    │ VLAN 20
    │ VLAN 99
    ▼
Proxmox
```

This trunk allows multiple logical networks to reach virtual machines through a single physical connection.

---

# VLAN-Aware Bridge

The primary Proxmox network bridge is configured as VLAN-aware.

Conceptually:

```text
Physical NIC
     │
     ▼
   vmbr0
VLAN-Aware Bridge
     │
     ├── VLAN 10
     ├── VLAN 20
     └── VLAN 99
```

The bridge behaves similarly to a virtual switch inside the hypervisor.

Virtual machines connect to the bridge through virtual network adapters.

This allows different VMs to participate in different logical networks without requiring a dedicated physical NIC for every VLAN.

---

# Proxmox Bridge Concept

A simplified view of the virtual networking architecture is:

```text
                 Cisco SW1
                     │
                802.1Q Trunk
                     │
                     ▼
              Physical NIC
                     │
                     ▼
                  vmbr0
            VLAN-Aware Bridge
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
     pfSense        DC1         Wazuh
```

The physical switch handles VLAN transport into the server.

Proxmox then provides virtual Layer 2 connectivity to the VMs.

---

# Why VLAN-Aware Networking Matters

Without VLAN-aware virtual networking, each logical network could require a separate physical connection.

Using VLAN tagging allows:

```text
One Physical Connection
          │
          ▼
Multiple Logical Networks
          │
     ┌────┼────┐
     │    │    │
    V10  V20  V99
```

This provides:

- Better hardware utilization
- Easier network expansion
- Simplified cabling
- Support for virtualized firewalls
- Greater flexibility when creating new VMs

It also closely resembles virtualization architectures used in enterprise environments.

---

# Management Network

The Proxmox host currently uses:

```text
IP Address:      10.0.0.5
Subnet:          10.0.0.0/24
Default Gateway: 10.0.0.1
```

The management interface currently resides on VLAN 10.

A future improvement is to move infrastructure management onto:

```text
VLAN 99 — Management
```

This would help separate hypervisor administration from the primary LAB-LAN network.

---

# Proxmox and Active Directory

DC1 runs as a Windows Server virtual machine on Proxmox.

DC1 provides:

- Active Directory Domain Services
- DNS
- DHCP
- Group Policy
- Domain authentication

The architecture therefore contains another important dependency:

```text
Proxmox
   │
   ▼
DC1 VM
   │
   ├── DNS
   ├── DHCP
   ├── Authentication
   └── Group Policy
```

If DC1 fails to start after a host outage, domain services may remain unavailable even if Proxmox itself is operational.

---

# Security Infrastructure on Proxmox

The hypervisor also hosts dedicated security platforms.

## Wazuh

```text
Hostname: wazuh01
IP:       10.0.0.7
Role:     SIEM
```

Wazuh provides:

- Centralized log collection
- Windows event monitoring
- Sysmon telemetry
- Security alerts
- Custom detection rules

---

## Tenable Nessus

```text
IP:   10.0.0.8
Role: Vulnerability Scanner
```

Nessus provides vulnerability-management capabilities for the lab.

It is used to evaluate systems such as Windows endpoints and identify security weaknesses that can then be remediated and rescanned.

Hosting both Wazuh and Nessus on Proxmox provides two complementary security capabilities:

```text
Wazuh
  │
  └── Detect activity and security events

Nessus
  │
  └── Identify vulnerabilities and misconfigurations
```

---

# Resource Allocation

Because multiple infrastructure systems share a single physical host, resource allocation is important.

Each VM receives assigned resources such as:

- Virtual CPUs
- Memory
- Storage
- Network adapters

The combined allocation must remain within the capacity of the Dell R710.

Conceptually:

```text
Dell R710 Resources
       │
       ├── pfSense
       ├── DC1
       ├── Wazuh
       ├── Nessus
       └── Client Systems
```

Poor resource planning could cause one workload to negatively affect other services.

---

# Memory Ballooning

Proxmox supports memory ballooning for compatible virtual machines.

Memory ballooning allows the hypervisor to dynamically adjust the amount of memory available to a VM between configured minimum and maximum values.

Conceptually:

```text
Configured Maximum RAM
        │
        ▼
        VM
        ▲
        │
Configured Minimum RAM
```

This allows memory to be used more efficiently when multiple VMs are running.

During the project, VM memory settings were reviewed after the physical host began experiencing high RAM utilization.

This provided practical experience with:

- VM memory allocation
- Ballooning devices
- Minimum memory settings
- Host-level resource pressure
- Balancing multiple workloads

---

# Snapshots

Snapshots are used before major infrastructure changes.

Examples include snapshots taken before or after:

- Windows Server configuration
- Group Policy changes
- Wazuh deployment
- Sysmon installation
- Detection-rule changes
- Major application installation
- System upgrades

A snapshot provides a rollback point if a configuration change causes unexpected problems.

Conceptually:

```text
Known Good State
      │
      ▼
   Snapshot
      │
      ▼
Configuration Change
      │
      ├── Successful → Continue
      │
      └── Failed → Roll Back
```

This is particularly useful in a learning environment where configuration changes are performed frequently.

---

# Snapshots vs Backups

Snapshots and backups serve different purposes.

A snapshot is useful for short-term rollback during testing.

A backup is intended for longer-term recovery if:

- Storage fails
- A VM becomes corrupted
- The host is lost
- A VM is accidentally deleted

A mature virtualization strategy should use both.

```text
Snapshots
   │
   └── Short-term rollback

Backups
   │
   └── Disaster recovery
```

Snapshots should not be treated as a replacement for independent backups.

---

# Proxmox Upgrade

The environment was upgraded to **Proxmox VE 9**.

The upgrade provided hands-on experience with:

- Linux package management
- Debian repositories
- Proxmox repositories
- Package upgrades
- Kernel changes
- Dependency cleanup
- Service validation
- Host reboot procedures

After the upgrade, package status was verified using:

```bash
apt update
```

and system health was checked using:

```bash
systemctl --failed
```

A healthy result showed:

```text
0 loaded units listed.
```

This confirmed that no systemd services remained in a failed state after the upgrade.

---

# Package Maintenance

Linux package maintenance is also performed directly on the Proxmox host.

Examples include:

```bash
apt update
apt upgrade
apt autoremove
```

These commands help:

- Refresh repository metadata
- Install security updates
- Install Proxmox updates
- Remove unused dependencies
- Remove superseded kernels when appropriate

Package cleanup is performed carefully because the hypervisor is a critical infrastructure component.

---

# Power Outage Recovery

The home lab has experienced unexpected power outages.

One outage resulted in several infrastructure issues, including:

- Virtual machines failing to start normally
- DC1 remaining offline
- Proxmox web management becoming temporarily unavailable
- Network troubleshooting becoming more complicated because pfSense is virtualized

The recovery process highlighted the importance of understanding infrastructure dependencies.

A simplified startup dependency is:

```text
Physical Server
      │
      ▼
Proxmox
      │
      ▼
pfSense
      │
      ├── Network Routing
      └── Internet Connectivity
      │
      ▼
DC1
      │
      ├── DNS
      ├── DHCP
      └── Active Directory
      │
      ▼
Other Services
```

---

# VM Startup Order

Because some systems depend on others, startup order is important.

A logical recovery sequence is:

```text
1. Proxmox Host
       │
       ▼
2. pfSense
       │
       ▼
3. DC1
       │
       ▼
4. Wazuh / Nessus
       │
       ▼
5. Client and Test Systems
```

This sequence ensures that networking and core identity services are available before dependent systems start.

---

# Troubleshooting Proxmox Connectivity

When the Proxmox management interface cannot be reached, troubleshooting begins with basic network connectivity.

The expected management address is:

```text
10.0.0.5
```

The troubleshooting path includes:

```text
Physical Server
      │
      ▼
Physical NIC
      │
      ▼
vmbr0
      │
      ▼
Cisco Trunk
      │
      ▼
VLAN 10
      │
      ▼
Client Network
```

Useful checks include:

```bash
ip addr
```

```bash
ip route
```

```bash
bridge link
```

```bash
systemctl status networking
```

and:

```bash
ping 10.0.0.1
```

The switch configuration should also be verified if the host cannot communicate with the network.

---

# Troubleshooting Virtual Machines

If Proxmox is reachable but an individual VM is not, the issue may exist inside the virtualization layer.

A typical troubleshooting process is:

```text
VM Running?
   │
   ▼
Virtual NIC Connected?
   │
   ▼
Correct Bridge?
   │
   ▼
Correct VLAN?
   │
   ▼
Guest IP Configuration?
   │
   ▼
Gateway Reachable?
```

This helps distinguish between:

- Hypervisor problems
- Virtual networking problems
- Guest operating system problems
- pfSense firewall problems
- Physical switch problems

---

# Virtualization Security Considerations

The Proxmox host is a high-value administrative system.

Compromise of the hypervisor could potentially provide access to multiple virtual machines.

For that reason, the long-term security architecture includes:

- Dedicated VLAN 99 management network
- Restricted administrative access
- Strong authentication
- Limited exposure of port `8006`
- Regular patching
- Backups
- Configuration snapshots
- Monitoring

The Proxmox management interface should never be unnecessarily exposed directly to the public Internet.

---

# Infrastructure Concentration Risk

Virtualization reduces hardware requirements but also concentrates multiple services onto one physical host.

For example:

```text
Dell R710 Failure
       │
       ▼
Proxmox Unavailable
       │
       ├── pfSense unavailable
       ├── DC1 unavailable
       ├── Wazuh unavailable
       ├── Nessus unavailable
       └── Client VMs unavailable
```

This represents a single point of failure.

In a production enterprise environment, this risk would typically be reduced using:

- Multiple hypervisor hosts
- High availability
- Shared or replicated storage
- Redundant networking
- Backup infrastructure

For this home lab, the single-host design provides an opportunity to understand these dependencies directly.

---

# Why Proxmox Was Chosen

Proxmox provides several capabilities useful for a cybersecurity home lab:

- KVM virtualization
- Linux container support
- Web-based administration
- Virtual networking
- VLAN-aware bridges
- Snapshots
- Backup support
- Resource management
- Console access
- Linux-based administration

It also provides experience with technologies and concepts similar to those found in enterprise hypervisors.

---

# Skills Demonstrated

This section of the project demonstrates practical experience with:

### Virtualization

- Proxmox VE
- KVM virtual machines
- Virtual machine deployment
- VM lifecycle management
- Snapshots
- Resource allocation
- Memory ballooning

### Networking

- Linux bridges
- VLAN-aware bridges
- 802.1Q trunking
- Virtual NICs
- Hypervisor networking
- Virtualized firewall architecture

### Linux Administration

- Debian-based administration
- Package management
- Systemd services
- Kernel updates
- Repository management
- Command-line troubleshooting

### Infrastructure Operations

- Hypervisor upgrades
- VM startup dependencies
- Power outage recovery
- Resource troubleshooting
- Service validation
- Infrastructure dependency mapping

---

# What I Learned

Building the Proxmox environment reinforced that virtualization is much more than creating virtual machines.

A hypervisor becomes part of the core infrastructure.

I learned how:

- Virtual networking connects directly to physical switching
- VLAN configuration must match across Cisco, Proxmox, and pfSense
- Virtualizing the firewall creates important host dependencies
- Infrastructure VM startup order matters after an outage
- Memory allocation affects the entire environment
- Snapshots are valuable before risky configuration changes
- Hypervisor upgrades require careful validation
- A single physical host can become a major point of failure
- Troubleshooting requires understanding both physical and virtual network paths

The most important lesson was that the virtualization layer connects nearly every other part of the home lab.

Problems at the Proxmox layer can appear as:

- Network failures
- DNS failures
- Authentication failures
- Security monitoring failures
- VPN failures

Understanding those dependencies makes infrastructure troubleshooting significantly easier.

---

# Proxmox Architecture Summary

```text
                    Dell PowerEdge R710
                             │
                             ▼
                         Proxmox VE 9
                          10.0.0.5
                             │
               VLAN-Aware Virtual Bridge
                             │
         ┌───────────────┬───┴────┬──────────────┐
         │               │        │              │
         ▼               ▼        ▼              ▼
      pfSense           DC1     Wazuh          Nessus
      10.0.0.1       10.0.0.6  10.0.0.7      10.0.0.8
         │               │        │              │
     Firewall          AD/DNS    SIEM       Vulnerability
     Routing            DHCP                 Management
     VPN
```

The Proxmox virtualization layer provides the compute foundation for the networking, identity, security monitoring, and vulnerability-management components of the home lab.

---

# Related Documentation

[← IP Addressing and Network Services](../01-network-infrastructure/ip-addressing.md)

[← Network Architecture](../01-network-infrastructure/network-architecture.md)

[← Main Project README](../README.md)
