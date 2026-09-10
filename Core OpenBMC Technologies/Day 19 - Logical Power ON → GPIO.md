# OpenBMC Learning Series — Day 19

## From Logical Power ON → GPIO → Real Hardware

### How a High-Level Management Request Becomes Platform-Specific Hardware Control

---

## 1. Introduction

In Day 18, we followed a host power-control request through the OpenBMC state-management layer:

```text
Remote Client
      ↓
Redfish
      ↓
bmcweb
      ↓
D-Bus
      ↓
Host / Chassis State
      ↓
State Management
      ↓
Platform Control
```

The next question is:

> **What actually happens after OpenBMC decides that the host should power ON?**

A real server cannot be powered on by simply setting a generic software variable. The platform may require GPIO outputs, GPIO inputs, power-enable signals, power-good feedback, reset signals, power-button pulses, CPLD interaction, I²C/other controller interaction, timing, watchdogs, and platform-specific sequencing.

The central idea is:

> **A BMC power operation is not simply “set a GPIO.” It is the translation of a high-level management intent into platform-specific control signals, followed by hardware feedback that tells the BMC whether the requested state was actually achieved.**

---

## 2. Day 18 → Day 19

Day 18 gave us:

```text
Power ON Request
       ↓
bmcweb
       ↓
D-Bus
       ↓
State Management
       ↓
Platform Control
```

Day 19 expands the last part:

```text
Power ON Request
       ↓
bmcweb
       ↓
D-Bus
       ↓
State / Power Control
       ↓
Platform-Specific Logic
       ↓
GPIO / CPLD / Other Controller
       ↓
Power Rails
       ↓
Host
```

There is also a feedback path:

```text
Host / Power Hardware
       ↓
Power-Good / Reset / POST / Status
       ↓
GPIO / Controller
       ↓
Power-Control Logic
       ↓
D-Bus State
```

So the complete system is a **control + feedback loop**.

---

## 3. What Does “Power ON” Really Mean?

When someone says:

```text
Power ON
```

it is tempting to imagine:

```text
GPIO = 1
```

But real server platforms are more complicated.

A platform may expose signals such as:

```text
POWER_BUTTON#
RESET_BUTTON#
POWER_GOOD
POST_COMPLETE
PLATFORM_RESET
PS_ON
CPU_RESET
```

The exact signals, names, polarity, and sequence are platform-specific.

The current OpenBMC `x86-power-control` implementation contains configurable signals including:

```text
PowerOut
PowerOk
ResetOut
NMIOut
SioPowerGood
SioOnControl
SIOS5
PostComplete
PlatformReset
PowerButton
ResetButton
IdButton
NMIButton
SlotPower
HpmStbyEn
```

These are examples supported by that implementation, not universal signals for every server.

---

## 4. GPIO Is a Hardware Signal Interface

A GPIO can be an output:

```text
BMC
 │
 │ GPIO output
 ▼
Platform hardware
```

or an input:

```text
Platform hardware
 │
 │ GPIO input
 ▼
BMC
```

Examples of outputs:

```text
Power Enable
Reset
Power Button
```

Examples of inputs:

```text
Power Good
POST Complete
Fault
Button Status
```

Therefore:

> **GPIO should be understood as a hardware signal interface between the BMC and the platform, not as the complete power-control policy.**

---

## 5. Control Signals vs Feedback Signals

### Control

```text
BMC → Hardware
```

Examples:

- Power Enable
- Power Button
- Reset
- Slot Power

### Feedback

```text
Hardware → BMC
```

Examples:

- Power Good
- POST Complete
- Reset Status
- Platform Status

The relationship is:

```text
             CONTROL
BMC ─────────────────────► Hardware
 ▲                           │
 │                           │
 │          FEEDBACK         │
 └───────────────────────────┘
```

A BMC should not assume that a control signal succeeded simply because it was asserted.

---

## 6. Where Does GPIO Come From?

GPIO is not a special OpenBMC-only mechanism.

The lower-level path is generally:

```text
Physical Pin
     ↓
BMC SoC GPIO Controller
     ↓
Linux GPIO subsystem
     ↓
Userspace / Driver Interface
     ↓
OpenBMC Application
```

The hardware relationship is described through platform configuration, commonly including Device Tree.

This connects directly to Day 9.

---

## 7. Day 9 → Day 19: Device Tree Revisited

In Day 9 we learned:

```text
Hardware
   ↓
Device Tree
   ↓
Linux Kernel
   ↓
Driver / Subsystem
   ↓
Software Interface
```

Now that knowledge becomes practical:

```text
GPIO Hardware
     ↓
Device Tree / Platform Description
     ↓
Linux GPIO Support
     ↓
Power-Control Application
     ↓
Actual GPIO Operation
```

The exact implementation varies by platform.

The important idea is:

> **Device Tree describes hardware relationships; the power-control software uses the resulting platform interfaces to implement the required behavior.**

---

## 8. A Real OpenBMC Implementation: x86-power-control

A useful real implementation to study is:

```text
openbmc/x86-power-control
```

It is an OpenBMC-compliant implementation of power control for x86 servers.

Its documented goals include:

- maintaining the Host state machine internally
- tracking state changes
- supporting hard power on/off/cycle
- supporting soft power on/off/cycle
- logging detected failures

It is designed to support platforms with power-control GPIOs similar to those represented in its configuration.

---

## 9. How x86-power-control Is Configured

The implementation uses:

```text
power-config-host0.json
```

The configuration can be customized for a platform.

The current project supports two kinds of signal definitions:

```text
GPIO
DBUS
```

A GPIO signal can be configured with a line name and polarity.

A D-Bus signal can instead refer to:

```text
D-Bus name
Object path
Interface
Property
```

Important point:

> **Not every power-control signal has to be a direct BMC GPIO.**

---

## 10. Why Support GPIO and D-Bus Signals?

A platform might expose:

```text
Power Good
```

through a direct GPIO.

Another platform might expose the same logical information through another service.

Conceptually:

```text
                 Power Control
                       │
              ┌────────┴────────┐
              ▼                 ▼
            GPIO               D-Bus
              │                 │
              ▼                 ▼
        Hardware signal    OpenBMC service
```

This lets the high-level power-control state machine remain relatively independent of how a particular signal is delivered.

---

## 11. The Power-Control State Machine

The current `x86-power-control` source does not model power as merely:

```text
ON
OFF
```

It contains internal states such as:

```text
on
waitForPowerOK
waitForSIOPowerGood
off
transitionToOff
gracefulTransitionToOff
cycleOff
transitionToCycleOff
gracefulTransitionToCycleOff
checkForWarmReset
```

This reveals an important fact:

> **Real power control is a state machine, not a single GPIO write.**

---

## 12. Why Are Intermediate States Necessary?

Consider:

```text
Power ON requested
 ↓
begin power operation
 ↓
wait for power-good
 ↓
wait for another platform-good signal
 ↓
verify expected state
 ↓
update host/chassis state
```

Conceptually:

```text
OFF
 ↓
PowerOnRequest
 ↓
WAIT_FOR_POWER_OK
 ↓
POWER_OK_ASSERTED
 ↓
WAIT_FOR_SIO_POWER_GOOD
 ↓
ON
```

If the expected feedback never arrives:

```text
WAIT_FOR_POWER_OK
        ↓
      timeout
        ↓
      failure
```

---

## 13. Timers Are Part of Power Control

The current `x86-power-control` source maintains timers for operations such as:

```text
PowerPulseMs
ForceOffPulseMs
ResetPulseMs
PowerCycleMs
SioPowerGoodWatchdogMs
PowerOKWatchdogMs
GracefulPowerOffS
WarmResetCheckMs
```

Current source defaults include values such as:

```text
PowerPulseMs = 200
ForceOffPulseMs = 15000
ResetPulseMs = 500
PowerCycleMs = 5000
PowerOKWatchdogMs = 8000
```

These are implementation defaults, not universal electrical requirements. Platform configuration can change the behavior.

---

## 14. Why Timing Matters

A platform may expect a power-button pulse rather than a permanently asserted level:

```text
HIGH
  │
  ├──── LOW ────┤
  │              │
  ▼              ▼
assert          release
```

Likewise, after enabling power, the BMC may need to wait for:

```text
Power Good
```

within a watchdog period.

Therefore:

> **Power control is a time-dependent hardware state machine.**

---

## 15. Power-Good Feedback

Conceptually:

```text
BMC:
"Start power-on."
        ↓
Platform:
"Power rails are coming up."
        ↓
Hardware:
"Rails are valid."
        ↓
POWER_GOOD = asserted
        ↓
BMC:
"Power is actually good."
```

This prevents the BMC from confusing:

```text
Requested power
```

with:

```text
Successful power
```

---

## 16. What If Power Good Never Arrives?

Suppose:

```text
Power ON requested
        ↓
Power control starts
        ↓
Wait for Power Good
        ↓
Timeout
```

Possible causes include:

- failed power rail
- voltage regulator problem
- power controller failure
- incorrect GPIO configuration
- wrong polarity
- hardware fault
- sequencing problem
- platform dependency not satisfied

The current x86 power-control source logs failures such as:

```text
system power good failed to assert
power okay failed to assert
```

This creates a direct bridge from:

```text
Hardware failure
```

to:

```text
OpenBMC logging
```

---

## 17. Power Button vs Power Enable

These are not necessarily the same operation.

### Power Enable

```text
BMC → enable power circuitry
```

### Power Button

```text
BMC → emulate a physical button press
```

A platform may use one, the other, or a combination.

For example:

```text
BMC
 ↓
Power Button Pulse
 ↓
Platform Controller
 ↓
Power Rails
```

Another platform might directly control a power-enable signal.

Therefore:

> **The management operation “Power ON” does not define the electrical implementation.**

---

## 18. Requested State vs Actual Hardware State

Day 18 introduced:

```text
RequestedHostTransition
CurrentHostState
```

Day 19 explains why both exist.

Imagine:

```text
RequestedHostTransition = On
```

while the power controller is:

```text
waitForPowerOK
```

The host is not yet fully running.

Only after the required transitions and feedback occur can the state become:

```text
CurrentHostState = Running
```

The Host D-Bus interface explicitly says that comparing `CurrentHostState` and `RequestedHostTransition` can indicate whether the system is in transition.

---

## 19. The Host State Interface

The interface is:

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

The current interface defines host states including:

```text
Off
TransitioningToOff
Standby
Running
TransitioningToRunning
Quiesced
DiagnosticMode
```

and transitions including:

```text
Off
On
Reboot
GracefulWarmReboot
ForceWarmReboot
```

Not every platform necessarily supports every transition; `AllowedHostTransitions` communicates the supported set.

---

## 20. Where Does State Management End and Power Control Begin?

This distinction is important.

A high-level state interface can say:

```text
RequestedHostTransition = On
```

but that does not imply that the same component universally performs the physical GPIO operation.

OpenBMC supports different implementations.

Conceptually:

```text
State Management
      │
      ├── phosphor-state-manager
      │
      └── x86-power-control
             │
             ▼
       platform-specific
       power-control logic
```

The OpenBMC BMC Boot Ready design explicitly notes that `x86-power-control` can be an alternative implementation to `phosphor-state-manager` for that logic.

Therefore:

> **Do not assume that every OpenBMC platform uses exactly the same power-control daemon.**

---

## 21. The BMC-Ready Dependency

The BMC state interface defines states including:

```text
Ready
NotReady
UpdateInProgress
```

`Ready` means the required BMC services have started and are running successfully.

OpenBMC's BMC Boot Ready design specifies that if a power-on or boot request is made to the Chassis or Host state object while the BMC is not `Ready`, the request can be queued and executed once `Ready` is reached.

Conceptually:

```text
Power Request
      ↓
BMC Ready?
   ↙       ↘
 NO         YES
 ↓           ↓
Queue       Execute
 ↓
BMC Ready
 ↓
Execute
```

This is why BMC readiness is more than a cosmetic status value.

---

## 22. The Complete Power-On Path

```text
Remote Management Client
          │
          │ Redfish
          ▼
        bmcweb
          │
          │ D-Bus
          ▼
RequestedHostTransition = On
          │
          ▼
State / Power-Control Implementation
          │
          ▼
Platform Power State Machine
          │
          ├──────────────┐
          │              │
          ▼              ▼
      GPIO Output     D-Bus Signal
          │              │
          └──────┬───────┘
                 ▼
        Platform Hardware
                 │
          Power Sequencing
                 │
                 ▼
          Power Good / Status
                 │
                 ▼
          Power-Control Logic
                 │
                 ▼
          CurrentHostState
                 │
                 ▼
              Running
```

This is the central architecture of Day 19.

---

## 23. Control Path vs Feedback Path

### Control path

```text
Client
  ↓
Redfish
  ↓
bmcweb
  ↓
D-Bus
  ↓
Power Control
  ↓
GPIO / Controller
  ↓
Hardware
```

### Feedback path

```text
Hardware
  ↓
Power Good / Reset / POST
  ↓
GPIO / Controller
  ↓
Power Control
  ↓
D-Bus State
  ↓
bmcweb / Redfish
  ↓
Client
```

Therefore:

```text
           CONTROL
Client ─────────────────► Hardware
   ▲                         │
   │                         │
   └─────────────────────────┘
             FEEDBACK
```

A BMC coordinates both directions.

---

## 24. Why Raw GPIO Should Not Become the Management API

A tempting design would be:

```text
Redfish
   ↓
Set GPIO 37 HIGH
```

This is a poor abstraction.

Another platform might use:

```text
GPIO 37
```

while another uses:

```text
I²C → CPLD
```

and another uses:

```text
Power Controller
```

The management interface should remain:

```text
Power ON
```

while the platform implementation changes.

Therefore:

```text
             Management abstraction
                    │
                 Power ON
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
     GPIO platform       CPLD platform
```

This is a key software-architecture lesson.

---

## 25. Debugging a Power-ON Failure

Suppose:

> **Redfish accepted my Power ON request, but the server is still OFF.**

Follow the complete chain.

### Step 1 — External request

Did the Redfish request reach `bmcweb`?

### Step 2 — D-Bus

Did `RequestedHostTransition` change?

### Step 3 — Power-control service

Is the relevant service running?

```bash
systemctl status <power-control-service>
```

### Step 4 — Current state

```bash
busctl get-property   <service>   /xyz/openbmc_project/state/host0   xyz.openbmc_project.State.Host   CurrentHostState
```

### Step 5 — Hardware control

Was the expected GPIO/controller action performed?

### Step 6 — Feedback

Did:

```text
Power Good
```

assert?

### Step 7 — Timing

Did a watchdog timeout occur?

### Step 8 — Host state

Did the host eventually reach:

```text
Running
```

This layered method is much more effective than randomly checking signals.

---

## 26. Useful Debugging Tools

### D-Bus

```bash
busctl tree
```

Find relevant D-Bus objects.

```bash
busctl introspect <service> <object-path>
```

Inspect interfaces and properties.

```bash
busctl get-property   <service>   <object-path>   xyz.openbmc_project.State.Host   CurrentHostState
```

Read current host state.

```bash
busctl get-property   <service>   <object-path>   xyz.openbmc_project.State.Host   RequestedHostTransition
```

Read requested transition.

### systemd

```bash
systemctl status <service>
```

Check whether the relevant service is running.

### Logs

```bash
journalctl -u <service>
```

Inspect service logs.

### Kernel

```bash
dmesg
```

Check kernel/driver messages.

### GPIO

Depending on the platform and Linux GPIO interface:

```bash
gpioinfo
gpioget
gpioset
```

Use direct GPIO manipulation carefully; forcing a platform signal can have real hardware consequences.

---

## 27. Reading the Real x86-power-control Source

The current source combines several OpenBMC/Linux technologies:

```text
C++
sdbusplus
libgpiod
systemd
Boost.Asio
JSON configuration
persistent state
logging
```

It maintains D-Bus interfaces for:

```text
xyz.openbmc_project.State.Host
xyz.openbmc_project.State.Chassis
```

and GPIO-backed configuration for platform signals.

This is an excellent example of several concepts from earlier days coming together in one real component.

---

## 28. The Internal Power State Machine

The current source defines an internal:

```cpp
enum class PowerState
```

with states such as:

```text
on
waitForPowerOK
waitForSIOPowerGood
off
transitionToOff
gracefulTransitionToOff
cycleOff
transitionToCycleOff
gracefulTransitionToCycleOff
checkForWarmReset
```

It also defines events such as:

```text
powerOKAssert
powerOKDeAssert
sioPowerGoodAssert
powerButtonPressed
powerOnRequest
powerOffRequest
powerCycleRequest
resetRequest
warmResetDetected
```

The useful mental model is:

```text
STATE + EVENT
     ↓
Transition
     ↓
New STATE
```

That is a classic event-driven state-machine design.

---

## 29. Why a State Machine Instead of One Giant Function?

Imagine:

```cpp
powerOn()
{
    setGPIO(...);
    sleep(...);
    setGPIO(...);
    sleep(...);
}
```

This quickly becomes difficult to maintain.

A state machine makes behavior explicit:

```text
OFF
 ↓ powerOnRequest
WAIT_FOR_POWER_OK
 ↓ powerOKAssert
WAIT_FOR_SIO_POWER_GOOD
 ↓ signal
ON
```

If an error occurs:

```text
WAIT_FOR_POWER_OK
 ↓ timeout
failure / recovery path
```

This structure makes asynchronous events, timeouts, feedback, recovery, and logging easier to reason about.

---

## 30. Power Control Is Platform-Specific

There is no single universal electrical power sequence for all OpenBMC systems.

The actual implementation depends on:

- BMC SoC
- motherboard
- CPLD
- power controller
- VRM
- GPIO topology
- reset architecture
- host processor
- firmware requirements

Therefore:

```text
Common OpenBMC abstraction
          │
          ▼
Platform-specific implementation
          │
          ▼
Actual electrical behavior
```

OpenBMC gives us reusable interfaces and components, but the platform integration is where the real hardware knowledge matters.

---

## 31. Day 18 → Day 19

### Day 18

> **What does “Power ON” mean inside OpenBMC?**

```text
Redfish
 ↓
bmcweb
 ↓
D-Bus
 ↓
Host State
 ↓
State Management
```

### Day 19

> **How does that logical request become a real hardware operation?**

```text
State Request
 ↓
Power-Control State Machine
 ↓
GPIO / D-Bus / Controller
 ↓
Power Sequencing
 ↓
Power Good
 ↓
Host State
```

Day 18 gave us the **logical control path**.

Day 19 introduces the **platform/hardware control path**.

---

## 32. Day 13 → Day 19 Progression

```text
Day 13
Source Code → Running Service

Day 14
Service → D-Bus API

Day 15
D-Bus → bmcweb → Redfish

Day 16
Redfish → bmcweb C++ → D-Bus → JSON

Day 17
Sensor → D-Bus → bmcweb → Redfish

Day 18
Redfish → State Management → System Control

Day 19
State Control → Power-Control Logic
          → GPIO / Controller
          → Hardware → Feedback
```

We are progressively moving:

```text
Architecture
     ↓
Software
     ↓
IPC
     ↓
Management API
     ↓
State Management
     ↓
Hardware Control
```

---

## 33. Final Mental Model

Keep this diagram in mind:

```text
                    REMOTE CLIENT
                         │
                      Redfish
                         │
                         ▼
                       bmcweb
                         │
                        D-Bus
                         │
                         ▼
                Host / Chassis State
                         │
                         ▼
                Power-Control Logic
                         │
                ┌────────┴────────┐
                ▼                 ▼
             GPIO             D-Bus /
           Controller        Other source
                │                 │
                └────────┬────────┘
                         ▼
                 Platform Hardware
                         │
                  Power Sequence
                         │
                         ▼
                    Power Good
                         │
                         ▼
                 Host / CPU / Rails
                         │
                         ▼
                    BMC Feedback
                         │
                         ▼
                    Current State
```

And the most important distinction is:

```text
REQUESTED STATE
      ≠
PHYSICAL STATE
```

The control logic exists to move the physical system toward the requested state and report what actually happened.

---

## 34. Key Takeaways

1. A BMC power operation is **not simply a GPIO toggle**.
2. GPIO provides hardware-level digital signals, while power-control policy sits above it.
3. GPIO can be used for both **control outputs** and **status inputs**.
4. Power-control implementations use state machines because real hardware transitions are asynchronous and time-dependent.
5. Power-good signals provide feedback that helps the BMC determine whether a power operation actually succeeded.
6. Timers and watchdogs help detect failed or stalled transitions.
7. `x86-power-control` is a real OpenBMC implementation combining D-Bus, GPIO, systemd, JSON configuration, timers, and logging.
8. `RequestedHostTransition` expresses the requested host transition; `CurrentHostState` represents the current host state.
9. The exact electrical implementation is platform-specific.
10. The high-level management interface should remain independent of whether a platform uses GPIO, a CPLD, I²C, or another controller.
11. BMC readiness can delay execution of host/chassis power requests.
12. Debugging should follow the complete chain from Redfish → D-Bus → power-control logic → hardware → feedback.

---

## 35. One-Line Summary

> **OpenBMC converts a high-level “Power ON” intent into a platform-specific power-control state machine that drives GPIOs or other controllers, waits for hardware feedback such as Power Good, and updates system state based on what actually happened.**

---

## 36. What's Next?

Day 19 showed:

```text
Power ON
   ↓
Power-Control Logic
   ↓
Hardware Signals
```

The next question is:

> **What happens when the platform has multiple power rails, dependencies, delays, and power-good signals that must occur in a specific order?**

That takes us to:

# Day 20 — OpenBMC Power Sequencing

We will move from:

```text
"Power ON"
```

to:

```text
Rail 1
  ↓
Power Good
  ↓
Rail 2
  ↓
Power Good
  ↓
CPU Power
  ↓
Reset Release
  ↓
POST
  ↓
Host Running
```

That will take us even closer to the actual electrical sequence of a server motherboard.

---

# References

1. OpenBMC — x86-power-control  
   https://github.com/openbmc/x86-power-control

2. OpenBMC — x86-power-control README  
   https://github.com/openbmc/x86-power-control/blob/master/README.md

3. OpenBMC — x86-power-control source (`power_control.cpp`)  
   https://github.com/openbmc/x86-power-control/blob/master/src/power_control.cpp

4. OpenBMC — x86-power-control build/configuration  
   https://github.com/openbmc/x86-power-control/blob/master/meson.build

5. OpenBMC — Host State D-Bus Interface  
   https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Host.interface.yaml

6. OpenBMC — phosphor-state-manager  
   https://github.com/openbmc/phosphor-state-manager

7. OpenBMC — Host State Manager  
   https://github.com/openbmc/phosphor-state-manager/blob/master/host_state_manager.hpp

8. OpenBMC — BMC Boot Ready Design  
   https://github.com/openbmc/docs/blob/master/designs/bmc-boot-ready.md

9. OpenBMC — Host Management  
   https://github.com/openbmc/docs/blob/master/host-management.md

10. OpenBMC — Development / Add New System  
    https://github.com/openbmc/docs/blob/master/development/add-new-system.md

11. OpenBMC — Anti-Patterns  
    https://github.com/openbmc/docs/blob/master/anti-patterns.md

---

## End of Day 19

### OpenBMC Learning Series

**From Architecture → Source Code → D-Bus → Redfish → Sensors → State Management → Real Hardware Control**
