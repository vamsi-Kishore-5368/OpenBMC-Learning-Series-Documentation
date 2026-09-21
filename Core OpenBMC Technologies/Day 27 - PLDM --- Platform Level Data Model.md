# OpenBMC Learning Series --- Day 27

# PLDM --- Platform Level Data Model

> **Theme:** Moving from the MCTP transport layer into standardized
> platform-management messages.

## 1. Day 27 Position in the Series

Day 25 introduced **IPMI and SEL**. Day 26 introduced **MCTP**, the
modern transport layer. Day 27 introduces **PLDM**, a DMTF-defined
platform-management protocol that commonly runs over MCTP.

``` text
Day 25: IPMI / SEL → Traditional management
Day 26: MCTP      → Transport between components
Day 27: PLDM      → Platform management data + commands
```

**MCTP = how the message gets there.**\
**PLDM = what the message means.**

OpenBMC describes PLDM as a standardized data model/message format for
platform management, including inventory, monitoring, control, events
and data transfer. \[1\]\[2\]

## 2. Why PLDM?

Modern servers contain many independently managed components: host
processors, BMCs, memory, PCIe devices, NVMe drives, GPUs, retimers,
network controllers, power controllers, fans, sensors and security
devices.

A management system needs standardized ways to discover capabilities,
identify resources, read sensors, control effecters, represent
inventory, receive events and manage firmware/BIOS-related data.

MCTP solves **transport**. PLDM supplies **management semantics**.

``` text
                    PLATFORM
                       |
        +--------------+--------------+
        |              |              |
      Sensors        FRUs          Devices
        |              |              |
        +--------------+--------------+
                       |
                     PLDM
                       |
                     MCTP
                       |
          +------------+------------+
          |            |            |
         I2C          PCIe        UART
```

## 3. What Is PLDM?

**PLDM = Platform Level Data Model.**

PLDM is a DMTF-defined family of message formats and commands for
platform management. It defines standardized data representations and
commands without tying the higher layer to one physical transport.

``` text
Application / Management Function
            |
           PLDM
            |
           MCTP
            |
      Physical Binding
```

OpenBMC documentation describes PLDM as useful for inside-the-box
communication such as BMC↔Host, BMC↔BMC, BMC↔network controller and
BMC↔other platform devices. \[1\]

## 4. PLDM vs MCTP

  -----------------------------------------------------------------------
  Layer                   Technology              Main responsibility
  ----------------------- ----------------------- -----------------------
  Management API          Redfish / D-Bus         External or local
                                                  management model

  Platform management     PLDM                    Management
                                                  commands/data

  Transport               MCTP                    Delivery between
                                                  endpoints

  Physical binding        I²C/SMBus, PCIe, UART,  Moves bits across
                          etc.                    hardware
  -----------------------------------------------------------------------

Example:

``` text
"Read temperature sensor"
        |
       PLDM   → command + data model
        |
       MCTP   → transports PLDM message
        |
   I2C/PCIe/UART
        |
      Device
```

## 5. PLDM Message Architecture

A PLDM message contains a common header followed by a type-specific
request or response payload.

``` text
+-------------------+
| PLDM Header       |
+-------------------+
| Type-specific     |
| Request/Response  |
| Payload           |
+-------------------+
```

Important concepts include the **Instance ID**, **PLDM Type**,
request/response indication and **Command**. OpenBMC exposes these
through `pldm_msg` and `libpldm` encode/decode APIs.

## 6. Instance ID

An **Instance ID** helps correlate a PLDM request with its corresponding
response.

``` text
Requester
   |
   | Instance ID = X
   | Request
   v
Responder
   |
   | Instance ID = X
   | Response
   v
Requester
```

OpenBMC allocates instance IDs for PLDM requests associated with MCTP
endpoints.

## 7. PLDM Types

PLDM groups commands into functional **Types**. The current OpenBMC
implementation includes support for multiple types, including:

-   Base
-   Platform Monitoring and Control
-   BIOS Control and Configuration
-   FRU
-   Firmware Update
-   SMBIOS
-   RDE
-   File Transfer
-   OEM extensions

The exact set depends on the implementation/version and endpoint
capabilities. The OpenBMC `pldmtool` source maintains mappings between
supported PLDM types and command codes. \[2\]\[3\]

## 8. Base PLDM Type

The Base Type provides fundamental PLDM operations such as:

``` text
GetTID
SetTID
GetPLDMTypes
GetPLDMVersion
GetPLDMCommands
SelectPLDMVersion
```

A useful discovery sequence is:

``` text
New Terminus
    ↓
GetTID
    ↓
GetPLDMTypes
    ↓
GetPLDMVersion
    ↓
GetPLDMCommands
    ↓
Use supported Type
```

OpenBMC's `pldmtool` exposes these commands and the responder implements
corresponding Base handlers. \[3\]\[4\]

## 9. Terminus and TID

A **PLDM Terminus** is a PLDM endpoint participating in platform
management. Each terminus is identified by a **TID --- Terminus ID**.

Do not confuse TID with MCTP EID:

``` text
MCTP layer  → Endpoint → EID
PLDM layer  → Terminus → TID
```

Mental model:

-   **EID:** where is the endpoint on the MCTP network?
-   **TID:** which PLDM terminus is being managed?

OpenBMC's `pldmd` maintains a terminus table and performs TID
initialization/discovery. \[5\]

## 10. PLDM Discovery in OpenBMC

High-level flow:

``` text
MCTP endpoint appears
        |
        v
      pldmd
        |
        v
Discover PLDM support
        |
        v
Get TID
        |
        v
Get PLDM Types
        |
        v
Get PLDM Versions
        |
        v
Get PLDM Commands
        |
        v
Retrieve PDRs
        |
        v
Create platform representation
        |
        v
Monitor / Control / Events
```

OpenBMC's PLDM design states that `pldmd` uses MCTP D-Bus interfaces to
enumerate endpoints, maintains a terminus table, initializes new
termini, discovers supported PLDM types/versions, retrieves PDRs when
supported, and then starts monitoring/control. \[5\]

## 11. What Is a PDR?

**PDR = Platform Descriptor Record.**

PDRs are central to PLDM Platform Monitoring and Control. They provide
structured information describing platform resources and relationships.

``` text
        PDR Repository
              |
      +-------+-------+
      |       |       |
    Sensor  Effecter  Entity
     PDR      PDR      PDR
      |       |       |
   Temp      Fan     CPU/DIMM
```

Think of the PDR repository as a standardized description of a platform
rather than a hard-coded list of devices.

## 12. PDR Repository

A PLDM terminus can expose a repository containing records such as:

``` text
PLDM Terminus
      |
      +--> Sensor PDR
      +--> Effecter PDR
      +--> Entity Association PDR
      +--> Terminus Locator PDR
      +--> FRU Record Set PDR
      +--> Auxiliary Name PDR
      +--> OEM PDR
```

OpenBMC can retrieve PDRs using commands such as `GetPDRRepositoryInfo`
and `GetPDR`. The current `pldmtool platform` implementation supports
retrieving an individual PDR, PDRs by type, PDRs for a terminus ID, or
all PDRs. \[6\]

## 13. Important PDR Types

### Numeric Sensor PDR

Describes a numeric sensor such as temperature, voltage, current or
power.

### State Sensor PDR

Describes state-based information such as power or device state.

### Numeric Effecter PDR

Describes a controllable numeric value such as fan speed or another
control target.

### State Effecter PDR

Describes a controllable state.

### Entity Association PDR

Describes relationships between platform entities.

### Terminus Locator PDR

Provides information used to locate a terminus; OpenBMC code handles
MCTP-EID terminus locator information. \[7\]

### FRU Record Set PDR

Describes FRU-related platform information.

The current OpenBMC platform implementation maps and processes these PDR
categories. \[6\]\[7\]

## 14. Entity Association

A platform is hierarchical:

``` text
System
 |
 +-- Motherboard
      |
      +-- CPU
      |    +-- Temperature Sensor
      |
      +-- DIMM
      |    +-- Temperature Sensor
      |
      +-- Fan
```

Entity Association PDRs help management software understand which
resources belong to which entities and subsystems.

## 15. Sensor PDR → D-Bus

A useful OpenBMC flow is:

``` text
Remote PLDM Terminus
        |
        | GetPDR
        v
    Sensor PDR
        |
        v
      pldmd
        |
        v
 Parse sensor information
        |
        v
    D-Bus Object
        |
        v
 OpenBMC Applications
```

The OpenBMC PLDM design describes creating terminus inventory, sensors
and effecters as D-Bus objects after discovery. \[5\]

## 16. PLDM Platform Monitoring and Control

The Platform Monitoring and Control Type is especially relevant for BMC
work.

It can represent:

**Sensors:** temperature, voltage, current, power and state.

**Effecters:** fan speed, power state, control values and device states.

**Events:** notifications about changes in platform state.

Example:

``` text
Temperature crosses threshold
          |
          v
     PLDM Event
          |
          v
        MCTP
          |
          v
        BMC
          |
          v
      OpenBMC
```

## 17. PLDM Events

PLDM supports event-driven communication as well as request/response
operations.

``` text
Device
  |
  | Sensor state changed
  v
PLDM Platform Event
  |
  v
MCTP
  |
  v
BMC
  |
  +--> D-Bus
  +--> Logging
  +--> Sensor state
  +--> Management exposure
```

OpenBMC's PLDM stack contains platform-event processing paths. \[7\]

## 18. PLDM FRU

**FRU = Field Replaceable Unit.**

PLDM provides standardized mechanisms for FRU-related platform
information.

Possible FRUs include:

-   Motherboard
-   Power supply
-   Fan module
-   Storage device
-   PCIe card
-   Memory module

FRU data can be connected to the platform entity model through the
relevant PDR structures.

## 19. PLDM BIOS

The BIOS Control and Configuration Type provides standardized mechanisms
for BIOS-related management.

``` text
BMC
 |
 | PLDM BIOS
 v
Host Firmware
 |
 +-- BIOS attributes
 +-- BIOS configuration
 +-- BIOS settings
```

## 20. PLDM Firmware Update

PLDM also defines a Firmware Update Type.

``` text
BMC
 |
 | PLDM Firmware Update
 v
Managed Device
 |
 +-- Package / component identification
 +-- Transfer
 +-- Verification
 +-- Activation
```

The OpenBMC PLDM repository contains a `fw-update` component. \[2\]\[8\]

## 21. PLDM Over MCTP

``` text
              PLDM
               |
       Management Semantics
               |
              MCTP
               |
            Transport
               |
      +--------+--------+
      |        |        |
     I2C      PCIe     UART
      |        |        |
      +--------+--------+
               |
             Device
```

This separation lets PLDM remain independent of the underlying physical
transport. OpenBMC's MCTP documentation explicitly describes the
separation between transport and physical bindings and identifies PLDM
as a higher-layer protocol carried over MCTP. \[9\]

## 22. OpenBMC PLDM Architecture

A simplified architecture is:

``` text
                 Redfish / Other Apps
                         |
                       D-Bus
                         |
              +----------+----------+
              |                     |
            pldmd               Other PLDM
              |                  Components
              |
          libpldm
              |
        Requester / Responder
              |
          MCTP Transport
              |
        AF_MCTP / MCTP APIs
              |
        Linux Kernel MCTP
              |
       MCTP Physical Binding
              |
        +-----+-----+-----+
        |           |     |
       I2C         PCIe  UART
```

The current OpenBMC repository contains `pldmd`, requester code,
responder libraries, `libpldm`, platform management, host-BMC handling,
firmware update and `pldmtool`. \[2\]

## 23. What Is pldmd?

`pldmd` is the OpenBMC PLDM daemon. Its responsibilities include
endpoint/terminus discovery, PLDM discovery, PDR processing, D-Bus
object creation, monitoring, control and event handling.

The OpenBMC design describes a lifecycle of:

``` text
Terminus initialization
        ↓
Terminus discovery
        ↓
Monitor and control
```

\[5\]

## 24. pldmd and MCTP D-Bus

A key OpenBMC integration point is:

``` text
             MCTP stack
                 |
                 | D-Bus
                 v
               pldmd
                 |
                 v
          PLDM Terminus Table
```

The PLDM design says `pldmd` watches MCTP D-Bus endpoint
additions/removals and updates its terminus table accordingly. \[5\]

## 25. libpldm

`libpldm` provides common PLDM functionality, especially encode/decode
APIs used to construct and interpret messages.

``` text
Application
    |
    | encode request
    v
 libpldm
    |
    v
PLDM message
```

Response:

``` text
PLDM response
    |
    v
 libpldm
    |
    | decode response
    v
Application data
```

The OpenBMC `pldmtool` uses `libpldm` encode/decode functions. \[10\]

## 26. Requester vs Responder

``` text
Requester
   |
   | PLDM Request
   v
Responder
   |
   | PLDM Response
   v
Requester
```

In OpenBMC, `pldmtool` acts as a BMC-side PLDM requester, while the
repository also contains responder functionality. \[10\]\[11\]

## 27. pldmtool

`pldmtool` is a BMC-side PLDM requester and debugging/learning tool.

Current documented subcommands include:

``` text
raw
base
bios
platform
fru
oem-ibm
```

Basic usage:

``` bash
pldmtool -h
pldmtool base -h
pldmtool platform -h
```

It uses `libpldm` encode/decode functions and communicates with `pldmd`.
\[10\]

## 28. PLDM Discovery with pldmtool

Useful Base operations include:

``` text
GetTID
GetPLDMTypes
GetPLDMVersion
GetPLDMCommands
```

Conceptually:

``` text
pldmtool
   |
   +--> GetTID
   +--> GetPLDMTypes
   +--> GetPLDMVersion
   +--> GetPLDMCommands
```

The current OpenBMC source exposes these commands through the Base
command implementation. \[3\]

## 29. Reading PDRs with pldmtool

The platform command supports PDR retrieval concepts such as:

``` text
Individual PDR
PDR by type
PDR by terminus ID
All PDRs
```

The implementation uses record handles and, where needed, transfer
handles/flags to retrieve PDR data. \[6\]

## 30. Raw PLDM Requests

If a command is not implemented in the normal `pldmtool` command set,
the tool supports a raw path:

``` bash
pldmtool raw -d <data>
```

The byte sequence must follow the applicable PLDM specification and
target endpoint requirements. \[10\]

## 31. Debugging pldmd

OpenBMC's current README documents verbose mode:

``` bash
echo 'PLDMD_ARGS="--verbose"' > /etc/default/pldmd
systemctl restart pldmd
```

Disable it with:

``` bash
rm /etc/default/pldmd
systemctl restart pldmd
```

This is useful when investigating endpoint discovery,
requests/responses, PDR processing, sensor creation and events. \[2\]

## 32. Source Tree --- Where to Start

Current OpenBMC PLDM areas include:

``` text
pldm/
├── common/
├── docs/
├── fw-update/
├── host-bmc/
├── libpldmresponder/
├── oem/
├── platform-mc/
├── pldmd/
├── requester/
├── pldmtool/
├── softoff/
├── test/
├── tools/
└── utilities/
```

Useful reading targets:

``` text
pldmd/             → daemon and platform integration
requester/         → outgoing PLDM requests
libpldmresponder/  → responder-side functionality
platform-mc/       → platform monitoring/control
fw-update/         → firmware update
pldmtool/          → CLI requester/debugging
host-bmc/          → host/BMC PLDM integration
```

The repository evolves, so always check the current upstream tree when
following exact paths. \[2\]

## 33. Source-Code Walkthrough --- GetPLDMTypes

``` text
pldmtool
   |
   v
Create Request
   |
   v
encode_get_types_req()
   |
   v
PLDM Request
   |
   v
Transport
   |
   v
Remote Terminus
   |
   v
PLDM Response
   |
   v
decode_pldm_base_get_pldm_types_resp()
   |
   v
Display supported Types
```

The current `pldm_base_cmd.cpp` implementation uses
`encode_get_types_req()` and `decode_pldm_base_get_pldm_types_resp()`.
\[3\]

## 34. Source-Code Walkthrough --- GetPDR

``` text
pldmtool platform
       |
       v
     GetPDR
       |
       v
encode_get_pdr_req()
       |
       v
Remote Terminus
       |
       v
GetPDR Response
       |
       v
decode_get_pdr_resp()
       |
       v
PDR Header / Type
       |
       v
Interpret record
```

The current platform implementation handles record handles, transfer
handles, transfer flags and multi-part PDR retrieval. \[6\]

## 35. PDR Discovery Example

Imagine a remote device exposes a temperature sensor, voltage sensor,
fan control and FRU information.

``` text
Remote Device
     |
     +-- Numeric Sensor PDR
     |      +-- Temperature
     |
     +-- Numeric Sensor PDR
     |      +-- Voltage
     |
     +-- Numeric Effecter PDR
     |      +-- Fan Control
     |
     +-- FRU Record Set PDR
            +-- Device inventory
```

`pldmd` can process these records and expose appropriate platform
information through D-Bus. \[5\]

## 36. Complete Host → BMC → Device Example

``` text
HOST
 |
 | PLDM Request
 v
MCTP
 |
 | EID routing
 v
BMC / Device
 |
 | PLDM processing
 v
Sensor information
 |
 | PLDM Response
 v
MCTP
 |
v
HOST
```

At each layer:

``` text
PLDM → command + data model
MCTP → transport + addressing
I2C/PCIe/etc. → physical binding
```

## 37. Example: Sensor Discovery

``` text
1. MCTP endpoint appears
        ↓
2. pldmd detects endpoint
        ↓
3. PLDM support discovered
        ↓
4. TID discovered
        ↓
5. Supported PLDM types discovered
        ↓
6. Platform type discovered
        ↓
7. PDR repository queried
        ↓
8. Sensor PDR found
        ↓
9. Sensor information parsed
        ↓
10. D-Bus sensor object created
        ↓
11. Sensor monitored
```

This flow is particularly useful for OpenBMC interviews. \[5\]

## 38. PLDM vs IPMI

  -----------------------------------------------------------------------
  Feature                 IPMI                    PLDM
  ----------------------- ----------------------- -----------------------
  Main role               Platform management     Platform
                                                  management/data model

  Typical modern          IPMI channels           Commonly MCTP
  transport                                       

  Data model              IPMI-specific           DMTF PLDM models

  Resource description    SDR                     PDR

  Inside-the-box          More constrained by     Explicitly designed for
  component communication original design         platform component
                                                  communication

  Functional organization IPMI command ecosystem  Multiple PLDM Types
  -----------------------------------------------------------------------

PLDM does not mean IPMI disappears. Both can coexist in a BMC.

## 39. PLDM vs Redfish

These technologies operate at different layers.

``` text
External Client
      |
   Redfish
      |
   bmcweb
      |
    D-Bus
      |
   OpenBMC
      |
    PLDM
      |
    MCTP
      |
    Device
```

A useful mental model:

**Redfish → northbound management interface**

**PLDM → platform-component management protocol**

**MCTP → transport**

## 40. PLDM vs D-Bus

``` text
D-Bus
→ Local IPC/message bus inside the BMC

PLDM
→ Standardized platform-management protocol between PLDM participants
```

Example:

``` text
Remote Device
      |
     PLDM
      |
     MCTP
      |
     BMC
      |
    D-Bus
      |
OpenBMC service
```

## 41. PLDM vs MCTP vs D-Bus vs Redfish

  -----------------------------------------------------------------------
  Technology              Think of it as          Main question
  ----------------------- ----------------------- -----------------------
  D-Bus                   Local IPC               How do OpenBMC services
                                                  communicate?

  MCTP                    Transport               How do platform
                                                  endpoints communicate?

  PLDM                    Platform protocol/data  What does the
                          model                   management message
                                                  mean?

  Redfish                 External management API How does a client
                                                  manage the system?
  -----------------------------------------------------------------------

A possible end-to-end path is:

``` text
Redfish
   ↓
bmcweb
   ↓
D-Bus
   ↓
OpenBMC service
   ↓
PLDM
   ↓
MCTP
   ↓
Physical Transport
```

The exact path depends on the use case.

## 42. PLDM and Event Logging

PLDM events can feed platform-management workflows.

``` text
Device detects fault
       |
       v
PLDM Platform Event
       |
       v
MCTP
       |
       v
BMC
       |
       +--> D-Bus
       +--> Logging
       +--> Sensor state
       +--> Redfish exposure
```

This connects Day 24's event/fault-management topic with Day 27.

## 43. Day 24 → Day 25 → Day 26 → Day 27

``` text
Day 24
Event Logging
      |
      v
Day 25
IPMI / SEL
      |
      v
Day 26
MCTP
      |
      v
Day 27
PLDM
```

Broader architecture:

``` text
                    Redfish
                       |
                    bmcweb
                       |
                     D-Bus
                       |
              OpenBMC Services
                       |
                      PLDM
                       |
                      MCTP
                       |
        +--------------+--------------+
        |              |              |
       I2C            PCIe           UART
        |              |              |
      Device         Device         Device
```

## 44. Interview Questions

### Q1. What is PLDM?

A DMTF-defined platform-management data model/message protocol used for
inventory, monitoring, control, events and related platform-management
functions.

### Q2. Is PLDM a transport protocol?

No. MCTP commonly provides the transport.

### Q3. What is the difference between MCTP and PLDM?

MCTP transports messages between endpoints; PLDM defines management
message semantics.

### Q4. What is a TID?

A PLDM Terminus ID.

### Q5. What is an EID?

An MCTP Endpoint ID.

### Q6. EID vs TID?

EID belongs to MCTP addressing; TID identifies a PLDM terminus.

### Q7. What is a PDR?

A Platform Descriptor Record that provides structured information about
platform resources, relationships, sensors, effecters, inventory and
related management information.

### Q8. Why are PDRs important?

They provide a standardized, discoverable description of platform
resources instead of requiring every resource to be hard-coded.

### Q9. What is pldmd?

The OpenBMC PLDM daemon responsible for PLDM platform integration such
as discovery, PDR processing, monitoring, control and events.

### Q10. What is libpldm?

A library providing common PLDM functionality, including message
encode/decode APIs.

### Q11. What is pldmtool?

A BMC-side PLDM requester/debugging tool for issuing PLDM commands and
displaying responses.

### Q12. What is a PLDM Type?

A functional grouping of PLDM commands, such as Base, Platform
Monitoring and Control, FRU, BIOS and Firmware Update.

### Q13. What happens when a new MCTP endpoint appears?

At a high level, `pldmd` learns about the endpoint through the MCTP
D-Bus interface, determines whether PLDM is supported, initializes the
PLDM terminus, discovers capabilities and processes PDRs. \[5\]

### Q14. How does a PLDM sensor reach OpenBMC?

``` text
Remote Sensor
   ↓
PLDM Sensor PDR
   ↓
MCTP
   ↓
pldmd
   ↓
D-Bus Sensor Object
   ↓
OpenBMC applications
```

### Q15. Does PLDM replace Redfish?

No. They serve different layers of the management architecture.

## 45. Practical Debugging Checklist

### Step 1 --- Check MCTP

``` text
Is the endpoint visible?
Is the EID correct?
Is the interface/network configured?
Is routing correct?
```

### Step 2 --- Check PLDM daemon

``` bash
systemctl status pldmd
```

### Step 3 --- Enable verbose logging

``` bash
echo 'PLDMD_ARGS="--verbose"' > /etc/default/pldmd
systemctl restart pldmd
```

### Step 4 --- Use pldmtool

``` bash
pldmtool -h
pldmtool base -h
pldmtool platform -h
```

### Step 5 --- Check discovery

``` text
GetTID
GetPLDMTypes
GetPLDMVersion
GetPLDMCommands
```

### Step 6 --- Check PDRs

``` text
PDR Repository Info
GetPDR
Sensor PDR
Effecter PDR
Entity Association PDR
```

### Step 7 --- Check D-Bus

``` text
Did pldmd create the expected object?
Is the sensor/effecter visible?
```

## 46. Common Mistakes

**Mistake:** "PLDM is a transport protocol."\
**Correction:** PLDM is the platform-management protocol/data model.

**Mistake:** "EID and TID are the same."\
**Correction:** EID → MCTP; TID → PLDM.

**Mistake:** "PDR means only sensor information."\
**Correction:** PDRs describe multiple categories of platform
information.

**Mistake:** "PLDM replaces D-Bus."\
**Correction:** D-Bus is local IPC inside OpenBMC.

**Mistake:** "PLDM replaces Redfish."\
**Correction:** Redfish and PLDM operate at different management layers.

**Mistake:** "MCTP defines what a sensor means."\
**Correction:** MCTP transports the message; PLDM defines
sensor-management semantics.

## 47. Mental Model to Remember

``` text
                 MANAGEMENT
                     |
                  Redfish
                     |
                  bmcweb
                     |
                   D-Bus
                     |
             OpenBMC Services
                     |
                    PLDM
                     |
                    MCTP
                     |
          Physical MCTP Binding
                     |
          +----------+----------+
          |          |          |
         I2C        PCIe       UART
          |          |          |
       Device     Device     Device
```

Remember:

``` text
EID = MCTP endpoint identity
TID = PLDM terminus identity
PDR = Platform description
PLDM = Management semantics
MCTP = Transport
```

## 48. Day 27 Summary

Covered:

-   PLDM fundamentals
-   DMTF and PLDM
-   PLDM Types
-   PLDM Base Type
-   TID and Terminus
-   PLDM message concepts
-   PDRs and PDR repositories
-   Sensor and Effecter PDRs
-   Entity Association PDRs
-   Terminus Locator and FRU PDRs
-   PLDM events
-   Platform Monitoring and Control
-   PLDM FRU
-   PLDM BIOS
-   PLDM Firmware Update
-   `pldmd`
-   `libpldm`
-   `pldmtool`
-   Requester/responder architecture
-   MCTP integration
-   D-Bus integration
-   Source-code navigation
-   Debugging
-   Interview questions

## 49. Final Day 25 → 26 → 27 Connection

``` text
                    OpenBMC
                       |
        +--------------+--------------+
        |                             |
     Traditional                   Modern
     Management                  Management
        |                             |
      IPMI                           PLDM
        |                             |
       SEL                            MCTP
                                      |
                              Physical Bindings
                                      |
                         +------------+------------+
                         |            |            |
                        I2C          PCIe         UART
```

The important progression is:

``` text
IPMI / SEL
    ↓
MCTP
    ↓
PLDM
```

This is not simply "one replaces another"; each addresses a different
part of the platform-management architecture.

## 50. Official Source Map

-   OpenBMC PLDM: https://github.com/openbmc/pldm
-   PLDM Stack Design:
    https://github.com/openbmc/docs/blob/master/designs/pldm-stack.md
-   PLDM Tool: https://github.com/openbmc/pldm/tree/master/pldmtool
-   MCTP Design:
    https://github.com/openbmc/docs/blob/master/designs/mctp/mctp.md
-   MCTP Kernel Design:
    https://github.com/openbmc/docs/blob/master/designs/mctp/mctp-kernel.md
-   DMTF: https://www.dmtf.org/standards/PLDM

## 51. Recommended Source-Code Reading Order

``` text
1. pldm/README.md
        ↓
2. docs/designs/pldm-stack.md
        ↓
3. pldmd/
        ↓
4. libpldm/
        ↓
5. requester/
        ↓
6. platform-mc/
        ↓
7. pldmtool/
        ↓
8. host-bmc/
        ↓
9. fw-update/
        ↓
10. tests
```

Then connect it back to:

``` text
Linux MCTP
   ↓
AF_MCTP
   ↓
MCTP D-Bus
   ↓
pldmd
```

This is where Day 26 MCTP knowledge becomes directly useful for
understanding OpenBMC's PLDM implementation.

## References

\[1\] OpenBMC PLDM Stack Design ---
https://github.com/openbmc/docs/blob/master/designs/pldm-stack.md\
\[2\] OpenBMC PLDM Repository --- https://github.com/openbmc/pldm\
\[3\] OpenBMC `pldmtool` Base Commands ---
https://github.com/openbmc/pldm/blob/master/pldmtool/pldm_base_cmd.cpp\
\[4\] OpenBMC PLDM Responder Base ---
https://github.com/openbmc/pldm/blob/master/libpldmresponder/base.cpp\
\[5\] OpenBMC PLDM Stack: endpoint, terminus and PDR discovery ---
https://github.com/openbmc/docs/blob/master/designs/pldm-stack.md\
\[6\] OpenBMC `pldmtool` Platform/PDR Commands ---
https://github.com/openbmc/pldm/blob/master/pldmtool/pldm_platform_cmd.cpp\
\[7\] OpenBMC Host PDR Handler ---
https://github.com/openbmc/pldm/blob/master/host-bmc/host_pdr_handler.cpp\
\[8\] OpenBMC PLDM Firmware Update ---
https://github.com/openbmc/pldm/tree/master/fw-update\
\[9\] OpenBMC MCTP Design ---
https://github.com/openbmc/docs/blob/master/designs/mctp/mctp.md\
\[10\] OpenBMC `pldmtool` README ---
https://github.com/openbmc/pldm/blob/master/pldmtool/README.md\
\[11\] OpenBMC PLDM Repository --- https://github.com/openbmc/pldm
