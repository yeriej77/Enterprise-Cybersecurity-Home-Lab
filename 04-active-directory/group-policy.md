# Active Directory Group Policy

## Overview

The home lab uses **Group Policy** to centrally manage Windows security and operating-system configuration across domain-joined systems.

The Active Directory domain is:

```text
ad.home.com
```

with policies managed from:

```text
DC1
10.0.0.6
```

Group Policy provides centralized control over settings such as:

- Password policy
- Account lockout
- Microsoft Defender
- Windows Update
- Windows security auditing
- Process creation auditing
- Command-line process logging
- Windows Firewall configuration
- Vulnerability-scanning prerequisites

Instead of configuring each Windows endpoint manually, policies can be defined centrally and applied to domain-joined computers.

---

# Why Group Policy?

Without Group Policy, security settings would need to be configured individually.

For example:

```text
Client01
   │
   ├── Configure Defender manually
   ├── Configure auditing manually
   ├── Configure Windows Update manually
   └── Configure firewall manually
```

As additional systems are added, this quickly becomes difficult to maintain.

With Group Policy:

```text
                       DC1
                        │
                        ▼
                  Group Policy
                        │
             ┌──────────┼──────────┐
             │          │          │
             ▼          ▼          ▼
         Client01   Future PC   Future PC
```

Security settings can be deployed consistently from one centralized location.

---

# Group Policy Architecture

The current Active Directory structure includes:

```text
ad.home.com
│
└── HomeLab
    │
    ├── Users
    ├── Admins
    └── Workstations
```

Client systems such as Client01 are placed within:

```text
HomeLab/Workstations
```

This allows workstation-oriented policies to be targeted to the systems that require them.

---

# Policy Deployment Process

The general Group Policy workflow is:

```text
Create GPO
    │
    ▼
Configure Security Settings
    │
    ▼
Link GPO to Appropriate Scope
    │
    ▼
Client Contacts DC1
    │
    ▼
Policy Downloads
    │
    ▼
Windows Applies Settings
    │
    ▼
Verify on Endpoint
```

A policy is not considered successfully deployed until its effect is verified on the client.

---

# Computer vs User Configuration

Group Policy contains two major configuration areas.

```text
Computer Configuration
```

applies settings based primarily on the computer object.

```text
User Configuration
```

applies settings based primarily on the user object.

For workstation security, many of the controls in this lab are implemented through:

```text
Computer Configuration
```

because they apply to the Windows system regardless of which user signs in.

---

# Security Controls Implemented

Group Policy has been used to configure several security controls in the lab.

These include:

```text
Account Security
│
├── Password Policy
└── Account Lockout


Endpoint Security
│
├── Microsoft Defender
└── Windows Update


Security Monitoring
│
├── Process Creation Auditing
├── Command-Line Auditing
└── Security Group Management Auditing


Vulnerability Management Support
│
├── WMI Firewall Rules
├── SMB Firewall Access
└── Local Administrative Group Configuration
```

These policies support both system hardening and security monitoring.

---

# Password Policy

Domain account policies were configured through Active Directory.

The objective was to centrally control how domain credentials are managed rather than allowing each endpoint to determine its own password behavior.

Password policies can define requirements involving areas such as:

```text
Password Length
Password Complexity
Password History
Password Age
```

The exact values used in the environment should be referenced from the live Group Policy configuration when documenting individual settings.

---

# Why Password Policy Matters

A centralized password policy provides consistency.

Without centralized policy:

```text
Computer 1 → Different Rules
Computer 2 → Different Rules
Computer 3 → Different Rules
```

With Active Directory:

```text
Domain Password Policy
        │
        ▼
Domain Accounts
```

This ensures domain users follow the same baseline requirements.

---

# Account Lockout Policy

An account lockout policy was configured to reduce repeated password-guessing attempts.

The configured lockout threshold is:

```text
5 failed logon attempts
```

Conceptually:

```text
Failed Attempt 1
       │
Failed Attempt 2
       │
Failed Attempt 3
       │
Failed Attempt 4
       │
Failed Attempt 5
       │
       ▼
Account Locked
```

The policy was tested directly to confirm that the lockout occurred as expected.

---

# Why Account Lockout Matters

Without lockout controls:

```text
Attacker
   │
   ▼
Repeated Password Attempts
   │
   ▼
Unlimited Guessing
```

With an account lockout threshold:

```text
Repeated Failed Attempts
         │
         ▼
Account Lockout
```

This helps reduce the effectiveness of simple online password-guessing attacks.

---

# Policy Validation

The lockout configuration was not considered complete simply because it appeared in Group Policy Management.

It was tested from the endpoint.

The validation process was:

```text
Configure Policy
      │
      ▼
Apply Policy
      │
      ▼
Attempt Incorrect Password
      │
      ▼
Repeat Until Threshold
      │
      ▼
Account Locked
```

This confirmed that the configuration was actually being enforced.

---

# Microsoft Defender Policy

Microsoft Defender settings were also managed through Group Policy.

The purpose of centrally configuring Defender is to maintain a consistent security baseline across Windows endpoints.

Conceptually:

```text
DC1
 │
 ▼
Defender GPO
 │
 ▼
Client01
 │
 ▼
Microsoft Defender Configuration
```

This provides a foundation for standardized endpoint protection.

---

# Why Centralize Defender Settings?

If every endpoint is configured independently:

```text
Client01 → Configuration A

Client02 → Configuration B

Client03 → Configuration C
```

security posture can become inconsistent.

Centralized configuration provides:

```text
Security Baseline
       │
       ▼
All Managed Endpoints
```

This becomes increasingly valuable as the domain expands.

---

# Windows Update Policy

Windows Update was also configured through Group Policy.

The lab uses a scheduled update configuration with updates scheduled around:

```text
3:00 AM
```

The goal is to keep Windows endpoints patched while minimizing interruption during normal lab use.

Conceptually:

```text
DC1
 │
 ▼
Windows Update GPO
 │
 ▼
Client01
 │
 ▼
Scheduled Update Behavior
```

---

# Why Patch Management Matters

Operating-system updates help address:

- Security vulnerabilities
- Reliability issues
- Compatibility problems
- Microsoft Defender updates
- Operating-system bugs

Centralized update policy helps ensure that endpoint patching does not depend entirely on manual user action.

---

# Security Auditing

Windows security auditing was expanded so endpoint activity could be monitored by Wazuh.

One of the most important settings enabled was:

```text
Audit Process Creation
```

This generates:

```text
Windows Security Event ID 4688
```

when a new process is created.

---

# Process Creation Auditing

Process creation auditing provides visibility into programs executed on a Windows system.

For example:

```text
User Executes Command
        │
        ▼
Windows Starts Process
        │
        ▼
Event ID 4688
        │
        ▼
Windows Security Log
```

This provides useful telemetry for security monitoring.

---

# Event ID 4688

The Windows event used for process creation is:

```text
4688
```

This event can include information such as:

```text
New Process Name
Creator Process
Security Context
Process Information
```

When combined with command-line auditing, it becomes significantly more useful for detection engineering.

---

# Command-Line Process Auditing

Process creation auditing was configured to include command-line information.

Without command-line logging:

```text
Process:
powershell.exe
```

may not provide enough context.

With command-line information:

```text
Process:
powershell.exe

Command:
<executed command>
```

the event provides much more useful security telemetry.

---

# Why Command-Line Logging Matters

Attackers and administrators often use the same executable names.

For example:

```text
powershell.exe
cmd.exe
net.exe
```

The executable alone does not explain what action was performed.

Command-line information makes it possible to distinguish between:

```text
Normal Administration
```

and potentially:

```text
Suspicious Activity
```

based on what was actually executed.

---

# Connection to Wazuh

Windows Security events are collected by the Wazuh agent.

The monitoring path is:

```text
Windows Activity
      │
      ▼
Windows Security Log
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
      │
      ▼
Security Alert
```

This turns Group Policy auditing configuration into usable SIEM telemetry.

---

# Example Detection Path

One of the detection-engineering scenarios involved Windows command execution.

Conceptually:

```text
Command Executed
      │
      ▼
Process Created
      │
      ▼
Event ID 4688
      │
      ▼
Wazuh Agent
      │
      ▼
Custom Detection Rule
      │
      ▼
Alert
```

This demonstrates the relationship between endpoint policy and centralized detection.

---

# Security Group Management Auditing

Auditing was later expanded to include security-group management activity.

Relevant Windows Security events include:

```text
4728
4729
4732
4733
4756
4757
```

These events can identify when users are added to or removed from security groups.

---

# Why Group Membership Matters

Group membership changes can alter a user's permissions.

For example:

```text
Standard User
      │
      ▼
Added to Privileged Group
      │
      ▼
Increased Access
```

This makes privileged-group modification an important activity to monitor.

---

# Security Group Event Monitoring

The general detection path becomes:

```text
Group Membership Changed
         │
         ▼
Windows Security Event
         │
         ▼
Wazuh Agent
         │
         ▼
Wazuh SIEM
         │
         ▼
Security Investigation
```

This provides visibility into potentially sensitive identity changes.

---

# Sysmon

Windows auditing was supplemented with **Sysmon**.

Sysmon provides additional endpoint telemetry beyond standard Windows event logging.

One important event is:

```text
Sysmon Event ID 1
```

which records process creation.

Conceptually:

```text
Process Execution
      │
      ├── Windows Security 4688
      │
      └── Sysmon Event ID 1
```

Using multiple telemetry sources can provide stronger detection coverage.

---

# Why Use Both 4688 and Sysmon?

Standard Windows auditing and Sysmon provide overlapping but different visibility.

Using both allows:

```text
Windows Security Events
        +
Sysmon Telemetry
        =
Richer Endpoint Visibility
```

This provides multiple ways to observe process execution.

It also helps during troubleshooting if a detection is not appearing from one expected event source.

---

# Group Policy and Vulnerability Management

Group Policy was also used to prepare Windows endpoints for authenticated Tenable Nessus scanning.

Authenticated scanning requires the scanner to communicate with Windows management services.

The environment required several supporting controls.

---

# WMI Firewall Configuration

Windows Management Instrumentation firewall rules were enabled to allow authenticated scanning functionality.

The required Windows Firewall rules included:

```text
Windows Management Instrumentation (DCOM-In)
Windows Management Instrumentation (WMI-In)
Windows Management Instrumentation (ASync-In)
```

These were enabled through centralized policy for the appropriate systems.

---

# SMB Access for Nessus

Authenticated Windows vulnerability scanning also required SMB connectivity.

A specific inbound firewall rule was created for:

```text
TCP 445
```

with access restricted to the Nessus scanner:

```text
10.0.0.8
```

The rule was designed specifically for the vulnerability-management workflow rather than opening SMB broadly to every source.

Conceptually:

```text
Nessus
10.0.0.8
   │
   │ TCP 445
   ▼
Windows Client
```

---

# Why Restrict the Source?

A broad firewall rule such as:

```text
Allow TCP 445 from Any
```

would expose SMB more widely than required.

A narrower rule:

```text
Source: 10.0.0.8
Port:   TCP 445
```

better follows least privilege.

The scanner receives the access it requires without unnecessarily expanding the attack surface.

---

# Nessus Administrative Group

A domain security group was created:

```text
Nessus Scan Admins
```

The scanning account:

```text
svc_nessus
```

was added to this group.

Group Policy was then used to add the domain group to the appropriate local administrative context on the Windows endpoint.

Conceptually:

```text
svc_nessus
     │
     ▼
Nessus Scan Admins
     │
     ▼
Group Policy
     │
     ▼
Local Administrators
     │
     ▼
Client01
```

This supported authenticated vulnerability scanning.

---

# Why Use a Group?

Using a dedicated domain group is easier to manage than individually configuring the service account on every workstation.

Instead of:

```text
Client01 → Add svc_nessus manually

Client02 → Add svc_nessus manually

Client03 → Add svc_nessus manually
```

the design becomes:

```text
Nessus Scan Admins
       │
       ▼
Group Policy
       │
       ▼
Managed Windows Systems
```

This provides centralized control.

---

# Authenticated Scan Preparation

The Windows endpoint configuration required several components to work together:

```text
Nessus Scanner
10.0.0.8
     │
     ▼
TCP 445
     │
     ▼
Windows Firewall
     │
     ▼
WMI / SMB
     │
     ▼
svc_nessus
     │
     ▼
Required Permissions
```

If any one of these components is missing, Nessus may fall back to an unauthenticated scan.

---

# Group Policy Verification

After creating or changing a GPO, the policy can be refreshed using:

```powershell
gpupdate /force
```

This forces the client to request updated Group Policy settings.

---

# gpresult

Applied policies can be reviewed using:

```powershell
gpresult /r
```

This helps determine whether:

- The computer contacted the domain
- The expected GPO applied
- The object is in the correct OU
- Policy filtering affected the result

---

# Why `gpresult` Matters

A policy appearing correctly configured in Group Policy Management does not prove that the endpoint received it.

The correct validation process is:

```text
Configure GPO
     │
     ▼
Link GPO
     │
     ▼
gpupdate /force
     │
     ▼
gpresult /r
     │
     ▼
Verify Actual Setting
```

The final endpoint state is what matters.

---

# Example Group Policy Troubleshooting

Suppose Client01 does not receive a security setting.

The troubleshooting sequence is:

```text
Is Client01 Domain Joined?
        │
        ▼
Is DNS Correct?
        │
        ▼
Can Client Reach DC1?
        │
        ▼
Is Client01 in Correct OU?
        │
        ▼
Is GPO Linked?
        │
        ▼
gpupdate /force
        │
        ▼
gpresult /r
        │
        ▼
Check Resulting Configuration
```

This prevents unnecessary changes to the policy itself when the problem may be DNS or OU placement.

---

# DNS Dependency

Group Policy relies on the broader Active Directory infrastructure.

The client needs to locate the domain controller through DNS.

```text
Client01
    │
    ▼
DNS
10.0.0.6
    │
    ▼
Locate DC1
    │
    ▼
Retrieve Group Policy
```

Incorrect DNS configuration can therefore cause Group Policy problems.

---

# Network Dependency

Group Policy also depends on basic network connectivity.

The dependency chain is:

```text
Client Network
      │
      ▼
VLAN 10
      │
      ▼
DC1 Reachable
      │
      ▼
DNS Working
      │
      ▼
Active Directory
      │
      ▼
Group Policy
```

Higher-level policy troubleshooting should not begin until these lower layers are confirmed.

---

# GPO Change Methodology

Changes to security policy are made incrementally.

A useful process is:

```text
Document Current Setting
        │
        ▼
Change GPO
        │
        ▼
Apply to Test Client
        │
        ▼
gpupdate /force
        │
        ▼
Verify
        │
        ▼
Monitor for Side Effects
```

This reduces the risk of introducing multiple problems at once.

---

# Testing Before Expansion

Client01 serves as an important test workstation.

A policy can first be validated against:

```text
Client01
```

before being applied more broadly as the environment grows.

This mirrors the concept of staged configuration deployment.

---

# Group Policy and Security Baselines

The policies implemented in the lab begin to form a centralized Windows security baseline.

Conceptually:

```text
Windows Security Baseline
          │
          ├── Account Security
          ├── Endpoint Protection
          ├── Patch Management
          ├── Security Auditing
          ├── Firewall Configuration
          └── Vulnerability-Scanning Support
```

As additional controls are implemented, the baseline can continue to mature.

---

# Policy vs Detection

An important concept demonstrated by the lab is the distinction between:

```text
Prevention
```

and:

```text
Detection
```

For example:

```text
Account Lockout
      │
      └── Preventive Control
```

while:

```text
Process Creation Auditing
      │
      └── Detective Control
```

Both are necessary parts of a broader security architecture.

---

# Group Policy and Defense-in-Depth

Group Policy sits in the middle of several security layers.

```text
pfSense
   │
   ▼
Network Security
   │
   ▼
Active Directory
   │
   ▼
Group Policy
   │
   ▼
Endpoint Security
   │
   ▼
Windows Auditing / Sysmon
   │
   ▼
Wazuh
   │
   ▼
Security Detection
```

This provides security controls at both the network and endpoint layers.

---

# Security Architecture Example

Consider suspicious command execution on Client01.

```text
User / Attacker
      │
      ▼
Command Execution
      │
      ▼
Windows Auditing
Configured by GPO
      │
      ▼
Event ID 4688
      │
      ▼
Wazuh Agent
      │
      ▼
Custom Wazuh Detection
      │
      ▼
Alert
```

This demonstrates how Active Directory administration contributes directly to security monitoring.

---

# Policy Documentation

Each important Group Policy should eventually be documented with:

```text
GPO Name
Purpose
Scope
Linked OU
Important Settings
Security Impact
Validation Method
```

This makes the configuration easier to:

- Troubleshoot
- Rebuild
- Audit
- Explain
- Maintain

---

# Current Security Controls

The Group Policy environment currently supports controls including:

```text
Password Policy
        │
Account Lockout
        │
Microsoft Defender
        │
Windows Update
        │
Process Creation Auditing
        │
Command-Line Auditing
        │
Security Group Management Auditing
        │
Windows Firewall Rules
        │
Nessus Authenticated Scanning
```

These controls connect identity, endpoint hardening, monitoring, and vulnerability management.

---

# Future Group Policy Improvements

As the environment grows, additional policies could include:

- PowerShell logging
- Windows Firewall hardening
- Microsoft Defender Attack Surface Reduction rules
- Removable-media controls
- Administrative logon restrictions
- LAPS
- Enhanced audit policies
- RDP restrictions
- Security baseline policies
- Credential-protection settings
- Privileged account policies
- Dedicated server policies
- Dedicated administrative workstation policies

Future settings should be tested in the lab before being treated as production-ready controls.

---

# LAPS as a Future Improvement

One valuable future project would be implementing Windows LAPS.

The objective would be to avoid using the same local administrator password across multiple workstations.

Conceptually:

```text
Client01
Local Admin Password A

Client02
Local Admin Password B

Client03
Local Admin Password C
```

with passwords managed centrally through Active Directory.

This would reduce the risk associated with reused local administrator credentials.

---

# Future Administrative Policies

The planned management architecture could eventually combine:

```text
Admins OU
     │
     ▼
Administrative Accounts
     │
     ▼
Dedicated Admin Workstation
     │
     ▼
VLAN 99
     │
     ▼
Infrastructure
```

Group Policy could then enforce additional restrictions on both the administrative account and administrative workstation.

---

# Troubleshooting Commands

Useful commands for Group Policy troubleshooting include:

## Force Policy Update

```powershell
gpupdate /force
```

## Show Applied Policies

```powershell
gpresult /r
```

## Generate Detailed HTML Report

```powershell
gpresult /h C:\gpresult.html
```

## Verify Domain Controller

```powershell
nltest /dsgetdc:ad.home.com
```

## Verify DNS

```powershell
nslookup dc1.ad.home.com
```

## Show Network Configuration

```powershell
ipconfig /all
```

These commands help determine whether the issue exists in:

```text
Network
DNS
Active Directory
OU Structure
GPO Processing
Endpoint Configuration
```

---

# Lessons Learned

Implementing Group Policy provided practical experience with:

- Centralized Windows administration
- Domain password policy
- Account lockout
- Microsoft Defender management
- Windows Update management
- Advanced security auditing
- Process creation auditing
- Windows Firewall configuration
- Group-based administration
- Vulnerability-scanning preparation
- Policy troubleshooting
- Endpoint validation

The biggest lesson was:

> **A Group Policy is not successfully implemented until the endpoint proves that it received and enforced the setting.**

Creating the GPO is only the beginning.

The complete process is:

```text
Design
  │
  ▼
Configure
  │
  ▼
Deploy
  │
  ▼
Verify
  │
  ▼
Monitor
```

---

# Group Policy Summary

```text
                           DC1
                        10.0.0.6
                            │
                            ▼
                       Group Policy
                            │
         ┌──────────────────┼───────────────────┐
         │                  │                   │
         ▼                  ▼                   ▼
  Account Security    Endpoint Security    Security Auditing
         │                  │                   │
         ├─ Password        ├─ Defender        ├─ Event 4688
         └─ Lockout         └─ Updates         ├─ Command Line
                                                └─ Group Changes
                                                     │
                                                     ▼
                                                  Client01
                                                     │
                                   ┌─────────────────┼───────────────┐
                                   │                 │               │
                                   ▼                 ▼               ▼
                                Wazuh             Nessus        AD Management
```

Group Policy provides the centralized configuration layer connecting Active Directory, endpoint security, SIEM monitoring, and vulnerability management.

---

# Related Documentation

[← Active Directory Organizational Unit Design](organizational-units.md)

[← Active Directory DNS and DHCP](dns-and-dhcp.md)

[← Windows Server 2025 and Active Directory](windows-server-2025.md)

[← Main Project README](../README.md)
