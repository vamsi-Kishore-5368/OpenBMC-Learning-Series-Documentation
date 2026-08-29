# OpenBMC Learning Series — Day 10

## Understanding D-Bus in OpenBMC

### How OpenBMC Services Communicate With Each Other

Welcome to **Day 10** of the OpenBMC Learning Series.

In the previous days, we moved from BMC hardware to Linux, Device Tree and drivers.

Now we move into one of the most important concepts in the OpenBMC userspace:

> **How do the different OpenBMC services communicate with each other?**

The answer is:

> **D-Bus**

---

## 1. Why Do We Need D-Bus?

OpenBMC is not one single application.

It consists of many services/components that perform different tasks, such as:

- Sensor monitoring
- Fan control
- Power control
- Inventory management
- Firmware management
- Chassis management
- Redfish
- Other platform services

These components need to exchange information and request actions from one another.

A simplified example:

```text
Sensor Service
      │
      │ Temperature = 72°C
      ↓
     ???
      ↓
Redfish Service
```

We need a mechanism that allows these independent processes to communicate.

That is where **D-Bus** comes in.

OpenBMC documentation describes D-Bus interfaces as the primary mechanism for inter-process communication between OpenBMC applications, while noting that other mechanisms can also be used. [1]

---

## 2. What Is D-Bus?

**D-Bus** is an **Inter-Process Communication (IPC)** system.

IPC means that separate processes can communicate and exchange information.

Conceptually:

```text
┌─────────────────┐
│ Sensor Service  │
└────────┬────────┘
         │
         │ D-Bus
         ↓
┌─────────────────┐
│ Other Service   │
└─────────────────┘
```

In OpenBMC, many services use D-Bus to communicate and expose system state.

So we can think of D-Bus as an important **communication layer between OpenBMC services**.

---

## 3. D-Bus in the OpenBMC Architecture

A simplified OpenBMC view looks like:

```text
                  OpenBMC
                     │
       ┌─────────────┼─────────────┐
       ↓             ↓             ↓
 Sensor Service  Power Service  Inventory
       │             │             │
       └─────────────┼─────────────┘
                     ↓
                   D-Bus
                     ↑
       ┌─────────────┼─────────────┐
       ↓             ↓             ↓
   Redfish       Chassis       Other Services
```

The services can communicate through D-Bus without every service needing a direct, custom connection to every other service.

This helps keep the architecture modular.

---

## 4. The D-Bus Object Model

To understand OpenBMC code, simply knowing that D-Bus is IPC is not enough.

We need to understand a few important concepts:

```text
Object
   │
   └── Interface
          ├── Properties
          ├── Methods
          └── Signals
```

These concepts appear repeatedly when working with OpenBMC.

---

## 5. Object

A D-Bus **object** represents a particular managed entity.

For example, an OpenBMC system may have an object representing a temperature sensor.

A simplified object path could look like:

```text
/xyz/openbmc_project/sensors/temperature/CPU
```

The exact object path depends on the service and implementation.

Think of an object as:

> **The thing we are talking about.**

For example:

```text
CPU Temperature Sensor
```

---

## 6. Object Path

The object is addressed using an **object path**.

Example:

```text
/xyz/openbmc_project/sensors/temperature/CPU
```

The hierarchy makes the object identifiable within the D-Bus system.

Conceptually:

```text
/xyz/openbmc_project/
        │
        └── sensors/
              │
              └── temperature/
                    │
                    └── CPU
```

OpenBMC's sensor documentation uses object paths under `/xyz/openbmc_project/sensors/...` for sensor objects. [2]

---

## 7. Interface

An **interface** defines a set of functionality and data associated with an object.

For example, a sensor object can expose information such as:

```text
Value
Unit
Warning Threshold
Critical Threshold
```

So the relationship can be viewed as:

```text
Object
  │
  └── Interface
         ├── Properties
         ├── Methods
         └── Signals
```

An object can expose one or more interfaces.

OpenBMC's D-Bus interface definitions are maintained separately in the `phosphor-dbus-interfaces` project. [3]

---

## 8. Properties

A D-Bus **property** represents data/state associated with an interface.

For example:

```text
Value = 72
Unit  = DegreesC
```

This allows other services to access information representing the current state of an object.

For OpenBMC, this is particularly important because platform state such as sensor readings, power state and inventory information can be represented through D-Bus objects and properties.

OpenBMC's sensor architecture, for example, maps sensors to D-Bus objects and exposes sensor properties through D-Bus interfaces. [2]

---

## 9. Methods

A D-Bus **method** represents an action that another process can request.

For example, conceptually:

```text
Power Service
      │
      │ Request
      ↓
   Power Control
      │
      ↓
    PowerOn()
```

A method is therefore something we can think of as:

> **"Please perform this operation."**

Methods can also have input and output arguments.

---

## 10. Signals

A D-Bus **signal** is used to announce that something happened.

For example:

```text
Temperature changes
        ↓
   D-Bus Signal
        ↓
Interested Services
```

Unlike a method call, a signal is generally an event notification.

Think of it as:

> **"Something changed or something happened."**

This becomes useful when services need to react to changes without constantly polling another service.

OpenBMC's sensor architecture specifically documents `PropertiesChanged` notifications when sensor or threshold values change. [2]

---

## 11. A Simple Way to Remember the Concepts

| D-Bus Concept | Think of it as |
|---|---|
| **Object** | What are we talking about? |
| **Object Path** | Where is that object addressed? |
| **Interface** | What functionality/data does it expose? |
| **Property** | What is its current state/data? |
| **Method** | Ask it to perform an action |
| **Signal** | Announce that something happened |

This mental model will be extremely useful when reading OpenBMC code.

---

## 12. Service / Bus Name

There is another concept we will encounter frequently:

> **Service name (bus name)**

A D-Bus service can own a well-known bus name so that other processes can address the service.

Conceptually:

```text
Service / Bus Name
        ↓
     Service
        ↓
   Object Path
        ↓
    Interface
        ↓
Properties / Methods / Signals
```

This gives us a useful way to mentally parse a D-Bus endpoint.

---

## 13. Practical Example — Temperature Sensor

Let's connect the concepts.

Suppose the BMC has a temperature sensor.

A simplified path is:

```text
Temperature Sensor
        ↓
Linux Driver / Hardware Interface
        ↓
OpenBMC Sensor Service
        ↓
D-Bus Object
        ↓
Temperature Property
        ↓
Redfish Service
        ↓
Management Client
```

For example, the D-Bus object could conceptually contain:

```text
Object:
/xyz/openbmc_project/sensors/temperature/CPU

Property:
Value = 72
```

A Redfish service can obtain the relevant information through the OpenBMC D-Bus architecture and expose it through the management interface.

OpenBMC documents this relationship for sensors: sensor daemons publish sensor objects on D-Bus, and BMCWeb/Redfish support obtains sensor information from those D-Bus objects. [2]

This connects several topics from our previous days:

```text
Hardware
   ↓
Linux Driver
   ↓
OpenBMC Service
   ↓
D-Bus
   ↓
Redfish
```

---

## 14. Methods vs Properties vs Signals

This distinction is worth remembering.

### Property

Represents **state**.

```text
Temperature = 72°C
```

### Method

Represents an **action/request**.

```text
PowerOn()
```

### Signal

Represents an **event notification**.

```text
TemperatureChanged
```

So:

```text
Property → What is the state?

Method   → Please do something.

Signal   → Something happened.
```

---

## 15. Where Does D-Bus Fit?

We can now extend the OpenBMC architecture:

```text
┌─────────────────────────────┐
│       Management Client     │
│       Redfish / IPMI        │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│       OpenBMC Services      │
│ Sensor / Power / Inventory  │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│            D-Bus            │
│      Service Communication  │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│        Linux Kernel         │
│ Drivers / Subsystems        │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│        BMC Hardware         │
└─────────────────────────────┘
```

This is a simplified conceptual architecture; the exact path varies depending on the service and hardware.

---

## 16. `busctl` — Looking at D-Bus

One of the most useful tools when working with D-Bus on a Linux/OpenBMC system is:

```bash
busctl
```

It allows us to inspect and interact with the D-Bus environment.

For example:

```bash
busctl list
```

can be used to list available bus names/services.

```bash
busctl tree
```

can be used to inspect object paths exposed on the bus.

And:

```bash
busctl introspect <SERVICE> <OBJECT_PATH>
```

can be used to inspect the interfaces, properties, methods and signals exposed by an object.

These commands become extremely useful when debugging a running OpenBMC system.

---

## 17. Why `busctl` Matters for OpenBMC Development

Suppose a sensor is expected to appear in OpenBMC but something isn't working.

Instead of immediately looking through large amounts of source code, we can inspect the running system.

A simplified debugging approach:

```text
Is the service running?
        ↓
Does the expected D-Bus service exist?
        ↓
Does the expected object exist?
        ↓
Does the expected interface exist?
        ↓
Does the property contain the expected value?
```

This gives us a practical way to understand what is happening at runtime.

---

## 18. D-Bus Is Not an OpenBMC-Only Protocol

This distinction is important.

D-Bus itself is a general **IPC mechanism**.

OpenBMC makes extensive use of D-Bus for communication between its services and for representing platform objects/state.

So avoid thinking:

> "D-Bus is an OpenBMC protocol."

A better mental model is:

> **D-Bus is an IPC system that OpenBMC uses as a major part of its service architecture.**

OpenBMC's own documentation describes D-Bus as the primary IPC mechanism between OpenBMC applications, while also noting that other IPC mechanisms exist. [1]

---

## 19. Connecting Day 8 → Day 9 → Day 10

Our learning is now starting to connect.

### Day 8

```text
Linux Kernel
     ↓
Subsystem
     ↓
Driver
     ↓
Hardware
```

### Day 9

```text
Device Tree
     ↓
Linux Kernel
     ↓
Driver Matching
     ↓
Hardware
```

### Day 10

```text
OpenBMC Services
        ↓
      D-Bus
        ↓
Other OpenBMC Services
```

Putting them together:

```text
Hardware
   ↓
Device Tree / Linux
   ↓
Driver / Subsystem
   ↓
OpenBMC Service
   ↓
D-Bus
   ↓
Other OpenBMC Services
   ↓
Redfish / Management Interface
```

This is the bigger picture we are building throughout the series.

---

## 20. Day 10 Mental Model

Remember these two layers:

```text
             OPENBMC
                │
       ┌────────┼────────┐
       ↓        ↓        ↓
    Sensor    Power   Inventory
       │        │        │
       └────────┼────────┘
                ↓
              D-Bus
                ↓
        Service Communication
```

And for an individual D-Bus object:

```text
Service / Bus Name
        ↓
Object Path
        ↓
Interface
   ├── Properties
   ├── Methods
   └── Signals
```

---

## 21. Key Takeaways

### ✅ 1.
D-Bus is an **Inter-Process Communication (IPC)** system.

### ✅ 2.
OpenBMC consists of many services that need to communicate.

### ✅ 3.
D-Bus provides an important communication mechanism between those services.

### ✅ 4.
A D-Bus object represents a managed entity.

### ✅ 5.
An object is addressed using an object path.

### ✅ 6.
An interface defines functionality/data exposed by an object.

### ✅ 7.
Properties represent state/data.

### ✅ 8.
Methods represent requested actions.

### ✅ 9.
Signals announce events or changes.

### ✅ 10.
`busctl` is a useful tool for inspecting D-Bus at runtime.

---

## References

1. **OpenBMC — Interface Overview**  
   https://github.com/openbmc/docs/blob/master/architecture/interface-overview.md

2. **OpenBMC — Sensor Architecture**  
   https://github.com/openbmc/docs/blob/master/architecture/sensor-architecture.md

3. **OpenBMC — phosphor-dbus-interfaces**  
   https://github.com/openbmc/phosphor-dbus-interfaces

4. **D-Bus Specification**  
   https://dbus.freedesktop.org/doc/dbus-specification.html

5. **systemd — busctl**  
   https://www.freedesktop.org/software/systemd/man/latest/busctl.html

6. **OpenBMC GitHub**  
   https://github.com/openbmc
