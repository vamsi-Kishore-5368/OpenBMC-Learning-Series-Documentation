# OpenBMC Learning Series — Day 4

## BMC Hardware Architecture

### What's Actually Inside a BMC?

Welcome to **Day 4** of my OpenBMC Learning Series.

In Day 1, we introduced the basic **Server Architecture** and understood where the **BMC** fits into a server.

In Day 2, we explored what a BMC actually does:

> **Monitor → Control → Manage**

In Day 3, we introduced the **BMC & OpenBMC architecture** and followed the path from:

> **BMC Hardware → Bootloader → Linux → systemd → D-Bus → OpenBMC Services → Management Interfaces**

Now we are going one level deeper.

The question for today is:

> **What hardware makes a BMC capable of running OpenBMC and managing a server?**

---

## 1. A BMC Is an Embedded Computer

A BMC is not simply a sensor-reading chip.

It is a **dedicated embedded computing platform** designed for server and platform management.

A simplified view is:

```text
                  BMC
        +-----------------------+
        |       BMC SoC         |
        |                       |
        |     Processor         |
        |     RAM               |
        |     Flash             |
        |     Network           |
        |                       |
        |   I2C / SMBus         |
        |   SPI / GPIO / UART   |
        |   Host Interfaces     |
        +-----------------------+
```

The BMC provides its own processing resources and hardware interfaces so that it can monitor and control the platform.

The exact hardware architecture varies by BMC SoC and server platform.

---

## 2. BMC SoC / Processor

At the center of the BMC is the **BMC System-on-Chip (SoC)**.

The SoC contains the processing resources and many of the peripherals required for platform management.

Its processor executes the BMC firmware and operating environment.

Conceptually:

```text
            BMC SoC
        +--------------+
        |  Processor   |
        |              |
        | Peripherals  |
        | Controllers  |
        +--------------+
```

Different vendors provide different BMC SoCs.

For example, BMC platforms may use SoCs from vendors such as **ASPEED** or **Nuvoton**.

The important concept for now is:

> **The BMC has its own processor that executes the BMC software.**

We will explore specific BMC SoCs in a future post.

---

## 3. BMC RAM

Just like any embedded computer, the BMC needs working memory.

The BMC has its own RAM for running its operating environment and applications.

This is **not the same as the host system's main RAM**.

A simplified comparison is:

```text
        HOST SYSTEM                 BMC
        +----------+                +----------+
        | Host CPU |                | BMC SoC  |
        +----+-----+                +----+-----+
             |                           |
        Host RAM                      BMC RAM
             |                           |
         Host OS                    OpenBMC
```

The host processor uses the host system's RAM.

The BMC processor uses BMC RAM.

This separation is an important reason why the BMC can operate as an independent management system.

---

## 4. BMC Flash / Non-Volatile Storage

The BMC also needs non-volatile storage for its firmware/software image.

A common implementation uses flash memory.

Conceptually:

```text
              BMC
               |
       +-------+-------+
       |               |
      RAM            Flash
       |               |
  Working Memory   Persistent
                   Firmware /
                   Software
```

A useful distinction is:

**RAM → Working memory**

**Flash → Persistent storage**

This becomes especially important later when we build an **OpenBMC image** and learn how the generated firmware image is prepared for a target platform.

---

## 5. BMC Network Interface

Remote management is one of the main purposes of a BMC.

The BMC therefore needs network connectivity.

A simplified flow is:

```text
Administrator
      |
      v
Management Network
      |
      v
BMC Network Interface
      |
      v
     BMC
```

The exact network implementation depends on the platform.

At this stage, the important idea is:

> **The BMC has network connectivity that allows management software or administrators to communicate with the management subsystem.**

Later we will explore BMC networking, Ethernet, Redfish, HTTPS, SSH and other management services in dedicated posts.

---

## 6. Hardware Interfaces

The BMC must communicate with many devices outside the BMC SoC.

Different hardware interfaces are used for different purposes.

Common interfaces include:

- I2C / SMBus
- SPI
- GPIO
- UART
- Host/platform interfaces

The exact set of interfaces depends on the BMC SoC and platform design.

---

## 7. I2C / SMBus

**I2C** and **SMBus** are commonly used to communicate with platform-management devices.

Depending on the platform, devices connected through these buses can include:

- Temperature sensors
- Fan controllers
- EEPROMs
- Voltage monitors
- Power-management devices

A simplified example:

```text
                 BMC
                  |
              I2C / SMBus
                  |
        +---------+---------+
        |         |         |
        v         v         v
   Temperature   EEPROM   Power /
      Sensor              Voltage
                           Monitor
```

For example, an I2C-connected temperature sensor may provide a temperature reading that is later exposed by the BMC's software stack.

We are introducing the interface here; the details of the I2C protocol will be covered separately.

---

## 8. SPI

**SPI** is another hardware interface used in embedded systems.

In BMC platforms, SPI can be used for devices such as flash memory and other platform-specific peripherals.

A simplified example is:

```text
BMC
 |
 +---- SPI ----> Flash
```

SPI is a synchronous serial interface.

We will explore SPI communication and timing in a future Embedded Systems post rather than going into protocol-level details here.

---

## 9. GPIO

**GPIO (General-Purpose Input/Output)** provides digital control and status signals.

In a server platform, GPIOs can be used for platform-specific functions such as:

- Power-related signals
- Reset signals
- Presence/status signals
- LEDs
- Other control signals

A simple example:

```text
          BMC
           |
          GPIO
           |
    +------+------+------+
    |      |      |      |
  Power  Reset  Status   LED
```

The exact GPIO usage is platform-specific.

---

## 10. UART

**UART** provides serial communication.

In BMC systems, UARTs can be useful for:

- Serial communication
- Console access
- Debugging
- Platform-specific communication

A simplified representation is:

```text
BMC
 |
 UART
 |
 +---- Serial Console
```

Serial console access is particularly useful during system bring-up and debugging.

---

## 11. Host-BMC Interfaces

The BMC does not only communicate with sensors and platform devices.

It also needs to communicate with the **host system**.

Depending on the platform, this can involve interfaces such as:

- LPC
- eSPI
- SMBus / I2C
- PCIe
- UART
- GPIO
- Other platform-specific interfaces

A simplified view is:

```text
                HOST
        +----------------+
        | CPU / OS       |
        +-------+--------+
                |
         Host-BMC Interface
                |
                v
               BMC
```

The exact host-BMC interface architecture varies considerably between platforms.

For example, the OpenBMC architecture documentation describes common physical interfaces such as LPC, PCIe, UART, I2C/I3C, SPI, PECI and GPIO, while explicitly noting that the actual interfaces vary by implementation. citeturn785562search2

The important concept is:

> **The BMC has dedicated paths for communicating with the host platform.**

We will explore host-BMC communication in much greater detail in a future post.

---

## 12. Sensors Are Usually External Devices

Another important concept:

> **A temperature sensor is not necessarily part of the BMC SoC.**

Often, the BMC communicates with external sensors and controllers.

For example:

```text
                BMC
                 |
               I2C
                 |
       +---------+---------+
       |         |         |
       v         v         v
 Temperature    Fan     Voltage
   Sensor     Controller Monitor
```

The BMC reads information from these devices and makes that information available to its software stack.

OpenBMC documentation includes examples of temperature sensors connected over I2C and exposed to OpenBMC through the sensor stack. citeturn785562search4turn785562search9

This connects the physical hardware to the software architecture we discussed in Day 3.

---

## 13. Power and Cooling Devices

Server platforms also contain hardware for power and thermal management.

The BMC can interact with devices such as:

- Fan controllers
- Power controllers
- Voltage monitors
- Current monitors
- Temperature sensors

A simplified hardware view is:

```text
                     BMC
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
       Sensors    Fan Control   Power Control
          |           |           |
     Temperature    Fan Speed   Power State
     Voltage
     Current
```

The exact control architecture depends on the server platform.

---

## 14. Putting the BMC Hardware Together

Now we can combine the major pieces:

```text
                         BMC
              +-----------------------+
              |       BMC SoC         |
              |                       |
              |     Processor         |
              |        |              |
              |    +---+---+          |
              |    |       |          |
              |   RAM     Flash        |
              |                       |
              |   Ethernet            |
              |   I2C / SMBus         |
              |   SPI                 |
              |   GPIO                |
              |   UART                |
              |   Host Interfaces     |
              +----------+------------+
                         |
          +--------------+---------------+
          |              |               |
          v              v               v
       Sensors       Power Devices    Host System
       EEPROMs       Fan Controllers    / CPU
```

This is the core hardware mental model for today.

---

## 15. Small Example — Reading a Temperature Sensor

Let's connect Day 4 hardware to the software architecture from Day 3.

Suppose a temperature sensor is connected to the BMC through I2C.

### Hardware path

```text
Temperature Sensor
        |
       I2C
        |
        v
BMC I2C Controller
        |
        v
      BMC SoC
```

Now add the software layers:

```text
Temperature Sensor
        |
       I2C
        |
        v
BMC I2C Controller
        |
        v
Linux Driver / Subsystem
        |
        v
OpenBMC Sensor Service
        |
        v
      D-Bus
        |
        v
 Redfish / Management Interface
        |
        v
    Administrator
```

This example shows how the **physical BMC hardware** connects to the **OpenBMC software architecture**.

It also gives us a preview of the concepts we will explore later in much greater detail.

---

## 16. BMC Hardware vs Host Hardware

It is important to keep the two systems separate.

| Host System | BMC |
|---|---|
| Host CPU | BMC SoC |
| Host RAM | BMC RAM |
| Host storage | BMC flash/storage |
| Runs host OS and applications | Runs BMC software/OpenBMC |
| Handles primary computing workload | Handles platform management |
| Uses host interfaces/devices | Uses management interfaces/devices |

The exact hardware arrangement varies by platform, but the key idea remains:

> **The BMC is its own embedded management computing environment.**

---

## 17. How Day 4 Connects to Day 3

In Day 3, we looked at the software architecture:

```text
BMC Hardware
      |
      v
Bootloader
      |
      v
Linux
      |
      v
systemd / D-Bus
      |
      v
OpenBMC Services
      |
      v
Management Interfaces
```

Today we zoomed into the first block:

```text
BMC Hardware
      |
      +-- BMC SoC
      +-- RAM
      +-- Flash
      +-- Ethernet
      +-- I2C / SMBus
      +-- SPI
      +-- GPIO
      +-- UART
      +-- Host Interfaces
```

This gives us a complete foundation:

> **Day 3 explained the software architecture.**

> **Day 4 explains the hardware underneath it.**

---

## 18. Key Takeaways

After Day 4, you should understand:

### ✅ 1. A BMC is a dedicated embedded computing platform.

### ✅ 2. The BMC has its own processor/SoC.

### ✅ 3. The BMC has its own RAM for working memory.

### ✅ 4. The BMC uses non-volatile storage such as flash for its firmware/software image.

### ✅ 5. The BMC provides network connectivity for management.

### ✅ 6. Interfaces such as I2C, SPI, GPIO and UART allow the BMC to communicate with platform devices.

### ✅ 7. The BMC also has interfaces for communicating with the host system.

### ✅ 8. Sensors and controllers are often external devices connected to the BMC.

### ✅ 9. The hardware interfaces form the bridge between physical platform devices and OpenBMC software.

### ✅ 10. A simple mental model is:

> **BMC SoC + RAM + Flash + Network + Hardware Interfaces + Connected Platform Devices**

---

## What's Next?

Now we understand:

**What a BMC is → What a BMC does → How OpenBMC is organized → What hardware is inside/around the BMC**

The next question is:

> **How does the BMC actually communicate with the host system?**

### Day 5 — BMC ↔ Host Communication

We will explore:

- Why the BMC needs to communicate with the host
- Host-BMC communication paths
- LPC
- eSPI
- I2C / SMBus
- PCIe
- UART
- GPIO
- Host IPMI
- The difference between a physical interface and a protocol

One concept at a time.

**OpenBMC Learning Series — From fundamentals to internals.**
