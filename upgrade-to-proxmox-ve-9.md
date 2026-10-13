# Proxmox VE 9 Upgrade

## Overview

As part of the continued maintenance and modernization of the home lab, the Dell PowerEdge R710 virtualization host was upgraded to **Proxmox VE 9**.

The upgrade was more than a routine application update because Proxmox is one of the most critical components of the environment.

The host supports infrastructure including:

- pfSense
- Windows Server 2025 Domain Controller
- Wazuh SIEM
- Tenable Nessus
- Windows client systems
- Linux virtual machines

Because several network and security services depend on Proxmox, the upgrade required careful planning, package management, reboot coordination, troubleshooting, and post-upgrade verification.

---

# Environment

The Proxmox host is:

| Setting | Value |
|---|---|
| Physical Server | Dell PowerEdge R710 |
| Hypervisor | Proxmox VE |
| Hostname | `pve` |
| Management IP | `10.0.0.5` |
| Default Gateway | `10.0.0.1` |
| Current Major Version | Proxmox VE 9 |
| Debian Base | Debian 13 "Trixie" |

The server currently operates as the primary virtualization platform for the home lab.

---

# Why the Upgrade Was Important

Keeping the hypervisor current is important for several reasons.

These include:

- Security updates
- Kernel updates
- Bug fixes
- Improved hardware and software support
- Current package dependencies
- Continued Proxmox support
- Long-term platform maintainability

Because the hypervisor hosts security-critical infrastructure, delaying major updates indefinitely would create additional operational and security risk.

---

# Infrastructure Dependency Considerations

Before performing the upgrade, it was important to understand how many systems depended on the Proxmox host.

```text
Dell PowerEdge R710
        │
        ▼
     Proxmox
        │
        ├── pfSense
        │      │
        │      └── Routing / Firewall / VPN
        │
        ├── DC1
        │      │
        │      ├── Active Directory
        │      ├── DNS
        │      └── DHCP
        │
        ├── Wazuh
        │      │
        │      └── SIEM
        │
        ├── Nessus
        │      │
        │      └── Vulnerability Management
        │
        └── Client / Test VMs
```

A failed hypervisor upgrade could therefore affect much more than the Proxmox management interface.

---

# Upgrade Risk

The most important architectural consideration was that **pfSense is virtualized on the same Proxmox host**.

This creates the following dependency:

```text
Proxmox
   │
   ▼
pfSense VM
   │
   ▼
Default Gateway
   │
   ▼
Home Lab Connectivity
```

Rebooting the hypervisor temporarily brings pfSense offline.

That means the lab can temporarily lose:

- Internet access
- Firewall services
- Inter-VLAN routing
- OpenVPN access
- Surfshark VPN connectivity

This had to be expected as part of the maintenance window.

---

# Virtual Machine Preparation

Before making major hypervisor changes, running workloads should be reviewed.

Particular attention is given to critical systems such as:

```text
pfSense
DC1
Wazuh
Nessus
```

The environment also makes use of snapshots around major configuration changes when appropriate.

For a hypervisor upgrade, the important objective is to know:

- Which VMs are running
- Which services depend on them
- Which systems need to return first
- Whether current configuration and recovery options are available

---

# Expected Startup Priority

After a Proxmox reboot, the logical service priority is:

```text
1. Proxmox
      │
      ▼
2. pfSense
      │
      ▼
3. DC1
      │
      ▼
4. Wazuh / Nessus
      │
      ▼
5. Client and Test Systems
```

This order reflects the dependencies of the lab.

pfSense restores network connectivity.

DC1 restores:

- DNS
- DHCP
- Active Directory
- Authentication

Security and client systems can then return afterward.

---

# Repository Transition

Proxmox VE 9 uses the Debian **Trixie** package base.

The system repositories were therefore transitioned to the appropriate Debian and Proxmox package sources.

After the repository configuration was prepared, package metadata was refreshed.

```bash
apt update
```

The system successfully reached the Trixie repositories.

Example package sources included:

```text
deb.debian.org/debian trixie
security.debian.org/debian-security trixie-security
deb.debian.org/debian trixie-updates
download.proxmox.com/debian/pve trixie
```

This confirmed that the server was obtaining packages from the expected Debian and Proxmox repositories.

---

# Package Upgrade

The upgrade involved updating the installed Debian and Proxmox packages to the new release.

Major distribution upgrades can modify:

- Proxmox packages
- Linux kernels
- System libraries
- Perl packages
- Networking components
- Systemd services
- Virtualization components
- Management services

For this reason, package output was monitored during the upgrade rather than treating it as a simple unattended update.

---

# Service Warning During Upgrade

During package processing, a Postfix-related service warning appeared:

```text
Failed to restart postfix.service:
Transaction contains conflicting jobs 'restart' and 'stop'
for postfix@-.service.

Probably contradicting requirement dependencies configured.
```

The package-management process also reported:

```text
Could not execute systemctl:
at /usr/bin/deb-systemd-invoke line 148.
```

This was an important troubleshooting point because errors during a major hypervisor upgrade should not simply be ignored.

The upgrade process was allowed to complete, followed by a controlled host reboot and post-upgrade service validation.

---

# Why Post-Upgrade Validation Matters

The presence of an error during package installation does not automatically prove that the entire upgrade failed.

Likewise, reaching the login prompt after a reboot does not automatically prove that every service is healthy.

The correct approach is:

```text
Upgrade
   │
   ▼
Review Errors
   │
   ▼
Reboot
   │
   ▼
Validate Services
   │
   ▼
Validate Packages
   │
   ▼
Validate Networking
   │
   ▼
Validate VMs
```

This prevents assumptions about system health.

---

# Reboot

After package installation completed, the host was rebooted.

```bash
reboot
```

The SSH session closed as expected when the server restarted.

Example:

```text
The system will reboot now!

Connection to 10.0.0.5 closed by remote host.
Connection to 10.0.0.5 closed.
```

Because pfSense also runs on the server, temporary network interruption was expected during this process.

---

# Post-Reboot Validation

Once the server returned online, several checks were performed.

The goal was to verify:

- Proxmox accessibility
- Network connectivity
- Package state
- Systemd health
- VM availability

---

# Repository Validation

The package repositories were checked again using:

```bash
apt update
```

The final result showed the expected Trixie repositories:

```text
Hit:1 http://security.debian.org/debian-security trixie-security InRelease
Hit:2 http://deb.debian.org/debian trixie InRelease
Hit:3 http://deb.debian.org/debian trixie-updates InRelease
Hit:4 http://download.proxmox.com/debian/pve trixie InRelease
```

The package manager reported:

```text
All packages are up to date.
```

This confirmed that the host was successfully using the new package sources and did not have additional pending updates at that time.

---

# Systemd Service Validation

One of the most important post-upgrade checks was:

```bash
systemctl --failed
```

The final result was:

```text
UNIT LOAD ACTIVE SUB DESCRIPTION

0 loaded units listed.
```

This was a strong indication that the earlier service warning had not left the system with any failed systemd units.

---

# Why `systemctl --failed` Is Important

A Linux server can appear operational while background services remain in a failed state.

Running:

```bash
systemctl --failed
```

provides a quick way to identify failed systemd units.

For a hypervisor, this is particularly important because failures could affect:

- Networking
- Storage
- Proxmox services
- Logging
- Mail notifications
- Virtual machine management
- Cluster services

The goal after an upgrade is not simply:

```text
Server boots
```

but:

```text
Server boots
+
Services healthy
+
Networking works
+
VMs operate
```

---

# Package Cleanup

After confirming that the upgraded environment was stable, unused packages were reviewed with:

```bash
apt autoremove
```

The cleanup identified packages that were no longer required after the distribution and Proxmox upgrade.

This included older libraries and an older Proxmox kernel package.

One of the superseded kernel packages identified during cleanup was:

```text
proxmox-kernel-6.8.12-23-pve-signed
```

Removing obsolete packages helps keep the host cleaner and reduces unnecessary disk usage.

---

# Kernel Cleanup Considerations

Kernel cleanup should be performed carefully on a virtualization host.

The goal is not to remove kernels simply because they are old.

Before removing an older kernel, it is important to verify that:

- The new kernel boots successfully
- The host is stable
- Networking works
- Proxmox services work
- Virtual machines can start
- A known-good recovery path exists

A safe sequence is:

```text
Install New Kernel
       │
       ▼
Reboot
       │
       ▼
Verify New Kernel
       │
       ▼
Verify Proxmox
       │
       ▼
Verify VMs
       │
       ▼
Remove Superseded Packages
```

---

# Package Maintenance After the Upgrade

Routine package maintenance continues after the major version upgrade.

Typical commands include:

```bash
apt update
```

to refresh package metadata.

```bash
apt upgrade
```

to install available package updates when appropriate.

And:

```bash
apt autoremove
```

to identify dependencies that are no longer required.

Package maintenance on a hypervisor is performed more carefully than on a normal workstation because many other systems depend on the host.

---

# Network Validation

The Proxmox management address remained:

```text
10.0.0.5
```

with pfSense continuing to serve as the gateway:

```text
10.0.0.1
```

Basic host networking can be validated with commands such as:

```bash
ip addr
```

```bash
ip route
```

and:

```bash
ping 10.0.0.1
```

The Proxmox web interface can then be validated at:

```text
https://10.0.0.5:8006
```

---

# Virtual Machine Validation

The upgrade is not complete from an operational perspective until hosted workloads are functioning.

Critical VMs include:

```text
pfSense
DC1
Wazuh
Nessus
Client Systems
```

Each system supports a different portion of the environment.

---

# pfSense Validation

pfSense should return first because it provides:

- Default gateway services
- Firewalling
- NAT
- VLAN routing
- OpenVPN
- Surfshark VPN routing

Basic verification includes confirming that:

```text
10.0.0.1
```

is reachable and internal systems regain network connectivity.

---

# DC1 Validation

DC1 should then be verified at:

```text
10.0.0.6
```

Important services include:

- Active Directory
- DNS
- DHCP
- Group Policy

Example tests from a Windows client include:

```powershell
ping 10.0.0.6
```

and:

```powershell
nslookup dc1.ad.home.com
```

---

# Wazuh Validation

Wazuh should return at:

```text
10.0.0.7
```

The SIEM infrastructure depends on several services, including:

- Wazuh Manager
- Wazuh Indexer
- Wazuh Dashboard
- Filebeat

After a hypervisor reboot, the system should be checked to confirm that agents reconnect and security events resume.

---

# Nessus Validation

The vulnerability-management server operates at:

```text
10.0.0.8
```

After the hypervisor returns, the Nessus web interface and scanning service should also be confirmed operational before considering the entire environment restored.

---

# Upgrade Troubleshooting Model

A major hypervisor upgrade can fail at several layers.

The troubleshooting model used for this environment is:

```text
Package Repositories
        │
        ▼
Package Installation
        │
        ▼
Kernel
        │
        ▼
Systemd Services
        │
        ▼
Host Networking
        │
        ▼
Proxmox Management
        │
        ▼
Virtual Machines
        │
        ▼
Application Services
```

Each layer should be validated separately.

---

# Why Rebooting Is Significant in This Environment

In a normal workstation environment, rebooting after an upgrade primarily affects one computer.

In this lab:

```text
Reboot Proxmox
      │
      ▼
All Hosted VMs Stop
      │
      ├── pfSense
      ├── DC1
      ├── Wazuh
      ├── Nessus
      └── Clients
```

This makes hypervisor reboot planning similar to a small maintenance window.

The experience reinforced the importance of understanding service dependencies before restarting infrastructure.

---

# Change Management Approach

The upgrade followed several principles that also apply to enterprise change management:

### Understand the Impact

Identify which services depend on the system being changed.

### Review Configuration

Know the current network, VM, and storage configuration.

### Plan for Downtime

Expect temporary loss of hosted services.

### Monitor the Upgrade

Do not ignore package errors or warnings.

### Reboot When Required

Major kernel and platform upgrades require running the updated software.

### Validate Afterward

Check more than just the management GUI.

### Clean Up Carefully

Remove obsolete packages only after confirming stability.

---

# Lessons Learned

The Proxmox VE 9 upgrade reinforced several important infrastructure-management concepts.

I learned that:

- Hypervisor upgrades affect every hosted workload
- Repository configuration is critical during Linux distribution upgrades
- Package-manager output should be reviewed carefully
- Service warnings require post-upgrade validation
- A successful reboot does not automatically mean every service is healthy
- `systemctl --failed` is an important health check
- Old packages should only be removed after the new environment is verified
- Virtualizing pfSense increases the impact of a hypervisor reboot
- VM startup dependencies matter after maintenance
- Hypervisor maintenance requires thinking about the environment as a complete system

The most important lesson was:

> **A major infrastructure upgrade is not finished when the installation completes. It is finished when the environment has been validated and the dependent services are operational again.**

---

# Security Relevance

Keeping Proxmox updated is also part of the security posture of the home lab.

The hypervisor is one of the highest-value systems in the environment.

If compromised, it could potentially provide access to:

- Firewall infrastructure
- Domain services
- Security monitoring
- Vulnerability-management systems
- Client VMs

Maintaining current packages and supported software therefore contributes to:

```text
Patch Management
        │
        ▼
Vulnerability Reduction
        │
        ▼
Infrastructure Security
```

The upgrade demonstrates the connection between normal systems administration and cybersecurity operations.

---

# Availability Considerations

The upgrade also highlighted the availability risks of the current single-host architecture.

Currently:

```text
Dell R710
   │
   ▼
Proxmox
   │
   ├── pfSense
   ├── DC1
   ├── Wazuh
   └── Nessus
```

If maintenance or failure takes the server offline, many critical services become unavailable simultaneously.

This is another reason the future lab design includes adding a second Proxmox server.

---

# Future Multi-Host Improvement

The planned architecture will eventually move toward:

```text
                 Proxmox Infrastructure
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
         Proxmox Node 1        Proxmox Node 2
          Dell R710            Future Server
              │                     │
         Core Services         Security / Testing
```

Besides helping with RAM capacity, a second server will provide more flexibility during future maintenance.

Instead of every VM depending on one physical host, workloads can eventually be distributed between multiple systems.

---

# Upgrade Validation Summary

The final validation included:

```text
Proxmox VE 9
      │
      ├── Trixie repositories reachable
      │
      ├── Packages up to date
      │
      ├── Host reboot successful
      │
      ├── systemctl --failed = 0
      │
      ├── Obsolete packages reviewed
      │
      ├── Network connectivity restored
      │
      └── Virtualized services restored
```

The final package check reported:

```text
All packages are up to date.
```

and the final service check reported:

```text
0 loaded units listed.
```

The upgrade was therefore successfully completed and validated.

---

# Skills Demonstrated

This portion of the project demonstrates practical experience with:

### Linux Administration

- Debian package management
- `apt update`
- Package upgrades
- `apt autoremove`
- Repository management
- Systemd
- Kernel upgrades
- Linux troubleshooting

### Proxmox Administration

- Major-version upgrades
- Hypervisor maintenance
- VM dependency planning
- Reboot coordination
- Post-upgrade validation

### Troubleshooting

- Package installation warnings
- Systemd service validation
- Network validation
- VM validation
- Dependency analysis

### Infrastructure Operations

- Change planning
- Maintenance windows
- Service restoration
- Package cleanup
- Availability planning
- Post-change verification

---

# Upgrade Summary

```text
                Existing Proxmox Environment
                           │
                           ▼
                  Upgrade Preparation
                           │
                           ▼
               Debian / Proxmox Repositories
                           │
                           ▼
                    Package Upgrade
                           │
                           ▼
                 Postfix Service Warning
                           │
                           ▼
                        Reboot
                           │
                           ▼
                  Proxmox VE 9 Starts
                           │
                    ┌──────┴──────┐
                    │             │
                    ▼             ▼
              apt update    systemctl --failed
                    │             │
                    ▼             ▼
             Packages Current   0 Failed Units
                    │             │
                    └──────┬──────┘
                           ▼
                     apt autoremove
                           │
                           ▼
                  Remove Old Packages
                           │
                           ▼
                 Validate Hosted Services
                           │
                           ▼
                Upgrade Successfully Complete
```

The Proxmox VE 9 upgrade provided practical experience performing a major infrastructure change while maintaining awareness of networking, virtualization, operating-system, and application dependencies.

---

# Related Documentation

[← Proxmox Resource and Memory Management](resource-management.md)

[← Proxmox Virtual Networking](virtual-networking.md)

[← Proxmox Virtualization Architecture](proxmox-architecture.md)

[← Main Project README](../README.md)
