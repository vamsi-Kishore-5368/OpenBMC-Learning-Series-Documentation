# OpenBMC Learning Series — Day 14

## D-Bus in Real OpenBMC Code — From D-Bus Concepts to an Actual D-Bus API

### From a Running OpenBMC Service to the API Other Components Actually Use

------------------------------------------------------------------------

## Introduction

In **Day 13**, we moved from OpenBMC architecture diagrams to real
source code.

We studied `phosphor-user-manager` and followed its entry point:

``` text
main()
  ↓
Create D-Bus connection
  ↓
Create ObjectManager
  ↓
Create UserMgr object
  ↓
Request D-Bus service name
  ↓
Enter event loop
```

That answered:

> **How does an OpenBMC C++ program become a running D-Bus service?**

Now we need to answer the next question:

> **Once the service is running, what exactly does it expose to the rest
> of OpenBMC?**

That is the purpose of this day.

We will connect the D-Bus concepts from Day 10 to real OpenBMC
implementation:

``` text
D-Bus Service
      ↓
Object Path
      ↓
Interface
      ↓
Properties / Methods / Signals
      ↓
ObjectManager
      ↓
Other OpenBMC Services / Clients
```

The goal is not to repeat the basic D-Bus introduction. Instead, we will
learn how to **read a real OpenBMC D-Bus API and understand how another
component can interact with it**.

------------------------------------------------------------------------

# 1. Quick Bridge from Day 13

In Day 13, we studied this real OpenBMC source:

``` cpp
int main(int /*argc*/, char** /*argv*/)
{
    auto bus = sdbusplus::bus::new_default();

    sdbusplus::server::manager_t objManager(
        bus, userManagerRoot);

    phosphor::user::UserMgr userMgr(
        bus, userManagerRoot);

    bus.request_name(USER_MANAGER_BUSNAME);

    bus.process_loop();
}
```

The important pieces were:

``` text
sdbusplus::bus::new_default()
        ↓
Connect to D-Bus

server::manager_t
        ↓
ObjectManager support

UserMgr
        ↓
Actual application/service object

request_name()
        ↓
Claim a D-Bus service name

process_loop()
        ↓
Wait for D-Bus requests
```

So the process becomes a running D-Bus service.

But what does a client actually see?

To answer that, we need to understand the D-Bus object model.

------------------------------------------------------------------------

# 2. The Five Things You Must Understand

When reading an OpenBMC D-Bus API, keep this mental model:

``` text
┌──────────────────────────────────────┐
│ Service Name                         │
│ "Who provides the API?"             │
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│ Object Path                          │
│ "Which object?"                     │
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│ Interface                            │
│ "Which API contract?"               │
└───────────────┬─────────┬────────────┘
                ↓         ↓
        ┌──────────┐  ┌──────────┐
        │Properties│  │ Methods  │
        │  State   │  │Actions   │
        └──────────┘  └──────────┘
                                                   ↓
             ┌──────────┐
             │ Signals  │
             │  Events  │
             └──────────┘
```

A useful way to remember it:

| D-Bus element    | Think of it as                                   |
|------------------|--------------------------------------------------|
| **Service Name** | The application/service providing the API        |
| **Object Path**  | A particular object managed by that service      |
| **Interface**    | The API contract implemented by the object       |
| **Property**     | State/data exposed by the interface              |
| **Method**       | An operation that a client can request           |
| **Signal**       | An event notification sent to interested clients |

These concepts are related, but they are **not interchangeable**.

------------------------------------------------------------------------

# 3. D-Bus Service Name — Who Provides the API?

A D-Bus service normally owns a **well-known bus name**.

For example, OpenBMC components use names such as:

``` text
xyz.openbmc_project.ObjectMapper
```

The `phosphor-user-manager` source also requests its configured service
name:

``` cpp
bus.request_name(USER_MANAGER_BUSNAME);
```

The exact string is supplied through the component’s configuration/build
system rather than being hard-coded in this `mainapp.cpp` snippet.

The important idea is:

``` text
Service Name
      ↓
Identifies the service/application
      ↓
Provides D-Bus objects and interfaces
```

A client can address a service by its well-known name instead of
depending on a temporary process-specific identity.

### Important distinction

A **service name is not the same thing as an object path**.

For example:

``` text
Service:
xyz.openbmc_project.SomeService

Object:
/xyz/openbmc_project/some_object
```

The service identifies **who provides** the object.

The object path identifies **which object** is being addressed.

------------------------------------------------------------------------

# 4. Object Path — Which Object?

An OpenBMC D-Bus service can expose multiple objects.

Objects are identified by **object paths**.

For example, the user manager creates objects under:

``` text
/xyz/openbmc_project/user
```

Individual user-related objects can exist below that namespace.

Conceptually:

``` text
/xyz/openbmc_project/user
        │
        ├── user1
        ├── user2
        └── user3
```

The exact objects present depend on system state.

An object path therefore answers:

> **Which object are we talking about?**

OpenBMC commonly organizes related objects beneath namespaces such as:

``` text
/xyz/openbmc_project/inventory
/xyz/openbmc_project/sensors
/xyz/openbmc_project/chassis
/xyz/openbmc_project/state
```

This hierarchy is useful for discovering and managing groups of related
objects.

------------------------------------------------------------------------

# 5. D-Bus Interface — What API Does the Object Expose?

An object can implement one or more **interfaces**.

Think of an interface as the API contract:

``` text
Object
  ↓
Interface
  ├── Properties
  ├── Methods
  └── Signals
```

OpenBMC’s `phosphor-dbus-interfaces` project stores standard interface
descriptions as YAML files.

An interface definition can describe:

``` yaml
description:
    ...

methods:
    ...

properties:
    ...

signals:
    ...
```

These definitions can be consumed by the `sdbus++` binding generator to
produce C++ bindings.

So there is an important relationship:

``` text
D-Bus Interface YAML
        ↓
      sdbus++
        ↓
Generated C++ bindings
        ↓
OpenBMC service implementation
        ↓
D-Bus API
```

This helps OpenBMC maintain consistent D-Bus interfaces across
components.

------------------------------------------------------------------------

# 6. Properties — The State of an Object

A D-Bus property represents data/state associated with an object and
interface.

For example, an interface might expose:

``` text
Present = true
Value = 42.5
Enabled = true
Status = "Running"
```

Conceptually:

``` text
Object
  ↓
Interface
  ↓
Properties
  ├── Property A
  ├── Property B
  └── Property C
```

A client can read a property without needing to know how the service
stores the underlying value internally.

For example:

``` text
Client
  ↓
Read Property
  ↓
D-Bus
  ↓
Service
  ↓
Return current value
```

This creates a useful separation:

``` text
Internal implementation
        ≠
External D-Bus API
```

A service can change its internal C++ implementation while preserving
the D-Bus contract expected by clients.

------------------------------------------------------------------------

# 7. Methods — Requesting an Operation

A D-Bus method represents an operation that a client can request from an
object.

Conceptually:

``` text
Client
   │
   │ Method Call
   ↓
D-Bus
   │
   ↓
Service Object
   │
   │ Execute operation
   ↓
Result / Error
```

A real OpenBMC example is the:

``` text
xyz.openbmc_project.Control.Host
```

interface.

Its current interface definition includes:

``` text
Method:
    Execute

Signals:
    CommandComplete

Properties:
    command
    result
```

The important distinction is:

``` text
Property → "What is the current state?"

Method   → "Please perform this operation."

Signal   → "Something happened."
```

This distinction is extremely important when reading OpenBMC code.

------------------------------------------------------------------------

# 8. Signals — Event Notifications

A signal is different from a method call.

A method is normally:

``` text
Client → Service
```

A signal is normally:

``` text
Service → Interested Clients
```

Conceptually:

``` text
Service
   │
   │ Event occurred
   ↓
D-Bus Signal
   │
   ├── Client A
   ├── Client B
   └── Client C
```

A client does not have to repeatedly ask:

``` text
"Did something change?"
"Did something change?"
"Did something change?"
```

Instead, it can subscribe to relevant signals.

Events can include:

``` text
Object added
Object removed
Property changed
Operation completed
State transition
```

One standard mechanism is:

``` text
org.freedesktop.DBus.Properties.PropertiesChanged
```

for property changes.

ObjectManager also defines:

``` text
InterfacesAdded
InterfacesRemoved
```

for changes to objects/interfaces in a managed object tree.

------------------------------------------------------------------------

# 9. ObjectManager — Managing Many Objects

Imagine a service manages:

``` text
100 sensors
50 inventory objects
20 user objects
10 control objects
```

A client needs an efficient way to discover these objects.

This is where:

``` text
org.freedesktop.DBus.ObjectManager
```

is useful.

The standard ObjectManager API provides:

``` text
GetManagedObjects()
```

which can return the objects, interfaces, and properties managed below
an ObjectManager path.

Conceptually:

``` text
ObjectManager
      │
      ├── Object 1
      │     ├── Interface A
      │     └── Interface B
      │
      ├── Object 2
      │     └── Interface A
      │
      └── Object 3
            ├── Interface A
            └── Interface C
```

This is particularly useful for services managing collections of related
objects.

------------------------------------------------------------------------

# 10. ObjectManager in Real OpenBMC Code

Go back to the Day 13 `mainapp.cpp`:

``` cpp
sdbusplus::server::manager_t objManager(
    bus, userManagerRoot);
```

This line creates an `sdbusplus` ObjectManager at:

``` text
/xyz/openbmc_project/user
```

Conceptually:

``` text
/xyz/openbmc_project/user
        │
        │ ObjectManager
        │
        ├── User object
        ├── User object
        ├── User object
        └── ...
```

The standard D-Bus ObjectManager model provides:

``` text
GetManagedObjects()
InterfacesAdded
InterfacesRemoved
```

This gives clients a standard way to discover and track objects in a
managed subtree.

------------------------------------------------------------------------

# 11. OpenBMC ObjectMapper — Finding the Right Service

There is another OpenBMC-specific component that becomes important when
working with D-Bus:

``` text
xyz.openbmc_project.ObjectMapper
```

ObjectMapper helps applications discover relationships between:

``` text
Object Path
      ↓
Service Name
      ↓
Interfaces
```

For example, a client may know:

``` text
"I need the service that implements this interface
for an object somewhere under this subtree."
```

Instead of hard-coding every service name, it can query the
ObjectMapper.

OpenBMC’s ObjectMapper provides APIs such as:

``` text
GetObject
GetSubTree
GetSubTreePaths
GetAssociatedSubTree
```

Conceptually:

``` text
Client
  │
  │ "Who provides this object/interface?"
  ↓
ObjectMapper
  │
  ├── Service Name
  ├── Object Path
  └── Interfaces
```

This is important in a dynamic OpenBMC system where services and objects
can vary with platform configuration and runtime state.

------------------------------------------------------------------------

# 12. The Complete D-Bus Model

Now combine everything:

``` text
                    D-Bus System Bus
                           │
             ┌─────────────┴─────────────┐
             │                           │
        Service A                   Service B
             │                           │
      Service Name                 Service Name
             │                           │
        Object Path                  Object Path
             │                           │
      ┌──────┴──────┐             ┌──────┴──────┐
      │             │             │             │
 Interface A   Interface B    Interface A   Interface C
      │             │             │             │
   Properties     Methods     Properties      Signals
      │             │             │             │
      └─────────────┴─────────────┴─────────────┘
                           │
                     Other Clients
```

The core hierarchy is:

``` text
Service Name
      ↓
Object Path
      ↓
Interface
      ↓
Properties / Methods / Signals
```

ObjectManager and ObjectMapper then provide mechanisms for discovering
and working with those objects and services.

------------------------------------------------------------------------

# 13. A Real OpenBMC Example: Host Control Interface

Let’s use a real interface definition instead of inventing a
hypothetical API.

OpenBMC’s:

``` text
xyz.openbmc_project.Control.Host
```

interface defines:

``` text
Method:
    Execute

Signals:
    CommandComplete

Properties:
    command
    result
```

The interface also defines command/result enumerations.

Conceptually:

``` text
xyz.openbmc_project.Control.Host
             │
       ┌─────┼───────────┐
       ↓     ↓           ↓
    Method  Properties  Signal
    Execute  command     CommandComplete
             result
```

This illustrates the three fundamental interaction styles:

``` text
Method
  → Ask the service to do something

Property
  → Read or represent state

Signal
  → Notify clients about an event
```

------------------------------------------------------------------------

# 14. From Interface Definition to C++ Code

Now connect this with `sdbusplus`.

OpenBMC maintains interface definitions in YAML:

``` text
phosphor-dbus-interfaces/
        │
        └── yaml/
             └── xyz/
                  └── openbmc_project/
                       └── ...
```

An interface YAML can contain:

``` yaml
methods:
    ...

properties:
    ...

signals:
    ...
```

Then:

``` text
YAML Interface
      ↓
sdbus++
      ↓
Generated C++ bindings
      ↓
Service class
      ↓
D-Bus
```

The service implementation can use generated interface classes,
depending on the component’s design.

This is why, when reading OpenBMC code, you may encounter types that
look like:

``` cpp
sdbusplus::xyz::openbmc_project::...
```

Those types are often generated from D-Bus interface definitions.

------------------------------------------------------------------------

# 15. How a Client Interacts with a Service

A client generally follows this conceptual sequence:

``` text
1. Identify the object/interface
             ↓
2. Find the providing service
             ↓
3. Create a D-Bus proxy
             ↓
4. Read a property
      OR
   call a method
      OR
   subscribe to a signal
             ↓
5. Service processes request
             ↓
6. Client receives result/event
```

In real OpenBMC applications, `sdbusplus` is commonly used for C++ D-Bus
communication.

The important point is:

> The client does not need to know the service’s internal C++
> implementation. It needs to understand the D-Bus contract.

------------------------------------------------------------------------

# 16. Exploring the API with `busctl`

We introduced `busctl` in Day 10.

Now we can use it with the mental model learned here.

### List services

``` bash
busctl list
```

``` text
→ Lists services currently present on the D-Bus.
```

### Explore an object’s tree

``` bash
busctl tree <service>
```

``` text
→ Shows the object paths exposed by a service.
```

### Inspect an object

``` bash
busctl introspect <service> <object-path>
```

``` text
→ Shows the interfaces, methods, properties, and signals exposed by that object.
```

### Read a property

``` bash
busctl get-property     <service>     <object-path>     <interface>     <property>
```

``` text
→ Reads a specific property from an interface.
```

### Monitor D-Bus traffic

``` bash
busctl monitor
```

``` text
→ Monitors D-Bus messages and lets you observe communication as it happens.
```

A useful investigation pattern is:

``` text
busctl list
      ↓
Find service
      ↓
busctl tree
      ↓
Find object path
      ↓
busctl introspect
      ↓
Find interface
      ↓
Inspect properties/methods/signals
```

This is one of the most practical workflows for debugging OpenBMC.

------------------------------------------------------------------------

# 17. Trace One Request End-to-End

Now put everything together.

Suppose an OpenBMC client wants to request an operation.

The high-level flow is:

``` text
┌───────────────┐
│ Client        │
└───────┬───────┘
        │
        │ D-Bus method call
        ↓
┌────────────────────────┐
│ D-Bus System Bus       │
└──────────┬─────────────┘
           │
           ↓
┌────────────────────────┐
│ Service Name           │
│ Identifies provider    │
└──────────┬─────────────┘
           │
           ↓
┌────────────────────────┐
│ Object Path            │
│ Identifies object      │
└──────────┬─────────────┘
           │
           ↓
┌────────────────────────┐
│ Interface              │
│ Identifies API         │
└──────────┬─────────────┘
           │
           ↓
┌────────────────────────┐
│ Method                 │
│ Performs operation     │
└──────────┬─────────────┘
           │
           ↓
      Service Logic
           │
           ↓
     Result / Error
           │
           ↓
        Client
```

If the operation causes an event, the service can additionally emit a
signal:

``` text
Service
   │
   ↓
D-Bus Signal
   │
   ├── Client A
   ├── Client B
   └── Client C
```

This is the basic communication pattern behind a large part of OpenBMC’s
internal software architecture.

------------------------------------------------------------------------

# 18. What to Look for When Reading OpenBMC Source

When you open a new OpenBMC component, don’t immediately read every
file.

Instead, ask these questions.

### Step 1 — What service name does it own?

Search for:

``` cpp
request_name(...)
```

or the configured service name.

------------------------------------------------------------------------

### Step 2 — What is the root object path?

Look for constants such as:

``` cpp
constexpr auto root = "...";
```

or constructor arguments containing D-Bus paths.

------------------------------------------------------------------------

### Step 3 — Which interfaces are implemented?

Look for generated interface types or classes derived from interface
bindings.

------------------------------------------------------------------------

### Step 4 — What properties exist?

Find property declarations and generated interface definitions.

Ask:

``` text
What state does this service expose?
```

------------------------------------------------------------------------

### Step 5 — What methods exist?

Look for methods exposed through generated bindings or registered
explicitly.

Ask:

``` text
What can another component request this service to do?
```

------------------------------------------------------------------------

### Step 6 — What signals exist?

Ask:

``` text
What events can this service notify other components about?
```

------------------------------------------------------------------------

### Step 7 — Is ObjectManager used?

Look for:

``` cpp
sdbusplus::server::manager_t
```

This is a strong clue that the service manages a subtree of D-Bus
objects.

------------------------------------------------------------------------

### Step 8 — Does it use ObjectMapper?

Look for:

``` text
xyz.openbmc_project.ObjectMapper
```

This often indicates that the component needs to discover
services/interfaces dynamically.

------------------------------------------------------------------------

# 19. The Day 13 → Day 14 Connection

This is the most important connection in this part of the series.

### Day 13

We learned:

``` text
How an OpenBMC source file
becomes a running service
```

Specifically:

``` text
main()
 ↓
sdbusplus bus
 ↓
ObjectManager
 ↓
Application object
 ↓
request_name()
 ↓
process_loop()
```

### Day 14

We learned:

``` text
What that running service exposes
to the rest of OpenBMC
```

Specifically:

``` text
Service Name
 ↓
Object Path
 ↓
Interface
 ↓
Properties / Methods / Signals
 ↓
ObjectManager
 ↓
Clients
```

Together:

``` text
             SOURCE CODE
                  │
                  ↓
             OpenBMC Service
                  │
                  ↓
             D-Bus Service
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
  Object Paths          Interfaces
                            │
                  ┌─────────┼─────────┐
                  ↓         ↓         ↓
             Properties  Methods   Signals
```

This is the bridge between **C++ implementation** and **OpenBMC’s
inter-process API**.

------------------------------------------------------------------------

# 20. A Practical Mental Model

Whenever you see an OpenBMC D-Bus API, translate it mentally like this:

``` text
Service Name
     ↓
"Who owns it?"

Object Path
     ↓
"Which object?"

Interface
     ↓
"Which API contract?"

Property
     ↓
"What state/data?"

Method
     ↓
"What operation?"

Signal
     ↓
"What event?"

ObjectManager
     ↓
"How do I discover/manage this object tree?"

ObjectMapper
     ↓
"Which service provides this object/interface?"
```

If you can answer those questions, you can understand a large portion of
an OpenBMC component without reading every line of its implementation.

------------------------------------------------------------------------

# 21. Key Takeaways

1.  **A D-Bus service exposes APIs, not its internal implementation.**
2.  **The service name identifies the provider.**
3.  **The object path identifies a particular object.**
4.  **The interface defines the API contract.**
5.  **Properties represent object/interface state and data.**
6.  **Methods represent operations requested by clients.**
7.  **Signals provide asynchronous event notifications.**
8.  **ObjectManager helps clients discover collections of managed
    objects.**
9.  **OpenBMC ObjectMapper helps applications discover services and
    interfaces for object paths.**
10. **`sdbusplus` provides the C++ D-Bus library, while `sdbus++`
    generates bindings from interface definitions.**
11. **`busctl` is one of the most useful tools for inspecting a running
    OpenBMC D-Bus system.**
12. **Day 13 showed how a service starts; Day 14 shows what API that
    service exposes.**

------------------------------------------------------------------------

# 22. What’s Next?

Now that we understand:

``` text
OpenBMC Source Code
        ↓
OpenBMC Service
        ↓
D-Bus Service
        ↓
Objects
        ↓
Interfaces
        ↓
Properties / Methods / Signals
```

the next step is to follow a **real OpenBMC operation across multiple
components**.

For example:

``` text
External request
      ↓
Redfish / IPMI
      ↓
OpenBMC application
      ↓
D-Bus
      ↓
Another OpenBMC service
      ↓
Hardware interaction
      ↓
D-Bus state/event
      ↓
Response
```

That will move us from understanding **individual services** to
understanding how **multiple OpenBMC services cooperate to perform an
actual system-management operation**.

------------------------------------------------------------------------

# References

- OpenBMC `phosphor-user-manager` — `mainapp.cpp`  
  https://github.com/openbmc/phosphor-user-manager/blob/master/mainapp.cpp

- OpenBMC `phosphor-user-manager`  
  https://github.com/openbmc/phosphor-user-manager

- OpenBMC `sdbusplus` README  
  https://github.com/openbmc/sdbusplus/blob/master/README.md

- OpenBMC `sdbusplus` Interface YAML documentation  
  https://github.com/openbmc/sdbusplus/blob/master/docs/yaml/interface.md

- OpenBMC `phosphor-dbus-interfaces`  
  https://github.com/openbmc/phosphor-dbus-interfaces

- OpenBMC `xyz.openbmc_project.Control.Host` interface  
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Control/Host.interface.yaml

- OpenBMC ObjectMapper documentation  
  https://github.com/openbmc/docs/blob/master/architecture/object-mapper.md

- D-Bus Specification — ObjectManager  
  https://dbus.freedesktop.org/doc/dbus-specification.html

------------------------------------------------------------------------

## Final Mental Model

``` text
                 OpenBMC Application
                         │
                         ↓
                  D-Bus Service
                         │
                  Service Name
                         │
                         ↓
                    Object Path
                         │
                         ↓
                     Interface
                         │
             ┌───────────┼───────────┐
             ↓           ↓           ↓
        Properties     Methods     Signals
             │           │           │
             └───────────┴───────────┘
                         │
                         ↓
                  Other Services
                         │
                         ↓
                    OpenBMC System
```

> **Day 13 taught us how to read an OpenBMC service.  
> Day 14 taught us how that service exposes its D-Bus API.**
