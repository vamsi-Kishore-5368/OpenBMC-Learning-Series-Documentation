# OpenBMC Learning Series — Day 7

## Why Linux on the BMC?

### Understanding the Linux Foundation Behind OpenBMC

Welcome to **Day 7** of my OpenBMC Learning Series.

In Day 6, we followed the BMC boot process:

> **Reset → Boot ROM → Bootloader → Linux → RootFS → systemd → OpenBMC Services**

Today, we answer a fundamental question:

> **Why does a modern BMC need Linux?**

And once Linux is running:

> **What exactly does Linux provide to OpenBMC?**

---

## 1. Why Does a BMC Need an Operating System?

A BMC is not simply reading one sensor or toggling one GPIO.

A modern BMC may need to:

- Monitor temperatures, voltages, fans and other sensors
- Control power, reset and cooling
- Manage hardware inventory
- Provide network connectivity
- Support Redfish and IPMI
- Handle logs and events
- Communicate with many hardware devices
- Run multiple management services simultaneously
- Provide persistent configuration and filesystems
- Handle security and user management

As the functionality grows, implementing everything directly as one large firmware application becomes increasingly difficult.

This is where an operating system becomes useful.

---

## 2. What Are the Alternatives?

A BMC does not inherently require Linux.

Different embedded systems can use different approaches.

### Bare-metal

```text
BMC Hardware
     ↓
Bare-metal Firmware
     ↓
Management Functions
```

This can be small and efficient, but the firmware has to provide or implement many system-level capabilities itself.

### RTOS

```text
BMC Hardware
     ↓
RTOS
     ↓
Tasks / Drivers / Applications
```

An RTOS provides useful facilities such as scheduling, synchronization, timers and task management.

### Linux

```text
BMC Hardware
     ↓
Linux Kernel
     ↓
Drivers / Kernel Subsystems
     ↓
Linux Userspace
     ↓
OpenBMC Services
```

For a feature-rich server BMC, Linux provides a mature operating-system foundation on which OpenBMC can build.

> **Linux is not the only possible choice for a BMC. It is a particularly powerful foundation for modern, feature-rich BMC software such as OpenBMC.**

---

## 3. Why Linux Is a Good Fit for OpenBMC

Linux already provides many capabilities that a complex management controller needs.

### Process Management
Linux can run multiple independent processes.

### Memory Management
Linux manages RAM and provides memory-management facilities for the kernel and userspace processes.

### Networking
Linux provides mature networking infrastructure used by BMC management interfaces.

### Filesystems
Linux provides filesystem support for configuration, logs, runtime information and application data.

### Hardware Drivers
Linux provides established driver frameworks and subsystems for I²C, SPI, GPIO, UART, Ethernet and hardware monitoring.

### Security
Linux provides established mechanisms for users, permissions, processes and other security controls.

### Service Management
OpenBMC uses **systemd** to manage processes and services.

OpenBMC's documentation describes systemd units, services and targets as the framework used to control which processes are started.

---

## 4. OpenBMC Is Built on Linux

This distinction is important.

> **BMC = Management Hardware**

> **Linux = Operating-System Foundation**

> **OpenBMC = Management Software Stack**

```text
┌─────────────────────────────────────┐
│             OpenBMC                 │
│                                     │
│ Redfish • IPMI • Sensors • Power   │
│ Inventory • Logging • Configuration│
└──────────────────┬──────────────────┘
                   ↓
┌─────────────────────────────────────┐
│          Linux Userspace            │
│       Processes / systemd           │
└──────────────────┬──────────────────┘
                   ↓
┌─────────────────────────────────────┐
│            Linux Kernel             │
│ Drivers • Networking • Filesystems  │
│ I²C • SPI • GPIO • UART • etc.      │
└──────────────────┬──────────────────┘
                   ↓
┌─────────────────────────────────────┐
│           BMC Hardware              │
│         SoC / Memory / Flash        │
│       Sensors / Controllers         │
└─────────────────────────────────────┘
```

This is one of the most important mental models in the series.

---

## 5. What Does the Linux Kernel Actually Do?

The Linux kernel sits between OpenBMC userspace and the BMC hardware.

```text
OpenBMC Service
       ↓
Linux Kernel
       ↓
Linux Subsystem / Driver
       ↓
BMC Hardware
```

The kernel manages core system resources such as:

- CPU
- Memory
- Devices
- Networking
- Filesystems
- Processes
- Interrupts

OpenBMC maintains a Linux kernel tree for use by the project, with a development philosophy that generally prefers upstream kernel code and carries patches where necessary.

---

## 6. Processes and Threads

OpenBMC consists of many programs and services.

When a program is running, Linux represents it as a **process**.

```text
Linux
 ├── systemd
 ├── bmcweb
 ├── Sensor Service
 ├── Network Services
 ├── Logging Services
 └── Other OpenBMC Services
```

A process can contain one or more **threads**:

```text
Process
 ├── Thread 1
 ├── Thread 2
 └── Thread 3
```

We will explore Linux scheduling and concurrency in more detail later.

---

## 7. Memory

The BMC has physical RAM.

Once Linux starts, RAM is used by different parts of the running system.

```text
             BMC RAM
                │
      ┌─────────┼─────────┐
      ↓         ↓         ↓
   Kernel    Processes   Buffers
                         / Runtime Data
```

This connects directly with Day 4:

> **Flash stores persistent software/data. RAM holds the actively running system.**

The exact memory layout depends on the SoC, kernel configuration and platform.

---

## 8. The Linux Filesystem

Once OpenBMC is running, you are working inside a Linux filesystem.

```text
/
├── bin/
├── dev/
├── etc/
├── proc/
├── run/
├── sys/
├── tmp/
├── usr/
└── var/
```

Some particularly useful directories for OpenBMC work are:

| Directory | Purpose |
|---|---|
| `/etc` | Configuration |
| `/dev` | Device interfaces |
| `/proc` | Kernel and process information |
| `/sys` | Kernel/device information |
| `/run` | Runtime state |
| `/var` | Variable data and logs |

These directories become extremely useful when debugging an OpenBMC system.

---

## 9. Linux Drivers

One of the most important Linux concepts for BMC development is the **device driver**.

A driver provides software support for interacting with a hardware device or subsystem.

```text
OpenBMC Service
       ↓
Linux Subsystem
       ↓
Linux Driver
       ↓
Hardware
```

For example:

```text
Temperature Sensor
       ↓
      I²C
       ↓
Linux I²C Driver / Subsystem
       ↓
Linux Kernel
       ↓
OpenBMC Sensor Service
```

OpenBMC's sensor architecture documents the use of Linux hardware-monitoring interfaces and mapping sensor information into D-Bus objects.

The exact implementation can vary by hardware and OpenBMC sensor application.

---

## 10. Device Tree

On many embedded Linux platforms, the **Device Tree** describes hardware to the Linux kernel.

```text
Device Tree
     ↓
Describes hardware
     ↓
Linux Kernel
     ↓
Drivers / Devices
```

A BMC platform may describe:

- I²C buses
- I²C devices
- GPIOs
- UARTs
- Ethernet/MAC
- Other platform devices

OpenBMC's machine-development documentation specifically describes adding machine Device Tree information for GPIOs, I²C buses/devices, UARTs and MAC devices.

We will explore **Device Tree and DTS files** in much greater detail in a future post.

---

## 11. systemd

We introduced systemd in Day 6.

Now we can see where it fits:

```text
Linux Kernel
      ↓
PID 1
      ↓
systemd
      ↓
Targets / Units
      ↓
OpenBMC Services
```

OpenBMC uses systemd to manage its processes.

- **Unit** — the basic framework used by systemd.
- **Service** — a type of unit used to define a process or service to run.
- **Target** — a unit used to group services and define synchronization points.

OpenBMC's official documentation explains these concepts and shows that OpenBMC has its own systemd target and service units.

---

## 12. Linux → OpenBMC

Now we can connect everything.

```text
                  OpenBMC
                     ↓
              OpenBMC Services
                     ↓
                  systemd
                     ↓
              Linux Userspace
                     ↓
                Linux Kernel
                     ↓
           Drivers / Subsystems
                     ↓
                BMC Hardware
```

> **Linux provides the operating-system foundation. OpenBMC provides the management software built on top of that foundation.**

---

## 13. Practical Example — Reading a Temperature Sensor

Suppose the BMC needs to read a temperature sensor connected through I²C.

A simplified conceptual path is:

```text
Temperature Sensor
       ↓
      I²C
       ↓
Linux I²C Driver / Subsystem
       ↓
Linux Hardware Monitoring Interface
       ↓
OpenBMC Sensor Service
       ↓
D-Bus
       ↓
Redfish / Other Management Interface
       ↓
Administrator
```

OpenBMC's sensor architecture documents how hardware-monitoring sensor information is represented as D-Bus objects. The exact implementation can vary by hardware and OpenBMC sensor application.

---

## 14. Why This Architecture Is Powerful

Without an operating-system foundation, much more infrastructure would need to be implemented and maintained by the BMC software itself.

With Linux:

```text
             OpenBMC
                ↓
     Uses existing Linux
        infrastructure
                ↓
 ┌──────────────┼──────────────┐
 ↓              ↓              ↓
Drivers      Networking     Filesystems
 ↓              ↓              ↓
Hardware     Management     Persistent Data
```

OpenBMC can therefore focus more on **server management functionality** instead of reinventing the complete operating-system layer.

---

## 15. An Important Clarification

It is tempting to say:

> "BMCs use Linux because Linux is the best operating system."

That is too broad.

The more accurate statement is:

> **Modern, feature-rich BMCs can benefit greatly from Linux because it provides a mature kernel, driver ecosystem, networking, filesystems, process management, security mechanisms and service-management infrastructure.**

Other approaches can be valid depending on:

- Hardware capability
- Product requirements
- Real-time requirements
- Resource constraints
- Security requirements
- Software architecture

The choice is an engineering decision.

---

## 16. Day 7 Mental Model

```text
┌──────────────────────────────┐
│          OpenBMC             │
│                              │
│ Management Services          │
│ Redfish / IPMI / Sensors     │
│ Inventory / Power / Logging  │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│       Linux Userspace        │
│   Processes / systemd / IPC  │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│        Linux Kernel          │
│ Drivers / Networking / FS    │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│        BMC Hardware          │
│ SoC / Sensors / I²C / GPIO   │
└──────────────────────────────┘
```

> **Hardware → Linux → OpenBMC → Management**

---

## 17. Key Takeaways

### ✅ 1.
A BMC does not inherently require Linux; bare-metal and RTOS approaches are also possible.

### ✅ 2.
Linux is a strong foundation for feature-rich BMC management systems.

### ✅ 3.
Linux provides processes, memory management, networking, filesystems, drivers and other core infrastructure.

### ✅ 4.
OpenBMC is built on top of Linux rather than replacing Linux.

### ✅ 5.
Linux drivers and kernel subsystems provide the bridge between OpenBMC software and BMC hardware.

### ✅ 6.
Device Tree can describe the hardware configuration to Linux on supported embedded platforms.

### ✅ 7.
systemd manages OpenBMC processes and services.

### ✅ 8.
D-Bus provides an important communication layer between OpenBMC services.

### ✅ 9.
A real sensor path can span:

> **Hardware → Linux driver/subsystem → OpenBMC service → D-Bus → Management interface**


## Official OpenBMC References

- OpenBMC & systemd  
  https://github.com/openbmc/docs/blob/master/architecture/openbmc-systemd.md
- OpenBMC Kernel Development  
  https://github.com/openbmc/docs/blob/master/kernel-development.md
- OpenBMC Sensor Architecture  
  https://github.com/openbmc/docs/blob/master/architecture/sensor-architecture.md
- Adding a New OpenBMC System  
  https://github.com/openbmc/docs/blob/master/development/add-new-system.md
- OpenBMC Architecture / Interface Overview  
  https://github.com/openbmc/docs/blob/master/architecture/interface-overview.md
