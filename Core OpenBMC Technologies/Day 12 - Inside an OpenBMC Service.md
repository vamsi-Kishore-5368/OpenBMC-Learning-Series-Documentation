# OpenBMC Learning Series — Day 12

## Inside an OpenBMC Service

### From Source Code to a Running BMC Service

Welcome to **Day 12** of the OpenBMC Learning Series.

So far, we have built the architecture step by step:

- Day 10 → D-Bus
- Day 11 → systemd
- Day 12 → **How an OpenBMC service is actually built and becomes a running service**

The goal today is to connect the pieces we have already learned.

> **Source Code → Build → Install → Start → Communicate**

---

## 1. What Is an OpenBMC Service?

OpenBMC is not one large application.

It consists of many applications/services responsible for different management functions.

Examples include:

- Sensor services
- Power/chassis management
- Inventory
- Logging
- Network services
- IPMI
- Redfish/BMCWeb
- State-management services

Conceptually:

```text
                         OpenBMC
                            │
          ┌─────────────────┼─────────────────┐
          ↓                 ↓                 ↓
      Sensor Service    Power Service    Inventory Service
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ↓
                          D-Bus
```

OpenBMC documentation describes D-Bus as the primary IPC mechanism between OpenBMC applications, while systemd controls the services and interfaces they provide.

---

## 2. What Does an OpenBMC Service Actually Look Like?

A service can contain several pieces:

```text
OpenBMC Service
      │
      ├── Source Code
      ├── Build Configuration
      ├── D-Bus Interface
      ├── systemd Service File
      └── Yocto / BitBake Recipe
```

These pieces have different jobs.

### Source Code

Implements the actual application logic.

### D-Bus Interface

Defines how the application exposes or consumes management data.

### systemd Service File

Tells systemd how the application should be managed and started.

### Yocto / BitBake Recipe

Describes how the application is fetched, configured, built, packaged and integrated into the OpenBMC image.

---

## 3. The Complete Journey

This is the most important mental model for Day 12:

```text
              SOURCE CODE
                   │
                   ↓
            C++ Application
                   │
                   ↓
              sdbusplus
                   │
                   ↓
             D-Bus Objects
             & Interfaces
                   │
                   ↓
             Service Binary
                   │
                   ↓
           systemd .service
                   │
                   ↓
             Yocto / BitBake
                   │
                   ↓
             OpenBMC Image
                   │
                   ↓
                  Boot
                   │
                   ↓
                systemd
                   │
                   ↓
            Running Service
                   │
                   ↓
                 D-Bus
                   │
                   ↓
        Other OpenBMC Services
```

This is a simplified conceptual flow. The exact build and runtime details vary between OpenBMC components.

---

## 4. Source Code → Application

At the beginning, we have the source code of the application.

A simplified project might look like:

```text
my-service/
├── src/
│   └── main.cpp
├── include/
├── meson.build
├── service file
└── recipe
```

This is only a conceptual example.

Real OpenBMC repositories do not all use exactly the same directory structure or build configuration.

The important idea is:

> **The source code becomes an executable application that can run on the BMC.**

---

## 5. Where Does `sdbusplus` Fit?

This connects directly to **Day 10 — D-Bus**.

We learned:

> **D-Bus provides communication between processes.**

But OpenBMC C++ applications need a practical way to work with D-Bus.

That's where **sdbusplus** comes in.

The `sdbusplus` project provides:

1. A C++ library for interacting with D-Bus.
2. `sdbus++`, a tool used to generate C++ bindings for D-Bus-based applications.

Conceptually:

```text
OpenBMC C++ Application
          │
          ↓
      sdbusplus
          │
          ↓
        D-Bus
          │
          ↓
Other OpenBMC Applications
```

So:

> **D-Bus is the IPC mechanism.**

> **sdbusplus provides a C++ interface for working with D-Bus.**

---

## 6. D-Bus Interface

A service can expose a D-Bus interface containing:

```text
Properties
Methods
Signals
```

Conceptually:

```text
Service
   │
   ↓
Object Path
   │
   ↓
Interface
   ├── Properties
   ├── Methods
   └── Signals
```

The `sdbusplus` project supports YAML definitions describing methods, properties, signals, paths and service names. Documentation and binding code can be generated from these definitions.

For example, conceptually:

```text
Interface:
xyz.openbmc_project.Sensor.Value

Properties:
    Value
    MaxValue
    MinValue
```

This interface becomes a contract between the service and other software that consumes the data.

---

## 7. What Happens Inside the Service?

A simplified service startup flow might look like:

```text
main()
  ↓
Initialize application
  ↓
Connect to D-Bus
  ↓
Create D-Bus objects/interfaces
  ↓
Register objects/interfaces
  ↓
Start event loop
  ↓
Wait for requests / signals / events
```

A real implementation can be considerably more complex, but this gives us the basic mental model.

---

## 8. The Service Executable

After compilation, the source code becomes an executable.

Conceptually:

```text
main.cpp
   ↓
Compiler / Build System
   ↓
my-service
```

The executable may ultimately be installed somewhere such as:

```text
/usr/bin/my-service
```

The exact path depends on the component.

Now we have an application that can actually run on the BMC.

But one question remains:

> **Who starts it?**

That brings us back to Day 11.

---

## 9. The `.service` File

systemd needs to know how the application should be managed.

A simplified service unit might look like:

```ini
[Unit]
Description=Example OpenBMC Service
After=network.target

[Service]
ExecStart=/usr/bin/my-service

[Install]
WantedBy=multi-user.target
```

The important line is:

```ini
ExecStart=/usr/bin/my-service
```

It tells systemd which executable to start.

Conceptually:

```text
.service file
      ↓
    systemd
      ↓
/usr/bin/my-service
      ↓
Running application
```

---

## 10. What About Yocto and BitBake?

OpenBMC uses the **Yocto Project** to manage configuration and creation of BMC images. BitBake is the build engine used by Yocto/OpenBMC.

A simplified flow is:

```text
Source Code
     ↓
BitBake Recipe (.bb)
     ↓
Fetch
     ↓
Configure
     ↓
Compile
     ↓
Install / Package
     ↓
Root Filesystem
     ↓
OpenBMC Image
```

The recipe tells the build system what is needed to build and package the component.

---

## 11. What Does a BitBake Recipe Do?

A simplified recipe can contain information such as:

```text
Source location
Dependencies
Build system
Configuration
Files to install
Packages to create
```

For example:

```text
my-service.bb
```

The exact syntax and contents depend on the component.

A real OpenBMC recipe can declare dependencies such as `sdbusplus`, `systemd`, and other libraries/components.

So the recipe acts as the bridge between:

```text
Application Source
       ↓
OpenBMC Build System
       ↓
BMC Image
```

---

## 12. Putting systemd + Yocto Together

### Yocto / BitBake

Builds and packages the application.

### systemd

Runs and manages the application at runtime.

Conceptually:

```text
              BUILD TIME
                 │
Source Code → BitBake → BMC Image
                              │
                              ↓
                         ──────────
                           BOOT
                         ──────────
                              │
                              ↓
                         systemd
                              │
                              ↓
                        Service starts
```

So:

> **BitBake helps put the service into the image.**

> **systemd helps run the service on the BMC.**

---

## 13. Practical Example — Sensor Service

Let's connect everything using a simplified temperature-sensor example.

```text
Temperature Sensor
        ↓
Linux Driver / Hardware Interface
        ↓
OpenBMC Sensor Application
        ↓
sdbusplus
        ↓
D-Bus Object / Interface
        ↓
Other OpenBMC Services
        ↓
Redfish / Management Interface
```

Now add the build/runtime side:

```text
Sensor Service Source Code
          ↓
      BitBake Recipe
          ↓
       Build Image
          ↓
          BMC
          ↓
        systemd
          ↓
    Sensor Service Starts
          ↓
         D-Bus
          ↓
     Sensor Data Available
```

This is the type of complete path we will increasingly follow in future days.

---

## 14. One Service — Many Technologies

A single OpenBMC service can therefore involve several technologies:

```text
             OpenBMC Service
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
   C++ Source    D-Bus       systemd
       │         /sdbusplus      │
       │            │            │
       └────────────┼────────────┘
                    ↓
              Yocto / BitBake
                    ↓
              BMC Firmware
```

This is why OpenBMC development requires understanding more than just C++.

You need to understand how the pieces connect.

---

## 15. Connecting Day 10 → Day 11 → Day 12

### Day 10 — D-Bus

> **How do OpenBMC applications communicate?**

```text
Service A
    │
  D-Bus
    │
Service B
```

### Day 11 — systemd

> **How are OpenBMC services started and managed?**

```text
systemd
   ↓
Service
```

### Day 12 — Inside the Service

> **How does the service get built and become part of the BMC?**

```text
Source
  ↓
Build
  ↓
Package
  ↓
BMC Image
  ↓
systemd
  ↓
Service
  ↓
D-Bus
```

This is the transition from **architecture learning → development learning**.

---

## 16. The Complete Mental Model

```text
                         BMC
                          │
                    Linux Kernel
                          │
                  Drivers / Subsystems
                          │
                  OpenBMC Services
                          │
             ┌────────────┴────────────┐
             ↓                         ↓
          systemd                    D-Bus
        Service lifecycle       Inter-service IPC
             │                         │
             └────────────┬────────────┘
                          ↓
                 Redfish / IPMI / etc.
```

And during development:

```text
Source Code
     ↓
Yocto / BitBake
     ↓
OpenBMC Image
     ↓
BMC Boot
     ↓
systemd
     ↓
Service
     ↓
D-Bus
```

---

## 17. Day 12 Mental Model

Remember these five words:

> **Source → Build → Install → Start → Communicate**

More specifically:

```text
Source Code
     ↓
Yocto / BitBake
     ↓
BMC Image
     ↓
systemd
     ↓
OpenBMC Service
     ↓
sdbusplus / D-Bus
     ↓
Other Services
```

---

## 18. Key Takeaways

### ✅ 1.
An OpenBMC service is an application responsible for a specific management function.

### ✅ 2.
The source code is compiled into an executable that can run on the BMC.

### ✅ 3.
`sdbusplus` provides C++ support for interacting with D-Bus and includes tooling for generating bindings.

### ✅ 4.
A D-Bus interface can define properties, methods and signals.

### ✅ 5.
A systemd `.service` file tells systemd how to manage/start an application.

### ✅ 6.
Yocto and BitBake are involved in building and generating the OpenBMC BMC image.

### ✅ 7.
At runtime, systemd manages the service while D-Bus enables communication between OpenBMC applications.

The big picture is:

> **Yocto/BitBake builds it.**

> **systemd runs it.**

> **D-Bus connects it.**

---

## References

1. OpenBMC — Interface Overview  
   https://github.com/openbmc/docs/blob/master/architecture/interface-overview.md

2. OpenBMC — sdbusplus  
   https://github.com/openbmc/sdbusplus

3. sdbusplus — D-Bus Interface YAML  
   https://github.com/openbmc/sdbusplus/blob/master/docs/yaml/interface.md

4. sdbusplus — Example Applications  
   https://github.com/openbmc/sdbusplus/tree/master/example

5. OpenBMC — OpenBMC & systemd  
   https://github.com/openbmc/docs/blob/master/architecture/openbmc-systemd.md

6. OpenBMC — Yocto Development  
   https://github.com/openbmc/docs/blob/master/yocto-development.md

7. OpenBMC — Example BitBake Recipe  
   https://github.com/openbmc/openbmc/blob/master/meta-phosphor/recipes-phosphor/dbus/phosphor-dbus-interfaces_git.bb

8. OpenBMC Documentation  
   https://github.com/openbmc/docs
