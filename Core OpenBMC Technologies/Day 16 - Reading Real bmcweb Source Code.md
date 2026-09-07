# 🚀 OpenBMC Learning Series — Day 16

## Reading Real bmcweb Source Code
### From a Redfish Request → bmcweb → D-Bus → JSON Response

---

## 1. Introduction

In the previous days, our learning path moved progressively closer to the real OpenBMC implementation:

```text
Day 13
OpenBMC Source Code
        ↓
Running Service

Day 14
Running Service
        ↓
D-Bus API

Day 15
D-Bus
   ↓
bmcweb
   ↓
Redfish

Day 16
Redfish Request
        ↓
Actual bmcweb C++ Code
        ↓
D-Bus
        ↓
JSON Response
```

Day 15 gave us the architecture. Now we open the actual `bmcweb` source code and answer:

> **What actually happens inside bmcweb when a Redfish client sends a GET request?**

The goal is not to understand the entire bmcweb repository. Instead, we learn how one real request moves through:

```text
HTTP request
    ↓
Redfish route
    ↓
C++ handler
    ↓
D-Bus operations
    ↓
JSON response
```

---

## 2. What Is bmcweb?

`bmcweb` is the web server used by OpenBMC for web-facing interfaces, including Redfish.

For our purpose, the important model is:

```text
Remote Client
      │
   HTTP/HTTPS
      ↓
   bmcweb
      │
     D-Bus
      ↓
OpenBMC Services
```

A useful mental model is:

> **bmcweb connects the external Redfish management model with the internal OpenBMC/D-Bus world.**

It does not simply forward D-Bus messages to the network. It implements Redfish resources and translates between the Redfish model and the OpenBMC/D-Bus model.

---

## 3. Where Does bmcweb Start?

A useful source file is:

```text
bmcweb/src/webserver_run.cpp
```

The current source contains:

```cpp
int runWebserver()
{
    boost::asio::io_context& io = getIoContext();

    App app;

    std::shared_ptr<sdbusplus::asio::connection> systemBus =
        std::make_shared<sdbusplus::asio::connection>(io);

    crow::connections::systemBus = systemBus.get();

    ...
}
```

For Redfish, it initializes:

```cpp
if constexpr (BMCWEB_REDFISH)
{
    redfish::RedfishService::getInstance(app);

    redfish::EventServiceManager::getInstance();

    ...
}
```

The application is then started with:

```cpp
app.run();

systemBus->request_name("xyz.openbmc_project.bmcweb");

io.run();
```

Conceptually:

```text
runWebserver()
      │
      ├── Create I/O context
      ├── Create bmcweb App
      ├── Create D-Bus connection
      ├── Initialize RedfishService
      ├── Initialize routes/features
      ├── Start application
      └── Run event loop
```

---

## 4. Why Does bmcweb Need a D-Bus Connection?

The Redfish client is outside the BMC, while OpenBMC services are inside the BMC.

Therefore bmcweb needs a communication path to the internal services:

```text
Remote Client
      │
   HTTP/HTTPS
      ↓
   bmcweb
      │
     D-Bus
      ↓
OpenBMC Services
```

The source explicitly creates an:

```cpp
sdbusplus::asio::connection
```

and makes it available to bmcweb components.

This directly connects to Day 14:

> **D-Bus is the internal communication mechanism; bmcweb uses it to obtain or modify information needed by Redfish resources.**

---

## 5. What Is a Redfish Route?

A Redfish URI must eventually be connected to C++ code.

bmcweb uses route registration such as:

```cpp
BMCWEB_ROUTE(app, "/redfish/v1/Managers/<str>/")
```

and specifies the HTTP method:

```cpp
.methods(boost::beast::http::verb::get)
```

and the handler:

```cpp
(std::bind_front(handleManagerGet, std::ref(app)));
```

Conceptually:

```text
GET /redfish/v1/Managers/bmc/
          ↓
Route matching
          ↓
/redfish/v1/Managers/<str>/
          ↓
handleManagerGet()
```

When reading a route, ask:

```text
1. Which URI?
2. Which HTTP method?
3. Which privilege?
4. Which handler?
```

---

## 6. A Real Route From Current bmcweb Source

The current `redfish-core/lib/managers.hpp` contains:

```cpp
BMCWEB_ROUTE(app, "/redfish/v1/Managers/<str>/")
    .privileges(redfish::privileges::getManager)
    .methods(boost::beast::http::verb::get)(
        std::bind_front(handleManagerGet, std::ref(app)));
```

This tells us:

### URI

```text
/redfish/v1/Managers/<str>/
```

### HTTP method

```text
GET
```

### Required privilege

```text
getManager
```

### Handler

```text
handleManagerGet()
```

For:

```text
/redfish/v1/Managers/bmc/
```

the `<str>` path parameter becomes the manager identifier.

---

## 7. The Handler: Where the Request Becomes C++ Logic

The corresponding handler is:

```cpp
inline void handleManagerGet(
    App& app,
    const crow::Request& req,
    const std::shared_ptr<bmcweb::AsyncResp>& asyncResp,
    const std::string& managerId)
```

The handler receives:

```text
App
Request
Async response
Manager ID
```

The path:

```text
/redfish/v1/Managers/<str>/
```

therefore becomes C++ data:

```text
managerId = "bmc"
```

The handler is the bridge between:

```text
HTTP request
      ↓
Redfish resource logic
```

---

## 8. First Thing the Handler Does: Route Setup

The handler begins with:

```cpp
if (!redfish::setUpRedfishRoute(app, req, asyncResp))
{
    return;
}
```

Before resource-specific work, bmcweb performs common Redfish route setup.

So do not mentally model a handler as:

```text
Immediately read D-Bus
```

Instead:

```text
Request
   ↓
Common Redfish setup
   ↓
Resource-specific processing
```

This is typical production-code structure: framework-level setup, authorization, validation and response preparation can occur before resource-specific logic.

---

## 9. Validate the Resource Identifier

The manager handler checks the requested manager ID:

```cpp
if (managerId != BMCWEB_REDFISH_MANAGER_URI_NAME)
{
    messages::resourceNotFound(
        asyncResp->res,
        "Manager",
        managerId);

    return;
}
```

Conceptually:

```text
GET /redfish/v1/Managers/<id>/
          ↓
Is <id> supported?
       /          Yes      No
      ↓        ↓
 Continue    Not Found
```

This demonstrates that a route is not merely a URI mapping. The handler contains actual application logic.

---

## 10. Building the Redfish Response

The handler starts constructing the Redfish response:

```cpp
asyncResp->res.jsonValue["@odata.id"] =
    boost::urls::format(
        "/redfish/v1/Managers/{}",
        managerId);

asyncResp->res.jsonValue["Id"] = managerId;

asyncResp->res.jsonValue["Name"] =
    "OpenBmc Manager";

asyncResp->res.jsonValue["Description"] =
    "Baseboard Management Controller";
```

The important lesson is:

> The handler is not simply returning raw D-Bus data.

It is constructing a Redfish resource using the Redfish JSON model.

```text
OpenBMC information
        ↓
   bmcweb logic
        ↓
Redfish JSON resource
```

---

## 11. Static Data vs D-Bus Data

A Redfish response can contain information that does not require a D-Bus lookup.

The manager handler directly sets or derives values such as:

```text
Name
Description
PowerState
ManagerType
UUID
OData identifiers
Redfish links
Actions
```

Other information is retrieved asynchronously from D-Bus.

So a response may be assembled from multiple sources:

```text
                 ┌── Static / derived values
                 │
Request → Handler├── D-Bus properties
                 │
                 └── Other OpenBMC helpers
                          ↓
                     JSON response
```

This is another reason why:

> **Redfish is not simply “D-Bus over HTTP.”**

---

## 12. A Real D-Bus Property Read

The current manager implementation contains:

```cpp
dbus::utility::getProperty<double>(
    "org.freedesktop.systemd1",
    "/org/freedesktop/systemd1",
    "org.freedesktop.systemd1.Manager",
    "Progress",
    ...);
```

Break it down:

```text
Service
  ↓
org.freedesktop.systemd1

Object
  ↓
/org/freedesktop/systemd1

Interface
  ↓
org.freedesktop.systemd1.Manager

Property
  ↓
Progress
```

This is exactly the D-Bus model from Day 14:

```text
Service → Object → Interface → Property
```

---

## 13. What Happens After the D-Bus Property Is Returned?

The operation is asynchronous. Its callback receives:

```cpp
const boost::system::error_code& ec,
double val
```

The code first checks:

```cpp
if (ec)
{
    messages::internalError(asyncResp->res);
    return;
}
```

If successful, the value is interpreted by bmcweb.

For example, the manager code checks whether:

```text
Progress < 1.0
```

and can set:

```text
Status.State = Starting
```

Otherwise it continues to additional state checks.

The important pattern is:

```text
D-Bus result
     ↓
Callback
     ↓
Interpret result
     ↓
Map to Redfish state
```

The D-Bus value is not automatically a Redfish value. bmcweb decides how it should be represented.

---

## 14. ObjectMapper: Finding the Correct D-Bus Object

The manager handler also uses:

```cpp
manager_utils::getValidManagerPath(
    asyncResp,
    managerId,
    std::bind_front(getManagerData, asyncResp));
```

OpenBMC's ObjectMapper provides APIs for discovering D-Bus objects and the services/interfaces that implement them.

Important ObjectMapper operations include:

```text
GetObject
    → Find services/interfaces implementing an object path

GetSubTree
    → Find objects, services and interfaces in a subtree

GetSubTreePaths
    → Find matching object paths
```

Conceptually:

```text
bmcweb
   ↓
ObjectMapper
   ↓
Which D-Bus object?
   ↓
Which service owns it?
   ↓
Which interfaces?
   ↓
Read required properties
```

This is especially useful because OpenBMC objects and services can be discovered dynamically.

---

## 15. Real Example: Manager Data From D-Bus

The manager source contains:

```text
getManagerData(...)
```

which receives information about:

```text
managerPath
serviceMap
```

It examines the interfaces available on the discovered object.

For example, if it finds:

```text
xyz.openbmc_project.Inventory.Decorator.Asset
```

it calls:

```cpp
dbus::utility::getAllProperties(
    *crow::connections::systemBus,
    connectionName,
    managerPath,
    "xyz.openbmc_project.Inventory.Decorator.Asset",
    ...);
```

The returned properties can include:

```text
PartNumber
SerialNumber
Manufacturer
Model
SparePartNumber
```

The callback then places those values into the Redfish response.

So:

```text
ObjectMapper
     ↓
Find manager object
     ↓
Find service + interfaces
     ↓
getAllProperties()
     ↓
D-Bus property values
     ↓
bmcweb
     ↓
Redfish JSON
```

---

## 16. How D-Bus Data Becomes JSON

The source contains assignments such as:

```cpp
if (partNumber != nullptr)
{
    asyncResp->res.jsonValue["PartNumber"] = *partNumber;
}

if (serialNumber != nullptr)
{
    asyncResp->res.jsonValue["SerialNumber"] = *serialNumber;
}

if (manufacturer != nullptr)
{
    asyncResp->res.jsonValue["Manufacturer"] = *manufacturer;
}
```

Suppose D-Bus provides:

```text
PartNumber = "ABC123"
SerialNumber = "SN001"
Manufacturer = "ExampleCorp"
```

bmcweb can construct:

```json
{
    "PartNumber": "ABC123",
    "SerialNumber": "SN001",
    "Manufacturer": "ExampleCorp"
}
```

The translation is:

```text
D-Bus property
       ↓
C++ value
       ↓
nlohmann::json
       ↓
Redfish response
```

---

## 17. Why Async Responses Are Used

bmcweb commonly uses:

```cpp
std::shared_ptr<bmcweb::AsyncResp>
```

because D-Bus operations are asynchronous.

A request may require several operations:

```text
HTTP request
    ↓
D-Bus lookup
    ↓
callback
    ↓
another D-Bus property read
    ↓
callback
    ↓
JSON completed
    ↓
HTTP response
```

A simplified mental model is:

```text
Request arrives
      ↓
Create AsyncResp
      ↓
Start asynchronous D-Bus operations
      ↓
Callbacks modify JSON
      ↓
Required work completes
      ↓
Response is returned
```

---

## 18. A Critical Source-Reading Skill

Not every function called from a handler is implemented in the same file.

For example:

```cpp
manager_utils::getValidManagerPath(...)
```

leads into utility code.

Likewise:

```cpp
dbus::utility::getProperty(...)
```

belongs to bmcweb's D-Bus utility layer.

Therefore:

```text
Handler
  ↓
Utility
  ↓
Helper
  ↓
D-Bus operation
```

is normal in a production repository.

### A repeatable reading method

**Step 1 — Start from the route**

```text
BMCWEB_ROUTE(...)
```

**Step 2 — Find the handler**

```text
handleXXXGet()
```

**Step 3 — Mark important helpers**

```text
setUpRedfishRoute()
getValidXXXPath()
getProperty()
getAllProperties()
```

**Step 4 — Open those helper implementations.**

**Step 5 — Stop when you reach the actual D-Bus operation.**

This is much more manageable than trying to understand the entire repository at once.

---

## 19. GET, PATCH and POST in Real bmcweb

For the Manager resource, current bmcweb registers:

```text
GET
/redfish/v1/Managers/<str>/
```

and:

```text
PATCH
/redfish/v1/Managers/<str>/
```

It also registers action routes such as:

```text
POST
/redfish/v1/Managers/<str>/Actions/Manager.Reset/
```

So the source-level mental model becomes:

```text
GET
 ↓
Read resource

PATCH
 ↓
Modify supported properties

POST
 ↓
Invoke a supported action
```

The exact implementation is resource-specific.

---

## 20. Real POST Example: Manager Reset

The current manager implementation contains a route for:

```text
/redfish/v1/Managers/<str>/Actions/Manager.Reset/
```

with:

```text
POST
```

The handler reads:

```text
ResetType
```

and supports:

```text
GracefulRestart
ForceRestart
```

The selected operation is translated into a BMC state transition.

For example:

```cpp
setBMCTransition(
    asyncResp,
    "xyz.openbmc_project.State.BMC.Transition.Reboot");
```

or:

```cpp
setBMCTransition(
    asyncResp,
    "xyz.openbmc_project.State.BMC.Transition.HardReboot");
```

The D-Bus property involved is:

```text
RequestedBMCTransition
```

on:

```text
xyz.openbmc_project.State.BMC
```

So the source-level flow is:

```text
Redfish POST
     ↓
Manager.Reset route
     ↓
handleManagerResetAction()
     ↓
Read ResetType
     ↓
Choose transition
     ↓
D-Bus setProperty()
     ↓
OpenBMC BMC state service
```

This is a concrete example of an external Redfish action becoming an internal D-Bus operation.

---

## 21. GET vs PATCH vs POST: Source-Level Mental Model

### GET

```text
Client
  ↓
Redfish GET
  ↓
Route
  ↓
Handler
  ↓
D-Bus read(s)
  ↓
Build JSON
  ↓
Response
```

### PATCH

```text
Client
  ↓
Redfish PATCH
  ↓
Route
  ↓
Handler
  ↓
Parse JSON body
  ↓
Validate supported properties
  ↓
D-Bus write(s)
  ↓
Response
```

### POST

```text
Client
  ↓
Redfish POST
  ↓
Action route
  ↓
Handler
  ↓
Parse action parameters
  ↓
Perform operation
  ↓
D-Bus method/property/action
  ↓
Response
```

---

## 22. Complete GET Flow

Suppose the client sends:

```bash
curl -k https://<BMC-IP>/redfish/v1/Managers/bmc/
```

The conceptual flow is:

```text
1. Client
   │
   │ HTTP GET
   ▼
2. bmcweb
   │
   │ Route matching
   ▼
3. /redfish/v1/Managers/<str>/
   │
   ▼
4. handleManagerGet()
   │
   ├── setUpRedfishRoute()
   ├── validate managerId
   ├── populate static Redfish fields
   ├── D-Bus property reads
   └── manager path/data lookup
   │
   ▼
5. ObjectMapper / D-Bus utilities
   │
   ▼
6. OpenBMC / system services
   │
   ▼
7. D-Bus results
   │
   ▼
8. bmcweb callbacks
   │
   ▼
9. nlohmann::json response
   │
   ▼
10. HTTP response
   │
   ▼
11. curl / Redfish client
```

This is the complete connection between remote management and internal OpenBMC software.

---

## 23. Why This Is Not "D-Bus Over HTTP"

A simplistic model would be:

```text
HTTP
 ↓
Take D-Bus message
 ↓
Send it over HTTP
```

That is not what the source shows.

Instead:

```text
HTTP Redfish request
        ↓
Redfish route
        ↓
C++ resource handler
        ↓
Interpret request
        ↓
Discover D-Bus objects
        ↓
Read/write/call D-Bus APIs
        ↓
Interpret returned values
        ↓
Construct Redfish JSON
        ↓
HTTP response
```

The models are different.

### D-Bus

```text
Service
Object
Interface
Property
Method
Signal
```

### Redfish

```text
Resource
Property
Action
Link
JSON representation
HTTP operation
```

bmcweb connects these two models.

---

## 24. Where Does This Code Live?

A useful high-level view is:

```text
bmcweb/
│
├── src/
│   ├── webserver_run.cpp
│   └── ...
│
├── redfish-core/
│   ├── lib/
│   │   ├── managers.hpp
│   │   ├── chassis.hpp
│   │   ├── sensors.hpp
│   │   ├── power.hpp
│   │   └── ...
│   │
│   ├── src/
│   │   └── utils/
│   │       └── dbus_utils.cpp
│   │
│   └── include/
│
├── http/
├── routing/
├── dbus/
└── main.cpp
```

Broadly:

```text
src/
    → application/server infrastructure

redfish-core/lib/
    → Redfish resource implementations

redfish-core/src/utils/
    → supporting Redfish/D-Bus utilities

dbus/
    → D-Bus-related infrastructure

routing/
    → HTTP routing infrastructure
```

The repository structure can evolve, so the current source tree remains the source of truth.

---

## 25. How to Read a New bmcweb Resource

Suppose we want to understand:

```text
/redfish/v1/Chassis/
```

Do not begin by reading thousands of lines.

### Step 1 — Find the route

Search for:

```text
BMCWEB_ROUTE
```

and:

```text
Chassis
```

### Step 2 — Find the HTTP method

Look for:

```cpp
.methods(...)
```

### Step 3 — Find the handler

Identify the relevant:

```text
handle...Get()
```

### Step 4 — Identify D-Bus helpers

Search for:

```text
getProperty
getAllProperties
getSubTree
getObject
async_method_call
setProperty
```

### Step 5 — Follow ObjectMapper calls

Determine:

```text
Which object path?
Which service?
Which interface?
```

### Step 6 — Follow JSON assignments

Look for:

```cpp
asyncResp->res.jsonValue[...]
```

### Step 7 — Reconstruct the complete flow

```text
URI
 ↓
Route
 ↓
Handler
 ↓
D-Bus lookup
 ↓
D-Bus operation
 ↓
JSON mapping
 ↓
HTTP response
```

---

## 26. A Very Important Observation About Async Code

bmcweb can initially look confusing because the source is not always visually sequential.

For example:

```cpp
dbus::utility::getProperty(
    ...,
    [](const boost::system::error_code& ec,
       const auto& value)
    {
        ...
    });
```

The D-Bus request starts now, but the callback executes later when the result arrives.

Mentally read it as:

```text
Start D-Bus request
       ↓
Continue event loop
       ↓
D-Bus response arrives
       ↓
Callback executes
       ↓
Update Redfish response
```

This is essential when reading bmcweb.

---

## 27. Error Handling Is Part of the Translation

Production code also handles failures.

For example:

```cpp
if (ec)
{
    messages::internalError(asyncResp->res);
    return;
}
```

Or when an object cannot be found:

```cpp
messages::resourceNotFound(...)
```

Therefore:

```text
D-Bus success
     ↓
Redfish data

OR

D-Bus failure
     ↓
Redfish error response
```

The translation layer includes error mapping as well as data mapping.

---

## 28. Day 15 vs Day 16

### Day 15

We learned:

```text
What is Redfish?
What is bmcweb?
How does bmcweb connect to D-Bus?
What does a Redfish resource look like?
How do GET/PATCH/POST work conceptually?
```

### Day 16

We opened the implementation:

```text
Where bmcweb starts
        ↓
How routes are registered
        ↓
How a route selects a handler
        ↓
How the handler validates requests
        ↓
How D-Bus properties are read
        ↓
How ObjectMapper helps discover objects
        ↓
How D-Bus data is translated into JSON
        ↓
How POST actions can trigger D-Bus operations
```

We have moved from architecture to implementation.

---

## 29. Day 13 → Day 16: Complete Learning Progression

```text
DAY 13
Reading OpenBMC Source
        │
        ▼
Source Code → Running Service


DAY 14
Understanding D-Bus API
        │
        ▼
Running Service → D-Bus API


DAY 15
Understanding Redfish
        │
        ▼
D-Bus → bmcweb → Redfish


DAY 16
Reading bmcweb Source
        │
        ▼
Redfish Request
        ↓
bmcweb Route
        ↓
C++ Handler
        ↓
D-Bus
        ↓
JSON Response
```

We are no longer only learning OpenBMC concepts.

We are learning how to read the implementation that connects those concepts together.

---

## 30. Complete Mental Model

```text
                 EXTERNAL WORLD
                       │
                       │ HTTP / HTTPS
                       ▼
              ┌─────────────────┐
              │     Redfish     │
              │  Resource Model │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │     bmcweb      │
              │                 │
              │  Route          │
              │    ↓            │
              │  Handler        │
              │    ↓            │
              │  D-Bus Helpers  │
              └────────┬────────┘
                       │
                       │ D-Bus
                       ▼
              ┌─────────────────┐
              │ OpenBMC Services│
              │                 │
              │ Sensors         │
              │ Inventory       │
              │ State           │
              │ Control         │
              │ Software        │
              └────────┬────────┘
                       │
                       ▼
                 BMC Hardware
```

Reverse direction:

```text
Hardware / Service State
          ↓
        D-Bus
          ↓
       bmcweb
          ↓
   Redfish JSON
          ↓
    HTTP Response
          ↓
   Remote Client
```

---

## 31. Key Takeaways

### 1. A Redfish URI is connected to real C++ code.

```text
BMCWEB_ROUTE
      ↓
Handler
```

### 2. The handler implements resource-specific behavior.

It validates requests, performs operations, retrieves data, and constructs the response.

### 3. bmcweb uses D-Bus to interact with internal OpenBMC services.

The current source uses `sdbusplus` and bmcweb D-Bus utilities with asynchronous operations.

### 4. ObjectMapper helps discover D-Bus objects and services.

This is especially useful when the correct object/service cannot simply be hard-coded.

### 5. D-Bus data is translated into Redfish JSON.

bmcweb implements the Redfish representation rather than merely forwarding D-Bus messages.

### 6. GET, PATCH and POST lead to different source-level behavior.

```text
GET   → read
PATCH → modify
POST  → action/operation
```

### 7. bmcweb is asynchronous.

D-Bus results often arrive through callbacks, so source-code execution is not always visually sequential.

---

## 32. What Should We Learn Next?

We have now learned how to read a real bmcweb route and trace its relationship with D-Bus.

The next natural step is to go deeper into an important OpenBMC use case:

```text
Redfish Sensor
      ↓
bmcweb sensor code
      ↓
ObjectMapper
      ↓
D-Bus Sensor.Value
      ↓
Threshold interfaces
      ↓
Redfish Sensor JSON
```

That would allow us to follow hardware sensor data all the way from an OpenBMC sensor service to the external Redfish API.

---

## 33. Final Mental Model

If you remember only one diagram from Day 16:

```text
        curl / Redfish Client
                 │
                 │ HTTP GET
                 ▼
       ┌───────────────────┐
       │   bmcweb Route    │
       └─────────┬─────────┘
                 │
                 ▼
       ┌───────────────────┐
       │   C++ Handler     │
       └─────────┬─────────┘
                 │
          D-Bus Helpers
                 │
                 ▼
       ┌───────────────────┐
       │ ObjectMapper /    │
       │ D-Bus Services    │
       └─────────┬─────────┘
                 │
                 ▼
          D-Bus Properties
          / Methods / Data
                 │
                 ▼
       ┌───────────────────┐
       │   bmcweb JSON     │
       │     Response      │
       └─────────┬─────────┘
                 │
                 ▼
          HTTP Response
                 │
                 ▼
             Client
```

### **Day 16 in one sentence:**

> **We moved from understanding what bmcweb does to reading the actual C++ code that turns a Redfish request into D-Bus operations and turns the results back into a Redfish JSON response.**

---

# References

1. OpenBMC `bmcweb` — `src/webserver_run.cpp`
2. OpenBMC `bmcweb` — `redfish-core/lib/managers.hpp`
3. OpenBMC `bmcweb` — `redfish-core/src/utils/dbus_utils.cpp`
4. OpenBMC `docs` — ObjectMapper architecture documentation
5. OpenBMC `bmcweb` — Redfish resource implementations
6. OpenBMC `phosphor-dbus-interfaces` — D-Bus interface definitions

## Official Source Links

- https://github.com/openbmc/bmcweb
- https://github.com/openbmc/bmcweb/blob/master/src/webserver_run.cpp
- https://github.com/openbmc/bmcweb/blob/master/redfish-core/lib/managers.hpp
- https://github.com/openbmc/bmcweb/blob/master/redfish-core/src/utils/dbus_utils.cpp
- https://github.com/openbmc/docs/blob/master/architecture/object-mapper.md
- https://github.com/openbmc/phosphor-dbus-interfaces

---

**OpenBMC Learning Series — Day 16**  
**From Redfish Architecture → Actual bmcweb Source Code**
