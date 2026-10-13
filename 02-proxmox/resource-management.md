# Proxmox Resource and Memory Management

## Overview

As the home lab expanded, the Dell PowerEdge R710 began hosting an increasing number of infrastructure and cybersecurity workloads simultaneously.

These workloads include systems such as:

- pfSense
- Windows Server 2025 Domain Controller
- Wazuh SIEM
- Tenable Nessus
- Windows client systems
- Linux virtual machines
- Additional testing systems

Running multiple services on a single physical server eventually resulted in high memory utilization on the Proxmox host.

This created an opportunity to troubleshoot real virtualization resource constraints and improve how memory is allocated across the environment.

---

# Current Virtualization Model

The environment currently uses a single physical virtualization server:

```text
Dell PowerEdge R710
        │
        ▼
    Proxmox VE
     10.0.0.5
        │
        ├── pfSense
        ├── DC1
        ├── Wazuh
        ├── Nessus
        ├── Client01
        └── Additional Lab VMs
```

All virtual machines currently share the physical resources available on this host.

These shared resources include:

- CPU
- RAM
- Storage
- Network interfaces

As additional systems were deployed, RAM became one of the primary resource constraints.

---

# The Memory Utilization Problem

During normal lab operation, the Proxmox host began showing very high memory utilization.

The issue became noticeable as more virtual machines were running at the same time.

Conceptually:

```text
Physical RAM
100%
 │
 │ ████████████████████████
 │
 │ pfSense
 │ DC1
 │ Wazuh
 │ Nessus
 │ Client VMs
 │ Linux VMs
 │ Proxmox Host
 │
 ▼
Very Little Available Memory
```

Although each individual virtual machine had a reasonable amount of RAM assigned to it, the combined allocation placed significant pressure on the physical host.

---

# Why This Happens

Virtualization makes it possible to run many systems on one server, but the underlying hardware still has finite resources.

For example:

```text
VM 1 RAM
   +
VM 2 RAM
   +
VM 3 RAM
   +
VM 4 RAM
   +
VM 5 RAM
   +
Hypervisor RAM
   =
Total Physical Memory Consumption
```

As the number of running VMs increases, the combined memory requirement can approach the physical capacity of the host.

This can result in:

- Reduced available memory
- Increased memory pressure
- Slower VM performance
- Host swapping
- VM startup failures
- Reduced flexibility when deploying new systems

---

# Identifying the Problem

The Proxmox web interface was used to monitor host-level memory utilization.

The host showed that a large percentage of available RAM was being consumed.

The next step was to determine whether the memory assignments for individual virtual machines could be optimized.

Important items reviewed included:

- VM configured memory
- Ballooning configuration
- Minimum memory
- Maximum memory
- Number of simultaneously running VMs
- Actual workload requirements

---

# Memory Allocation in Proxmox

Each virtual machine can be assigned a maximum amount of RAM.

A simple static configuration looks like:

```text
Physical Host
     │
     ├── VM1 → Fixed RAM
     ├── VM2 → Fixed RAM
     ├── VM3 → Fixed RAM
     └── VM4 → Fixed RAM
```

If every virtual machine reserves its full configured amount, memory utilization can increase quickly even when some VMs are not actively using all of their assigned RAM.

This is where memory ballooning becomes useful.

---

# Memory Ballooning

Proxmox supports memory ballooning for compatible virtual machines.

Memory ballooning allows Proxmox to dynamically adjust the amount of RAM available to a guest between configured minimum and maximum values.

Instead of permanently assigning the maximum amount of RAM:

```text
VM
│
└── 100% of configured RAM always reserved
```

ballooning allows:

```text
Configured Maximum
        │
        ▼
   ┌──────────┐
   │    VM    │
   └──────────┘
        ▲
        │
Configured Minimum
```

The virtual machine can use additional memory when required while allowing Proxmox to reclaim unused memory when the host needs it.

---

# Ballooning Device

The virtual machines in the environment were already configured with the Proxmox ballooning device.

This meant the underlying capability existed, but the minimum memory values also needed to be reviewed.

The presence of a ballooning device alone does not automatically guarantee that memory will be used efficiently.

The configuration must include appropriate:

```text
Minimum RAM
```

and:

```text
Maximum RAM
```

values.

---

# Minimum vs Maximum Memory

With ballooning enabled, a VM can be configured using two important values.

## Maximum Memory

This represents the largest amount of RAM the VM is allowed to use.

```text
Maximum
   │
   ▼
VM can grow up to this value
```

---

## Minimum Memory

This represents the amount of memory Proxmox should attempt to guarantee to the virtual machine.

```text
Maximum Memory
      │
      │
      │ Dynamic Range
      │
      ▼
Minimum Memory
```

This creates a flexible allocation range.

For example:

```text
              Maximum RAM
                   │
                   ▼
             ┌───────────┐
             │ Available │
             │ when VM   │
             │ needs it  │
             ├───────────┤
             │ Minimum   │
             │ Reserved  │
             └───────────┘
```

The exact values should depend on the operating system and workload.

---

# Adjusting Minimum Memory

During troubleshooting, the minimum memory settings of the virtual machines were reviewed and adjusted.

The goal was not simply to reduce RAM everywhere.

Critical infrastructure still needs enough memory to operate reliably.

Instead, the objective was to create more flexible memory allocation across the environment.

Conceptually:

```text
Before

VM1 ───────── Large Fixed Allocation
VM2 ───────── Large Fixed Allocation
VM3 ───────── Large Fixed Allocation
VM4 ───────── Large Fixed Allocation


After

VM1 ── Minimum ───────── Maximum
VM2 ── Minimum ───────── Maximum
VM3 ── Minimum ───────── Maximum
VM4 ── Minimum ───────── Maximum
```

This allows the host to use RAM more efficiently.

---

# Critical Infrastructure Considerations

Not every VM should be treated equally.

Some systems are more important to overall lab availability.

For example:

```text
pfSense
   │
   └── Network connectivity

DC1
   │
   ├── Active Directory
   ├── DNS
   └── DHCP

Wazuh
   │
   └── Security monitoring
```

These systems require enough memory to remain stable.

Memory optimization therefore has to balance:

```text
Efficiency
    +
Availability
    +
Performance
```

rather than simply assigning the smallest possible amount of RAM.

---

# pfSense Resource Priority

pfSense provides:

- Routing
- Firewall services
- NAT
- OpenVPN
- Surfshark VPN routing
- VLAN routing

If pfSense experiences resource exhaustion, connectivity throughout the environment could be affected.

Therefore, its resource allocation should remain conservative and stable.

---

# Domain Controller Resource Priority

DC1 provides several core services:

```text
Active Directory
DNS
DHCP
Group Policy
```

Memory pressure on the domain controller could affect authentication, name resolution, or DHCP services.

For this reason, DC1 should always maintain sufficient guaranteed memory.

---

# Wazuh Resource Requirements

Wazuh is one of the more resource-intensive systems in the environment.

The platform includes components such as:

- Wazuh Manager
- Wazuh Indexer
- Wazuh Dashboard
- Filebeat

Security telemetry must also be indexed and stored.

As the number of agents and events increases, Wazuh memory requirements can also increase.

This makes SIEM workloads an important consideration when planning virtualization capacity.

---

# Nessus Resource Requirements

Tenable Nessus can also create temporary increases in resource utilization during vulnerability scans.

Scanning activity can use:

- CPU
- RAM
- Network bandwidth

The VM may not require peak resources continuously, which makes careful resource allocation useful.

---

# Monitoring Host Memory

Host memory utilization should be monitored over time rather than relying on a single snapshot.

Important indicators include:

```text
Total RAM
Used RAM
Available RAM
Swap Usage
VM Memory
Ballooning Activity
```

The objective is to identify whether memory usage is:

```text
Normal workload utilization
```

or:

```text
Sustained resource exhaustion
```

before performance becomes severely affected.

---

# Linux Memory Reporting

Memory utilization on the Proxmox host can also be reviewed from the command line.

A useful command is:

```bash
free -h
```

This displays:

- Total memory
- Used memory
- Free memory
- Shared memory
- Buffers/cache
- Available memory
- Swap usage

Example structure:

```text
               total    used    free   shared   buff/cache   available
Mem:
Swap:
```

The `available` value is often more useful than simply looking at `free` memory because Linux intentionally uses unused RAM for caching.

---

# Process-Level Memory Usage

Running processes can be inspected using:

```bash
top
```

or:

```bash
htop
```

These tools help determine whether an individual host process is consuming an unusually large amount of memory.

This is useful for distinguishing:

```text
VM memory pressure
```

from:

```text
Host process memory pressure
```

---

# Checking Virtual Machine Memory

Proxmox allows resource consumption to be viewed for individual virtual machines.

When investigating memory pressure, I compare:

```text
Configured Memory
       │
       ▼
Observed VM Usage
       │
       ▼
Actual Workload Requirement
```

If a VM consistently uses much less than its configured maximum, it may be a candidate for more flexible ballooning settings.

---

# Resource Oversubscription

Virtualization platforms can sometimes allocate more potential resources than are physically available because not every VM uses its maximum simultaneously.

Conceptually:

```text
VM Maximum RAM Total
        >
Physical Server RAM
```

This is called resource oversubscription.

It can improve hardware utilization, but it also creates risk.

If every VM suddenly needs its maximum amount of memory:

```text
Multiple VMs Request RAM
          │
          ▼
Host Memory Exhaustion
```

For that reason, oversubscription must be monitored carefully.

---

# Why Simply Adding More VMs Is Not Sustainable

The current single-server architecture has a practical limit.

Every additional service increases demand on:

```text
CPU
RAM
Storage
Networking
```

Eventually:

```text
More Services
     │
     ▼
More VM Resources
     │
     ▼
Higher Host Utilization
     │
     ▼
Physical Hardware Limit
```

Resource tuning can delay this limit, but it cannot eliminate the physical limitations of the server.

---

# Current Mitigation Strategy

The immediate strategy for managing RAM utilization includes:

1. Reviewing VM memory assignments
2. Using memory ballooning where appropriate
3. Adjusting minimum memory values
4. Avoiding unnecessarily large VM allocations
5. Monitoring host resource utilization
6. Shutting down temporary lab systems when they are not required
7. Prioritizing critical infrastructure workloads

This allows the existing Dell R710 to continue supporting the environment efficiently.

---

# Future Solution — Second Proxmox Server

The long-term plan is to add a **second physical server** to the home lab.

The primary motivation is to increase compute and memory capacity and distribute workloads across more than one virtualization host.

The future architecture will look more like:

```text
                    Home Lab
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
       Proxmox Host 1       Proxmox Host 2
       Dell R710            Future Server
             │                   │
       ┌─────┼─────┐        ┌────┼────┐
       │     │     │        │    │    │
      VMs   VMs   VMs      VMs  VMs  VMs
```

Instead of every virtual machine competing for the RAM in one physical server, workloads can be distributed between the two hosts.

---

# How a Second Server Helps the RAM Problem

Currently:

```text
All Workloads
      │
      ▼
One Physical Server
      │
      ▼
One RAM Pool
```

Future design:

```text
                Workloads
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
      Server 1             Server 2
      RAM Pool             RAM Pool
```

This increases the total amount of physical memory available to the lab.

It also makes it possible to move resource-intensive workloads away from critical infrastructure.

For example:

```text
Proxmox Host 1
│
├── pfSense
├── DC1
└── Core Infrastructure


Proxmox Host 2
│
├── Wazuh
├── Nessus
├── Test Clients
└── Security Lab Systems
```

The final workload placement will depend on the hardware specifications of the second server.

---

# Benefits of Adding a Second Host

A second virtualization server would provide several benefits.

### Increased RAM Capacity

The most immediate benefit is additional physical memory for virtual machines.

### Workload Distribution

Resource-intensive security systems can be separated from critical network infrastructure.

### Greater Flexibility

New VMs can be deployed without placing all additional load on one server.

### Maintenance Flexibility

Some workloads could eventually be moved between hosts during maintenance.

### Reduced Resource Contention

Wazuh, Nessus, Windows Server, and testing systems would no longer need to compete for the same physical memory pool.

---

# Future Proxmox Cluster

A second server may also provide an opportunity to build a Proxmox cluster.

Conceptually:

```text
               Proxmox Cluster
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
        Node 1                 Node 2
     Dell R710             Future Server
```

A cluster could provide centralized management of multiple Proxmox nodes.

Possible future learning objectives include:

- Multi-node Proxmox administration
- Cluster networking
- VM migration
- Shared management
- Resource distribution
- Quorum concepts
- Backup strategy
- High-availability concepts

A second node by itself does not automatically provide full high availability, so HA would require additional architecture and planning.

---

# Single Point of Failure Consideration

The current design places nearly all virtual infrastructure on one server.

```text
Dell R710 Failure
      │
      ▼
Proxmox Offline
      │
      ├── pfSense Offline
      ├── DC1 Offline
      ├── Wazuh Offline
      ├── Nessus Offline
      └── Client VMs Offline
```

A second server would help reduce this infrastructure concentration.

Even if high availability is not immediately implemented, distributing workloads across two physical hosts would reduce the impact of resource exhaustion on a single machine.

---

# Future Workload Distribution

One possible future design could be:

```text
PROXMOX NODE 1
Core Infrastructure

├── pfSense
├── DC1
└── Critical Network Services


PROXMOX NODE 2
Security / Testing

├── Wazuh
├── Nessus
├── Windows Test Systems
├── Linux Test Systems
└── Security Lab Workloads
```

This separation would prevent intensive scanning or SIEM activity from consuming resources required by critical network services.

The final design will be based on measured workload requirements rather than arbitrary placement.

---

# Capacity Planning

The RAM issue demonstrated the importance of capacity planning.

Before deploying additional virtual machines, several questions should be considered:

```text
How much RAM does the VM actually need?

How much RAM does it normally use?

Does it require guaranteed memory?

Can ballooning be enabled?

Is the workload always running?

Is it resource-intensive only during specific operations?

How will the new VM affect total host utilization?
```

These questions become increasingly important as the environment grows.

---

# Performance vs Efficiency

Resource allocation requires balancing two competing goals:

```text
Performance
     │
     ▼
Give VM more resources
```

versus:

```text
Efficiency
     │
     ▼
Avoid wasting host resources
```

Assigning too little memory can hurt VM performance.

Assigning too much memory can reduce the resources available to other systems.

The goal is therefore:

```text
Right-size the workload
```

rather than:

```text
Allocate as much as possible
```

---

# Troubleshooting Workflow

When the Proxmox host experiences high memory utilization, the troubleshooting process is:

```text
High RAM Usage
      │
      ▼
Check Host Memory
      │
      ▼
Review VM Allocations
      │
      ▼
Identify Large Consumers
      │
      ▼
Check Ballooning
      │
      ▼
Review Minimum RAM
      │
      ▼
Adjust Allocations
      │
      ▼
Monitor Performance
```

If high utilization continues despite appropriate right-sizing:

```text
Sustained Capacity Problem
          │
          ▼
Add Physical Resources
          │
          ▼
Second Proxmox Server
```

---

# Resource Management as a Security Concern

Resource exhaustion can also affect security.

If the host runs out of available memory:

```text
Wazuh may stop processing events

Nessus scans may fail

DC1 performance may degrade

pfSense availability may be affected
```

This means capacity planning contributes to the:

```text
Availability
```

portion of the CIA security triad.

Security infrastructure must have enough resources to remain operational.

---

# Lessons Learned

Troubleshooting the Proxmox memory issue provided practical experience with virtualization capacity management.

I learned that:

- VM resource allocations must be considered collectively
- A running VM does not necessarily need its full maximum RAM continuously
- Ballooning can improve memory flexibility
- Minimum memory settings affect how resources can be reclaimed
- Critical infrastructure needs guaranteed resources
- Security platforms can consume significant RAM
- Host-level monitoring is essential
- Resource tuning cannot overcome physical hardware limits indefinitely
- Infrastructure growth eventually requires additional physical capacity

The biggest lesson was:

> **Virtualization does not create unlimited resources. It allows physical resources to be used more efficiently.**

As the environment grows, capacity planning becomes just as important as configuration.

---

# Current vs Future Architecture

## Current

```text
               Dell R710
                   │
                   ▼
               Proxmox
                   │
         Single Resource Pool
                   │
      ┌────────────┼────────────┐
      │            │            │
   Core VMs   Security VMs   Test VMs
```

## Future

```text
                     Lab Infrastructure
                            │
                 ┌──────────┴──────────┐
                 │                     │
                 ▼                     ▼
          Proxmox Node 1         Proxmox Node 2
          Core Services          Security / Lab
                 │                     │
           RAM Pool #1             RAM Pool #2
```

This future expansion will increase available memory, reduce resource contention, and provide additional opportunities to learn multi-host virtualization.

---

# Future Improvements

Planned resource-management improvements include:

- Add a second physical Proxmox server
- Distribute workloads between virtualization hosts
- Continue monitoring VM memory utilization
- Right-size VM allocations based on actual usage
- Improve backup architecture
- Explore Proxmox clustering
- Learn VM migration
- Investigate high-availability architecture
- Improve power-failure resilience
- Document resource baselines for critical VMs

---

# Skills Demonstrated

This portion of the project demonstrates practical experience with:

### Virtualization

- Proxmox VE
- VM resource allocation
- Memory ballooning
- Minimum and maximum RAM
- Resource oversubscription
- Capacity planning

### Troubleshooting

- Host memory analysis
- VM memory analysis
- Performance troubleshooting
- Resource contention analysis
- Infrastructure dependency analysis

### Systems Administration

- Linux memory monitoring
- `free`
- `top`
- `htop`
- Hypervisor administration

### Infrastructure Planning

- Workload right-sizing
- Capacity forecasting
- Multi-host planning
- Infrastructure scalability
- Availability planning

---

# Resource Management Summary

```text
Current Environment

Dell PowerEdge R710
        │
        ▼
    Proxmox VE
        │
        ▼
Shared CPU / RAM
        │
 ┌──────┼────────┬────────┐
 │      │        │        │
pfSense DC1    Wazuh    Nessus
                 │
                 ▼
          High RAM Usage
                 │
                 ▼
       Resource Optimization
                 │
        ┌────────┴────────┐
        │                 │
   Ballooning        Right-Sizing
        │                 │
        └────────┬────────┘
                 │
                 ▼
        Short-Term Solution
                 │
                 ▼
         Future Expansion
                 │
                 ▼
      Second Proxmox Server
```

The current optimization work improves the efficiency of the Dell R710, while the planned second server will provide the additional physical capacity required as the home lab continues to grow.

---

# Related Documentation

[← Proxmox Virtual Networking](virtual-networking.md)

[← Proxmox Virtualization Architecture](proxmox-architecture.md)

[← IP Addressing and Network Services](../01-network-infrastructure/ip-addressing.md)

[← Main Project README](../README.md)
