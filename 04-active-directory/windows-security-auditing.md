# Windows Security Auditing and Sysmon

## Overview

The home lab uses **Windows Security Auditing and Sysmon** to generate detailed endpoint telemetry for centralized monitoring in Wazuh.

The objective is not simply to collect Windows logs.

The goal is to create a complete detection pipeline:

```text
User / Process Activity
        │
        ▼
Windows Security Auditing
        │
        +
      Sysmon
        │
        ▼
Windows Event Logs
        │
        ▼
Wazuh Agent
        │
        ▼
Wazuh SIEM
        │
        ▼
Detection Rules
        │
        ▼
Security Alerts
```

This provides visibility into activity occurring on systems such as:

- DC1
- Client01

and creates the telemetry required for custom detection engineering.

---

# Monitored Windows Systems

The primary Windows systems monitored in the environment include:

| System | Role | Monitoring |
|---|---|---|
| DC1 | Windows Server 2025 Domain Controller | Windows Security Logs + Sysmon + Wazuh |
| Client01 | Domain-joined Windows workstation | Windows Security Logs + Wazuh |

DC1 is located at:

```text
10.0.0.6
```

and performs:

```text
Active Directory
DNS
DHCP
Group Policy
```

Because of its critical role, monitoring activity on the domain controller is especially important.

---

# Why Windows Auditing Matters

Windows performs thousands of actions during normal operation.

Without appropriate auditing, many security-relevant actions may not provide enough information for investigation.

For example:

```text
User Executes Command
       │
       ▼
Windows Performs Action
       │
       ▼
No Useful Audit Event
```

With auditing enabled:

```text
User Executes Command
       │
       ▼
Windows Performs Action
       │
       ▼
Security Event Generated
       │
       ▼
Wazuh Collects Event
       │
       ▼
Detection Logic Evaluates Activity
```

This transforms normal operating-system activity into security telemetry.

---

# Audit Strategy

The auditing architecture focuses on events that can provide useful evidence of:

```text
Process Execution
Account Creation
Security Group Changes
Administrative Activity
Potential Privilege Changes
```

The objective is to collect enough information to answer questions such as:

```text
What process executed?

Who executed it?

What command was used?

Was an account created?

Was a user added to a privileged group?

What system generated the event?
```

---

# Windows Security Event Log

Windows stores security-related audit events in:

```text
Windows Logs
└── Security
```

This log contains events related to:

- Authentication
- Account management
- Process creation
- Security-group changes
- Policy changes
- Privilege use

The Wazuh agent can collect relevant Windows events and forward them to the SIEM.

---

# Process Creation Auditing

One of the first major auditing controls implemented in the lab was:

```text
Audit Process Creation
```

This generates:

```text
Windows Security Event ID 4688
```

when Windows creates a new process.

---

# Event ID 4688

Event ID:

```text
4688
```

represents:

```text
A new process has been created
```

Conceptually:

```text
User Executes Program
        │
        ▼
Windows Creates Process
        │
        ▼
Event ID 4688
        │
        ▼
Security Log
```

Examples of processes that may appear include:

```text
cmd.exe
powershell.exe
net.exe
whoami.exe
```

---

# Why Process Creation Is Important

Process execution is one of the most useful sources of endpoint security telemetry.

Many administrative and attacker activities involve launching processes.

For example:

```text
Command Shell
PowerShell
Account Management Utilities
Network Utilities
System Administration Tools
```

A process creation event provides evidence that an executable actually ran.

---

# Command-Line Auditing

Process-name logging by itself provides limited context.

For example:

```text
powershell.exe
```

only tells us that PowerShell started.

It does not explain what PowerShell was asked to do.

The environment therefore enables command-line information for process creation events.

Conceptually:

```text
Process:
powershell.exe

Command Line:
<command executed>
```

This provides significantly more context for investigations.

---

# Why Command-Line Logging Matters

Consider:

```text
net.exe
```

The executable alone does not tell an analyst what happened.

The command could relate to:

```text
User Enumeration
Group Enumeration
Account Management
Network Administration
```

Command-line logging helps distinguish between these activities.

This becomes especially valuable when creating SIEM detection rules.

---

# Process Creation Flow

The complete process-monitoring path is:

```text
User
  │
  ▼
Command
  │
  ▼
Windows Process
  │
  ▼
Security Event 4688
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
Alert
```

This was one of the first telemetry pipelines built in the home lab.

---

# Verifying Process Creation Auditing

The configured audit policy can be reviewed from Windows using:

```powershell
auditpol /get /subcategory:"Process Creation"
```

The expected configuration should indicate that process creation auditing is enabled.

A broader view can be obtained using:

```powershell
auditpol /get /category:*
```

---

# Generating a Test Event

A simple command can be executed to generate process activity.

For example:

```powershell
whoami
```

Windows should create a process for:

```text
whoami.exe
```

The corresponding Security event can then be reviewed.

---

# Viewing Event ID 4688

Recent process creation events can be reviewed with PowerShell:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4688
} -MaxEvents 10
```

This provides a quick way to confirm that process creation events are being generated before troubleshooting the SIEM.

---

# Why Validate Locally First?

If Wazuh does not show an expected event, the first question should be:

```text
Did Windows generate the event?
```

rather than immediately assuming:

```text
Wazuh is broken.
```

The troubleshooting order should be:

```text
Activity Occurs
      │
      ▼
Windows Generates Event?
      │
      ▼
Wazuh Agent Collects Event?
      │
      ▼
Wazuh Server Receives Event?
      │
      ▼
Rule Matches?
```

This makes detection troubleshooting much more systematic.

---

# Account Creation Monitoring

Account creation is another important identity-security event.

Windows Security Event ID:

```text
4720
```

represents:

```text
A user account was created
```

Account creation is important to monitor because unauthorized users can potentially be created to establish persistence.

---

# Account Creation Detection Flow

Conceptually:

```text
Account Created
      │
      ▼
Windows Event 4720
      │
      ▼
Wazuh Agent
      │
      ▼
Wazuh
      │
      ▼
Security Alert / Investigation
```

The context of the event determines whether the activity is legitimate administration or suspicious behavior.

---

# Security Group Management Auditing

The auditing configuration was later expanded to monitor changes to Windows security groups.

Relevant event IDs include:

| Event ID | Activity |
|---:|---|
| `4728` | Member added to a security-enabled global group |
| `4729` | Member removed from a security-enabled global group |
| `4732` | Member added to a security-enabled local group |
| `4733` | Member removed from a security-enabled local group |
| `4756` | Member added to a security-enabled universal group |
| `4757` | Member removed from a security-enabled universal group |

These events provide visibility into changes that may affect user privileges.

---

# Why Group Changes Matter

A user can gain significant privileges simply by being added to a group.

For example:

```text
Standard User
      │
      ▼
Added to Privileged Group
      │
      ▼
Additional Permissions
```

The account itself may not change.

Instead, its authorization changes because of group membership.

Monitoring group changes therefore helps detect potential privilege escalation or unauthorized administrative activity.

---

# Group Change Monitoring Flow

```text
Administrator / Attacker
          │
          ▼
Changes Group Membership
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
Alert / Investigation
```

This is particularly important on a domain controller.

---

# Domain Controller Monitoring

DC1 is a high-value monitoring target because it manages:

```text
Users
Groups
Authentication
DNS
DHCP
Group Policy
```

A suspicious event occurring on a normal workstation may affect one endpoint.

A suspicious administrative event occurring on a domain controller could potentially affect the entire domain.

This makes DC1 telemetry especially valuable.

---

# Sysmon

Standard Windows Security auditing was supplemented with **Microsoft Sysmon**.

Sysmon provides additional endpoint telemetry and writes its events into a dedicated Windows Event Log.

The log is located at:

```text
Applications and Services Logs
└── Microsoft
    └── Windows
        └── Sysmon
            └── Operational
```

---

# Sysmon Installation on DC1

Sysmon was installed on DC1 using Microsoft Sysinternals.

A tools directory was first created:

```powershell
mkdir C:\Tools
```

The directory was then opened:

```powershell
cd C:\Tools
```

Sysmon was downloaded:

```powershell
Invoke-WebRequest `
  -Uri "https://download.sysinternals.com/files/Sysmon.zip" `
  -OutFile "Sysmon.zip"
```

The archive was extracted:

```powershell
Expand-Archive .\Sysmon.zip -DestinationPath .\Sysmon -Force
```

The Sysmon directory was opened:

```powershell
cd C:\Tools\Sysmon
```

and Sysmon was installed using:

```powershell
.\Sysmon64.exe -accepteula -i
```

The installed version during this phase of the project was:

```text
Sysmon v15.22
```

---

# Sysmon Event ID 1

One of the most useful Sysmon events is:

```text
Event ID 1
```

which represents:

```text
Process Create
```

This overlaps conceptually with Windows Security Event ID 4688 but can provide additional endpoint context.

---

# Windows 4688 vs Sysmon Event 1

The environment can observe process execution from two telemetry sources.

```text
Process Executes
      │
      ├───────────────┐
      │               │
      ▼               ▼
Windows 4688      Sysmon Event 1
      │               │
      └───────┬───────┘
              ▼
             Wazuh
```

This creates richer visibility than depending on only one event source.

---

# Why Use Both?

Windows Security auditing is built into the operating system and integrates closely with domain security policy.

Sysmon provides additional endpoint-focused telemetry.

Together:

```text
Windows Security Logs
         +
       Sysmon
         =
Richer Endpoint Visibility
```

This can improve both detection and investigation.

---

# Verifying Sysmon

Sysmon process creation events can be reviewed with:

```powershell
Get-WinEvent `
  -FilterHashtable @{
      LogName='Microsoft-Windows-Sysmon/Operational'
      Id=1
  } `
  -MaxEvents 10
```

This provides a quick command-line method for verifying that Sysmon is generating process events.

---

# Event Viewer Validation

Events can also be reviewed using Event Viewer.

For Security auditing:

```text
Event Viewer
└── Windows Logs
    └── Security
```

For Sysmon:

```text
Event Viewer
└── Applications and Services Logs
    └── Microsoft
        └── Windows
            └── Sysmon
                └── Operational
```

Validating events locally is an important troubleshooting step before investigating log forwarding.

---

# Wazuh Agent

Wazuh agents are installed on both:

```text
DC1
Client01
```

The Windows agents forward endpoint telemetry toward the Wazuh infrastructure.

The monitored flow is:

```text
Windows
   │
   ▼
Event Logs
   │
   ▼
Wazuh Agent
   │
   ▼
Wazuh Server
10.0.0.7
```

---

# Wazuh Infrastructure

The Wazuh server operates at:

```text
10.0.0.7
```

The environment includes Wazuh components used for:

```text
Event Collection
Event Processing
Indexing
Dashboard Visualization
Security Detection
```

The Windows auditing configuration provides the raw telemetry that these components analyze.

---

# Telemetry vs Detection

Collecting an event and detecting suspicious behavior are not the same thing.

For example:

```text
Event ID 4688 Received
```

only confirms:

```text
A process executed.
```

Detection logic must determine whether the activity is security-relevant.

Conceptually:

```text
Raw Event
   │
   ▼
Detection Rule
   │
   ├── Normal → No Important Alert
   │
   └── Suspicious → Security Alert
```

This distinction became important when building custom Wazuh rules.

---

# Detection Engineering

Once Windows telemetry was successfully reaching Wazuh, custom detection rules could be developed.

The workflow became:

```text
Generate Activity
      │
      ▼
Confirm Windows Event
      │
      ▼
Confirm Wazuh Ingestion
      │
      ▼
Inspect Event Fields
      │
      ▼
Build Detection Rule
      │
      ▼
Test
      │
      ▼
Tune
      │
      ▼
Validate Alert
```

This is a much stronger process than writing a rule before understanding the underlying telemetry.

---

# Example Command Detection

A command-related detection can conceptually follow:

```text
User Executes Command
        │
        ▼
Process Created
        │
        ▼
4688 / Sysmon Event 1
        │
        ▼
Wazuh Receives Event
        │
        ▼
Rule Evaluates Process / Command
        │
        ▼
Alert
```

This approach was used as the basis for the custom Windows command detections developed later in the project.

---

# Detection Rule Troubleshooting

Custom detection rules do not always work correctly on the first attempt.

Troubleshooting requires determining which stage failed.

```text
Was Activity Generated?
        │
        ▼
Was Windows Event Generated?
        │
        ▼
Did Wazuh Receive It?
        │
        ▼
Did Decoder Parse It?
        │
        ▼
Did Rule Conditions Match?
        │
        ▼
Was Alert Created?
```

This layered approach proved extremely useful during Wazuh rule development.

---

# Event Source Differences

An expected process event may appear from:

```text
Windows Security
```

or:

```text
Sysmon
```

depending on the configured telemetry and collection path.

During detection development, understanding the actual event provider became important.

A rule designed for one event source may not correctly match another even when both describe similar activity.

---

# Why Event Fields Matter

Detection rules should be based on the fields actually present in the event.

Examples may include information describing:

```text
Process Name
Command Line
User
Parent Process
Computer
Event ID
Provider
```

Before creating a detection rule, the raw event should be inspected to determine which fields are available.

---

# Example Investigation Questions

When reviewing suspicious process activity, useful questions include:

```text
Which system generated the event?

Which account executed the process?

What executable ran?

What command-line arguments were used?

What process launched it?

Was the activity expected?

Did related identity changes occur?
```

Combining Windows Security and Sysmon telemetry can provide additional context for these questions.

---

# Security Group Investigation

For group-management activity, important questions include:

```text
Which group changed?

Which user was added or removed?

Who performed the change?

Which system generated the event?

Was the change authorized?

Did the account receive elevated permissions?
```

These questions help turn a raw Windows event into an investigation.

---

# Defense-in-Depth

Windows auditing is one layer of the broader lab architecture.

```text
Network Segmentation
        │
        ▼
      pfSense
        │
        ▼
Active Directory
        │
        ▼
Group Policy
        │
        ▼
Windows Security Auditing
        │
        +
      Sysmon
        │
        ▼
       Wazuh
        │
        ▼
Detection and Investigation
```

Each layer provides a different security capability.

---

# Prevention vs Detection

The environment contains both preventive and detective controls.

For example:

```text
Account Lockout
      │
      └── Prevention / Mitigation
```

while:

```text
Event ID 4688
      │
      └── Detection Visibility
```

and:

```text
Wazuh Custom Rules
      │
      └── Detection Logic
```

Security requires both preventing what can be prevented and detecting what still occurs.

---

# Troubleshooting Missing Events

If an expected alert is missing, troubleshooting begins with the source.

```text
No Wazuh Alert
     │
     ▼
Check Windows Event Viewer
     │
     ├── Event Missing
     │       │
     │       ▼
     │  Audit Configuration Problem
     │
     └── Event Present
             │
             ▼
       Check Wazuh Agent
             │
             ▼
       Check Wazuh Ingestion
             │
             ▼
       Check Detection Rule
```

This prevents unnecessary changes to Wazuh when Windows never generated the event.

---

# Auditing Verification Commands

Useful Windows commands include:

## Review Process Creation Policy

```powershell
auditpol /get /subcategory:"Process Creation"
```

## Review Audit Policy

```powershell
auditpol /get /category:*
```

## Review Recent 4688 Events

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4688
} -MaxEvents 10
```

## Review Recent Account Creation Events

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4720
} -MaxEvents 10
```

## Review Sysmon Process Events

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-Sysmon/Operational'
    Id=1
} -MaxEvents 10
```

These commands help validate telemetry directly from the endpoint.

---

# Group Policy Verification

Because several auditing settings are deployed through Group Policy, policy application should also be verified.

Refresh Group Policy using:

```powershell
gpupdate /force
```

Then review applied policies:

```powershell
gpresult /r
```

A more detailed HTML report can be generated using:

```powershell
gpresult /h C:\gpresult.html
```

---

# Troubleshooting Order

A reliable troubleshooting sequence is:

```text
1. Verify Domain Connectivity
          │
          ▼
2. Verify Group Policy Applied
          │
          ▼
3. Verify Audit Policy
          │
          ▼
4. Generate Test Activity
          │
          ▼
5. Verify Local Windows Event
          │
          ▼
6. Verify Wazuh Agent
          │
          ▼
7. Verify Event in Wazuh
          │
          ▼
8. Verify Detection Rule
          │
          ▼
9. Verify Alert
```

This makes it easier to determine exactly where a detection pipeline is failing.

---

# Logging Considerations

Increasing audit visibility also increases the amount of telemetry generated.

More telemetry can mean:

```text
More Events
   │
   ▼
More Storage
   │
   ▼
More SIEM Processing
```

Therefore, logging should focus on events that provide useful security value.

Collecting everything without a purpose can make meaningful events harder to find.

---

# Sensitive Data in Command Lines

Command-line logging can occasionally capture sensitive information if users place secrets directly into commands.

For example:

```text
Password
Token
API Key
Credential
```

should not be entered directly into command-line arguments when avoidable.

This is both an operational and security consideration when enabling detailed process logging.

---

# GitHub Documentation Safety

Raw Windows logs and screenshots should be reviewed before being committed to a public repository.

Sensitive information that should not be published includes:

```text
Passwords
Private Keys
Authentication Tokens
VPN Credentials
Personal Information
Sensitive Internal Credentials
```

The portfolio should demonstrate security monitoring without exposing information that could weaken the environment.

---

# Snapshots

Snapshots were taken around important monitoring milestones.

Examples include:

```text
Wazuh Agent Installation
Group Policy Changes
Custom Detection Rules
Sysmon Installation
```

This provides known-good recovery points while experimenting with endpoint monitoring and detection configuration.

---

# Current Monitoring State

The current Windows monitoring architecture includes:

```text
Windows Server 2025
       │
       ├── Windows Security Events
       │       │
       │       ├── 4688 Process Creation
       │       ├── 4720 Account Creation
       │       └── Group Management Events
       │
       ├── Sysmon
       │       │
       │       └── Event ID 1 Process Creation
       │
       ▼
    Wazuh Agent
       │
       ▼
     Wazuh SIEM
       │
       ▼
 Custom Detection Rules
```

The same architecture can be expanded as additional Windows endpoints are added.

---

# Future Improvements

Future auditing and monitoring improvements may include:

- PowerShell Script Block Logging
- PowerShell Module Logging
- Additional Sysmon configuration
- Logon event monitoring
- Privileged logon monitoring
- Scheduled task monitoring
- Service creation monitoring
- Registry-change monitoring
- Additional account-management detections
- Additional privilege-escalation detections
- Wazuh rule tuning
- MITRE ATT&CK mapping
- Alert severity tuning
- Centralized detection documentation

These improvements will be added incrementally and tested before being treated as reliable detections.

---

# Skills Demonstrated

This portion of the project demonstrates practical experience with:

## Windows Security

- Windows Security Auditing
- Advanced Audit Policy
- Event Viewer
- Process creation monitoring
- Account-management auditing
- Security-group auditing

## Sysmon

- Sysinternals Sysmon
- Sysmon installation
- Sysmon Event ID 1
- Process telemetry
- Event-channel validation

## Active Directory

- Group Policy deployment
- Domain security monitoring
- User and group activity
- Privileged-change visibility

## SIEM

- Endpoint log forwarding
- Wazuh agents
- Event ingestion
- Detection-rule development
- Alert validation

## Detection Engineering

- Telemetry validation
- Event-field analysis
- Rule testing
- Detection troubleshooting
- Multiple telemetry sources

---

# What I Learned

Implementing Windows auditing and Sysmon reinforced that effective detection starts at the endpoint.

A SIEM cannot detect activity that was never logged.

The complete process is:

```text
Configure Logging
       │
       ▼
Generate Telemetry
       │
       ▼
Collect Telemetry
       │
       ▼
Understand the Event
       │
       ▼
Write Detection Logic
       │
       ▼
Test
       │
       ▼
Tune
       │
       ▼
Alert
```

One of the most important lessons was:

> **Before troubleshooting a detection rule, verify that the endpoint generated the event you expect.**

Another major lesson was:

> **Telemetry and detection are not the same thing. Logs provide evidence; detection rules give that evidence security meaning.**

---

# Windows Security Monitoring Summary

```text
                 Windows Endpoint
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
 Windows Security Logs            Sysmon
          │                         │
    ┌─────┼─────┐                   │
    │     │     │                   │
   4688  4720  Group               Event 1
              Changes               │
    │     │     │                   │
    └─────┴─────┴──────────┬────────┘
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
                     Security Alerts
```

Windows Security Auditing and Sysmon provide the endpoint telemetry foundation for the detection-engineering work performed in the Wazuh SIEM.

---

# Related Documentation

[← Active Directory Group Policy](group-policy.md)

[← Active Directory Organizational Unit Design](organizational-units.md)

[← Windows Server 2025 and Active Directory](windows-server-2025.md)

[← Main Project README](../README.md)
