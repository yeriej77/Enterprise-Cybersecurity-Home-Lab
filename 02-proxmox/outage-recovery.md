# Proxmox Power Outage Recovery

## Overview

One of the most valuable troubleshooting experiences in the home lab occurred after an unexpected power outage.

When power was restored, the environment did not return to its normal state automatically.

Several critical systems were affected:

- The Proxmox web interface was initially unreachable
- DC1 failed to start normally
- pfSense was running
- Network and virtualization services had to be validated individually
- Infrastructure dependencies made troubleshooting more complicated

This incident demonstrated how a physical power failure can affect virtualization, networking, Active Directory, DNS, DHCP, and security services simultaneously.

---

# Environment

The affected infrastructure included:

| System | Role | IP Address |
|---|---|---:|
| Proxmox | Hypervisor | `10.0.0.5` |
| pfSense | Router / Firewall | `10.0.0.1` |
| DC1 | AD / DNS / DHCP | `10.0.0.6` |
| Wazuh | SIEM | `10.0.0.7` |
| Nessus | Vulnerability Scanner | `10.0.0.8` |

All major virtualized services depend on the Dell PowerEdge R710 running Proxmox.

---

# Incident

After an unexpected power outage, the Dell server powered back on, but the environment was not fully operational.

The observed state included:

```text
Physical Server
      │
      ▼
Proxmox Booted
      │
      ├── pfSense → Running
      ├── DC1 → Not Starting Correctly
      └── Proxmox Web GUI → Unreachable
```

This created an unusual troubleshooting situation because some services were available while others were not.

---

# Initial Symptoms

The primary symptoms were:

- pfSense appeared to be running
- Internal network connectivity was inconsistent
- The Proxmox web interface could not be reached normally
- DC1 was not operating correctly
- Services depending on Active Directory and DNS were affected

The environment required troubleshooting from the infrastructure layer upward.

---

# Why the Incident Was Complicated

The home lab contains several service dependencies.

For example:

```text
Physical Server
      │
      ▼
Proxmox
      │
      ├── pfSense
      │      │
      │      └── Gateway / Firewall / Routing
      │
      └── DC1
             │
             ├── DNS
             ├── DHCP
             └── Active Directory
```

If Proxmox networking is unavailable, the management interface may not be reachable.

If pfSense is unavailable, routing may fail.

If DC1 is unavailable, DNS and domain services may fail.

Because these systems depend on one another, a failure can appear to be caused by the wrong component.

---

# Recovery Strategy

The environment was recovered by troubleshooting the infrastructure in layers.

The recovery process followed this general order:

```text
Physical Hardware
      │
      ▼
Proxmox Host
      │
      ▼
Host Networking
      │
      ▼
pfSense
      │
      ▼
DC1
      │
      ▼
Security Services
      │
      ▼
Client Systems
```

This helped isolate problems instead of making multiple unrelated configuration changes.

---

# Step 1 — Verify the Physical Server

The first step was confirming that the Dell PowerEdge R710 had successfully powered on.

Checks included:

- Server power
- Boot completion
- Console availability
- Physical network link

Because remote administration was unavailable, direct access to the server console became important.

---

# Step 2 — Verify Proxmox

The next step was verifying that Proxmox had successfully booted.

Important items to confirm included:

```text
Proxmox OS Running
Network Interfaces Present
Linux Bridge Present
Management IP Correct
VMs Visible
```

The expected Proxmox management IP is:

```text
10.0.0.5
```

---

# Proxmox Network Validation

Useful Linux commands for validating the host include:

```bash
ip addr
```

to confirm interface addressing.

```bash
ip route
```

to verify routing.

```bash
bridge link
```

to inspect the bridge interfaces.

The default gateway should point to:

```text
10.0.0.1
```

which is pfSense.

---

# Step 3 — Verify pfSense

pfSense was one of the first critical virtual machines to verify.

pfSense provides:

- Default gateway services
- Firewalling
- NAT
- VLAN routing
- OpenVPN
- Surfshark VPN routing

The expected address is:

```text
10.0.0.1
```

Because pfSense was running, this confirmed that the virtual machine itself had started.

However, that did not automatically prove that all network paths were functioning correctly.

---

# Step 4 — Verify DC1

DC1 failed to return normally after the outage.

This was significant because DC1 provides:

```text
Active Directory
DNS
DHCP
Group Policy
```

The expected address is:

```text
10.0.0.6
```

Without DC1, several symptoms can appear throughout the network.

These include:

- Failed hostname resolution
- Domain authentication issues
- DHCP problems
- Group Policy failures
- Difficulty locating internal services

---

# Why DNS Can Make an Outage Look Worse

A major lesson from the recovery process was that DNS failures can make a network appear completely broken.

For example:

```text
Ping 10.0.0.1
      │
      └── Works

Ping 1.1.1.1
      │
      └── Works

Resolve website name
      │
      └── Fails
```

This indicates that IP connectivity may still exist even though DNS is unavailable.

Because DC1 is the internal DNS server, its failure can make the environment appear to have a wider network outage.

---

# Service Dependency Chain

The incident reinforced the importance of understanding service dependencies.

```text
Client
  │
  ├── Gateway ─────────► pfSense
  │                     10.0.0.1
  │
  ├── DNS ─────────────► DC1
  │                     10.0.0.6
  │
  ├── Authentication ──► DC1
  │
  └── DHCP ────────────► DC1
```

A failure in DC1 affects multiple services even when pfSense is functioning normally.

---

# Step 5 — Verify Security Services

Once networking and domain services were restored, security services could be checked.

These include:

```text
Wazuh
10.0.0.7
```

and:

```text
Nessus
10.0.0.8
```

These services are important, but they are not required before the network and domain infrastructure are operational.

---

# Correct Recovery Priority

The incident helped establish a logical recovery sequence.

```text
1. Physical Server
      │
      ▼
2. Proxmox
      │
      ▼
3. pfSense
      │
      ▼
4. DC1
      │
      ▼
5. Wazuh / Nessus
      │
      ▼
6. Client Systems
```

This minimizes the number of secondary errors encountered during recovery.

---

# VM Startup Order

The outage demonstrated that VM startup order matters.

A better startup strategy is:

```text
Proxmox
   │
   ▼
pfSense
   │
   ▼
DC1
   │
   ▼
Security Servers
   │
   ▼
Client VMs
```

Starting dependent systems before networking or DNS is available can create unnecessary errors.

---

# Virtualized Firewall Consideration

The incident also highlighted a unique risk in the current design.

pfSense runs as a VM inside Proxmox.

This means:

```text
Proxmox Failure
      │
      ▼
pfSense Failure
      │
      ▼
Network Gateway Failure
```

The hypervisor is therefore part of the network infrastructure, not just the compute infrastructure.

---

# Single Point of Failure

The Dell R710 currently represents a major single point of failure.

```text
Dell R710
   │
   ▼
Proxmox
   │
   ├── pfSense
   ├── DC1
   ├── Wazuh
   ├── Nessus
   └── Client VMs
```

If the physical server becomes unavailable, most of the lab becomes unavailable simultaneously.

---

# Future Second Proxmox Host

One of the planned improvements to the home lab is adding a second physical Proxmox server.

The future design may look like:

```text
                 Lab Infrastructure
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
      Proxmox Node 1        Proxmox Node 2
        Dell R710            Future Server
             │                     │
        Core Services        Security / Test
```

This will help with:

- RAM capacity
- Workload distribution
- Maintenance flexibility
- Reduced infrastructure concentration
- Future clustering experience

---

# Power Protection

The outage also demonstrated the importance of power protection.

A future improvement is to use a UPS capable of supporting the critical infrastructure long enough to:

- Survive short outages
- Perform controlled shutdowns
- Reduce abrupt filesystem interruption
- Protect the virtualization host
- Protect network infrastructure

Conceptually:

```text
Utility Power
     │
     ▼
    UPS
     │
     ├── Dell R710
     ├── Cisco Switch
     └── Network Equipment
```

---

# Why Graceful Shutdown Matters

An abrupt power loss is different from a normal shutdown.

A controlled shutdown allows:

```text
Applications Stop
      │
      ▼
Guest OS Shutdown
      │
      ▼
VM Shutdown
      │
      ▼
Proxmox Shutdown
      │
      ▼
Server Power Off
```

An unexpected outage skips this controlled sequence.

This can increase the risk of:

- VM corruption
- Filesystem problems
- Database issues
- Service startup failures
- Inconsistent application states

---

# Recovery Validation

The environment should not be considered fully recovered simply because Proxmox loads.

Each layer should be validated.

## Proxmox

```text
https://10.0.0.5:8006
```

## pfSense

```powershell
ping 10.0.0.1
```

## DC1

```powershell
ping 10.0.0.6
```

## DNS

```powershell
nslookup dc1.ad.home.com
```

## Wazuh

Confirm that agents return online and events continue arriving.

## Nessus

Confirm the scanning service and web interface are accessible.

---

# Windows Client Validation

Once DC1 is available, a domain workstation can be used to verify the environment.

Useful tests include:

```powershell
ipconfig /all
```

Confirm:

```text
Gateway: 10.0.0.1
DNS:     10.0.0.6
```

Then test:

```powershell
ping 10.0.0.1
```

```powershell
ping 10.0.0.6
```

and:

```powershell
nslookup dc1.ad.home.com
```

This validates both basic network and Active Directory DNS connectivity.

---

# Layered Troubleshooting Model

The outage reinforced a troubleshooting methodology that can be reused for future incidents.

```text
Layer 1
Physical Hardware
      │
      ▼
Layer 2
Switching / VLANs
      │
      ▼
Layer 3
IP / Gateway / Routing
      │
      ▼
Infrastructure
Proxmox / pfSense
      │
      ▼
Core Services
DNS / DHCP / AD
      │
      ▼
Applications
Wazuh / Nessus / Clients
```

Troubleshooting from the bottom upward reduces guesswork.

---

# Avoiding Random Configuration Changes

One important lesson was not to begin changing firewall, VLAN, or DNS settings simply because services were unavailable.

Instead:

```text
Observe
   │
   ▼
Verify
   │
   ▼
Isolate
   │
   ▼
Change
   │
   ▼
Test
```

This prevents introducing additional problems during an already unstable environment.

---

# Incident Response Perspective

Although the root cause was a power outage rather than a cyberattack, the recovery process used many of the same principles found in incident response.

### Identification

Determine what is unavailable.

### Triage

Identify the most critical affected systems.

### Containment

Avoid unnecessary configuration changes.

### Recovery

Restore services in dependency order.

### Validation

Confirm that each service is functioning normally.

### Lessons Learned

Identify changes that can reduce future impact.

---

# Availability and the CIA Triad

This incident primarily affected the:

```text
Availability
```

component of the CIA triad.

The systems were not necessarily compromised, but users and dependent services could not reliably access them.

This demonstrates that cybersecurity includes more than preventing attackers.

Security also requires ensuring that critical services remain available when needed.

---

# Lessons Learned

The power outage provided several important infrastructure lessons.

I learned that:

- A running hypervisor does not guarantee that every VM is healthy
- A running pfSense VM does not guarantee that DNS is working
- DNS outages can look like Internet outages
- VM startup order matters
- Infrastructure dependencies must be documented
- Direct console access is valuable when networking fails
- Virtualizing the firewall increases dependency on the hypervisor
- A single physical host creates a large failure domain
- Power protection is an important availability control
- Recovery should be performed in dependency order

The most important lesson was:

> **Restore the foundation first, then restore the services that depend on it.**

---

# Improvements Identified

The incident resulted in several planned improvements:

- Add a second Proxmox host
- Improve VM startup ordering
- Continue maintaining snapshots and backups
- Improve disaster-recovery documentation
- Add UPS protection
- Document infrastructure dependencies
- Separate management traffic using VLAN 99
- Improve monitoring of critical services
- Develop a repeatable recovery checklist

---

# Future Recovery Checklist

A future power-outage recovery can follow:

```text
[ ] Verify server power
[ ] Verify Cisco switch connectivity
[ ] Verify Proxmox boot
[ ] Verify Proxmox IP 10.0.0.5
[ ] Verify vmbr0
[ ] Verify pfSense VM
[ ] Verify gateway 10.0.0.1
[ ] Verify DC1
[ ] Verify DNS 10.0.0.6
[ ] Verify DHCP
[ ] Verify domain authentication
[ ] Verify Wazuh
[ ] Verify Nessus
[ ] Verify client connectivity
[ ] Confirm logs/events are flowing
```

This creates a more repeatable recovery process.

---

# Skills Demonstrated

This incident demonstrated practical experience with:

### Infrastructure Troubleshooting

- Hypervisor recovery
- VM startup troubleshooting
- Network troubleshooting
- DNS troubleshooting
- Service dependency analysis

### Proxmox Administration

- Console access
- VM status validation
- Virtual networking
- Startup dependencies

### Windows Infrastructure

- Active Directory availability
- DNS validation
- DHCP dependencies
- Domain connectivity testing

### Cybersecurity Operations

- Availability analysis
- Incident triage
- Recovery planning
- Infrastructure resilience
- Lessons-learned analysis

---

# Incident Summary

```text
                  Unexpected Power Outage
                           │
                           ▼
                    Dell R710 Reboots
                           │
                           ▼
                    Proxmox Starts
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
           pfSense Running        DC1 Problem
                │                     │
                ▼                     ▼
        Gateway Available      DNS / AD Impact
                │                     │
                └──────────┬──────────┘
                           ▼
                    Layered Recovery
                           │
                    ┌──────┴──────┐
                    │             │
                    ▼             ▼
               Networking     Core Services
                    │             │
                    └──────┬──────┘
                           ▼
                  Environment Restored
```

The outage demonstrated how tightly connected the virtualization, networking, and identity layers of the home lab have become.

It also provided practical experience recovering a multi-service environment after an unexpected infrastructure failure.

---

# Related Documentation

[← Proxmox VE 9 Upgrade](upgrade-to-proxmox-ve-9.md)

[← Proxmox Resource and Memory Management](resource-management.md)

[← Proxmox Virtual Networking](virtual-networking.md)

[← Proxmox Virtualization Architecture](proxmox-architecture.md)

[← Main Project README](../README.md)
