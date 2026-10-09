# VLAN Design and Network Segmentation

## Overview

VLANs are used in this home lab to separate systems based on their role and level of trust.

Instead of placing every device into one large Layer 2 network, the environment uses logical segmentation to create separate security zones.

The current VLAN design is:

| VLAN | Name | Purpose | Status |
|---|---|---|---|
| VLAN 10 | LAB-LAN | Trusted lab infrastructure | Active |
| VLAN 20 | IOT | Smart and IoT devices | Active |
| VLAN 99 | Management | Infrastructure administration | Planned |
| VLAN 1 | Default | Limited/default use | Existing |

The long-term objective is to use pfSense firewall policies to control communication between these networks.

---

# Why Network Segmentation?

Without VLAN segmentation, devices connected to the same switch could exist within the same broadcast domain.

For example:

```text
Flat Network

Client01
   │
DC1
   │
Proxmox
   │
IoT Device
   │
Smart TV
