# 🚀 OpenBMC Learning Series --- Day 22

## EEPROM & FRU Data in OpenBMC

### From I²C/SMBus → EEPROM → FRU Fields → D-Bus → Inventory

------------------------------------------------------------------------

## 1. Introduction

In **Day 21**, we learned how OpenBMC represents physical hardware using
inventory, D-Bus, associations and Redfish. The next question is:

> **Where does the hardware identification information actually come
> from?**

For many server platforms, an important source is non-volatile memory
such as an **EEPROM** connected through **I²C/SMBus**. That memory can
contain structured **FRU (Field Replaceable Unit)** information such as
manufacturer, product name, part number, serial number and other
serviceability data.

The Day 22 architecture is:

``` text
Physical Hardware
      │ I²C / SMBus
      ▼
   EEPROM
      │ Raw FRU bytes
      ▼
  FRU Parser
      │ Structured fields
      ▼
 FruDevice / D-Bus
      │
      ▼
 Entity Manager
      │
      ▼
  Inventory
      │
      ▼
   bmcweb
      │
      ▼
   Redfish
```

> **Day 21:** How OpenBMC represents hardware. **Day 22:** How OpenBMC
> can obtain and interpret hardware identification data.

------------------------------------------------------------------------

## 2. What Is EEPROM?

**EEPROM** means **Electrically Erasable Programmable Read-Only
Memory**. It is non-volatile memory, so information can remain stored
when power is removed.

A board may contain a small EEPROM holding identification or other
platform data:

``` text
┌─────────────────────────┐
│ EEPROM                  │
├─────────────────────────┤
│ Manufacturer            │
│ Product Name            │
│ Part Number             │
│ Serial Number           │
│ Version                 │
│ Other platform data     │
└─────────────────────────┘
```

The key distinction is:

> **EEPROM is the storage medium. FRU is the structured information
> format.**

Not every EEPROM is a FRU EEPROM.

------------------------------------------------------------------------

## 3. Why Does the BMC Need FRU Information?

A BMC needs to understand what hardware exists in the system:

``` text
Baseboard
├── CPU
├── DIMMs
├── Power Supplies
├── Fans
├── Backplane
└── Add-in cards
```

Management software may need to answer:

-   Which board is this?
-   Who manufactured it?
-   What is its part number?
-   What is its serial number?
-   Which configuration applies to it?
-   Which sensors belong to it?

FRU information is therefore useful for inventory, serviceability, asset
tracking, configuration, IPMI and higher-level management interfaces.

------------------------------------------------------------------------

## 4. I²C / SMBus → EEPROM

A typical connection looks like:

``` text
BMC
 │
 │ I²C Controller
 ▼
I²C Bus
 │
 ├── EEPROM @ 0x50
 ├── Temperature Sensor @ 0x48
 └── Other Devices
```

The exact bus number and address are platform-specific.

The low-level path is:

``` text
BMC SoC
  ↓
I²C Controller
  ↓
Linux I²C subsystem
  ↓
EEPROM
  ↓
Raw bytes
```

At this point the data is just bytes. The FRU parser gives those bytes
meaning.

------------------------------------------------------------------------

## 5. I²C/SMBus vs EEPROM vs FRU

These concepts are easy to mix up:

  Concept       Meaning
  ------------- ------------------------------------------------------
  I²C / SMBus   Communication mechanism
  EEPROM        Non-volatile storage device
  FRU           Structured hardware information format
  Inventory     OpenBMC software representation of physical hardware

So:

``` text
I²C / SMBus → communicates
EEPROM      → stores
FRU         → structures information
Inventory   → represents hardware in software
```

------------------------------------------------------------------------

## 6. FRU Is Not the Same as Inventory

A FRU record and an OpenBMC inventory object are different abstractions.

**FRU** primarily provides identification/serviceability information
associated with a replaceable unit.

**Inventory** represents physical objects in the OpenBMC software model.

Therefore:

``` text
FRU identity
    ↓
Entity Manager / other consumers
    ↓
Inventory object
    ↓
Associations
```

Also remember:

> **Not every inventory item is a FRU, and not every inventory item has
> to be discovered from an EEPROM.**

------------------------------------------------------------------------

## 7. FRU Information Layout

The IPMI FRU Information Storage Definition defines a common layout:

``` text
FRU Information Device
│
├── Common Header
├── Internal Use Area      (optional)
├── Chassis Info Area      (optional)
├── Board Info Area        (optional)
├── Product Info Area      (optional)
└── MultiRecord Area       (optional)
```

The Common Header provides offsets used to locate the other areas. The
areas are variable-length rather than one universal fixed layout.

------------------------------------------------------------------------

## 8. Common Header

Conceptually, the Common Header contains:

``` text
┌──────────────────────────────┐
│ Format Version               │
├──────────────────────────────┤
│ Internal Use Offset          │
├──────────────────────────────┤
│ Chassis Info Offset          │
├──────────────────────────────┤
│ Board Info Offset            │
├──────────────────────────────┤
│ Product Info Offset          │
├──────────────────────────────┤
│ MultiRecord Offset            │
├──────────────────────────────┤
│ Pad                          │
├──────────────────────────────┤
│ Checksum                     │
└──────────────────────────────┘
```

The offsets are specified in units of 8 bytes. This is why a parser
should not assume a fixed byte location for every field.

The parser should:

1.  Read the Common Header.
2.  Validate its format/checksum.
3.  Obtain the relevant area offset.
4.  Parse the selected area.
5.  Validate the area/record checksum.

------------------------------------------------------------------------

## 9. Chassis Information Area

The Chassis Info Area contains chassis-related information such as:

``` text
Chassis Type
Chassis Part Number
Chassis Serial Number
```

Additional custom fields may exist.

A system can contain multiple FRU information devices, but chassis
information is not expected to be duplicated indiscriminately across
every device.

------------------------------------------------------------------------

## 10. Board Information Area

The Board Info Area describes the board or replaceable assembly
associated with the FRU device.

Typical fields include:

``` text
Manufacturer
Product Name
Serial Number
Part Number
FRU-related field
```

OpenBMC Entity Manager configurations can use properties such as:

``` text
BOARD_MANUFACTURER
BOARD_PRODUCT_NAME
BOARD_PART_NUMBER
BOARD_SERIAL_NUMBER
```

------------------------------------------------------------------------

## 11. Product Information Area

The Product Info Area can contain:

``` text
Manufacturer
Product Name
Part Number
Version
Serial Number
Asset Tag
FRU-related field
```

For example:

``` text
PRODUCT_MANUFACTURER  = ExampleCorp
PRODUCT_PRODUCT_NAME  = ServerBoard-X
PRODUCT_PART_NUMBER   = SB-X-001
PRODUCT_SERIAL_NUMBER = SN123456
PRODUCT_VERSION       = 1.0
```

These are example values only; actual contents are platform/manufacturer
dependent.

------------------------------------------------------------------------

## 12. MultiRecord Area

The MultiRecord Area contains one or more records, each with its own
record type and format.

``` text
MultiRecord Area
│
├── Record 1 → Type A
├── Record 2 → Type B
└── Record N → Type N
```

It provides an extension mechanism for additional standardized or
OEM-defined information without changing the basic Chassis/Board/Product
area definitions.

------------------------------------------------------------------------

## 13. Type/Length Fields and Checksums

Many FRU fields use a **type/length** encoding:

``` text
┌──────────────┬──────────────────────┐
│ Type/Length  │ Field Data           │
└──────────────┴──────────────────────┘
```

The parser must interpret the encoding instead of assuming every field
is a fixed-length ASCII string.

Checksums are equally important. Conceptually:

``` text
Raw EEPROM
    ↓
Validate Header
    ↓
Validate Area / Record
    ↓
Parse Fields
```

If validation fails, the software should not blindly treat the contents
as trusted FRU data.

------------------------------------------------------------------------

## 14. Raw EEPROM Bytes → Structured FRU

Suppose an EEPROM contains raw bytes:

``` text
01 00 00 08 10 18 00 ...
```

Those bytes have no useful human meaning until interpreted according to
the FRU format.

``` text
Raw Bytes
   ↓
Common Header
   ↓
Board/Product/Chassis offset
   ↓
Area parser
   ↓
Type/Length decoding
   ↓
Checksum validation
   ↓
Structured properties
```

For example:

``` text
Raw FRU bytes
      ↓
BOARD_PRODUCT_NAME
      ↓
"Management Board"
```

------------------------------------------------------------------------

## 15. Where Linux Fits

OpenBMC runs Linux on the BMC. Linux provides the I²C subsystem and the
hardware-driver infrastructure used to communicate with devices.

``` text
Application
    ↓
Linux userspace
    ↓
Linux kernel / I²C subsystem
    ↓
I²C controller
    ↓
EEPROM
```

This is another example of the layered architecture studied earlier:
applications do not normally manipulate controller registers directly
when a kernel subsystem provides the appropriate abstraction.

------------------------------------------------------------------------

## 16. How OpenBMC Can Discover FRU EEPROMs

Entity Manager documents `fru-device` as a common detection daemon that
scans available I²C buses for IPMI FRU EEPROM devices.

Conceptually:

``` text
I²C buses
   ↓
fru-device
   ↓
FRU EEPROM discovery
   ↓
FRU parsing
   ↓
FruDevice D-Bus objects
```

However, this is **not a universal requirement for every OpenBMC
platform**. Other detection sources can include SMBIOS, PCIe/CPU-bus
discovery, GPIO presence, device-tree/platform configuration and other
platform-specific daemons.

------------------------------------------------------------------------

## 17. The `fru-device` Service

Entity Manager contains a systemd service named:

``` text
xyz.openbmc_project.FruDevice.service
```

Its service definition runs:

``` text
/usr/libexec/entity-manager/fru-device
```

and owns the D-Bus name:

``` text
xyz.openbmc_project.FruDevice
```

So a useful mental model is:

``` text
systemd
  ↓
fru-device
  ↓
I²C / FRU discovery
  ↓
D-Bus
```

------------------------------------------------------------------------

## 18. FruDevice D-Bus Representation

Entity Manager documentation demonstrates objects under a path such as:

``` text
/xyz/openbmc_project/FruDevice/...
```

with the interface:

``` text
xyz.openbmc_project.FruDevice
```

Properties can include:

``` text
PRODUCT_MANUFACTURER
PRODUCT_PRODUCT_NAME
PRODUCT_PART_NUMBER
PRODUCT_SERIAL_NUMBER
PRODUCT_VERSION
PRODUCT_ASSET_TAG
BOARD_PRODUCT_NAME
...
```

The important transformation is:

``` text
EEPROM bytes
    ↓
FRU parser
    ↓
FruDevice properties
```

------------------------------------------------------------------------

## 19. Real OpenBMC Source: `ipmi-fru-parser`

The OpenBMC `ipmi-fru-parser` repository contains a small entry point in
`readeeprom.cpp`.

The important part is:

``` cpp
auto bus = sdbusplus::bus::new_default();

rc = validateFRUArea(
    fruid,
    eeprom_file.c_str(),
    bus);
```

The source also describes the input as an EEPROM file and then validates
the FRU area and updates the Inventory DB.

The architectural lesson is more important than the small amount of
code:

``` text
EEPROM data
   ↓
validateFRUArea(...)
   ↓
FRU validation / parsing
   ↓
D-Bus / inventory update
```

------------------------------------------------------------------------

## 20. Why Does the Parser Need D-Bus?

The parser is not useful merely because it can decode bytes.

The useful result is:

``` text
Storage
  ↓
Meaningful hardware identity
  ↓
OpenBMC software state
```

The D-Bus connection lets the parsed information participate in the
OpenBMC IPC/data model.

This is consistent with the architecture from earlier days:

``` text
Hardware → Linux → Service → D-Bus → Other Services
```

------------------------------------------------------------------------

## 21. Entity Manager: Turning Detection into Configuration

Entity Manager's configuration can contain a `Probe` such as:

``` json
{
  "Name": "Intel Front Panel",
  "Probe": "xyz.openbmc_project.FruDevice({'BOARD_PRODUCT_NAME': 'FFPANEL'})"
}
```

The meaning is:

``` text
Find a FruDevice
      ↓
Read BOARD_PRODUCT_NAME
      ↓
Does it match "FFPANEL"?
      ↓
YES → load the corresponding configuration
```

Entity Manager documentation describes Probe as the condition that
causes a configuration to be exported to D-Bus.

------------------------------------------------------------------------

## 22. Probe vs Exposes

A very useful distinction:

### Probe

> **"Do I have this hardware?"**

Example:

``` text
BOARD_PRODUCT_NAME == "FFPANEL"
```

### Exposes

> **"What configuration should I export when it matches?"**

Example:

``` json
"Exposes": [
  {
    "Address": "$address",
    "Bus": "$bus",
    "Name": "Front Panel FRU",
    "Type": "EEPROM"
  }
]
```

So:

``` text
FRU identity
     ↓
Probe
     ↓
Exposes
     ↓
D-Bus configuration
```

------------------------------------------------------------------------

## 23. FRU Fields → Inventory Asset Data

A real Entity Manager configuration demonstrates direct use of FRU
fields:

``` json
"xyz.openbmc_project.Inventory.Decorator.Asset": {
    "Manufacturer": "$BOARD_MANUFACTURER",
    "Model": "$BOARD_PRODUCT_NAME",
    "PartNumber": "$BOARD_PART_NUMBER",
    "SerialNumber": "$BOARD_SERIAL_NUMBER"
}
```

This gives a concrete transformation:

``` text
BOARD_MANUFACTURER  → Manufacturer
BOARD_PRODUCT_NAME  → Model
BOARD_PART_NUMBER   → PartNumber
BOARD_SERIAL_NUMBER → SerialNumber
```

This is one of the clearest real-world examples of FRU data becoming
inventory metadata.

------------------------------------------------------------------------

## 24. Inventory.Item

The OpenBMC `xyz.openbmc_project.Inventory.Item` interface provides
basic attributes for inventory objects.

Important properties include:

``` text
PrettyName
Present
```

and associations can describe relationships such as:

``` text
containing
contained_by
powered_by
sensors
monitored_by
```

So the model becomes:

``` text
FRU identity
    ↓
Inventory object
    ├── PrettyName
    ├── Present
    ├── Asset data
    └── Associations
```

------------------------------------------------------------------------

## 25. FRU → Inventory Is Not One-to-One

Avoid this oversimplification:

``` text
One EEPROM = One Inventory Item
```

A better model is:

``` text
FRU device
    ↓
identity / detection information
    ↓
D-Bus
    ↓
Entity Manager / other consumers
    ↓
one or more software representations
```

One physical object can also have multiple D-Bus interfaces. The
software model is a graph, not a simple EEPROM-to-object table.

------------------------------------------------------------------------

## 26. FRU + Inventory + Sensors

This connects directly to **Day 17**.

Day 17 focused on:

``` text
Sensor source
   ↓
Sensor.Value
   ↓
D-Bus
   ↓
bmcweb
   ↓
Redfish
```

Inventory can associate hardware with sensors:

``` text
Power Supply
   │
   └── sensors
        ├── Voltage
        ├── Power
        └── Temperature
```

So:

``` text
FRU       → What is this component?
Inventory → How is it represented?
Sensor    → What is happening to it?
Association → How are they related?
```

------------------------------------------------------------------------

## 27. FRU → bmcweb → Redfish

Once the hardware is represented in D-Bus, higher-level services can
consume that structured information.

The conceptual path is:

``` text
FRU EEPROM
    ↓
FRU Parser / FruDevice
    ↓
D-Bus
    ↓
Entity Manager / Inventory
    ↓
bmcweb
    ↓
Redfish
```

The key lesson is:

> **bmcweb should not need to understand raw EEPROM bytes just to expose
> hardware information.**

The lower layers turn hardware data into a software model; bmcweb
translates the model into an external management representation.

------------------------------------------------------------------------

## 28. FRU and IPMI

FRU information is historically associated with IPMI. OpenBMC contains
FRU-related IPMI components and configuration.

Conceptually the same structured information can support multiple
management paths:

``` text
             FRU Data
             /      \
            /        \
       D-Bus          IPMI
         ↓
     Inventory
         ↓
      bmcweb
         ↓
      Redfish
```

The exact implementation depends on the platform and enabled OpenBMC
packages.

------------------------------------------------------------------------

## 29. Why Use FRU Properties in a Probe?

Suppose two hardware variants use different board identities:

``` text
Board-A → BOARD_PRODUCT_NAME = "Board-A"
Board-B → BOARD_PRODUCT_NAME = "Board-B"
```

Entity Manager can use those values to select the appropriate
configuration:

``` text
FRU Identity
     ↓
Probe Match
     ↓
Platform Configuration
```

This reduces the need to hard-code every platform difference into one
large daemon.

------------------------------------------------------------------------

## 30. Template Variables

Entity Manager configurations can use FRU-derived values as variables,
for example:

``` json
"Manufacturer": "$BOARD_MANUFACTURER",
"Model": "$BOARD_PRODUCT_NAME",
"PartNumber": "$BOARD_PART_NUMBER",
"SerialNumber": "$BOARD_SERIAL_NUMBER"
```

So:

``` text
FruDevice
   ↓
FRU properties
   ↓
Entity Manager variables
   ↓
Inventory / configuration
```

This is a practical bridge between discovery and platform configuration.

------------------------------------------------------------------------

## 31. Full Example: Management Board

Imagine a board contains:

``` text
Manufacturer  = ExampleCorp
Product       = Management Board X
Part Number   = MB-X-001
Serial Number = MB123456
```

The end-to-end path can be:

``` text
1. Physical Board
       ↓
2. I²C EEPROM
       ↓
3. Raw FRU bytes
       ↓
4. FRU parser
       ↓
5. FruDevice D-Bus
       ↓
6. Entity Manager Probe
       ↓
7. Inventory / configuration
       ↓
8. bmcweb
       ↓
9. Redfish
```

The important transformation is:

``` text
Physical bytes
      ↓
Meaningful fields
      ↓
OpenBMC objects
      ↓
Remote management model
```

------------------------------------------------------------------------

## 32. Debugging: Work Bottom-Up

Suppose a PSU is physically installed but its information does not
appear in Redfish.

Do not start with bmcweb.

Use:

``` text
Physical PSU
    ↓
I²C connection
    ↓
EEPROM detected?
    ↓
FRU valid?
    ↓
FruDevice exists?
    ↓
Expected properties present?
    ↓
Entity Manager Probe matches?
    ↓
Inventory exists?
    ↓
Associations correct?
    ↓
bmcweb discovers it?
    ↓
Redfish resource appears?
```

This identifies the layer where the failure occurs.

------------------------------------------------------------------------

## 33. Step 1 --- Check I²C / EEPROM

Useful Linux tools on a development/debug system include:

``` bash
i2cdetect -l
i2cdetect -y <bus>
```

But use them carefully.

Do not blindly probe or write devices on a production server. Devices
may be behind multiplexers, may require platform-specific access, or may
not safely support generic probing.

The first question is simply:

``` text
Correct bus?
Correct address?
Device responds?
```

------------------------------------------------------------------------

## 34. Step 2 --- Check FRU Validity

An EEPROM responding does **not** prove that its contents are valid FRU
data.

Check:

``` text
Common Header
    ↓
Format version
    ↓
Offsets
    ↓
Checksum
    ↓
Area format
    ↓
Area checksum
    ↓
Fields
```

If parsing fails, investigate the EEPROM contents and FRU programming
before moving upward.

------------------------------------------------------------------------

## 35. Step 3 --- Check FruDevice

Use D-Bus inspection:

``` bash
busctl list | grep FruDevice
```

Then:

``` bash
busctl tree xyz.openbmc_project.FruDevice
```

For a discovered object:

``` bash
busctl introspect \
  xyz.openbmc_project.FruDevice \
  /xyz/openbmc_project/FruDevice/<object>
```

Look for properties such as:

``` text
PRODUCT_MANUFACTURER
PRODUCT_PRODUCT_NAME
PRODUCT_PART_NUMBER
PRODUCT_SERIAL_NUMBER
BOARD_PRODUCT_NAME
BOARD_MANUFACTURER
```

------------------------------------------------------------------------

## 36. Step 4 --- Check Entity Manager

If FruDevice is correct but inventory is missing, inspect:

-   Probe condition
-   Expected property name
-   Expected property value
-   Configuration file
-   Exposes section
-   Associations

For example:

``` text
Probe expects:
BOARD_PRODUCT_NAME = "FFPANEL"
```

but the real FruDevice says:

``` text
BOARD_PRODUCT_NAME = "FrontPanel"
```

The Probe will not match.

------------------------------------------------------------------------

## 37. Step 5 --- Check Inventory and Associations

Verify that the expected inventory object exists and inspect:

``` text
PrettyName
Present
Asset information
Associations
```

If the inventory item exists but its sensors do not appear, the problem
may be an association rather than FRU parsing.

Think in layers:

``` text
Identity       → FRU
Representation → Inventory
Relationship   → Association
Measurement    → Sensor
External API   → Redfish
```

------------------------------------------------------------------------

## 38. Step 6 --- Check bmcweb / Redfish

Only after the D-Bus model is correct should you investigate:

``` text
bmcweb
  ↓
ObjectMapper / D-Bus discovery
  ↓
Redfish resource mapping
  ↓
JSON response
```

This is the same source-reading/debugging approach used in Days 15--17.

------------------------------------------------------------------------

## 39. Common Mistakes

### Mistake 1: EEPROM detected = FRU working

False.

``` text
EEPROM detected ✓
FRU valid       ✗
```

### Mistake 2: FRU = Inventory

False.

FRU provides structured identification information; Inventory models
physical objects in OpenBMC.

### Mistake 3: Every FRU comes from EEPROM

False.

OpenBMC platforms can use other discovery mechanisms such as SMBIOS,
PCIe-related discovery, GPIO presence and platform configuration.

### Mistake 4: bmcweb reads EEPROM directly

Do not use that as the normal architectural mental model.

Prefer:

``` text
EEPROM → Parser → D-Bus → Inventory → bmcweb → Redfish
```

------------------------------------------------------------------------

## 40. Source-Reading Strategy

When studying a real platform, use this order:

1.  Find `xyz.openbmc_project.FruDevice`.
2.  Find `xyz.openbmc_project.FruDevice.service`.
3.  Find the `fru-device` implementation.
4.  Read FRU parser code such as `ipmi-fru-parser`.
5.  Search for properties such as `PRODUCT_PRODUCT_NAME` and
    `BOARD_PRODUCT_NAME`.
6.  Search Entity Manager configuration for
    `xyz.openbmc_project.FruDevice` probes.
7.  Study `Probe` and `Exposes`.
8.  Follow the resulting inventory objects.
9.  Finally follow the data into bmcweb/Redfish.

This is the same **architecture → source → runtime** method we have been
using since Day 13.

------------------------------------------------------------------------

## 41. Day 21 → Day 22

### Day 21 --- Hardware Representation

``` text
Physical Hardware
      ↓
Inventory
      ↓
D-Bus
      ↓
bmcweb
      ↓
Redfish
```

### Day 22 --- Hardware Identification Source

``` text
I²C / SMBus
      ↓
EEPROM
      ↓
FRU
      ↓
FruDevice
      ↓
Entity Manager
      ↓
Inventory
```

So:

> **Day 21 = How hardware is represented.**
>
> **Day 22 = How hardware identity can enter that model.**

------------------------------------------------------------------------

## 42. Day 17 → Day 22

Day 17 followed sensor measurements:

``` text
Sensor Source → Sensor.Value → D-Bus → bmcweb → Redfish
```

Day 22 follows hardware identity:

``` text
FRU EEPROM → FruDevice → D-Bus → Inventory → bmcweb → Redfish
```

Together:

``` text
                 OpenBMC
                    │
        ┌───────────┴───────────┐
        │                       │
     Identity               Measurement
        │                       │
       FRU                    Sensor
        │                       │
        └──────────┬────────────┘
                   ▼
                 D-Bus
                   │
               Inventory
                   │
                bmcweb
                   │
                Redfish
```

The BMC needs to know both:

> **What hardware is this?**

and:

> **What is happening to it?**

------------------------------------------------------------------------

## 43. Day 19 → Day 22

Day 19 followed a control path:

``` text
Management Request
      ↓
State Management
      ↓
Power Control
      ↓
GPIO
      ↓
Hardware
```

Day 22 follows an information path:

``` text
Hardware Identity
      ↓
EEPROM
      ↓
FRU
      ↓
D-Bus
      ↓
Inventory
      ↓
Management API
```

A real BMC needs both directions:

``` text
CONTROL PATH:
Management → BMC → Hardware

INFORMATION PATH:
Hardware → BMC → Management
```

------------------------------------------------------------------------

## 44. The Bigger OpenBMC Picture

``` text
                    Remote Management
                           │
                 ┌─────────┴─────────┐
                 │                   │
              Redfish              IPMI
                 │                   │
                 └─────────┬─────────┘
                           ▼
                         D-Bus
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
     Sensors            Inventory           State
        │                  │                  │
   Sensor.Value       FRU / Asset       Host/BMC State
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                           ▼
                    Platform Services
                           │
                  ┌────────┴────────┐
                  │                 │
                GPIO              I²C
                  │                 │
                  ▼                 ▼
              Hardware          EEPROM / Sensors
```

Day 22 adds the identity path:

``` text
Hardware → I²C/SMBus → EEPROM → FRU → D-Bus → Inventory → Redfish
```

------------------------------------------------------------------------

## 45. Day 22 Mental Model

Remember these five layers:

### 1. EEPROM

**Where the bytes are stored.**

### 2. FRU

**How hardware information is structured.**

### 3. FruDevice / Parser

**How raw bytes become meaningful properties.**

### 4. Entity Manager / Inventory

**How OpenBMC uses identity information to model hardware.**

### 5. bmcweb / Redfish

**How that hardware information becomes remotely consumable.**

In one line:

``` text
EEPROM stores
    ↓
FRU structures
    ↓
Parser interprets
    ↓
D-Bus exposes
    ↓
Entity Manager models
    ↓
Redfish presents
```

------------------------------------------------------------------------

## 46. Key Takeaways

1.  **EEPROM is storage; FRU is structured information.**
2.  **I²C/SMBus commonly provides the communication path to EEPROM
    devices.**
3.  **The FRU Common Header contains offsets to information areas.**
4.  **Chassis, Board, Product and MultiRecord areas have different
    purposes and are not all mandatory.**
5.  **Checksums are used to validate FRU data.**
6.  **OpenBMC can expose parsed FRU information through
    `xyz.openbmc_project.FruDevice`.**
7.  **Entity Manager can use FruDevice properties in Probe conditions.**
8.  **Exposes configuration describes what Entity Manager should
    export/configure after a match.**
9.  **FRU and Inventory are related but are not the same abstraction.**
10. **Not every EEPROM contains FRU data, and not every hardware
    discovery path uses EEPROM.**
11. **Inventory associations connect hardware with other objects such as
    sensors and power supplies.**
12. **bmcweb should consume structured OpenBMC data rather than being
    treated as a raw EEPROM parser.**
13. **Debug bottom-up: bus → EEPROM → FRU → FruDevice → Entity Manager →
    Inventory → bmcweb → Redfish.**

------------------------------------------------------------------------

## 47. What's Next?

We now know how OpenBMC can discover hardware identity.

The next question is:

> **How does OpenBMC detect abnormal hardware conditions and decide that
> something is in a warning or critical state?**

That naturally leads to:

``` text
Sensor Value
      ↓
Thresholds
      ↓
Warning / Critical
      ↓
Events / Faults
      ↓
Logging
      ↓
Redfish / Management
```

This connects the sensor work from Day 17 with the inventory, state and
control work from Days 18--22.

------------------------------------------------------------------------

## 48. References

### OpenBMC

-   `openbmc/entity-manager`
    -   `README.md`
    -   `CONFIG_FORMAT.md`
    -   `schemas/README.md`
    -   `schemas/global.json`
    -   `docs/associations.md`
    -   `docs/my_first_sensors.md`
    -   `service_files/xyz.openbmc_project.FruDevice.service`
    -   Example platform configurations
-   `openbmc/ipmi-fru-parser`
    -   `readeeprom.cpp`
    -   FRU parsing implementation files
-   `openbmc/phosphor-dbus-interfaces`
    -   `yaml/xyz/openbmc_project/Inventory/Item.interface.yaml`
    -   Inventory Item documentation
-   `openbmc/openbmc`
    -   `meta-phosphor/recipes-phosphor/configuration/entity-manager_git.bb`

### FRU specification

-   **Platform Management FRU Information Storage Definition v1.0**
    -   Common Header
    -   Chassis Information Area
    -   Board Information Area
    -   Product Information Area
    -   MultiRecord Area
    -   Type/Length fields
    -   Checksum validation

------------------------------------------------------------------------

## 49. Final Architecture

``` text
                 PHYSICAL HARDWARE
                        │
                        │ I²C / SMBus
                        ▼
                  ┌───────────┐
                  │  EEPROM   │
                  └─────┬─────┘
                        │
                        │ Raw Bytes
                        ▼
                 ┌──────────────┐
                 │  FRU Parser  │
                 └──────┬───────┘
                        │
                        │ Structured FRU Fields
                        ▼
              ┌──────────────────────┐
              │  FruDevice / D-Bus   │
              └──────────┬───────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ Entity Manager  │
                │ Probe / Exposes │
                └────────┬────────┘
                         │
                         ▼
                  ┌─────────────┐
                  │  Inventory  │
                  └──────┬──────┘
                         │
                         ▼
                    ┌─────────┐
                    │ bmcweb  │
                    └────┬────┘
                         │
                         ▼
                    ┌─────────┐
                    │ Redfish │
                    └────┬────┘
                         │
                         ▼
                REMOTE MANAGEMENT
```

### The key idea

> **OpenBMC does not need to expose raw EEPROM bytes to management
> software. It can convert low-level hardware identification data into
> structured software objects that the rest of OpenBMC can understand
> and manage.**

------------------------------------------------------------------------

## 🔥 Day 22 Summary

``` text
EEPROM
  ↓
FRU
  ↓
FruDevice
  ↓
D-Bus
  ↓
Entity Manager
  ↓
Inventory
  ↓
bmcweb
  ↓
Redfish
```

**From a tiny EEPROM on a hardware board → to a meaningful hardware
object visible to a remote management client.**

------------------------------------------------------------------------

*OpenBMC Learning Series --- Day 22*
