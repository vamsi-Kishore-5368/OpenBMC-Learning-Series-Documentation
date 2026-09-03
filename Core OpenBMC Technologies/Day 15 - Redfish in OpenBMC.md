# 🚀 OpenBMC Learning Series — Day 15

## Redfish in OpenBMC — From D-Bus to a Remote Management API

> **Day 13:** How to read an OpenBMC service  
> **Day 14:** What does that service expose on D-Bus?  
> **Day 15:** How does a remote management client access OpenBMC functionality?

---

## Introduction

In the previous days, we moved deeper into the OpenBMC software stack.

We started by reading an actual OpenBMC component and understanding how source code becomes a running service. Then, in Day 14, we examined the D-Bus model used by OpenBMC services: service names, object paths, interfaces, properties, methods, and signals.

But there is still an important question:

> **How does software outside the BMC communicate with these OpenBMC services?**

A remote management application does not normally connect directly to an OpenBMC D-Bus service.

Instead, OpenBMC can expose management functionality through **Redfish**, with **bmcweb** providing the Redfish web service.

The high-level path is:

```text
Remote Management Client
          │
          │ HTTP / HTTPS
          ▼
       bmcweb
          │
          │ D-Bus
          ▼
   OpenBMC Services
          │
          ▼
 BMC Hardware / Sensors / Host
```

> **Central idea:** D-Bus is primarily an internal communication mechanism inside OpenBMC, while Redfish provides a standardized management API for external clients. bmcweb connects these worlds.

---

# 1. What Is Redfish?

**Redfish** is a standard, RESTful management interface designed for managing servers and infrastructure.

It provides a standardized resource model that management software can access over HTTP/HTTPS.

Instead of a management application needing to understand the internal implementation of every vendor's BMC, it can communicate using standardized Redfish resources and schemas.

A Redfish client may send a request such as:

```http
GET /redfish/v1/
```

and receive a JSON representation of the Redfish service root.

The Redfish Service Root is located at:

```text
/redfish/v1/
```

and acts as the starting point for navigating the resources exposed by a Redfish service.

---

# 2. Why Do We Need Redfish?

At this point, we already know about:

- D-Bus
- IPMI
- MCTP
- PLDM

So a natural question is:

> **Why do we need another management interface?**

The answer is that these technologies solve different problems.

D-Bus is primarily used for **communication between applications inside OpenBMC**.

Redfish is intended to provide a **standardized management interface to external clients**.

Conceptually:

```text
                 INSIDE THE BMC

     OpenBMC Service
             │
             │ D-Bus
             ▼
     Other OpenBMC Service
```

while:

```text
                 OUTSIDE THE BMC

     Management Application
             │
             │ HTTP / HTTPS
             ▼
           Redfish
             │
             ▼
          bmcweb
             │
             │ D-Bus
             ▼
     OpenBMC Services
```

So Redfish does not replace D-Bus.

They operate at different levels.

---

# 3. D-Bus vs Redfish

| | D-Bus | Redfish |
|---|---|---|
| Main purpose | Internal IPC | External management API |
| Typical communication | OpenBMC application ↔ OpenBMC application | Management client ↔ BMC |
| Model | Objects, interfaces, properties, methods, signals | Resources, properties, actions, links |
| Typical transport | D-Bus | HTTP/HTTPS |
| Typical consumer | OpenBMC services | Remote management software |
| OpenBMC component | D-Bus services | bmcweb |

A Redfish client does **not** need to know how the underlying OpenBMC service is implemented.

For example, the client should not need to know:

```text
Which C++ class?
Which D-Bus service?
Which object path?
Which sdbusplus binding?
Which systemd unit?
```

It interacts with the standardized Redfish interface.

---

# 4. What Is bmcweb?

`bmcweb` is the OpenBMC web server that implements Redfish and other web-based management interfaces.

For Day 15, the most useful mental model is:

```text
                    bmcweb

HTTP/HTTPS
    │
    ▼
Redfish request
    │
    ▼
Redfish resource implementation
    │
    ▼
D-Bus interaction
    │
    ▼
OpenBMC services
```

OpenBMC's bmcweb documentation describes its Redfish implementation and the Redfish schemas/resources supported by the project.

In many resource implementations, bmcweb performs D-Bus lookups and property/method operations, then constructs the Redfish response.

It is therefore useful to think of bmcweb as a **bridge/translation layer** between the external Redfish model and internal OpenBMC services.

> **Important:** This does not mean every Redfish resource maps one-to-one to exactly one D-Bus object. The mapping is resource-specific and can involve multiple D-Bus objects, interfaces, associations, or implementation logic.

---

# 5. The Redfish Service Root

The starting point of a Redfish service is:

```text
/redfish/v1/
```

This is the **Redfish Service Root**.

It provides links to other resources.

For example, an OpenBMC bmcweb installation can expose resources such as:

```text
/redfish/v1/
    ├── AccountService/
    ├── Chassis/
    ├── Managers/
    ├── Systems/
    ├── EventService/
    ├── SessionService/
    ├── Tasks/
    └── UpdateService/
```

The exact set of supported resources depends on the implementation and configuration.

The important concept is:

```text
/redfish/v1/
      │
      ├── Managers
      ├── Systems
      ├── Chassis
      ├── AccountService
      └── ...
```

The Service Root is therefore the entry point into the Redfish resource tree.

---

# 6. Understanding Redfish Resources

Redfish organizes managed entities as **resources**.

For example:

```text
/redfish/v1/Chassis/chassis0
```

can represent a chassis resource.

A resource can contain properties describing its current state.

Conceptually:

```json
{
    "@odata.id": "/redfish/v1/Chassis/chassis0",
    "Id": "chassis0",
    "Name": "Chassis",
    "PowerState": "On"
}
```

The exact properties depend on the Redfish schema for that resource.

Common Redfish resource metadata includes:

```text
@odata.id
@odata.type
Id
Name
```

The schema defines what properties mean and what data types and constraints apply.

---

# 7. Properties and Actions

This is where Redfish becomes similar to the D-Bus model we learned in Day 14, but the terminology is different.

### D-Bus

```text
Properties → State / Data
Methods    → Operations
Signals    → Events
```

### Redfish

```text
Properties → Resource state / data
Actions    → Operations
Events     → Eventing mechanisms
```

For example, a Redfish resource can expose properties describing its current state:

```json
{
    "PowerState": "On",
    "Status": {
        "State": "Enabled",
        "Health": "OK"
    }
}
```

When an operation does not fit naturally as a property update, Redfish can expose an **Action**.

For example, a chassis can expose a reset action.

Conceptually:

```text
POST
/redfish/v1/Chassis/chassis0/Actions/Chassis.Reset
```

The exact URI and supported action parameters depend on the Redfish resource schema and implementation.

---

# 8. Redfish URI Structure

A Redfish URI identifies a resource.

A simplified example is:

```text
/redfish/v1/Chassis/chassis0
```

Break it down:

```text
/redfish
    │
    └── v1
         │
         └── Chassis
               │
               └── chassis0
```

Another example:

```text
/redfish/v1/Chassis/chassis0/Sensors
```

Here:

```text
/redfish/v1/
       ↓
    Chassis
       ↓
    chassis0
       ↓
    Sensors
```

The URI hierarchy allows a client to navigate the Redfish resource model.

---

# 9. JSON: The Data Returned to the Client

Redfish resource responses are represented as JSON payloads.

For example:

```json
{
    "@odata.id": "/redfish/v1/Chassis/chassis0",
    "@odata.type": "#Chassis.v1_22_0.Chassis",
    "Id": "chassis0",
    "Name": "Chassis",
    "PowerState": "On"
}
```

The schema determines what properties mean and what data types and constraints apply.

This is an important difference from simply returning arbitrary JSON.

Redfish is a **schema-driven management model**.

The client can use the Redfish schema to understand the meaning of the properties it receives.

---

# 10. How Does bmcweb Connect Redfish to D-Bus?

Now we reach the most important OpenBMC-specific part.

Suppose an OpenBMC component exposes information on D-Bus.

For example:

```text
D-Bus Object
    │
    └── Interface
          │
          ├── Property A
          ├── Property B
          └── Property C
```

bmcweb can discover the relevant D-Bus object/interface and obtain the information it needs.

It then builds the corresponding Redfish representation:

```text
D-Bus
  │
  │ Read properties / perform operations
  ▼
bmcweb
  │
  │ Build Redfish resource
  ▼
JSON response
```

The important point is:

> **bmcweb translates between two different models.**

Internal model:

```text
D-Bus
Service
Object Path
Interface
Property
Method
```

External model:

```text
Redfish
Resource
URI
Property
Action
```

---

# 11. Real Example: Chassis and Sensors

A useful real example is the Chassis resource and its sensors.

Current bmcweb documentation exposes sensor-related Redfish resources under the Chassis hierarchy, including:

```text
/redfish/v1/Chassis/{ChassisId}/Sensors/
/redfish/v1/Chassis/{ChassisId}/Sensors/{Id}/
```

It also exposes thermal resources containing temperature and fan information.

Conceptually:

```text
Hardware Sensor
      │
      ▼
OpenBMC Sensor Infrastructure
      │
      ▼
D-Bus
      │
      ▼
bmcweb
      │
      ▼
Redfish Sensor Resource
      │
      ▼
JSON
```

The actual implementation is more detailed than a simple one-to-one mapping.

For example, current bmcweb sensor code uses D-Bus subtree/association lookups and constructs Redfish collections and sensor properties.

This is a good example of why we should think of bmcweb as a **translation/aggregation layer**, rather than assuming a direct object-to-object mapping.

---

# 12. Real Source Code: bmcweb Chassis Example

Let's connect the architecture to actual source code.

In bmcweb's Chassis implementation, a Redfish route is registered for:

```text
/redfish/v1/Chassis/
```

and another route handles:

```text
/redfish/v1/Chassis/<str>/
```

The route implementation can then perform D-Bus operations.

For example, the current bmcweb Chassis implementation contains calls that retrieve D-Bus properties such as:

```text
xyz.openbmc_project.Chassis.Intrusion
```

and the `Status` property.

It also uses D-Bus mapper functionality to discover objects and interfaces.

Conceptually:

```text
HTTP GET
   │
   ▼
bmcweb Chassis route
   │
   ▼
D-Bus mapper / property lookup
   │
   ▼
OpenBMC D-Bus object
   │
   ▼
Property value
   │
   ▼
Redfish JSON response
```

This is exactly the kind of source-code connection we will learn to trace in later OpenBMC days.

---

# 13. What Happens During a Redfish GET?

Let's trace one request.

Suppose a client asks for:

```http
GET /redfish/v1/Chassis/chassis0
```

The high-level flow is:

```text
1. Management Client
          │
          │ HTTP GET
          ▼
2. bmcweb
          │
          │ Match Redfish route
          ▼
3. Chassis implementation
          │
          │ D-Bus queries
          ▼
4. OpenBMC D-Bus services
          │
          │ Return properties/data
          ▼
5. bmcweb
          │
          │ Build JSON
          ▼
6. HTTP Response
          │
          ▼
7. Management Client
```

So the client sees:

```text
Redfish JSON
```

while bmcweb may internally interact with:

```text
D-Bus services
D-Bus objects
D-Bus interfaces
D-Bus properties
D-Bus associations
```

---

# 14. GET vs PATCH vs POST

Redfish commonly uses HTTP methods according to the operation being performed.

### GET — Read

```http
GET /redfish/v1/Chassis/chassis0
```

Used to retrieve a resource.

### PATCH — Modify

Conceptually:

```http
PATCH /redfish/v1/Chassis/chassis0
```

with a JSON body containing supported property changes.

Not every property is writable, and the resource schema determines what can be modified.

### POST — Perform an operation / create something

For example, a Redfish Action is generally invoked using POST.

Conceptually:

```http
POST /redfish/v1/Chassis/chassis0/Actions/Chassis.Reset
```

The supported action and parameters depend on the resource schema.

So we can remember:

```text
GET    → Read
PATCH  → Modify supported properties
POST   → Invoke actions / create resources where supported
```

---

# 15. Exploring Redfish with curl

If we have access to a BMC running bmcweb, we can investigate Redfish directly.

For example:

```bash
curl -k https://<BMC-IP>/redfish/v1/
```

Here:

```text
curl
  ↓
HTTPS request
  ↓
bmcweb
  ↓
Redfish Service Root
  ↓
JSON response
```

We can then explore a resource:

```bash
curl -k https://<BMC-IP>/redfish/v1/Chassis/
```

and, if the resource exists:

```bash
curl -k https://<BMC-IP>/redfish/v1/Chassis/<ChassisId>
```

### What does `-k` mean?

It tells curl not to verify the server's TLS certificate.

This can be useful during development when the BMC uses a self-signed certificate.

**It should not be treated as a recommended production security practice.**

---

# 16. Authentication and Authorization

A real BMC should not allow unrestricted management operations to unauthenticated clients.

The simplified flow is:

```text
Client
   │
   │ HTTPS
   ▼
Authentication
   │
   ▼
Authorization
   │
   ▼
Redfish Resource
```

Authentication answers:

> **Who are you?**

Authorization answers:

> **What are you allowed to do?**

bmcweb supports multiple authentication mechanisms. The exact mechanisms and configuration can vary.

For Day 15, the important point is simply:

> **Redfish is a management interface, so access control is part of the BMC's management architecture.**

A dedicated future topic can examine Redfish authentication and sessions in much greater detail.

---

# 17. Where Does Redfish Code Live in bmcweb?

When we eventually study bmcweb source code, one useful area to recognize is:

```text
bmcweb/
└── redfish-core/
    └── lib/
```

Current bmcweb source contains resource-specific implementation files such as:

```text
chassis.hpp
sensors.hpp
power.hpp
...
```

These implementations contain:

- Redfish route registration
- request handling
- Redfish JSON construction
- D-Bus lookups
- D-Bus property access
- validation
- error handling

For example:

```text
redfish-core/lib/chassis.hpp
```

contains Chassis Redfish route implementations and D-Bus interactions.

Similarly:

```text
redfish-core/lib/sensors.hpp
```

contains sensor-related Redfish handling and D-Bus-based sensor discovery.

This gives us a practical way to approach bmcweb source code later.

---

# 18. Reading a bmcweb Route

A simplified mental model of a bmcweb route looks like:

```cpp
BMCWEB_ROUTE(app, "/redfish/v1/...")
    .methods(http::verb::get)
    (
        [](const auto& req,
           const auto& asyncResp)
        {
            // Find required D-Bus data

            // Read D-Bus properties

            // Build Redfish JSON response
        });
```

The real implementation is more sophisticated and uses helper functions, asynchronous callbacks, privileges, validation, and resource-specific logic.

When reading the source, ask:

```text
1. Which Redfish URI is this route implementing?

2. Which HTTP methods are supported?

3. Which D-Bus interfaces are being queried?

4. Which D-Bus properties are being read?

5. Is ObjectMapper/association data being used?

6. How is the Redfish JSON response constructed?

7. What happens if the D-Bus operation fails?
```

This becomes our **bmcweb source-reading method**.

---

# 19. Important: Redfish Is Not "D-Bus Over HTTP"

It is tempting to say:

```text
Redfish = D-Bus over HTTP
```

But that is not technically accurate.

Redfish and D-Bus have different models and semantics.

A better description is:

```text
D-Bus
  ↓
Internal OpenBMC data + service model

bmcweb
  ↓
Translation / resource implementation

Redfish
  ↓
Standardized external management model
```

A Redfish resource can aggregate information from multiple internal sources.

Similarly, a Redfish operation may involve multiple internal operations.

Therefore:

> **Redfish should be understood as a standardized management model, not merely a transport wrapper around D-Bus.**

---

# 20. Day 13 → Day 14 → Day 15

Now we can connect the last three days.

## Day 13 — Read the Service

```text
C++ Source
    ↓
sdbusplus
    ↓
Service
    ↓
systemd
    ↓
Running OpenBMC Component
```

We asked:

> **How does an OpenBMC component become a running service?**

## Day 14 — Understand the D-Bus API

```text
Service
    ↓
D-Bus
    ↓
Object
    ↓
Interface
    ↓
Properties / Methods / Signals
```

We asked:

> **What API does the service expose internally?**

## Day 15 — Expose It to the Outside World

```text
OpenBMC Service
       │
       │ D-Bus
       ▼
     bmcweb
       │
       │ Redfish
       ▼
Remote Management Client
```

We ask:

> **How can external management software access OpenBMC functionality?**

This is the progression we wanted.

---

# 21. Complete OpenBMC Management Flow

We can now combine the architecture we have learned so far.

```text
                    REMOTE CLIENT
                         │
                         │ HTTP / HTTPS
                         ▼
                 ┌───────────────┐
                 │    bmcweb     │
                 │ Redfish Server│
                 └───────┬───────┘
                         │
                         │ D-Bus
                         ▼
                ┌──────────────────┐
                │ OpenBMC Services │
                └────────┬─────────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
           Sensors     Power      Chassis
              │          │          │
              └──────────┼──────────┘
                         ▼
                   BMC Hardware
```

And the request flow:

```text
Client
  │
  │ GET /redfish/v1/...
  ▼
bmcweb
  │
  │ D-Bus query
  ▼
OpenBMC Service
  │
  │ property / operation
  ▼
Hardware / subsystem
  │
  ▼
OpenBMC Service
  │
  ▼
bmcweb
  │
  │ JSON
  ▼
Client
```

---

# 22. What We Learned Today

### 1. Redfish

A standardized management API for servers and infrastructure.

### 2. Redfish Service Root

```text
/redfish/v1/
```

is the starting point of a Redfish service.

### 3. Redfish Resources

Managed entities are represented using standardized resource models and schemas.

### 4. bmcweb

OpenBMC's web server provides the Redfish implementation and connects external Redfish requests with internal OpenBMC functionality.

### 5. D-Bus and Redfish are different layers

```text
D-Bus   → Internal OpenBMC communication
Redfish → External management interface
```

### 6. bmcweb is the bridge

```text
Redfish
   ↕
bmcweb
   ↕
D-Bus
```

### 7. Redfish is not simply D-Bus over HTTP

bmcweb performs resource-specific translation and can aggregate information from multiple internal sources.

---

# 23. The Mental Model to Remember

If you remember only one diagram from Day 15, remember this:

```text
                 EXTERNAL WORLD
                       │
                       │ HTTPS
                       ▼
                  ┌─────────┐
                  │ Redfish │
                  └────┬────┘
                       │
                       ▼
                  ┌─────────┐
                  │ bmcweb  │
                  └────┬────┘
                       │
                       │ D-Bus
                       ▼
               ┌───────────────┐
               │ OpenBMC       │
               │ Services      │
               └───────┬───────┘
                       │
                       ▼
                BMC Hardware
```

> **External client → Redfish → bmcweb → D-Bus → OpenBMC services → hardware**

This is one of the most important architectural connections in the OpenBMC learning series so far.

---


# 📚 References

1. OpenBMC bmcweb — Redfish documentation  
   https://github.com/openbmc/bmcweb/blob/master/docs/Redfish.md

2. OpenBMC bmcweb source repository  
   https://github.com/openbmc/bmcweb

3. OpenBMC bmcweb Chassis implementation  
   https://github.com/openbmc/bmcweb/blob/master/redfish-core/lib/chassis.hpp

4. OpenBMC bmcweb Sensors implementation  
   https://github.com/openbmc/bmcweb/blob/master/redfish-core/lib/sensors.hpp

5. DMTF Redfish Specification  
   https://www.dmtf.org/standards/redfish

6. DMTF Redfish Schema Index  
   https://redfish.dmtf.org/redfish/schema_index

7. DMTF Redfish Developer Hub  
   https://redfish.dmtf.org/

---

## 🎯 Final Takeaway

```text
Day 13
Read the OpenBMC service
        ↓
Day 14
Understand its D-Bus API
        ↓
Day 15
Understand how external clients
reach OpenBMC through Redfish
        ↓
Next
Read the actual bmcweb implementation
```

**The architecture is becoming connected:**

> **Source Code → OpenBMC Service → D-Bus API → bmcweb → Redfish → Remote Management Client**
