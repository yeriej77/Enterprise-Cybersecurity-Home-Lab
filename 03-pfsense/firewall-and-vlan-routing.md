# pfSense Firewall Rules and VLAN Routing

## Overview

pfSense provides the primary Layer 3 routing and firewall enforcement for the segmented networks in the home lab.

The Cisco switch is responsible for placing devices into VLANs and transporting VLAN traffic, while pfSense determines whether traffic is permitted to move:

- Between VLANs
- Between internal networks
- To the Internet
- Through the Surfshark VPN
- From remote OpenVPN clients
- Toward sensitive infrastructure

This separation of responsibilities is important.

```text
Cisco Switch
     │
     │ VLAN Transport
     ▼
  Proxmox
     │
     │ Virtual Transport
     ▼
   pfSense
     │
     │ Routing + Firewall Policy
     ▼
Destination Network
```

The goal is not simply to create VLANs.

The goal is to use VLANs as **security boundaries** and enforce communication between those boundaries through pfSense firewall rules.

---

# Current Network Segmentation

The home lab currently uses the following VLAN design:

| VLAN | Name | Purpose | Status |
|---|---|---|---|
| VLAN 10 | LAB-LAN | Trusted infrastructure and lab systems | Active |
| VLAN 20 | IOT | Smart and IoT devices | Active |
| VLAN 99 | Management | Dedicated infrastructure management | Planned |

The primary LAB-LAN uses:

```text
10.0.0.0/24
```

with pfSense as the default gateway:

```text
10.0.0.1
```

The exact VLAN 20 IP subnet is intentionally not documented here until it is verified directly from the live pfSense configuration.

This avoids publishing assumed or inaccurate network information.

---

# Why VLANs Require Routing

Devices within the same VLAN can communicate at Layer 2 without sending traffic through a router.

For example:

```text
Client01
10.0.0.x
    │
    ▼
VLAN 10
    │
    ▼
DC1
10.0.0.6
```

Because both systems are on the same local network, traffic can remain within VLAN 10.

Communication between separate VLANs is different.

```text
VLAN 20 Device
      │
      ▼
Cisco SW1
      │
      ▼
pfSense
      │
      ▼
VLAN 10 Resource
```

pfSense performs the Layer 3 routing between the networks.

This means it can also apply firewall policy before forwarding the traffic.

---

# VLANs as Security Zones

The VLAN architecture is designed around different levels of trust.

```text
VLAN 10
LAB-LAN
Trusted Infrastructure
       │
       │
       ▼
    pfSense
       ▲
       │
       │
VLAN 20
IOT
Less-Trusted Devices
```

A future VLAN 99 will provide another security zone:

```text
VLAN 99
Management
Highly Restricted
```

The long-term goal is:

```text
Management
    >
Trusted Lab
    >
IoT
```

in terms of how much access each network receives.

---

# Why IoT Devices Are Segmented

IoT and smart devices are intentionally separated from the primary infrastructure network.

These systems may have:

- Limited endpoint security controls
- Proprietary operating systems
- Infrequent security updates
- Limited visibility into installed software
- Cloud-dependent services
- Limited logging
- Unknown background communications

A smart device should therefore not automatically have unrestricted access to systems such as:

```text
DC1
Proxmox
Wazuh
Nessus
pfSense Management
Cisco Management
```

VLAN 20 creates a separate Layer 2 network.

pfSense then provides the policy boundary.

---

# Firewall Enforcement Model

pfSense uses a stateful firewall.

Rules are normally applied to traffic as it **enters pfSense through an interface**.

For example:

```text
VLAN 20 Device
      │
      ▼
VLAN 20 Interface
      │
      ▼
Firewall Rules
      │
      ▼
Routing Decision
```

Therefore, rules controlling what IoT devices may initiate are typically placed on the VLAN 20 interface.

This is an important concept when designing pfSense policies.

---

# Default-Deny Concept

A secure firewall design should not assume that everything should be permitted.

Conceptually:

```text
Traffic Arrives
      │
      ▼
Does a rule allow it?
      │
   ┌──┴──┐
   │     │
  Yes    No
   │     │
   ▼     ▼
 Pass   Block
```

The design should explicitly permit required traffic rather than relying on overly broad rules.

---

# Firewall Rule Processing

Firewall rules are evaluated from top to bottom.

```text
Rule 1
  │
  ├── Match → Apply Rule
  │
  └── No Match
       │
       ▼
Rule 2
  │
  ├── Match → Apply Rule
  │
  └── No Match
       │
       ▼
Rule 3
```

This makes rule order important.

For example:

```text
PASS VLAN20 → ANY
```

placed above:

```text
BLOCK VLAN20 → LAB-LAN
```

could make the block ineffective for traffic already matched by the broader pass rule.

Rule design therefore requires both:

```text
Correct Rule
+
Correct Rule Position
```

---

# VLAN 20 Security Objective

The IoT network was designed to provide a restricted network for smart devices.

The intended security model is:

```text
IoT Device
    │
    ▼
VLAN 20
    │
    ▼
pfSense
    │
    ├── Internet access → Controlled
    │
    ├── LAB-LAN access → Restricted
    │
    └── Management access → Restricted
```

The objective is to provide the connectivity IoT devices require without giving them unnecessary access to critical systems.

---

# Initial VLAN 20 Policy

During development of the lab, VLAN 20 firewall rules were used to experiment with restricting traffic between IoT devices and other networks.

One configuration goal was to prevent unnecessary Internet access while still allowing selected internal communication.

Conceptually:

```text
VLAN 20
   │
   ├── Internal Resource → Allowed where required
   │
   └── Internet → Restricted
```

This was useful for learning how pfSense interface rules influence traffic flow.

---

# Evolving VLAN 20 Policy

The design later evolved toward allowing selected VLAN 20 Internet traffic through the Surfshark VPN.

The intended traffic path becomes:

```text
IoT Device
    │
    ▼
VLAN 20
    │
    ▼
pfSense Firewall Rule
    │
    ▼
Surfshark Gateway
    │
    ▼
OpenVPN Tunnel
    │
    ▼
Internet
```

This introduces additional components beyond a normal allow rule.

The configuration must account for:

- VLAN interface
- Firewall policy
- VPN gateway
- Policy-based routing
- NAT
- DNS
- OpenVPN tunnel status

---

# Policy-Based Routing

Normally, pfSense uses its routing table to select where traffic should go.

Policy-based routing allows a firewall rule to explicitly specify a gateway.

For example:

```text
VLAN 20 Traffic
      │
      ▼
Firewall Rule
Gateway = Surfshark
      │
      ▼
Surfshark OpenVPN
      │
      ▼
Internet
```

This makes it possible for different networks to use different outbound paths.

---

# Example Routing Strategy

Conceptually, the lab can support:

```text
VLAN 10 — LAB-LAN
      │
      ▼
pfSense
      │
      ▼
Normal ISP Gateway
      │
      ▼
Internet
```

while:

```text
VLAN 20 — IOT
      │
      ▼
pfSense
      │
      ▼
Surfshark Gateway
      │
      ▼
VPN Tunnel
      │
      ▼
Internet
```

The exact policy depends on the firewall rule assigned to each source network.

---

# Why a Connected VPN Is Not Enough

One of the important lessons from configuring Surfshark was that an OpenVPN tunnel can show:

```text
Connected
```

without client traffic actually using that tunnel.

The tunnel status only proves that:

```text
pfSense
   │
   ▼
VPN Provider
```

has established a connection.

It does not automatically prove:

```text
Client
   │
   ▼
pfSense
   │
   ▼
VPN Tunnel
```

is the active client traffic path.

To route client traffic through the VPN, the firewall policy and gateway selection must also be correct.

---

# NAT and Internet Access

Routing traffic to an outbound interface is only part of the process.

Private internal addresses must also be translated appropriately for outbound communication.

Conceptually:

```text
IoT Device
Private IP
    │
    ▼
pfSense
    │
    ├── Firewall
    ├── Routing
    └── NAT
         │
         ▼
Surfshark Tunnel
         │
         ▼
Internet
```

If routing is correct but outbound NAT is not, traffic may still fail.

---

# Firewall and NAT Are Different Functions

This distinction is important.

A firewall rule answers:

```text
Is this traffic allowed?
```

Routing answers:

```text
Where should this traffic go?
```

NAT answers:

```text
How should the source or destination address be translated?
```

All three may need to work correctly.

```text
Firewall
   +
Routing
   +
NAT
   =
Successful Traffic Flow
```

---

# Internet Access Troubleshooting

If a VLAN device can communicate internally but cannot reach the Internet, the switch is not automatically the problem.

For example:

```text
IoT Device
   │
   ├── Can reach gateway
   │
   ├── Can reach internal resource
   │
   └── Cannot reach Internet
```

This indicates that Layer 2 VLAN connectivity may already be working.

Troubleshooting should then continue toward:

```text
Firewall Rule
    │
    ▼
Gateway Selection
    │
    ▼
NAT
    │
    ▼
VPN Tunnel
    │
    ▼
DNS
```

---

# Layered Troubleshooting Process

A reliable troubleshooting sequence for VLAN connectivity is:

```text
1. Physical Connectivity
        │
        ▼
2. Switch Port VLAN
        │
        ▼
3. Cisco Trunk
        │
        ▼
4. Proxmox Bridge
        │
        ▼
5. pfSense VLAN Interface
        │
        ▼
6. Client IP Configuration
        │
        ▼
7. Default Gateway
        │
        ▼
8. Firewall Rule
        │
        ▼
9. Gateway Selection
        │
        ▼
10. NAT
        │
        ▼
11. DNS
```

This prevents unnecessary changes at the wrong layer.

---

# Example Troubleshooting Scenario

Suppose an IoT device has no Internet connectivity.

## Step 1 — Verify the Client Address

Confirm the device received the expected address for VLAN 20.

Check:

```text
IP Address
Subnet Mask
Default Gateway
DNS
```

If the client does not have correct addressing, there is no reason to start modifying firewall rules yet.

---

# Step 2 — Verify the Switch Port

The physical port used by the IoT device should be assigned to VLAN 20.

Example Cisco concept:

```cisco
interface GigabitEthernet1/0/3
 switchport mode access
 switchport access vlan 20
```

Verify using:

```cisco
show vlan brief
```

---

# Step 3 — Verify the Trunk

VLAN 20 must be transported from the Cisco switch toward Proxmox.

The trunk is:

```text
Gi1/0/47
```

Verify using:

```cisco
show interfaces trunk
```

VLAN 20 should be included in the allowed VLAN list.

---

# Step 4 — Verify Proxmox

The Proxmox bridge must transport the VLAN traffic correctly.

The primary bridge is:

```text
vmbr0
```

and is VLAN-aware.

Useful commands include:

```bash
bridge vlan show
```

and:

```bash
ip addr
```

---

# Step 5 — Verify pfSense Interface

pfSense must have the corresponding VLAN interface configured and active.

Items to verify include:

```text
VLAN Exists
Interface Assigned
Interface Enabled
Correct Addressing
Correct Parent Interface
```

The exact VLAN 20 subnet should be taken directly from the live configuration.

---

# Step 6 — Verify Gateway Reachability

The client should first be able to reach its VLAN gateway.

If the local pfSense gateway cannot be reached:

```text
Client
   X
pfSense
```

the problem exists before Internet routing.

Likely areas include:

- Addressing
- Switch VLAN
- Trunking
- Proxmox bridge
- pfSense interface configuration

---

# Step 7 — Verify Firewall Rules

If the gateway is reachable, check the VLAN 20 firewall rules.

Questions include:

```text
Is there a pass rule?

Does the source match VLAN 20?

Does the destination match?

Is the protocol correct?

Is another rule matching first?

Is a gateway assigned?
```

Rule order should also be reviewed.

---

# Step 8 — Verify Gateway Selection

If VLAN 20 should use Surfshark, confirm that the relevant pass rule uses the Surfshark gateway.

Conceptually:

```text
Source:
VLAN20 net

Destination:
Allowed destination

Action:
Pass

Gateway:
Surfshark VPN Gateway
```

The exact object names depend on the live pfSense configuration.

---

# Step 9 — Verify OpenVPN Status

Confirm that the Surfshark OpenVPN client is actually connected.

A healthy VPN connection should have:

- Tunnel established
- Remote endpoint reachable
- Gateway available
- Traffic counters changing when traffic uses the tunnel

A connected status alone should not be treated as complete validation.

---

# Step 10 — Verify NAT

If the correct gateway is selected but Internet connectivity still fails, outbound NAT should be reviewed.

Traffic originating from VLAN 20 must be translated correctly when leaving through the VPN.

The exact NAT mode may be:

```text
Automatic
Hybrid
Manual
```

depending on the configured pfSense environment.

The live configuration should be checked rather than assumed.

---

# Step 11 — Test IP Connectivity

Before blaming DNS, test an external IP address.

For example:

```text
External IP Reachable?
```

If an external IP works but websites do not resolve:

```text
Routing = Likely Working
DNS = Investigate
```

---

# DNS Considerations

DNS can complicate VLAN troubleshooting.

A device may successfully reach the Internet by IP but fail when using hostnames.

```text
Internet by IP
     │
     └── Works

Internet by Name
     │
     └── Fails
```

This strongly suggests DNS rather than routing.

IoT DNS policy should be documented based on the actual deployed configuration rather than assuming that IoT clients use DC1.

---

# Firewall Logging

Firewall logging is useful for determining whether pfSense is blocking traffic.

Logging can help identify:

- Source IP
- Destination IP
- Interface
- Protocol
- Port
- Rule responsible for the decision

Conceptually:

```text
Client Traffic
      │
      ▼
pfSense
      │
   Blocked
      │
      ▼
Firewall Log
```

This is more reliable than assuming a packet was blocked simply because an application did not work.

---

# Packet Capture

When firewall logs are not enough, packet capture can provide direct visibility.

Useful capture points may include:

```text
VLAN 20 Interface
LAN Interface
WAN Interface
Surfshark OpenVPN Interface
```

The objective is to determine where the traffic stops.

For example:

```text
Packet seen on VLAN20?
        │
        ├── No → Client / VLAN problem
        │
        └── Yes
              │
              ▼
Packet seen on VPN interface?
        │
        ├── No → Firewall / routing
        │
        └── Yes
              │
              ▼
Return traffic received?
```

This transforms troubleshooting from guessing into observation.

---

# Rule Design for IoT Isolation

A mature IoT policy could conceptually follow:

```text
Rule 1
Allow required DNS
       │
       ▼
Rule 2
Allow required internal services
       │
       ▼
Rule 3
Block access to protected internal networks
       │
       ▼
Rule 4
Allow Internet through approved gateway
```

The exact rules should reflect the actual requirements of the IoT devices.

The objective is not to create restrictions simply for the sake of restrictions.

It is to allow only what the devices need.

---

# Avoiding Overly Broad Rules

A rule such as:

```text
VLAN20 net → any
```

may be convenient during initial testing.

However, it can allow more access than intended.

A more mature policy distinguishes between:

```text
Internal Networks
Management Networks
Internet
Required Services
```

This allows the network to better follow least privilege.

---

# Future RFC1918 Blocking Strategy

One possible design pattern is to prevent a less-trusted VLAN from reaching internal private networks while still allowing Internet access.

Conceptually:

```text
VLAN 20
   │
   ├── Internal Private Networks → Block
   │
   └── Internet → Allow
```

This may be implemented using firewall aliases representing internal networks.

The exact design should be tested carefully because some IoT systems may legitimately require access to local controllers or services.

---

# Firewall Aliases

Aliases can improve firewall-rule readability and management.

Instead of repeatedly entering several individual systems:

```text
10.0.0.5
10.0.0.6
10.0.0.7
10.0.0.8
```

an alias could conceptually represent:

```text
INFRASTRUCTURE_SERVERS
```

Then a rule could read:

```text
VLAN20 → INFRASTRUCTURE_SERVERS → Block
```

This makes the firewall policy easier to understand and maintain.

Aliases will be documented only when they are actually implemented.

---

# Future Management VLAN Policy

VLAN 99 is planned for infrastructure management.

The intended access model is:

```text
Administrative Workstation
          │
          ▼
       VLAN 99
          │
          ├── Proxmox
          ├── pfSense
          ├── Cisco SW1
          └── Security Tools
```

while:

```text
VLAN 20
   │
   X
VLAN 99
```

and normal user systems would only receive management access where explicitly required.

---

# Why Management Isolation Matters

Management interfaces provide significant control over the environment.

For example:

### Proxmox

Could provide access to:

- Virtual machines
- VM consoles
- Virtual networking
- Storage
- Backups

### pfSense

Could provide control over:

- Firewall rules
- NAT
- VPNs
- VLAN routing
- DNS-related settings

### Cisco SW1

Could provide control over:

- VLANs
- Trunks
- Access ports
- Layer 2 connectivity

For this reason, management services deserve stronger segmentation than normal client traffic.

---

# Defense-in-Depth

Firewall segmentation works alongside other security controls.

```text
Cisco VLANs
      │
      ▼
pfSense Firewall
      │
      ▼
Windows Firewall
      │
      ▼
Active Directory / GPO
      │
      ▼
Sysmon / Windows Auditing
      │
      ▼
Wazuh SIEM
      │
      ▼
Nessus
```

If one security control fails, other layers can still provide protection or visibility.

---

# Security Scenario

Consider a compromised IoT device.

Without firewall segmentation:

```text
Compromised Smart Device
        │
        ├── Proxmox
        ├── DC1
        ├── Wazuh
        └── Nessus
```

With segmentation:

```text
Compromised Smart Device
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
Protected Infrastructure
```

Segmentation does not make compromise impossible.

It reduces the number of systems the compromised device can directly reach.

---

# Common Troubleshooting Mistake

A common mistake is assuming:

```text
No Internet
=
Firewall Problem
```

The real cause could instead be:

```text
Wrong IP Address
Wrong VLAN
Trunk Misconfiguration
Proxmox Bridge
Wrong Gateway
VPN Failure
NAT
DNS
```

Firewall troubleshooting should therefore always be part of a complete network-path analysis.

---

# Another Common Mistake

Another mistake is temporarily creating:

```text
Allow Any → Any
```

and leaving it in place after testing.

Broad troubleshooting rules should be:

1. Clearly identified
2. Used only when necessary
3. Tested
4. Removed or narrowed afterward

Otherwise, temporary troubleshooting can permanently weaken segmentation.

---

# Change Methodology

Firewall changes should be performed incrementally.

```text
Backup Configuration
       │
       ▼
Document Current Rules
       │
       ▼
Make One Change
       │
       ▼
Apply
       │
       ▼
Test
       │
   ┌───┴───┐
   │       │
 Works   Fails
   │       │
   ▼       ▼
Keep    Investigate
```

Making multiple simultaneous changes to:

- Firewall rules
- NAT
- VPN settings
- Gateways
- DNS

makes it difficult to identify which change fixed or caused the problem.

---

# What I Learned

Implementing VLAN routing and firewall policies reinforced the difference between:

```text
Segmentation
```

and:

```text
Security Enforcement
```

A VLAN alone creates logical separation.

pfSense firewall policy determines what that separation actually means.

I gained hands-on experience with:

- Inter-VLAN routing
- Stateful firewall rules
- Rule direction
- Rule ordering
- Network segmentation
- Policy-based routing
- VPN gateways
- NAT
- DNS troubleshooting
- Packet capture
- Layered network troubleshooting

The biggest lesson was:

> **Creating a VLAN does not automatically secure it. Security comes from controlling what traffic is allowed to cross the boundary.**

---

# Firewall and VLAN Routing Summary

```text
                         pfSense
                            │
                  Routing + Firewall
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
         ▼                  ▼                  ▼
      VLAN 10            VLAN 20           VLAN 99
      LAB-LAN              IOT            Management
       Active             Active           Planned
         │                  │                  │
         ▼                  ▼                  ▼
   Trusted Systems    Less-Trusted IoT    Admin Systems
                            │
                            ▼
                    Policy-Based Routing
                            │
                            ▼
                    Surfshark OpenVPN
                            │
                            ▼
                         Internet
```

pfSense converts the VLAN design from simple network separation into an enforceable security architecture.

---

# Current and Planned State

## Implemented

- VLAN 10 LAB-LAN
- VLAN 20 IOT
- Cisco VLAN trunking
- Proxmox VLAN-aware transport
- pfSense inter-VLAN routing
- pfSense firewall enforcement
- Surfshark OpenVPN client
- Policy-based routing experimentation
- IoT traffic restrictions

## Planned / Continuing Work

- Verify and document the final VLAN 20 addressing plan
- Complete reliable IoT Internet routing through Surfshark
- Deploy VLAN 99 management network
- Restrict infrastructure management access
- Improve firewall aliases
- Refine least-privilege policies
- Expand firewall logging and monitoring
- Document final production rule sets

---

# Related Documentation

[← pfSense Firewall and Routing Architecture](pfsense-architecture.md)

[← VLAN Design and Network Segmentation](../01-network-infrastructure/vlan-design.md)

[← Proxmox Virtual Networking](../02-proxmox/virtual-networking.md)

[← Cisco Switch Configuration](../01-network-infrastructure/cisco-switch.md)

[← Main Project README](../README.md)
