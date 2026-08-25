# OpenBMC Learning Series — Day 6

## BMC Boot Process

### From Power-On to a Running OpenBMC System

Welcome to **Day 6** of my OpenBMC Learning Series.

In Day 1, we introduced the **Server Architecture** and understood where the BMC fits into a server.

In Day 2, we explored what a BMC actually does:

> **Monitor → Control → Manage**

In Day 3, we explored the overall **BMC & OpenBMC architecture**.

In Day 4, we looked at the **physical hardware of the BMC**.

In Day 5, we explored **BMC ↔ Host communication**.

Now we are going one level deeper:

> **What happens when the BMC receives power and starts running?**

The goal of Day 6 is to follow the BMC from **reset and early boot** all the way to a fully running **OpenBMC userspace**.

---

## 1. The Big Picture

A simplified BMC boot flow can be viewed as:

```text
BMC Power ON
      |
      v
Reset / Reset Vector*
      |
      v
Boot ROM
      |
      v
Bootloader Stages*
      |
      v
U-Boot / Bootloader
      |
      v
Linux Kernel
      |
      v
Initramfs / Root Filesystem*
      |
      v
PID 1 / systemd
      |
      v
OpenBMC Services
      |
      v
BMC Ready
```

`*` = platform-dependent.

This is a **conceptual boot flow**, not a universal sequence for every BMC.

OpenBMC's documented architecture uses **U-Boot** for system initialization and bootstrap. It describes the bootloader loading the kernel, initrd and device tree before transferring control to Linux.

---

## 2. BMC Boot Is Separate From Host Boot

The BMC and host are separate computing environments.

Therefore, they have separate boot processes.

A simplified comparison is:

```text
          HOST                         BMC
            |                           |
      Host Firmware                Reset / ROM
            |                           |
        Host OS                    Bootloader
            |                           |
      Applications                 Linux Kernel
                                      |
                               OpenBMC Services
```

The exact boot order and firmware stages depend on the platform.

This is an important continuation of the idea we introduced earlier:

> **The BMC is an independent management computer.**

---

## 3. Reset and the Reset Vector

When the BMC processor is released from reset, it begins execution according to the processor/SoC's reset mechanism.

The **reset vector** is the processor-defined entry point used for the earliest instruction execution after reset.

A simplified model is:

```text
Power / Reset
      |
      v
Reset Vector
      |
      v
Early Boot Code
      |
      v
Boot ROM
```

The exact reset-vector address and early execution mechanism depend on the BMC SoC architecture.

For this series, the important concept is:

> **The processor needs a defined starting point from which the earliest boot code can begin executing.**

---

## 4. Boot ROM

The **Boot ROM** is code built into the BMC SoC.

It executes during the earliest stages of startup and helps establish the initial boot process.

A simplified flow is:

```text
Reset
  |
  v
Boot ROM
  |
  v
Next Boot Stage
```

The Boot ROM is typically responsible for early initialization and locating or loading the next boot stage according to the SoC/platform design.

At this stage, we do not need to study the implementation details of a particular SoC.

---

## 5. Primary and Secondary Bootloader Stages

Embedded systems can use more than one bootloader stage.

For example, a platform may have a chain such as:

```text
Boot ROM
   |
   v
First-Stage / Primary Bootloader*
   |
   v
Second-Stage / Full Bootloader*
   |
   v
Linux Kernel
```

The exact terminology varies between platforms.

A first-stage loader may perform minimal initialization and load a larger second-stage bootloader.

However:

> **Not every OpenBMC platform uses exactly this two-stage bootloader structure.**

Therefore, we should treat **primary/secondary bootloader stages as a generic embedded-systems concept**, not as a mandatory OpenBMC sequence.

---

## 6. U-Boot / Main Bootloader

OpenBMC uses **Das U-Boot** as the bootloader in its documented boot architecture.

The bootloader performs important early tasks such as:

- System initialization
- Selecting the boot image
- Loading the Linux kernel
- Loading supporting boot data
- Passing boot information to Linux
- Starting the kernel

OpenBMC's flash-layout documentation describes U-Boot loading the compressed kernel, initrd and device tree into memory and then transferring control to the Linux kernel.

A simplified view is:

```text
                U-Boot
                   |
        +----------+----------+
        |          |          |
        v          v          v
      Kernel     Initrd      Device Tree
        \          |          /
         \         |         /
          +--------+--------+
                   |
                   v
              Linux Kernel
```

The exact image format and storage layout depend on the platform.

---

## 7. Where Does the Boot Software Come From?

In Day 4, we learned that the BMC has **non-volatile storage such as flash**.

Now we can connect that concept to the boot process.

A simplified representation is:

```text
                 BMC FLASH
                     |
        +------------+------------+
        |            |            |
        v            v            v
   Bootloader      Kernel       RootFS
                                  |
                             Initramfs*
```

OpenBMC documents multiple filesystem and storage layouts. For example, a documented BMC image can contain U-Boot, kernel, read-only filesystem and read-write filesystem components, while the exact layout depends on the platform and storage configuration.

Therefore:

> **Do not assume that every BMC stores or arranges its boot components in exactly the same way.**

---

## 8. Linux Kernel

After the bootloader has loaded the required components, it transfers control to the **Linux kernel**.

```text
Bootloader
     |
     v
Linux Kernel
```

The kernel then begins initializing the operating system environment.

This includes initialization related to:

- Processor resources
- Memory
- Devices
- Drivers
- Filesystems
- Networking
- Kernel subsystems

The kernel also receives platform information from boot-time data such as the **device tree**, where applicable.

The important transition is:

> **Bootloader prepares the system and starts Linux; Linux then takes control of the running system.**

---

## 9. Device Tree

Many embedded Linux platforms use a **Device Tree** to describe hardware to the Linux kernel.

A simplified view is:

```text
Bootloader
    |
    +---- Linux Kernel
    |
    +---- Device Tree
```

The device tree can describe information about hardware such as:

- CPUs
- Memory
- Buses
- Devices
- Interrupts
- GPIOs
- Other platform resources

OpenBMC's documented boot flow specifically describes the bootloader passing the device tree to the kernel along with other boot information.

We will dedicate a future post to **Device Tree in OpenBMC and Embedded Linux**.

---

## 10. Initramfs and Root Filesystem

Starting the kernel is not the end of the boot process.

Linux also needs a userspace environment.

Depending on the platform and image layout, this may involve:

- An **initramfs**
- A persistent **root filesystem**
- Or a combination of both

A simplified model is:

```text
Linux Kernel
     |
     v
Initramfs / RootFS
     |
     v
Userspace
```

OpenBMC's documented filesystem layouts show that the exact initialization process varies. In some configurations, an initramfs locates and mounts the root filesystem and writable filesystem layers before starting systemd; other platform implementations may use different arrangements.

Therefore:

> **Root filesystem and initramfs are important parts of the BMC boot process, but their exact role depends on the platform configuration.**

---

## 11. PID 1 and the Init System

Once Linux userspace begins, the system needs a process responsible for bringing up the rest of userspace.

This process becomes **PID 1**.

A simplified historical model is:

```text
Linux Kernel
     |
     v
PID 1
     |
     v
Services
```

Traditional Linux systems may use mechanisms such as:

- SysVinit
- BusyBox init

These can use initialization scripts such as:

```text
/etc/init.d/
```

The exact initialization manager is a distribution/system choice.

---

## 12. systemd in OpenBMC

OpenBMC uses **systemd** to manage its processes and services.

The OpenBMC systemd documentation describes:

- **Units** as the basic framework used by systemd
- **Services** as units that define processes to run
- **Targets** as synchronization points and groups of services to start

On an OpenBMC system, `default.target` is used during initial power-up unless an alternate target is specified, and the target relationships bring up the appropriate services.

A simplified flow is:

```text
Linux Kernel
     |
     v
PID 1
     |
  systemd
     |
     +-------------------+
     |                   |
     v                   v
 Targets              Services
     |                   |
     +---------+---------+
               |
               v
        OpenBMC Userspace
```

---

## 13. What About Init Scripts?

It is useful to understand the distinction.

Traditional init systems can use scripts such as:

```text
/etc/init.d/<service>
```

With systemd, service management is primarily represented through **units and targets**.

For example:

```text
Traditional Init:

Kernel
  |
  v
init
  |
  v
/etc/init.d/*
  |
  v
Services
```

Compared with:

```text
OpenBMC:

Kernel
  |
  v
systemd
  |
  v
Units / Targets
  |
  v
OpenBMC Services
```

The Yocto documentation describes this distinction between SysVinit/BusyBox init scripts and systemd units/targets.

Some OpenBMC platform-specific initialization scripts can still exist for particular early filesystem or boot tasks, so **init scripts are relevant**, but they should not be confused with the primary OpenBMC service-management model.

---

## 14. OpenBMC Services Start

Once systemd has established the userspace environment, the OpenBMC services required by the platform are started.

These can include services related to:

- Sensors
- Inventory
- Chassis and power management
- Networking
- Logging
- Redfish
- IPMI
- Configuration
- Firmware management
- Other platform-specific functionality

A simplified view is:

```text
                 systemd
                    |
        +-----------+-----------+
        |           |           |
        v           v           v
     Sensors     Network     Inventory
        |           |           |
        +-----------+-----------+
                    |
                    v
              BMC Ready
```

The exact services and dependency relationships depend on the OpenBMC image and platform.

---

## 15. Complete BMC Boot Flow

Now we can combine everything we have learned.

```text
                         BMC POWER ON
                              |
                              v
                       RESET / RESET VECTOR
                              |
                              v
                           BOOT ROM
                              |
                              v
                    BOOTLOADER STAGES*
                              |
                              v
                         U-BOOT
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
           Kernel          Initrd        Device Tree
              \               |               /
               \              |              /
                +-------------+-------------+
                              |
                              v
                        LINUX KERNEL
                              |
                              v
                    INITRAMFS / ROOTFS*
                              |
                              v
                           PID 1
                              |
                              v
                           systemd
                              |
                  +-----------+-----------+
                  |           |           |
                  v           v           v
               D-Bus      Network     OpenBMC
                                      Services
                              |
                              v
                         BMC READY
```

`*` = platform-dependent.

This should be our main mental model for the BMC boot process.

---

## 16. Small Real-World Example

Imagine a server is powered on.

The BMC has its own standby/management power domain on this platform, so it can begin booting independently of the host operating system.

A simplified sequence is:

```text
BMC Power Applied
        |
        v
Reset / Boot ROM
        |
        v
Bootloader
        |
        v
Linux Kernel
        |
        v
RootFS / Initramfs
        |
        v
systemd
        |
        v
OpenBMC Services
        |
        v
Management Interfaces Available
```

At this point, the BMC can provide its configured management functionality even though the host may not yet have completed its own boot process.

The exact power behavior is platform-dependent.

---

## 17. What Happens If the Host Is OFF?

One of the important ideas from Day 2 was that the BMC can provide a management path independent of the host OS.

The boot architecture makes that possible on platforms where the BMC has the required power and hardware support.

Conceptually:

```text
             SERVER
        +------------------+
        |                  |
        |    HOST = OFF    |
        |                  |
        |    BMC = ON      |
        |       |          |
        |       v          |
        |    OpenBMC       |
        |       |          |
        |       v          |
        |  Management      |
        +------------------+
```

The BMC can therefore boot its own software environment and provide management functionality independently of the host operating system.

---

## 18. What If Boot Fails?

The layered boot process also gives us a useful debugging model.

If the BMC does not become available, we can ask:

```text
Did reset complete?
       |
       v
Did Boot ROM start?
       |
       v
Did the bootloader run?
       |
       v
Did the bootloader load the kernel?
       |
       v
Did Linux start?
       |
       v
Did the root filesystem mount?
       |
       v
Did PID 1 / systemd start?
       |
       v
Did the required OpenBMC services start?
```

This gives us a practical way to think about BMC boot failures.

Later in the series, we can turn this into a complete **BMC boot-debugging workflow**.

---

## 19. Where Does Yocto Fit?

We have not discussed Yocto in detail yet, but it is useful to place it in the overall picture.

Eventually, we will learn how the OpenBMC build system creates the software image that gets deployed to the BMC.

Conceptually:

```text
Yocto / BitBake
       |
       v
OpenBMC Image
       |
       v
BMC Flash / Storage
       |
       v
Bootloader
       |
       v
Linux
       |
       v
OpenBMC
```

OpenBMC build artifacts include image components such as U-Boot, kernel and filesystem partitions, although the exact image and storage layout depends on the platform.

We will explore **Yocto, OpenEmbedded and BitBake** in dedicated posts later.

---

## 20. Key Takeaways

After completing Day 6, you should understand:

### ✅ 1. The BMC has its own boot process.

### ✅ 2. The processor begins execution through the platform's reset/early-boot mechanism.

### ✅ 3. Boot ROM performs the earliest boot-stage work.

### ✅ 4. Some embedded platforms use multiple bootloader stages.

### ✅ 5. OpenBMC's documented boot architecture uses U-Boot for system initialization and bootstrap.

### ✅ 6. The bootloader can load the Linux kernel, initrd and device tree before transferring control to Linux.

### ✅ 7. Linux needs a userspace environment provided by the root filesystem and, depending on the platform, an initramfs.

### ✅ 8. The system eventually starts PID 1 and the init system.

### ✅ 9. OpenBMC uses systemd to manage its services and targets.

### ✅ 10. OpenBMC services are then started and the BMC becomes ready to provide management functionality.

### ✅ 11. The exact boot stages, storage layout and filesystem initialization vary by platform.

### ✅ 12. A useful mental model is:

> **Reset → Boot ROM → Bootloader → Linux → RootFS → PID 1/systemd → OpenBMC Services → BMC Ready**


## References

- OpenBMC Flash Layout and Filesystem Documentation
  https://github.com/openbmc/docs/blob/master/architecture/code-update/flash-layout.md

- OpenBMC & systemd
  https://github.com/openbmc/docs/blob/master/architecture/openbmc-systemd.md

- OpenBMC Code Update
  https://github.com/openbmc/docs/blob/master/architecture/code-update/code-update.md

- OpenBMC Cheatsheet — Kernel / FIT image example
  https://github.com/openbmc/docs/blob/master/cheatsheet.md

- Yocto — Selecting an Initialization Manager
  https://github.com/openbmc/openbmc/blob/master/upstream-layers/yocto-docs/documentation/dev-manual/init-manager.rst
