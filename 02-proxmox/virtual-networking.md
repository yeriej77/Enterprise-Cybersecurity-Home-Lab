# Proxmox Virtual Networking

## Overview

Proxmox virtual networking connects the physical Cisco switching infrastructure to the virtual machines running on the Dell PowerEdge R710.

The design allows multiple VLANs to travel over a single physical connection between the Cisco switch and the Proxmox host.

The key components are:

- Cisco Catalyst switch
- 802.1Q trunk
- Proxmox physical network interface
- VLAN-aware Linux bridge
- Virtual network interfaces
- pfSense virtual firewall
- Virtual servers and workstations

The primary Proxmox bridge is:

```text
vmbr0
```

and is configured as VLAN-aware.

---

# High-Level Architecture

The networking path can be viewed as:

```text
                  Cisco SW1
                  10.0.0.2
                      │
                      │ Gi1/0/47
                      │ 802.1Q Trunk
                      │
                      │ VLAN 1
                      │ VLAN 10
                      │ VLAN 20
                      │ VLAN 99
                      │
                      ▼
               Proxmox Host
                10.0.0.5
                      │
                      ▼
                    vmbr0
              VLAN-Aware Bridge
                      │
          ┌───────────┼────────────┐
          │           │            │
          ▼           ▼            ▼
       pfSense       DC1         Wazuh
```

This design allows physical and virtual systems to participate in the same logical networks.

---

# Cisco-to-Proxmox Trunk

The primary connection between the Cisco switch and Proxmox uses:

```text
Interface: Gi1/0/47
Mode:      802.1Q Trunk
```

The trunk carries:

```text
VLAN 1
VLAN 10
VLAN 20
VLAN 99
```

VLAN 10 is configured as the native VLAN.

The Cisco configuration is conceptually:

```cisco
interface GigabitEthernet1/0/47
 description Proxmox-Trunk
 switchport mode trunk
 switchport trunk native vlan 10
 switchport trunk allowed vlan 1,10,20,99
```

This single link allows the Proxmox host to receive traffic belonging to multiple logical networks.

---

# Why a Trunk Is Required

Without VLAN trunking, multiple physical connections could be required to support separate networks.

For example:

```text
Physical NIC 1 → LAB-LAN
Physical NIC 2 → IOT
Physical NIC 3 → Management
```

Using an 802.1Q trunk allows these networks to share one physical link:

```text
                  Physical Link
                       │
        ┌──────────────┼──────────────┐
        │              │              │
      VLAN 10        VLAN 20        VLAN 99
      LAB-LAN           IOT         Management
```

The VLAN identifier allows network equipment to determine which logical network each Ethernet frame belongs to.

---

# Proxmox Linux Bridge

Proxmox uses Linux bridges to connect virtual machines to physical networks.

The primary bridge in this environment is:

```text
vmbr0
```

Conceptually, `vmbr0` behaves like a virtual Ethernet switch.

```text
Physical NIC
     │
     ▼
   vmbr0
     │
     ├── VM Network Interface
     ├── VM Network Interface
     ├── VM Network Interface
     └── Proxmox Management
```

Virtual machines do not need to connect directly to a physical Ethernet port.

Instead, their virtual NICs attach to the bridge.

---

# VLAN-Aware Bridge

`vmbr0` is configured as VLAN-aware.

This allows it to transport 802.1Q-tagged frames.

Conceptually:

```text
Cisco Trunk
     │
     ▼
Physical NIC
     │
     ▼
   vmbr0
VLAN-Aware
     │
     ├── VLAN 10
     ├── VLAN 20
     └── VLAN 99
```

Without VLAN awareness, tagged traffic could not be handled correctly across the virtual networking environment.

---

# Example Proxmox Bridge Configuration

The exact physical Linux interface name depends on the server hardware and Proxmox configuration.

A typical Proxmox VLAN-aware bridge follows this structure:

```text
auto vmbr0
iface vmbr0 inet static
    address 10.0.0.5/24
    gateway 10.0.0.1
    bridge-ports <physical-interface>
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
```

The important configuration concepts are:

```text
bridge-ports
```

which connects the physical NIC to the bridge, and:

```text
bridge-vlan-aware yes
```

which enables VLAN-aware forwarding.

The exact physical NIC name should be taken directly from the running Proxmox configuration rather than assumed.

---

# Proxmox Management Address

The Proxmox host currently uses:

```text
Hostname:         pve
IP Address:       10.0.0.5
Subnet:           10.0.0.0/24
Default Gateway:  10.0.0.1
Management VLAN:  VLAN 10
```

The management interface is accessible at:

```text
https://10.0.0.5:8006
```

The current design places Proxmox management traffic on the LAB-LAN.

A future goal is to migrate infrastructure management to:

```text
VLAN 99 — Management
```

---

# Native VLAN 10

The Cisco trunk currently uses:

```text
Native VLAN: 10
```

This means untagged traffic crossing the trunk is associated with VLAN 10.

Conceptually:

```text
Untagged Frame
      │
      ▼
Cisco Trunk
Native VLAN 10
      │
      ▼
LAB-LAN
```

The Proxmox management interface currently uses this network.

---

# Tagged vs Untagged Traffic

The trunk carries both native and tagged traffic.

## Native Traffic

VLAN 10 is currently the native VLAN.

```text
VLAN 10
   │
   └── Untagged across native trunk path
```

## Tagged Traffic

Other VLAN traffic is identified using an 802.1Q VLAN tag.

```text
VLAN 20
   │
   └── 802.1Q Tagged

VLAN 99
   │
   └── 802.1Q Tagged
```

This distinction is important when troubleshooting communication between:

- Cisco
- Proxmox
- pfSense
- Virtual machines

---

# Virtual Machine Network Interfaces

Every Proxmox VM can have one or more virtual network interfaces.

Conceptually:

```text
Virtual Machine
      │
      ▼
Virtual NIC
      │
      ▼
   vmbr0
      │
      ▼
Physical Network
```

The virtual NIC behaves like an Ethernet adapter from the guest operating system's perspective.

For example, Windows Server sees a normal network adapter even though it is connected through the Proxmox virtual bridge.

---

# pfSense Virtual Networking

pfSense is one of the most important VMs in the environment because it provides routing and firewall services.

Its networking responsibilities include:

- WAN connectivity
- LAN connectivity
- Inter-VLAN routing
- Firewall enforcement
- NAT
- OpenVPN remote access
- Surfshark VPN routing

Conceptually:

```text
                   pfSense VM
                       │
          ┌────────────┴────────────┐
          │                         │
         WAN                       LAN
          │                         │
      Internet                 Internal Trunk
                                    │
                          ┌─────────┼─────────┐
                          │         │         │
                       VLAN 10   VLAN 20   VLAN 99
```

The exact names of pfSense virtual adapters depend on how the VM is configured, so interface names should be documented from the live pfSense configuration rather than assumed.

---

# pfSense and VLAN Routing

pfSense performs Layer 3 routing between VLAN networks.

The general traffic path is:

```text
VLAN 20 Device
      │
      ▼
Cisco SW1
      │
      ▼
802.1Q Trunk
      │
      ▼
Proxmox vmbr0
      │
      ▼
pfSense
      │
   Firewall
      │
      ▼
Destination Network
```

The switch provides Layer 2 transport.

Proxmox provides virtual Layer 2 transport.

pfSense provides Layer 3 routing and firewall enforcement.

---

# Separation of Responsibilities

Understanding which device performs each function is important.

| Component | Primary Responsibility |
|---|---|
| Cisco SW1 | Physical switching and VLAN transport |
| Proxmox `vmbr0` | Virtual switching and VLAN transport |
| pfSense | Routing, firewalling, NAT, and VPN |
| DC1 | AD, DNS, DHCP |
| Guest VM | Endpoint/server networking |

A connectivity issue can therefore originate at several different layers.

---

# Example VLAN 20 Traffic Path

An IoT device on VLAN 20 follows a path similar to:

```text
IoT Device
    │
    ▼
Cisco Access Port
VLAN 20
    │
    ▼
Cisco SW1
    │
    ▼
Gi1/0/47 Trunk
    │
    │ Tagged VLAN 20
    ▼
Proxmox Physical NIC
    │
    ▼
vmbr0
    │
    ▼
pfSense
    │
    ├── Firewall Rules
    ├── Routing
    └── Gateway Policy
            │
            ▼
         Internet
```

If Surfshark policy-based routing is enabled for the network, traffic can then exit through the configured VPN gateway.

---

# Example LAB-LAN Traffic

A LAB-LAN system such as Client01 can communicate with DC1 without requiring inter-VLAN routing because both systems belong to the same logical network.

```text
Client01
   │
   ▼
Cisco SW1
   │
   ▼
VLAN 10
   │
   ▼
DC1
```

Traffic destined outside the local subnet is sent to:

```text
10.0.0.1
```

which is pfSense.

---

# Physical and Virtual Switching Together

One of the most important concepts in the environment is that the physical and virtual switches work together.

```text
Physical Network
      │
      ▼
 Cisco SW1
      │
      ▼
802.1Q Trunk
      │
      ▼
Physical NIC
      │
      ▼
Virtual Network
      │
      ▼
   vmbr0
      │
      ▼
Virtual Machines
```

A configuration mismatch at either layer can break communication.

---

# VLAN Configuration Must Match

The VLAN design must remain consistent across:

```text
Cisco
  │
  ▼
Proxmox
  │
  ▼
pfSense
```

For example, if VLAN 20 exists on pfSense but is not allowed on the Cisco trunk:

```text
pfSense VLAN 20
       │
       ▼
Proxmox
       │
       ▼
Cisco Trunk
       X
VLAN 20 Not Allowed
```

the traffic cannot reach the physical VLAN 20 device.

Likewise, if the Cisco switch sends tagged frames but Proxmox is not configured to transport them correctly, connectivity can fail before traffic reaches pfSense.

---

# Troubleshooting Philosophy

Virtual networking problems can be confusing because a single traffic flow crosses both physical and virtual infrastructure.

I troubleshoot the environment one layer at a time.

```text
Endpoint
   │
   ▼
IP Configuration
   │
   ▼
Physical / Virtual NIC
   │
   ▼
VLAN Assignment
   │
   ▼
Switch Port
   │
   ▼
Cisco Trunk
   │
   ▼
Proxmox Bridge
   │
   ▼
pfSense Interface
   │
   ▼
Firewall Rule
   │
   ▼
Routing
```

This avoids randomly changing firewall or VLAN settings without first identifying where the traffic stops.

---

# Proxmox Connectivity Checks

On the Proxmox host, basic network configuration can be inspected using:

```bash
ip addr
```

This shows:

- Physical interfaces
- Linux bridges
- IP addresses
- Interface state

---

## Check Routing

```bash
ip route
```

The host should have a route for the local network and a default route through:

```text
10.0.0.1
```

---

## Check Bridge Information

Useful commands include:

```bash
bridge link
```

and:

```bash
bridge vlan show
```

These can help verify bridge membership and VLAN handling.

---

# Proxmox Network Configuration File

The persistent Linux networking configuration can be inspected with:

```bash
cat /etc/network/interfaces
```

This is one of the most important files when troubleshooting Proxmox networking.

It contains configuration for:

- Physical interfaces
- Linux bridges
- Static addresses
- Gateways
- VLAN awareness

Changes to this file should be made carefully because incorrect networking configuration can make the Proxmox management interface unreachable.

---

# Cisco Verification

The physical trunk can be verified from SW1 using:

```cisco
show interfaces trunk
```

The expected trunk should include:

```text
Gi1/0/47
```

and show:

```text
Native VLAN: 10
Allowed VLANs: 1,10,20,99
```

---

## Verify VLANs

```cisco
show vlan brief
```

This confirms that the VLANs exist on the switch.

---

## Inspect the Trunk Interface

```cisco
show running-config interface GigabitEthernet1/0/47
```

This verifies the actual trunk configuration.

---

# pfSense Verification

If the physical and Proxmox layers appear correct, troubleshooting continues inside pfSense.

Items to verify include:

- Interface status
- VLAN configuration
- Assigned interfaces
- Interface IP addresses
- Firewall rules
- Gateway selection
- NAT
- Routing table

A VLAN can exist correctly at the switching layer while still failing because the corresponding pfSense interface or firewall rule is incorrect.

---

# Example Troubleshooting Scenario

Suppose an IoT device on VLAN 20 cannot reach the Internet.

Instead of immediately changing firewall rules, the troubleshooting process should verify the entire path.

```text
1. Does the IoT device have an IP address?
               │
               ▼
2. Is the switch port assigned to VLAN 20?
               │
               ▼
3. Does VLAN 20 exist on SW1?
               │
               ▼
4. Is VLAN 20 allowed on Gi1/0/47?
               │
               ▼
5. Is vmbr0 VLAN-aware?
               │
               ▼
6. Does VLAN 20 reach pfSense?
               │
               ▼
7. Does pfSense have the correct interface?
               │
               ▼
8. Is there an allow rule?
               │
               ▼
9. Is the correct gateway selected?
               │
               ▼
10. Is NAT functioning?
```

This layered approach helps isolate the actual failure.

---

# Real Troubleshooting Experience

During development of the lab, network connectivity problems occurred where virtual machines could run correctly but could not communicate with the network.

These situations required checking both:

```text
Physical Network
```

and:

```text
Virtual Network
```

rather than assuming that a running VM automatically had working connectivity.

Troubleshooting included validating:

- Cisco VLAN configuration
- Trunk settings
- Proxmox bridge configuration
- VM network adapters
- pfSense firewall rules
- Routing
- DNS

This reinforced the importance of understanding the complete packet path.

---

# Management Interface Recovery

Another important troubleshooting scenario occurred after power outages and host networking problems.

At one point, the Proxmox server was reachable through an unexpected address rather than its normal:

```text
10.0.0.5
```

management address.

Recovering the environment required validating:

- Proxmox interface configuration
- Bridge configuration
- pfSense status
- Cisco connectivity
- Default gateway configuration

This highlighted how dependent remote hypervisor administration is on correct virtual and physical networking.

---

# Virtualized Firewall Considerations

Running pfSense on the same Proxmox server introduces unique considerations.

The hypervisor needs networking in order to be administered normally, while the virtualized firewall provides much of the network's routing.

Conceptually:

```text
Proxmox
   │
   └── Hosts pfSense
           │
           └── Provides network gateway
```

This creates a dependency loop that must be considered during:

- Reboots
- Upgrades
- Power outages
- Network configuration changes

Console access to the Proxmox server can therefore be extremely valuable when network access is unavailable.

---

# Configuration Safety

Network changes on the Proxmox host should be performed carefully.

A configuration mistake involving:

```text
vmbr0
```

can potentially disconnect:

- Proxmox management
- pfSense
- DC1
- Wazuh
- Nessus
- Other virtual machines

Before significant network changes, it is useful to:

- Review the current configuration
- Take appropriate VM snapshots where relevant
- Save known-good configuration information
- Ensure console access is available
- Make one change at a time
- Test connectivity immediately

---

# Security Benefits

The virtual networking architecture supports several security controls.

## VLAN Segmentation

Different device categories can operate in separate Layer 2 networks.

## Central Firewall Enforcement

Inter-VLAN communication passes through pfSense.

## IoT Isolation

IoT devices can be separated from trusted infrastructure.

## VPN Integration

pfSense can provide both:

```text
Remote-Access VPN
```

and:

```text
Outbound Surfshark VPN
```

## Future Management Isolation

VLAN 99 can eventually isolate infrastructure administration from normal endpoint traffic.

---

# Security Limitation

The Proxmox management interface currently resides on VLAN 10.

This means management traffic is not yet fully separated from other trusted LAB-LAN systems.

The planned design is:

```text
VLAN 10
LAB-LAN
     │
     X
Direct Management Access
     │
     ▼
VLAN 99
Management
     │
     ├── Proxmox
     ├── pfSense
     └── Cisco SW1
```

Only authorized administrative systems should eventually be permitted to access the management network.

---

# Why the Design Matters

The virtual networking layer demonstrates how physical and virtual infrastructure interact.

A packet may travel through:

```text
Windows VM
    │
    ▼
Virtual NIC
    │
    ▼
Proxmox Bridge
    │
    ▼
Physical NIC
    │
    ▼
Cisco Switch
    │
    ▼
pfSense
```

or in another scenario:

```text
Physical IoT Device
       │
       ▼
Cisco Switch
       │
       ▼
Proxmox Bridge
       │
       ▼
pfSense VM
```

Understanding both directions is essential for troubleshooting virtualized network infrastructure.

---

# What I Learned

Building and troubleshooting the Proxmox networking environment reinforced several concepts.

I gained practical experience with:

- Linux bridges
- VLAN-aware bridges
- 802.1Q VLAN tagging
- Native VLANs
- Cisco trunk ports
- Virtual NICs
- Virtualized firewalls
- Inter-VLAN routing
- Linux networking commands
- Physical-to-virtual packet flow

One of the most important lessons was:

> A virtual machine can be running perfectly while its network path is still broken.

Successful communication depends on every layer being configured correctly:

```text
Guest OS
   ↓
Virtual NIC
   ↓
Proxmox Bridge
   ↓
Physical NIC
   ↓
Cisco Switch
   ↓
VLAN
   ↓
pfSense
   ↓
Firewall / Routing
```

This layered understanding significantly improved my ability to troubleshoot network outages.

---

# Virtual Networking Summary

```text
                    Cisco SW1
                    10.0.0.2
                        │
                   Gi1/0/47
                 802.1Q Trunk
                        │
          VLAN 1, VLAN 10, VLAN 20, VLAN 99
                        │
                        ▼
                Dell PowerEdge R710
                        │
                        ▼
                    Proxmox
                    10.0.0.5
                        │
                        ▼
                      vmbr0
                VLAN-Aware Bridge
                        │
           ┌────────────┼─────────────┐
           │            │             │
           ▼            ▼             ▼
        pfSense        DC1          Wazuh
        10.0.0.1     10.0.0.6     10.0.0.7
           │
           ▼
   Routing / Firewall
           │
     ┌─────┼─────┐
     │     │     │
   VLAN10 VLAN20 VLAN99
```

The combination of Cisco trunking, Proxmox VLAN-aware bridging, and pfSense routing provides the foundation for network segmentation throughout the home lab.

---

# Related Documentation

[← Proxmox Virtualization Architecture](proxmox-architecture.md)

[← Cisco Switch Configuration](../01-network-infrastructure/cisco-switch.md)

[← VLAN Design and Network Segmentation](../01-network-infrastructure/vlan-design.md)

[← Network Architecture](../01-network-infrastructure/network-architecture.md)

[← Main Project README](../README.md)
