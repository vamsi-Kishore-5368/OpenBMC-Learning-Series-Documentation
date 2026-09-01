# OpenBMC Learning Series — Day 13

## Reading Real OpenBMC Source Code

### From Architecture Diagrams → Actual OpenBMC Code

Welcome to **Day 13** of the OpenBMC Learning Series.

Until now, we have learned OpenBMC mainly through architecture and concepts:

- D-Bus
- systemd
- OpenBMC services
- Yocto / BitBake
- Service architecture

Now we take the next step:

> **Let's open a real OpenBMC component and understand how its pieces connect.**

---

## 1. Why Read Real Source Code?

Knowing:

```text
Service → D-Bus → systemd → Yocto
```

is useful.

But as an OpenBMC developer, we eventually need to answer:

- Where does the application start?
- Where is the D-Bus connection created?
- What D-Bus object does it expose?
- What is its service/bus name?
- Where is the systemd unit?
- How does Yocto build and install it?

So instead of another theoretical example, we will follow a real component.

---

# 2. Our Example — `phosphor-user-manager`

For Day 13, we use **phosphor-user-manager** as our first source-code example.

OpenBMC's user-management architecture describes it as the common user-management service and documents its D-Bus API. The service uses the D-Bus name `xyz.openbmc_project.User.Manager`. [1]

The repository is compact enough to make it a good first source-code exercise.

---

# 3. Start With the Repository

A simplified view is:

```text
phosphor-user-manager/
│
├── mainapp.cpp
├── user_mgr.cpp
├── user_mgr.hpp
├── users.cpp
├── users.hpp
├── file.hpp
├── configure.ac
└── ...
```

We do not need to understand every file immediately.

A good source-reading strategy is:

> **Find the entry point first.**

For this component, that entry point is:

```text
mainapp.cpp
```

---

# 4. Find `main()`

The current `mainapp.cpp` contains:

```cpp
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

This small function already tells us a lot. [2]

Let's read it step by step.

---

# 5. Step 1 — Create a D-Bus Connection

```cpp
auto bus = sdbusplus::bus::new_default();
```

The application creates a D-Bus connection using `sdbusplus`.

We learned earlier:

> **D-Bus → IPC mechanism**

> **sdbusplus → C++ interface for working with D-Bus**

Conceptually:

```text
OpenBMC Application
        ↓
    sdbusplus
        ↓
       D-Bus
```

The service now has a D-Bus connection it can use.

---

# 6. Step 2 — Create the Object Manager

```cpp
sdbusplus::server::manager_t objManager(
    bus, userManagerRoot);
```

The source defines:

```cpp
constexpr auto userManagerRoot =
    "/xyz/openbmc_project/user";
```

So conceptually:

```text
D-Bus
  │
  └── /xyz/openbmc_project/user
          │
          └── User-management objects
```

The ObjectManager mechanism helps clients discover objects and their interfaces.

---

# 7. Step 3 — Create the User Manager Object

```cpp
phosphor::user::UserMgr userMgr(
    bus, userManagerRoot);
```

This creates the main user-manager object.

We can think of it as:

```text
main()
  ↓
UserMgr
  ↓
User-management functionality
```

The implementation can then be followed into:

```text
user_mgr.hpp
       ↓
user_mgr.cpp
```

This is a useful source-reading technique:

> **When `main()` creates an object, follow that class into its header and implementation.**

---

# 8. Step 4 — Claim the D-Bus Service Name

```cpp
bus.request_name(USER_MANAGER_BUSNAME);
```

The application requests ownership of its D-Bus service name.

OpenBMC documents the user-manager service as:

```text
xyz.openbmc_project.User.Manager
```

Conceptually:

```text
D-Bus
  │
  └── xyz.openbmc_project.User.Manager
              │
              ↓
       phosphor-user-manager
```

Other applications can then communicate with this service through D-Bus.

---

# 9. Step 5 — Enter the Event Loop

Finally:

```cpp
bus.process_loop();
```

The application waits for requests/events instead of immediately exiting.

Conceptually:

```text
Initialize
    ↓
Connect to D-Bus
    ↓
Create objects
    ↓
Claim service name
    ↓
Wait for D-Bus activity
```

This is what turns the application into a continuously running service.

---

# 10. The Whole `main()` in One Picture

```text
             main()
               │
               ↓
      Create D-Bus connection
               │
               ↓
       Create ObjectManager
               │
               ↓
        Create UserMgr
               │
               ↓
       Claim D-Bus name
               │
               ↓
        Enter event loop
               │
               ↓
       Running service
```

A small `main()` can therefore reveal the basic architecture of the service.

---

# 11. Where Are the D-Bus Objects?

The `main()` function does not contain all the user-management logic.

We follow:

```text
mainapp.cpp
     ↓
user_mgr.hpp
     ↓
user_mgr.cpp
     ↓
users.hpp
     ↓
users.cpp
```

For example, `users.cpp` contains a `Users` constructor that receives a `sdbusplus::bus_t&` together with a D-Bus path and other user-management state. [3]

This shows:

> **D-Bus is integrated into the application's object model, not just initialized in `main()`.**

---

# 12. Connecting Bus Name, Object Path and Interface

We can now combine the concepts from Day 10:

```text
Bus Name
   │
   ↓
xyz.openbmc_project.User.Manager
   │
   ↓
Object Path
   │
   ↓
/xyz/openbmc_project/user/...
   │
   ↓
D-Bus Interface
   │
   ├── Properties
   ├── Methods
   └── Signals
```

The exact objects and interfaces depend on the implementation.

The important skill is learning how to trace them through the source code.

---

# 13. Now Connect It to systemd

From Day 11:

> **systemd manages services.**

So our next question is:

> **How does this executable become a service that systemd can start?**

Conceptually:

```text
C++ Source
    ↓
Executable
    ↓
systemd Service Unit
    ↓
systemd
    ↓
Running Process
```

The exact service-unit files can vary between components and build configurations, but the relationship remains the same.

---

# 14. Now Connect It to Yocto / BitBake

From Day 12:

> **Yocto / BitBake builds and packages the software into the BMC image.**

Conceptually:

```text
phosphor-user-manager
        │
        ↓
      Recipe
        │
        ↓
      BitBake
        │
        ↓
      Package
        │
        ↓
   OpenBMC Image
        │
        ↓
        BMC
```

OpenBMC's package-group configuration currently lists `phosphor-user-manager` as a runtime dependency of the user-management application group. [4]

---

# 15. The Complete Picture

Now connect everything:

```text
                SOURCE CODE
                     │
                     ↓
               mainapp.cpp
                     │
                     ↓
             sdbusplus / D-Bus
                     │
                     ↓
              UserMgr Objects
                     │
                     ↓
             Service Binary
                     │
             ┌───────┴───────┐
             ↓               ↓
          systemd          D-Bus
             │               │
             ↓               ↓
       Start / Manage    Communicate
             │               │
             └───────┬───────┘
                     ↓
                Running BMC
```

And during the build:

```text
Source Repository
       ↓
Yocto / BitBake
       ↓
Package
       ↓
OpenBMC Image
       ↓
BMC
```

---

# 16. A Repeatable Method for Reading OpenBMC Code

When you open an unfamiliar OpenBMC component, don't try to understand everything at once.

Use this sequence:

### Step 1 — Find the entry point

```text
main()
```

### Step 2 — Find the D-Bus connection

```text
sdbusplus / D-Bus
```

### Step 3 — Find the objects

```text
Object / Class
```

### Step 4 — Find the interfaces

```text
Properties
Methods
Signals
```

### Step 5 — Find the identifiers

```text
Bus name
Object paths
Interface names
```

### Step 6 — Find the runtime integration

```text
systemd service
```

### Step 7 — Find the build integration

```text
Yocto / BitBake recipe
```

### Step 8 — Connect everything

```text
Source → Build → Start → Communicate
```

This method can be reused for many OpenBMC components.

---

# 17. We Are NOT Going to Read Every Recipe

There are many OpenBMC components and recipes.

We do **not** need to study every single one.

Instead, we will selectively examine important components to understand recurring patterns.

For example:

```text
Simple Service
      ↓
D-Bus Service
      ↓
Sensor Service
      ↓
State Management
      ↓
Redfish / bmcweb
      ↓
Entity Manager
      ↓
Firmware Management
```

The goal is not to memorize individual recipes.

The goal is:

> **Learn how to understand an unfamiliar OpenBMC component.**

---

# 18. Day 13 Mental Model

Remember:

> **Find the entry point → follow D-Bus → follow the objects → find systemd → find the recipe.**

Or simply:

```text
          REAL OPENBMC COMPONENT

               Repository
                   ↓
                main()
                   ↓
              Application
                   ↓
              sdbusplus
                   ↓
                 D-Bus
                   ↓
               systemd
                   ↓
            Running Service
                   ↑
             Yocto / BitBake
```

We have now moved from:

> **"I know what OpenBMC services are."**

to:

> **"I know how to start reading an actual OpenBMC service."**

---

## 🔜 Day 14

# D-Bus in Real OpenBMC Code

We will go one level deeper into the code we just saw:

- Bus name
- Object path
- Interface
- Properties
- Methods
- Signals
- ObjectManager
- How another service communicates with the object

**From reading the service → to understanding its D-Bus API.**

One concept at a time. 🚀

---

## References

1. OpenBMC — User Management Architecture  
   https://github.com/openbmc/docs/blob/master/architecture/user-management.md

2. OpenBMC — `phosphor-user-manager/mainapp.cpp`  
   https://github.com/openbmc/phosphor-user-manager/blob/master/mainapp.cpp

3. OpenBMC — `phosphor-user-manager/users.cpp`  
   https://github.com/openbmc/phosphor-user-manager/blob/master/users.cpp

4. OpenBMC — `packagegroup-obmc-apps.bb`  
   https://github.com/openbmc/openbmc/blob/master/meta-phosphor/recipes-phosphor/packagegroups/packagegroup-obmc-apps.bb

5. OpenBMC Documentation  
   https://github.com/openbmc/docs
