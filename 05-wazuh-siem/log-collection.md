# Wazuh Log Collection and Telemetry Pipeline

## Overview

The home lab uses Wazuh to collect and centralize security telemetry from Windows systems.

The primary objective is to take endpoint activity that normally exists only inside Windows Event Viewer and make it searchable and usable inside the SIEM.

The current monitoring pipeline includes:

```text
Windows Security Logs
        +
      Sysmon
        │
        ▼
   Wazuh Agent
        │
        ▼
   Wazuh Manager
      10.0.0.7
        │
        ▼
 Event Processing
        │
        ▼
 Wazuh Indexer
        │
        ▼
 Wazuh Dashboard
```

The primary Windows endpoints currently sending telemetry are:

```text
DC1
Client01
```

These logs provide the foundation for the custom detection rules developed later in the project.

---

# Log Collection Architecture

The complete collection path is:

```text
Security Activity
       │
       ▼
Windows Generates Event
       │
       ▼
Windows Event Log
       │
       ▼
Wazuh Agent
       │
       ▼
Network Transport
       │
       ▼
Wazuh Manager
       │
       ▼
Parsing / Rule Evaluation
       │
       ▼
Indexer
       │
       ▼
Dashboard
```

Every stage must function correctly for an event to become useful security telemetry.

---

# Current Log Sources

The environment currently uses two major Windows telemetry sources.

```text
Windows Security Event Log
```

and:

```text
Microsoft Sysmon
```

These sources provide overlapping but complementary visibility into activity occurring on Windows systems.

---

# Windows Security Event Log

Windows Security auditing provides visibility into operating-system and identity activity.

The relevant log is:

```text
Windows Logs
└── Security
```

Events collected from this log include activity such as:

- Process creation
- Account creation
- Group membership changes
- Administrative actions
- Authentication-related events

The exact visibility depends on the Windows audit policies enabled through Group Policy.

---

# Process Creation Auditing

One of the primary Windows events used in the lab is:

```text
Event ID 4688
```

which represents:

```text
A new process has been created
```

This event provides visibility into programs launched on the endpoint.

Examples include:

```text
powershell.exe
cmd.exe
net.exe
whoami.exe
```

The general collection path is:

```text
Process Executes
      │
      ▼
Windows Event 4688
      │
      ▼
Wazuh Agent
      │
      ▼
Wazuh Server
      │
      ▼
Search / Detection
```

---

# Command-Line Information

Process creation auditing becomes much more useful when command-line information is included.

For example:

```text
Process:
net.exe
```

provides limited context.

With command-line logging:

```text
Process:
net.exe

Command Line:
net user ...
```

the SIEM has more information available for detection and investigation.

This is especially important when building custom rules around administrative commands.

---

# Why Command-Line Visibility Matters

The same executable can be used for legitimate or suspicious purposes.

For example:

```text
powershell.exe
```

may be used by an administrator for normal management.

It can also be used to execute malicious scripts.

The executable name alone does not provide enough context.

The command line can help answer:

```text
What exactly was executed?
```

which makes detection logic more precise.

---

# Account Creation Events

The environment also monitors account-management activity.

One important Windows event is:

```text
Event ID 4720
```

which represents:

```text
A user account was created
```

The event flow is:

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
Investigation / Alert
```

Monitoring account creation is valuable because unauthorized accounts can be used for persistence.

---

# Security Group Management Events

Windows auditing was expanded to monitor security-group membership changes.

Important event IDs include:

| Event ID | Activity |
|---:|---|
| `4728` | Member added to a security-enabled global group |
| `4729` | Member removed from a security-enabled global group |
| `4732` | Member added to a security-enabled local group |
| `4733` | Member removed from a security-enabled local group |
| `4756` | Member added to a security-enabled universal group |
| `4757` | Member removed from a security-enabled universal group |

These events are particularly important on DC1.

---

# Why Group Membership Changes Matter

Permissions in Active Directory are often granted through group membership.

Conceptually:

```text
Standard User
      │
      ▼
Added to Security Group
      │
      ▼
New Permissions
```

If the group is privileged, the security impact can be significant.

This makes group-management events valuable SIEM telemetry.

---

# DC1 Telemetry

DC1 is one of the most important log sources in the environment.

```text
Hostname: DC1
IP Address: 10.0.0.6
Role: Windows Server 2025 Domain Controller
```

DC1 provides:

```text
Active Directory
DNS
DHCP
Group Policy
```

Logs from DC1 can therefore contain activity related to:

- Users
- Groups
- Authentication
- Administrative changes
- Domain services
- Process execution

---

# Client01 Telemetry

Client01 is the primary domain-joined workstation used for endpoint testing.

It provides telemetry related to:

- User activity
- Process execution
- PowerShell
- Command Prompt
- Administrative utilities
- Detection testing

This makes Client01 useful for safely generating known events and verifying Wazuh behavior.

---

# Sysmon

Microsoft Sysmon was added to provide additional endpoint telemetry.

Sysmon writes events to:

```text
Applications and Services Logs
└── Microsoft
    └── Windows
        └── Sysmon
            └── Operational
```

On DC1, Sysmon was installed as:

```text
Sysmon v15.22
```

---

# Sysmon Event ID 1

One of the primary Sysmon events used in the lab is:

```text
Event ID 1
```

which represents:

```text
Process Create
```

The path is:

```text
Process Executes
      │
      ▼
Sysmon
      │
      ▼
Event ID 1
      │
      ▼
Wazuh Agent
      │
      ▼
Wazuh
```

---

# Windows 4688 vs Sysmon Event 1

Both Windows Security auditing and Sysmon can provide process-creation visibility.

```text
                     Process Execution
                            │
               ┌────────────┴────────────┐
               │                         │
               ▼                         ▼
       Windows Security              Sysmon
          Event 4688                Event ID 1
               │                         │
               └────────────┬────────────┘
                            ▼
                         Wazuh
```

This provides more than one way to observe process activity.

---

# Why Use Multiple Telemetry Sources?

Multiple sources improve troubleshooting and detection coverage.

For example:

```text
Expected 4688 Missing
        │
        ▼
Check Sysmon Event 1
```

If Sysmon contains the activity, the problem may relate to:

```text
Windows Audit Policy
```

rather than:

```text
Process Execution
```

This distinction proved useful during detection development.

---

# Telemetry Validation Philosophy

The home lab follows a simple rule:

> **Verify the event at the source before troubleshooting the SIEM.**

The validation sequence is:

```text
Generate Activity
      │
      ▼
Check Windows Event Viewer
      │
      ▼
Confirm Event Exists
      │
      ▼
Search Wazuh
      │
      ▼
Confirm Event Arrived
```

If the event does not exist locally, Wazuh cannot collect it.

---

# Local Windows Validation

Process creation events can be checked locally using PowerShell.

For Event ID 4688:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4688
} -MaxEvents 10
```

This helps determine whether Windows auditing is working before investigating the Wazuh agent.

---

# Sysmon Validation

Sysmon process events can be checked using:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-Sysmon/Operational'
    Id=1
} -MaxEvents 10
```

If the event appears locally, the next step is confirming that Wazuh collects it.

---

# Account Creation Validation

Recent account-creation events can be reviewed with:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4720
} -MaxEvents 10
```

This is useful when testing identity-related detections.

---

# Audit Policy Validation

The Windows audit configuration should also be verified.

For process creation:

```powershell
auditpol /get /subcategory:"Process Creation"
```

A broader view can be obtained using:

```powershell
auditpol /get /category:*
```

This confirms whether Windows is configured to generate the required events.

---

# Group Policy Dependency

Several Windows auditing settings are deployed through Group Policy.

The dependency chain is:

```text
DC1
  │
  ▼
Group Policy
  │
  ▼
Client
  │
  ▼
Audit Policy
  │
  ▼
Windows Events
```

If Group Policy does not apply correctly, the expected audit events may never be generated.

---

# Group Policy Validation

Group Policy can be refreshed using:

```powershell
gpupdate /force
```

Applied policies can then be checked using:

```powershell
gpresult /r
```

This helps verify that the auditing configuration actually reached the endpoint.

---

# Wazuh Agent Collection

Once Windows generates an event, the Wazuh agent must collect it.

Conceptually:

```text
Windows Event Log
       │
       ▼
    Wazuh Agent
       │
       ▼
    Wazuh Server
```

An active agent provides the transport between the endpoint and the SIEM.

---

# Agent Status Is Only Step One

Seeing:

```text
Agent: Active
```

confirms communication with the Wazuh server.

It does not automatically prove:

```text
Every Required Event Log Is Being Collected
```

Therefore, event-level verification is still necessary.

---

# Searching by Agent

One of the most useful investigation filters is:

```text
agent.name
```

Examples:

```text
agent.name: DC1
```

and:

```text
agent.name: Client01
```

This isolates events from a specific endpoint.

---

# Searching for Process Activity

Once the correct endpoint is selected, searches can focus on process activity.

Examples of useful process names include:

```text
powershell.exe
cmd.exe
net.exe
```

These searches help confirm that process telemetry is available before writing detection logic.

---

# Why Search Before Writing Rules?

A detection rule should be based on the fields Wazuh actually receives.

The workflow should be:

```text
Generate Event
      │
      ▼
Find Event
      │
      ▼
Inspect Fields
      │
      ▼
Choose Detection Fields
      │
      ▼
Write Rule
```

Writing the rule first can lead to incorrect assumptions about field names or event structure.

---

# Event Provider Matters

Two events describing similar activity may come from different Windows providers.

For example:

```text
Windows Security
```

versus:

```text
Microsoft-Windows-Sysmon
```

The field structure may differ.

A detection written for one provider may therefore not behave correctly with another provider.

---

# Process Telemetry Troubleshooting

During the project, expected process telemetry did not always appear exactly where initially expected.

This required checking:

```text
Windows Security Event 4688
```

and:

```text
Sysmon Event ID 1
```

rather than assuming only one source would contain the required process activity.

This helped separate:

```text
Missing Detection
```

from:

```text
Missing Telemetry
```

---

# Missing Event Troubleshooting

If an event is missing from Wazuh:

```text
Event Missing
     │
     ▼
Check Windows
     │
     ├── Missing Locally
     │       │
     │       ▼
     │  Audit / Sysmon Problem
     │
     └── Present Locally
             │
             ▼
        Check Agent
             │
             ▼
       Check Collection
             │
             ▼
        Search Wazuh
```

This narrows the problem systematically.

---

# Missing Alert vs Missing Event

These are two different problems.

## Missing Event

```text
Expected activity cannot be found in Wazuh
```

Possible causes include:

- Windows did not generate the event
- Audit policy not enabled
- Sysmon not generating the event
- Wazuh agent not collecting the log
- Agent connectivity problem

## Missing Alert

```text
Event exists in Wazuh but custom alert does not appear
```

Possible causes include:

- Wrong rule field
- Incorrect match condition
- Rule hierarchy issue
- Conflicting rule
- Rule syntax problem

Understanding the difference prevents unnecessary troubleshooting.

---

# Log Collection vs Detection

The pipeline contains two distinct stages.

```text
COLLECTION
Windows → Agent → Wazuh
```

and:

```text
DETECTION
Wazuh Event → Rule → Alert
```

A problem in collection must be fixed before detection logic can work correctly.

---

# Example Detection Pipeline

For a Windows command:

```text
Command Executed
      │
      ▼
Process Created
      │
      ▼
4688 or Sysmon Event 1
      │
      ▼
Wazuh Agent
      │
      ▼
Wazuh Event
      │
      ▼
Custom Rule
      │
      ▼
Alert
```

Each stage can be individually tested.

---

# Event Field Analysis

Once an event reaches Wazuh, the parsed fields should be examined.

Useful information can include:

```text
Agent Name
Event ID
Provider
Computer
User
Process Name
Command Line
Parent Process
Timestamp
```

The exact fields available depend on the event source and Wazuh parsing.

Detection logic should use fields confirmed from real events.

---

# Why Raw Event Review Matters

A dashboard view may simplify event information.

During detection development, reviewing the actual event structure helps answer:

```text
Which field contains the executable?

Which field contains the command line?

What provider generated the event?

What event ID is present?
```

This prevents rules from being based on assumptions.

---

# Wazuh Search Workflow

A practical search workflow is:

```text
1. Filter by agent.name
2. Narrow by event ID
3. Search for process name
4. Inspect individual event
5. Identify useful fields
6. Build or troubleshoot rule
```

This reduces the amount of unrelated telemetry during analysis.

---

# Example: PowerShell Investigation

Suppose PowerShell activity is being tested on Client01.

The workflow becomes:

```text
Execute PowerShell
      │
      ▼
Verify Windows Process Event
      │
      ▼
Search:
agent.name: Client01
      │
      ▼
Find powershell.exe
      │
      ▼
Inspect Command Line
      │
      ▼
Determine Detection Logic
```

This creates an evidence-based detection workflow.

---

# Example: `net.exe` Investigation

The same methodology can be used for:

```text
net.exe
```

For example:

```text
Execute Test Command
       │
       ▼
Check 4688 / Sysmon
       │
       ▼
Search Wazuh
       │
       ▼
Inspect Event Fields
       │
       ▼
Test Custom Rule
```

This approach was used during the development of custom Windows command detections.

---

# Log Collection and Rule Development

The custom detection rules depend on reliable log collection.

The relationship is:

```text
Windows Auditing
       │
       ▼
Log Collection
       │
       ▼
Field Parsing
       │
       ▼
Rule Matching
       │
       ▼
Alert
```

If telemetry changes, detection logic may also need to be updated.

---

# Wazuh Rule Troubleshooting

When a rule fails, the first question should be:

```text
Can I find the event in Wazuh?
```

If the answer is:

```text
No
```

the problem is probably earlier in the collection pipeline.

If:

```text
Yes
```

then rule logic can be investigated.

---

# Rule Conflict Example

During custom-rule development, an earlier rule:

```text
100100
```

interfered with the expected behavior of later rules.

This demonstrated that:

```text
Event Collection Working
```

does not automatically mean:

```text
Correct Detection Rule Matching
```

Rule relationships still need to be evaluated separately.

---

# Working Custom Rules

After troubleshooting, custom rules including:

```text
100103
100104
```

successfully matched the intended Windows activity.

These detections depended on the Windows telemetry pipeline described on this page.

---

# Security Group Telemetry

Group-management events can also be investigated by filtering DC1 activity.

Conceptually:

```text
agent.name: DC1
       │
       ▼
Security Group Event
       │
       ▼
Inspect User / Group Change
```

This provides identity-security visibility directly from the domain controller.

---

# Monitoring Critical Systems

Not every event has the same security significance.

For example:

```text
Process Event on Test Workstation
```

may have limited impact.

While:

```text
Privileged Group Modification on DC1
```

may represent a much more serious security event.

Asset context therefore matters when interpreting logs.

---

# Data Quality

Detection quality depends on log quality.

Poor telemetry can include:

```text
Missing Command Line
Missing Event IDs
Incorrect Agent
Wrong Event Source
Incomplete Audit Policy
```

Better telemetry produces more useful detections.

Conceptually:

```text
Better Logging
     │
     ▼
Better Context
     │
     ▼
Better Detection
```

---

# Logging Volume

Increasing endpoint auditing also increases event volume.

For example:

```text
More Process Events
       │
       ▼
More Wazuh Events
       │
       ▼
More Indexing
       │
       ▼
More Storage / Processing
```

This is important because the Wazuh server shares physical resources with other lab workloads through Proxmox.

Logging should therefore provide useful security value rather than generating unnecessary noise.

---

# Resource Considerations

Wazuh must process and index incoming telemetry.

As additional endpoints and log sources are added:

```text
More Agents
     +
More Events
     =
More SIEM Resource Usage
```

This connects log-collection planning directly to Proxmox resource management.

---

# Time Synchronization

Accurate timestamps are important for security investigations.

Events from different systems need to be compared chronologically.

Conceptually:

```text
Client Event
12:01:05
     │
     ▼
DC Event
12:01:07
     │
     ▼
Wazuh Alert
12:01:08
```

Incorrect system time can make an investigation difficult.

Maintaining consistent time across infrastructure systems is therefore an important operational practice.

---

# Security of Collected Logs

Centralized logs may contain sensitive information.

Examples include:

- Usernames
- Hostnames
- Command lines
- Internal addresses
- Security-group information
- Administrative activity

Access to the Wazuh dashboard should therefore be restricted to authorized users.

---

# Command-Line Data Sensitivity

Detailed command-line auditing can occasionally capture sensitive data if secrets are typed directly into commands.

Administrators should avoid placing:

```text
Passwords
API Keys
Tokens
Private Secrets
```

directly into command-line arguments when possible.

This reduces the likelihood that secrets are stored in endpoint or SIEM logs.

---

# GitHub Documentation Safety

Raw event exports and screenshots should be reviewed before being published publicly.

Do not commit:

```text
Passwords
Tokens
Private Keys
VPN Credentials
Sensitive Authentication Data
```

The portfolio should demonstrate monitoring architecture without exposing secrets.

---

# Troubleshooting Workflow

The complete troubleshooting workflow for missing telemetry is:

```text
Expected Activity
      │
      ▼
Was Activity Executed?
      │
      ▼
Did Windows Generate Event?
      │
      ▼
Is Audit Policy Correct?
      │
      ▼
Is Wazuh Agent Running?
      │
      ▼
Can Agent Reach 10.0.0.7?
      │
      ▼
Is Event Collected?
      │
      ▼
Can Event Be Found in Wazuh?
      │
      ▼
Are Expected Fields Present?
      │
      ▼
Can Detection Rule Match?
```

This method significantly reduces guesswork.

---

# Current Collection State

The current environment successfully collects Windows security telemetry from:

```text
DC1
Client01
```

Available telemetry includes:

```text
Windows Security Events
Process Creation Events
Account Management Events
Security Group Events
Sysmon Process Events
```

This telemetry is searchable in Wazuh and is being used for custom detection development.

---

# Future Log Sources

Future improvements may include collecting additional telemetry such as:

- PowerShell Script Block Logging
- PowerShell Module Logging
- Additional Sysmon events
- Authentication activity
- Failed logons
- Privileged logons
- Scheduled task creation
- Service creation
- Registry changes
- Windows Firewall events
- Linux authentication logs
- pfSense firewall logs

Each new source should be added because it supports a specific monitoring or detection goal.

---

# Future Detection Pipeline

As logging expands, the monitoring architecture can evolve toward:

```text
Windows
Linux
pfSense
Active Directory
Security Tools
      │
      ▼
Central Wazuh SIEM
      │
      ▼
Normalized Telemetry
      │
      ▼
Detection Rules
      │
      ▼
MITRE ATT&CK Mapping
      │
      ▼
Security Investigation
```

This will make Wazuh the central visibility platform for the entire home lab.

---

# Skills Demonstrated

This portion of the project demonstrates practical experience with:

## Windows Logging

- Windows Security Event Log
- Advanced Audit Policy
- Event ID 4688
- Event ID 4720
- Security group events
- Command-line auditing

## Sysmon

- Sysmon Event ID 1
- Process monitoring
- Event channel validation
- Additional endpoint telemetry

## Wazuh

- Windows event collection
- Agent-based log forwarding
- Event searching
- `agent.name` filtering
- Event field analysis

## Troubleshooting

- Source-event validation
- Audit-policy validation
- Agent troubleshooting
- Missing-event troubleshooting
- Missing-alert troubleshooting
- Multi-source telemetry comparison

## Detection Engineering

- Telemetry analysis
- Field identification
- Evidence-based rule development
- Detection pipeline validation

---

# What I Learned

Building the log-collection pipeline reinforced that a SIEM is only one part of a monitoring system.

The full chain is:

```text
Activity
   │
   ▼
Logging Configuration
   │
   ▼
Endpoint Event
   │
   ▼
Agent Collection
   │
   ▼
SIEM Ingestion
   │
   ▼
Parsing
   │
   ▼
Detection
```

The most important lesson was:

> **If an event is missing, start at the endpoint and work forward through the pipeline.**

Another major lesson was:

> **Never build a detection around what you assume the event contains. Generate the activity, inspect the real telemetry, and build the detection from the fields that actually exist.**

This approach became the foundation for the custom Wazuh detection rules developed later in the project.

---

# Log Collection Summary

```text
                    Windows Environment
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
             DC1                    Client01
              │                         │
     ┌────────┴────────┐                │
     │                 │                │
     ▼                 ▼                ▼
Security Logs        Sysmon       Security Logs
     │                 │                │
     └────────┬────────┴────────┬───────┘
              │                 │
              ▼                 ▼
         Wazuh Agents      Endpoint Events
              │
              ▼
            wazuh01
            10.0.0.7
              │
              ▼
         Wazuh Manager
              │
              ▼
        Parsed Telemetry
              │
              ▼
       Detection Rules
              │
              ▼
       Security Alerts
```

The log-collection layer provides the telemetry foundation required for effective custom detection engineering in the home lab.

---

# Related Documentation

[← Wazuh Windows Agents](windows-agents.md)

[← Wazuh SIEM Deployment](wazuh-deployment.md)

[← Windows Security Auditing and Sysmon](../04-active-directory/windows-security-auditing.md)

[← Active Directory Group Policy](../04-active-directory/group-policy.md)

[← Main Project README](../README.md)
