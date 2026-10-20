# Wazuh Custom Detection Rules

## Overview

After Windows telemetry was successfully reaching Wazuh, the next phase of the home lab focused on **custom detection engineering**.

Instead of relying only on Wazuh's built-in rules, custom rules were created to detect specific Windows command-line activity generated during controlled testing.

The custom Windows detection rules are stored in:

```text
local_windows_cmd_rules.xml
```

The project included:

- Identifying useful Windows telemetry
- Inspecting parsed Wazuh fields
- Writing custom detection logic
- Testing Windows command execution
- Troubleshooting Wazuh Manager restart failures
- Resolving conflicting rules
- Validating alerts in the Wazuh Dashboard
- Iteratively tuning detections

Working custom rules included:

```text
100103
100104
```

The project provided practical experience with the full detection-engineering lifecycle rather than simply enabling prebuilt SIEM alerts.

---

# Detection Engineering Architecture

The detection pipeline is:

```text
Windows Activity
      │
      ▼
Windows Event / Sysmon
      │
      ▼
Wazuh Agent
      │
      ▼
Wazuh Manager
      │
      ▼
Decoder / Parsed Fields
      │
      ▼
Custom Detection Rule
      │
      ▼
Security Alert
      │
      ▼
Wazuh Dashboard
```

Every stage must work correctly before a custom detection can succeed.

---

# Why Build Custom Rules?

Built-in SIEM rules provide broad security coverage, but custom detections allow monitoring to be tailored to the environment.

For example, the lab can specifically monitor activity involving Windows administrative utilities such as:

```text
net.exe
```

and command-line patterns associated with account-management behavior.

This allows detections to be based on:

```text
What activity matters in this environment?
```

rather than relying entirely on generic vendor rules.

---

# Detection Development Philosophy

The most important principle used during this project was:

> **Build detections from real telemetry rather than assumed telemetry.**

The workflow is:

```text
Generate Activity
      │
      ▼
Confirm Local Event
      │
      ▼
Find Event in Wazuh
      │
      ▼
Inspect Parsed Fields
      │
      ▼
Write Rule
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

This prevents rules from being written against fields that do not actually exist in the incoming event.

---

# Telemetry Sources

The Windows detection work uses telemetry including:

```text
Windows Security Event ID 4688
```

for process creation and:

```text
Sysmon Event ID 1
```

for process creation telemetry from Sysmon.

Additional identity-related Windows events are also available for:

- Account creation
- Security-group membership changes
- Administrative actions

---

# Process Creation Detection

A typical process detection begins when a user executes a command.

```text
User Executes Command
        │
        ▼
Process Created
        │
        ▼
Windows Logs Event
        │
        ▼
Wazuh Agent Collects Event
        │
        ▼
Wazuh Parses Event
        │
        ▼
Custom Rule Evaluates Fields
```

The rule can then determine whether the process or command line represents activity worth alerting on.

---

# Why Command-Line Context Matters

Consider the process:

```text
net.exe
```

The executable itself provides only limited information.

The command line provides significantly more context.

For example, `net.exe` can be used for:

```text
User Management
Group Management
Network Administration
Resource Enumeration
```

Detection logic therefore becomes more useful when both:

```text
Process Name
```

and:

```text
Command Line
```

are available.

---

# Account-Management Command Testing

One of the custom detection workflows focused on Windows account-management activity involving the:

```text
net user
```

command.

Conceptually:

```text
net.exe
   │
   ▼
Account-Management Command
   │
   ▼
Windows Process Event
   │
   ▼
Wazuh
   │
   ▼
Custom Rule
   │
   ▼
Alert
```

This provided a controlled way to validate process and command-line detections.

---

# Why Account-Creation Commands Matter

Creating an account can be legitimate administrative activity.

However, account creation is also a common persistence technique.

An attacker who gains sufficient access may attempt to create another account to maintain access to a system.

Conceptually:

```text
Compromise
    │
    ▼
Create Account
    │
    ▼
Establish Persistence
```

This makes account-management activity useful from a detection perspective.

---

# Windows Event ID 4720

Windows can also record account creation directly using:

```text
Event ID 4720
```

which represents:

```text
A user account was created
```

This creates an opportunity for multiple detection approaches.

```text
Command-Based Detection
        │
        └── Process / Command Line


Identity-Based Detection
        │
        └── Event ID 4720
```

Using multiple telemetry sources can strengthen detection coverage.

---

# Custom Rule File

The custom Windows command rules are maintained in:

```text
local_windows_cmd_rules.xml
```

Keeping custom detections in a dedicated file helps separate locally developed rules from Wazuh's default rules.

Conceptually:

```text
Wazuh Rules
│
├── Built-In Rules
│
└── Local Custom Rules
     │
     └── local_windows_cmd_rules.xml
```

This makes custom logic easier to maintain and troubleshoot.

---

# Rule IDs

Custom Wazuh rules use unique rule IDs.

Working rules created during this project included:

```text
100103
100104
```

These were part of the Windows command-detection workflow.

A previous rule:

```text
100100
```

also became important during troubleshooting because its behavior interfered with later rule matching.

---

# Why Unique Rule IDs Matter

Each custom rule needs an identifier that does not conflict with another rule.

Conceptually:

```text
Rule 100103
      │
      └── Detection A


Rule 100104
      │
      └── Detection B
```

Rule IDs also make investigation easier because alerts can be searched and referenced by a consistent identifier.

---

# Detection Rule Lifecycle

The custom-rule lifecycle used in the lab was:

```text
Identify Behavior
      │
      ▼
Generate Event
      │
      ▼
Inspect Event
      │
      ▼
Create Rule
      │
      ▼
Validate Configuration
      │
      ▼
Reload / Restart Wazuh
      │
      ▼
Repeat Test
      │
      ▼
Review Alert
      │
      ▼
Tune
```

This process was repeated until the expected behavior was detected reliably.

---

# Step 1 — Generate Known Activity

The first step is generating controlled activity on a monitored Windows endpoint.

For example:

```text
Client01
   │
   ▼
Execute Known Test Command
```

Using known activity makes it possible to determine exactly what the SIEM should observe.

---

# Step 2 — Verify the Windows Event

Before editing Wazuh rules, the corresponding Windows telemetry should be confirmed locally.

For process creation:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4688
} -MaxEvents 10
```

For Sysmon:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-Sysmon/Operational'
    Id=1
} -MaxEvents 10
```

If the activity is not logged locally, a Wazuh rule cannot detect it.

---

# Step 3 — Find the Event in Wazuh

Once the event is confirmed locally, the next step is verifying that it reaches the SIEM.

Useful filters include:

```text
agent.name: Client01
```

or:

```text
agent.name: DC1
```

The event can then be narrowed by:

- Event ID
- Process name
- Provider
- Command line
- Timestamp

---

# Step 4 — Inspect Parsed Fields

Before creating a rule, the actual Wazuh event fields should be reviewed.

Useful fields can include information describing:

```text
Event ID
Process Name
Command Line
User
Computer
Event Provider
Parent Process
```

The exact field names depend on how Wazuh parses the source event.

This is why the real event should be inspected before writing the rule.

---

# Step 5 — Write the Detection

Once the useful fields are known, a custom rule can evaluate them.

Conceptually:

```text
IF
Process = net.exe

AND
Command Line Contains Expected Account Activity

THEN
Generate Alert
```

The purpose is to describe the suspicious or security-relevant behavior precisely enough to produce useful alerts.

---

# Detection Specificity

A rule that simply detects:

```text
net.exe
```

may be too broad.

The program can be used for legitimate administration.

A more useful detection can consider:

```text
Process
+
Command-Line Behavior
```

to create greater context.

---

# Broad vs Specific Detection

An overly broad detection behaves like:

```text
net.exe
   │
   ▼
ALERT
```

This can create alerts for normal administrative activity.

A more specific detection may behave conceptually like:

```text
net.exe
   │
   ▼
Relevant Command Pattern?
   │
 ┌─┴─┐
 │   │
No  Yes
 │   │
 ▼   ▼
No  Alert
Alert
```

This reduces unnecessary noise.

---

# Rule Testing

After writing a rule, the same controlled activity is executed again.

```text
Test Command
     │
     ▼
Windows Event
     │
     ▼
Wazuh
     │
     ▼
Custom Rule
     │
     ▼
Expected Alert?
```

If the alert does not appear, the detection pipeline is investigated layer by layer.

---

# Wazuh Manager Restart Failure

During custom-rule development, one version of the local rule configuration caused the Wazuh Manager restart to fail.

This was an important troubleshooting experience.

Conceptually:

```text
Edit Rule File
      │
      ▼
Restart Wazuh Manager
      │
      ▼
Service Fails
```

The configuration had to be corrected before the manager could return to a healthy state.

---

# Why This Matters

A malformed SIEM configuration can affect more than one individual detection.

If the Wazuh Manager cannot operate correctly:

```text
Agents
   │
   ▼
Wazuh Manager
   X
```

central security-event processing can be impacted.

Custom detection changes therefore need to be treated as controlled configuration changes.

---

# Safe Rule-Change Workflow

A safer workflow is:

```text
Back Up Working Rule File
        │
        ▼
Make One Small Change
        │
        ▼
Validate Configuration
        │
        ▼
Restart / Reload
        │
        ▼
Check Wazuh Manager
        │
        ▼
Generate Test Event
```

This makes it easier to identify which change introduced a problem.

---

# Service Validation

After changing custom rules, Wazuh Manager health should be verified.

For example:

```bash
systemctl status wazuh-manager
```

The service should return to an active state before continuing detection tests.

If it fails, configuration and logs should be reviewed before adding more changes.

---

# Rule Conflict Troubleshooting

Another challenge involved an earlier rule:

```text
100100
```

matching activity in a way that prevented the expected later rule behavior.

The expected detection was therefore not simply a question of:

```text
Does my rule syntax work?
```

It also required understanding:

```text
Which rule is already matching this event?
```

---

# Why Rule Conflicts Matter

Detection rules can have relationships and dependencies.

Conceptually:

```text
Incoming Event
      │
      ▼
Rule 100100
      │
      ▼
Match
      │
      ▼
Different Outcome Than Expected
```

while the intended workflow might have been:

```text
Incoming Event
      │
      ▼
Expected Parent / Matching Logic
      │
      ▼
Rule 100103 / 100104
      │
      ▼
Expected Alert
```

Understanding existing rule behavior became necessary to resolve the conflict.

---

# First Match Is Not Always the Final Goal

An event producing an alert does not automatically mean it matched the most useful rule.

For example:

```text
Event
  │
  ▼
Generic Rule
  │
  ▼
Generic Alert
```

may technically work.

However, the intended result may be:

```text
Event
  │
  ▼
Specific Custom Detection
  │
  ▼
More Useful Alert
```

Detection testing therefore includes verifying the actual rule ID that triggered.

---

# Working Rules

After troubleshooting the rule logic and conflicts, the custom detection workflow successfully used:

```text
Rule 100103
Rule 100104
```

The rules produced the expected results during controlled testing.

This confirmed that:

```text
Windows Telemetry
      │
      ▼
Wazuh Ingestion
      │
      ▼
Custom Rule Logic
      │
      ▼
Expected Alert
```

was functioning end to end.

---

# Detection Validation

A rule is not considered complete simply because Wazuh accepts the XML configuration.

The actual success criterion is:

```text
Expected Activity
       │
       ▼
Expected Rule
       │
       ▼
Expected Alert
```

All three must match the intended design.

---

# Testing by Rule ID

When validating a custom detection, the alert should be checked for the expected:

```text
rule.id
```

This confirms that the specific custom detection fired rather than an unrelated built-in or generic rule.

For example:

```text
rule.id: 100103
```

or:

```text
rule.id: 100104
```

can be used when reviewing the resulting alerts.

---

# Agent Filtering During Testing

Tests can also be restricted to the system where the activity was generated.

For example:

```text
agent.name: Client01
```

combined with the custom rule ID provides a much narrower investigation scope.

Conceptually:

```text
Client01
+
Rule 100103
=
Specific Test Result
```

---

# Detection Troubleshooting Model

If the custom alert does not appear:

```text
No Alert
   │
   ▼
Was Activity Generated?
   │
   ▼
Was Local Event Generated?
   │
   ▼
Did Wazuh Receive Event?
   │
   ▼
Are Expected Fields Present?
   │
   ▼
Did Another Rule Match?
   │
   ▼
Is Custom Rule Valid?
   │
   ▼
Does Condition Match Real Data?
```

This prevents randomly rewriting detection logic without identifying the actual problem.

---

# Missing Telemetry vs Bad Rule

These are two completely different problems.

## Missing Telemetry

```text
Event cannot be found in Wazuh
```

Possible causes:

- Windows auditing
- Sysmon
- Wazuh agent
- Log collection
- Connectivity

## Bad Detection Logic

```text
Event exists in Wazuh
but custom alert does not
```

Possible causes:

- Wrong field
- Wrong value
- Rule conflict
- Incorrect dependency
- Rule syntax
- Overly specific match condition

The first step is always determining which type of failure is occurring.

---

# Rule Tuning

Detection engineering does not stop when a rule fires once.

The rule should also be evaluated for:

```text
False Positives
False Negatives
Unnecessary Noise
Missing Variations
```

A detection that alerts on too much activity may become difficult to use operationally.

---

# False Positives

Suppose a detection triggers on every use of:

```text
net.exe
```

Normal administration could generate many unnecessary alerts.

This creates:

```text
Too Many Alerts
      │
      ▼
Analyst Noise
      │
      ▼
Important Alert Easier to Miss
```

This is known as alert fatigue.

---

# False Negatives

The opposite problem occurs when the detection is too narrow.

For example:

```text
Only One Exact Command Format
```

may miss another command that performs the same action.

Detection logic must balance specificity with sufficient coverage.

---

# Detection Tuning Goal

The objective is:

```text
Interesting Security Behavior
             │
             ▼
          Alert
```

while:

```text
Normal Activity
             │
             ▼
      Minimal Noise
```

This requires repeated testing.

---

# Detection as a Hypothesis

A useful way to think about detection engineering is:

```text
Hypothesis:
If this behavior occurs,
the available telemetry should
allow me to identify it.
```

The detection test then determines whether that hypothesis is true.

---

# Detection Development Example

A simplified account-management detection workflow is:

```text
Behavior:
Account Management Command
        │
        ▼
Expected Telemetry:
Process / Command Line
        │
        ▼
Data Source:
4688 / Sysmon
        │
        ▼
Detection:
Custom Wazuh Rule
        │
        ▼
Validation:
Controlled Test
```

This provides a repeatable way to design additional detections.

---

# Multiple Detection Opportunities

The same security behavior can often be detected in multiple ways.

For example, account creation could generate:

```text
Process Telemetry
       │
       └── Command used to create account
```

and:

```text
Windows Event 4720
       │
       └── Account actually created
```

These detections answer slightly different questions.

---

# Behavioral vs Outcome Detection

A command-based detection can identify:

```text
Attempted Behavior
```

while an account-management event can identify:

```text
Resulting Change
```

Conceptually:

```text
Command Executed
      │
      ▼
Attempt Detection
```

and:

```text
Account Created
      │
      ▼
Outcome Detection
```

Using both approaches can increase confidence.

---

# Security Group Detection Opportunities

The same approach can be extended to privileged-group changes.

For example:

```text
Group Management Command
        │
        ▼
Process Detection
```

combined with:

```text
Event 4728 / 4732 / 4756
        │
        ▼
Membership Change Detection
```

This creates multiple sources of evidence for security-sensitive actions.

---

# Defense-in-Depth for Detection

Detection itself can also use defense-in-depth.

```text
Command Telemetry
       │
       +
Windows Identity Event
       │
       +
Sysmon
       │
       ▼
Correlated Investigation
```

No single telemetry source has to provide every detail.

---

# Documentation of Detections

Each mature custom detection should eventually document:

```text
Rule ID
Rule Name
Purpose
Data Source
Event ID
Relevant Fields
Test Procedure
Expected Alert
Known False Positives
MITRE ATT&CK Mapping
Status
```

This makes the detection easier to maintain and explain.

---

# Detection Documentation Template

A future standardized format could look like:

```text
Rule ID:
100xxx

Detection:
Description of behavior

Data Source:
Windows Security / Sysmon

Test:
Controlled command

Expected Result:
Wazuh alert

Status:
Validated
```

This can be expanded as additional detections are built.

---

# MITRE ATT&CK Mapping

A future improvement is mapping custom detections to relevant MITRE ATT&CK techniques.

This provides a standard way to describe what attacker behavior a rule is intended to identify.

The process would be:

```text
Custom Detection
      │
      ▼
Identify Behavior
      │
      ▼
Map to ATT&CK Technique
      │
      ▼
Document Detection Coverage
```

Mappings should be based on the actual behavior detected rather than added only for appearance.

---

# Detection Severity

Custom rules can also be assigned alert severity based on the importance of the behavior.

Conceptually:

```text
Low-Risk Administrative Activity
          │
          ▼
Lower Alert Severity


High-Risk Privilege Change
          │
          ▼
Higher Alert Severity
```

Severity should reflect operational importance rather than simply making every custom rule critical.

---

# Production vs Lab Detections

The detections in this project are built in a controlled home-lab environment.

A rule that works well in the lab may require additional tuning before being used in a production enterprise environment.

Production environments typically have:

- More users
- More endpoints
- More administrative activity
- More automation
- More legitimate use of command-line tools

This can significantly change false-positive behavior.

---

# Safe Testing

Detection tests are performed in the controlled lab environment.

The objective is to generate predictable telemetry without disrupting production systems.

This allows:

```text
Known Action
      │
      ▼
Known Expected Event
      │
      ▼
Known Expected Alert
```

which makes troubleshooting easier.

---

# Change Management

Custom detection changes can affect SIEM availability.

A safe rule-management workflow is:

```text
Working Rule File
      │
      ▼
Backup
      │
      ▼
Edit Small Section
      │
      ▼
Validate
      │
      ▼
Restart / Reload Manager
      │
      ▼
Check Service
      │
      ▼
Run Test
      │
      ▼
Verify Alert
```

This methodology became especially important after experiencing a Wazuh Manager restart failure.

---

# Snapshots

Proxmox snapshots were taken around major Wazuh detection milestones.

These provided recovery points before or after:

```text
Agent Deployment
Audit Policy Changes
Custom Rule Changes
Sysmon Installation
```

This allows experimentation while maintaining a known-good environment.

---

# Current Detection State

The custom Windows detection environment currently has:

```text
Windows Telemetry
      │
      ▼
Wazuh Collection
      │
      ▼
local_windows_cmd_rules.xml
      │
      ▼
Rules 100103 / 100104
      │
      ▼
Successful Test Alerts
```

The earlier rule-conflict and manager-restart issues were resolved, and the intended custom alerts successfully matched during testing.

---

# Future Detection Ideas

Future rules may focus on behaviors such as:

- PowerShell execution
- Suspicious PowerShell command lines
- New user-account creation
- Privileged-group additions
- Privileged-group removals
- Scheduled task creation
- New Windows service creation
- Remote administration
- Failed logon patterns
- Administrative tool usage
- Security-log tampering
- Sysmon process relationships

Each detection should begin with identifying a meaningful behavior and confirming that appropriate telemetry exists.

---

# Detection Engineering Roadmap

The future workflow is:

```text
Behavior
   │
   ▼
Telemetry
   │
   ▼
Detection
   │
   ▼
Testing
   │
   ▼
Tuning
   │
   ▼
MITRE Mapping
   │
   ▼
Documentation
```

This provides a consistent framework for expanding the SIEM.

---

# Skills Demonstrated

This project demonstrates practical experience with:

## Detection Engineering

- Behavioral detection design
- Custom SIEM rules
- Event-field analysis
- Detection testing
- Detection tuning
- False-positive considerations

## Wazuh

- Custom XML rules
- Local rule files
- Rule IDs
- Parent/child rule behavior
- Rule conflicts
- Manager configuration troubleshooting

## Windows Security

- Event ID 4688
- Event ID 4720
- Security group auditing
- Sysmon Event ID 1
- Command-line telemetry

## Troubleshooting

- Missing event analysis
- Missing alert analysis
- Rule conflict resolution
- Service restart failures
- Field validation
- End-to-end pipeline testing

## Security Operations

- Alert validation
- Detection documentation
- Event investigation
- Endpoint context
- SIEM change management

---

# What I Learned

Building custom Wazuh rules was one of the most important parts of the SIEM project.

It demonstrated that detection engineering is not simply:

```text
Write Rule
   │
   ▼
Get Alert
```

The real process is:

```text
Understand Behavior
       │
       ▼
Understand Telemetry
       │
       ▼
Inspect Real Data
       │
       ▼
Write Detection
       │
       ▼
Troubleshoot
       │
       ▼
Test
       │
       ▼
Tune
       │
       ▼
Validate
```

One of the biggest lessons was:

> **If the event exists but the expected alert does not, investigate the rule logic and rule relationships rather than the endpoint.**

Another major lesson was:

> **A working detection is not just one that fires. It should fire for the behavior you intended, using the rule you intended, while minimizing unnecessary alerts.**

The custom-rule project transformed the Wazuh deployment from a log-collection platform into a practical detection-engineering environment.

---

# Custom Detection Summary

```text
                    Windows Activity
                           │
                           ▼
                 Security / Sysmon Event
                           │
                           ▼
                      Wazuh Agent
                           │
                           ▼
                      Wazuh Manager
                           │
                           ▼
                 Parsed Event Fields
                           │
                           ▼
             local_windows_cmd_rules.xml
                           │
                  ┌────────┴────────┐
                  │                 │
                  ▼                 ▼
             Rule 100103       Rule 100104
                  │                 │
                  └────────┬────────┘
                           ▼
                     Custom Alert
                           │
                           ▼
                    Wazuh Dashboard
```

The custom detection rules demonstrate an end-to-end workflow from endpoint activity to centralized SIEM alerting.

---

# Related Documentation

[← Wazuh Log Collection and Telemetry Pipeline](log-collection.md)

[← Wazuh Windows Agents](windows-agents.md)

[← Wazuh SIEM Deployment](wazuh-deployment.md)

[← Windows Security Auditing and Sysmon](../04-active-directory/windows-security-auditing.md)

[← Main Project README](../README.md)
