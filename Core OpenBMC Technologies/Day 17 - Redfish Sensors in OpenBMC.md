# 🚀 OpenBMC Learning Series — Day 17

## Redfish Sensors in OpenBMC
### From Sensor Data → D-Bus → bmcweb → Redfish

---

## 1. Introduction

Days 13–16 progressively moved us from architecture to implementation:

```text
Day 13: Source Code → Running Service
Day 14: Running Service → D-Bus API
Day 15: D-Bus → bmcweb → Redfish
Day 16: Redfish Request → bmcweb C++ → D-Bus → JSON
```

Now we take a practical OpenBMC use case:

> **How does an actual sensor value become a Redfish sensor resource?**

Imagine:

```text
CPU temperature = 42 °C
```

How does that value travel through OpenBMC?

```text
Physical / Platform Sensor
        ↓
Sensor-producing software
        ↓
D-Bus Sensor Object
        ↓
ObjectMapper / Associations
        ↓
bmcweb sensor logic
        ↓
Redfish Sensor Resource
        ↓
JSON
        ↓
Remote Client
```

The goal of Day 17 is to connect this model to the current OpenBMC source code.

---

# 2. What Is an OpenBMC Sensor?

A sensor represents a measurable system value such as:

- Temperature
- Voltage
- Current
- Fan speed
- Power
- Pressure
- Humidity
- Frequency
- Utilization

OpenBMC's `xyz.openbmc_project.Sensor.Value` interface defines the common sensor-reading model and recognizes namespaces including `temperature`, `voltage`, `current`, `power`, `fan_tach`, `humidity`, `pressure`, `frequency`, and others.

Source:
https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Sensor/Value.interface.yaml

The important idea is:

> **OpenBMC represents sensor readings as D-Bus objects so that other software components can consume them.**

---

# 3. The D-Bus Sensor Object

Sensor objects live below:

```text
/xyz/openbmc_project/sensors
```

with a hierarchy based on sensor type.

For example:

```text
/xyz/openbmc_project/sensors/
    temperature/
        cpu0
```

or:

```text
/xyz/openbmc_project/sensors/
    voltage/
        vin0
```

The exact names depend on the platform.

OpenBMC documentation describes `/xyz/openbmc_project/sensors` as the sensor hierarchy and notes that the path categorizes the sensor; it does not necessarily represent physical topology.

Source:
https://github.com/openbmc/docs/blob/master/host-management.md

---

# 4. The `xyz.openbmc_project.Sensor.Value` Interface

The most important D-Bus interface for a normal sensor is:

```text
xyz.openbmc_project.Sensor.Value
```

Its current definition includes:

```text
Value
MaxValue
MinValue
Unit
```

`Value` is a `double` representing the sensor reading.

`Unit` describes the measurement unit. Examples include:

```text
DegreesC → temperature
Volts     → voltage
Amperes   → current
Watts     → power
RPMS      → fan tachometer
```

Source:
https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Sensor/Value.interface.yaml

A simplified object therefore looks like:

```text
D-Bus Object
/xyz/openbmc_project/sensors/temperature/cpu0
          │
          └── xyz.openbmc_project.Sensor.Value
                    │
                    ├── Value = 42.0
                    ├── Unit = DegreesC
                    ├── MaxValue = ...
                    └── MinValue = ...
```

---

# 5. Where Does the Value Come From?

The D-Bus sensor object is not necessarily the physical sensor itself.

A platform-specific sensor application may obtain the measurement from hardware or another subsystem and then expose the result through D-Bus.

For example:

```text
Temperature Sensor / Hardware
          ↓
Linux / hardware access
          ↓
OpenBMC sensor application
          ↓
D-Bus Sensor.Value
```

The exact producer depends on the platform and configuration.

The `dbus-sensors` project is one common implementation path and uses `sdbusplus` object-server infrastructure for sensor objects.

Source:
https://github.com/openbmc/dbus-sensors/blob/master/src/sensor.hpp

Therefore:

> **Do not assume every OpenBMC sensor originates from one single application.**

The common integration point is the D-Bus sensor interface.

---

# 6. Why D-Bus Is the Sensor Integration Point

Once a sensor exposes:

```text
xyz.openbmc_project.Sensor.Value
```

other components do not need to know how the hardware measurement was obtained.

They can consume:

```text
D-Bus service
      ↓
Object path
      ↓
Sensor.Value interface
      ↓
Value / Unit / limits
```

This creates an abstraction:

```text
Hardware-specific implementation
             ↓
        Standard D-Bus API
             ↓
      Multiple consumers
```

Potential consumers include:

```text
Redfish
IPMI
Telemetry
Control algorithms
Monitoring applications
Other OpenBMC services
```

---

# 7. Connect This to Day 16

Day 16 taught us that bmcweb performs D-Bus discovery and data retrieval.

For sensors, the current implementation is centered around:

```text
bmcweb/redfish-core/lib/sensors.hpp
```

The file is large, so we should not read it linearly.

Instead, we trace:

```text
Chassis
   ↓
Sensor associations
   ↓
Sensor object discovery
   ↓
D-Bus properties
   ↓
Redfish JSON
```

Current source:
https://github.com/openbmc/bmcweb/blob/master/redfish-core/lib/sensors.hpp

---

# 8. Why Chassis Associations Matter

Suppose a BMC has many sensor objects.

A Redfish request such as:

```text
/redfish/v1/Chassis/<id>/Thermal/
```

cannot simply return every sensor in the system.

bmcweb needs to determine:

> **Which sensors belong to this chassis/resource?**

The current sensor implementation builds a chassis sensor path and queries its:

```text
/all_sensors
```

association using:

```cpp
dbus::utility::getAssociationEndPoints(...)
```

This returns the sensor object paths associated with that chassis.

Conceptually:

```text
Redfish Chassis
      ↓
Chassis D-Bus object
      ↓
all_sensors association
      ↓
Relevant sensor object paths
```

---

# 9. Filtering the Sensor List

After discovering the chassis-associated sensors, bmcweb reduces the list according to the requested sensor type.

Conceptually:

```text
Thermal request
      ↓
Temperature-related sensors

Power request
      ↓
Voltage / power-related sensors
```

The current `sensors.hpp` contains `reduceSensorList()` for this purpose.

This means the flow is not:

```text
Get every sensor
```

It is:

```text
Requested Redfish resource
        ↓
Relevant chassis
        ↓
Associated sensors
        ↓
Requested sensor category
        ↓
Active sensor list
```

---

# 10. ObjectMapper Finds the Sensor Objects

This is one of the most important parts of the current source.

`getObjectsWithConnection()` sets:

```cpp
const std::string path =
    "/xyz/openbmc_project/sensors";

constexpr std::array<std::string_view, 1> interfaces = {
    "xyz.openbmc_project.Sensor.Value"};
```

and calls:

```cpp
dbus::utility::getSubTree(
    path,
    2,
    interfaces,
    ...);
```

The source comment explicitly identifies this as an ObjectMapper call to find sensor objects.

Source:
https://github.com/openbmc/bmcweb/blob/master/redfish-core/lib/sensors.hpp

Conceptually:

```text
bmcweb
   ↓
ObjectMapper / getSubTree()
   ↓
/xyz/openbmc_project/sensors
   ↓
xyz.openbmc_project.Sensor.Value
   ↓
Matching sensor objects
```

---

# 11. What Does `getSubTree()` Give bmcweb?

The mapper response contains matching object paths and the services/interfaces associated with them.

The current code processes the `MapperGetSubTreeResponse` and collects the D-Bus connections for the requested sensors.

Conceptually:

```text
Object Path
      ↓
Service / Connection
      ↓
Interfaces
```

This gives bmcweb enough information to perform subsequent D-Bus operations.

---

# 12. Simplified Sensor Discovery Example

Imagine ObjectMapper finds:

```text
Object:
 /xyz/openbmc_project/sensors/temperature/cpu0

Service:
 xyz.openbmc_project.Hwmon

Interface:
 xyz.openbmc_project.Sensor.Value
```

bmcweb now knows:

```text
Which object?
    ↓
Which service?
    ↓
Which interface?
```

It can then retrieve the required sensor information.

The actual service and sensor names are platform-dependent; this example is illustrative.

---

# 13. Reading Sensor Properties

The standard D-Bus sensor interface provides:

```text
Value
Unit
MaxValue
MinValue
```

Additional interfaces can provide availability, operational state, thresholds, purpose, and other metadata.

For example, OpenBMC defines interfaces such as:

```text
Sensor.Value
Sensor.ValueMutability
State.Decorator.Availability
State.Decorator.OperationalStatus
Sensor.Purpose
```

The exact combination depends on the sensor.

Source:
https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Sensor/Value.interface.yaml

---

# 14. Threshold Information

A sensor is more useful than a single number.

For example:

```text
Current Value
      ↓
42 °C

Warning High
      ↓
70 °C

Critical High
      ↓
85 °C
```

OpenBMC's sensor ecosystem supports threshold information.

The OpenBMC host-management documentation lists common sensor properties including:

```text
CriticalHigh
CriticalLow
CriticalAlarmHigh
CriticalAlarmLow
WarningHigh
WarningLow
```

along with `Value` and `Unit`.

Source:
https://github.com/openbmc/docs/blob/master/host-management.md

The exact threshold interfaces used can evolve, so the current D-Bus interface definition for the platform should be treated as the source of truth.

---

# 15. Inventory Associations

A sensor can also be associated with an inventory item.

The current `Sensor.Value` interface defines an optional:

```text
inventory
```

association:

```text
Sensor
  ── inventory ──→
Inventory.Item
```

The reverse association is:

```text
sensors
```

The current `Inventory.Item` interface also defines the corresponding sensor association using `xyz.openbmc_project.Sensor.Value`.

Sources:

https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Sensor/Value.interface.yaml

https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Inventory/Item.interface.yaml

Conceptually:

```text
CPU / Board / Chassis
        ↑
     measured by
        │
Temperature Sensor
```

---

# 16. How bmcweb Builds the Sensor JSON

The current `sensors.hpp` contains:

```cpp
objectInterfacesToJson(...)
```

Its purpose is explicitly to build a JSON representation of a sensor.

It receives:

```text
sensorName
sensorType
chassisSubNode
interfacesDict
sensorJson
inventoryItem
```

and passes the interface property dictionaries to:

```cpp
sensor_utils::objectPropertiesToJson(...)
```

to populate the JSON object.

Source:
https://github.com/openbmc/bmcweb/blob/master/redfish-core/lib/sensors.hpp

So:

```text
D-Bus interfaces
      ↓
D-Bus properties
      ↓
sensor_utils
      ↓
nlohmann::json
      ↓
Redfish sensor resource
```

---

# 17. The Key Translation

Suppose a sensor has:

```text
Interface:
xyz.openbmc_project.Sensor.Value

Properties:
Value = 42.0
Unit  = DegreesC
```

bmcweb does not expose that D-Bus object directly.

It converts the information into the Redfish sensor representation.

Conceptually:

```text
D-Bus
--------------------------------
/xyz/openbmc_project/sensors/temperature/cpu0

Sensor.Value:
Value = 42.0
Unit  = DegreesC

             ↓ bmcweb

Redfish
--------------------------------
Sensor resource:

Reading      → 42
ReadingUnits → Cel
```

The exact final Redfish payload depends on the resource/schema and the other interfaces available.

The important lesson is the translation:

> **D-Bus sensor data → bmcweb interpretation → Redfish representation**

---

# 18. Why bmcweb Needs More Than `Value`

Consider:

```text
CPU0 temperature
Board inlet temperature
```

Both could report:

```text
Value = 42
Unit = DegreesC
```

But Redfish also needs resource identity and context.

bmcweb needs to construct information such as:

```text
Sensor name
Sensor type
Chassis relationship
Inventory relationship
Availability
Health/state
Threshold information
Redfish URI
```

That is why the implementation is much more than:

```cpp
read Value;
return Value;
```

It is constructing a complete management resource.

---

# 19. Complete Sensor GET Flow

Suppose a client requests:

```bash
curl -k \
  https://<BMC-IP>/redfish/v1/Chassis/<id>/Thermal/
```

A simplified source-level flow is:

```text
1. Redfish Client
       │
       │ GET
       ▼
2. bmcweb
       │
       ▼
3. Chassis / Thermal processing
       │
       ▼
4. Find chassis D-Bus object
       │
       ▼
5. Follow all_sensors association
       │
       ▼
6. Get relevant sensor object paths
       │
       ▼
7. ObjectMapper getSubTree()
       │
       ▼
8. Find services implementing Sensor.Value
       │
       ▼
9. Read sensor interfaces/properties
       │
       ▼
10. Convert D-Bus data to JSON
       │
       ▼
11. Build Redfish sensor resource
       │
       ▼
12. HTTP response
       │
       ▼
13. Redfish Client
```

This is the central flow of Day 17.

---

# 20. The Sensor Pipeline From Hardware to Redfish

Zooming out:

```text
              HARDWARE
                 │
        Temperature / Voltage
                 │
                 ▼
       Linux / Platform Access
                 │
                 ▼
       OpenBMC Sensor Producer
                 │
                 ▼
              D-Bus
                 │
     /xyz/openbmc_project/sensors
                 │
                 ▼
       Sensor.Value Interface
                 │
                 ▼
       ObjectMapper / Associations
                 │
                 ▼
              bmcweb
                 │
          sensors.hpp /
          sensor_utils.hpp
                 │
                 ▼
          Redfish Resource
                 │
                 ▼
               JSON
                 │
                 ▼
          Remote Client
```

This is the complete conceptual model.

---

# 21. Where `dbus-sensors` Fits

`dbus-sensors` is one common OpenBMC sensor producer.

Its current sensor implementation uses:

```cpp
sdbusplus::asio::object_server
```

and works with interfaces including:

```text
xyz.openbmc_project.Sensor.Value
xyz.openbmc_project.Sensor.ValueMutability
xyz.openbmc_project.State.Decorator.Availability
xyz.openbmc_project.State.Decorator.OperationalStatus
```

Source:
https://github.com/openbmc/dbus-sensors/blob/master/src/sensor.hpp

But:

> **`dbus-sensors` is one implementation path, not a requirement that every sensor must originate there.**

Other services can expose compatible D-Bus sensor interfaces.

---

# 22. A Sensor Is an API, Not Just a Hardware Reading

At the hardware level:

```text
ADC
PMBus
Tachometer
Thermal sensor
```

may all use different access mechanisms.

OpenBMC can abstract those differences:

```text
Hardware-specific access
        ↓
Standard D-Bus sensor interface
        ↓
Common consumers
```

Therefore bmcweb does not need to know:

```text
Which ADC?
Which I²C transaction?
Which PMBus register?
Which hardware controller?
```

It needs to understand the D-Bus sensor contract.

---

# 23. Why This Architecture Scales

Imagine a server with:

```text
20 temperature sensors
30 voltage sensors
10 fan sensors
5 power sensors
```

A common D-Bus representation means consumers can use the same discovery model:

```text
Many sensor producers
        ↓
Common D-Bus interfaces
        ↓
Many consumers
```

For example:

```text
              Sensor.Value
                  │
       ┌──────────┼──────────┐
       ↓          ↓          ↓
   bmcweb       IPMI      Telemetry
       ↓
    Redfish
```

This separation makes the architecture easier to extend.

---

# 24. Sensor Data vs Sensor Identity

Another important distinction:

### Sensor data

```text
Value
Unit
Thresholds
Availability
```

### Sensor identity/context

```text
Name
Type
Object Path
Chassis association
Inventory association
Redfish URI
```

bmcweb needs both.

A value such as:

```text
42 °C
```

has little meaning by itself.

We need:

```text
42 °C
   ↓
CPU0 temperature
   ↓
Chassis 1
   ↓
Redfish Sensor resource
```

The D-Bus associations and bmcweb mapping logic help establish that context.

---

# 25. Reading `sensors.hpp` Efficiently

The current `sensors.hpp` is large, so selective source reading is the right approach.

Search for:

```text
getAssociationEndPoints
getSubTree
Sensor.Value
objectInterfacesToJson
objectPropertiesToJson
```

Then build a small call graph:

```text
Chassis sensor request
        ↓
getChassis()
        ↓
all_sensors association
        ↓
getObjectsWithConnection()
        ↓
getSubTree()
        ↓
D-Bus sensor objects
        ↓
objectInterfacesToJson()
        ↓
sensor_utils::objectPropertiesToJson()
        ↓
Redfish JSON
```

This is the same source-reading technique from Day 16:

> **Start from the external behavior and follow only the functions needed to explain the data path.**

---

# 26. Debugging a Missing Redfish Sensor

Suppose:

```text
Redfish sensor is missing.
```

Follow the pipeline.

### Step 1 — Does the D-Bus sensor exist?

Look under:

```text
/xyz/openbmc_project/sensors
```

### Step 2 — Does it implement:

```text
xyz.openbmc_project.Sensor.Value
```

### Step 3 — Does it have:

```text
Value
Unit
```

### Step 4 — Is it associated with the expected chassis?

Check the relevant association.

### Step 5 — Can ObjectMapper discover it?

Check the mapper response.

### Step 6 — Does bmcweb discover the object?

Inspect bmcweb behavior/logs.

### Step 7 — Does it appear in Redfish?

```bash
curl -k https://<BMC-IP>/redfish/v1/...
```

So debugging becomes:

```text
Hardware
  ↓
Sensor producer
  ↓
D-Bus
  ↓
Associations
  ↓
ObjectMapper
  ↓
bmcweb
  ↓
Redfish
```

---

# 27. Useful BMC-Side Investigation Commands

Once logged into a development BMC, inspect D-Bus directly.

For example:

```bash
busctl tree <sensor-service>
```

Then inspect a sensor:

```bash
busctl introspect \
  <service> \
  /xyz/openbmc_project/sensors/temperature/<sensor>
```

And read the value:

```bash
busctl get-property \
  <service> \
  /xyz/openbmc_project/sensors/temperature/<sensor> \
  xyz.openbmc_project.Sensor.Value \
  Value
```

The exact service name and object path are platform-dependent.

The investigation sequence is:

```text
Find object
   ↓
Inspect interfaces
   ↓
Read Value
   ↓
Inspect Unit / other interfaces
   ↓
Compare with Redfish
```

---

# 28. Comparing D-Bus and Redfish Sensor Data

### D-Bus

```text
Object Path:
 /xyz/openbmc_project/sensors/temperature/cpu0

Interface:
 xyz.openbmc_project.Sensor.Value

Properties:
 Value = 42.0
 Unit  = DegreesC
```

### Redfish

```text
Resource:
 /redfish/v1/Chassis/<id>/Thermal/...

Properties:
 Reading
 ReadingUnits
 Status
 Name
 Id
 @odata.id
 ...
```

The representations are not identical.

```text
D-Bus model
     ↓
bmcweb translation
     ↓
Redfish model
```

This is the architectural bridge we have been studying since Day 15.

---

# 29. Day 16 → Day 17

### Day 16

```text
Redfish Request
      ↓
bmcweb Route
      ↓
C++ Handler
      ↓
D-Bus
      ↓
JSON
```

### Day 17

```text
Sensor Data
      ↓
D-Bus Sensor Object
      ↓
ObjectMapper / Associations
      ↓
bmcweb sensors.hpp
      ↓
Redfish Sensor
```

The new concept is:

> **The D-Bus sensor object is the common integration point between sensor-producing software and higher-level management interfaces.**

---

# 30. Day 13 → Day 17: Our Learning Journey

```text
DAY 13
Source Code
     ↓
Running Service

DAY 14
Service
     ↓
D-Bus API

DAY 15
D-Bus
     ↓
bmcweb
     ↓
Redfish

DAY 16
Redfish Request
     ↓
bmcweb C++
     ↓
D-Bus
     ↓
JSON

DAY 17
Sensor / Hardware Data
     ↓
Sensor Service
     ↓
D-Bus Sensor.Value
     ↓
ObjectMapper + Associations
     ↓
bmcweb sensors.hpp
     ↓
Redfish Sensor
```

We have moved from learning individual components to tracing real information through the OpenBMC stack.

---

# 31. Key Takeaways

### 1. Sensors are represented through D-Bus.

The common interface is:

```text
xyz.openbmc_project.Sensor.Value
```

### 2. Sensor objects live under the OpenBMC sensor hierarchy.

```text
/xyz/openbmc_project/sensors/...
```

### 3. `Value` and `Unit` form the basic sensor reading.

Other interfaces can add availability, operational state, thresholds, purpose, and metadata.

### 4. Associations provide context.

They can connect sensors to chassis/inventory resources.

### 5. ObjectMapper helps bmcweb discover sensor objects.

The current sensor implementation uses `getSubTree()` against the sensor hierarchy and `Sensor.Value`.

### 6. bmcweb translates D-Bus sensor information into Redfish resources.

It does not simply expose the D-Bus object directly.

### 7. Debugging should follow the complete pipeline.

```text
Hardware
 ↓
Sensor Producer
 ↓
D-Bus
 ↓
Association / Discovery
 ↓
bmcweb
 ↓
Redfish
```

---

# 32. Final Mental Model

Remember:

```text
                  HARDWARE
                     │
                     ▼
             Sensor Producer
                     │
                     ▼
                  D-Bus
                     │
       /xyz/openbmc_project/sensors
                     │
                     ▼
          Sensor.Value Interface
                     │
                     ▼
        ObjectMapper / Associations
                     │
                     ▼
                  bmcweb
                     │
             sensors.hpp
                     │
                     ▼
             Redfish Resource
                     │
                     ▼
                   JSON
                     │
                     ▼
              Remote Client
```

### **Day 17 in one sentence:**

> **We followed sensor information from the OpenBMC sensor/D-Bus layer through ObjectMapper and bmcweb until it becomes a Redfish sensor resource that a remote client can consume.**

---

# References

1. OpenBMC `bmcweb` — `redfish-core/lib/sensors.hpp`
2. OpenBMC `bmcweb` — `redfish-core/include/utils/sensor_utils.hpp`
3. OpenBMC `phosphor-dbus-interfaces` — `Sensor.Value`
4. OpenBMC `phosphor-dbus-interfaces` — `Inventory.Item`
5. OpenBMC `dbus-sensors` — sensor implementation
6. OpenBMC documentation — Host Management / Sensors
7. OpenBMC documentation — Telemetry / D-Bus sensors

## Official Source Links

- https://github.com/openbmc/bmcweb/blob/master/redfish-core/lib/sensors.hpp
- https://github.com/openbmc/bmcweb/blob/master/redfish-core/include/utils/sensor_utils.hpp
- https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Sensor/Value.interface.yaml
- https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Inventory/Item.interface.yaml
- https://github.com/openbmc/dbus-sensors/blob/master/src/sensor.hpp
- https://github.com/openbmc/docs/blob/master/host-management.md
- https://github.com/openbmc/docs/blob/master/designs/telemetry.md

---

**OpenBMC Learning Series — Day 17**  
**From Sensor Data → D-Bus → bmcweb → Redfish**
