# Windows Server 2025 and Active Directory

## Overview

The home lab uses **Windows Server 2025** as the primary Active Directory Domain Controller.

The server is named:

```text
DC1
```

and uses the static IP address:

```text
10.0.0.6
```

DC1 provides several of the most important identity and network services in the environment:

- Active Directory Domain Services
- DNS
- DHCP
- Group Policy
- Domain authentication
- Computer account management
- User and administrative account management
- Centralized Windows security policy

The Active Directory domain is:

```text
ad.home.com
```

with the NetBIOS domain name:

```text
HOMELAB
```

DC1 acts as the identity foundation for the Windows portion of the home lab.

---

# Environment

| Setting | Value |
|---|---|
| Server | Windows Server 2025 |
| Hostname | `DC1` |
| IP Address | `10.0.0.6` |
| Active Directory Domain | `ad.home.com` |
| NetBIOS Domain | `HOMELAB` |
| Default Gateway | `10.0.0.1` |
| DNS Server | `10.0.0.6` |
| Primary Role | Domain Controller |
| Virtualization | Proxmox VE |

DC1 runs as a virtual machine on the Proxmox hypervisor.

---

# Active Directory Architecture

The Windows identity architecture is:

```text
                   ad.home.com
                        │
                        ▼
                       DC1
                    10.0.0.6
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
         AD DS          DNS          DHCP
          │
          ▼
     Group Policy
          │
          ▼
 Domain-Joined Clients
```

DC1 centralizes authentication and configuration for domain-connected Windows systems.

---

# Why Active Directory?

Without Active Directory, every Windows computer would need to be managed independently.

For example:

```text
Computer 1
   ├── Local Users
   ├── Local Password Policy
   └── Local Security Settings

Computer 2
   ├── Local Users
   ├── Local Password Policy
   └── Local Security Settings
```

With Active Directory:

```text
                  DC1
                   │
                   ▼
            Active Directory
                   │
        ┌──────────┼──────────┐
        │          │          │
        ▼          ▼          ▼
     Client01   Workstation   Server
```

Users, computers, security policies, and administrative settings can be managed centrally.

---

# Active Directory Domain

The domain created for the home lab is:

```text
ad.home.com
```

This domain provides the logical identity namespace for Windows systems.

Examples of domain identities include:

```text
HOMELAB\username
```

or:

```text
username@ad.home.com
```

Domain-joined devices can authenticate users against the centralized Active Directory database hosted by DC1.

---

# NetBIOS Domain Name

The NetBIOS domain name is:

```text
HOMELAB
```

This allows identities to be referenced using the traditional Windows format:

```text
HOMELAB\username
```

while the DNS-based Active Directory domain remains:

```text
ad.home.com
```

---

# Domain Controller

DC1 was promoted to an Active Directory Domain Controller.

The server provides the directory service used to store objects such as:

```text
Users
Computers
Groups
Organizational Units
Policies
```

The domain controller also participates in authentication and authorization decisions throughout the Windows environment.

---

# Authentication Flow

A simplified domain logon works like this:

```text
User
  │
  ▼
Client01
  │
  ▼
Locate Domain Controller
  │
  ▼
DC1
ad.home.com
  │
  ▼
Authenticate User
  │
  ▼
Apply Domain Policies
  │
  ▼
User Session
```

This provides centralized identity management instead of relying only on local Windows accounts.

---

# DNS Dependency

Active Directory depends heavily on DNS.

DC1 also hosts the internal DNS service at:

```text
10.0.0.6
```

Domain clients use DC1 for DNS so they can locate Active Directory resources.

For example:

```text
Client01
    │
    │ DNS Query
    ▼
DC1
10.0.0.6
    │
    ▼
ad.home.com
```

---

# Active Directory DNS Records

Active Directory automatically uses DNS service records to help clients locate domain services.

One important example is:

```text
_ldap._tcp.dc._msdcs.ad.home.com
```

This can be tested using:

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.ad.home.com
```

Successful results confirm that clients can locate domain-controller LDAP services through DNS.

---

# Why Clients Use DC1 for DNS

A domain-joined workstation should use:

```text
10.0.0.6
```

as its DNS server.

Using a public DNS resolver directly as the primary DNS server can prevent the computer from locating internal Active Directory records.

The normal client configuration therefore follows:

```text
Client
  │
  ▼
DC1 DNS
10.0.0.6
  │
  ├── Internal Name → Resolve Locally
  │
  └── External Name → Forward
```

---

# Domain-Joined Workstation

Client01 was successfully joined to:

```text
ad.home.com
```

This changed the workstation from a standalone Windows computer into a centrally managed domain endpoint.

Conceptually:

```text
Before

Client01
   │
   └── Local Management


After

Client01
   │
   ▼
ad.home.com
   │
   ▼
DC1
```

The workstation can now receive:

- Domain authentication
- Group Policy
- Central security configuration
- Domain DNS
- Administrative policies

---

# Domain Join Process

The general domain-join process involves:

```text
Client receives network configuration
        │
        ▼
Client uses DC1 for DNS
        │
        ▼
Client resolves ad.home.com
        │
        ▼
Client contacts DC1
        │
        ▼
Domain credentials verified
        │
        ▼
Computer account created
        │
        ▼
Restart
        │
        ▼
Domain logon available
```

DNS must work before the domain-join process can succeed reliably.

---

# Verifying Domain Membership

On a Windows workstation, domain information can be checked using:

```powershell
systeminfo
```

or:

```powershell
Get-ComputerInfo
```

Additional domain information can be checked with:

```powershell
whoami
```

and:

```powershell
whoami /fqdn
```

The workstation should identify itself as part of:

```text
ad.home.com
```

---

# Testing Domain Controller Connectivity

Basic connectivity can be verified using:

```powershell
ping 10.0.0.6
```

DNS can then be tested using:

```powershell
nslookup dc1.ad.home.com
```

Domain-controller discovery can also be tested using:

```powershell
nltest /dsgetdc:ad.home.com
```

This provides more useful evidence than simply confirming that the server responds to ICMP.

---

# Organizational Unit Structure

The domain uses Organizational Units to organize users, administrators, and computers.

The current structure includes:

```text
ad.home.com
│
└── HomeLab
    │
    ├── Users
    │
    ├── Admins
    │
    └── Workstations
```

This provides a cleaner structure than leaving all objects in the default Active Directory containers.

---

# Why Organizational Units Matter

Organizational Units allow administrators to organize objects based on their purpose.

For example:

```text
HomeLab
│
├── Users
│   └── Standard user accounts
│
├── Admins
│   └── Administrative accounts
│
└── Workstations
    └── Domain-connected endpoints
```

This becomes especially important when applying Group Policy.

---

# Separating Administrative Accounts

Administrative identities are placed separately from normal user accounts.

Conceptually:

```text
Normal User
     │
     ▼
HomeLab/Users


Administrative User
     │
     ▼
HomeLab/Admins
```

This provides a foundation for applying stronger controls to privileged accounts.

---

# Workstation Organization

Domain-connected endpoints can be organized inside:

```text
HomeLab/Workstations
```

This allows workstation-specific Group Policies to be applied without automatically applying those same policies to servers or administrative accounts.

---

# Group Policy

Group Policy is used to centrally configure Windows security and operating-system settings.

Instead of changing the same setting manually on every workstation:

```text
DC1
  │
  ▼
Group Policy
  │
  ├── Client01
  ├── Future Client
  └── Future Workstations
```

Policies can be applied automatically when computers authenticate to the domain.

---

# Security Policies Implemented

The lab has used Group Policy to configure security controls including:

- Password policy
- Account lockout
- Microsoft Defender
- Windows Update
- Security auditing
- Process creation auditing
- Remote Desktop-related configuration

These settings are documented in greater detail in the dedicated Group Policy page.

---

# Account Lockout Policy

An account lockout policy was configured and tested.

The environment was configured so repeated failed authentication attempts trigger an account lockout.

The policy was validated by intentionally entering an incorrect password multiple times.

After reaching the configured threshold, the account was locked as expected.

This provided practical experience implementing and testing identity-security controls rather than only configuring them.

---

# Why Account Lockout Matters

Without an account lockout policy:

```text
Attacker
   │
   ▼
Unlimited Password Attempts
```

With a lockout threshold:

```text
Repeated Failed Attempts
        │
        ▼
Configured Threshold
        │
        ▼
Account Locked
```

This helps reduce the effectiveness of repeated password-guessing attempts.

---

# Windows Security Auditing

Windows auditing was enabled to provide visibility into endpoint activity.

One important audit category enabled in the environment is:

```text
Process Creation
```

Windows Security Event ID:

```text
4688
```

records newly created processes.

This allows endpoint actions to be forwarded to the Wazuh SIEM for detection and analysis.

---

# Process Command-Line Auditing

Process creation auditing becomes more valuable when command-line arguments are included.

For example:

```text
powershell.exe
```

alone provides limited context.

With command-line logging:

```text
powershell.exe <command>
```

security monitoring can provide significantly more information about what actually executed.

This becomes important for detecting suspicious administrative or attacker behavior.

---

# Security Group Management Auditing

Additional auditing was later enabled for Windows security group management.

Important events include:

```text
4728
4729
4732
4733
4756
4757
```

These events can help identify users being added to or removed from security groups.

Changes to privileged groups are especially important from a security-monitoring perspective.

---

# Sysmon Integration

Sysmon was also installed in the Windows environment to improve endpoint telemetry.

Sysmon provides more detailed process and system activity than standard Windows logs alone.

For example:

```text
Sysmon Event ID 1
```

records process creation.

This telemetry can be forwarded to Wazuh for additional detection engineering.

---

# Active Directory and Wazuh

The identity environment is integrated with the SIEM architecture.

```text
Windows Endpoint
      │
      ▼
Windows Event Logs / Sysmon
      │
      ▼
Wazuh Agent
      │
      ▼
Wazuh Server
10.0.0.7
      │
      ▼
Detection Rules
```

This creates visibility into both user activity and administrative changes.

---

# Example Security Monitoring Scenario

Suppose an account is added to a privileged group.

The flow can become:

```text
Administrative Action
        │
        ▼
Windows Security Event
        │
        ▼
Wazuh Agent
        │
        ▼
Wazuh Manager
        │
        ▼
Detection Rule
        │
        ▼
Security Alert
```

This connects Active Directory administration with SIEM monitoring.

---

# Active Directory and Nessus

The Windows domain is also part of the vulnerability-management project.

The environment includes a dedicated scanning account:

```text
svc_nessus
```

and a domain group:

```text
Nessus Scan Admins
```

This group is used to provide the permissions necessary for authenticated vulnerability scanning of Windows endpoints.

---

# Authenticated Vulnerability Scanning

Authenticated scans provide much more visibility than unauthenticated network scans.

Instead of only seeing:

```text
Open Ports
Network Services
Externally Visible Software
```

authenticated scanning can evaluate deeper operating-system information and configuration.

Active Directory and Group Policy were used to help prepare Windows systems for authenticated Nessus scanning.

---

# Centralized Administration

DC1 makes it possible to manage several security controls centrally.

For example:

```text
DC1
 │
 ├── Users
 ├── Groups
 ├── Computer Accounts
 ├── DNS
 ├── DHCP
 └── Group Policy
      │
      ▼
 Domain Clients
```

This mirrors the centralized administration model found in enterprise Windows environments.

---

# Security Benefits

Active Directory provides several security advantages when configured correctly.

These include:

- Centralized identities
- Consistent password policies
- Account lockout
- Group-based authorization
- Centralized workstation policies
- Security auditing
- Administrative separation
- Centralized endpoint configuration

However, Active Directory is also a high-value target.

Compromise of the domain controller could have a significant impact on the rest of the environment.

---

# DC1 as a Critical Asset

DC1 provides:

```text
Identity
DNS
DHCP
Group Policy
Authentication
```

This makes:

```text
10.0.0.6
```

one of the most critical systems in the home lab.

If DC1 is unavailable:

```text
DC1 Down
   │
   ├── DNS Impact
   ├── DHCP Impact
   ├── Authentication Impact
   └── Group Policy Impact
```

The power-outage recovery experience demonstrated how important this dependency is.

---

# Administrative Security

Administrative accounts should be treated differently from normal user accounts.

The OU structure provides a foundation for separating:

```text
Standard Users
```

from:

```text
Administrative Users
```

Future improvements can apply stronger controls to administrative identities and restrict where privileged credentials can be used.

---

# Principle of Least Privilege

The long-term identity-management objective is:

> Users and systems should receive only the permissions required to perform their roles.

Instead of making every user a local or domain administrator:

```text
User
  │
  ▼
Required Access Only
```

permissions should be assigned based on operational need.

This reduces the impact of compromised credentials.

---

# Active Directory Troubleshooting

Several different services can cause domain issues.

A useful troubleshooting process is:

```text
Client Network Connectivity
        │
        ▼
Client IP Configuration
        │
        ▼
DNS Server = 10.0.0.6?
        │
        ▼
Can DC1 Be Reached?
        │
        ▼
Can ad.home.com Be Resolved?
        │
        ▼
Can Domain Controller Be Located?
        │
        ▼
Authentication
        │
        ▼
Group Policy
```

This prevents authentication problems from immediately being blamed on Active Directory when the actual cause may be networking or DNS.

---

# Common Domain Troubleshooting Commands

## Display Network Configuration

```powershell
ipconfig /all
```

---

## Test DC1

```powershell
ping 10.0.0.6
```

---

## Test DNS

```powershell
nslookup dc1.ad.home.com
```

---

## Test Active Directory SRV Records

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.ad.home.com
```

---

## Locate a Domain Controller

```powershell
nltest /dsgetdc:ad.home.com
```

---

## Review Group Policy

```powershell
gpresult /r
```

---

## Refresh Group Policy

```powershell
gpupdate /force
```

These commands help separate networking, DNS, domain, and policy problems.

---

# Group Policy Verification

A Group Policy should not be considered successfully deployed simply because it exists in Group Policy Management.

The endpoint should also be checked.

For example:

```powershell
gpresult /r
```

can be used to verify which policies actually applied.

This was used in the lab to confirm that workstation policies were reaching Client01.

---

# Configuration Testing

Security configurations were tested rather than simply assumed to work.

Examples include:

```text
Account Lockout
     │
     ▼
Intentional Failed Logons
     │
     ▼
Account Locked
```

and:

```text
Group Policy
     │
     ▼
gpupdate
     │
     ▼
gpresult
     │
     ▼
Policy Confirmed
```

This validation approach is used throughout the home lab.

---

# Snapshots and Change Management

Because DC1 provides critical services, snapshots are taken around important configuration milestones.

Examples include:

- Initial Windows Server deployment
- Active Directory configuration
- Group Policy changes
- Security auditing changes
- Wazuh agent deployment
- Sysmon deployment

A typical process is:

```text
Known Good State
      │
      ▼
Snapshot
      │
      ▼
Configuration Change
      │
      ▼
Validation
```

This provides a rollback point when testing new configurations.

---

# Power Outage Recovery

DC1 has also been involved in infrastructure recovery after an unexpected power outage.

After one outage:

```text
Proxmox
   │
   ├── pfSense → Running
   └── DC1 → Startup Problem
```

This affected services that depended on the domain controller.

The incident reinforced the importance of restoring infrastructure in dependency order:

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
Security Services
   │
   ▼
Clients
```

---

# Future Active Directory Improvements

As the lab continues to grow, additional improvements may include:

- Additional Organizational Units
- More granular Group Policies
- Administrative workstation restrictions
- More detailed privileged-account controls
- Expanded Windows auditing
- Additional security groups
- Service-account hardening
- Improved log forwarding
- More Active Directory attack simulations
- Additional detection rules
- Dedicated management network integration
- Backup and recovery testing

---

# Security Architecture

Active Directory sits at the center of several security layers.

```text
                    pfSense
                       │
                       ▼
                  Network Access
                       │
                       ▼
                      DC1
                Active Directory
                       │
             ┌─────────┼─────────┐
             │         │         │
             ▼         ▼         ▼
           Users     Groups     GPO
             │
             ▼
        Windows Clients
             │
             ▼
        Windows Auditing
             │
             ▼
           Wazuh
             │
             ▼
       Security Alerts
```

This connects identity, endpoint configuration, logging, and detection into one architecture.

---

# What I Learned

Building the Active Directory environment provided practical experience with:

- Windows Server 2025
- Active Directory Domain Services
- Domain controllers
- Domain joining
- DNS
- DHCP
- Organizational Units
- Users and groups
- Group Policy
- Account lockout
- Windows security auditing
- Process creation logging
- Domain troubleshooting
- SIEM integration

One of the most important lessons was:

> **Active Directory depends heavily on DNS, and many apparent domain problems are actually DNS or network problems.**

The environment also demonstrated that centralized identity management becomes much more powerful when combined with:

```text
Group Policy
+
Security Auditing
+
SIEM Monitoring
+
Vulnerability Management
```

---

# Active Directory Summary

```text
                         ad.home.com
                              │
                              ▼
                             DC1
                          10.0.0.6
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
           ▼                  ▼                  ▼
     Active Directory       DNS                DHCP
           │
           ▼
     Organizational Units
           │
    ┌──────┼────────┐
    │      │        │
    ▼      ▼        ▼
  Users  Admins  Workstations
                   │
                   ▼
             Group Policy
                   │
                   ▼
                Client01
                   │
                   ▼
          Security Auditing
                   │
                   ▼
                 Wazuh
```

DC1 provides the centralized identity and Windows-management foundation for the home lab.

---

# Related Documentation

[← pfSense Firewall and Routing Architecture](../03-pfsense/pfsense-architecture.md)

[← IP Addressing and Network Services](../01-network-infrastructure/ip-addressing.md)

[← Proxmox Virtualization Architecture](../02-proxmox/proxmox-architecture.md)

[← Main Project README](../README.md)
