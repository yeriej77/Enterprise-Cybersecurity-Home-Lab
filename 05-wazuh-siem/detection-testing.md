# Wazuh Detection Testing and Validation

## Overview

Custom detections are only useful if they can be tested and shown to work reliably.

After deploying Wazuh, configuring Windows auditing, installing agents, and creating custom rules, the next phase of the project focused on **controlled detection testing**.

The testing methodology follows a complete end-to-end path:

```text
Generate Known Activity
        │
        ▼
Verify Windows Telemetry
        │
        ▼
Verify Wazuh Ingestion
        │
        ▼
Verify Parsed Fields
        │
        ▼
Verify Custom Rule Match
        │
        ▼
Verify Alert
        │
        ▼
Document Result
```

This approach validates the entire detection pipeline rather than assuming that a rule works simply because Wazuh accepts its configuration.

---

# Testing Environment

The primary systems used for detection testing are:

| System | Role | IP Address |
|---|---|---:|
| DC1 | Windows Server 2025 Domain Controller | `10.0.0.6` |
| Client01 | Domain-joined Windows workstation | DHCP |
| Wazuh | SIEM Server | `10.0.0.7` |

The monitored Windows endpoints use:

```text
Wazuh Agent 4.14.6
```

and send security telemetry to:

```text
wazuh01
10.0.0.7
```

---

# Detection Test Philosophy

Every detection test should answer four questions:

```text
1. Did the activity occur?

2. Did Windows log it?

3. Did Wazuh receive it?

4. Did the correct rule alert on it?
```

If any one of these stages fails, the test should not be considered successful.

---

# Validation Layers

A detection can fail at several different points.

```text
Test Activity
     │
     ▼
Windows Logging
     │
     ▼
Wazuh Agent
     │
     ▼
Wazuh Manager
     │
     ▼
Parsing
     │
     ▼
Custom Rule
     │
     ▼
Dashboard Alert
```

The purpose of testing is to identify exactly which layer is working or failing.

---

# Test Preparation

Before generating security events, the environment should be verified.

## Confirm the Wazuh Agent

On the Windows endpoint, confirm that the Wazuh service is running.

Example:

```powershell
Get-Service | Where-Object {$_.Name -like "*Wazuh*"}
```

The service should be in a running state.

---

# Confirm Connectivity

The endpoint should be able to reach:

```text
10.0.0.7
```

Basic testing can include:

```powershell
ping 10.0.0.7
```

This does not validate the complete Wazuh application path, but it confirms basic IP connectivity.

---

# Confirm Agent Status

Inside Wazuh, verify that the endpoint appears as:

```text
Active
```

or:

```text
Online
```

for the system being tested.

Examples include:

```text
DC1
Client01
```

---

# Confirm Windows Auditing

Process creation auditing should be verified before running process-based tests.

```powershell
auditpol /get /subcategory:"Process Creation"
```

The output should indicate that auditing is enabled for the expected event type.

---

# Test 1 — Basic Process Creation

The first test verifies that normal Windows process execution produces telemetry.

A simple command can be executed:

```powershell
whoami
```

The expected process is:

```text
whoami.exe
```

---

# Expected Result

The expected telemetry path is:

```text
whoami.exe
     │
     ▼
Windows Process Creation
     │
     ▼
Event ID 4688
and/or
Sysmon Event ID 1
     │
     ▼
Wazuh
```

This test does not necessarily need to generate a high-priority security alert.

Its purpose is to verify the underlying process-telemetry pipeline.

---

# Verify Windows Security Event 4688

On the endpoint:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4688
} -MaxEvents 10
```

The recent process activity should be reviewed.

---

# Verify Sysmon Event ID 1

If Sysmon is installed on the endpoint:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-Sysmon/Operational'
    Id=1
} -MaxEvents 10
```

This provides another way to confirm process execution telemetry.

---

# Verify the Event in Wazuh

Search for the endpoint.

For example:

```text
agent.name: Client01
```

or:

```text
agent.name: DC1
```

Then narrow the results to the process or corresponding event.

The purpose is to confirm:

```text
Windows Generated Event
+
Wazuh Received Event
```

---

# Test 2 — Windows Command-Line Activity

The next test verifies that Wazuh can observe command-line utilities relevant to administrative activity.

An example executable is:

```text
net.exe
```

The test should be performed using a controlled command in the lab environment.

The important elements to validate are:

```text
Process Name
Command Line
User
Agent
Event Source
```

---

# Expected Command Telemetry

The expected path is:

```text
Controlled net.exe Command
          │
          ▼
Windows Process Event
          │
          ▼
Command-Line Information
          │
          ▼
Wazuh Agent
          │
          ▼
Wazuh Event
```

The event should then contain enough information for the custom detection logic to evaluate it.

---

# Test 3 — Controlled Account Creation

One of the primary security tests involves Windows account-management activity.

In an isolated test environment, a temporary account can be created using an administrative command.

Example format:

```powershell
net user <TestUser> <TestPassword> /add
```

Use a temporary lab-only account and do not place real passwords in screenshots, GitHub documentation, or SIEM examples.

---

# Expected Telemetry

Account creation can create more than one useful source of evidence.

```text
net user ... /add
       │
       ├───────────────┐
       │               │
       ▼               ▼
Process Event     Account Event
4688 / Sysmon        4720
       │               │
       └───────┬───────┘
               ▼
             Wazuh
```

This provides both:

```text
Command / Behavior Evidence
```

and:

```text
Account Creation Evidence
```

---

# Verify Event ID 4720

A Windows account-creation event can be checked locally with:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4720
} -MaxEvents 10
```

The event should correspond to the temporary test account.

---

# Remove the Test Account

After validation, remove temporary accounts that are no longer required.

Example format:

```powershell
net user <TestUser> /delete
```

Test artifacts should not remain in the environment unnecessarily.

---

# Custom Rule Validation

The custom rule file is:

```text
local_windows_cmd_rules.xml
```

Working custom rule IDs include:

```text
100103
100104
```

During testing, the objective is not simply to see any Wazuh alert.

The expected custom rule must be confirmed.

---

# Search by Rule ID

Custom alerts can be narrowed using:

```text
rule.id: 100103
```

or:

```text
rule.id: 100104
```

The exact rule expected for a particular test should be confirmed from the current:

```text
local_windows_cmd_rules.xml
```

configuration.

This prevents documentation from assuming a rule-to-behavior relationship that may later change during tuning.

---

# Agent and Rule Validation

A strong test query combines the endpoint and expected rule.

Conceptually:

```text
agent.name: Client01
AND
rule.id: 100103
```

or:

```text
agent.name: DC1
AND
rule.id: 100104
```

This provides stronger evidence that:

```text
The correct endpoint
```

generated:

```text
The correct custom detection.
```

---

# Test 4 — Security Group Changes

Security-group auditing provides another useful detection scenario.

Relevant Windows events include:

```text
4728
4729
4732
4733
4756
4757
```

These represent different additions and removals from security-enabled groups.

---

# Group Change Test Model

A controlled test can follow:

```text
Select Test User
      │
      ▼
Add User to Test Group
      │
      ▼
Windows Security Event
      │
      ▼
Wazuh
      │
      ▼
Verify Telemetry
      │
      ▼
Remove User
```

This validates identity-related telemetry separately from process execution.

---

# Why Group Changes Are Important

A group membership change can alter effective permissions.

```text
User
  │
  ▼
Added to Group
  │
  ▼
New Authorization
```

Privileged-group changes therefore deserve greater attention than ordinary process execution.

---

# Testing on DC1

Group-management tests should be reviewed carefully on:

```text
DC1
```

because DC1 manages the Active Directory environment.

Useful Wazuh filtering begins with:

```text
agent.name: DC1
```

and then narrows to the relevant Windows Event ID.

---

# Local Event Verification

Before troubleshooting Wazuh, confirm that Windows generated the expected event.

This follows the same principle as process detection:

```text
No Local Event
      │
      ▼
Do Not Troubleshoot Wazuh Yet
```

The logging source must work first.

---

# Test 5 — Sysmon Process Detection

Sysmon provides a second process telemetry source.

A controlled process can be launched and validated through:

```text
Sysmon Event ID 1
```

The event should appear in:

```text
Microsoft-Windows-Sysmon/Operational
```

---

# Why Test Sysmon Separately?

Windows Security Event 4688 and Sysmon Event ID 1 describe similar activity but come from different event providers.

A rule designed around one source may not work with the other.

Testing both helps identify exactly which event source a detection depends on.

---

# Event Provider Validation

When reviewing a Wazuh event, confirm:

```text
Provider
Event ID
Process
Command Line
Agent
```

Do not assume a process event came from Windows Security merely because the behavior was process creation.

---

# Detection Test Matrix

A useful validation matrix is:

| Test | Endpoint | Expected Telemetry | Expected Result |
|---|---|---|---|
| Basic process execution | Client01 / DC1 | `4688` or Sysmon `1` | Event visible in Wazuh |
| Command-line activity | Client01 | Process + command line | Event searchable |
| Account creation | Test Windows endpoint | `4688` / Sysmon + `4720` | Account activity visible |
| Custom command detection | Test endpoint | Custom rule fields | Rule `100103` or `100104` as configured |
| Security group change | DC1 | `4728/4729/4732/4733/4756/4757` | Group change visible |
| Sysmon process test | DC1 | Sysmon `1` | Event visible in Wazuh |

The exact custom-rule mapping should always be verified against the current rule file.

---

# Expected vs Actual Results

Each detection test should document:

```text
Test Activity:
What was performed?

Expected Windows Event:
What should Windows generate?

Expected Wazuh Event:
What should arrive in the SIEM?

Expected Rule:
Which custom rule should match?

Actual Result:
What actually happened?

Status:
Pass / Fail
```

This provides clear evidence that a detection was intentionally validated.

---

# Suggested Test Record

A test record can be documented like this:

```text
Test Name:
Windows Account Management Detection

Endpoint:
Client01

Activity:
Controlled lab account-management command

Expected Telemetry:
Windows process event
Account-management event if applicable

Expected Custom Rule:
Verify against current custom rule configuration

Actual Result:
Expected event and custom alert observed

Status:
PASS
```

This format can be reused for future detections.

---

# Pass Criteria

A custom detection test should only be marked:

```text
PASS
```

when all required stages succeed.

```text
Activity Performed        ✓
Local Event Generated     ✓
Event Reaches Wazuh       ✓
Expected Fields Present   ✓
Expected Rule Matches     ✓
Alert Visible             ✓
```

If the wrong rule fires, the test should not be considered fully successful.

---

# Failure Classification

When a test fails, classify the problem.

## Logging Failure

```text
Local Windows Event Missing
```

Investigate:

- Audit Policy
- Group Policy
- Sysmon
- Windows configuration

---

## Collection Failure

```text
Local Event Exists
Wazuh Event Missing
```

Investigate:

- Wazuh agent
- Event-channel collection
- Connectivity
- Agent configuration

---

## Parsing Failure

```text
Event Arrives
Expected Field Missing
```

Investigate:

- Event provider
- Decoder
- Parsed fields
- Event format

---

## Detection Failure

```text
Event Exists
Expected Fields Exist
Custom Rule Does Not Fire
```

Investigate:

- Rule conditions
- Field names
- Rule dependencies
- Rule conflict
- Match logic

---

# Rule Conflict Testing

A major troubleshooting lesson involved the earlier custom rule:

```text
100100
```

which affected the behavior of later custom detections.

When a different rule fires than expected:

```text
Event
  │
  ▼
Unexpected Rule
  │
  ▼
Investigate Rule Relationships
```

rather than immediately changing the endpoint configuration.

---

# Verify the Actual Rule

When an alert appears, always check:

```text
rule.id
```

A visible alert is not sufficient evidence that the intended custom detection worked.

For example:

```text
Expected:
100103

Actual:
100100
```

means the detection still requires investigation.

---

# Wazuh Manager Validation

Rule changes can affect the Wazuh Manager.

After updating custom rules, verify:

```bash
systemctl status wazuh-manager
```

The service should remain healthy.

A malformed rule configuration previously caused a manager restart failure during this project, reinforcing the importance of validating SIEM configuration changes.

---

# Change Testing Workflow

The rule-change workflow is:

```text
Working Configuration
       │
       ▼
Backup
       │
       ▼
Make One Rule Change
       │
       ▼
Validate
       │
       ▼
Restart / Reload
       │
       ▼
Check Manager
       │
       ▼
Generate Test Event
       │
       ▼
Verify Alert
```

Making one change at a time makes failures much easier to troubleshoot.

---

# Positive Testing

Positive testing asks:

> Does the rule fire when the intended behavior occurs?

Example:

```text
Expected Suspicious Behavior
          │
          ▼
        Rule Fires
```

This proves that the detection can identify the targeted activity.

---

# Negative Testing

Negative testing asks:

> Does the rule stay quiet when similar but benign activity occurs?

Conceptually:

```text
Normal Administrative Activity
           │
           ▼
       No Alert
```

Negative testing helps identify overly broad rules.

---

# Why Negative Testing Matters

A detection that fires during every normal administrative action can create:

```text
Alert Noise
    │
    ▼
Alert Fatigue
    │
    ▼
Reduced Analyst Attention
```

A useful detection should distinguish between the intended behavior and ordinary activity as much as practical.

---

# False Positive Evaluation

When a rule generates an alert, ask:

```text
Could a legitimate administrator commonly perform this activity?
```

If the answer is yes, the alert may still be valuable, but its:

- Severity
- Description
- Conditions
- Context

may need adjustment.

---

# False Negative Evaluation

A detection should also be tested using reasonable variations of the same behavior.

For example:

```text
Behavior Variant A
Behavior Variant B
Behavior Variant C
```

If only one exact representation matches, the rule may be too narrow.

Testing variations helps identify gaps.

---

# Detection Confidence

Multiple telemetry sources can increase confidence.

For example:

```text
Process Detection
      │
      +
Account Creation Event
      │
      ▼
Higher Confidence
```

If both the command activity and resulting account event are observed, the investigation has stronger evidence.

---

# Event Correlation Mindset

Even when Wazuh rules are tested individually, security investigations should consider related events.

For example:

```text
Process Creation
       │
       ▼
Account Creation
       │
       ▼
Group Membership Change
```

Together, these events may tell a much stronger story than any one event alone.

---

# Timeline Analysis

Accurate timestamps allow related events to be reviewed in sequence.

Example:

```text
14:01:05
net.exe executed

14:01:06
User account created

14:01:12
User added to group
```

This sequence provides significantly more investigative context than isolated alerts.

---

# Evidence Collection

Useful evidence for each test may include screenshots showing:

- Windows Event Viewer
- PowerShell event query output
- Wazuh event search
- Custom rule ID
- Agent name
- Relevant process
- Relevant event ID
- Alert details

Screenshots should be cropped to show the important evidence clearly.

---

# Recommended Screenshot Structure

A strong detection test can use three screenshots.

```text
1. Windows Source Event

2. Wazuh Raw / Parsed Event

3. Final Custom Alert
```

This visually demonstrates the complete detection pipeline.

---

# Screenshot Security

Before publishing screenshots, review them for:

```text
Passwords
Tokens
Private Keys
Personal Information
VPN Credentials
Sensitive Account Data
```

Sensitive information should not be committed to a public GitHub repository.

---

# Screenshot Directory

Detection screenshots can be organized under:

```text
05-wazuh-siem/screenshots/
```

For example:

```text
05-wazuh-siem/
├── detection-testing.md
└── screenshots/
    ├── process-event-4688.png
    ├── sysmon-event-1.png
    ├── wazuh-rule-100103.png
    └── wazuh-rule-100104.png
```

Only add screenshots that clearly demonstrate something useful.

---

# Screenshot Naming

Descriptive filenames are preferable to:

```text
Screenshot1.png
Screenshot2.png
```

Use names such as:

```text
client01-process-4688.png

dc1-sysmon-process-event.png

wazuh-rule-100103-alert.png

wazuh-rule-100104-alert.png
```

This makes the repository easier to navigate.

---

# Example Markdown Screenshot

A screenshot can be added using:

```markdown
![Wazuh custom detection alert](screenshots/wazuh-rule-100103-alert.png)
```

A short explanation should be placed near the image describing what it proves.

---

# Detection Evidence Standard

A useful rule for portfolio evidence is:

> Every screenshot should prove something.

For example:

```text
Screenshot:
Wazuh alert with rule.id 100103

Proves:
The custom detection successfully matched during controlled testing.
```

Avoid adding screenshots only for decoration.

---

# Test Documentation Table

As more detections are built, results can be summarized in a table.

| Detection | Data Source | Endpoint | Result |
|---|---|---|---|
| Process telemetry | Windows 4688 / Sysmon 1 | Client01 | Pass |
| Account-management telemetry | Windows Security | Test endpoint | Pass |
| Custom Rule `100103` | Wazuh Custom Rules | Test endpoint | Pass |
| Custom Rule `100104` | Wazuh Custom Rules | Test endpoint | Pass |
| Security-group telemetry | Windows Security | DC1 | Pass |

The custom rule descriptions should be updated from the actual rule file if their behavior changes later.

---

# Reproducibility

A good detection should be reproducible.

That means another test should produce the same expected result.

```text
Test 1
   │
   ▼
Expected Alert


Test 2
   │
   ▼
Expected Alert


Test 3
   │
   ▼
Expected Alert
```

A detection that works only once may indicate inconsistent telemetry or overly fragile logic.

---

# Regression Testing

Whenever a custom detection is changed, previously working behavior should be tested again.

```text
Modify Rule
    │
    ▼
Test New Behavior
    │
    ▼
Retest Existing Behavior
```

This helps ensure that fixing one problem did not break another detection.

---

# Detection Lifecycle

The overall detection lifecycle is:

```text
Design
  │
  ▼
Implement
  │
  ▼
Test
  │
  ▼
Validate
  │
  ▼
Tune
  │
  ▼
Document
  │
  ▼
Retest
```

Detection engineering is an ongoing process rather than a one-time configuration task.

---

# Current Validation State

The current Wazuh environment has successfully demonstrated:

```text
Windows Telemetry Generation
            │
            ▼
Wazuh Agent Collection
            │
            ▼
Central SIEM Ingestion
            │
            ▼
Custom Detection Rules
            │
            ▼
Rules 100103 / 100104
            │
            ▼
Expected Alerts
```

The custom rules are functioning after resolving earlier configuration and rule-matching issues.

---

# Future Detection Tests

Future validation scenarios may include:

- PowerShell execution
- PowerShell Script Block Logging
- New Windows user creation
- Privileged-group additions
- Privileged-group removals
- Failed authentication patterns
- Scheduled task creation
- Windows service creation
- Remote administration activity
- Security-log clearing
- Sysmon parent-child process relationships
- Suspicious administrative utilities

Each scenario should follow the same evidence-based testing process.

---

# Future MITRE ATT&CK Validation

Future detections can also be mapped to relevant MITRE ATT&CK techniques.

The workflow would be:

```text
Test Behavior
      │
      ▼
Validate Detection
      │
      ▼
Identify ATT&CK Technique
      │
      ▼
Document Mapping
```

ATT&CK mapping should be based on the behavior actually detected rather than applied generically.

---

# Detection Testing Checklist

A repeatable test can use:

```text
[ ] Wazuh agent online
[ ] Audit policy verified
[ ] Test activity executed
[ ] Local Windows event confirmed
[ ] Event found in Wazuh
[ ] Correct agent confirmed
[ ] Expected fields confirmed
[ ] Expected custom rule confirmed
[ ] Alert reviewed
[ ] False-positive behavior considered
[ ] Test artifacts cleaned up
[ ] Evidence captured
[ ] Result documented
```

This provides a consistent validation process for future custom detections.

---

# Skills Demonstrated

This project demonstrates practical experience with:

## Detection Validation

- Controlled security testing
- Positive testing
- Negative testing
- Regression testing
- Detection reproducibility

## Windows Security

- Event ID 4688
- Event ID 4720
- Group-management events
- Sysmon Event ID 1
- Command-line telemetry

## Wazuh

- Event searching
- Agent filtering
- Rule-ID validation
- Custom rules
- Alert analysis
- Manager health validation

## Detection Engineering

- Test-case design
- Event-field validation
- Rule troubleshooting
- False-positive analysis
- False-negative analysis
- Evidence collection

## Security Operations

- Timeline analysis
- Event correlation
- Investigation workflow
- Detection documentation

---

# What I Learned

Detection testing reinforced that:

> **Seeing an alert is not enough. The correct activity must generate the correct telemetry and trigger the correct rule.**

A complete test requires:

```text
Known Activity
      │
      ▼
Known Telemetry
      │
      ▼
Known Detection
      │
      ▼
Known Result
```

Another important lesson was:

> **Detection troubleshooting should be evidence-driven. Start at the endpoint, verify each stage, and only change the component that is actually failing.**

This approach turned the Wazuh environment from a simple SIEM installation into a repeatable detection-engineering lab.

---

# Detection Testing Summary

```text
                     Controlled Test
                           │
                           ▼
                     Windows Endpoint
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
        Security Events              Sysmon
        4688 / 4720 /              Event ID 1
        Group Events                   │
              │                         │
              └────────────┬────────────┘
                           ▼
                       Wazuh Agent
                           │
                           ▼
                       Wazuh SIEM
                           │
                           ▼
                    Parsed Telemetry
                           │
                           ▼
                  Custom Detection
                           │
                           ▼
                    100103 / 100104
                           │
                           ▼
                       Alert
                           │
                           ▼
                  Evidence + Validation
```

The detection-testing process provides repeatable evidence that the custom security monitoring pipeline works from endpoint activity through final SIEM alerting.

---

# Related Documentation

[← Wazuh Custom Detection Rules](custom-detection-rules.md)

[← Wazuh Log Collection and Telemetry Pipeline](log-collection.md)

[← Wazuh Windows Agents](windows-agents.md)

[← Wazuh SIEM Deployment](wazuh-deployment.md)

[← Windows Security Auditing and Sysmon](../04-active-directory/windows-security-auditing.md)

[← Main Project README](../README.md)
