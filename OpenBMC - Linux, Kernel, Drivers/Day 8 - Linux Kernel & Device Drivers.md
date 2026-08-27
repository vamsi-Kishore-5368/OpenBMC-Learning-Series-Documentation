# OpenBMC Learning Series — Day 8

## Linux Kernel & Device Drivers

### How Does OpenBMC Actually Talk to Hardware?

Welcome to **Day 8** of the OpenBMC Learning Series.

In Day 7, we understood why Linux is a strong foundation for a modern BMC and where OpenBMC sits in the software stack.

Today, we go one level deeper:

> **How does OpenBMC actually communicate with the hardware?**

The key concept is:

> **The Linux kernel and its device drivers provide the bridge between OpenBMC software and BMC hardware.**

---

## 1. The Big Picture

```text
OpenBMC
   ↓
Linux Userspace
   ↓
Linux Kernel
   ↓
Kernel Subsystem / Driver
   ↓
BMC Hardware
```

A useful mental model is:

> **OpenBMC software → Linux → Driver / Subsystem → Hardware**

---

## 2. What Exactly Is the Linux Kernel?

The **Linux kernel** is the core component of the Linux operating system.

It manages and provides controlled access to resources such as:

- CPU
- Memory
- Hardware devices
- Processes
- Networking
- Filesystems
- Interrupts

For a BMC, the kernel provides the infrastructure needed for OpenBMC applications and services to interact with the underlying hardware.

```text
OpenBMC Applications
          ↓
     Linux Kernel
          ↓
       Hardware
```

---

## 3. Kernel Space vs Userspace

Linux separates normal applications from the privileged kernel environment.

### Userspace

This is where OpenBMC services and other applications run.

Examples:

- OpenBMC management services
- Redfish components
- Sensor applications
- Network services

### Kernel Space

This is where the Linux kernel and kernel drivers operate.

```text
┌─────────────────────────────┐
│          USERSPACE          │
│ OpenBMC Services / Apps     │
└──────────────┬──────────────┘
               │
        Kernel Interface
               ↓
┌─────────────────────────────┐
│         KERNEL SPACE        │
│ Kernel / Drivers / Subsys.  │
└──────────────┬──────────────┘
               ↓
          BMC Hardware
```

Applications generally do not access hardware registers directly. They use interfaces provided by the kernel and its subsystems/drivers.

---

## 4. What Is a Device Driver?

A **device driver** is kernel software that provides support for interacting with a particular device or class of hardware.

Think of it as a software bridge:

```text
Application
     ↓
Kernel Interface
     ↓
Device Driver
     ↓
Hardware
```

For BMC development, drivers can be involved with:

- I²C devices
- GPIO controllers
- SPI devices
- UARTs
- Ethernet controllers
- Hardware-monitoring devices
- Other platform peripherals

---

## 5. Drivers vs Linux Subsystems

These terms are related but not identical.

A **Linux subsystem** provides a common framework for a class of devices.

A **driver** implements support for a particular device or hardware implementation within that framework.

For example:

```text
                 Linux Kernel
                      │
                      ↓
                I²C Subsystem
                      │
             ┌────────┴────────┐
             ↓                 ↓
       I²C Controller       I²C Device
           Driver             Driver
             │                 │
             └────────┬────────┘
                      ↓
                   Hardware
```

The same general idea applies to GPIO, SPI, networking and hardware-monitoring subsystems.

---

## 6. `/dev`, `/sys` and `/proc`

When working on a running OpenBMC system, these Linux virtual interfaces are particularly useful.

### `/dev`

Contains device nodes and interfaces exposed to userspace.

### `/sys`

Exposes information about devices, drivers, kernel objects and system configuration through **sysfs**.

For BMC development, `/sys` is especially useful when investigating how Linux represents hardware.

```text
Hardware
   ↓
Linux Kernel
   ↓
sysfs
   ↓
/sys/...
```

### `/proc`

A virtual filesystem exposing information about processes and various kernel/system state.

Examples:

```text
/proc/cpuinfo
/proc/meminfo
/proc/interrupts
```

These are useful during system investigation and debugging.

---

## 7. Device Tree

On many embedded Linux platforms, the **Device Tree** is used to describe hardware to the Linux kernel.

It can describe:

- I²C controllers and devices
- GPIO controllers
- UARTs
- SPI devices
- Ethernet/MAC devices
- Other platform hardware

```text
Device Tree
     ↓
Describes Hardware
     ↓
Linux Kernel
     ↓
Driver / Subsystem
     ↓
Hardware
```

For example:

```text
I²C Controller
      │
      └── Temperature Sensor
```

> **Device Tree describes the hardware to Linux; it does not itself act as the hardware driver.**

We will study DTS/DTSI files and Device Tree bindings in much greater detail later.

---

## 8. Linux Subsystems Important to BMCs

A BMC contains many different kinds of peripherals.

A simplified picture is:

```text
                       Linux Kernel
                            │
       ┌────────────┬───────┼────────┬───────────┐
       ↓            ↓       ↓        ↓           ↓
      I²C          GPIO     SPI     UART      Ethernet
       ↓            ↓       ↓        ↓           ↓
    Sensors      Signals  Devices  Console     Network
```

The important idea is:

> **OpenBMC services can use Linux kernel interfaces and subsystems instead of implementing every low-level hardware detail themselves.**

---

## 9. Practical Example — Temperature Sensor

Suppose a temperature sensor is connected to the BMC through I²C.

A simplified path can look like:

```text
Temperature Sensor
        ↓
       I²C
        ↓
Linux I²C Subsystem / Driver
        ↓
Linux Kernel Interface
        ↓
OpenBMC Sensor Service
        ↓
      D-Bus
        ↓
     Redfish
        ↓
   Administrator
```

At a high level:

1. The physical sensor measures temperature.
2. The sensor communicates over I²C.
3. Linux provides I²C subsystem/driver support.
4. An OpenBMC sensor component obtains the information.
5. D-Bus allows OpenBMC components to exchange the sensor data.
6. Redfish can expose the information to a remote administrator.

The exact components and path can vary by hardware and OpenBMC implementation.

---

## 10. Another Example — GPIO

Consider a GPIO used for a platform control signal.

```text
OpenBMC Service
       ↓
Linux GPIO Interface
       ↓
GPIO Driver / Subsystem
       ↓
BMC GPIO Controller
       ↓
Physical GPIO Pin
       ↓
Platform Hardware
```

The OpenBMC service does not need to directly manipulate the GPIO controller's hardware registers.

Linux provides the kernel-side abstraction.

---

## 11. How Does OpenBMC Reach Hardware?

Now we can combine the concepts from previous days.

```text
Administrator
      ↓
Redfish / IPMI
      ↓
OpenBMC Service
      ↓
D-Bus
      ↓
Linux Interface
      ↓
Kernel Subsystem
      ↓
Driver
      ↓
Hardware
```

This is not one universal path for every operation. Different devices and OpenBMC components can use different Linux interfaces and kernel subsystems.

The important idea is the **layered architecture**.

---

## 12. Why Don't OpenBMC Applications Directly Access Hardware?

Direct hardware access from every application would make the system difficult to maintain.

Instead:

```text
             Applications
          ┌──────┼──────┐
          ↓      ↓      ↓
       Service Service Service
          └──────┼──────┘
                 ↓
          Linux Interfaces
                 ↓
       Kernel Subsystems/Drivers
                 ↓
              Hardware
```

This creates a cleaner separation of responsibilities and allows hardware-specific logic to remain closer to the kernel/driver layer.

---

## 13. Kernel Modules — A First Introduction

You will often hear the term **kernel module**.

A kernel module is code that can be loaded into the Linux kernel to extend its functionality.

```text
Linux Kernel
     ↑
Kernel Module
     ↑
Driver / Functionality
```

Not every driver is necessarily a separately loadable module; some drivers can be built directly into the kernel.

So remember:

> **Driver = hardware-support functionality.**

> **Module = one way kernel code can be packaged and loaded.**

We will explore kernel configuration and modules separately later.

---

## 14. Interrupts

Hardware sometimes needs to notify the CPU that an event has occurred.

This can happen through an **interrupt**.

```text
Hardware Event
      ↓
   Interrupt
      ↓
Linux Kernel
      ↓
Driver / Handler
      ↓
Relevant Software
```

Interrupts are another important Linux concept that becomes useful when studying device drivers.

---

## 15. Connecting Day 7 and Day 8

Day 7 answered:

> **Why Linux?**

Day 8 answers:

> **How does Linux help OpenBMC communicate with hardware?**

Together:

```text
                 OpenBMC
                    ↓
          Linux Userspace
                    ↓
               systemd
                    ↓
             Linux Kernel
                    ↓
        Kernel Subsystems / Drivers
                    ↓
              BMC Hardware
```

> **Linux provides the operating-system and hardware-access foundation. OpenBMC provides the management software built on top of it.**

---

## 16. Day 8 Mental Model

```text
┌─────────────────────────────┐
│         OpenBMC             │
│      Management Logic       │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│       Linux Userspace       │
│      Processes / Services   │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│        Linux Kernel         │
│   Subsystems + Drivers      │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│        BMC Hardware         │
│ I²C / GPIO / SPI / UART ... │
└─────────────────────────────┘
```

> **OpenBMC provides the management logic. Linux provides the operating-system and hardware-access foundation.**

---

## 17. Key Takeaways

### ✅ 1.
The Linux kernel is the core of the Linux operating system.

### ✅ 2.
OpenBMC services normally run in userspace.

### ✅ 3.
Linux drivers and subsystems provide kernel-side support for hardware.

### ✅ 4.
`/dev`, `/sys` and `/proc` are useful when inspecting and debugging a running Linux/OpenBMC system.

### ✅ 5.
Device Tree can describe embedded hardware to the Linux kernel.

### ✅ 6.
A driver is not the same thing as a subsystem.

### ✅ 7.
A kernel module is a way of packaging/loading kernel code; not every driver has to be a separately loadable module.

### ✅ 8.
Different BMC peripherals use different Linux subsystems.

### ✅ 9.
A simplified sensor path can be:

> **Hardware → Linux → OpenBMC Service → D-Bus → Redfish**

### ✅ 10.
OpenBMC is layered on top of Linux rather than directly replacing the kernel.

---

## What's Next?

We now understand:

> **Why Linux is used on a BMC → what the kernel does → what drivers do → how Linux represents hardware → how OpenBMC reaches hardware.**

### Day 9 — Deep Dive into Linux Device Tree

We will explore:

- What is a Device Tree?
- DTS vs DTSI
- Nodes and properties
- `compatible`
- `reg`
- `interrupts`
- GPIO properties
- I²C device descriptions
- How a Device Tree node connects to a Linux driver
- How this appears in an OpenBMC machine

One concept at a time. 🚀

---

## References

- OpenBMC Documentation  
  https://github.com/openbmc/docs

- OpenBMC Kernel Development  
  https://github.com/openbmc/docs/blob/master/kernel-development.md

- OpenBMC Systemd Architecture  
  https://github.com/openbmc/docs/blob/master/architecture/openbmc-systemd.md

- Linux Kernel Documentation  
  https://docs.kernel.org/

- Devicetree Specification  
  https://www.devicetree.org/specifications/

- Linux I²C Documentation  
  https://docs.kernel.org/i2c/

- Linux GPIO Documentation  
  https://docs.kernel.org/driver-api/gpio/

- Linux SPI Documentation  
  https://docs.kernel.org/spi/

- Linux sysfs Documentation  
  https://docs.kernel.org/filesystems/sysfs.html
