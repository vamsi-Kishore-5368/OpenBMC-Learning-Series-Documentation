# OpenBMC Learning Series — Day 5

## BMC ↔ Host Communication

### How Does the BMC Communicate With the Host?

Welcome to **Day 5** of my OpenBMC Learning Series.

In Day 1, we introduced the **Server Architecture** and understood where the BMC fits into a server.

In Day 2, we explored the primary responsibilities of a BMC:

> **Monitor → Control → Manage**

In Day 3, we explored the overall **BMC & OpenBMC architecture**.

In Day 4, we looked at the **physical hardware of the BMC**.

Now we need to answer another important question:

> **If the Host and BMC are separate computing environments, how do they communicate with each other?**

The answer is not a single interface.

Different server platforms can use different physical interfaces, transport mechanisms, and management protocols.

---

## 1. Why Does the BMC Need to Communicate With the Host?

The BMC does much more than monitor external sensors.

It may also need to exchange information with the host platform for tasks such as:

- Host power and state management
- Host IPMI communication
- Platform events
- Firmware and platform management
- Host console access
- Host health and status information
- Other platform-specific management functions

A simplified view is:

```text
             HOST SYSTEM
        +---------------------+
        | CPU / Memory / OS    |
        | Firmware / Devices   |
        +----------+----------+
                   |
                   |
          Host ↔ BMC Communication
                   |
                   v
                  BMC
```

The exact communication architecture varies between BMC and host implementations.

The official OpenBMC architecture documentation also treats the host-BMC interface set as platform-dependent rather than as one universal architecture.

---

## 2. Communication Layers: Interface, Transport and Data Model

A very important concept in host-BMC communication is that there are **multiple communication layers**.

In embedded systems, interfaces such as **UART, SPI and I²C** have their own communication rules and protocols.

In the OpenBMC host-BMC architecture, we additionally distinguish:

1. **Physical / Bus Interfaces**
2. **Transport Protocols**
3. **Management Protocols / Data Models**

A simplified model is:

```text
          PHYSICAL / BUS INTERFACE
                    |
                    v
             TRANSPORT LAYER
                    |
                    v
       MANAGEMENT PROTOCOL / DATA MODEL
                    |
                    v
                  BMC
```

The important point is:

> **Interfaces such as UART, SPI and I²C are themselves defined with communication rules. In the OpenBMC architecture, higher-level transport and management layers are considered separately from those physical or bus interfaces.**

OpenBMC's interface overview explicitly separates **host-BMC physical interfaces**, **transport protocols**, and **data models**.

---

## 3. Host-BMC Physical / Bus Interfaces

Common host-BMC physical interfaces include:

- LPC
- eSPI
- PCIe
- I²C / SMBus
- UART
- GPIO
- SPI
- Other platform-specific interfaces

A conceptual view is:

```text
               HOST
                 |
     +-----------+-----------+
     |           |           |
    LPC         eSPI        PCIe
     |           |           |
     +-----------+-----------+
                 |
        +--------+--------+
        |        |        |
       I²C      UART     GPIO
        |        |        |
        +--------+--------+
                 |
                BMC
```

Not every platform uses all of these interfaces.

The actual architecture depends on the server and BMC design.

---

## 4. LPC

**LPC (Low Pin Count)** is a legacy interface that has been used for communication between host platforms and BMCs.

A simplified representation is:

```text
HOST  ───────── LPC ─────────  BMC
```

Depending on the platform, LPC can provide a path for host-management traffic.

For example, host-facing IPMI can use LPC-based mechanisms such as **KCS**.

We will explore KCS and host IPMI in more detail in a future post.

---

## 5. eSPI

**eSPI (Enhanced Serial Peripheral Interface)** is another host-side interface used on some platforms.

A simplified view is:

```text
HOST  ───────── eSPI ─────────  BMC
```

eSPI can provide a host-to-BMC communication path for platform-management functions.

For Day 5, the important idea is:

> **eSPI is an interface/transport mechanism used by some platforms; it is not itself the same thing as a higher-level management protocol such as PLDM.**

---

## 6. I²C / SMBus

We introduced I²C/SMBus in Day 4 as a way for the BMC to communicate with sensors, EEPROMs and power-management devices.

Depending on the platform, I²C/SMBus can also be involved in host-BMC communication.

```text
HOST  ───── I²C / SMBus ─────  BMC
```

This demonstrates an important embedded-systems concept:

> **The same bus technology can be used for different purposes depending on the system architecture.**

---

## 7. PCIe

PCIe can also be used as a host-BMC communication path on some platforms.

```text
HOST  ───────── PCIe ─────────  BMC
```

We are not going into PCIe link training, enumeration, BARs or other PCIe internals here.

The important point is:

> **PCIe can be one of the host-BMC communication paths on some systems.**

---

## 8. UART and Host Console

UART provides serial communication.

One important use in BMC systems is access to the **host's serial console**.

A simplified flow is:

```text
HOST
  |
  | UART / Serial
  v
 BMC
  |
  v
Host Console Service
  |
  v
Management Client
```

OpenBMC provides host-console access through the BMC management path, including network IPMI and SSH-based console access.

This is especially useful when the host is unavailable through normal network access.

---

## 9. GPIO

**GPIO (General-Purpose Input/Output)** provides simple digital status and control signals.

For example:

```text
HOST  ───── GPIO ─────  BMC

         Status
         Reset
         Presence
         Control
```

GPIO is different from higher-level protocols such as IPMI or PLDM.

It is primarily used as a platform-level electrical signal interface.

A platform may use GPIO for:

- Power-related signals
- Reset
- Presence detection
- Status indications
- Other platform-specific control signals

---

## 10. Host IPMI

Now we move to a higher-level management protocol.

**IPMI (Intelligent Platform Management Interface)** is a widely used platform-management protocol.

In an in-band host-to-BMC scenario, IPMI commands can reach the BMC through host-facing channels such as **KCS or BT**, depending on the platform.

A simplified flow is:

```text
Host Software
      |
      | IPMI Command
      v
   KCS / BT
      |
      v
     BMC
      |
     ipmid
      |
      v
Management Logic
```

OpenBMC's IPMI architecture describes **KCS and BT as host-facing, session-less IPMI channels** and explains how their commands are processed by the BMC's `ipmid` infrastructure.

The useful mental model is:

> **KCS / BT → host-facing channels**

> **IPMI → management commands and protocol**

---

## 11. MCTP

**MCTP (Management Component Transport Protocol)** provides a transport mechanism for management communication across different physical media.

A simplified view is:

```text
Host / Platform
       |
       v
  Physical Interface
       |
       v
      MCTP
       |
       v
      BMC
```

OpenBMC documents MCTP as a host-BMC transport protocol and notes support for MCTP over interfaces such as LPC, PCIe and UART.

A useful mental model is:

> **MCTP provides a common transport layer for management communication across different physical bindings.**

---

## 12. PLDM

**PLDM (Platform Level Data Model)** is a standardized management protocol and set of data models used for platform-management communication.

PLDM can be used for functions such as:

- Platform monitoring
- Control
- Inventory
- Firmware update
- Other management operations

A simplified relationship is:

```text
                 PLDM
                  |
                  v
                 MCTP
                  |
                  v
          Physical Transport
                  |
                  v
          Host / BMC / Device
```

This gives us an important distinction:

> **MCTP provides transport, while PLDM defines standardized management messages and data models.**

The OpenBMC PLDM architecture documentation describes PLDM as an application-layer communication protocol and MCTP as a common transport layer over multiple physical channels.

---

## 13. Multiple Communication Paths

There is no single universal host-BMC interface.

A platform may use different combinations of interfaces, transports and protocols.

A conceptual view is:

```text
                         HOST
                           |
        +------------------+------------------+
        |                  |                  |
       LPC                eSPI               PCIe
        |                  |                  |
       I²C                UART               GPIO
        |                  |                  |
        +------------------+------------------+
                           |
                           v
                          BMC
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
         Host IPMI       MCTP       Platform-Specific
                           |
                          PLDM
```

The actual combination depends on the platform.

This is why a specific server architecture must be understood from its hardware design and implementation details rather than from one universal diagram.

---

## 14. Example 1 — Host Sends an IPMI Command

Suppose software running on the host needs to communicate with the BMC using host IPMI.

A simplified flow is:

```text
Host Software
      |
      v
IPMI Command
      |
      v
KCS / BT
      |
      v
BMC
      |
      v
ipmid
      |
      v
IPMI Command Handler
```

The BMC receives the command, processes it and returns the appropriate response.

OpenBMC's IPMI architecture describes the flow from host-facing channels to the BMC-side IPMI daemon.

---

## 15. Example 2 — Administrator Accesses the Host Console

Now consider a different direction.

The administrator wants to access the host's serial console remotely.

A simplified flow is:

```text
Administrator
      |
      v
Management Network
      |
      v
     BMC
      |
      v
Host Serial Console
      |
      v
     UART
      |
      v
     HOST
```

Here, the BMC acts as the management bridge between the remote administrator and the host's console.

OpenBMC provides host-console infrastructure to support this management path.

---

## 16. Connecting Day 4 and Day 5

In Day 4, we learned about BMC hardware interfaces:

```text
BMC
 |
 +-- I²C / SMBus
 +-- SPI
 +-- GPIO
 +-- UART
 +-- Network
 +-- Host Interfaces
```

Today, we are asking:

> **Which communication paths can connect the host and BMC, and what higher-level protocols or data models can use those paths?**

A useful conceptual model is:

```text
        PHYSICAL / BUS INTERFACE
                    |
                    v
             TRANSPORT LAYER
                    |
                    v
       MANAGEMENT PROTOCOL / DATA MODEL
                    |
                    v
               BMC SOFTWARE
```

This is the bridge between the **hardware architecture** and the **software architecture** we have been building throughout the series.

---

## 17. Why This Matters for OpenBMC

Understanding host-BMC communication is important because many OpenBMC features depend on information flowing across this boundary.

Examples include:

- Host power/state management
- Host IPMI
- Host console
- PLDM communication
- Platform events
- Host firmware interactions

The broader architecture can be viewed as:

```text
               HOST
                 |
        Host-BMC Interfaces
                 |
                 v
                BMC
                 |
             OpenBMC
                 |
        Management Services
```

The details vary by platform, but the core idea remains:

> **The BMC provides a dedicated management path between the host platform and management software.**

---

## 18. Key Takeaways

After completing Day 5, you should understand:

### ✅ 1. The BMC and host are separate computing environments that may need to exchange management information.

### ✅ 2. Host-BMC communication can use different physical interfaces depending on the platform.

### ✅ 3. Common interfaces include LPC, eSPI, PCIe, I²C/SMBus, UART and GPIO.

### ✅ 4. UART, SPI and I²C are themselves standardized communication interfaces/protocols with defined communication rules.

### ✅ 5. In the OpenBMC architecture, we additionally distinguish physical interfaces from higher-level transport protocols and management data models.

### ✅ 6. Host IPMI can use host-facing channels such as KCS and BT.

### ✅ 7. MCTP provides a transport mechanism for management communication.

### ✅ 8. PLDM defines standardized management messages and data models that can operate over transports such as MCTP.

### ✅ 9. Host console access is another important BMC-host communication use case.

### ✅ 10. The exact communication architecture varies between server platforms.

### ✅ 11. A useful mental model is:

> **Physical / Bus Interface → Transport → Management Protocol / Data Model → BMC Software**

---

## What's Next?

Now we understand:

**What a BMC is → What it does → How OpenBMC is organized → What hardware it contains → How it communicates with the host**

The next question is:

> **How does a BMC actually start running all of this software?**

### Day 6 — BMC Boot Process

We will follow the BMC from:

**Power ON → Boot ROM → Bootloader → Linux Kernel → Root Filesystem → OpenBMC Services**

One concept at a time.

**OpenBMC Learning Series — From fundamentals to internals.**

---

## References

- OpenBMC Architecture — Interface Overview  
  https://github.com/openbmc/docs/blob/master/architecture/interface-overview.md

- OpenBMC IPMI Architecture  
  https://github.com/openbmc/docs/blob/master/architecture/ipmi-architecture.md

- OpenBMC PLDM Stack Design  
  https://github.com/openbmc/docs/blob/master/designs/pldm-stack.md
