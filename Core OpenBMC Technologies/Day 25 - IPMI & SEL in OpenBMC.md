# OpenBMC Learning Series — Day 25

# IPMI & SEL in OpenBMC

### From Sensor / Fault Event → phosphor-sel-logger → SEL Record → IPMI → ipmitool

---

## 1. Introduction

In Day 24, we followed the path:

```text
Fault Condition
      ↓
Event Decision
      ↓
phosphor-logging
      ↓
D-Bus Event Log
      ↓
Inventory Association
      ↓
Redfish / IPMI
```

Day 25 goes deeper into the **traditional IPMI event path**.

The focus is:

```text
Sensor / Fault Condition
        ↓
     Event
        ↓
phosphor-sel-logger
        ↓
     SEL Record
        ↓
IPMI SEL Repository / Interface
        ↓
      ipmitool
        ↓
Remote Administrator
```

The important idea is:

> **SEL is the IPMI representation of system events.**

OpenBMC can connect modern D-Bus-based sensor/event infrastructure with this traditional server-management model.

---

# 2. What Is IPMI?

**IPMI — Intelligent Platform Management Interface** is a standardized interface for monitoring and managing computer systems independently of the host operating system.

The BMC can provide management functions such as:

- sensor access,
- chassis power control,
- event logging,
- FRU information,
- management over a network,
- host-facing IPMI interfaces.

OpenBMC implements a subset of the IPMI specification.

OpenBMC's architecture documentation distinguishes:

```text
Network / Out-of-band IPMI
        ↓
      RMCP+
        ↓
     netipmid

Host / In-band IPMI
        ↓
      ipmid
```

These are different paths into the BMC.

---

# 3. Why Does IPMI Still Matter?

Modern systems increasingly use Redfish.

However, IPMI remains important because:

- many server-management tools understand it,
- existing data-center infrastructure uses it,
- administrators use `ipmitool`,
- legacy platforms depend on IPMI semantics,
- some BMC workflows still expose IPMI-compatible interfaces.

Therefore an OpenBMC engineer should understand both:

```text
Modern management
        ↓
      Redfish
```

and:

```text
Traditional management
        ↓
       IPMI
```

---

# 4. IPMI in the OpenBMC Architecture

A simplified model is:

```text
                       BMC
                        │
       ┌────────────────┼────────────────┐
       │                │                │
       ▼                ▼                ▼
    Redfish           IPMI           D-Bus
       │                │                │
    bmcweb        ┌─────┴─────┐        │
                  │           │        │
             Network IPMI  Host IPMI  │
                  │           │        │
               netipmid      ipmid     │
```

D-Bus is an internal communication mechanism.

Redfish and IPMI are management interfaces exposed to clients.

---

# 5. What Is SEL?

**SEL = System Event Log**

The SEL records system-management events.

Examples:

```text
Temperature threshold exceeded
Fan failure
Voltage threshold event
Power event
Watchdog event
Platform-specific hardware event
```

A conceptual SEL record is:

```text
Record ID
Timestamp
Generator ID
Record Type
Sensor Information
Event Direction
Event Data
```

The exact fields depend on the SEL record type.

---

# 6. SEL Is Not the Same as a Journal

This distinction is extremely important.

### Journal

Used primarily for system/software logging:

```text
service started
D-Bus error
daemon restarted
driver error
debug information
```

Access:

```bash
journalctl
```

### SEL

Used for IPMI system-event records:

```text
CPU temperature threshold exceeded
Fan failure
Voltage alarm
```

Access:

```bash
ipmitool sel list
```

Conceptually:

```text
Journal
  → software/runtime diagnostics

SEL
  → IPMI system-event records
```

OpenBMC can store SEL records in journal-backed infrastructure, but that does not make every journal message an SEL record.

---

# 7. OpenBMC phosphor-sel-logger

The OpenBMC project provides:

```text
phosphor-sel-logger
```

Its purpose is to handle requests to add IPMI SEL records and to monitor configured event types for automatic SEL logging.

The project documentation states that SEL records stored through its logging path are identified using:

```text
MESSAGE_ID
```

with additional IPMI-specific metadata.

Source:

https://github.com/openbmc/phosphor-sel-logger

---

# 8. Core phosphor-sel-logger Metadata

The project documents metadata including:

```text
IPMI_SEL_RECORD_ID
IPMI_SEL_RECORD_TYPE
IPMI_SEL_GENERATOR_ID
IPMI_SEL_SENSOR_PATH
IPMI_SEL_EVENT_DIR
IPMI_SEL_DATA
```

Conceptually:

```text
SEL Record
   │
   ├── Record ID
   ├── Record Type
   ├── Generator ID
   ├── Sensor Path
   ├── Event Direction
   └── Event Data
```

This is much more structured than an ordinary text message.

---

# 9. What Is a SEL Record ID?

The SEL Record ID identifies a particular SEL record.

In the OpenBMC SEL Logger metadata it is represented as:

```text
IPMI_SEL_RECORD_ID
```

The documented representation is a two-byte unique SEL Record ID.

Example concept:

```text
Record 1
Record 2
Record 3
...
```

The actual allocation and persistence behavior depend on the implementation/configuration.

---

# 10. What Is Record Type?

The SEL record type determines how the remaining record data should be interpreted.

The SEL Logger documents:

```text
IPMI_SEL_RECORD_TYPE
```

for distinguishing system and OEM records.

Conceptually:

```text
Record Type
    │
    ├── System Event
    │
    └── OEM Event
```

System records follow standardized IPMI structures.

OEM records provide vendor/platform-specific encoding.

---

# 11. What Is Generator ID?

The Generator ID identifies the source/generator of the event.

OpenBMC's SEL Logger documentation describes:

```text
IPMI_SEL_GENERATOR_ID
```

as the IPMI Generator ID, usually associated with the IPMB slave address of the requester.

Think of it as:

```text
Who generated this event?
```

It is part of the IPMI representation of the event.

---

# 12. What Is Sensor Path?

OpenBMC internally represents sensors using D-Bus object paths.

For example:

```text
/xyz/openbmc_project/sensors/temperature/CPU_Temp
```

The SEL Logger metadata can preserve the relevant sensor path:

```text
IPMI_SEL_SENSOR_PATH
```

This creates an important bridge:

```text
D-Bus Sensor
      ↓
Sensor Object Path
      ↓
SEL Event
```

This is one of the places where OpenBMC's D-Bus architecture meets IPMI.

---

# 13. What Is Event Direction?

An event can have a direction:

```text
Assert
Deassert
```

### Assert

The fault/condition became active.

Example:

```text
Temperature crossed critical threshold
```

### Deassert

The condition returned to the non-fault state.

Example:

```text
Temperature returned below the threshold
```

OpenBMC records this through:

```text
IPMI_SEL_EVENT_DIR
```

---

# 14. What Is Event Data?

SEL events contain event-specific data.

The SEL Logger interface supports:

### System events

Up to three bytes of SEL data.

### OEM events

Up to thirteen bytes depending on the record type.

The raw event data is represented in the logging metadata as:

```text
IPMI_SEL_DATA
```

The meaning of those bytes depends on the record format.

---

# 15. SEL System Event Record

A simplified conceptual structure is:

```text
┌──────────────────────────────┐
│ Record ID                    │
├──────────────────────────────┤
│ Record Type                  │
├──────────────────────────────┤
│ Timestamp                   │
├──────────────────────────────┤
│ Generator ID                 │
├──────────────────────────────┤
│ Event Message Revision       │
├──────────────────────────────┤
│ Sensor Type                  │
├──────────────────────────────┤
│ Sensor Number                │
├──────────────────────────────┤
│ Event Direction / Event Type │
├──────────────────────────────┤
│ Event Data 1                 │
├──────────────────────────────┤
│ Event Data 2                 │
├──────────────────────────────┤
│ Event Data 3                 │
└──────────────────────────────┘
```

This is a conceptual representation; exact wire/storage encoding follows the applicable IPMI record definition.

---

# 16. Sensor Number vs D-Bus Sensor Path

This is one of the most important OpenBMC concepts.

Modern OpenBMC internally uses D-Bus paths:

```text
/xyz/openbmc_project/sensors/temperature/CPU_Temp
```

IPMI traditionally uses:

```text
Sensor Number = 0x##
```

Therefore there must be a mapping.

Conceptually:

```text
D-Bus Sensor
      │
      │ mapping
      ▼
IPMI Sensor Number
      │
      ▼
SEL Record
```

The OpenBMC platform configuration contains IPMI sensor mappings.

---

# 17. IPMI Sensor Inventory

OpenBMC uses configuration data to map sensors to IPMI identities.

A simplified example:

```yaml
0x08:
  sensorType: 0x07
  path: /system/chassis/motherboard/cpu0
```

The exact mappings are platform-specific.

This means:

```text
Sensor Number
      ↓
Sensor Type
      ↓
D-Bus / Inventory Path
```

The OpenBMC development documentation explicitly notes that IPMI sensor inventory differs between systems and must be defined for the target platform.

---

# 18. Why the Mapping Is Necessary

Suppose D-Bus knows:

```text
CPU_Temp
```

but IPMI expects:

```text
Sensor Number = 0x08
```

An IPMI client does not necessarily understand the internal D-Bus object name.

So OpenBMC needs:

```text
CPU_Temp
    ↓
IPMI mapping
    ↓
0x08
```

This lets:

```bash
ipmitool sensor
```

and:

```bash
ipmitool sel list
```

use traditional IPMI identifiers.

---

# 19. phosphor-sel-logger D-Bus Interface

The SEL Logger exposes an IPMI logging interface at:

```text
Service:
xyz.openbmc_project.Logging.IPMI

Path:
/xyz/openbmc_project/Logging/IPMI

Interface:
xyz.openbmc_project.Logging.IPMI
```

The interface definition is maintained in:

```text
phosphor-dbus-interfaces
```

The interface provides methods for adding System and OEM SEL records.

---

# 20. IpmiSelAdd

The SEL Logger source registers an operation conceptually equivalent to:

```text
IpmiSelAdd
```

The System-event path accepts information including:

```text
message
sensor path
SEL data
assert/deassert
generator ID
```

The daemon adds the appropriate SEL metadata.

This gives us:

```text
Application
     ↓
D-Bus
     ↓
IpmiSelAdd
     ↓
phosphor-sel-logger
     ↓
SEL Record
```

---

# 21. IpmiSelAddOem

For OEM records the interface provides:

```text
IpmiSelAddOem
```

The OEM path uses:

```text
message
SEL data
record type
```

The record-specific interpretation is determined by the OEM record format.

Therefore:

```text
System SEL
→ standardized system-event structure

OEM SEL
→ platform/vendor-specific structure
```

---

# 22. Automatic Event Monitoring

SEL Logger does not only support manually requested records.

It can also monitor selected event types.

For example, the project documents a:

```text
Threshold Event Monitor
```

which matches:

```text
PropertiesChanged
```

on:

```text
xyz.openbmc_project.Sensor.Threshold
```

It then examines newly asserted threshold events and can generate SEL records.

The conceptual path is:

```text
Sensor Threshold
       ↓
PropertiesChanged
       ↓
SEL Logger Monitor
       ↓
Threshold Event
       ↓
SEL Record
```

---

# 23. This Connects Directly to Day 23

Day 23:

```text
Sensor.Value
      ↓
Threshold
      ↓
Alarm
```

Day 25:

```text
Threshold Event
      ↓
PropertiesChanged
      ↓
phosphor-sel-logger
      ↓
SEL
```

Therefore the complete path becomes:

```text
Hardware
   ↓
Sensor
   ↓
Threshold
   ↓
Alarm
   ↓
D-Bus signal
   ↓
SEL Logger
   ↓
SEL Record
```

This is one of the most important connections in the series.

---

# 24. SEL Logger Monitoring Is Configurable

Do not assume every BMC automatically logs every event.

The OpenBMC Yocto recipe documents configuration options for event monitoring, including:

```text
log-threshold
log-pulse
log-watchdog
log-alarm
log-host
```

The recipe also states that event monitoring is disabled by default and can be enabled through configuration.

Therefore:

```text
Sensor Event
      ≠
Guaranteed SEL Record
```

The platform build determines which monitoring features are enabled.

---

# 25. Yocto Configuration

The OpenBMC recipe contains PACKAGECONFIG options for SEL monitoring.

Conceptually:

```text
PACKAGECONFIG
      │
      ├── log-threshold
      ├── log-pulse
      ├── log-watchdog
      ├── log-alarm
      ├── log-host
      ├── send-to-logger
      └── sel-delete
```

This is an important lesson for OpenBMC development:

> A feature existing in a repository does not mean it is enabled in every BMC image.

The final image is determined by Yocto configuration.

---

# 26. send-to-logger

The recipe documents:

```text
send-to-logger
```

as a configuration option that changes how SEL logging is integrated.

The current implementation can use:

```text
phosphor-logging
```

or the alternative logging mechanism described by the project.

This is another reason to inspect the actual target image rather than assuming one universal implementation.

---

# 27. SEL and phosphor-logging

Day 24 introduced:

```text
phosphor-logging
```

Day 25 introduces:

```text
phosphor-sel-logger
```

They are related, but they are not identical components.

Conceptually:

```text
phosphor-logging
       │
       │ structured OpenBMC events
       │
       └─────────────┐
                     │
                     ▼
              Management logs


phosphor-sel-logger
       │
       │ IPMI SEL semantics
       ▼
    SEL Record
```

Depending on build/configuration, SEL records can be integrated with the broader logging infrastructure.

---

# 28. Important Correction: Event Log ≠ SEL

Do not say:

> Every phosphor-logging event is an IPMI SEL record.

That is not generally correct.

Instead:

```text
OpenBMC Event
      │
      ├── phosphor-logging path
      │
      └── SEL logging path
             │
             └── when configured/appropriate
```

A platform can expose different event information through different management interfaces.

---

# 29. IPMI SEL Interface vs IPMI Network Interface

These are also different concepts.

### SEL interface

```text
/xyz/openbmc_project/Logging/IPMI
```

is an internal D-Bus interface used by the SEL logger.

### IPMI network interface

A remote client may communicate using:

```text
RMCP+
```

through the network IPMI implementation.

So:

```text
D-Bus IPMI SEL interface
          ≠
Network IPMI protocol
```

The first is internal BMC IPC.

The second is an external management protocol.

---

# 30. Out-of-Band IPMI

OpenBMC documentation describes network IPMI as:

```text
RMCP+
```

and identifies:

```text
netipmid
```

as the network-facing implementation.

A remote administrator can use:

```bash
ipmitool -I lanplus ...
```

to communicate with the BMC.

Conceptually:

```text
Remote PC
   ↓
ipmitool
   ↓
RMCP+
   ↓
BMC Network
   ↓
netipmid
   ↓
IPMI handlers
   ↓
BMC resources
```

---

# 31. Host / In-Band IPMI

OpenBMC also has host-facing IPMI.

The OpenBMC architecture documentation identifies:

```text
phosphor-host-ipmid
```

as the host-endpoint IPMI implementation.

Conceptually:

```text
Host OS
   ↓
Host IPMI interface
   ↓
phosphor-host-ipmid
   ↓
D-Bus / BMC services
```

This is different from:

```text
Remote network IPMI
```

---

# 32. IPMI Command Flow

A simplified remote command:

```bash
ipmitool -I lanplus \
    -H <BMC-IP> \
    -U <USER> \
    -P <PASSWORD> \
    sel list
```

flows approximately as:

```text
ipmitool
    ↓
IPMI-over-LAN
    ↓
BMC network IPMI service
    ↓
IPMI command handler
    ↓
SEL implementation
    ↓
SEL records
    ↓
Response
    ↓
ipmitool
```

The exact internal call path depends on the command and implementation.

---

# 33. `ipmitool sel list`

The most useful first command is:

```bash
ipmitool sel list
```

It displays the current SEL entries.

Typical conceptual output:

```text
ID   Date/Time            | Sensor      | Event
1    09/18/2026 10:15:22  | CPU Temp    | Upper Critical
2    09/18/2026 10:18:41  | Fan2        | Lower Critical
```

The exact formatting depends on `ipmitool` and the BMC implementation.

---

# 34. `ipmitool sel elist`

For a more detailed event representation:

```bash
ipmitool sel elist
```

This can provide additional decoded information.

Use it when you need more than a basic event listing.

---

# 35. `ipmitool sel info`

To inspect SEL repository information:

```bash
ipmitool sel info
```

This is useful for checking:

- SEL capabilities,
- number of entries,
- free space,
- timestamps,
- repository-related information.

OpenBMC provides this and other SEL operations through its IPMI implementation.

---

# 36. Clearing the SEL

The OpenBMC IPMI cheat sheet documents:

```bash
ipmitool sel clear
```

This clears SEL information.

Important:

> Clearing an SEL removes the stored event history and should be treated as a destructive administrative operation.

It is useful during controlled testing, but should not be casually used on a production system.

---

# 37. Deleting a Single SEL Event

OpenBMC documentation also provides:

```bash
ipmitool sel delete <number>
```

when supported by the target implementation/configuration.

This is different from:

```bash
ipmitool sel clear
```

because it targets a particular event rather than clearing the repository.

---

# 38. SEL Time

SEL timestamps matter during fault analysis.

Useful commands include:

```bash
ipmitool sel time get
```

and, depending on implementation:

```bash
ipmitool sel time set ...
```

When correlating:

```text
Sensor event
+
Journal timestamp
+
SEL timestamp
+
Redfish timestamp
```

time synchronization becomes important.

---

# 39. SEL Clock vs System Clock

When debugging timestamps, do not blindly assume every clock is identical.

You may need to compare:

```text
BMC system time
SEL time
Host time
NTP synchronization
Redfish event timestamp
```

A timestamp mismatch can make a correctly logged event appear to have happened at the wrong time.

---

# 40. SEL Record Generation from a Threshold

Consider:

```text
CPU_Temp = 92°C
CriticalHigh = 90°C
```

The threshold monitor detects:

```text
CriticalHigh alarm asserted
```

The flow can become:

```text
Sensor
   ↓
Threshold interface
   ↓
PropertiesChanged
   ↓
Threshold Event Monitor
   ↓
phosphor-sel-logger
   ↓
Sensor path + SEL data
   ↓
SEL Record
```

The SEL record can then be queried through the IPMI interface.

---

# 41. Assertion and Deassertion

Suppose:

```text
90°C = Critical High
```

Temperature rises:

```text
88°C → 92°C
```

The threshold event can assert:

```text
ASSERT
```

Later:

```text
92°C → 85°C
```

The condition can deassert:

```text
DEASSERT
```

Conceptually:

```text
Temperature
    │
92°C│       ASSERT
    │        ▲
90°C│────────┼──────── Threshold
    │        │
85°C│        ▼ DEASSERT
    │
    └────────────────── Time
```

Whether both transitions are logged depends on the event-monitor configuration.

---

# 42. Why Event Direction Matters

Without direction:

```text
CPU temperature threshold
```

does not tell us whether:

```text
the problem started
```

or:

```text
the problem cleared
```

Therefore:

```text
Event Direction
       ↓
Assert / Deassert
```

is important for interpreting the fault lifecycle.

---

# 43. Sensor Type

IPMI uses standardized sensor types.

Examples conceptually include:

```text
Temperature
Voltage
Fan
Processor
Power Supply
```

The sensor type helps an IPMI client interpret the event.

The D-Bus world may identify a sensor through:

```text
interface
object path
properties
```

while IPMI represents it using:

```text
sensor number
sensor type
event data
```

---

# 44. IPMI SDR and SEL

Another important distinction:

### SDR

**Sensor Data Record**

Describes sensor information used by IPMI.

### SEL

**System Event Log**

Records events involving the system.

Conceptually:

```text
SDR
 ↓
"What sensor is this?"

SEL
 ↓
"What event happened?"
```

For example:

```text
SDR:
Sensor 0x08 = CPU Temperature

SEL:
Sensor 0x08 exceeded upper critical threshold
```

---

# 45. `ipmitool sensor` vs `ipmitool sel`

These commands answer different questions.

### Sensor state

```bash
ipmitool sensor
```

asks:

```text
What are the current sensor readings?
```

### Event history

```bash
ipmitool sel list
```

asks:

```text
What important events were recorded?
```

Therefore:

```text
sensor → current state

sel → historical event records
```

---

# 46. `ipmitool sdr`

The SDR command:

```bash
ipmitool sdr
```

is useful for understanding the IPMI sensor inventory.

Conceptually:

```text
D-Bus / Platform Sensor
       ↓
IPMI Sensor Mapping
       ↓
SDR
       ↓
Sensor Number / Metadata
```

This is especially useful when an SEL entry contains only an IPMI sensor number.

---

# 47. Debugging an Unknown SEL Sensor

Suppose:

```text
ipmitool sel list
```

shows:

```text
Sensor #0x08
```

You need to find which hardware it represents.

A useful approach:

```text
SEL Sensor Number
       ↓
IPMI Sensor Inventory
       ↓
D-Bus Path
       ↓
Inventory Object
       ↓
Physical Hardware
```

This is why platform-specific IPMI sensor mappings are critical.

---

# 48. OpenBMC Source Tree for SEL

Useful repositories/files include:

```text
openbmc/phosphor-sel-logger
```

for SEL logging.

```text
openbmc/phosphor-dbus-interfaces
```

for the D-Bus IPMI logging interface.

```text
openbmc/phosphor-host-ipmid
```

for host-endpoint IPMI.

```text
openbmc/openbmc
/meta-phosphor/recipes-phosphor/sel-logger
```

for Yocto integration/configuration.

---

# 49. Source Code: SEL Logger Service

The SEL Logger source creates a D-Bus service:

```text
xyz.openbmc_project.Logging.IPMI
```

at:

```text
/xyz/openbmc_project/Logging/IPMI
```

and registers methods for adding:

```text
System SEL
OEM SEL
```

This is visible directly in the current `sel_logger.cpp` implementation.

---

# 50. Source Code: SEL Interface Definition

The D-Bus interface is defined in:

```text
phosphor-dbus-interfaces/yaml/
xyz/openbmc_project/Logging/IPMI.interface.yaml
```

Its purpose is explicitly described as providing an:

```text
IPMI System Event Log (SEL) logging interface
```

under:

```text
/xyz/openbmc_project/Logging/IPMI
```

This is the contract between SEL-producing applications and the SEL Logger service.

---

# 51. Source Code: Host IPMI

The current `phosphor-host-ipmid` source contains SEL-related structures and constants.

For example, its sensor handler defines:

```text
ipmiSELObject =
xyz.openbmc_project.Logging.IPMI
```

and:

```text
ipmiSELPath =
/xyz/openbmc_project/Logging/IPMI
```

This demonstrates how IPMI-side logic can interact with the SEL logging D-Bus interface.

---

# 52. Source Code: Yocto Recipe

The SEL Logger recipe is:

```text
meta-phosphor/recipes-phosphor/sel-logger/
phosphor-sel-logger_git.bb
```

It:

- builds `phosphor-sel-logger`,
- defines configuration options,
- installs its systemd service,
- declares dependencies,
- controls optional event monitoring.

This is the point where source-code capability becomes an actual feature in the BMC image.

---

# 53. Systemd Service

The Yocto recipe installs:

```text
xyz.openbmc_project.Logging.IPMI.service
```

So when debugging the running BMC, check:

```bash
systemctl status xyz.openbmc_project.Logging.IPMI.service
```

and:

```bash
journalctl -u xyz.openbmc_project.Logging.IPMI.service
```

The exact service availability depends on whether the feature is included in the target image.

---

# 54. Debugging Checklist

When SEL is not appearing:

### Step 1

Check the service:

```bash
systemctl status xyz.openbmc_project.Logging.IPMI.service
```

### Step 2

Check D-Bus:

```bash
busctl list | grep -i logging
```

### Step 3

Inspect the IPMI logging object:

```bash
busctl introspect \
    xyz.openbmc_project.Logging.IPMI \
    /xyz/openbmc_project/Logging/IPMI
```

### Step 4

Check the sensor:

```bash
busctl tree xyz.openbmc_project.Sensor
```

or inspect the relevant sensor service/object.

### Step 5

Check the threshold:

```text
xyz.openbmc_project.Sensor.Threshold
```

### Step 6

Check SEL:

```bash
ipmitool sel list
```

### Step 7

Check journal:

```bash
journalctl -u xyz.openbmc_project.Logging.IPMI.service
```

---

# 55. Debugging the Complete Path

Use this model:

```text
                 WHY NO SEL?
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
     Sensor        Threshold     SEL Logger
        │             │             │
        ▼             ▼             ▼
     Value        Alarm/Event     Service
        │             │             │
        └─────────────┴─────────────┘
                      │
                      ▼
                   SEL Record
                      │
                      ▼
                  IPMI Interface
                      │
                      ▼
                   ipmitool
```

Debug from left to right.

---

# 56. Common Mistake: "Sensor Is Critical, So SEL Must Exist"

Not necessarily.

The actual chain depends on:

```text
Sensor implementation
+
Threshold monitoring
+
SEL monitoring configuration
+
Platform mapping
+
SEL Logger service
+
IPMI implementation
```

A sensor can correctly report a critical condition while no SEL record is generated if the appropriate SEL monitoring path is disabled or not configured.

---

# 57. Common Mistake: "SEL Is Stored in D-Bus"

Do not oversimplify this.

The D-Bus interface:

```text
/xyz/openbmc_project/Logging/IPMI
```

is an interface for adding/handling SEL records.

The current `phosphor-sel-logger` implementation documents journal-backed storage and metadata for SEL records, with configuration options that affect integration with `phosphor-logging`.

Therefore distinguish:

```text
D-Bus interface
```

from:

```text
persistent SEL storage
```

---

# 58. Common Mistake: "IPMI = ipmitool"

`ipmitool` is a client utility.

It is not the BMC's IPMI implementation.

Think:

```text
ipmitool
   ↓
IPMI protocol
   ↓
BMC IPMI service
   ↓
BMC resources
```

So:

```text
ipmitool ≠ IPMI implementation
```

---

# 59. Common Mistake: "Redfish Replaced IPMI Internally"

Redfish and IPMI are management interfaces.

OpenBMC can support both:

```text
                 OpenBMC
                /       \
               /         \
          Redfish        IPMI
             │             │
          bmcweb       IPMI stack
```

They may expose overlapping information using different models.

The internal architecture does not reduce to:

```text
Redfish OR IPMI
```

---

# 60. IPMI vs Redfish

| Concept | IPMI | Redfish |
|---|---|---|
| Style | Command/protocol oriented | REST/JSON resource model |
| Typical client | `ipmitool` | `curl`, Redfish client |
| Sensor representation | Sensor number / SDR | Resource URI / JSON |
| Event history | SEL | LogService / Entries |
| Internal OpenBMC communication | D-Bus integration | bmcweb + D-Bus |
| Modern data model | Older/compact | Resource-oriented |
| Platform extensions | OEM commands/records | OEM properties/extensions |

Neither table column should be treated as a complete description of every implementation.

---

# 61. Complete Example — CPU Temperature

Assume:

```text
CPU_Temp = 92°C
CriticalHigh = 90°C
```

### Step 1 — Sensor

```text
CPU_Temp
Value = 92
```

### Step 2 — Threshold

```text
CriticalHigh = 90
```

### Step 3 — Alarm

```text
CriticalAlarmHigh = true
```

### Step 4 — D-Bus signal

```text
PropertiesChanged
```

### Step 5 — SEL monitor

```text
Threshold Event Monitor
```

### Step 6 — SEL Logger

```text
phosphor-sel-logger
```

### Step 7 — SEL Record

Conceptually:

```text
Sensor Type
Sensor Number
Event Direction = Assert
Event Data
Generator ID
Timestamp
```

### Step 8 — Query

```bash
ipmitool sel list
```

### Step 9 — Recovery

Temperature falls:

```text
92°C → 85°C
```

The corresponding deassert event may be recorded if configured.

---

# 62. Complete Example — Fan Failure

Assume:

```text
Fan2 target = 8000 RPM
Fan2 feedback = 0 RPM
```

Flow:

```text
Fan Sensor
    ↓
Fan Monitoring
    ↓
Fault Condition
    ↓
D-Bus Event
    ↓
SEL Logger
    ↓
SEL Record
    ↓
Sensor Number
    ↓
IPMI
    ↓
ipmitool sel list
```

To identify the physical fan:

```text
SEL Sensor Number
      ↓
IPMI Sensor Mapping
      ↓
D-Bus Sensor
      ↓
Inventory
      ↓
Fan2
```

This combines:

```text
Sensor
+
IPMI mapping
+
Inventory
+
SEL
```

---

# 63. Complete End-to-End Architecture

The whole learning series now connects:

```text
                         HARDWARE
                            │
                            ▼
                         SENSOR
                            │
                            ▼
                       SENSOR VALUE
                            │
                            ▼
                        THRESHOLD
                            │
                            ▼
                           ALARM
                            │
                 ┌──────────┴──────────┐
                 │                     │
                 ▼                     ▼
        phosphor-logging       SEL Event Monitor
                 │                     │
                 ▼                     ▼
             D-Bus Log          phosphor-sel-logger
                 │                     │
                 ▼                     ▼
             Event Log              SEL
                 │                     │
                 ▼                     ▼
              Redfish                IPMI
                 │                     │
                 ▼                     ▼
             bmcweb                ipmitool
```

This is the key architecture to remember.

---

# 64. Where Inventory Fits

The architecture becomes even richer when hardware identity is included:

```text
Sensor
   │
   ├── D-Bus Sensor Path
   │
   └── IPMI Sensor Number
             │
             ▼
       Inventory Mapping
             │
             ▼
        Physical Device
```

For example:

```text
0x08
 ↓
CPU_Temp
 ↓
CPU0
```

The exact mapping is platform-specific.

---

# 65. Where FRU Fits

FRU information provides hardware identity.

So:

```text
FRU
 ↓
Inventory
 ↓
Sensor
 ↓
Event
 ↓
SEL
```

A fault can therefore be interpreted in terms of a real physical component.

Example:

```text
FRU:
Fan Module 2

Sensor:
Fan2 RPM

Fault:
Fan2 nonfunctional

SEL:
Fan2 failure
```

---

# 66. Day 21 → Day 25 Connection

The series now has a strong dependency chain:

```text
Day 21
Inventory
    ↓
Day 22
FRU / EEPROM
    ↓
Day 23
Sensor Thresholds
    ↓
Day 24
Event Logging
    ↓
Day 25
IPMI SEL
```

Each topic builds on the previous one.

---

# 67. Interview Question

### Q: What happens when a sensor threshold is crossed in OpenBMC?

A strong answer:

> A sensor value is monitored against configured thresholds. When a relevant alarm state changes, the monitoring component can react to the D-Bus state change. Depending on platform configuration, an event can be recorded through OpenBMC logging and/or the SEL logging path. `phosphor-sel-logger` can monitor threshold events and create IPMI SEL records. The resulting SEL can then be accessed through the BMC's IPMI interface, for example using `ipmitool sel list`.

This answer is more accurate than saying:

> Sensor crosses threshold → automatically creates SEL.

---

# 68. Interview Question

### Q: What is the difference between SEL and Redfish Event Log?

Answer:

> SEL is the IPMI representation of system events, using IPMI-specific fields such as record ID, sensor number, event direction and event data. Redfish exposes management information through a REST/JSON resource model and can expose event logs through LogService resources. OpenBMC can support both interfaces, but their internal representations and translation paths are different.

---

# 69. Interview Question

### Q: What is phosphor-sel-logger?

Answer:

> `phosphor-sel-logger` is an OpenBMC component responsible for handling IPMI SEL logging. It exposes a D-Bus interface for adding System and OEM SEL records and can monitor configured event types, such as threshold events, and generate SEL records automatically.

---

# 70. Interview Question

### Q: What is the difference between sensor number and D-Bus sensor path?

Answer:

> A D-Bus sensor path is an internal OpenBMC object identifier, while an IPMI sensor number is part of the traditional IPMI sensor model. OpenBMC uses platform-specific mappings to connect the two representations.

---

# 71. Interview Question

### Q: Does every D-Bus event become an SEL?

Answer:

> No. SEL generation depends on the relevant event source, monitoring logic, platform configuration and whether the event is configured for SEL logging.

---

# 72. Interview Question

### Q: What is the difference between `ipmitool sensor` and `ipmitool sel list`?

Answer:

> `ipmitool sensor` primarily shows current sensor readings, while `ipmitool sel list` shows recorded system event history.

---

# 73. Interview Question

### Q: What is SDR?

Answer:

> SDR stands for Sensor Data Record. It provides IPMI-side metadata describing sensors, while SEL records describe events involving the system. SDR helps interpret an IPMI sensor number; SEL records describe what happened.

---

# 74. Practical Command Sheet

### Discover IPMI logging service

```bash
busctl list | grep -i ipmi
```

### Inspect SEL D-Bus interface

```bash
busctl introspect \
    xyz.openbmc_project.Logging.IPMI \
    /xyz/openbmc_project/Logging/IPMI
```

### Check SEL logger

```bash
systemctl status xyz.openbmc_project.Logging.IPMI.service
```

### Check SEL logger logs

```bash
journalctl -u xyz.openbmc_project.Logging.IPMI.service
```

### IPMI SEL information

```bash
ipmitool sel info
```

### List SEL records

```bash
ipmitool sel list
```

### Extended SEL listing

```bash
ipmitool sel elist
```

### Current sensors

```bash
ipmitool sensor
```

### SDR

```bash
ipmitool sdr
```

### SEL time

```bash
ipmitool sel time get
```

### Clear SEL — destructive

```bash
ipmitool sel clear
```

### Delete one SEL entry — if supported

```bash
ipmitool sel delete <record-id>
```

---

# 75. Remote IPMI Example

For a BMC reachable over the network:

```bash
ipmitool \
    -I lanplus \
    -H <BMC-IP> \
    -U <USERNAME> \
    -P <PASSWORD> \
    sel list
```

The management path is approximately:

```text
ipmitool
    ↓
RMCP+
    ↓
Network
    ↓
BMC IPMI service
    ↓
SEL implementation
    ↓
SEL data
```

Use secure credential handling in real environments; do not place production passwords directly in shell history.

---

# 76. Source-Code Investigation Workflow

When investigating an SEL issue:

### Step 1

Find the sensor.

```text
D-Bus sensor path
```

### Step 2

Find the threshold.

```text
xyz.openbmc_project.Sensor.Threshold
```

### Step 3

Find the D-Bus property change.

```text
PropertiesChanged
```

### Step 4

Find the SEL monitor.

```text
phosphor-sel-logger
```

### Step 5

Find the D-Bus SEL method.

```text
IpmiSelAdd
IpmiSelAddOem
```

### Step 6

Find the platform mapping.

```text
Sensor Number
        ↓
D-Bus Path
```

### Step 7

Query the SEL.

```bash
ipmitool sel list
```

This lets you trace:

```text
Source → Trigger → Logger → Record → Management Interface
```

---

# 77. What Can Go Wrong?

### Problem 1

Sensor threshold changes but no SEL appears.

Check:

```text
SEL monitoring configuration
```

### Problem 2

SEL appears with an unexpected sensor number.

Check:

```text
IPMI sensor inventory mapping
```

### Problem 3

SEL exists but timestamp is wrong.

Check:

```text
BMC time
SEL time
NTP
```

### Problem 4

SEL Logger service is absent.

Check:

```text
Yocto image configuration
PACKAGECONFIG
```

### Problem 5

`ipmitool` cannot access the SEL remotely.

Check:

```text
network IPMI
RMCP+
credentials
network configuration
IPMI service
```

---

# 78. The Most Important Mental Model

Remember this:

```text
D-Bus
  ↓
OpenBMC internal representation

IPMI
  ↓
Traditional management protocol

SEL
  ↓
IPMI event history

phosphor-sel-logger
  ↓
OpenBMC component connecting event sources to SEL logging

ipmitool
  ↓
Client used to query/manage IPMI
```

---

# 79. Day 24 vs Day 25

| Day 24 | Day 25 |
|---|---|
| Event Logging | IPMI SEL |
| `phosphor-logging` | `phosphor-sel-logger` |
| Structured OpenBMC event | IPMI event record |
| D-Bus Event Log | SEL interface |
| Redfish EventLog | IPMI SEL |
| Hardware association | Sensor Number / IPMI mapping |
| `lg2::commit()` | `IpmiSelAdd` / SEL event path |
| Fault record | Traditional system-event record |

The two days are connected, but they should not be treated as identical mechanisms.

---

# 80. Final Architecture

```text
                         HARDWARE
                            │
                            ▼
                          SENSOR
                            │
                            ▼
                         VALUE
                            │
                            ▼
                       THRESHOLD
                            │
                            ▼
                           ALARM
                            │
                  PropertiesChanged
                            │
             ┌──────────────┴──────────────┐
             │                             │
             ▼                             ▼
      phosphor-logging             phosphor-sel-logger
             │                             │
             ▼                             ▼
       Event Log                         SEL
             │                             │
             ▼                             ▼
          Redfish                         IPMI
             │                             │
             ▼                             ▼
          bmcweb                       ipmitool
```

---

# 81. What We Learned Today

- IPMI provides a traditional interface for BMC/server management.
- SEL means **System Event Log**.
- SEL records important system events using IPMI-specific structures.
- `phosphor-sel-logger` handles IPMI SEL logging in OpenBMC.
- SEL Logger exposes the D-Bus interface:
  `/xyz/openbmc_project/Logging/IPMI`.
- System and OEM SEL events have different record formats.
- SEL metadata includes record ID, record type, generator ID, sensor path, event direction and event data.
- D-Bus sensor paths and IPMI sensor numbers are different representations.
- Platform-specific mappings connect OpenBMC sensors to IPMI sensor numbers.
- SEL Logger can monitor threshold events through D-Bus `PropertiesChanged`.
- SEL monitoring is configurable and is not necessarily enabled in every image.
- `ipmitool sensor` shows current sensor information.
- `ipmitool sdr` helps inspect IPMI sensor metadata.
- `ipmitool sel list` shows recorded SEL events.
- `ipmitool sel info` provides SEL repository information.
- `ipmitool sel clear` clears SEL history.
- Network IPMI and host IPMI are different paths.
- `ipmitool` is a client, not the BMC IPMI implementation.
- SEL and OpenBMC Event Logs are related but are not the same thing.

---

# 82. One-Line Summary

> **OpenBMC connects modern D-Bus-based sensor and event infrastructure with the traditional IPMI System Event Log through `phosphor-sel-logger`, allowing important platform events to be represented as SEL records and accessed through IPMI clients such as `ipmitool`.**

---

# 83. Final Mental Model

If you remember only one diagram:

```text
Sensor / Hardware
       ↓
Sensor Value
       ↓
Threshold
       ↓
Alarm / Event
       ↓
PropertiesChanged
       ↓
phosphor-sel-logger
       ↓
SEL Record
       ↓
IPMI
       ↓
ipmitool
       ↓
Administrator
```

And the relationship with Day 24:

```text
Day 24
Fault
 ↓
Event
 ↓
phosphor-logging
 ↓
D-Bus Event Log
 ↓
Redfish / Management

Day 25
Fault / Threshold Event
 ↓
SEL Logger
 ↓
IPMI SEL
 ↓
ipmitool
```


---

# References

1. OpenBMC `phosphor-sel-logger`  
   https://github.com/openbmc/phosphor-sel-logger

2. OpenBMC `phosphor-sel-logger` README — SEL metadata, D-Bus interface, automatic event monitoring  
   https://github.com/openbmc/phosphor-sel-logger/blob/master/README.md

3. OpenBMC `phosphor-sel-logger` source — D-Bus SEL interface and record creation  
   https://github.com/openbmc/phosphor-sel-logger/blob/master/src/sel_logger.cpp

4. OpenBMC `phosphor-dbus-interfaces` — IPMI SEL interface  
   https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Logging/IPMI.interface.yaml

5. OpenBMC `openbmc` — phosphor-sel-logger Yocto recipe and configuration  
   https://github.com/openbmc/openbmc/blob/master/meta-phosphor/recipes-phosphor/sel-logger/phosphor-sel-logger_git.bb

6. OpenBMC architecture documentation — Network IPMI / RMCP+ and Host IPMI  
   https://github.com/openbmc/docs/blob/master/architecture/interface-overview.md

7. OpenBMC development documentation — IPMI sensor/inventory mappings and SEL inventory  
   https://github.com/openbmc/docs/blob/master/development/add-new-system.md

8. OpenBMC IPMI tool cheat sheet — SEL commands  
   https://github.com/openbmc/docs/blob/master/IPMITOOL-cheatsheet.md

9. OpenBMC `phosphor-host-ipmid`  
   https://github.com/openbmc/phosphor-host-ipmid

10. OpenBMC event-logging design  
    https://github.com/openbmc/docs/blob/master/designs/event-logging.md

---

# End of Day 25

### OpenBMC Learning Series

**From Sensor Fault → D-Bus Event → SEL → IPMI → ipmitool**

> Detect the condition.  
> Translate it into an IPMI event.  
> Record it as SEL.  
> Expose it through IPMI.  
> Diagnose it with `ipmitool`.

**Learn. Build. Go Deeper.**
