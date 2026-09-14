# 🚀 OpenBMC Learning Series — Day 23

## Sensor Thresholds & Fault Detection in OpenBMC
### From Sensor Value → Threshold → Alarm → Event → Fault Handling

---

## 1. Introduction

Day 17 answered:

> **What is the sensor value?**

Day 21–22 answered:

> **What hardware does that sensor belong to, and where can its identity come from?**

Day 23 asks the next question:

> **When does a sensor value become abnormal, and what does OpenBMC do about it?**

A simplified example:

```text
CPU Temperature = 72°C
WarningHigh    = 70°C
CriticalHigh   = 80°C

72 > 70  → Warning condition
72 < 80  → Not yet Critical
```

If the value becomes 85°C:

```text
85 > 80 → Critical condition
```

The central idea is:

> **A sensor value tells us what is happening. A threshold gives OpenBMC a rule for deciding when that value becomes abnormal.**

---

## 2. What Is a Sensor Threshold?

A threshold is a boundary used to interpret a sensor value.

For a temperature sensor:

```text
             CriticalHigh
                  ▲
                  │
             WarningHigh
                  ▲
                  │
               NORMAL
                  │
             WarningLow
                  ▼
                  │
             CriticalLow
```

OpenBMC commonly represents:

```text
WarningHigh
WarningLow
CriticalHigh
CriticalLow
```

The exact values are platform-specific. OpenBMC's sensor documentation exposes these threshold concepts through D-Bus. citeturn0search4turn1search3

---

## 3. Why Are Thresholds Needed?

A raw measurement does not automatically tell us whether it requires attention.

```text
Value = 85°C
```

becomes meaningful when compared with:

```text
WarningHigh  = 70°C
CriticalHigh = 80°C
```

Therefore:

```text
Sensor Value
     +
Thresholds
     ↓
Condition / Meaning
```

This is the transition from **monitoring** to **condition detection**.

---

## 4. Warning vs Critical

### Warning

The sensor has moved outside the preferred operating range and may require attention.

### Critical

The sensor has crossed a more severe boundary.

For example:

```text
70°C → Warning
80°C → Critical
```

OpenBMC's threshold error definitions describe `CriticalHigh` as the sensor exceeding its upper bound and `CriticalLow` as exceeding its lower bound. citeturn1search6

Important:

> **Critical does not automatically mean shutdown.**

The threshold condition can be consumed by logging, monitoring, cooling or protective-control software depending on the platform.

---

## 5. High vs Low Thresholds

### High

The value is too large:

```text
Value > High Threshold
```

Examples:

- Temperature
- Voltage
- Power

### Low

The value is too small:

```text
Value < Low Threshold
```

Examples:

- Voltage
- Fan speed
- Pressure
- Temperature

Example:

```text
Voltage = 10.5V
CriticalLow = 11.0V

10.5 < 11.0
      ↓
CriticalAlarmLow
```

---

## 6. Sensor.Value — The Measurement

The standard interface is:

```text
xyz.openbmc_project.Sensor.Value
```

It provides properties such as:

```text
Value
MaxValue
MinValue
Unit
```

The current OpenBMC interface defines `Value` as the sensor reading and places Sensor.Value objects under the OpenBMC sensor namespace. citeturn1search7

Example:

```text
/xyz/openbmc_project/sensors/temperature/CPU_Temp

Value = 72.5
Unit  = DegreesC
```

The threshold layer builds on top of this:

```text
Value
  +
Thresholds
  ↓
Alarm state
```

---

## 7. Warning Threshold Interface

OpenBMC defines:

```text
xyz.openbmc_project.Sensor.Threshold.Warning
```

with:

```text
WarningHigh
WarningLow

WarningAlarmHigh
WarningAlarmLow
```

The OpenBMC sensor architecture documentation maps these YAML properties directly to D-Bus properties. citeturn1search3turn1search4

Think of them as:

```text
WarningHigh
    → Upper warning boundary

WarningLow
    → Lower warning boundary

WarningAlarmHigh
    → Current high-warning alarm state

WarningAlarmLow
    → Current low-warning alarm state
```

---

## 8. Critical Threshold Interface

The corresponding critical interface is:

```text
xyz.openbmc_project.Sensor.Threshold.Critical
```

with concepts such as:

```text
CriticalHigh
CriticalLow

CriticalAlarmHigh
CriticalAlarmLow
```

So a sensor can conceptually expose:

```text
xyz.openbmc_project.Sensor.Value
xyz.openbmc_project.Sensor.Threshold.Warning
xyz.openbmc_project.Sensor.Threshold.Critical
```

These are **D-Bus contracts**. The component that implements and evaluates them depends on the sensor/platform architecture.

---

## 9. Threshold Value vs Alarm Property

This distinction is essential.

```text
WarningHigh = 70
```

means:

> Where is the warning boundary?

But:

```text
WarningAlarmHigh = true
```

means:

> Is the high-warning alarm currently asserted?

Likewise:

```text
CriticalHigh
    → boundary

CriticalAlarmHigh
    → current alarm state
```

Remember:

> **Threshold = rule. Alarm = current result of that rule.**

---

## 10. Complete Temperature Example

Suppose:

```text
Value        = 72°C
WarningHigh  = 70°C
CriticalHigh = 80°C
```

Then:

```text
72 > 70
72 < 80
```

Conceptually:

```text
WarningAlarmHigh = true
CriticalAlarmHigh = false
```

If the value becomes:

```text
85°C
```

then the critical-high condition is reached.

The exact alarm state published by a sensor implementation depends on that implementation's threshold logic.

---

## 11. Threshold Crossing and Recovery

Threshold monitoring is not only about detecting a problem.

A condition can also clear.

```text
65°C
 ↓
72°C
 ↓
85°C
 ↓
75°C
 ↓
65°C
```

Conceptually:

```text
Normal
  ↓
Warning
  ↓
Critical
  ↓
Warning
  ↓
Normal
```

A monitoring component therefore needs to observe both:

```text
Alarm asserted
Alarm deasserted
```

OpenBMC's threshold alarm logger creates event logs when supported alarms are asserted and informational logs when they are deasserted. citeturn1search1turn1search2

---

## 12. Hysteresis

A value can fluctuate around a threshold:

```text
69.9
70.1
69.8
70.2
69.9
70.1
```

Without appropriate hysteresis, an alarm could repeatedly toggle:

```text
OFF → ON → OFF → ON → OFF
```

This is alarm chatter.

Hysteresis separates the assertion and clearing behavior.

```text
        Alarm ON
           ▲
           │
      assertion
           │
      threshold
           │
         hysteresis
           │
       clear point
           ▼
        Alarm OFF
```

The exact hysteresis mechanism and values are implementation-specific; do not assume every OpenBMC sensor uses the same strategy.

---

## 13. Thresholds Are Not the Same as Faults

A threshold alarm means:

```text
Sensor value crossed a configured boundary.
```

A fault is a broader system concept.

For example:

```text
Critical threshold
       ↓
Alarm
       ↓
Logging / monitoring
       ↓
Possible protective action
```

Therefore:

> **Threshold crossing is a condition. Fault handling is what the system chooses to do about that condition.**

---

## 14. Who Evaluates the Threshold?

There is no requirement that every OpenBMC platform use one universal threshold engine.

Threshold behavior can be implemented by:

- sensor daemons,
- PLDM sensor implementations,
- monitoring applications,
- platform-specific services.

The architecture is:

```text
Sensor Producer
      ↓
Value / Threshold State
      ↓
D-Bus
      ↓
Logging / Monitoring / Control
```

The current `phosphor-fan-presence` project explicitly describes Sensor Monitoring as taking actions based on sensor thresholds and values. citeturn1search0

---

## 15. Real OpenBMC Source — ThresholdAlarmLogger

A useful real implementation is:

```text
openbmc/phosphor-fan-presence/
    sensor-monitor/
        threshold_alarm_logger.cpp
```

It monitors:

```text
xyz.openbmc_project.Sensor.Threshold.Warning
xyz.openbmc_project.Sensor.Threshold.Critical
xyz.openbmc_project.Sensor.Threshold.PerformanceLoss
```

and creates event logs when alarm properties change. citeturn1search1turn1search2

This gives us a real example of:

```text
D-Bus Alarm
     ↓
Consumer
     ↓
Event Log
```

---

## 16. How ThresholdAlarmLogger Watches D-Bus

The source installs D-Bus matches for `PropertiesChanged` under:

```text
/xyz/openbmc_project/sensors
```

for the relevant threshold interfaces.

Conceptually:

```text
Threshold property changes
          ↓
D-Bus PropertiesChanged
          ↓
propertiesChanged()
          ↓
checkProperties()
```

This directly connects to Day 14:

> **D-Bus signals allow one OpenBMC component to react to state changes made by another component.**

---

## 17. Alarm → Event Log

The source maps alarms such as:

```text
WarningAlarmHigh
      ↓
WarningHigh
      ↓
Warning severity
```

and:

```text
CriticalAlarmHigh
      ↓
CriticalHigh
      ↓
Critical severity
```

When the alarm clears, the source uses a clearing status and informational severity. citeturn1search2

The simplified flow is:

```text
PropertiesChanged
       ↓
Alarm changed?
       ↓
     YES
       ↓
createEventLog()
       ↓
Read sensor value
       ↓
Read threshold value
       ↓
Find hardware callout
       ↓
Create logging entry
```

---

## 18. Sensor Value + Threshold Value + Hardware Context

When creating an event, the logger attempts to obtain:

```text
Sensor Value
Threshold Value
Sensor Type
Inventory / callout path
```

For example:

```text
Sensor:
CPU_Temp

Value:
85°C

Threshold:
80°C

Alarm:
CriticalAlarmHigh = true
```

The source also looks for associations such as `inventory` and `chassis` to identify a related hardware/callout path. citeturn1search2

This connects:

```text
Sensor
  +
Threshold
  +
Inventory / FRU
  ↓
Useful Fault Context
```

---

## 19. Why Inventory Callout Matters

Day 22 taught us:

```text
FRU
 ↓
Inventory
```

Day 23 uses that context during fault handling.

Instead of only:

```text
Critical temperature alarm
```

the event can be associated with the hardware object represented by an inventory path.

Conceptually:

```text
Sensor
  ↓
Alarm
  ↓
Inventory association
  ↓
Physical component
```

This is important for serviceability.

---

## 20. Threshold Logging vs Protective Action

These are separate consumers.

### Logging

```text
Threshold Alarm
      ↓
Event Log
```

### Monitoring / Protection

```text
Threshold Alarm
      ↓
Policy
      ↓
Possible action
```

The current `phosphor-fan-presence` Sensor Monitor documentation describes:

```text
HardShutdown
SoftShutdown
```

monitoring, including configurable delays before a power-off action if the alarm remains active. citeturn1search1

Therefore:

> **A critical alarm does not inherently mean immediate shutdown.**

A separate monitoring/control policy must implement that behavior.

---

## 21. Thresholds and Power Control

Connect Day 19 and Day 20:

### Control path

```text
Management Request
       ↓
State Manager
       ↓
Power Control
       ↓
GPIO
       ↓
Hardware
```

### Protection path

```text
Hardware Condition
       ↓
Sensor
       ↓
Threshold
       ↓
Alarm
       ↓
Monitoring Policy
       ↓
Possible Power Action
```

OpenBMC therefore supports both:

```text
CONTROL
Management → Hardware

MONITORING / PROTECTION
Hardware → OpenBMC → Protective Action
```

---

## 22. Thresholds and Redfish

bmcweb is the external management layer.

The conceptual path is:

```text
Sensor Value / Alarm
        ↓
D-Bus
        ↓
bmcweb
        ↓
Redfish
        ↓
Remote Management Client
```

The exact Redfish representation depends on the resource/schema and bmcweb implementation.

The important lesson is:

> **Threshold state is an internal OpenBMC concept; Redfish provides the standardized external management representation.**

---

## 23. Thresholds and IPMI

OpenBMC's IPMI SDR implementation consumes D-Bus sensor threshold properties such as:

```text
CriticalLow
CriticalHigh
WarningLow
WarningHigh
```

to determine sensor range information. citeturn0search12

Thus:

```text
                D-Bus Sensor
                     │
             ┌───────┴───────┐
             ▼               ▼
          Redfish           IPMI
```

One internal sensor model can support multiple management protocols.

---

## 24. Threshold vs Functional State

Do not confuse:

```text
Threshold Alarm
```

with:

```text
Sensor Functional State
```

A sensor can be working correctly while reporting an abnormal value:

```text
Functional = true
Value = 85°C
CriticalAlarmHigh = true
```

That means:

> The sensor is functioning and has detected a critical condition.

This is different from:

> The sensor itself has failed.

OpenBMC documentation distinguishes sensor measurement/threshold state from functional state represented in inventory. citeturn0search4

---

## 25. Scaling Matters

OpenBMC sensor values can use a scale.

For example:

```text
Value = 34625
Scale = -3
```

means:

```text
34625 × 10^-3
= 34.625°C
```

Threshold values can use the corresponding representation. citeturn0search4

When debugging, always inspect:

```text
Value
Scale
Unit

WarningHigh
WarningLow

CriticalHigh
CriticalLow
```

Otherwise, a raw number can easily be misinterpreted.

---

## 26. Virtual Sensors Can Also Have Thresholds

Thresholds are not limited to direct physical sensors.

OpenBMC's virtual-sensor design supports:

```json
"Thresholds":
{
    "CriticalHigh": 90,
    "CriticalLow": 20,
    "WarningHigh": 70,
    "WarningLow": 30
}
```

for calculated sensors. citeturn0search3

The architecture can therefore be:

```text
Sensor A
   +
Sensor B
   ↓
Virtual Sensor
   ↓
Calculated Value
   ↓
Thresholds
```

---

## 27. Thresholds and Cooling

Threshold information can participate in thermal-management architectures.

For example:

```text
Temperature increases
        ↓
Thermal policy
        ↓
Cooling response
        ↓
Fan speed changes
```

OpenBMC's system-porting documentation describes `phosphor-fan-control` as controlling fan speed using conditions such as temperatures. citeturn1search10

However:

> Do not assume that `WarningHigh` directly commands a fan.

The actual cooling algorithm and configuration determine how sensor values and thresholds are used.

---

## 28. Debugging with busctl

First locate the sensor:

```bash
busctl tree <service>     /xyz/openbmc_project/sensors
```

Then inspect it:

```bash
busctl introspect     <service>     /xyz/openbmc_project/sensors/temperature/CPU_Temp
```

Look for:

```text
xyz.openbmc_project.Sensor.Value
xyz.openbmc_project.Sensor.Threshold.Warning
xyz.openbmc_project.Sensor.Threshold.Critical
```

Read a value:

```bash
busctl get-property     <service>     /xyz/openbmc_project/sensors/temperature/CPU_Temp     xyz.openbmc_project.Sensor.Value     Value
```

Then inspect the threshold and alarm properties.

Exact service names are platform dependent.

---

## 29. Debugging Threshold Changes

Because alarms are D-Bus properties, a change can be observed through:

```text
PropertiesChanged
```

A useful general diagnostic tool is:

```bash
busctl monitor
```

The investigation is:

```text
Sensor value changes
       ↓
Threshold condition changes
       ↓
Alarm property changes?
       ↓
PropertiesChanged?
       ↓
Monitoring consumer receives it?
       ↓
Event log created?
```

This is the same event-driven debugging method introduced in Day 14.

---

## 30. Complete Debugging Checklist

If a temperature alarm is missing:

```text
[1] Sensor object exists?
        ↓
[2] Sensor.Value exists?
        ↓
[3] Value is changing?
        ↓
[4] Unit / Scale correct?
        ↓
[5] Warning interface exists?
        ↓
[6] Critical interface exists?
        ↓
[7] Threshold values correct?
        ↓
[8] Alarm properties change?
        ↓
[9] PropertiesChanged emitted?
        ↓
[10] Monitoring/logger service running?
        ↓
[11] Event log created?
        ↓
[12] Inventory association correct?
        ↓
[13] bmcweb/Redfish result correct?
```

Always debug from the bottom upward.

---

## 31. Common Mistake — “Threshold = Alarm”

Incorrect:

```text
WarningHigh = 70
```

does not mean:

```text
WarningAlarmHigh = true
```

Correct:

```text
WarningHigh
    → boundary

WarningAlarmHigh
    → current alarm state
```

---

## 32. Common Mistake — “Critical Means Shutdown”

Not necessarily.

A critical alarm can be consumed by:

```text
Logging
Monitoring
Cooling
Notification
Protective control
```

A shutdown occurs only if the relevant platform/application policy implements it.

---

## 33. Common Mistake — “Every Sensor Has Every Threshold”

Not necessarily.

A sensor may expose:

```text
Warning only
Critical only
High only
Low only
Both high and low
```

depending on its implementation and configuration.

The threshold logger source also handles optional/missing threshold properties. citeturn1search2

Therefore:

> **Inspect the actual D-Bus object instead of assuming every property exists.**

---

## 34. Common Mistake — “bmcweb Evaluates the Threshold”

Not necessarily.

A threshold alarm may already be generated internally before bmcweb sees it.

A cleaner model is:

```text
Sensor Producer
      ↓
Threshold State
      ↓
D-Bus
      ↓
Logging / Monitoring / Control
      ↓
bmcweb / Redfish
```

This is another reason not to think of Redfish as “D-Bus over HTTP.”

---

## 35. Day 17 → Day 23

### Day 17

```text
Sensor Value
     ↓
D-Bus
     ↓
bmcweb
     ↓
Redfish
```

Question:

> **What is the value?**

### Day 23

```text
Sensor Value
     ↓
Threshold
     ↓
Alarm
     ↓
Logging / Monitoring / Control
     ↓
Redfish / IPMI
```

Question:

> **Is the value abnormal, and what should OpenBMC do about it?**

---

## 36. Day 21 → Day 23

Day 21 introduced Inventory.

Day 23 shows why inventory associations matter during fault handling:

```text
Sensor
  │
  ├── Value
  ├── Threshold
  └── inventory association
           │
           ▼
       FRU / Item
```

Now an event can be connected to the physical hardware represented by the inventory model.

---

## 37. Day 22 → Day 23

Day 22:

```text
EEPROM
  ↓
FRU
  ↓
Inventory
```

Day 23:

```text
Sensor
  ↓
Threshold
  ↓
Alarm
  ↓
Inventory / FRU Context
```

Together:

```text
              Hardware
                 │
       ┌─────────┴─────────┐
       ▼                   ▼
      FRU                Sensor
       │                   │
       ▼                   ▼
   Inventory           Threshold
       │                   │
       └─────────┬─────────┘
                 ▼
             Fault/Event
```

---

## 38. Day 19 → Day 23

Day 19 followed the control direction:

```text
Power Request
     ↓
State Management
     ↓
Power Control
     ↓
GPIO
     ↓
Hardware
```

Day 23 adds the protection direction:

```text
Hardware Condition
     ↓
Sensor
     ↓
Threshold
     ↓
Alarm
     ↓
Monitoring Policy
     ↓
Possible Power Action
```

OpenBMC therefore has both:

```text
CONTROL:
Management → Hardware

MONITORING / PROTECTION:
Hardware → OpenBMC → Action / Management
```

---

## 39. Source-Reading Strategy

When investigating threshold behavior in a new OpenBMC platform:

### Step 1 — Find the sensor

```text
/xyz/openbmc_project/sensors/...
```

### Step 2 — Find the value interface

```text
xyz.openbmc_project.Sensor.Value
```

### Step 3 — Find threshold interfaces

```text
Sensor.Threshold.Warning
Sensor.Threshold.Critical
```

### Step 4 — Inspect properties

```text
Value
WarningHigh
WarningLow
CriticalHigh
CriticalLow
WarningAlarmHigh
WarningAlarmLow
CriticalAlarmHigh
CriticalAlarmLow
```

### Step 5 — Search source code

Search for:

```text
WarningAlarmHigh
CriticalAlarmHigh
PropertiesChanged
```

### Step 6 — Find consumers

```text
Alarm
 ↓
Logger
 ↓
Monitoring
 ↓
Control
```

This is the same architecture → source → runtime method developed since Day 13.

---

## 40. Complete End-to-End Example

Assume:

```text
CPU_Temp
Value        = 85°C
WarningHigh  = 70°C
CriticalHigh = 80°C
```

Flow:

```text
CPU Hardware
     ↓
Temperature Sensor
     ↓
Sensor.Value = 85°C
     ↓
Threshold condition
     ↓
CriticalAlarmHigh
     ↓
D-Bus PropertiesChanged
     ↓
ThresholdAlarmLogger / Monitoring
     ↓
Read Value + Threshold + Callout
     ↓
Event Log
     ↓
Possible thermal/protective action
     ↓
Redfish / IPMI
```

This is the complete path from:

> **measurement → abnormal condition → system response**

---

## 41. Complete OpenBMC Monitoring Model

```text
                     HARDWARE
                        │
        ┌───────────────┼────────────────┐
        │               │                │
        ▼               ▼                ▼
      Sensors          FRU             State
        │               │                │
        ▼               ▼                ▼
     Value          Identity          Current State
        │               │                │
        ▼               ▼                │
   Thresholds       Inventory             │
        │               │                │
        └───────┬───────┴────────────────┘
                ▼
              D-Bus
                │
        ┌───────┼────────┐
        ▼       ▼        ▼
     Logging  Control  Management
        │       │        │
        │       │        ▼
        │       │     Redfish/IPMI
        │       │
        │       ▼
        │    Hardware
        ▼
   Serviceability
```

---

## 42. Day 23 Mental Model

Remember these five layers:

### 1. Value

> **What is the sensor measuring?**

### 2. Threshold

> **Where is the boundary?**

### 3. Alarm

> **Has the boundary condition been reached?**

### 4. Event / Monitoring

> **What should OpenBMC do about it?**

### 5. Management

> **How does the outside world learn about it?**

One line:

```text
Value → Threshold → Alarm → Action/Event → Management
```

---

## 43. Key Takeaways

1. A sensor value alone does not say whether a condition is abnormal.
2. Thresholds provide boundaries for interpreting sensor values.
3. OpenBMC commonly models high/low and warning/critical thresholds.
4. `Sensor.Value` provides the measurement.
5. `Sensor.Threshold.Warning` and `.Critical` provide threshold/alarm contracts.
6. `WarningHigh` is a boundary; `WarningAlarmHigh` is an alarm state.
7. `CriticalHigh` is a boundary; `CriticalAlarmHigh` is an alarm state.
8. Threshold crossing and fault handling are different concepts.
9. Threshold alarms can be consumed by logging, monitoring and control components.
10. D-Bus `PropertiesChanged` allows consumers to react to alarm changes.
11. Events can include sensor value, threshold value and inventory/callout context.
12. Critical does not automatically mean shutdown.
13. Not every sensor exposes every threshold interface.
14. Virtual sensors can also have thresholds.
15. Inventory associations connect sensor conditions to physical hardware.
16. Debugging should proceed bottom-up.



---

## 45. References

### OpenBMC Documentation

- `openbmc/docs/architecture/sensor-architecture.md`
- `openbmc/docs/host-management.md`
- `openbmc/docs/development/add-new-system.md`
- `openbmc/docs/designs/virtual-sensors.md`

### OpenBMC D-Bus Interfaces

- `xyz.openbmc_project.Sensor.Value`
- `xyz.openbmc_project.Sensor.Threshold.Warning`
- `xyz.openbmc_project.Sensor.Threshold.Critical`
- `xyz.openbmc_project.Sensor.Threshold.errors`

### OpenBMC Implementations

- `openbmc/phosphor-fan-presence`
  - `sensor-monitor`
  - `threshold_alarm_logger.cpp`
  - `docs/sensor-monitor/README.md`
- `openbmc/phosphor-pid-control`
- `openbmc/phosphor-host-ipmid`

---

# 46. Final Mental Picture

```text
                 SENSOR HARDWARE
                       │
                       ▼
                 Sensor Value
                       │
                       ▼
              ┌─────────────────┐
              │   THRESHOLDS    │
              │                 │
              │ Warning High    │
              │ Warning Low     │
              │ Critical High   │
              │ Critical Low    │
              └────────┬────────┘
                       │
                       ▼
                  Alarm State
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Logging      Monitoring    Control
          │            │            │
          └────────────┼────────────┘
                       ▼
                     D-Bus
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
          Inventory  bmcweb   Services
                       │
                       ▼
                    Redfish
                       │
                       ▼
              REMOTE MANAGEMENT
```

### 🔥 The key idea

> **OpenBMC turns a raw sensor measurement into meaningful system information by comparing it against thresholds, representing the resulting alarm state through D-Bus, and allowing logging, monitoring, control and management components to react to it.**

---

## 🚀 Day 23 Summary

```text
Sensor Value
     ↓
Threshold
     ↓
Alarm
     ↓
D-Bus Event
     ↓
Logging / Monitoring / Control
     ↓
Inventory / FRU Context
     ↓
Redfish / IPMI
```

**From “What is the temperature?” → to “Is the temperature dangerous?” → to “What should OpenBMC do about it?”**

---

*OpenBMC Learning Series — Day 23*
