# Active Directory Organizational Unit Design

## Overview

The Active Directory environment uses **Organizational Units (OUs)** to organize users, administrative accounts, and domain-joined workstations.

The Active Directory domain is:

```text
ad.home.com
```

with the NetBIOS domain name:

```text
HOMELAB
```

The primary OU structure created for the home lab is:

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

This structure provides a foundation for:

- Centralized administration
- Group Policy deployment
- User organization
- Administrative-account separation
- Workstation management
- Future delegation
- Security policy targeting

Rather than leaving all Active Directory objects in the default containers, the environment uses a dedicated structure designed around the purpose of each object.

---

# Active Directory Structure

The current logical structure is:

```text
ad.home.com
│
└── HomeLab
    │
    ├── Users
    │
    │   └── Standard domain users
    │
    ├── Admins
    │
    │   └── Administrative accounts
    │
    └── Workstations
        └── Domain-joined client computers
```

The top-level:

```text
HomeLab
```

OU groups the custom lab objects together.

This makes the environment easier to understand and manage as additional systems and users are added.

---

# What Is an Organizational Unit?

An Organizational Unit is a container within Active Directory.

OUs can contain objects such as:

```text
Users
Computers
Groups
Other Organizational Units
```

They are commonly used to organize directory objects and provide administrative boundaries for management and policy application.

Conceptually:

```text
Active Directory Domain
        │
        ▼
Organizational Unit
        │
   ┌────┼────┐
   │    │    │
   ▼    ▼    ▼
 Users PCs  Groups
```

---

# Why Use Organizational Units?

Without a structured OU design, Active Directory can quickly become difficult to manage.

For example:

```text
Domain
│
├── User
├── Computer
├── Admin
├── User
├── Computer
├── Service Account
└── Computer
```

This makes it harder to understand which objects should receive which policies.

Using OUs creates a structure such as:

```text
HomeLab
│
├── Users
│
├── Admins
└── Workstations
```

Each object type can then be managed according to its purpose.

---

# HomeLab OU

The primary custom OU is:

```text
HomeLab
```

This serves as the parent container for the custom domain structure.

Conceptually:

```text
ad.home.com
     │
     ▼
  HomeLab
```

The purpose of this OU is to keep home-lab objects organized separately from the default Active Directory containers.

---

# Users OU

Standard domain user accounts are organized under:

```text
HomeLab/Users
```

Conceptually:

```text
HomeLab
   │
   ▼
 Users
   │
   ├── Standard User
   ├── Standard User
   └── Future Users
```

This OU is intended for normal user identities rather than privileged administrative accounts.

---

# Why Separate Standard Users?

Standard users should not automatically receive administrative privileges.

Separating them into their own OU provides a foundation for applying user-specific policies.

Examples of future policies could include:

- Desktop restrictions
- User security settings
- Logon configuration
- Control Panel restrictions
- PowerShell restrictions where appropriate
- Drive mappings
- Security baselines

The exact policies can evolve as the lab grows.

---

# Admins OU

Administrative accounts are separated into:

```text
HomeLab/Admins
```

Conceptually:

```text
HomeLab
   │
   ▼
 Admins
   │
   ├── Administrative Account
   └── Future Administrative Accounts
```

This helps distinguish privileged identities from standard user accounts.

---

# Why Separate Administrative Accounts?

Administrative accounts represent a higher security risk than normal user accounts.

If an administrator account is compromised, an attacker may gain significantly more control over the environment.

Separating administrative accounts provides a foundation for stronger controls.

Conceptually:

```text
Standard User
     │
     ▼
Limited Permissions


Administrative User
     │
     ▼
Privileged Permissions
```

These account types should not necessarily receive identical security policies.

---

# Administrative Account Strategy

A good administrative model separates:

```text
Daily User Activity
```

from:

```text
Administrative Activity
```

Conceptually:

```text
Normal Account
     │
     └── Email / Browsing / Normal Work


Administrative Account
     │
     └── Server / Domain Administration
```

This reduces the amount of time privileged credentials are exposed during normal activity.

The dedicated Admins OU provides the structural foundation for this model.

---

# Workstations OU

Domain-connected client systems are organized under:

```text
HomeLab/Workstations
```

For example:

```text
HomeLab
   │
   ▼
Workstations
   │
   ▼
Client01
```

Client01 is joined to:

```text
ad.home.com
```

and can receive workstation-specific Group Policy settings.

---

# Why Separate Workstations?

Windows client systems often require different security settings from servers or domain controllers.

Examples include:

- Microsoft Defender configuration
- Windows Update policy
- Screen-lock settings
- Remote Desktop configuration
- Security auditing
- Process creation auditing
- Windows Firewall configuration

By keeping workstations in a dedicated OU, policies can be applied specifically to client systems.

---

# OU Structure and Group Policy

One of the primary reasons for creating OUs is Group Policy.

Conceptually:

```text
Group Policy Object
       │
       ▼
Linked to OU
       │
       ▼
Objects Inside OU
       │
       ▼
Receive Policy
```

For example:

```text
Workstation Security GPO
         │
         ▼
HomeLab/Workstations
         │
         ▼
      Client01
```

This allows policy to be targeted without applying the same configuration everywhere in the domain.

---

# Example Policy Targeting

Suppose a policy enables additional Windows security auditing.

Instead of applying it indiscriminately:

```text
Entire Domain
     │
     ▼
Every Object
```

it can be linked to:

```text
HomeLab/Workstations
```

resulting in:

```text
Security Auditing GPO
        │
        ▼
Workstations OU
        │
        ▼
Client01
```

This provides more controlled policy deployment.

---

# Users vs Computers in Group Policy

Group Policy contains two major configuration areas:

```text
Computer Configuration
```

and:

```text
User Configuration
```

Computer policies normally apply based on the computer object's location in Active Directory.

User policies normally apply based on the user's location.

For example:

```text
HomeLab/Workstations
        │
        ▼
Computer Configuration
```

while:

```text
HomeLab/Users
        │
        ▼
User Configuration
```

Understanding this distinction is important when troubleshooting why a policy did or did not apply.

---

# Group Policy Inheritance

Policies can be inherited through the Active Directory hierarchy.

Conceptually:

```text
Domain
  │
  ▼
HomeLab
  │
  ▼
Workstations
```

A policy linked higher in the hierarchy may affect objects lower in the structure depending on its configuration.

This makes OU planning important before a large number of GPOs are created.

---

# Avoiding One Giant OU

A simple environment could technically place everything into one OU:

```text
HomeLab
│
├── User
├── Admin
├── Client01
├── Server
└── Service Account
```

However, that reduces flexibility.

A structured model:

```text
HomeLab
│
├── Users
├── Admins
└── Workstations
```

makes it easier to create different policies for different types of objects.

---

# Organizational Units vs Groups

Organizational Units and security groups are not the same thing.

## Organizational Units

Primarily help with:

```text
Organization
Group Policy
Administrative Delegation
```

## Security Groups

Primarily help with:

```text
Permissions
Access Control
Authorization
```

For example:

```text
OU:
HomeLab/Workstations
```

can organize computer objects.

A security group can instead represent:

```text
Nessus Scan Admins
```

and grant specific permissions or membership.

---

# Example Difference

Consider a user who needs special access.

Moving the user into:

```text
Admins OU
```

does not automatically make the user an administrator.

Likewise, placing a computer in:

```text
Workstations OU
```

does not automatically grant the computer access to resources.

The OU determines organizational and policy placement.

Permissions are generally controlled through:

```text
Security Groups
ACLs
Local Groups
User Rights
```

---

# Security Group Example

The environment uses a domain group named:

```text
Nessus Scan Admins
```

This group supports the authenticated vulnerability-scanning configuration.

The purpose is very different from an OU.

Conceptually:

```text
Organizational Unit
      │
      └── Organizes Objects


Security Group
      │
      └── Assigns / Represents Access
```

Understanding this difference is fundamental to Active Directory administration.

---

# Service Accounts

The environment also includes a service account used for vulnerability scanning:

```text
svc_nessus
```

The account supports authenticated Nessus scans of Windows endpoints.

Service accounts should be treated differently from normal user accounts because they are used by applications or infrastructure services rather than interactive users.

A future OU structure may therefore include:

```text
HomeLab
│
├── Users
├── Admins
├── Workstations
└── Service Accounts
```

if additional service accounts are introduced.

This is a planned structural improvement rather than part of the current OU design.

---

# Current vs Future Structure

## Current

```text
ad.home.com
│
└── HomeLab
    │
    ├── Users
    ├── Admins
    └── Workstations
```

## Possible Future Design

```text
ad.home.com
│
└── HomeLab
    │
    ├── Users
    ├── Admins
    ├── Service Accounts
    ├── Workstations
    ├── Servers
    └── Security Systems
```

Additional OUs should only be created when they provide a real administrative or policy-management benefit.

---

# Avoiding Unnecessary Complexity

A larger OU structure is not automatically better.

For example:

```text
HomeLab
│
├── Department1
│   ├── Users
│   ├── Computers
│   └── Groups
├── Department2
│   ├── Users
│   ├── Computers
│   └── Groups
```

would add unnecessary complexity if the lab does not actually contain those departments.

The current design remains intentionally simple.

```text
Users
Admins
Workstations
```

are sufficient for the current environment while still allowing future expansion.

---

# Active Directory Users and Computers

OU management can be performed through:

```text
Active Directory Users and Computers
```

This console allows administrators to:

- Create OUs
- Create users
- Create groups
- Manage computer objects
- Move objects
- Disable accounts
- Reset passwords
- Review memberships
- Modify object properties

It provides the primary graphical interface for managing the current directory structure.

---

# Moving a Computer into the Workstations OU

When a computer first joins a domain, its object may initially appear in the default Computers container.

To apply workstation-specific organizational structure and Group Policy, the computer object can be moved into:

```text
HomeLab/Workstations
```

Conceptually:

```text
Domain Join
    │
    ▼
Computer Object Created
    │
    ▼
Move to Workstations OU
    │
    ▼
Workstation GPO Applies
```

---

# Client01

Client01 is the primary domain-joined workstation used for testing policies and security configurations.

Conceptually:

```text
ad.home.com
│
└── HomeLab
    │
    └── Workstations
        │
        └── Client01
```

This allows workstation-targeted security policies to be tested in a controlled environment.

---

# Group Policy Testing

After changing an OU or GPO configuration, Group Policy can be refreshed on a client using:

```powershell
gpupdate /force
```

Applied policies can then be reviewed using:

```powershell
gpresult /r
```

These commands help verify that the expected policy actually reached the endpoint.

---

# Why Verification Matters

Creating a GPO and linking it to an OU does not automatically prove that the policy applied successfully.

The complete path is:

```text
Create GPO
    │
    ▼
Configure Policy
    │
    ▼
Link to OU
    │
    ▼
Place Correct Object in OU
    │
    ▼
Update Group Policy
    │
    ▼
Verify on Endpoint
```

Each stage can affect the final result.

---

# Troubleshooting OU and GPO Problems

If a policy is not applying, several areas should be checked.

```text
Correct GPO?
    │
    ▼
Correct OU Link?
    │
    ▼
Correct User / Computer Object?
    │
    ▼
Correct OU?
    │
    ▼
Network Connectivity?
    │
    ▼
DNS Working?
    │
    ▼
DC1 Reachable?
    │
    ▼
gpupdate /force
    │
    ▼
gpresult /r
```

This provides a systematic troubleshooting process.

---

# Example Problem

Suppose Client01 does not receive a workstation security policy.

The first question should not automatically be:

```text
Is Group Policy broken?
```

Instead, verify:

```text
Is Client01 in HomeLab/Workstations?
```

If the computer object remains in the wrong container or OU, the intended GPO may never apply.

---

# OU Design and Security

OU design itself is not a security control that prevents network access.

However, it enables centralized security configuration.

For example:

```text
Workstations OU
      │
      ▼
Security GPO
      │
      ├── Defender
      ├── Auditing
      ├── Windows Update
      └── Other Controls
```

This makes the OU structure an important part of the overall security architecture.

---

# Administrative Separation

The Admins OU creates a logical distinction between normal and privileged accounts.

```text
HomeLab
│
├── Users
│   └── Standard Identities
│
└── Admins
    └── Privileged Identities
```

This provides a foundation for future administrative controls such as:

- Stronger authentication requirements
- Restricted logon locations
- Additional auditing
- Privileged workstation policies
- Tighter account management procedures

---

# Least Privilege

The OU design supports the broader principle of least privilege.

Users should receive:

```text
Only the permissions required for their role
```

rather than:

```text
Administrator access by default
```

The separation between normal users and administrative identities makes this principle easier to implement.

---

# Protecting Privileged Accounts

Administrative accounts should receive additional attention because they may have control over:

```text
Active Directory
DNS
DHCP
Group Policy
Domain Computers
Security Groups
```

A compromised privileged identity can therefore affect a significant portion of the environment.

Separating privileged accounts organizationally is one step toward a stronger administrative model.

---

# Future Administrative Tiering

As the home lab grows, the administrative model could become more restrictive.

For example:

```text
Administrative Workstation
        │
        ▼
Management VLAN
        │
        ▼
Administrative Account
        │
        ▼
Infrastructure
```

This would combine:

- Network segmentation
- Dedicated administrative identities
- Group Policy
- Restricted management access

into a more mature security design.

---

# Relationship to VLAN 99

The planned VLAN 99 management network can eventually complement the Active Directory administrative structure.

Conceptually:

```text
Admins OU
   │
   ▼
Privileged Identity
   │
   ▼
Jump / Admin Workstation
   │
   ▼
VLAN 99
   │
   ▼
Infrastructure Management
```

This would provide both identity-level and network-level separation for administration.

---

# Relationship to Wazuh

Administrative and user activity can also generate Windows security events.

```text
User / Administrator
       │
       ▼
Windows Activity
       │
       ▼
Event Logs
       │
       ▼
Wazuh Agent
       │
       ▼
Wazuh SIEM
```

This allows the security-monitoring layer to observe actions performed by identities managed through Active Directory.

---

# Security Group Changes

Security-group membership changes are especially important to monitor.

Examples of relevant Windows event IDs include:

```text
4728
4729
4732
4733
4756
4757
```

These events can indicate users being added to or removed from security groups.

Monitoring group membership is important because changes can affect user privileges.

---

# Example Privilege Monitoring Scenario

Consider:

```text
User Added to Privileged Group
          │
          ▼
Windows Security Event
          │
          ▼
Wazuh Agent
          │
          ▼
Wazuh
          │
          ▼
Security Alert
```

This connects Active Directory identity administration to security detection.

---

# Relationship to Nessus

Active Directory also supports the vulnerability-management architecture.

The:

```text
Nessus Scan Admins
```

group and:

```text
svc_nessus
```

service account support authenticated scanning of Windows systems.

This demonstrates that Active Directory is not only used for human users.

It also provides identity and access-control functions for infrastructure services.

---

# Delegation

Organizational Units can also be used for administrative delegation.

For example, in a larger environment:

```text
Help Desk
   │
   ▼
Permission to Manage Users OU
```

without granting:

```text
Full Domain Administration
```

The current home lab does not require complex delegation, but the OU design provides a foundation for learning this concept later.

---

# Why Delegation Matters

Without delegation, administrative access can become overly broad.

For example:

```text
Need to Reset Passwords
        │
        ▼
Domain Admin
```

would provide significantly more permission than required.

A better design could allow:

```text
Help Desk Account
        │
        ▼
Reset Password Permission
        │
        ▼
Users OU Only
```

This is another application of least privilege.

---

# OU Naming

Clear names are used for the current structure:

```text
HomeLab
Users
Admins
Workstations
```

Simple names make the environment easier to understand.

An administrator reviewing the directory can quickly determine the purpose of each container without needing additional explanation.

---

# Documentation Benefits

Documenting the OU structure provides several benefits.

It helps with:

- Group Policy troubleshooting
- Account administration
- Security reviews
- Future expansion
- Disaster recovery
- Portfolio documentation

For example, if a workstation is not receiving a policy, the documented OU structure provides an immediate reference for where that object is expected to reside.

---

# Current OU Architecture

```text
                    ad.home.com
                         │
                         ▼
                      HomeLab
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
        Users          Admins      Workstations
          │              │              │
          ▼              ▼              ▼
      Standard       Privileged      Client01
      Accounts        Accounts
```

This provides a simple but scalable foundation for centralized Windows administration.

---

# Future Improvements

Possible future improvements include:

- Dedicated Service Accounts OU
- Dedicated Servers OU
- More granular workstation OUs
- Additional Group Policy targeting
- Administrative delegation
- Privileged account restrictions
- Separate policies for administrators
- Additional computer-security baselines
- Integration with VLAN 99 management access
- Additional domain-joined systems

Changes should be driven by actual administrative requirements rather than creating unnecessary complexity.

---

# Skills Demonstrated

This portion of the project demonstrates practical experience with:

## Active Directory Administration

- Organizational Units
- User objects
- Computer objects
- Administrative accounts
- Security groups
- Service accounts

## Group Policy Design

- Policy targeting
- OU linking concepts
- User vs computer configuration
- Policy inheritance
- Endpoint verification

## Identity Security

- Standard vs privileged accounts
- Least privilege
- Administrative separation
- Service-account management
- Security-group monitoring

## Troubleshooting

- Object placement
- Group Policy application
- Domain connectivity
- DNS dependencies
- Policy verification

---

# What I Learned

Building the OU structure reinforced that Active Directory organization should support how the environment is actually administered.

OUs should not be created only to make the directory look organized.

They should help answer questions such as:

```text
Which users receive this policy?

Which computers receive this configuration?

Which accounts are privileged?

Where should new workstations be placed?

Where should future service accounts be managed?
```

The most important lesson was:

> **Organizational Units provide structure for administration and Group Policy, while security groups primarily provide access and authorization.**

Understanding that difference makes Active Directory easier to design and troubleshoot.

---

# Organizational Unit Summary

```text
                         ad.home.com
                              │
                              ▼
                           HomeLab
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
          Users             Admins         Workstations
            │                 │                 │
            ▼                 ▼                 ▼
     Standard Users      Privileged         Client01
                           Users                │
                                                ▼
                                         Workstation GPOs
```

The OU design provides the organizational foundation for centralized security policy, identity administration, and future expansion of the Windows domain.

---

# Related Documentation

[← Active Directory DNS and DHCP](dns-and-dhcp.md)

[← Windows Server 2025 and Active Directory](windows-server-2025.md)

[← IP Addressing and Network Services](../01-network-infrastructure/ip-addressing.md)

[← Main Project README](../README.md)
