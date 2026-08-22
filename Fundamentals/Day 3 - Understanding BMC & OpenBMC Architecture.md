# OpenBMC Learning Series — Day 3

## Understanding BMC & OpenBMC Architecture

### From BMC Hardware to OpenBMC Services

Welcome to **Day 3** of my OpenBMC Learning Series.

In **Day 1**, we introduced the basic server architecture and understood the relationship between the **Host System, BMC, and OpenBMC**.

In **Day 2**, we explored what a BMC actually does:

> **Monitor → Control → Manage**

Now we know *what* the BMC does.

The next question is:

> **How is the BMC/OpenBMC system organized to perform all these functions?**

This is where we introduce **BMC & OpenBMC architecture**.

---

## 1. The Big Picture

A BMC is a dedicated management computer embedded in a server platform.

It has its own hardware, and software runs on that hardware to provide management functionality.

### OpenBMC Architecture at a Glance

![OpenBMC Architecture — Day 3](images/day3-openbmc-architecture.png)

*Figure 1 — A simplified, conceptual view of the BMC/OpenBMC software stack and its relationship with BMC hardware and management clients.*

A simplified architecture looks like this:

```text
               BMC PLATFORM
                    │
                    ▼
               BMC HARDWARE
                    │
                    ▼
               BOOTLOADER
                    │
                    ▼
          LINUX KERNEL + DRIVERS
                    │
                    ▼
                 SYSTEMD
                    │
                    ▼
                  D-BUS
                    │
                    ▼
               OPENBMC SERVICES
                    │
     ┌──────────────├──────────────┐
     ▼              ▼              ▼
  Sensors       Inventory       Control
     │              │              │
     └──────────────┼──────────────┘
                    ▼
       MANAGEMENT INTERFACES
              ┌─────┴─────┐
              ▼           ▼
           Redfish      IPMI
```

This is a **conceptual architecture**. Actual BMC hardware, interfaces, services, and software components vary between platforms.

---

## 2. BMC Hardware vs OpenBMC Software

One of the most important concepts to understand is:

> **BMC and OpenBMC are not the same thing.**

### BMC

The **BMC** is the management controller/hardware platform.

It contains or connects to resources such as:

- Processor / SoC
- RAM
- Flash
- Network interface
- GPIO
- I²C / SMBus
- SPI
- UART
- Host/platform interfaces

### OpenBMC

**OpenBMC is an open-source, Linux-based software stack/firmware project for BMC platforms.**

It provides the software components required to turn the BMC hardware into a functional management platform.

A simplified view:

```text
                BMC
                 │
        ┌────────┴────────┐
        │                 │
     HARDWARE           SOFTWARE
        │                 │
     BMC SoC           OpenBMC
     RAM               Linux
     Flash             Services
     Network           D-Bus
     Interfaces        Management APIs
```

Therefore:

> **BMC = Management Hardware**
>
> **OpenBMC = Software Stack running on the BMC**

---

## 3. BMC Hardware

We are not going deeply into BMC hardware today—that will be covered in **Day 4**.

For now, think of the BMC as an embedded computer.

```text
                  BMC
        ┌─────────────────────┐
        │                     │
        │      BMC SoC        │
        │                     │
        │    Processor        │
        │    RAM              │
        │    Flash            │
        │    Network          │
        │                     │
        │    I²C / SPI        │
        │    GPIO / UART      │
        │    Host Interfaces  │
        │                     │
        └─────────────────────┘
```

The BMC hardware provides the physical resources required by the software.

---

## 4. Bootloader

Before OpenBMC's Linux environment can run, the BMC needs to boot.

A simplified boot sequence is:

```text
BMC Power-On
     │
     ▼
Bootloader
     │
     ▼
Linux Kernel
     │
     ▼
OpenBMC Userspace
```

The **bootloader** is responsible for performing early platform initialization and loading/starting the next stage of the system.

A common bootloader used in embedded Linux systems is **U-Boot**, although the exact boot architecture depends on the platform.

We don't need to study bootloader internals yet.

For now, remember:

> **Bootloader → starts the Linux-based BMC environment.**

---

## 5. Linux Kernel and Drivers

Once the bootloader starts the Linux environment, the **Linux kernel** becomes the core of the running operating system.

The kernel communicates with the underlying hardware through drivers.

Conceptually:

```text
OpenBMC Service
       │
       ▼
Linux Kernel
       │
       ▼
Driver / Kernel Subsystem
       │
       ▼
Hardware
```

For example, a hardware device connected through an I²C bus may be accessed through the Linux kernel's I²C subsystem and appropriate driver support.

This creates an important separation:

```text
Application / Service
        │
        ▼
Kernel Interface
        │
        ▼
Driver / Subsystem
        │
        ▼
Hardware
```

We will explore Linux drivers and hardware access in much greater detail later.

---

## 6. systemd

OpenBMC runs on Linux, so it needs mechanisms for starting and managing system services.

This is where **systemd** becomes important.

A simplified representation is:

```text
                    systemd
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   Sensor Service  Network Service  Inventory
        │              │              │
        ▼              ▼              ▼
     Hardware       Network         Platform
```

systemd acts as the system and service manager.

It can:

- Start services
- Stop services
- Restart services
- Manage service dependencies
- Coordinate system startup

This becomes particularly important because OpenBMC consists of many different services rather than one giant application.

---

## 7. D-Bus — Communication Between Services

Now we reach one of the most important concepts in OpenBMC:

# **D-Bus**

OpenBMC contains many separate software components.

These components need to communicate with each other.

D-Bus provides a **message-based inter-process communication (IPC) mechanism** for communication between processes/services.

A simplified example:

```text
Sensor Service
      │
      ▼
    D-Bus
      │
      ▼
Redfish Service
```

For example, a sensor-related service can expose hardware information through D-Bus, and another service can consume that information.

This gives us a very important mental model:

> **D-Bus acts as a communication layer between many OpenBMC services.**

We will dedicate an entire future post to D-Bus because it is one of the most important concepts for understanding OpenBMC internally.

---

## 8. OpenBMC Services

Above the core Linux/system infrastructure, OpenBMC provides numerous software components and services that implement platform-management functionality.

Conceptually:

```text
                    OpenBMC
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     Sensors        Inventory       Control
        │              │              │
        ▼              ▼              ▼
     Hardware        Platform        Chassis
      Health         Information     Management
```

Examples of functionality include:

### Sensor Management

Provides information about things such as:

- Temperature
- Fan speed
- Voltage
- Power

### Inventory Management

Maintains information about platform components and their properties.

### Chassis / Power Management

Handles platform-management operations such as power and reset, according to the platform's implementation.

### Firmware Management

Provides functionality related to firmware update and management.

### Configuration

Handles platform-specific configuration and management data.

The exact services and implementation differ between OpenBMC platforms.

---

## 9. Management Interfaces

Now we reach the upper layer.

How does an administrator or management application communicate with the BMC?

Through management interfaces.

Two important examples are:

### Redfish

A modern, HTTP/REST-based management interface used for server and data-center management.

### IPMI

A traditional platform-management interface that remains widely used in server environments.

Conceptually:

```text
                 OpenBMC
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
       Redfish               IPMI
          │                   │
          └─────────┬─────────┘
                    ▼
             Management Client
```

These interfaces provide ways for external management software to interact with the BMC.

---

## 10. Putting Everything Together

Now let's combine all the layers.

```text
                    USER / ADMINISTRATOR
                            │
                            ▼
                  MANAGEMENT INTERFACES
                     ┌───────┴───────┐
                     ▼               ▼
                  Redfish           IPMI
                     │               │
                     └───────┬───────┘
                             ▼
                     OPENBMC SERVICES
               ┌─────────────┼─────────────┐
               ▼             ▼             ▼
            Sensors       Inventory      Control
               │             │             │
               └─────────────┼─────────────┘
                             ▼
                           D-Bus
                             │
                          systemd
                             │
                             ▼
                   Linux Kernel + Drivers
                             │
                             ▼
                         Bootloader
                             │
                             ▼
                        BMC HARDWARE
                   ┌───────────┴───────────┐
                   │ BMC SoC / RAM / Flash │
                   │ Network / I²C / GPIO  │
                   │ SPI / UART / etc.     │
                   └───────────────────────┘
```

This is the architecture we should keep in mind as we continue learning OpenBMC.

---

## 11. A Small Real-World Example

Let's take a simple example:

> **An administrator wants to check the server's temperature.**

The information travels conceptually through several layers.

```text
Temperature Sensor
        │
        ▼
Linux Hardware Interface / Driver
        │
        ▼
Sensor-related OpenBMC Service
        │
        ▼
      D-Bus
        │
        ▼
Management Interface
     (e.g. Redfish)
        │
        ▼
Administrator
```

The important thing is not memorizing every component.

The important thing is understanding the **layered architecture**.

The hardware provides the data.

Linux provides hardware access.

OpenBMC services process/expose management information.

D-Bus allows services to communicate.

Management interfaces make the information available to external clients.

---

## 12. Why This Architecture Matters

Why not simply create one program that talks directly to every sensor and hardware device?

Because a server-management system is complex.

There can be:

- Many sensors
- Many hardware devices
- Multiple management functions
- Multiple external interfaces
- Different platform implementations

Separating functionality into components makes the system easier to:

- Develop
- Maintain
- Debug
- Extend
- Replace
- Adapt to different platforms

This modular architecture is one of the important characteristics of OpenBMC.

---

## 13. A Simple Mental Model

If the complete architecture feels overwhelming, remember this:

```text
        HARDWARE
            │
            ▼
       BOOTLOADER
            │
            ▼
          LINUX
            │
            ▼
         SYSTEMD
            │
            ▼
          D-BUS
            │
            ▼
     OPENBMC SERVICES
            │
            ▼
  MANAGEMENT INTERFACES
            │
            ▼
      ADMINISTRATOR
```

Or even more simply:

> **Hardware → Linux → Services → Communication → Management**

---

## 14. BMC vs OpenBMC — Final Reminder

This distinction should become second nature.

### BMC

The dedicated management controller/platform.

### OpenBMC

The open-source Linux-based software stack used to provide BMC management functionality.

Therefore:

```text
              BMC PLATFORM
                   │
          ┌────────┴────────┐
          │                 │
       Hardware          Software
                            │
                            ▼
                         OpenBMC
                            │
                   ┌────────┼────────┐
                   ▼        ▼        ▼
                 Linux    D-Bus   Services
```

---

## 15. Key Takeaways

After Day 3, you should understand:

### ✅ 1. A BMC is a dedicated embedded management platform.

### ✅ 2. OpenBMC is the software stack that runs on supported BMC hardware.

### ✅ 3. The BMC starts through a boot process involving a bootloader and Linux.

### ✅ 4. Linux provides the kernel and hardware-driver layer.

### ✅ 5. systemd manages many of the services running on the BMC.

### ✅ 6. D-Bus provides communication between many OpenBMC processes/services.

### ✅ 7. OpenBMC contains multiple management components rather than one monolithic application.

### ✅ 8. Interfaces such as Redfish and IPMI allow external management clients to interact with the BMC.

### ✅ 9. The architecture can be viewed as:

> **Hardware → Bootloader → Linux → systemd/D-Bus → OpenBMC Services → Management Interfaces**

---

## What's Next?

Now that we understand the **overall BMC and OpenBMC architecture**, we can go one level deeper.

### Day 4 — BMC Hardware Architecture

**What's actually inside a BMC?**

We will explore:

- BMC SoC
- Processor
- RAM
- Flash
- Ethernet
- I²C / SMBus
- SPI
- GPIO
- UART
- Host interfaces
- Sensors and peripheral devices

The goal will be to connect the **physical BMC hardware** with the software architecture we learned today.

---

**OpenBMC Learning Series — One concept at a time.**
