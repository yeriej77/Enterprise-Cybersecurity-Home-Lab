# Surfshark VPN and Policy-Based Routing

## Overview

The home lab uses **Surfshark OpenVPN on pfSense** as an alternative outbound Internet gateway.

This project is different from the OpenVPN remote-access server.

The remote-access VPN allows an administrator to connect **into** the home lab.

The Surfshark configuration allows selected devices or networks inside the home lab to send traffic **out through a commercial VPN provider**.

```text
Remote Access VPN
Remote User
    │
    ▼
pfSense
    │
    ▼
Home Lab


Surfshark VPN
Home Lab
    │
    ▼
pfSense
    │
    ▼
Surfshark
    │
    ▼
Internet
```

The Surfshark project provided hands-on experience with:

- OpenVPN client configuration
- VPN gateways
- Encryption compatibility
- Policy-based routing
- NAT
- Public-IP verification
- DNS troubleshooting
- VLAN routing
- Packet-path analysis

---

# Architecture

The high-level architecture is:

```text
                    Internal Networks
                           │
                           ▼
                        pfSense
                           │
                 Firewall / Routing
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
          Normal ISP Gateway     Surfshark Gateway
                │                     │
                ▼                     ▼
             Internet           OpenVPN Tunnel
                                      │
                                      ▼
                               Surfshark Server
                                      │
                                      ▼
                                   Internet
```

pfSense can decide which outbound path should be used based on firewall policy.

---

# Why Use an Outbound VPN?

The goal was not to send every system through Surfshark automatically.

Instead, the project explored using pfSense to selectively route certain devices or network segments through a VPN.

This provides a design such as:

```text
LAB-LAN
   │
   ▼
Normal ISP
   │
   ▼
Internet


IOT
   │
   ▼
pfSense
   │
   ▼
Surfshark
   │
   ▼
Internet
```

This is an example of **policy-based routing**.

---

# pfSense as an OpenVPN Client

In this configuration, pfSense acts as the VPN client.

```text
pfSense
   │
   ▼
OpenVPN Client
   │
   ▼
Surfshark VPN Endpoint
```

This differs from the home-lab remote-access configuration, where pfSense acts as the OpenVPN server.

---

# Two OpenVPN Roles

The environment therefore uses OpenVPN in two separate ways.

## OpenVPN Server

Purpose:

```text
Remote Administrator
        │
        ▼
      pfSense
        │
        ▼
     Home Lab
```

Used for:

- Remote management
- Internal resource access
- Secure administration

---

## OpenVPN Client

Purpose:

```text
Home Lab
    │
    ▼
pfSense
    │
    ▼
Surfshark
    │
    ▼
Internet
```

Used for:

- Alternative Internet egress
- VPN policy routing
- Network-based outbound traffic control

Keeping these architectures separate makes troubleshooting significantly easier.

---

# Surfshark Configuration Process

The Surfshark OpenVPN configuration was imported into pfSense and configured as an OpenVPN client.

The process involved several components:

```text
Surfshark Configuration
        │
        ▼
pfSense OpenVPN Client
        │
        ▼
VPN Tunnel
        │
        ▼
VPN Interface / Gateway
        │
        ▼
Firewall Policy
        │
        ▼
Outbound NAT
        │
        ▼
Client Traffic
```

Each component must work correctly before a device can successfully use the VPN.

---

# Initial VPN Connection Problems

The initial OpenVPN client configuration did not connect successfully.

One issue involved cipher compatibility between the Surfshark configuration and the OpenVPN implementation used by pfSense.

This required reviewing the OpenVPN configuration rather than assuming that the downloaded configuration file would work unchanged.

The troubleshooting process included:

- Reviewing OpenVPN logs
- Checking encryption settings
- Verifying the remote VPN endpoint
- Confirming protocol settings
- Restarting the OpenVPN client
- Monitoring connection status

---

# Cipher Compatibility

OpenVPN configurations depend on both sides supporting compatible cryptographic settings.

Conceptually:

```text
pfSense OpenVPN
       │
       │ Encryption Parameters
       ▼
Surfshark Server
```

If the client and server cannot agree on supported settings:

```text
VPN Tunnel
    X
Connection Fails
```

Resolving the cipher configuration allowed the client to successfully establish the Surfshark tunnel.

---

# VPN Endpoint Troubleshooting

Different Surfshark VPN endpoints were also tested during troubleshooting.

This helped determine whether the issue was related to:

- A specific VPN server
- OpenVPN configuration
- Endpoint availability
- Routing
- Encryption parameters

Testing alternate endpoints is useful because a VPN configuration can be correct while a specific remote server is unavailable or behaving differently.

---

# Successful Tunnel

After troubleshooting the configuration, pfSense successfully established the Surfshark OpenVPN connection.

The resulting path was:

```text
pfSense
   │
   ▼
Encrypted OpenVPN Tunnel
   │
   ▼
Surfshark VPN Server
   │
   ▼
Internet
```

At this point, the VPN tunnel itself was operational.

However, a connected tunnel does not automatically mean LAN or VLAN traffic is using it.

---

# The Most Important VPN Lesson

One of the most important lessons from this project was:

> **A VPN showing "connected" only confirms that pfSense can reach the VPN provider. It does not prove that client traffic is traveling through the tunnel.**

These are separate questions:

```text
Is the VPN connected?
```

and:

```text
Is my client actually using the VPN?
```

Both need to be validated independently.

---

# Creating an Outbound VPN Path

For internal clients to use Surfshark, the traffic path must include:

```text
Client
   │
   ▼
pfSense
   │
   ▼
Firewall Rule
   │
   ▼
Surfshark Gateway
   │
   ▼
Outbound NAT
   │
   ▼
OpenVPN Tunnel
   │
   ▼
Internet
```

A failure in any of these components can prevent successful VPN routing.

---

# VPN Gateway

Once the OpenVPN client is connected, pfSense can use the VPN connection as an alternative gateway.

Conceptually:

```text
Available Gateways
      │
      ├── ISP Gateway
      │
      └── Surfshark Gateway
```

Firewall policies can then determine which gateway traffic should use.

---

# Policy-Based Routing

Policy-based routing allows pfSense to route traffic according to firewall rules rather than relying only on the system's default route.

For example:

```text
Source:
VLAN 20

Action:
Pass

Gateway:
Surfshark
```

produces the logical path:

```text
VLAN 20
   │
   ▼
pfSense Firewall Rule
   │
   ▼
Surfshark Gateway
   │
   ▼
OpenVPN
   │
   ▼
Internet
```

---

# Normal Routing vs Policy Routing

Without a gateway specified in the firewall rule:

```text
Client
   │
   ▼
pfSense
   │
   ▼
Default Gateway
   │
   ▼
ISP
```

With Surfshark selected:

```text
Client
   │
   ▼
pfSense
   │
   ▼
Policy Rule
   │
   ▼
Surfshark Gateway
   │
   ▼
VPN
```

This makes it possible to control Internet egress at the network level.

---

# Public IP Verification

One of the easiest ways to verify whether traffic is actually using the VPN is to check the public IP address.

Before VPN routing:

```text
Client
   │
   ▼
ISP
   │
   ▼
ISP Public IP
```

After successful Surfshark routing:

```text
Client
   │
   ▼
pfSense
   │
   ▼
Surfshark
   │
   ▼
VPN Public IP
```

During testing, the externally visible public IP changed when traffic successfully used the Surfshark connection.

This provided evidence that the packet path had changed.

---

# Why Public IP Testing Is Useful

A tunnel status page only shows:

```text
pfSense ↔ Surfshark
```

A public-IP test validates:

```text
Client → pfSense → Surfshark → Internet
```

This is a much stronger end-to-end test.

---

# Outbound NAT

Policy routing alone may not be sufficient.

Traffic leaving through the VPN also needs appropriate NAT.

Conceptually:

```text
Internal Client
Private Address
      │
      ▼
pfSense
      │
   Outbound NAT
      │
      ▼
Surfshark Interface
      │
      ▼
Internet
```

Without correct NAT, the VPN may be connected and the gateway may be selected while client traffic still fails.

---

# Firewall, Routing, and NAT

These functions should be treated separately.

## Firewall

Answers:

```text
Is the traffic allowed?
```

## Routing

Answers:

```text
Where should the traffic go?
```

## NAT

Answers:

```text
What source address should be presented when it leaves?
```

Successful Surfshark routing therefore depends on:

```text
Firewall
   +
Routing
   +
NAT
   =
Working VPN Egress
```

---

# VLAN 20 Integration

One of the primary design goals is to use the Surfshark connection for the IoT network.

The intended design is:

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
Proxmox
    │
    ▼
pfSense
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

This allows IoT devices to use a different Internet egress path from trusted LAB-LAN infrastructure.

---

# Why Route IoT Through the VPN?

The design provides an opportunity to separate IoT traffic not only at Layer 2 but also at the Internet-routing layer.

Conceptually:

```text
Trusted Infrastructure
       │
       ▼
      ISP


IoT Devices
       │
       ▼
   Surfshark
```

This demonstrates how firewall policy can influence not just whether traffic is permitted but also **where it exits the network**.

---

# Current VLAN 20 Status

VLAN 20 itself is operational as a segmented network.

The Surfshark OpenVPN client is also operational.

The remaining work is to continue refining and validating the complete VLAN 20 Internet path through Surfshark so that:

```text
VLAN 20
   │
   ▼
pfSense
   │
   ▼
Surfshark
   │
   ▼
Internet
```

works reliably without unnecessarily exposing the IoT network to trusted infrastructure.

The configuration should only be marked complete after end-to-end testing confirms the expected behavior.

---

# Why Document Incomplete Work?

A portfolio does not need to pretend every configuration worked perfectly on the first attempt.

The troubleshooting process demonstrates:

- Problem isolation
- Network analysis
- VPN debugging
- Firewall understanding
- Iterative testing

Documenting the current state also prevents future troubleshooting from starting from incomplete assumptions.

---

# IoT Firewall Policy

The long-term goal for VLAN 20 is:

```text
IoT Devices
    │
    ├── Trusted Infrastructure → Restricted
    │
    ├── Management Network → Restricted
    │
    └── Internet → Surfshark
```

This combines:

- Segmentation
- Firewall policy
- Policy-based routing
- VPN egress

into one security design.

---

# DNS and VPN Routing

DNS must also be considered when moving traffic through a VPN.

A client can have:

```text
Working VPN Routing
```

but still fail to browse if:

```text
DNS Resolution Fails
```

Testing should therefore separate:

```text
IP Connectivity
```

from:

```text
DNS Connectivity
```

---

# Troubleshooting DNS Separately

A useful test sequence is:

```text
Can the device reach an external IP?
        │
        ├── No
        │    │
        │    ▼
        │ Routing / Firewall / NAT / VPN
        │
        └── Yes
             │
             ▼
        Can it resolve a hostname?
             │
             ├── No → DNS Problem
             │
             └── Yes → Connectivity Working
```

This avoids changing VPN routing when the actual issue is DNS.

---

# IPv4 and IPv6 Testing

VPN behavior may differ between IPv4 and IPv6.

During testing, IPv4 connectivity through the VPN was validated while IPv6 did not provide the same successful path.

This is important because a system may attempt IPv6 connectivity even when the VPN policy was designed primarily for IPv4.

The lesson is to explicitly test both protocols rather than assuming one successful test represents all traffic.

---

# Avoiding VPN Bypass

A policy-routing design should consider what happens if the Surfshark gateway becomes unavailable.

Without additional controls, traffic may potentially follow another available route depending on the firewall and gateway configuration.

For privacy-sensitive or tightly controlled networks, the desired behavior may instead be:

```text
Surfshark Available
        │
        ▼
Internet Allowed


Surfshark Unavailable
        │
        ▼
Internet Blocked
```

This is sometimes described as a VPN kill-switch style design.

Such behavior should be implemented and tested deliberately rather than assumed.

---

# Fail-Closed Concept

For a network that should only use Surfshark:

```text
VLAN 20
   │
   ▼
Surfshark Gateway
   │
   ├── UP → Internet
   │
   └── DOWN → Block
```

This prevents unintended fallback through the normal ISP path.

This remains an area for further refinement in the lab.

---

# Packet Capture

Packet capture is extremely useful when troubleshooting VPN routing.

The question becomes:

```text
Where does the packet disappear?
```

Possible capture points include:

```text
VLAN 20
   │
   ▼
pfSense VLAN Interface
   │
   ▼
Surfshark OpenVPN Interface
   │
   ▼
WAN / Internet
```

If traffic appears on VLAN 20 but never appears on the OpenVPN interface, the issue is likely related to firewall policy or routing.

---

# OpenVPN Traffic Verification

OpenVPN tunnel traffic itself can also be observed.

This verifies communication between:

```text
pfSense
```

and:

```text
Surfshark VPN Server
```

Seeing VPN packets confirms the tunnel endpoint is communicating, but client traffic still needs separate verification.

---

# Troubleshooting Flow

If a client should use Surfshark but has no Internet:

```text
Client Has Correct IP?
        │
        ▼
Gateway Reachable?
        │
        ▼
Firewall Rule Matches?
        │
        ▼
Surfshark Gateway Selected?
        │
        ▼
OpenVPN Connected?
        │
        ▼
Outbound NAT Correct?
        │
        ▼
Packets Enter VPN?
        │
        ▼
Return Traffic Received?
        │
        ▼
DNS Working?
```

This provides a repeatable troubleshooting method.

---

# Example Failure Scenario

Suppose:

```text
Surfshark = Connected
```

but a client still shows the ISP public address.

That indicates:

```text
Tunnel Works
```

but:

```text
Policy Routing Does Not
```

Troubleshooting should then focus on:

- Firewall rule
- Rule order
- Gateway selection
- Source network
- NAT

rather than rebuilding the VPN tunnel.

---

# Example Failure Scenario 2

Suppose the public IP changes to Surfshark but websites fail by name.

That indicates:

```text
VPN Routing = Working
```

while:

```text
DNS = Possible Failure
```

This significantly narrows the troubleshooting scope.

---

# Example Failure Scenario 3

Suppose the VLAN 20 client can reach its gateway but cannot reach an external IP.

The packet path should be checked:

```text
VLAN 20
   │
   ▼
pfSense
   │
   ▼
Firewall Rule
   │
   ▼
Surfshark Gateway
   │
   ▼
NAT
   │
   ▼
VPN
```

The fact that the local gateway responds suggests switching and basic VLAN connectivity are already functioning.

---

# Security Benefits

The Surfshark project provides several useful security concepts.

### Centralized VPN Enforcement

Individual clients do not need to run their own Surfshark applications.

```text
Clients
   │
   ▼
pfSense
   │
   ▼
VPN
```

### Network-Level Policy

Devices can be routed based on network placement.

### IoT Isolation

Less-trusted devices can use a separate security and egress policy.

### Reduced Client Configuration

The gateway handles VPN connectivity centrally.

---

# Centralized VPN vs Client VPN Software

Without pfSense integration:

```text
Device 1 → Surfshark App
Device 2 → Surfshark App
Device 3 → Surfshark App
```

With pfSense:

```text
Device 1 ┐
Device 2 ├──► pfSense ──► Surfshark
Device 3 ┘
```

This is particularly useful for IoT devices that cannot install VPN software themselves.

---

# Security Limitation

A commercial VPN does not replace firewalling, endpoint security, or segmentation.

Surfshark changes the outbound network path.

It does not automatically:

- Secure a compromised endpoint
- Prevent malware execution
- Replace network segmentation
- Replace Wazuh monitoring
- Replace firewall policy

It is one component of the broader architecture.

---

# Defense-in-Depth

The outbound VPN exists alongside other controls:

```text
VLAN Segmentation
       │
       ▼
pfSense Firewall
       │
       ▼
Policy Routing
       │
       ▼
Surfshark VPN
       │
       ▼
Internet
```

Other security layers remain responsible for endpoint and identity protection.

---

# Monitoring the VPN

The Surfshark connection should be monitored for:

- OpenVPN connection status
- Gateway status
- Packet counters
- Public IP
- Client connectivity
- DNS functionality

A VPN that silently disconnects can change the expected traffic path.

---

# Public IP as a Health Check

One useful operational test is:

```text
Expected VPN Public IP?
        │
        ├── Yes → Routing likely correct
        │
        └── No → Investigate gateway path
```

This does not replace firewall or packet-level inspection, but it is a quick end-to-end check.

---

# Change Management

VPN configuration changes can affect large portions of the network.

The safest workflow is:

```text
Document Current Configuration
          │
          ▼
Back Up pfSense
          │
          ▼
Make One Change
          │
          ▼
Apply
          │
          ▼
Test OpenVPN
          │
          ▼
Test Gateway
          │
          ▼
Test Client Public IP
          │
          ▼
Test DNS
```

Testing each stage makes failures easier to isolate.

---

# Configuration Backup

Before major Surfshark, firewall, NAT, or gateway changes, the pfSense configuration should be backed up.

Important configuration areas include:

- OpenVPN clients
- Gateways
- Firewall rules
- NAT
- Interfaces
- VLANs
- DNS settings

This provides a recovery path if a routing change causes widespread connectivity problems.

---

# What Should Not Be Published

The public GitHub repository should document the architecture without exposing VPN credentials.

Do not publish:

```text
Surfshark username
Surfshark password
Private keys
OpenVPN authentication secrets
Complete private client configuration
pfSense configuration backups containing credentials
```

Screenshots should also be reviewed before publishing to ensure credentials are not visible.

---

# Current Project State

The project has successfully demonstrated:

```text
Surfshark OpenVPN Client
        │
        ▼
Successful Tunnel
        │
        ▼
Alternative VPN Gateway
        │
        ▼
Policy-Routing Testing
        │
        ▼
Public IP Verification
```

Further work is continuing around the final VLAN 20 egress policy and ensuring that IoT traffic reliably uses the intended VPN path.

---

# Future Improvements

Planned improvements include:

- Complete reliable VLAN 20 Surfshark routing
- Implement fail-closed behavior where appropriate
- Prevent unintended ISP fallback
- Refine outbound NAT
- Improve gateway monitoring
- Improve VPN firewall aliases
- Test DNS behavior across the VPN
- Review IPv6 behavior
- Improve logging
- Document final rule sets
- Test recovery when the Surfshark tunnel goes offline

---

# Skills Demonstrated

This project demonstrates practical experience with:

## VPN Technologies

- OpenVPN client configuration
- Commercial VPN integration
- VPN endpoint troubleshooting
- Encryption compatibility
- Tunnel validation

## pfSense

- Gateway configuration
- Firewall rules
- Policy-based routing
- Outbound NAT
- VPN interfaces

## Networking

- Routing
- NAT
- VLANs
- Public IP testing
- IPv4/IPv6 considerations
- DNS troubleshooting
- Packet-path analysis

## Troubleshooting

- Cipher compatibility
- VPN endpoints
- Gateway selection
- Rule matching
- NAT
- DNS
- Packet capture

---

# What I Learned

The Surfshark project demonstrated that there are several separate stages between establishing a VPN and successfully routing a client through it.

```text
OpenVPN Connected
        │
        ▼
Gateway Available
        │
        ▼
Firewall Rule Matches
        │
        ▼
Correct Gateway Selected
        │
        ▼
NAT Works
        │
        ▼
Traffic Enters Tunnel
        │
        ▼
Return Traffic Works
        │
        ▼
DNS Works
```

If any one of these fails, the client may lose connectivity or bypass the intended VPN path.

The biggest lesson was:

> **Always verify the complete packet path instead of assuming that a connected VPN means client traffic is protected by it.**

Public-IP testing, routing checks, firewall inspection, and packet capture provide evidence of what the traffic is actually doing.

---

# Surfshark Architecture Summary

```text
                          Internal Networks
                                 │
                       ┌─────────┴─────────┐
                       │                   │
                       ▼                   ▼
                  VLAN 10               VLAN 20
                  LAB-LAN                  IOT
                       │                   │
                       │                   ▼
                       │             Policy Rule
                       │                   │
                       ▼                   ▼
                     pfSense ───────► Surfshark Gateway
                       │                   │
                       ▼                   ▼
                 Normal ISP           OpenVPN Tunnel
                       │                   │
                       ▼                   ▼
                    Internet         Surfshark Server
                                           │
                                           ▼
                                        Internet
```

The Surfshark integration adds another routing option to the home lab and demonstrates how pfSense can centrally control Internet egress for different network segments.

---

# Related Documentation

[← pfSense OpenVPN Remote Access](remote-access-openvpn.md)

[← pfSense Firewall Rules and VLAN Routing](firewall-and-vlan-routing.md)

[← pfSense Firewall and Routing Architecture](pfsense-architecture.md)

[← VLAN Design and Network Segmentation](../01-network-infrastructure/vlan-design.md)

[← Proxmox Virtual Networking](../02-proxmox/virtual-networking.md)

[← Main Project README](../README.md)
