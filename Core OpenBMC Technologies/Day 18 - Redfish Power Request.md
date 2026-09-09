# OpenBMC Learning Series — Day 18

## OpenBMC State Management — From Redfish Power Request → D-Bus → State Manager → System Control

### From Monitoring the System to Controlling the System

---

## 1. Introduction

Day 17 followed sensor data from the sensor layer through D-Bus, ObjectMapper/associations, bmcweb, and finally Redfish. Day 18 moves to the opposite direction: **control**.

The question is:

> How does a high-level request such as **Power ON the host** become an actual system operation?

The high-level path is:

```text
Remote Management Client
        │
     Redfish
        ▼
      bmcweb
        │
      D-Bus
        ▼
 OpenBMC State Interface
        │
        ▼
phosphor-state-manager
        │
        ▼
systemd / platform control
        │
        ▼
      Host System
```

Redfish expresses management intent; D-Bus carries the internal request; the state-management implementation decides how that request is executed.

---

## 2. Why Does OpenBMC Need State Management?

A BMC cannot treat every request as a simple `power_on()` call. The system can be in different states, and a requested transition may or may not be valid in the current state.

Conceptually:

```text
BMC      → NotReady / Ready / Quiesced
Chassis  → Off / On / failure-related states
Host     → Off / Running / transitioning / other states
```

State management separates:

- current state
- requested state
- allowed transitions
- transition execution

That separation prevents higher-level clients from needing to know every hardware-specific detail.

---

## 3. The Three Major State Domains

OpenBMC commonly models three major state domains:

```text
              State Management
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
       BMC        Chassis        Host
```

They are related, but they are **not the same state machine**.

- **BMC state** → state of the management controller.
- **Chassis state** → physical chassis/power domain.
- **Host state** → host firmware/system state.

---

## 4. BMC State

`phosphor-state-manager` describes BMC states including:

```text
NotReady
Ready
Quiesced
```

The BMC can receive a reboot request through `RequestedBMCTransition`.

A critical distinction is:

> **BMC Ready does not mean the host is running.**

It means the BMC has reached the management-ready condition. The state-manager documentation relates BMC readiness to systemd targets such as `multi-user.target`.

---

## 5. Host State D-Bus Interface

The host state contract is defined by:

```text
xyz.openbmc_project.State.Host
```

Important properties include:

```text
RequestedHostTransition
AllowedHostTransitions
CurrentHostState
RestartCause
```

### RequestedHostTransition

Represents the requested host transition, for example `On`, `Off`, or `Reboot` where supported.

### CurrentHostState

A read-only representation of the host's current state.

### AllowedHostTransitions

Describes which transitions the implementation supports.

This gives us the core model:

```text
RequestedHostTransition
        │
        │ What do we want?
        ▼
   State Manager
        │
        │ What happened?
        ▼
 CurrentHostState
```

---

## 6. Why Separate Requested and Current State?

Consider a host that is initially off:

```text
CurrentHostState = Off
RequestedHostTransition = Off
```

A client requests power-on:

```text
RequestedHostTransition = On
```

The system may then transition before reaching its new steady state:

```text
Off → Transitioning → Running
```

Therefore:

> A requested transition is **not automatically proof** that the physical system has reached that state.

Comparing requested and current state is an important debugging technique.

---

## 7. Following One Real Operation: Power ON

We will use one operation throughout this document:

> **Power ON the host.**

Conceptually:

```text
Remote Client
      │
      │ Redfish
      ▼
    bmcweb
      │
      │ D-Bus
      ▼
State.Host
      │
      │ RequestedHostTransition = On
      ▼
State Manager
      │
      ▼
Platform / systemd control
      │
      ▼
Host powers on
```

This is the control-side counterpart of Day 17's sensor pipeline.

---

## 8. bmcweb's Role

From Days 15 and 16, we already know that bmcweb connects the external Redfish model to internal OpenBMC services.

For state control, the conceptual flow is:

```text
Redfish request
      ↓
bmcweb route / handler
      ↓
D-Bus request
      ↓
State-management service
```

bmcweb is **not** the host state machine. It translates the external management request into the internal OpenBMC operation.

---

## 9. D-Bus as the Internal Control Path

The same D-Bus mental model from Day 14 applies:

```text
Service
  ↓
Object
  ↓
Interface
  ↓
Property / Method
```

For host state management, the interface is:

```text
xyz.openbmc_project.State.Host
```

and the important control property is:

```text
RequestedHostTransition
```

The important new lesson is:

> A D-Bus property can represent not only information, but also a **requested system operation**.

---

## 10. phosphor-state-manager

The common OpenBMC implementation is `phosphor-state-manager`.

Its repository describes it as implementing states and state requests defined by `phosphor-dbus-interfaces`.

The host implementation contains the D-Bus-backed class for `xyz.openbmc_project.State.Host` and handlers such as:

```cpp
Transition requestedHostTransition(Transition value) override;

HostState currentHostState(HostState value) override;
```

This is where the D-Bus contract becomes actual C++ state-management logic.

---

## 11. What Happens When the Requested Transition Changes?

Conceptually:

```text
RequestedHostTransition = On
             │
             ▼
      Validate request
             │
             ▼
    Check current state
             │
             ▼
 Check allowed transition
             │
             ▼
      Execute transition
```

The current host-state manager source also contains an `executeTransition()` path.

The important architectural distinction is:

```text
D-Bus interface
      ↓
API / contract
      ↓
State manager
      ↓
Implementation
```

---

## 12. State Transition and systemd

The host-state manager maintains mappings between host states/transitions and systemd targets.

It also monitors systemd job signals such as:

```text
JobNew
JobRemoved
```

So the control path can be understood as:

```text
D-Bus request
      ↓
State manager
      ↓
systemd target / job
      ↓
Platform services
      ↓
Host/system control
```

The exact target names and platform actions are configuration-dependent.

---

## 13. Why Is systemd Involved?

OpenBMC already uses systemd to manage services, targets, ordering, and dependencies.

State-management operations can therefore integrate with:

- service startup/shutdown
- target dependencies
- ordered operations
- job completion
- failure handling

So systemd is not merely the thing that starts OpenBMC services. It can participate in coordinating system-state transitions.

---

## 14. Complete Host Power-On Flow

Putting the layers together:

```text
1. Remote Client
        │
        │ Redfish
        ▼
2. bmcweb
        │
        │ D-Bus
        ▼
3. xyz.openbmc_project.State.Host
        │
        │ RequestedHostTransition = On
        ▼
4. phosphor-state-manager
        │
        ▼
5. systemd / platform control
        │
        ▼
6. Power / boot sequence
        │
        ▼
7. Host starts
        │
        ▼
8. CurrentHostState changes
```

This is the complete control path at the architectural level.

---

## 15. What If the BMC Is Not Ready?

This is one of the most important OpenBMC state-management problems.

Imagine:

```text
BMC  = NotReady
Host = Off
```

A user requests:

```text
Power On
```

Some systems require BMC services to finish initialization before host power-on is safe.

The OpenBMC **BMC Boot Ready** design addresses this by allowing host/chassis requested state changes to be queued until the BMC reaches `Ready`.

Conceptually:

```text
Power-On Request
       │
       ▼
   BMC Ready?
    /      \
   NO      YES
   │         │
   ▼         ▼
 Queue     Execute
   │
   │ BMC reaches Ready
   ▼
Execute request
```

The design specifically describes queuing requests and executing them after BMC readiness is reached.

---

## 16. Why Queue the Request?

The BMC Boot Ready design considered approaches such as:

1. Do not expose the objects until the backend is ready.
2. Reject the request while the BMC is not ready.
3. Queue the request until the BMC is ready.

The design favors queuing as the more user-friendly behavior for the described scenario.

This hides many internal initialization dependencies from the external client.

---

## 17. Host State vs Chassis State

Host and chassis state must not be treated as identical.

For example:

```text
Chassis = On
Host    = Off
```

can be a valid intermediate condition.

Conceptually:

```text
Host State
→ host firmware/system condition

Chassis State
→ physical chassis/power-domain condition
```

A host power operation and a chassis power operation may therefore involve different state interfaces.

---

## 18. Why Do We Need Both?

Consider these situations:

```text
Chassis = On
Host    = Off
```

The chassis may be powered while the host has not yet booted.

Or:

```text
Chassis = Off
Host    = Off
```

The two states are related, but they represent different layers of the system.

This separation is important for power sequencing and platform-specific control.

---

## 19. Inspecting Host State with busctl

On a running BMC, start by discovering the relevant objects:

```bash
busctl tree
```

Then inspect the host object:

```bash
busctl introspect \
  <service> \
  /xyz/openbmc_project/state/host0
```

Read the current state:

```bash
busctl get-property \
  <service> \
  /xyz/openbmc_project/state/host0 \
  xyz.openbmc_project.State.Host \
  CurrentHostState
```

Read the requested transition:

```bash
busctl get-property \
  <service> \
  /xyz/openbmc_project/state/host0 \
  xyz.openbmc_project.State.Host \
  RequestedHostTransition
```

**Do not assume the service name** on a particular platform; discover it from the running D-Bus system.

---

## 20. Requesting a Transition Through D-Bus

The control concept is to set:

```text
RequestedHostTransition
```

to a supported transition value such as:

```text
xyz.openbmc_project.State.Host.Transition.On
```

The exact supported values are defined by the interface and platform implementation.

The important learning point is:

```text
D-Bus Property
      ↓
State Manager
      ↓
Transition Logic
      ↓
System Control
```

---

## 21. OpenBMC REST API Example

OpenBMC's host-management documentation demonstrates direct interaction with the state object through its REST interface.

A power-on request is represented as:

```bash
curl -k \
  -H "X-Auth-Token: $token" \
  -H "Content-Type: application/json" \
  -d '{"data": "xyz.openbmc_project.State.Host.Transition.On"}' \
  -X PUT \
  https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition
```

This example is useful because it exposes the underlying state-management concept directly.

For modern external management, Redfish provides the higher-level standardized interface.

---

## 22. Redfish → State Management

Our management architecture can now be visualized as:

```text
External Client
      │
   Redfish
      │
      ▼
    bmcweb
      │
    D-Bus
      │
      ▼
┌──────────────────────┐
│ OpenBMC State Model  │
│ BMC / Chassis / Host │
└──────────────────────┘
      │
      ▼
phosphor-state-manager
      │
      ▼
systemd / platform logic
      │
      ▼
     Host
```

This is the control-side architecture we have been building since Days 13–16.

---

## 23. What Happens During a Failed Transition?

A request does not guarantee success.

For example:

```text
RequestedHostTransition = On
CurrentHostState        = Off
```

The state manager may attempt the transition, but power sequencing, dependencies, hardware, or platform-specific logic may prevent the host from reaching the requested state.

Useful evidence includes:

```text
CurrentHostState
RequestedHostTransition
systemd jobs
service logs
boot progress
platform power-control logs
```

Useful tools include:

```bash
busctl
systemctl
journalctl
```

---

## 24. Debugging a Host Power-On Problem

Suppose:

> The Redfish power-on request was accepted, but the host never starts.

Trace the request backwards:

```text
1. Did bmcweb receive the request?
             ↓
2. Was the D-Bus request generated?
             ↓
3. Does State.Host exist?
             ↓
4. Did RequestedHostTransition change?
             ↓
5. Is the BMC Ready?
             ↓
6. Is the transition allowed?
             ↓
7. Did the state manager execute it?
             ↓
8. Did the corresponding systemd/platform action run?
             ↓
9. Did the physical power sequence succeed?
             ↓
10. Did CurrentHostState change?
```

This is the same layered debugging method used for sensors in Day 17.

---

## 25. Interface vs Implementation

OpenBMC separates the D-Bus contract from its implementation.

```text
phosphor-dbus-interfaces
          │
          │ defines contract
          ▼
xyz.openbmc_project.State.Host
          │
          ▼
phosphor-state-manager
          │
          │ implements behavior
          ▼
Actual state management
```

This is a recurring OpenBMC design pattern:

> **The interface defines the contract; the backend implementation provides the behavior.**

OpenBMC documentation also notes that `x86-power-control` can provide an alternative implementation for relevant platform functionality.

---

## 26. Monitoring vs Control

Compare Day 17 and Day 18.

### Day 17 — Monitoring

```text
Sensor
  ↓
D-Bus
  ↓
bmcweb
  ↓
Redfish
  ↓
Client
```

### Day 18 — Control

```text
Client
  ↓
Redfish
  ↓
bmcweb
  ↓
D-Bus
  ↓
State Manager
  ↓
Platform Control
  ↓
System
```

The two directions can be summarized as:

```text
MONITORING:
System → BMC → Client

CONTROL:
Client → BMC → System
```

That is one of the most useful mental models for BMC architecture.

---

## 27. Source Code Reading — Where to Look

### phosphor-dbus-interfaces

Host state contract:

```text
yaml/xyz/openbmc_project/State/Host.interface.yaml
```

Look here for:

- properties
- types
- transitions
- errors
- semantic meaning

### phosphor-state-manager

Host implementation:

```text
host_state_manager.hpp
```

Look here for:

- D-Bus implementation
- transition handling
- current-state handling
- systemd target mappings
- systemd job monitoring

### bmcweb

Look at Redfish routes and handlers to understand how external management requests become internal D-Bus operations.

---

## 28. A Repeatable Debugging Method

When a control operation fails, follow the same direction as the request:

```text
Redfish
  ↓
bmcweb
  ↓
D-Bus
  ↓
State Interface
  ↓
State Manager
  ↓
systemd / platform control
  ↓
Hardware
```

At each layer ask:

> Did the request arrive here?

Then ask:

> Did this layer transform or execute it correctly?

This prevents jumping directly to hardware debugging when the failure may actually be at the API or D-Bus layer.

---

## 29. Day 13 → Day 18 Progression

```text
Day 13
Source Code → Running Service

Day 14
Service → D-Bus API

Day 15
D-Bus → bmcweb → Redfish

Day 16
Redfish Request → bmcweb C++ → D-Bus → JSON

Day 17
Sensor Data → D-Bus → bmcweb → Redfish

Day 18
Redfish Control Request → bmcweb → D-Bus
              → State Manager → System Control
```

We have moved from:

> **How does OpenBMC expose information?**

to:

> **How does OpenBMC change the state of the system?**

---

## 30. Final Mental Model

Keep this architecture in mind:

```text
                    REDFISH
                       │
                       ▼
                    BMCWEB
                       │
                       ▼
                     D-BUS
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     BMC State     Chassis State   Host State
        │              │              │
        └──────────────┼──────────────┘
                       ▼
              STATE MANAGEMENT
                       │
                       ▼
             SYSTEMD / PLATFORM
                 CONTROL LOGIC
                       │
                       ▼
                    HARDWARE
```

And the two fundamental directions are:

```text
MONITOR:
Hardware → OpenBMC → D-Bus → bmcweb → Redfish → Client

CONTROL:
Client → Redfish → bmcweb → D-Bus → State Manager → Hardware
```

---

## 31. Key Takeaways

1. OpenBMC separates BMC, chassis, and host state.
2. `RequestedHostTransition` represents a requested host operation.
3. `CurrentHostState` represents the current host state.
4. Requested state and current state are not the same thing.
5. `phosphor-state-manager` implements the common OpenBMC state-management model.
6. Host state management integrates with systemd and platform-specific control logic.
7. BMC readiness can affect when host/chassis requests are executed.
8. D-Bus can represent both information and requested operations.
9. bmcweb translates external management requests into internal OpenBMC operations.
10. The final hardware-control mechanism is platform dependent.

---

## 32. One-Line Summary

> **OpenBMC turns a high-level management request such as “Power On” into a D-Bus state transition, which is interpreted by the state-management layer and ultimately executed through systemd and platform-specific control logic.**

---



# References

1. OpenBMC — phosphor-state-manager  
   https://github.com/openbmc/phosphor-state-manager

2. OpenBMC — Host State D-Bus Interface  
   https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Host.interface.yaml

3. OpenBMC — Host State Manager Source  
   https://github.com/openbmc/phosphor-state-manager/blob/master/host_state_manager.hpp

4. OpenBMC — Host Management  
   https://github.com/openbmc/docs/blob/master/host-management.md

5. OpenBMC — BMC Boot Ready Design  
   https://github.com/openbmc/docs/blob/master/designs/bmc-boot-ready.md

6. OpenBMC — State Management and External Interfaces  
   https://github.com/openbmc/docs/blob/master/designs/state-management-and-external-interfaces.md

7. OpenBMC — bmcweb  
   https://github.com/openbmc/bmcweb

8. OpenBMC — bmcweb Manager Implementation  
   https://github.com/openbmc/bmcweb/blob/master/redfish-core/lib/managers.hpp

---

## End of Day 18

**OpenBMC Learning Series**

> From Architecture → Source Code → D-Bus → Redfish → Sensors → State Management → Real System Control
