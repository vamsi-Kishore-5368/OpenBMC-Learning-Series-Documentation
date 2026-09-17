# OpenBMC Learning Series --- Day 24

# OpenBMC Event Logging & Fault Management

### From Fault Condition → D-Bus Event → phosphor-logging → Event Log → Redfish / IPMI

------------------------------------------------------------------------

## 1. Introduction

In Day 23, we learned what happens when a sensor value crosses a
configured threshold. Day 24 answers the next question: once an
important fault condition is detected, how is it recorded, associated
with hardware, managed, and exposed to remote management software?

The central flow is:

``` text
Fault Condition
      ↓
Event Creation
      ↓
phosphor-logging
      ↓
D-Bus Event Log Object
      ↓
Managed Event Record
      ↓
Redfish / IPMI
      ↓
Remote Management
```

## 2. What Is an Event Log?

An Event Log is a structured record of an important event that OpenBMC
chooses to preserve and expose through management interfaces. Events can
represent sensor threshold crossings, fan failures, power-related
faults, platform errors, firmware/software faults, or other
platform-defined conditions.

The important difference from plain text logging is structure. An event
can carry a message, severity, timestamp, additional metadata,
resolution state, and associations to relevant D-Bus/inventory objects.

## 3. Event Log vs Journal Log

These are not the same thing.

**Journal/system logs** are primarily useful for software/runtime
diagnostics such as service startup, D-Bus failures, driver messages,
debugging, and systemd activity. They are commonly inspected with
`journalctl`.

**Event Logs** represent structured platform events/fault records that
can be consumed by OpenBMC management components and external management
interfaces.

Therefore:

``` text
journal log ≠ Event Log
```

The same incident can produce both, but they serve different purposes.

## 4. Why Does OpenBMC Need Structured Event Logs?

Suppose a CPU temperature reaches 92°C while `CriticalHigh` is 90°C. A
useful management record should answer what happened, how severe it was,
when it happened, which hardware is involved, and what extra diagnostic
information is available.

That record can then be consumed by local tools, Redfish clients,
IPMI/SEL mechanisms, and service engineers.

## 5. Day 23 → Day 24

Day 23:

``` text
Sensor Value → Threshold → Alarm / Fault Condition
```

Day 24:

``` text
Alarm / Fault Condition → Event → Event Log → Callout / Metadata → Redfish / IPMI
```

So Day 23 asks: **When is a sensor condition abnormal?** Day 24 asks:
**Once an important event is detected, how is it recorded and exposed?**

## 6. High-Level Architecture

``` text
Hardware
   ↓
Sensor / Platform Monitor
   ↓
Threshold / Fault Condition
   ↓
Event Decision
   ↓
phosphor-logging
   ↓
D-Bus Logging Entry
   ↓
 ┌───────────────┐
 ▼               ▼
Redfish        IPMI/SEL
 ▼               ▼
Remote Mgmt    Remote Mgmt
```

The exact producer and downstream path are platform/configuration
dependent.

## 7. What Is phosphor-logging?

`phosphor-logging` provides OpenBMC mechanisms for event and journal
logging, structured error/event definitions, metadata, and D-Bus
event-log integration. Its current documentation describes Event Logs as
D-Bus interfaces owned by the logging manager under paths such as
`/xyz/openbmc_project/logging/entry/X`.

## 8. The Event Log D-Bus Object

A typical event entry has a path such as:

``` text
/xyz/openbmc_project/logging/entry/1
/xyz/openbmc_project/logging/entry/2
/xyz/openbmc_project/logging/entry/3
```

The core interface is:

``` text
xyz.openbmc_project.Logging.Entry
```

Conceptually:

``` text
Logging Entry
 ├── Message
 ├── Severity
 ├── Timestamp
 ├── Resolved
 └── AdditionalData
```

An entry can also have association-related interfaces.

## 9. Event Log Object Hierarchy

``` text
/xyz
 └── openbmc_project
      └── logging
           └── entry
                ├── 1
                ├── 2
                └── ...
```

Each numbered object represents a separate event record.

## 10. Core Event Information

**Message** describes what happened. **Severity** describes the
importance of the event. **Timestamp** records when it occurred/was
created. **Resolved** can represent lifecycle state where supported.
**AdditionalData** carries event-specific metadata. **Associations** can
connect the event with an inventory object or other relevant D-Bus
object.

## 11. Message vs Severity vs AdditionalData

Think of the fields as three different questions:

``` text
Message        → What happened?
Severity       → How serious is it?
AdditionalData → What extra structured context is available?
```

For example, an event might say
`CPU temperature exceeded critical threshold`, have a critical severity,
and include sensor/threshold/callout metadata.

## 12. What Is AdditionalData?

`AdditionalData` carries structured metadata associated with an event.
The current logging documentation describes key/value-style metadata and
gives callout-related examples such as `CALLOUT_ERRNO` and
`CALLOUT_DEVICE_PATH`.

Do not assume every event contains the same keys. The available metadata
depends on the event definition and producer.

## 13. What Is an Inventory Callout?

A callout connects a fault event with the hardware that should be
investigated.

``` text
Event Log
   ↓
Association / Callout
   ↓
Inventory Item
   ↓
Affected Hardware
```

For example, `Fan2 became nonfunctional` is more useful when the event
can be associated with the inventory object representing Fan2.

## 14. Why Inventory Associations Matter

Day 21 introduced Inventory as the software representation of physical
hardware. Day 24 adds the reverse relationship: an event can point back
to that hardware representation.

``` text
Fault Event → Inventory → Physical Hardware
```

This turns a generic message into a hardware-aware diagnostic record.

## 15. Event Definition vs Event Instance

An **event definition** describes what an event type means: its name,
message, severity, metadata, and generated interfaces.

An **event instance** is one actual occurrence at runtime.

``` text
Definition: CPU critical temperature event
Instance: Entry 42, timestamp ..., sensor value 92°C
```

Definition describes the type; instance records the occurrence.

## 16. YAML Event Definitions

OpenBMC uses YAML definitions as part of its structured event/error
workflow. The current `phosphor-logging` documentation describes
generating C++ event/error types and logging-related definitions from
YAML.

``` text
YAML Definition
      ↓
Code Generation
      ↓
Generated Event Type
      ↓
Application
      ↓
Event Creation
```

## 17. Why Use Event Definitions?

A shared definition avoids different applications inventing inconsistent
messages, severities, and metadata for the same event concept.

``` text
Common Definition
 ├── Message
 ├── Severity
 ├── Metadata
 └── Generated Interfaces
```

This makes structured event handling more predictable.

## 18. Modern Event Creation with lg2

The current `phosphor-logging` documentation recommends the modern
logging API based on `lg2`. A representative example from the project
documentation is:

``` cpp
lg2::commit(
    sdbusplus::event::xyz::openbmc_project::Logging::Cleared(
        "NUMBER_OF_LOGS", count));
```

The key idea is:

``` text
YAML Event Definition
        ↓
Generated Event Type
        ↓
lg2::commit(...)
        ↓
Structured Event
```

The example event is illustrative of the API pattern; the event name and
arguments depend on the actual definition.

## 19. Why lg2::commit() Matters

The structured approach moves event definition into a reusable,
generated type rather than requiring each application to manually
construct every event field.

``` text
YAML → Generated Type → lg2::commit() → Logging Infrastructure
```

The current project README identifies this YAML + generated-code +
`lg2::commit()` approach as the preferred event-creation workflow.

## 20. Legacy D-Bus Create Method

A D-Bus method also exists for creating event logs:

``` text
Service:      xyz.openbmc_project.Logging
Object:       /xyz/openbmc_project/logging
Interface:    xyz.openbmc_project.Logging.Create
Method:       Create
```

The current `phosphor-logging` documentation marks this method as
**deprecated**. It is useful to recognize when reading older code, but
new code should generally follow the modern event-definition and `lg2`
workflow.

## 21. Who Owns the Event Log Objects?

The event-log objects are maintained by the logging-manager component
commonly referred to as `phosphor-log-manager`.

Conceptually:

``` text
Application / Monitor
        ↓
Event Creation
        ↓
Logging Infrastructure
        ↓
phosphor-log-manager
        ↓
/xyz/openbmc_project/logging/entry/X
```

This separates event production from management of the event records.

## 22. Event Producer vs Event Manager

An **event producer** detects or knows about the condition. Examples
include sensor monitors, fan monitors, power-control components, and
platform-specific daemons.

The **event manager** maintains event records.

``` text
Producer → Event → Logging Infrastructure → Event Entry
```

The producer does not have to own the complete event database lifecycle.

## 23. Day 23 Real-Source Connection

In Day 23 we studied `threshold_alarm_logger.cpp`. It monitors sensor
alarm state changes and can create logging events.

The conceptual bridge is:

``` text
Sensor.Value
     +
Threshold Alarm
     ↓
ThresholdAlarmLogger
     ↓
Event Logging
```

The exact action remains dependent on the monitoring implementation and
configuration.

## 24. PropertiesChanged → Event Logging

A common D-Bus pattern is:

``` text
Sensor
  ↓
PropertiesChanged
  ↓
Alarm property changes
  ↓
Monitoring component
  ↓
Event creation
```

For a threshold event, a change such as `CriticalAlarmHigh` can be
observed by a monitoring/logging component.

## 25. Event Logging Does Not Mean Every Alarm Becomes an Event

Not every sensor transition should create a persistent event. Otherwise
a frequently changing sensor could generate an unbounded number of
records.

Therefore event creation is policy/application dependent.

``` text
Normal value change → usually no Event Log
Threshold crossing → possible event
Critical failure    → possible event
Repeated condition  → platform-specific handling
```

## 26. Event Logging vs Protective Action

Recording a fault and reacting to a fault are separate concerns.

``` text
Critical condition
       │
 ┌─────┴─────┐
 ▼           ▼
Log Event   Protective Action
 ▼           ▼
Record      Cooling / Shutdown /
Fault       Power or other policy
```

An Event Log entry does not itself imply that a shutdown or other
protective action will occur.

## 27. Severity

Severity communicates the importance of an event. Conceptually, events
can range from informational conditions through warnings/errors to
critical conditions.

Do not confuse severity with a sensor threshold or an alarm property:

``` text
Threshold → boundary
Alarm     → current condition
Severity  → importance of recorded event
```

The exact enum/mapping is defined by the relevant interface and
management layer.

## 28. Threshold vs Alarm vs Event

These three terms are closely related but not interchangeable.

``` text
Threshold
   ↓
Alarm
   ↓
Event
```

Example:

``` text
CriticalHigh = 90°C
Value = 92°C
CriticalAlarmHigh = true
Event = CPU temperature exceeded critical threshold
```

The threshold is the boundary, the alarm represents the condition, and
the event is the recorded occurrence.

## 29. Event Lifecycle

A simplified lifecycle is:

``` text
Condition detected
       ↓
Event created
       ↓
Event entry managed
       ↓
Event exposed to management
       ↓
Condition may recover
       ↓
Event may become resolved
       ↓
Event may later be deleted according to policy
```

Not every event automatically follows every step.

## 30. Resolved vs Deleted

These are different operations.

**Resolved** means the record can remain while indicating that the
underlying condition has been resolved, where supported.

**Deleted** means the event record itself is removed.

Therefore:

``` text
Resolved ≠ Deleted
```

## 31. Event Storage and Retention

BMC storage is limited, so event logs require retention/clearing
policies. The exact policy depends on the image and platform and can
involve maximum entries, clearing, overwriting, or other
storage-management behavior.

Do not assume every platform has the same retention policy.

## 32. Event Log → Redfish

bmcweb provides the Redfish management interface. Its current Redfish
documentation describes EventLog entries under a LogService and
documents two possible EventLog implementations:

1.  A default journal/rsyslog-backed implementation.
2.  An optional D-Bus-backed implementation that reads
    `phosphor-logging` D-Bus log entries when
    `BMCWEB_ENABLE_REDFISH_DBUS_LOG_ENTRIES` is enabled.

The two implementations are mutually exclusive.

## 33. Redfish Event Log Model

A common OpenBMC Redfish layout is:

``` text
/redfish/v1/
  Systems/
    system/
      LogServices/
        EventLog/
          Entries/
```

A LogEntry can expose fields such as `Message`, `Created`, `EntryType`,
`Severity`, `Resolved`, and `AdditionalDataURI` depending on the
implementation/schema.

The actual location can vary. bmcweb also supports a Manager-based
EventLog location, so always inspect the target BMC's Service Root and
LogService resources.

## 34. Redfish Translation

``` text
D-Bus Event Log
      ↓
    bmcweb
      ↓
Redfish LogEntry
      ↓
     JSON
      ↓
Remote Client
```

This is a translation between the internal OpenBMC/D-Bus model and the
external Redfish data model. It is not simply "D-Bus over HTTP."

## 35. Event Log → IPMI

OpenBMC also has IPMI-related event logging mechanisms.
`phosphor-sel-logger` can monitor selected event types and log SEL
records, including configurable threshold-event monitoring.

Conceptually:

``` text
Platform / Threshold Event
        ↓
phosphor-sel-logger
        ↓
IPMI SEL
        ↓
ipmitool / Remote IPMI Client
```

Do not assume every `phosphor-logging` event automatically becomes an
IPMI SEL record; the path depends on the event source and platform
configuration.

## 36. Event Log → Multiple Management Views

``` text
             Structured Event
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
   OpenBMC Logging       SEL-related path
          │                   │
          ▼                   ▼
       Redfish              IPMI
          │                   │
          └─────────┬─────────┘
                    ▼
             Remote Management
```

Different management protocols can represent the same underlying
platform condition differently.

## 37. Example: Temperature Fault

Consider:

``` text
CPU_Temp = 92°C
CriticalHigh = 90°C
```

The conceptual sequence is:

``` text
Sensor.Value = 92
       ↓
CriticalHigh = 90
       ↓
CriticalAlarmHigh = true
       ↓
Monitoring Component
       ↓
Structured Event
       ↓
phosphor-logging
       ↓
/xyz/openbmc_project/logging/entry/X
       ↓
Inventory Association
       ↓
Redfish / IPMI where configured
```

An event may contain the message, severity, timestamp, sensor/threshold
metadata, and a hardware association.

## 38. Example: Fan Failure

Suppose Fan2 has a target of 8000 RPM but feedback falls to zero.

``` text
Fan feedback
     ↓
Fan monitoring
     ↓
Nonfunctional condition
     ↓
Event logging
     ↓
Callout → Fan2 inventory object
     ↓
Redfish / IPMI where configured
```

The important point is the connection between the fault and the
inventory representation of the affected hardware.

## 39. Event Logging and Inventory

Day 21 introduced inventory as the software representation of physical
hardware. Day 24 adds an event relationship:

``` text
Physical Hardware
      ↓
Inventory Item
      ↑
      │ association/callout
      │
   Event Log
```

This is the foundation of hardware-aware diagnostics.

## 40. Debugging: Discover Logging Objects

Start with the actual services on the BMC:

``` bash
busctl list
```

Then inspect logging-related objects if the relevant service is exposed:

``` bash
busctl tree xyz.openbmc_project.Logging
```

Do not blindly assume the service name on every image. Discover it first
when necessary.

## 41. Debugging: Inspect an Event Entry

Once an entry is identified:

``` bash
busctl introspect \
    xyz.openbmc_project.Logging \
    /xyz/openbmc_project/logging/entry/1
```

This lets you see the interfaces and properties actually exposed by that
object on the target BMC.

## 42. Debugging: Read Event Properties

For example:

``` bash
busctl get-property \
    xyz.openbmc_project.Logging \
    /xyz/openbmc_project/logging/entry/1 \
    xyz.openbmc_project.Logging.Entry \
    Message
```

And:

``` bash
busctl get-property \
    xyz.openbmc_project.Logging \
    /xyz/openbmc_project/logging/entry/1 \
    xyz.openbmc_project.Logging.Entry \
    Severity
```

Use `busctl introspect` first if you are unsure which
interfaces/properties are present.

## 43. Debugging: Inspect the Journal

The journal is different from the Event Log, but it is essential when
debugging the logging subsystem and event producer.

``` bash
journalctl
```

If the target image exposes the relevant service unit:

``` bash
journalctl -u <logging-service>
```

Discover actual unit names with:

``` bash
systemctl list-units | grep -i log
```

## 44. Debugging: OpenBMC Helpers

`obmcutil` contains helper functionality for listing, showing, and
deleting logging entries on supported images. Its implementation
demonstrates discovery of `xyz.openbmc_project.Logging.Entry` objects
through ObjectMapper and reading their properties.

The exact command availability and image behavior should be checked on
the target BMC.

## 45. Debugging: Event Exists but No Redfish Entry

If:

``` text
/xyz/openbmc_project/logging/entry/X
```

exists but the event is not visible through Redfish, inspect the bmcweb
EventLog implementation and build configuration.

The current bmcweb documentation explicitly distinguishes the
journal/rsyslog-backed and D-Bus-backed implementations. Therefore D-Bus
event existence and Redfish EventLog visibility are related but not
identical.

## 46. Debugging: Event Exists but No Hardware Callout

If the event exists but does not identify the affected component,
inspect:

-   event metadata,
-   callout-related AdditionalData,
-   inventory object paths,
-   association definitions,
-   ObjectMapper discovery,
-   the event producer.

The logging manager may be functioning correctly while the producer or
association information is incomplete.

## 47. Event Logging and ObjectMapper

ObjectMapper provides dynamic D-Bus discovery and association-related
operations throughout OpenBMC.

For event/hardware relationships, the conceptual path is:

``` text
Event
 ↓
Association / Inventory Path
 ↓
ObjectMapper / D-Bus Discovery
 ↓
Actual Service / Object
```

This follows the same D-Bus architecture used elsewhere in the series.

## 48. Event Logging Is Part of Fault Management

Fault management is broader than writing a log entry.

``` text
Detect
  ↓
Classify
  ↓
Record
  ↓
Associate
  ↓
Expose
  ↓
React
  ↓
Recover / Resolve
```

In OpenBMC these stages can involve sensors, monitoring components,
event definitions, phosphor-logging, inventory associations, bmcweb,
IPMI/SEL mechanisms, and platform control.

Not every event passes through every stage.

## 49. Complete End-to-End Example

Combine Days 17, 21, 23, and 24:

``` text
CPU Sensor
    ↓
Sensor.Value
    ↓
CriticalHigh
    ↓
CriticalAlarmHigh
    ↓
Monitoring Component
    ↓
Structured Event
    ↓
phosphor-logging
    ↓
Logging.Entry
    ↓
Inventory Association
    ↓
Redfish / IPMI
```

The important transition is:

``` text
Measured condition → diagnosed event → hardware-aware record → management view
```

## 50. How to Read Real Event-Logging Source Code

Use this sequence rather than reading an entire repository line-by-line:

``` text
1. Find the event producer
2. Find the trigger/callback
3. Find the alarm or fault decision
4. Find lg2::commit() or the older logging mechanism
5. Find the YAML event definition
6. Find callout/association handling
7. Find the resulting Logging.Entry object
8. Trace Redfish or IPMI exposure
```

Useful search terms include:

``` text
PropertiesChanged
CriticalAlarmHigh
CriticalAlarmLow
WarningAlarmHigh
WarningAlarmLow
lg2::commit
CALLOUT_
Association.Definitions
Logging.Entry
```

## 51. Important Architecture Distinction

Do not model the architecture as a universal direct connection:

``` text
Sensor → Event Log
```

A more accurate model is:

``` text
Sensor
  ↓
D-Bus Sensor State
  ↓
Monitoring / Policy Component
  ↓
Event-Creation Decision
  ↓
Logging Subsystem
```

The same sensor condition can have different policies: log only, log
plus warning, log plus cooling action, log plus shutdown, or feed an
IPMI SEL path.

## 52. Event Logging vs Telemetry

Telemetry asks:

``` text
What is the system measuring?
```

Example:

``` text
CPU temperature = 72°C
```

Event logging asks:

``` text
What important event occurred?
```

Example:

``` text
CPU temperature crossed critical threshold
```

Therefore:

``` text
Telemetry   → measurement/data
Event Log   → significant recorded event
```

They complement each other.

## 53. Event Logging vs Inventory

Inventory answers:

``` text
What hardware exists?
```

Event logging answers:

``` text
What important event happened to that hardware?
```

Together:

``` text
Inventory + Event → Hardware-aware fault management
```

## 54. Event Logging vs Redfish

Keep the layers separate:

``` text
D-Bus
→ Internal OpenBMC communication

Event Log
→ Structured internal fault/event record

bmcweb
→ External management interface / translator

Redfish
→ Standardized external management model
```

So the management path can be:

``` text
Internal Event
   ↓
D-Bus / Logging
   ↓
bmcweb
   ↓
Redfish EventLog
   ↓
Remote Client
```

## 55. Practical Command Checklist

``` bash
# Discover services
busctl list

# Inspect logging tree when available
busctl tree xyz.openbmc_project.Logging

# Inspect an event entry
busctl introspect \
  xyz.openbmc_project.Logging \
  /xyz/openbmc_project/logging/entry/1

# Read message
busctl get-property \
  xyz.openbmc_project.Logging \
  /xyz/openbmc_project/logging/entry/1 \
  xyz.openbmc_project.Logging.Entry Message

# Read severity
busctl get-property \
  xyz.openbmc_project.Logging \
  /xyz/openbmc_project/logging/entry/1 \
  xyz.openbmc_project.Logging.Entry Severity

# Inspect runtime logs
journalctl

# Discover log-related services
systemctl list-units | grep -i log

# Inspect Redfish LogServices
curl -k https://<BMC-IP>/redfish/v1/Systems/system/LogServices/
```

The actual D-Bus service names, paths, and Redfish locations must be
checked on the target image.

## 56. Common Mistakes

**Mistake 1:** Every threshold automatically creates an Event Log. → Not
necessarily; event creation is policy/application dependent.

**Mistake 2:** Event Log and journal are the same. → They are different
logging mechanisms.

**Mistake 3:** Every Event Log automatically appears in Redfish. → Not
necessarily; bmcweb configuration matters.

**Mistake 4:** Every Event Log becomes an IPMI SEL record. → Not
necessarily; SEL has its own path/configuration.

**Mistake 5:** Resolved means deleted. → No: `Resolved ≠ Deleted`.

**Mistake 6:** The logging manager detects every fault. → No; event
producers/monitoring components generally detect or decide what should
be logged.

## 57. Day 24 Mental Model

``` text
CONDITION
   ↓
DETECTION
   ↓
EVENT DECISION
   ↓
STRUCTURED EVENT
   ↓
phosphor-logging
   ↓
Logging.Entry / D-Bus
   ↓
Inventory Association
   ↓
 ┌─────────┴─────────┐
 ▼                   ▼
Redfish            IPMI/SEL
 ▼                   ▼
Remote Mgmt       Remote Mgmt
```

## 58. Day 17 → Day 24 Learning Journey

``` text
Day 17 → Sensor Data → D-Bus → Redfish
Day 21 → Physical Hardware → Inventory → D-Bus
Day 22 → EEPROM → FRU → Inventory
Day 23 → Sensor Value → Threshold → Alarm
Day 24 → Alarm/Fault → Event → Logging → Callout → Redfish/IPMI
```

The series is moving from observing the system to understanding,
recording, and managing faults.

## 59. The Bigger OpenBMC Fault-Management Picture

``` text
                    HARDWARE
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
          Sensors             Identity
             │                   │
             ▼                   ▼
        Sensor.Value           FRU
             │                   │
             ▼                   ▼
        Threshold            Inventory
             │                   │
             ▼                   │
           Alarm ◄───────────────┘
             │
             ▼
           Event
             │
             ▼
      phosphor-logging
             │
       ┌─────┴─────┐
       ▼           ▼
    Redfish       IPMI
       │           │
       └─────┬─────┘
             ▼
      Remote Management
```

## 60. Key Takeaways

-   Event Logs are structured records of important events.
-   Event Logs are different from journal logs.
-   `phosphor-logging` provides the event logging infrastructure and
    structured event workflow.
-   Event entries are represented on D-Bus under
    `/xyz/openbmc_project/logging/entry/X`.
-   `xyz.openbmc_project.Logging.Entry` is the core entry interface.
-   `AdditionalData` carries event-specific metadata.
-   Associations can connect an event to an inventory object.
-   YAML event definitions support structured event/error generation.
-   `lg2::commit()` is the modern event-creation mechanism documented by
    `phosphor-logging`.
-   The older `Logging.Create` D-Bus method is deprecated.
-   `phosphor-log-manager` maintains the D-Bus event log objects.
-   Redfish and IPMI provide external management views, but their exact
    paths/configuration are implementation dependent.
-   Event logging and protective action are separate concerns.
-   `Resolved` and `Deleted` are different lifecycle concepts.

## 61. One-Line Summary

> **OpenBMC Event Logging converts important fault conditions into
> structured, hardware-aware event records that can be managed
> internally over D-Bus and exposed through external management
> interfaces such as Redfish and IPMI.**

## 62. Final Mental Model

``` text
Sensor / Hardware Condition
            ↓
      Threshold / Fault
            ↓
        Event Decision
            ↓
      Structured Event
            ↓
      phosphor-logging
            ↓
     Logging.Entry / D-Bus
            ↓
    Inventory Association
            ↓
       ┌────┴────┐
       ▼         ▼
    Redfish    IPMI/SEL
       │         │
       └────┬────┘
            ▼
    Remote Management
```

## 63. What's Next?

We have now connected:

``` text
Sensors → Thresholds → Fault Detection → Event Logging → Inventory Association → Redfish / IPMI
```

A natural next step is **IPMI and SEL in greater depth**:

``` text
Sensor / Platform Event
        ↓
      IPMI
        ↓
     SEL Record
        ↓
 Sensor Number / Event Type
        ↓
    ipmitool sel
```

This will connect the modern D-Bus/OpenBMC architecture with the
traditional server-management model used by IPMI.

## 64. References

1.  OpenBMC `phosphor-logging`:
    https://github.com/openbmc/phosphor-logging
2.  OpenBMC `phosphor-logging` README:
    https://github.com/openbmc/phosphor-logging/blob/master/README.md
3.  OpenBMC `phosphor-fan-presence`:
    https://github.com/openbmc/phosphor-fan-presence
4.  OpenBMC `bmcweb`: https://github.com/openbmc/bmcweb
5.  OpenBMC `bmcweb` Redfish documentation:
    https://github.com/openbmc/bmcweb/blob/master/docs/Redfish.md
6.  OpenBMC Host Management documentation:
    https://github.com/openbmc/docs/blob/master/host-management.md
7.  OpenBMC ObjectMapper architecture:
    https://github.com/openbmc/docs/blob/master/architecture/object-mapper.md
8.  OpenBMC architecture/interface overview:
    https://github.com/openbmc/docs/blob/master/architecture/interface-overview.md
9.  OpenBMC `phosphor-state-manager`:
    https://github.com/openbmc/phosphor-state-manager
10. OpenBMC `phosphor-sel-logger`:
    https://github.com/openbmc/meta-phosphor/tree/master/recipes-phosphor/sel-logger

------------------------------------------------------------------------

# End of Day 24

### OpenBMC Learning Series

**From Fault Condition → Event → Logging → Hardware Association →
Redfish / IPMI**

> Detect the problem.\
> Record it structurally.\
> Associate it with hardware.\
> Expose it to management software.

**Learn. Build. Go Deeper.**
