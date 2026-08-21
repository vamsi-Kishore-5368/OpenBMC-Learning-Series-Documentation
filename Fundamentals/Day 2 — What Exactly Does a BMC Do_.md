# OpenBMC Learning Series — Day 2

## What Exactly Does a BMC Do?

In Day 1, we introduced the basic server architecture and understood the relationship between the **Host System, BMC, and OpenBMC**.

We established:

```text
Server
 ├── Host System
 │    ├── CPU
 │    ├── RAM
 │    └── Storage
 │
 └── BMC
      └── OpenBMC Software
```

Now we need to answer a fundamental question:

> **What exactly does the BMC do?**

A simple way to remember the responsibilities of a BMC is:

```text
             BMC
              │
      ┌───────┼────────┐
      ↓       ↓        ↓
   MONITOR  CONTROL   MANAGE
```

The BMC monitors the health of the platform, controls selected hardware functions, and provides management capabilities to administrators.

---

# 1. BMC as a Hardware Management Controller

The BMC is a dedicated controller responsible for platform management.

It is not the same as the host CPU.

The host CPU primarily executes the operating system and application workloads.

The BMC, on the other hand, is concerned with the **health, status and management of the platform hardware**.

A simplified view is:

```text
              SERVER
                 │
       ┌─────────┴─────────┐
       │                   │
   HOST SYSTEM             BMC
       │                   │
 CPU / RAM / OS       Hardware Management
                           │
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
          Monitor       Control       Remote
          Hardware      Hardware      Management
```

The exact responsibilities and hardware interfaces vary from one server platform to another.

---

# 2. Responsibility #1 — Monitoring

One of the primary jobs of a BMC is to monitor the health of the system.

A server contains many sensors that provide information about the hardware.

Examples include:

- Temperature sensors
- Fan-speed sensors
- Voltage sensors
- Power sensors
- Current sensors
- Other platform-specific sensors

A simplified flow is:

```text
Temperature Sensor ──┐
Fan Sensor ──────────┤
Voltage Sensor ──────┤
Power Sensor ────────┤
                     ↓
                    BMC
                     ↓
             Hardware Health
```

The BMC can collect this information and make it available to management software.

---

# 3. Temperature Monitoring

Consider a temperature sensor monitoring a component inside the server.

The sensor provides a measurement to the platform's management subsystem.

Conceptually:

```text
Temperature Sensor
        │
        ↓
       BMC
        │
        ↓
Temperature Information
```

The BMC can then make the information available to management applications.

This allows an administrator or higher-level management software to determine whether the system is operating within expected conditions.

---

# 4. Fan Monitoring

Servers require cooling because components such as CPUs and other hardware generate heat.

Fan sensors can provide information such as fan speed or fan status.

Conceptually:

```text
Fan
 │
 └── Fan Sensor
        │
        ↓
       BMC
        │
        ↓
   Fan Status
```

If a fan is not operating as expected, the BMC can expose that information to the management layer.

Depending on the platform design and control policy, the BMC may also participate in fan-control operations.

---

# 5. Voltage and Power Monitoring

A server also needs its electrical conditions to be monitored.

The platform may expose information such as:

- Voltage
- Current
- Power
- Power-state information

A simplified model is:

```text
Voltage / Power Sensors
          │
          ↓
         BMC
          │
          ↓
   Platform Health Data
```

This information can be useful for detecting abnormal hardware conditions and for platform management.

---

# 6. Responsibility #2 — Hardware Control

The BMC does not only observe the system.

It can also control selected platform functions.

Common examples include:

- Power control
- Reset control
- Fan control
- LED control
- Watchdog-related actions
- Other platform-specific hardware controls

A simplified representation is:

```text
                    BMC
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
     Power          Fan          Reset
     Control       Control       Control
```

The exact controls available depend on how the server hardware is designed.

---

# 7. Power Control

One of the most important BMC capabilities is platform power management.

For example, a management system may request that a server be:

```text
ON
 │
 ↓
OFF
 │
 ↓
ON
```

This can be used to perform a **power cycle**.

The important concept is that the BMC can manage platform power independently of normal operating-system applications.

---

# 8. Fan and Cooling Control

The BMC can also participate in platform cooling management.

A simplified concept is:

```text
Temperature
     │
     ↓
   Sensor
     │
     ↓
    BMC
     │
     ↓
Cooling / Fan Control
```

For example, if the platform temperature increases, the platform's management logic can adjust cooling behavior according to its configured policies.

The exact algorithm and implementation are platform-specific.

---

# 9. Responsibility #3 — Remote Management

This is one of the most important reasons BMCs are used in servers.

Imagine a server located inside a large data center.

The administrator may be hundreds of kilometers away.

Instead of physically accessing the server, the administrator can communicate with the BMC through a management network.

```text
       Administrator
              │
              │
      Management Network
              │
              ↓
             BMC
              │
       ┌──────┼──────┐
       ↓      ↓      ↓
   Sensors  Power   Console
```

This enables **remote platform management**.

Depending on the platform and software configuration, management functionality can be exposed through interfaces such as:

- Redfish
- IPMI
- SSH
- Remote console/KVM
- Other management interfaces

OpenBMC provides software components for these types of platform-management capabilities.

---

# 10. Out-of-Band Management

The concept above is commonly referred to as **out-of-band management**.

The important idea is that the management path is separate from the normal workload path.

For example:

```text
             SERVER
        ┌────────────────┐
        │                │
        │  HOST SYSTEM   │
        │  CPU / OS      │
        │                │
        │       │        │
        │       │        │
        │      BMC       │
        │       │        │
        └───────┼────────┘
                │
                ↓
        Management Network
                │
                ↓
           Administrator
```

This is particularly useful when the host operating system is unavailable.

---

# 11. What Happens if the Host OS Crashes?

Consider this situation:

```text
Host OS
   ↓
Crashes
```

The host operating system may no longer respond to normal software-based management requests.

However, the BMC is a separate management controller and is designed to operate independently of the host workload.

Therefore, depending on the platform and its power architecture, the BMC can continue providing management functionality even when the host OS is unavailable.

This is one of the most important concepts to understand:

> **The BMC provides a management path that is independent of the host operating system.**

---

# 12. A Real-World Example

Imagine a server running inside a data center.

The administrator receives an alert that the server is becoming too hot.

A simplified sequence could be:

```text
Temperature increases
        │
        ↓
Temperature Sensor
        │
        ↓
       BMC
        │
        ├── Collect temperature information
        │
        ├── Expose hardware health information
        │
        └── Platform management may take
            appropriate action
```

The administrator can then access the server's management interface remotely.

The administrator may inspect:

- Temperature
- Fan status
- Voltage
- Power state
- Hardware health

And, depending on the platform, perform an appropriate management operation.

---

# 13. The Three Core Responsibilities

We can summarize the BMC using three words:

```text
              ┌───────────┐
              │    BMC    │
              └─────┬─────┘
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
    MONITOR       CONTROL       MANAGE
       │            │            │
       ↓            ↓            ↓
    Sensors       Power       Remote
    Health        Reset       Access
    Status        Fans        Network
```

### MONITOR

Understand what is happening inside the platform.

Examples:

- Temperature
- Fan speed
- Voltage
- Power
- Hardware status

### CONTROL

Perform selected hardware-management operations.

Examples:

- Power
- Reset
- Fan
- LEDs

### MANAGE

Provide a way for administrators and management software to interact with the platform.

Examples:

- Redfish
- IPMI
- Remote console
- Management network

---

# 14. BMC vs Host CPU

It is important not to confuse the responsibilities of the host CPU and BMC.

| Host CPU | BMC |
|---|---|
| Runs the host OS | Runs management software |
| Executes applications | Manages platform hardware |
| Handles normal workloads | Monitors hardware |
| Processes user workloads | Controls selected platform functions |
| Uses the normal system resources | Has its own management resources |
| Can become unavailable when the OS crashes | Designed to provide an independent management path |

This separation is fundamental to understanding server management.

---

# 15. Where OpenBMC Fits

So far, we have discussed what the **BMC hardware** does.

But the BMC needs software to perform these functions.

This is where OpenBMC enters the picture.

```text
             BMC PLATFORM
                  │
        ┌─────────┴─────────┐
        │                   │
     Hardware             Software
        │                   │
    BMC Processor        OpenBMC
    Sensors              Linux
    Interfaces           Services
    GPIO / I²C           Management APIs
```

OpenBMC provides the software stack used to implement platform-management functionality on supported BMC hardware.

Later in the series, we will go inside this software stack and understand how components communicate and interact with the underlying hardware.

---

# 16. Key Takeaways

After Day 2, remember these points:

### 1. The BMC monitors hardware.

It collects and exposes information such as temperature, fan, voltage and power data.

### 2. The BMC controls hardware.

It can perform selected operations such as power, reset, fan and LED control depending on the platform.

### 3. The BMC provides remote management.

Administrators can interact with the platform through a management network.

### 4. The BMC is independent of the host OS.

It provides a separate management path that can remain available when the host OS is unavailable, subject to the platform's power architecture.

### 5. OpenBMC provides the software stack.

The BMC is the management hardware; OpenBMC provides the Linux-based software used to implement management functionality on that hardware.

---

# 17. Day 2 Mental Model

The simplest way to remember today's topic is:

```text
                  BMC
                   │
       ┌───────────┼───────────┐
       ↓           ↓           ↓
    MONITOR      CONTROL      MANAGE
       │           │           │
    Sensors      Power       Remote
    Health       Reset       Access
    Status       Fans        Network
```

> **BMC = Monitor + Control + Manage**

This three-part mental model will be useful as we move into the internal architecture of OpenBMC.

---

## What's Next?

### Day 3 — BMC Hardware Architecture

We know **what a BMC does**.

Now we need to understand:

> **What is actually inside a BMC?**

We'll explore concepts such as:

- BMC processor
- BMC RAM
- BMC flash
- GPIO
- I²C / SMBus
- SPI
- UART
- Network interface
- Host-BMC interfaces
- Sensors and hardware connections

From there, we'll gradually move from **BMC hardware → BMC firmware → OpenBMC software**.

---

## References

- OpenBMC Project: https://github.com/openbmc/openbmc
- OpenBMC Documentation: https://github.com/openbmc/docs
- OpenBMC Architecture: https://github.com/openbmc/docs/tree/master/architecture