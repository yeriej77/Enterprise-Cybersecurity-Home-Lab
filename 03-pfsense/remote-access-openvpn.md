# pfSense OpenVPN Remote Access

## Overview

The home lab uses **OpenVPN on pfSense** to provide secure remote access to internal infrastructure.

The objective of the project was to allow an authorized remote computer to connect through the Internet and access resources inside the home lab without exposing individual management services directly to the public Internet.

The remote-access VPN provides connectivity to systems such as:

- pfSense
- Proxmox
- Windows Server 2025
- Active Directory resources
- Wazuh
- Nessus
- Other authorized lab systems

The VPN was successfully configured and tested from a Windows computer.

---

# VPN Architecture

The remote-access VPN follows this general design:

```text
Remote Windows Computer
          │
          │ Internet
          ▼
      Public Network
          │
          ▼
       pfSense
   OpenVPN Server
      UDP 1194
          │
          ▼
   VPN Client Network
     10.8.0.0/24
          │
          ▼
    pfSense Routing
          │
          ▼
      LAB-LAN
     10.0.0.0/24
```

Remote clients are placed on a separate VPN network rather than directly joining the LAB-LAN subnet.

---

# OpenVPN Server

The OpenVPN server is hosted directly on pfSense.

The configured server is:

```text
HomeLab Remote Access
```

and uses:

```text
Protocol: OpenVPN UDP
Port:     1194
IP Mode:  IPv4
```

The remote VPN client network is:

```text
10.8.0.0/24
```

The primary internal LAB-LAN is:

```text
10.0.0.0/24
```

---

# Why OpenVPN Was Used

OpenVPN provides an encrypted tunnel between a remote endpoint and the pfSense firewall.

Instead of exposing services such as:

```text
Proxmox :8006
Nessus :8834
Windows RDP
Wazuh Dashboard
pfSense Management
```

directly to the Internet, a remote administrator first establishes the VPN tunnel.

```text
Internet
    │
    ▼
OpenVPN
    │
    ▼
Internal Access
```

This creates a much safer administrative access model.

---

# Remote Access Security Model

Without a VPN, remote access could require opening multiple management ports to the Internet.

Conceptually:

```text
Internet
   │
   ├── RDP
   ├── Proxmox
   ├── Nessus
   └── Other Management Services
```

The preferred design is:

```text
Internet
   │
   ▼
OpenVPN
   │
   ▼
pfSense Firewall
   │
   ▼
Authorized Internal Resources
```

Only the VPN service needs to be externally reachable for this remote-access path.

---

# Separate VPN Address Space

VPN clients receive addresses from:

```text
10.8.0.0/24
```

instead of receiving addresses from:

```text
10.0.0.0/24
```

This separation provides several benefits.

It allows pfSense to distinguish:

```text
Local LAB-LAN traffic
```

from:

```text
Remote VPN traffic
```

and apply different firewall policies to each.

---

# Why the VPN Uses a Separate Subnet

A separate VPN subnet improves:

- Traffic identification
- Firewall policy control
- Troubleshooting
- Network segmentation
- Remote-access auditing

Conceptually:

```text
VPN Clients
10.8.0.0/24
      │
      ▼
   pfSense
      │
 Firewall Rules
      │
      ▼
LAB-LAN
10.0.0.0/24
```

Remote clients must pass through pfSense before reaching internal resources.

---

# VPN Client Connection Process

The connection process follows this general sequence:

```text
Remote Computer
      │
      ▼
OpenVPN Client
      │
      ▼
Internet
      │
      ▼
pfSense UDP 1194
      │
      ▼
Authentication / TLS
      │
      ▼
VPN Tunnel Established
      │
      ▼
Client Receives VPN Address
      │
      ▼
Internal Routes Available
```

Once the tunnel is established, traffic destined for approved internal networks can traverse the VPN.

---

# OpenVPN Client Export Utility

pfSense's **OpenVPN Client Export Utility** was used to generate the client configuration.

The configured server appeared as:

```text
HomeLab Remote Access UDP4:1194
```

The export utility provides client packages or configuration files that contain the settings required to connect to the pfSense OpenVPN server.

---

# Client Configuration Deployment

The exported OpenVPN configuration was transferred to the Windows computer that would be used for remote administration.

The configuration contains the information required for the OpenVPN client to locate and authenticate to the VPN server.

The basic process was:

```text
pfSense
   │
   ▼
Client Export Utility
   │
   ▼
OpenVPN Configuration
   │
   ▼
Windows Computer
   │
   ▼
OpenVPN Client
```

---

# Initial Connection Testing

After importing the configuration on the Windows computer, the OpenVPN client attempted to establish the connection.

During initial testing, the client became stuck while attempting to connect.

This required troubleshooting the VPN path rather than assuming that exporting the configuration automatically guaranteed a working connection.

---

# VPN Troubleshooting Model

The connection was approached layer by layer.

```text
Client Configuration
        │
        ▼
Internet Connectivity
        │
        ▼
Public VPN Endpoint
        │
        ▼
UDP 1194
        │
        ▼
pfSense OpenVPN Server
        │
        ▼
Authentication
        │
        ▼
Tunnel Establishment
        │
        ▼
VPN Routing
        │
        ▼
Firewall Policy
        │
        ▼
Internal Resources
```

A successful VPN deployment requires all of these layers to function.

---

# Client-Side Checks

When the client could not connect correctly, the remote endpoint had to be checked first.

Important areas included:

- OpenVPN client installed
- Correct configuration imported
- Internet connectivity available
- VPN configuration associated with the intended server
- Client logs reviewed during connection attempts

A client-side failure can occur before traffic ever reaches pfSense.

---

# Server-Side Checks

On pfSense, the corresponding OpenVPN server configuration must also be validated.

Important items include:

```text
Server enabled
Correct protocol
Correct port
Correct tunnel network
Certificates available
Firewall path available
```

The configured service uses:

```text
UDP 1194
```

---

# Firewall Requirement

The OpenVPN connection must be allowed to reach pfSense from the Internet-facing side.

Conceptually:

```text
Internet
    │
    │ UDP 1194
    ▼
pfSense WAN
    │
    ▼
OpenVPN Server
```

If the traffic cannot reach the OpenVPN service, the client will not be able to establish a tunnel.

---

# Successful VPN Connection

After troubleshooting, the OpenVPN connection was successfully established.

The remote Windows computer could then access resources inside the home lab.

The successful path was:

```text
Remote Windows PC
       │
       ▼
Encrypted OpenVPN Tunnel
       │
       ▼
     pfSense
       │
       ▼
10.8.0.0/24 VPN Network
       │
       ▼
LAB-LAN Resources
10.0.0.0/24
```

This confirmed that both tunnel establishment and internal routing were working.

---

# Internal Resource Access

Once connected, the VPN client was able to reach internal resources.

Examples include infrastructure located on:

```text
10.0.0.0/24
```

such as:

```text
pfSense     10.0.0.1
Cisco SW1   10.0.0.2
Proxmox     10.0.0.5
DC1         10.0.0.6
Wazuh       10.0.0.7
Nessus      10.0.0.8
```

Access to internal systems depends on the firewall rules applied to OpenVPN traffic.

---

# OpenVPN Firewall Rules

Establishing the VPN tunnel does not automatically mean a remote client should have unlimited access.

pfSense can apply firewall rules to traffic originating from the OpenVPN network.

Conceptually:

```text
VPN Client
10.8.0.x
    │
    ▼
OpenVPN Interface
    │
    ▼
pfSense Firewall Rules
    │
 ┌──┴──┐
 │     │
Allow Block
 │
 ▼
Internal Resource
```

This makes it possible to control exactly what remote users may access.

---

# Least-Privilege Remote Access

A basic lab configuration may begin with broader access for testing.

A more mature configuration can restrict VPN clients to only required services.

For example:

```text
VPN Clients
     │
     ├── Proxmox Management → Allow
     ├── Wazuh Dashboard → Allow
     ├── Nessus → Allow
     ├── RDP to Authorized Systems → Allow
     └── Other Networks → Restrict
```

The long-term objective is to provide only the access required for administration.

---

# Remote Access vs Direct Exposure

The VPN dramatically reduces the need to expose management services publicly.

For example, instead of:

```text
Internet
   │
   ▼
Proxmox :8006
```

the design becomes:

```text
Internet
   │
   ▼
OpenVPN
   │
   ▼
Private Network
   │
   ▼
Proxmox :8006
```

The Proxmox management service remains on the internal network.

---

# Remote Administration Workflow

A typical remote administration session follows:

```text
1. Connect to the Internet
        │
        ▼
2. Launch OpenVPN Client
        │
        ▼
3. Connect to HomeLab VPN
        │
        ▼
4. Receive 10.8.0.x Address
        │
        ▼
5. Access Authorized Internal Resources
```

This provides one secure entry point into the lab.

---

# Testing the VPN Address

After connection, the client should receive an address belonging to:

```text
10.8.0.0/24
```

The exact assigned host address can vary.

On Windows, it can be reviewed using:

```powershell
ipconfig
```

or:

```powershell
ipconfig /all
```

The OpenVPN adapter should show the tunnel-related network configuration.

---

# Testing Internal Connectivity

After the tunnel is established, internal connectivity can be tested incrementally.

First test pfSense:

```powershell
ping 10.0.0.1
```

Then another infrastructure system:

```powershell
ping 10.0.0.5
```

or:

```powershell
ping 10.0.0.6
```

These tests help determine whether routing between:

```text
10.8.0.0/24
```

and:

```text
10.0.0.0/24
```

is functioning.

---

# Application-Level Testing

ICMP connectivity is only one test.

The next step is verifying the actual service required.

For example:

```text
Proxmox Web Interface
https://10.0.0.5:8006
```

or:

```text
Nessus
https://10.0.0.8:8834
```

This distinguishes basic network connectivity from application-level access.

---

# VPN Route Distribution

Remote clients need to know that the internal network is reachable through the VPN tunnel.

Conceptually:

```text
Destination:
10.0.0.0/24
     │
     ▼
OpenVPN Tunnel
```

Without the appropriate route, the client may establish the VPN successfully but continue trying to reach internal systems through its normal local gateway.

---

# Tunnel Up Does Not Mean Routing Works

This is an important troubleshooting lesson.

A VPN can show:

```text
CONNECTED
```

while:

```text
10.0.0.6
```

is still unreachable.

That means:

```text
Tunnel establishment
```

and:

```text
Internal routing
```

must be tested separately.

---

# Routing and Firewall Dependencies

Successful internal access requires:

```text
VPN Tunnel
    +
Route to LAB-LAN
    +
pfSense Firewall Rule
    +
Return Path
```

Conceptually:

```text
VPN Client
10.8.0.x
    │
    ▼
pfSense
    │
    ▼
10.0.0.6
    │
    ▼
Response
    │
    ▼
pfSense
    │
    ▼
VPN Client
```

Both directions must work.

---

# DNS Considerations

A remote VPN client may be able to reach:

```text
10.0.0.6
```

while still failing to resolve:

```text
dc1.ad.home.com
```

This indicates a DNS issue rather than a tunnel problem.

Internal Active Directory DNS is provided by:

```text
DC1
10.0.0.6
```

If remote users need internal hostname resolution, the VPN DNS configuration must support that requirement.

---

# Testing Internal DNS

A remote Windows client can test the internal DNS server directly:

```powershell
nslookup dc1.ad.home.com 10.0.0.6
```

A successful lookup demonstrates that:

- DC1 is reachable
- DNS is responding
- The internal record exists

---

# Remote Access and Active Directory

The VPN can also support administration of domain resources remotely.

Conceptually:

```text
Remote Administrator
       │
       ▼
OpenVPN
       │
       ▼
LAB-LAN
       │
       ▼
DC1
ad.home.com
```

This makes remote management possible without exposing Active Directory services directly to the Internet.

---

# Internet Traffic While Connected

Remote-access VPN traffic and Internet traffic are separate design decisions.

A VPN client may use:

```text
VPN Tunnel → Internal Networks
```

while normal Internet traffic continues using:

```text
Remote Client's Local Internet Connection
```

Alternatively, pfSense can be configured to route additional client traffic through the VPN.

The intended behavior should be explicitly designed rather than assumed.

---

# Split Tunnel Concept

A split-tunnel design sends only home-lab traffic through OpenVPN.

```text
Remote PC
   │
   ├── 10.0.0.0/24 → VPN
   │
   └── Internet → Local ISP
```

This reduces unnecessary VPN traffic while still providing access to internal systems.

---

# Full-Tunnel Concept

A full-tunnel configuration sends Internet traffic through the VPN as well.

```text
Remote PC
   │
   ▼
OpenVPN
   │
   ▼
pfSense
   │
   ▼
Internet
```

This requires additional routing, firewall, NAT, and possibly gateway-policy considerations.

---

# OpenVPN vs Surfshark

The home lab uses OpenVPN in two very different ways.

## HomeLab Remote Access

```text
Remote User
     │
     ▼
pfSense
     │
     ▼
Home Lab
```

Purpose:

```text
Enter the lab remotely
```

---

## Surfshark OpenVPN Client

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

Purpose:

```text
Send selected lab traffic outward through a VPN
```

These configurations should not be confused simply because both use OpenVPN.

---

# Security Advantages

The remote-access VPN provides several security benefits.

### Encryption

Traffic between the remote computer and pfSense is protected by the VPN tunnel.

### Reduced Public Exposure

Management services can remain internal.

### Centralized Access Control

pfSense controls which VPN clients can reach internal resources.

### Separate Address Space

Remote users can be identified by their `10.8.0.0/24` source addresses.

### Logging

VPN connection activity can be reviewed centrally on pfSense.

---

# Security Considerations

The VPN itself becomes an important security boundary.

Security depends on:

- Protecting VPN credentials
- Protecting private keys
- Maintaining certificates
- Keeping pfSense updated
- Limiting firewall permissions
- Removing unnecessary VPN access
- Reviewing logs
- Protecting exported client configuration files

A VPN should not be treated as equivalent to unlimited trusted LAN access.

---

# Client Configuration Protection

Exported VPN configurations should be treated as sensitive.

Depending on the configuration, the client package may contain information associated with the user's VPN identity and authentication.

Client files should therefore not be:

- Published in GitHub
- Shared publicly
- Stored in unsecured locations
- Reused by unauthorized users

The portfolio documents the architecture without publishing secrets or production VPN configuration files.

---

# Important GitHub Security Practice

The following should **never** be committed to the public repository:

```text
Private keys
VPN passwords
Certificate private keys
OpenVPN client secrets
Authentication credentials
pfSense configuration backups containing secrets
```

Documentation should demonstrate the design without exposing the credentials that protect it.

---

# Troubleshooting a Stuck VPN Connection

When a VPN client becomes stuck while connecting, troubleshooting should follow the connection path.

```text
Client
  │
  ▼
Does Internet work?
  │
  ▼
Does traffic reach the public VPN endpoint?
  │
  ▼
Is UDP 1194 reachable?
  │
  ▼
Does pfSense receive the connection?
  │
  ▼
Does authentication succeed?
  │
  ▼
Does TLS negotiation succeed?
  │
  ▼
Is a tunnel address assigned?
```

This is more effective than repeatedly reinstalling the client without identifying where the negotiation fails.

---

# Tunnel Troubleshooting

After the client reports a successful connection:

```text
Connected
```

the next troubleshooting path is:

```text
VPN Address Assigned?
        │
        ▼
Internal Route Installed?
        │
        ▼
pfSense Reachable?
        │
        ▼
LAB-LAN Reachable?
        │
        ▼
DNS Working?
        │
        ▼
Applications Reachable?
```

Connection and usability are two separate validation phases.

---

# pfSense OpenVPN Status

pfSense provides OpenVPN status information that can help confirm:

- Server status
- Connected clients
- Tunnel addresses
- Session information
- Traffic activity

This provides server-side evidence that the remote client successfully established the tunnel.

---

# Logs

OpenVPN logs are important during troubleshooting.

Logs may help identify issues related to:

- Authentication
- Certificates
- TLS negotiation
- Routing
- Client configuration
- Connection establishment

The correct approach is to use the logs to determine where the connection fails rather than guessing.

---

# Packet Capture

If necessary, pfSense packet capture can help confirm whether VPN traffic reaches the firewall.

Conceptually:

```text
Remote Client
      │
      │ UDP 1194
      ▼
pfSense WAN
```

If packets arrive but OpenVPN does not establish the connection, investigation can move toward the application or authentication layer.

If packets never arrive, investigation should remain focused on the network path.

---

# Remote-Access Troubleshooting Flow

```text
OpenVPN Client Will Not Connect
           │
           ▼
Check Client Internet
           │
           ▼
Check Client Configuration
           │
           ▼
Check VPN Endpoint
           │
           ▼
Check UDP 1194
           │
           ▼
Check pfSense OpenVPN Service
           │
           ▼
Check Logs
           │
           ▼
Check Authentication / TLS
           │
           ▼
Tunnel Established
           │
           ▼
Check 10.8.0.x Address
           │
           ▼
Check Internal Route
           │
           ▼
Check Firewall Rule
           │
           ▼
Test 10.0.0.0/24
```

---

# Current Result

The remote-access VPN is operational.

The final working architecture is:

```text
Remote Windows PC
       │
       ▼
    Internet
       │
       ▼
pfSense OpenVPN
    UDP 1194
       │
       ▼
VPN Address
10.8.0.0/24
       │
       ▼
pfSense Firewall
       │
       ▼
LAB-LAN
10.0.0.0/24
       │
       ├── Proxmox
       ├── DC1
       ├── Wazuh
       ├── Nessus
       └── Other Authorized Resources
```

This allows remote administration of the home lab through an encrypted VPN tunnel.

---

# Future Improvements

Possible improvements to the remote-access VPN include:

- Further restrict VPN firewall rules
- Limit administrative access by service
- Improve VPN logging and monitoring
- Review certificate lifecycle procedures
- Improve client revocation procedures
- Document remote-access recovery procedures
- Integrate management VLAN access after VLAN 99 is deployed
- Continue reviewing whether split-tunnel or full-tunnel behavior is appropriate

---

# Skills Demonstrated

This project demonstrates practical experience with:

## VPN Administration

- OpenVPN
- pfSense OpenVPN server
- Client configuration export
- Windows VPN client deployment
- Tunnel networking
- Remote-access VPNs

## Networking

- VPN subnets
- Routing
- Firewall rules
- Return paths
- Internal network access
- DNS troubleshooting

## Security

- Encrypted remote administration
- Reduced public service exposure
- Certificate-based VPN concepts
- Access control
- Least privilege
- Credential protection

## Troubleshooting

- VPN connection failures
- Tunnel validation
- Route testing
- Firewall testing
- DNS testing
- Application-level testing
- Log analysis

---

# What I Learned

Building the OpenVPN remote-access environment reinforced that a VPN involves much more than establishing an encrypted tunnel.

Several components must work together:

```text
Network Reachability
       +
OpenVPN
       +
Authentication
       +
Tunnel Addressing
       +
Routing
       +
Firewall Rules
       +
DNS
       =
Working Remote Access
```

I also learned to separate two questions:

> **Can the VPN client connect?**

from:

> **Can the connected VPN client reach the resources it needs?**

A successful tunnel only solves the first problem.

Routing, firewall policies, DNS, and application access determine whether the VPN is actually useful.

---

# Remote-Access VPN Summary

```text
                    Remote Administrator
                            │
                            ▼
                         Internet
                            │
                            ▼
                         pfSense
                       OpenVPN Server
                        UDP 1194
                            │
                            ▼
                     10.8.0.0/24
                       VPN Clients
                            │
                            ▼
                     Firewall Policy
                            │
                            ▼
                      10.0.0.0/24
                         LAB-LAN
                            │
        ┌───────────┬───────┼───────┬───────────┐
        │           │       │       │           │
        ▼           ▼       ▼       ▼           ▼
     Proxmox       DC1    Wazuh   Nessus     pfSense
     10.0.0.5   10.0.0.6  .7      .8          .1
```

The completed VPN provides a secure remote entry point into the home lab while keeping internal management services behind the pfSense firewall.

---

# Related Documentation

[← pfSense Firewall Rules and VLAN Routing](firewall-and-vlan-routing.md)

[← pfSense Firewall and Routing Architecture](pfsense-architecture.md)

[← Proxmox Virtual Networking](../02-proxmox/virtual-networking.md)

[← IP Addressing and Network Services](../01-network-infrastructure/ip-addressing.md)

[← Main Project README](../README.md)
