# OpenBMC Learning Series --- Day 20

## OpenBMC Power Sequencing --- From Power Rails → Power-Good → Host Boot

### How OpenBMC coordinates a platform-specific power-on sequence

------------------------------------------------------------------------

## Introduction

In **Day 19**, we moved from a logical Power ON request to the real
hardware-control layer:

``` text
Logical Power ON
      ↓
State Management
      ↓
Power Control
      ↓
GPIO / CPLD / Power Controller
      ↓
Real Hardware
```

But one important question remains:

> **What actually happens after the BMC asserts a power-control
> signal?**

A server does not normally become operational by simply setting one GPIO
HIGH.

A real platform can require:

-   standby power to already be present,
-   one or more power-enable signals,
-   power rails to reach valid levels,
-   power-good feedback,
-   timing constraints,
-   reset sequencing,
-   CPU/platform initialization,
-   POST progress,
-   and finally a transition to a running host state.

This is **power sequencing**.

The exact electrical sequence is **platform-specific**. OpenBMC provides
software interfaces and implementations that coordinate the sequence,
while board-level devices such as CPLDs, VRMs, power controllers and
other hardware may perform part of the actual electrical sequencing.

> **OpenBMC controls and observes the power sequence; it does not mean
> the BMC directly generates every server power rail.**

------------------------------------------------------------------------

# 1. What Is Power Sequencing?

Power sequencing is the controlled process of bringing a system from a
powered-off or standby condition into a valid operating state.

A simplified conceptual sequence is:

``` text
Power ON Request
      ↓
Enable / request power
      ↓
Wait for required power-good feedback
      ↓
Enable next stage
      ↓
Verify required conditions
      ↓
Release reset
      ↓
Host starts executing firmware
      ↓
POST / boot progress
      ↓
Host Running
```

The important word is **sequence**.

The platform may have dependencies such as:

``` text
Rail A valid
   ↓
Enable Rail B
   ↓
Rail B Power-Good
   ↓
Enable / release next stage
   ↓
Reset release
```

The exact order, signal names, voltage rails and timing depend on the
server design.

------------------------------------------------------------------------

# 2. Why Can't Everything Be Turned ON at Once?

Different parts of a server depend on other parts being ready first.

A simplified platform might conceptually have:

``` text
Standby Power
     ↓
Main Power Enable
     ↓
Main Rails Stable
     ↓
CPU / Memory Power Valid
     ↓
Power-Good
     ↓
Reset Release
     ↓
CPU Starts
     ↓
BIOS / Firmware
     ↓
POST
```

If reset is released before required power conditions are valid, the
processor or other components may not start correctly.

If a required rail never becomes valid, the platform should not blindly
continue as if the host were running.

Therefore, power control needs:

**Ordering + Timing + Feedback**

------------------------------------------------------------------------

# 3. The BMC Is Not the Same Thing as the Power Supply

This is one of the most important concepts in server hardware.

The BMC is a management controller.

It can:

-   request power transitions,
-   drive control signals,
-   monitor feedback signals,
-   communicate with power controllers,
-   monitor temperatures and voltages,
-   detect failures,
-   update host state,
-   and report the result through management interfaces.

But the actual electrical conversion and regulation may be performed by:

-   VRMs,
-   DC/DC converters,
-   power controllers,
-   CPLDs,
-   PMBus devices,
-   board-level logic,
-   and the system power-delivery network.

A simplified architecture is:

``` text
                 BMC
                  │
        Control / Feedback
                  │
                  ▼
        ┌─────────────────┐
        │ Platform Logic  │
        │ CPLD / Controller│
        └────────┬────────┘
                 │
          Enable / Control
                 │
                 ▼
        ┌─────────────────┐
        │ Power Controllers│
        │ / VRMs / PMBus   │
        └────────┬────────┘
                 │
                 ▼
            Power Rails
                 │
                 ▼
        CPU / Memory / Platform
```

The exact architecture differs by platform.

------------------------------------------------------------------------

# 4. Standby Power vs Main Power

Many server platforms have power available to portions of the system
even when the host is considered OFF.

Conceptually:

``` text
AC Input
   │
   ▼
Standby Power
   │
   ├── BMC
   ├── Management Logic
   └── Power-Control Logic

Power ON request
   │
   ▼
Main Power Enable
   │
   ▼
Main System Rails
   │
   ├── CPU
   ├── Memory
   ├── Platform Devices
   └── Other Loads
```

This is why a BMC can often remain reachable while the host is powered
off.

The exact rail names and topology are platform-specific.

------------------------------------------------------------------------

# 5. Power Enable Signals

A power-control implementation may use signals that request a power
transition.

Examples can include platform-specific signals such as:

``` text
POWER_OUT
PS_ON
SIO_ONCONTROL
CPU_PWR_EN
Power Button
```

These names should **not** be treated as universal OpenBMC signal names.

Different hardware platforms expose different electrical interfaces.

OpenBMC can abstract those details behind higher-level host/chassis
state interfaces.

------------------------------------------------------------------------

# 6. Power-Good Signals

A control signal answers:

> **"Please turn this part of the system on."**

A power-good signal answers:

> **"The required power condition is now valid."**

Conceptually:

``` text
BMC / Controller
      │
      │ Enable
      ▼
Power Controller
      │
      │ Power Rail
      ▼
Hardware
      │
      │ Power-Good
      ▼
BMC / Controller
```

This creates a feedback loop:

``` text
CONTROL
BMC ───────────────► Hardware

FEEDBACK
BMC ◄────────────── Hardware
```

Examples of platform-specific feedback signals can include:

``` text
POWER_GOOD
PS_PWROK
SIO_POWER_GOOD
POST_COMPLETE
RESET_STATUS
```

Again, the exact signals are platform-dependent.

------------------------------------------------------------------------

# 7. Why Power-Good Feedback Matters

Suppose OpenBMC asserts a power-enable signal.

That does **not automatically mean** that the host is powered and ready.

There are two different facts:

``` text
Command:
"Turn the platform ON."

Reality:
"Did the platform actually reach the expected power state?"
```

This is why OpenBMC power-control logic observes feedback.

For example:

``` text
Request Power ON
      ↓
Assert control signal
      ↓
Wait
      ↓
Power-Good?
   /       \
 YES        NO
  ↓          ↓
Continue   Timeout / Failure
```

Without feedback, software could incorrectly report success while the
hardware remains in a failed state.

------------------------------------------------------------------------

# 8. Timing Is Part of the Power Sequence

Power sequencing is not only about signal values.

Timing can matter too.

A control implementation may need to:

-   hold a signal for a defined pulse duration,
-   wait for a power-good signal,
-   wait before releasing reset,
-   enforce watchdog timeouts,
-   delay a retry,
-   or wait between power-cycle stages.

In the OpenBMC `x86-power-control` implementation, configuration and
source code include timing-related behavior such as power pulses,
force-off pulses, reset pulses, power-cycle timing and power-good
watchdog handling.

The exact values are implementation/configuration details and should not
be interpreted as universal server timings.

------------------------------------------------------------------------

# 9. Power Button vs Power-Rail Control

A useful distinction:

### Power Button

A power-button signal may behave like a user-facing control request:

``` text
Power Button
     ↓
Platform Logic
     ↓
Power-On Sequence
```

### Power Enable

A power-enable signal can directly control or request a hardware power
stage:

``` text
Power Enable
     ↓
Power Controller / CPLD
     ↓
Power Rails
```

The relationship between these signals is platform-specific.

OpenBMC's power-control layer can model both kinds of signals as part of
a larger state machine.

------------------------------------------------------------------------

# 10. Reset Is Part of the Sequence

Power and reset are closely related.

A simplified conceptual sequence is:

``` text
Power Rails
    ↓
Stable
    ↓
Power-Good
    ↓
Keep Host in Reset
    ↓
Required Platform Conditions Valid
    ↓
Release Reset
    ↓
CPU Starts
```

The exact reset behavior depends on the hardware architecture.

This is why a system can have:

``` text
Power = ON
```

while the host is not yet:

``` text
Host = Running
```

The host may still be:

``` text
TransitioningToRunning
```

or another intermediate state.

------------------------------------------------------------------------

# 11. Requested State vs Actual State

OpenBMC's Host D-Bus interface explicitly separates the desired host
transition from the current host state.

### RequestedHostTransition

Represents what the user/system wants.

Example:

``` text
RequestedHostTransition = On
```

### CurrentHostState

Represents what the host is actually doing.

Possible states include:

``` text
Off
TransitioningToOff
Standby
TransitioningToRunning
Running
Quiesced
DiagnosticMode
```

Therefore:

``` text
RequestedHostTransition = On
                 ≠
CurrentHostState = Running
```

during the transition.

This distinction is fundamental to understanding power sequencing.

------------------------------------------------------------------------

# 12. Complete Logical Power-On Flow

Putting the software layers together:

``` text
Remote Management Client
          │
          ▼
       Redfish
          │
          ▼
        bmcweb
          │
          ▼
        D-Bus
          │
          ▼
State.Host / State.Chassis
          │
          ▼
   State / Power Control
          │
          ▼
GPIO / D-Bus / CPLD Interface
          │
          ▼
Platform Power Logic
          │
          ▼
Power Rails
          │
          ▼
Power-Good Feedback
          │
          ▼
Reset / POST / Boot
          │
          ▼
    Host Running
```

The actual implementation can differ by platform.

------------------------------------------------------------------------

# 13. Where Does the State Manager Fit?

OpenBMC's state interfaces provide the management-level model.

For the Host interface:

``` text
RequestedHostTransition
CurrentHostState
AllowedHostTransitions
RestartCause
```

The state manager implementation reacts to transitions and coordinates
system behavior.

The key idea is:

``` text
Management Intent
       ↓
State Transition
       ↓
Platform-Specific Implementation
       ↓
Hardware Sequence
       ↓
Observed Result
       ↓
Updated State
```

The state-management layer should not be confused with the electrical
power controller.

------------------------------------------------------------------------

# 14. Where Does x86-power-control Fit?

`x86-power-control` is an OpenBMC-compliant power-control implementation
for x86 servers.

Its stated design goals include:

1.  Maintaining the Host state machine internally.
2.  Producing the requested power-control result or logging a detected
    failure.
3.  Supporting common hard and soft power operations such as
    on/off/cycle.

It uses a platform configuration file such as:

``` text
power-config-host0.json
```

and supports signal definitions through GPIO or D-Bus.

This makes it a useful real-world component for understanding how the
abstract state model reaches platform-specific power-control logic.

------------------------------------------------------------------------

# 15. GPIO and D-Bus Signals in x86-power-control

The implementation supports signal definitions of two broad types.

### GPIO

For platforms where the application can access the GPIO directly:

``` json
{
    "Name": "PostComplete",
    "LineName": "POST_COMPLETE",
    "Type": "GPIO"
}
```

Conceptually:

``` text
OpenBMC Application
        ↓
Linux GPIO Interface
        ↓
Physical GPIO
        ↓
Platform Signal
```

### D-Bus

A platform can also expose an event through D-Bus:

``` json
{
    "Name": "PowerButton",
    "DbusName": "xyz.openbmc_project.Chassis.Event",
    "Path": "/xyz/openbmc_project/Chassis/Event",
    "Interface": "xyz.openbmc_project.Chassis.Event",
    "Property": "PowerButton_Host1",
    "Type": "DBUS"
}
```

Conceptually:

``` text
Platform Event
      ↓
D-Bus Property
      ↓
x86-power-control
      ↓
Power State Machine
```

This illustrates that the power-control logic can consume both direct
hardware signals and higher-level D-Bus events.

------------------------------------------------------------------------

# 16. The Power-Control State Machine

A power-control implementation needs more than:

``` text
ON
OFF
```

It may need intermediate states.

The current x86-power-control source contains internal states such as:

``` text
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

The exact state machine is implementation-specific.

The important engineering concept is:

``` text
Request
  ↓
Intermediate State
  ↓
Wait for Event / Timeout
  ↓
Next State
  ↓
Feedback
  ↓
Final State
```

------------------------------------------------------------------------

# 17. Power ON Is Event-Driven

A realistic controller does not simply execute:

``` cpp
power_on();
return success;
```

Instead, it may behave conceptually like:

``` text
Receive ON request
      ↓
Enter transition state
      ↓
Assert control signal
      ↓
Start timer / watchdog
      ↓
Wait for feedback
      ↓
Feedback received?
   /          \
 YES           NO
 ↓             ↓
Next stage   Timeout
 ↓             ↓
Continue     Log failure
 ↓
Final state
```

This event-driven behavior is a key reason the controller needs state,
timers and event handling.

------------------------------------------------------------------------

# 18. Watchdogs and Timeouts

What happens if:

``` text
Power ON
   ↓
Wait for POWER_GOOD
   ↓
POWER_GOOD never arrives
```

The controller cannot wait forever.

A timeout/watchdog can provide a boundary:

``` text
Expected feedback
       ↓
   Timer starts
       ↓
Feedback arrives?
   /          \
 YES           NO
 ↓             ↓
Continue      Failure
             detection
                 ↓
             Logging
                 ↓
        State remains / moves
        to appropriate failure
```

The exact recovery behavior depends on the platform and implementation.

------------------------------------------------------------------------

# 19. What Happens When Power Sequencing Fails?

Possible failure points include:

``` text
Power Enable
     ↓
Power Rail
     ↓
Power-Good
     ↓
Reset
     ↓
POST
     ↓
Boot
```

### Failure A --- No Power-Good

``` text
Enable asserted
      ↓
POWER_GOOD never asserted
      ↓
Timeout
```

### Failure B --- Unexpected Reset

``` text
Power stable
      ↓
Host starts
      ↓
Reset / POST signal changes unexpectedly
      ↓
Controller detects abnormal transition
```

### Failure C --- Host never reaches Running

``` text
Power sequence succeeds
      ↓
Host begins boot
      ↓
Boot progress stops
      ↓
CurrentHostState never reaches Running
```

These are different failures and should be debugged at different layers.

------------------------------------------------------------------------

# 20. BMC Ready Is Also Part of the Control Path

OpenBMC has a design for handling power-on requests made before the BMC
is fully ready.

If a power-on or boot request reaches the Host or Chassis state objects
while the BMC is not `Ready`, the request can be queued.

Conceptually:

``` text
Power ON Request
      ↓
BMC Ready?
   /       \
 NO         YES
 ↓           ↓
Queue      Execute
 ↓
Wait
 ↓
BMC becomes Ready
 ↓
Execute request
```

This prevents platform power-on logic from racing against required BMC
initialization.

------------------------------------------------------------------------

# 21. Host State vs Chassis Power State

OpenBMC has separate concepts for the host and chassis.

For example:

``` text
Chassis:
CurrentPowerState

Host:
CurrentHostState
```

These should not automatically be treated as identical.

A chassis can be powered while the host is:

``` text
TransitioningToRunning
```

or:

``` text
Quiesced
```

Similarly, a platform may have standby power while the host firmware is:

``` text
Off
```

This separation makes the state model useful for real hardware.

------------------------------------------------------------------------

# 22. Why GPIO Is Not the Management API

A remote user should not normally have to think:

``` text
Set GPIO 37 HIGH
```

Instead, the management API should express intent:

``` text
Power ON the host
```

Then OpenBMC translates that intent into platform-specific behavior:

``` text
Power ON
   ↓
Host State
   ↓
Power Control
   ↓
GPIO / D-Bus / CPLD
   ↓
Electrical Sequence
```

This abstraction allows different platforms to implement the same
high-level management operation differently.

------------------------------------------------------------------------

# 23. Generic Power Sequence vs Real Platform Sequence

A conceptual sequence might look like:

``` text
1. Standby available
2. Power ON requested
3. Main power enabled
4. Power-good asserted
5. Required platform conditions verified
6. Reset released
7. CPU starts
8. BIOS / firmware executes
9. POST completes
10. Host reaches Running
```

But a real platform may have:

-   more rails,
-   more dependencies,
-   additional CPLD states,
-   VRM handshakes,
-   PMBus monitoring,
-   platform-specific reset behavior,
-   additional watchdogs,
-   different POST signals.

Therefore:

> **Never assume that one OpenBMC power sequence is universal across all
> servers.**

------------------------------------------------------------------------

# 24. Day 19 vs Day 20

The distinction between these two days is important.

### Day 19

``` text
Logical Power ON
      ↓
Power Control
      ↓
GPIO / Hardware
```

The focus was:

> **How software intent reaches physical control signals.**

### Day 20

``` text
Power Control
      ↓
Power Rails
      ↓
Feedback
      ↓
Timing
      ↓
Reset
      ↓
POST
      ↓
Host Running
```

The focus is:

> **What the hardware-control sequence actually has to accomplish.**

------------------------------------------------------------------------

# 25. Day 17 → Day 20

The learning progression now looks like:

``` text
Day 17
Sensor Data
   ↓
D-Bus
   ↓
bmcweb
   ↓
Redfish

Day 18
Redfish
   ↓
State Manager
   ↓
System Control

Day 19
Power ON Request
   ↓
Power Control
   ↓
GPIO
   ↓
Hardware

Day 20
Power Control
   ↓
Power Rails
   ↓
Power-Good
   ↓
Reset
   ↓
POST
   ↓
Host Running
```

We are moving from:

> **Monitoring → Management → Control → Physical Sequence**

------------------------------------------------------------------------

# 26. Real Source-Code Reading Strategy

When reading an OpenBMC power-control implementation, do not begin by
reading every line.

Instead, identify these pieces.

### Step 1 --- Find the configuration

Look for:

``` text
power-config-host0.json
```

Ask:

> What signals does this platform define?

### Step 2 --- Find the signal abstraction

Look for:

``` text
GPIO
DBUS
LineName
Property
```

Ask:

> How does the application receive or drive this signal?

### Step 3 --- Find the state machine

Look for states such as:

``` text
on
off
waitForPowerOK
waitForSIOPowerGood
```

Ask:

> What does the controller wait for?

### Step 4 --- Find events

Look for:

``` text
powerOnRequest
powerOKAssert
powerOKDeassert
POST complete
reset
```

Ask:

> What causes the state machine to move?

### Step 5 --- Find timers/watchdogs

Ask:

> What happens if expected feedback never arrives?

### Step 6 --- Find D-Bus state updates

Ask:

> How does the internal hardware state become an OpenBMC Host state?

This approach makes a large C++ source file much easier to understand.

------------------------------------------------------------------------

# 27. Debugging a Power-Sequencing Problem

Suppose:

``` text
Power ON requested
        ↓
Host never becomes Running
```

Debug from high level to low level.

### Layer 1 --- BMC / Host state

``` bash
busctl get-property \
xyz.openbmc_project.State.Host \
/xyz/openbmc_project/state/host0 \
xyz.openbmc_project.State.Host \
CurrentHostState
```

### Layer 2 --- Requested transition

``` bash
busctl get-property \
xyz.openbmc_project.State.Host \
/xyz/openbmc_project/state/host0 \
xyz.openbmc_project.State.Host \
RequestedHostTransition
```

### Layer 3 --- Power-control service

``` bash
systemctl status x86-power-control
```

### Layer 4 --- Service logs

``` bash
journalctl -u x86-power-control
```

### Layer 5 --- Kernel / GPIO information

``` bash
dmesg | grep -i gpio
gpioinfo
```

### Layer 6 --- Hardware feedback

Verify the actual platform signals:

``` text
POWER_GOOD
RESET
POST_COMPLETE
PS_PWROK
etc.
```

The exact signal names depend on the platform.

------------------------------------------------------------------------

# 28. A Practical Failure Example

Imagine:

``` text
User requests Power ON
        ↓
RequestedHostTransition = On
        ↓
Power control asserts enable
        ↓
Controller waits for POWER_GOOD
        ↓
POWER_GOOD never arrives
```

The correct mental model is not:

> "OpenBMC failed to set GPIO."

Instead ask:

``` text
Did the request reach the state manager?
        ↓
Did power-control receive the request?
        ↓
Was the control signal asserted?
        ↓
Did the power controller react?
        ↓
Did the rail become valid?
        ↓
Did POWER_GOOD assert?
        ↓
Did the state machine observe it?
        ↓
Did the host progress to the next stage?
```

This is how a real embedded firmware engineer investigates a power
problem.

------------------------------------------------------------------------

# 29. A Useful Control + Feedback Model

The complete concept can be reduced to two paths.

### Control Path

``` text
Management Request
       ↓
OpenBMC State
       ↓
Power-Control Logic
       ↓
GPIO / D-Bus / CPLD
       ↓
Power Controller
       ↓
Power Rails
```

### Feedback Path

``` text
Power Rails
       ↓
Power-Good / Status
       ↓
GPIO / D-Bus
       ↓
Power-Control Logic
       ↓
OpenBMC State
       ↓
Management Client
```

Together:

``` text
             CONTROL
BMC ─────────────────────────► HOST HARDWARE
 │                                  │
 │                                  │
 ◄──────────────────────────────────┘
             FEEDBACK
```

That feedback loop is the heart of reliable power management.

------------------------------------------------------------------------

# 30. Important Engineering Principle

The most important lesson from Day 20 is:

> **A power-control command is an intent; a power sequence is a
> controlled state transition driven by hardware feedback.**

Therefore:

``` text
Power ON
```

does not mean:

``` text
Set GPIO = 1
```

It means something closer to:

``` text
Request transition
      ↓
Execute platform-specific sequence
      ↓
Observe hardware feedback
      ↓
Handle timing / failures
      ↓
Advance through states
      ↓
Reach verified host state
```

------------------------------------------------------------------------

# 31. Complete Day 20 Mental Model

Remember this:

``` text
                 OPENBMC
                    │
                    ▼
          RequestedHostTransition
                    │
                    ▼
             State Management
                    │
                    ▼
              Power Control
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
   Control Signals         D-Bus Events
        │                       │
        └───────────┬───────────┘
                    ▼
          Platform Power Logic
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
     Power Enable         Power Controller
                              │
                              ▼
                         Power Rails
                              │
                              ▼
                        Power-Good
                              │
                              ▼
                           Reset
                              │
                              ▼
                            POST
                              │
                              ▼
                       Host Running
                              │
                              ▼
                    CurrentHostState
```

> **OpenBMC does not simply "turn the server on." It coordinates a
> platform-specific sequence, observes the hardware response, and
> represents the resulting state through its management architecture.**

------------------------------------------------------------------------

# 32. Key Takeaways

### 1. Power ON is a sequence

It is not simply a single GPIO operation.

### 2. Power rails have dependencies

The platform must bring required hardware into valid operating
conditions in the correct order.

### 3. Feedback is essential

Power-good and other status signals tell OpenBMC what actually happened.

### 4. Timing matters

Pulse widths, delays, watchdogs and timeouts are part of reliable power
control.

### 5. Reset is part of the startup sequence

Power being available does not automatically mean the CPU is executing.

### 6. Requested state and actual state are different

``` text
RequestedHostTransition
        ≠
CurrentHostState
```

during transitions.

### 7. The sequence is platform-specific

Different servers may use different GPIOs, CPLDs, VRMs, power
controllers and feedback signals.

### 8. OpenBMC provides abstraction

The external interface can remain high-level:

``` text
Power ON
```

while the underlying implementation is platform-specific.

### 9. Debugging should follow the control path

``` text
Request
 → State
 → Power Control
 → Signal
 → Hardware
 → Feedback
 → State
```

### 10. The real goal is verified state

The controller should not merely issue a command; it should determine
whether the expected hardware transition occurred.

------------------------------------------------------------------------


# 34. Final Mental Model

If you remember only one diagram from Day 20, remember this:

``` text
        MANAGEMENT INTENT
               │
               ▼
        "Power ON Host"
               │
               ▼
       OpenBMC State Model
               │
               ▼
         Power Control
               │
               ▼
     Platform-Specific Logic
               │
               ▼
        Control Signals
               │
               ▼
          Power Rails
               │
               ▼
       Hardware Feedback
               │
               ▼
        Power-Good / POST
               │
               ▼
          Reset Release
               │
               ▼
           Host Boot
               │
               ▼
        CurrentHostState
               │
               ▼
             Running
```

> **OpenBMC does not simply "turn the server on." It coordinates a
> platform-specific sequence, observes the hardware response, and
> represents the resulting state through its management architecture.**

------------------------------------------------------------------------

# References

1.  **OpenBMC x86 Power Control**\
    https://github.com/openbmc/x86-power-control

2.  **x86-power-control README**\
    https://github.com/openbmc/x86-power-control/blob/master/README.md

3.  **OpenBMC Host State Interface**\
    https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Host.interface.yaml

4.  **OpenBMC BMC State Interface**\
    https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/BMC.interface.yaml

5.  **phosphor-state-manager**\
    https://github.com/openbmc/phosphor-state-manager

6.  **Host State Manager Source**\
    https://github.com/openbmc/phosphor-state-manager/blob/master/host_state_manager.hpp

7.  **BMC Boot Ready Design**\
    https://github.com/openbmc/docs/blob/master/designs/bmc-boot-ready.md

8.  **OpenBMC `obmcutil`**\
    https://github.com/openbmc/phosphor-state-manager/blob/master/obmcutil

------------------------------------------------------------------------

## OpenBMC Learning Series Progress

``` text
Day 17 → Sensor Data → D-Bus → bmcweb → Redfish
Day 18 → Redfish → State Manager → System Control
Day 19 → Logical Power ON → GPIO → Real Hardware
Day 20 → Power Rails → Power-Good → Reset → Host Boot
```

**From software intent → to real electrical behavior.**
