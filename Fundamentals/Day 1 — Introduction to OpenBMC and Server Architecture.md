# OpenBMC Learning Series — Day 1

## Introduction to OpenBMC and Server Architecture

Welcome to my **OpenBMC Learning Series**.

The goal of this series is to understand OpenBMC **from the ground up** — starting with basic server architecture and gradually moving toward the internals of OpenBMC, including Linux, Yocto, BitBake, D-Bus, systemd, sensors, IPMI, Redfish, PLDM, MCTP, development, debugging, and more.

The learning path will follow:

```text
Server
   ↓
BMC
   ↓
OpenBMC
   ↓
Linux
   ↓
Yocto / OpenEmbedded
   ↓
OpenBMC Architecture
   ↓
D-Bus / systemd / Services
   ↓
Sensors / Hardware Management
   ↓
IPMI / Redfish / PLDM / MCTP
   ↓
OpenBMC Development
   ↓
Debugging & Real-World Applications
```

The objective is not simply to memorize OpenBMC components, but to understand **how the pieces fit together**.

---

# 1. What is a Server?

A server is a computer system designed to provide computing resources and services to other systems or users.

A typical server contains several major hardware components:

- CPU
- Main memory (RAM)
- Persistent storage
- Network interfaces
- Power-management hardware
- Cooling systems
- Sensors
- A Baseboard Management Controller (BMC)

The CPU, RAM and storage form the primary computing system that runs the operating system and applications.

The BMC belongs to a separate management subsystem whose purpose is to monitor and control the server hardware.

---

# 2. Understanding the Host System

The **host system** is the main computing system of the server.

A simplified view is:

```text
        HOST SYSTEM
 ┌─────────────────────────┐
 │                         │
 │   RAM ←→ CPU ←→ Storage │
 │                         │
 │     Runs OS / Apps      │
 │                         │
 └─────────────────────────┘
```

### CPU

The CPU (Central Processing Unit) executes instructions and performs computations.

It is responsible for running the operating system and applications on the host.

### RAM

RAM (Random Access Memory) provides temporary working memory for the host system.

Programs and data that are actively being used are loaded into RAM.

Main system RAM is physically separate from the CPU, although CPUs may contain smaller cache memories such as L1, L2 and L3 cache.

### Storage

Storage provides persistent data storage.

Examples include:

- HDD
- SSD
- NVMe SSD

The operating system, applications and persistent data can be stored here.

---

# 3. What is a BMC?

BMC stands for:

> **Baseboard Management Controller**

A BMC is a dedicated management controller present in many server platforms.

Instead of being responsible for running the main server workload, the BMC is responsible for **monitoring and managing the platform hardware**.

A simplified view is:

```text
              SERVER
 ┌──────────────────────────────┐
 │                              │
 │       HOST SYSTEM            │
 │                              │
 │      CPU ─── RAM ─── Storage │
 │                              │
 │              │               │
 │              │               │
 │             BMC              │
 │              │               │
 │     ┌────────┼────────┐      │
 │     │        │        │      │
 │  Sensors   Fans    Power      │
 │                              │
 └──────────────────────────────┘
```

The BMC can interact with various hardware components and provide management capabilities to administrators.

---

# 4. What Does a BMC Do?

The exact capabilities depend on the platform and implementation, but common BMC responsibilities include:

### Hardware Monitoring

The BMC can monitor information such as:

- Temperature
- Fan speed
- Voltage
- Power
- Hardware health
- System events

### Hardware Control

The BMC can also perform management operations such as:

- Power on
- Power off
- Power cycle
- Fan control
- Hardware reset
- Firmware-related operations

OpenBMC lists host-management capabilities including power, cooling, LEDs, inventory, events and watchdog functionality.

### Remote Management

A major advantage of a BMC is that administrators can interact with the server remotely through a management network.

Depending on the platform and configuration, management functionality can include interfaces such as:

- IPMI
- Redfish
- SSH
- Remote KVM
- Virtual media
- REST interfaces

The OpenBMC architecture documentation describes the BMC's management network and interfaces such as Redfish and IPMI.

---

# 5. Why is the BMC Important?

Consider a server located inside a data center.

Suppose the host operating system becomes unresponsive.

Without a management controller, an administrator may need physical access to the server.

With a BMC, many management operations can be performed remotely.

For example:

```text
Administrator
      │
      │ Management Network
      ▼
     BMC
      │
      ├── Check Temperature
      ├── Check Fan Status
      ├── Check Power Status
      ├── Access Console
      └── Power Cycle Server
```

This is commonly associated with **out-of-band management**.

The management path is separate from the normal workload running on the host.

---

# 6. BMC and Host: What is the Difference?

This distinction is extremely important when learning OpenBMC.

| Host System | BMC |
|---|---|
| Runs the main OS | Runs management software |
| Executes user workloads | Monitors/manages hardware |
| Uses CPU, RAM and storage for applications | Has its own management processor and resources |
| Handles normal computing | Handles platform management |
| Connected to the workload/data network | Often connected to a management network |
| Can become unavailable if the OS crashes | Designed to provide management independently |

The exact hardware architecture varies between platforms, so the connections between the BMC and host are implementation-dependent. The OpenBMC architecture documentation explicitly notes that host/BMC interfaces vary considerably across platforms.

---

# 7. Where Does OpenBMC Come In?

Now we reach the main topic of this series.

We have:

```text
SERVER
  │
  ├── Host System
  │      ├── CPU
  │      ├── RAM
  │      └── Storage
  │
  └── BMC
```

But the BMC needs software.

This is where **OpenBMC** comes in.

OpenBMC is an open-source Linux distribution/software stack for management controllers used in systems such as servers, top-of-rack switches and RAID appliances. The project uses technologies including Yocto, OpenEmbedded, systemd and D-Bus to build and customize the software for different platforms.

So a useful mental model is:

```text
              SERVER
                 │
       ┌─────────┴─────────┐
       │                   │
   HOST SYSTEM             BMC
       │                   │
   CPU/RAM/Storage     OpenBMC Software
                           │
                    Linux + Services
                           │
              Hardware Management
```

---

# 8. Hardware vs Software

One of the most important concepts to understand from the beginning is:

### BMC ≠ OpenBMC

They are related, but they are not the same thing.

**BMC**

→ Hardware management controller

**OpenBMC**

→ Software stack / Linux distribution that runs on the management controller

A simple analogy:

```text
Computer
   │
   ├── Hardware
   └── Operating System

BMC Platform
   │
   ├── BMC Hardware
   └── OpenBMC Software
```

This distinction will become very important when we start exploring OpenBMC source code.

---

# 9. A Small Real-World Example

Imagine a server running inside a data center.

The host is running a Linux operating system.

Now imagine that the server temperature becomes too high.

A simplified flow could look like:

```text
Temperature Sensor
        │
        ▼
       BMC
        │
        ├── Detect temperature
        │
        ├── Report health information
        │
        └── Take appropriate management action
```

An administrator can also access the BMC remotely through the management network and inspect the server's health.

This is the basic idea behind platform management.

Later in this series, we will learn **how OpenBMC actually represents sensors, communicates between services, exposes management APIs, and interacts with hardware.**

---

# 10. Understanding the Diagram

The Day 1 diagram represents a simplified server architecture.

### Host System

Contains:

```text
RAM ↔ CPU ↔ Storage
```

This is the primary computing system that runs the operating system and applications.

### Management Subsystem

Contains:

```text
             BMC
              │
     ┌────────┼────────┐
     │        │        │
Temperature  Fan    Voltage
 Sensors    Sensors Sensors
              │
        Power Control
```

The BMC communicates with hardware components and exposes management functionality.

### Management Network

The BMC can also communicate over a management network.

This allows administrators or management software to interact with the BMC remotely.

---

# 11. The Bigger Picture

At the end of this series, we want to understand a complete flow such as:

```text
                MANAGEMENT CLIENT
                       │
                       │
                Management Network
                       │
                       ▼
                    BMC
                       │
                  OpenBMC
                       │
        ┌──────────────┼──────────────┐
        │              │              │
      D-Bus         systemd       BMCWeb
        │              │              │
        └──────────────┼──────────────┘
                       │
                  Hardware Layer
                       │
        ┌──────────────┼──────────────┐
        │              │              │
     Sensors          Fans         Power
```

We will gradually build this understanding instead of trying to learn everything at once.

---

# 12. What We Will Learn in This Series

The series will progressively cover:

### Fundamentals

- Server architecture
- BMC
- Out-of-band management
- BMC hardware
- BMC boot process

### Linux

- Embedded Linux
- Processes
- Filesystems
- systemd
- Services
- Device Tree
- Kernel and drivers

### Yocto / OpenEmbedded

- Yocto
- OpenEmbedded
- BitBake
- Layers
- Recipes
- Configuration
- Images
- Cross-compilation

### OpenBMC Architecture

- OpenBMC source tree
- Phosphor architecture
- D-Bus
- systemd services
- Entity Manager
- State managers
- Sensors
- Hardware abstraction

### Management Interfaces

- IPMI
- Redfish
- REST
- SSH
- KVM
- Virtual Media

### Advanced Technologies

- PLDM
- MCTP
- Firmware update
- Security
- Secure boot
- Host/BMC communication

### Development

- Building OpenBMC
- Running OpenBMC in QEMU
- Creating/modifying recipes
- Writing OpenBMC applications
- Creating D-Bus interfaces
- Debugging services
- Reading OpenBMC source code

---

# 13. Key Takeaways

Before moving to Day 2, remember these five points:

1. **A server has a primary host system and a management subsystem.**

2. **The host system contains components such as CPU, RAM and storage and runs the main operating system and applications.**

3. **A BMC is a dedicated controller for monitoring and managing the server platform.**

4. **The BMC can provide remote management capabilities through a management network.**

5. **OpenBMC is the open-source Linux-based software stack used to build the software running on BMCs.**

The official OpenBMC project describes OpenBMC as a Linux distribution for management controllers and provides documentation covering architecture, development and usage.

---

## Official References

- [OpenBMC Project](https://github.com/openbmc/openbmc)
- [OpenBMC Documentation](https://github.com/openbmc/docs)
- [OpenBMC Interface Overview](https://github.com/openbmc/docs/blob/master/architecture/interface-overview.md)

---

## Series Progress

**Day 1 ✅ — Introduction to OpenBMC & Server Architecture**

**Day 2 → What Exactly Does a BMC Do?**

**Day 3 → BMC Hardware Architecture**

**Day 4 → How Does a BMC Work When the Host is OFF?**

**Day 5 → What is OpenBMC?**

> One concept at a time.  
> From fundamentals to OpenBMC internals.