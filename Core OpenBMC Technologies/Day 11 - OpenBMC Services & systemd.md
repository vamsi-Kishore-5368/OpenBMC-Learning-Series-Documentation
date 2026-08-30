# OpenBMC Learning Series — Day 11

## OpenBMC Services & systemd

### How OpenBMC Starts, Runs and Manages Its Services

Welcome to **Day 11** of the OpenBMC Learning Series.

In Day 10, we learned how OpenBMC services communicate using **D-Bus**.

Now we answer another important question:

> **Who starts and manages all these OpenBMC services?**

The answer is:

> **systemd**

OpenBMC uses systemd to manage its processes and uses systemd units, targets and service dependencies to control system behavior.

---

## 1. What Is an OpenBMC Service?

OpenBMC is not one single application.

It is made up of many processes and services, each responsible for a particular function.

Examples include:

- Sensor services
- Power and chassis management
- Inventory
- Logging
- Network services
- IPMI
- Redfish / BMCWeb
- State-management services

A simplified view is:

```text
                 OpenBMC
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
    Sensors       Power       Inventory
       │            │            │
       └────────────┼────────────┘
                    ↓
                  D-Bus
```

---

## 2. Who Starts These Services?

When the BMC boots, Linux needs to bring up the required userspace components.

OpenBMC uses **systemd** as its service manager.

A simplified boot path is:

```text
BMC Power ON
     ↓
Bootloader
     ↓
Linux Kernel
     ↓
systemd
     ↓
OpenBMC Services
```

So systemd becomes an important part of the transition from:

> **Linux has booted**

to:

> **The BMC management software is running.**

---

## 3. What Does systemd Actually Manage?

OpenBMC uses several systemd unit types. Three concepts are especially useful here.

### Unit

A **unit** is a basic object managed by systemd.

### Service

A **service unit** defines a process/application that systemd can start and manage.

Example:

```text
example.service
```

### Target

A **target** can act as a synchronization point and can group units that should be brought to a particular system state.

So:

```text
systemd
   │
   ├── Services
   ├── Targets
   └── Other Units
```

OpenBMC makes extensive use of service and target units.

---

## 4. A `.service` File

A systemd service is commonly described using a `.service` unit file.

A simplified example:

```ini
[Unit]
Description=Example OpenBMC Service

[Service]
ExecStart=/usr/bin/example-service

[Install]
WantedBy=multi-user.target
```

This example is intentionally simplified.

The important idea is that the unit file describes information systemd needs to manage the application.

For example:

```text
What is the service?
        ↓
How should it be started?
        ↓
When should it be started?
        ↓
What does it depend on?
```

On an OpenBMC system, systemd unit files can be found in locations such as:

```text
/lib/systemd/system/
/etc/systemd/system/
/run/systemd/system/
```

---

## 5. Service Lifecycle

A simplified service lifecycle looks like:

```text
              systemd
                 │
                 ↓
            Start Service
                 │
                 ↓
          ┌──────────────┐
          │   RUNNING    │
          └──────┬───────┘
                 │
          ┌──────┴───────┐
          ↓              ↓
       Stop           Failure
          ↓              ↓
      INACTIVE       Restart /
                     Recovery
```

The exact behavior depends on the service's systemd configuration.

systemd provides a structured way to start, stop, order and recover services.

---

## 6. Service Dependencies

OpenBMC contains many services that depend on other services or system states.

For example:

```text
Service A
   ↓
needs Service B
   ↓
Service B must be available first
```

systemd provides dependency and ordering mechanisms for this.

Some important relationships include:

```text
Requires
Wants
After
Before
```

For example:

```text
Service A
   │
   └── After → Service B
```

means systemd should order A after B.

A dependency is not necessarily the same thing as ordering, so these concepts should be understood separately when working with real unit files.

---

## 7. Targets in OpenBMC

Targets are particularly important in OpenBMC.

Instead of thinking only in terms of individual services, OpenBMC can group system behavior around higher-level states.

A simplified example:

```text
                Target
                  │
        ┌─────────┼─────────┐
        ↓         ↓         ↓
    Service A  Service B  Service C
```

OpenBMC uses targets for operations such as host startup, shutdown and power-state transitions.

Examples include:

```text
obmc-host-start@0.target
obmc-host-startmin@0.target
obmc-host-stop@0.target
obmc-host-shutdown.target
```

The exact targets and relationships depend on the OpenBMC platform/configuration.

---

## 8. systemd + D-Bus

This is the most important connection between **Day 10 and Day 11**.

Remember:

> **D-Bus helps services communicate.**

And:

> **systemd manages services and their lifecycle.**

Conceptually:

```text
                  systemd
                     │
              starts/manages
                     ↓
             OpenBMC Service
                     │
                  D-Bus
                     ↓
            Other OpenBMC Services
```

These are two different responsibilities:

```text
systemd → Service lifecycle

D-Bus   → Service communication
```

Keeping this distinction clear is extremely useful when reading OpenBMC architecture and source code.

---

## 9. A Practical Example

Imagine a service responsible for exposing BMC state.

A simplified flow could be:

```text
BMC boots
    ↓
systemd starts the service
    ↓
Service initializes
    ↓
Service creates/uses D-Bus objects
    ↓
Other OpenBMC services communicate
    ↓
Redfish / IPMI can expose or control state
```

This connects several layers we've studied so far.

---

## 10. `systemctl` — Working With systemd

On a running BMC, one of the main tools for interacting with systemd is:

```bash
systemctl
```

Useful commands include:

```bash
systemctl status <service>
systemctl start <service>
systemctl stop <service>
systemctl restart <service>
systemctl list-units
systemctl list-unit-files
```

### What does each command do?

- `systemctl status <service>` → Shows the **current state and recent information** about a service.
- `systemctl start <service>` → **Starts** a service.
- `systemctl stop <service>` → **Stops** a service.
- `systemctl restart <service>` → **Restarts** a service.
- `systemctl list-units` → Lists **currently loaded/active units**.
- `systemctl list-unit-files` → Lists **installed unit files and their enablement state**.

---

## 11. Checking Service Logs

When a service fails, we also need to know **why**.

For that, systemd's journal can be inspected using:

```bash
journalctl -u <service>
```

This shows journal entries associated with the specified service.

A useful debugging flow is:

```text
Service not working
       ↓
systemctl status
       ↓
Check journal
       ↓
journalctl -u <service>
       ↓
Inspect dependencies / configuration
```

---

## 12. A Simple Debugging Example

Suppose an OpenBMC service is not running.

Instead of immediately opening the source code, we can start with:

```bash
systemctl status <service>
```

If it failed:

```bash
journalctl -u <service>
```

Then investigate:

```text
Is the service installed?
        ↓
Is the unit loaded?
        ↓
Did it start?
        ↓
Did it fail?
        ↓
What does the journal say?
        ↓
Does it depend on another service?
        ↓
Does it depend on a D-Bus object/interface?
```

This is a practical way to approach OpenBMC debugging.

---

## 13. OpenBMC Service → systemd → D-Bus

Let's combine the two days.

### Day 10

We learned:

```text
Service A
    │
    ↓
  D-Bus
    │
    ↓
Service B
```

### Day 11

We now add systemd:

```text
             systemd
                │
        ┌───────┴───────┐
        ↓               ↓
    Service A        Service B
        │               │
        └────── D-Bus ──┘
```

So the two technologies solve different problems:

| Technology | Main responsibility |
|---|---|
| **systemd** | Starts, stops, orders and manages services |
| **D-Bus** | Allows services/processes to communicate |

---

## 14. Connecting Everything We Have Learned

Our architecture is becoming much clearer:

```text
BMC Hardware
     ↓
Device Tree
     ↓
Linux Kernel
     ↓
Drivers / Subsystems
     ↓
OpenBMC Services
     ↓
┌───────────────┐
│    systemd    │ ← manages services
└───────────────┘
     │
     ↓
┌───────────────┐
│    D-Bus      │ ← service communication
└───────────────┘
     ↓
Redfish / IPMI / Other Interfaces
```

This is a simplified conceptual view, but it gives us the right mental model before we start reading actual OpenBMC source code.

---

## 15. Day 11 Mental Model

Remember these two lines:

> **systemd manages the service.**

> **D-Bus helps the service communicate.**

And remember:

```text
.service  → Individual service definition

.target   → Group/state/synchronization point

systemctl → Interact with systemd

journalctl → Inspect service logs
```

---

## 16. Key Takeaways

### ✅ 1.
OpenBMC consists of many independent services/processes.

### ✅ 2.
OpenBMC uses **systemd** to manage its processes.

### ✅ 3.
A **service unit** describes a process/application managed by systemd.

### ✅ 4.
A **target** can group services and represent a system state or synchronization point.

### ✅ 5.
systemd provides dependency and ordering mechanisms.

### ✅ 6.
`systemctl` is the primary command-line tool for interacting with systemd units.

### ✅ 7.
`journalctl` helps investigate service logs.

### ✅ 8.
D-Bus and systemd have different responsibilities:

> **systemd → manages**

> **D-Bus → communicates**

---

## 🔜 What's Next?

Now we know:

```text
Hardware
   ↓
Linux
   ↓
Drivers
   ↓
OpenBMC Services
   ↓
systemd + D-Bus
```

The next step is to start looking at **how an OpenBMC service is actually built**.

### Day 12 — Inside an OpenBMC Service

We can explore:

- Source code structure
- `sdbusplus`
- D-Bus interfaces
- Service executable
- `.service` files
- Yocto/BitBake integration
- How everything gets built into the BMC image

One concept at a time. 🚀

---

## References

1. **OpenBMC — OpenBMC & systemd**  
   https://github.com/openbmc/docs/blob/master/architecture/openbmc-systemd.md

2. **OpenBMC — Interface Overview**  
   https://github.com/openbmc/docs/blob/master/architecture/interface-overview.md

3. **OpenBMC — Monitoring and Logging systemd Target and Service Failures**  
   https://github.com/openbmc/docs/blob/master/designs/target-fail-monitoring.md

4. **OpenBMC — Development: Hello World**  
   https://github.com/openbmc/docs/blob/master/development/devtool-hello-world.md

5. **systemd Documentation**  
   https://systemd.io/

6. **OpenBMC GitHub**  
   https://github.com/openbmc
